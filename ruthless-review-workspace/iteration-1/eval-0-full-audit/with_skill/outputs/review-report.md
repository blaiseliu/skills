# Ruthless Review: Lectio Divina API

## 1. Executive Summary (CEO Lens)

The single biggest systemic risk in this codebase is zero transient fault handling across four separate layers — API functions, service layer, repository layer, and SDK client. Every outbound HTTP call and every database connection is a single point of failure. A 200ms network blip or a PostgreSQL connection timeout will produce a user-facing 500, not a retry. In a serverless architecture where cold starts and connection churn are normal, this isn't a theoretical concern — it's a production incident waiting for the wrong millisecond.

The hidden cost is contract sprawl. The same verse response type exists in four separate locations (Core DTOs, Client DTOs, SDK DTOs, and TypeScript interfaces), each with slightly different property names and type conventions. Every API change requires manual synchronization across all four locations. The React app (`lectio-memorize`) and Blazor app (`LectioDivina.Client`) share no code and use completely separate auth systems — Supabase for React, Entra External ID for Blazor. You are paying the maintenance cost of two applications while getting the feature velocity of one.

Verdict: This codebase is not healthy enough to scale without a dedicated stabilization investment. The architecture is conceptually sound — Clean Architecture with proper layer separation, well-structured functions, reasonable DI wiring. But the operational gaps (retry, observability, contract management) will compound every time a new feature or a new client SDK is added. I'd estimate 2-3 weeks of focused resilience and contract consolidation work before adding significant new capabilities.

---

## 2. Architectural Breakdown (Tech Lead Lens)

### Pillar 1: Macro Architecture & Contracts

**Finding: DTO quadruplication across tiers (CRITICAL)**

`src/LectioDivina.Core/.../SingleVerseRequest.cs` (class with `{ get; set; }`)
`src/LectioDivina.Client/Models/Requests/SingleVerseRequest.cs` (record with `[property: Required]`)
`sdk/dotnet/.../LectioDivinaClient.cs:VerseInfo` (class, different naming — `TranslationDetails` vs `TranslationInfo`)
`lectio-memorize/src/types/index.ts:VerseInfo` (TypeScript interface, different shape with `id: number` vs `Code: string`)

Four independent representations of the same API contract. The naming differences are especially dangerous — what the Core calls `TranslationInfo`, the SDK calls `TranslationDetails`, and the Core also uses `TranslationText` in the parallel response. A developer updating the copyright field has to find and change it in four places, and the only way to catch a miss is a runtime deserialization failure.

**Finding: CLAUDE.md explicitly documents the duplication but doesn't flag it as debt**

From `CLAUDE.md` line: "When updating API contracts, update both sets of DTOs." This tells developers to do manual synchronization. The correct solution is a single source of truth with generated clients.

### Pillar 2: Backend / API Tier

**Finding: `BuildServiceProvider()` inside `ConfigureServices` creates a second DI container (CRITICAL)**

`src/LectioDivina.Api/Program.cs:44-46`:
```csharp
services.AddSingleton(new DatabaseConfiguration(
    connectionString,
    services.BuildServiceProvider().GetRequiredService<ILogger<DatabaseConfiguration>>()
));
```
This call to `BuildServiceProvider()` builds an entire second dependency injection container just to resolve one `ILogger`. In .NET, this is a documented anti-pattern that:
- Creates duplicate singleton instances (two `ILogger<DatabaseConfiguration>` instances exist)
- Causes validation errors in modern .NET versions (the container warns about this)
- Makes `DatabaseConfiguration` a captive dependency — it's registered as Singleton but was resolved from a temporary container that gets disposed

The fix is straightforward — use the factory overload:
```csharp
services.AddSingleton(sp => new DatabaseConfiguration(
    connectionString,
    sp.GetRequiredService<ILogger<DatabaseConfiguration>>()
));
```

**Finding: Repositories log and rethrow — adding noise, not resilience (HIGH)**

`src/LectioDivina.Data/.../VerseRepository.cs:42-47` (repeated in every method):
```csharp
catch (Exception ex)
{
    _logger.LogError(ex, "Error getting verse ...");
    throw;
}
```
This pattern adds zero resilience — the exception is logged at the repository level, then logged again at the function level (`SingleVerseFunction.cs:67`), then returned as a generic 500. Two log entries for one failure, no retry logic. If you add retry (e.g., Polly), it belongs in the repository where you can distinguish between transient Postgres errors (deadlocks, connection timeouts — retryable) and permanent errors (bad SQL, constraint violations — don't retry).

**Finding: `new HttpClient()` per serverless invocation (HIGH)**

`lectio-memorize/functions/GetVerse.cs:23` and `GetBooks.cs:19`:
```csharp
var client = new LectioDivinaClient(baseUrl, apiKey);
```
`LectioDivinaClient` internally creates `new HttpClient { BaseAddress = ... }`. In a serverless function, each invocation creates a new HttpClient — which creates a new socket — which isn't immediately released when the function completes because of TCP `TIME_WAIT`. Under any load beyond trivial, this exhausts the available ephemeral ports. The fix is `IHttpClientFactory` with the `AddHttpClient<T>()` pattern (already used correctly in the Blazor client's `Program.cs` for the Refit client and Entra auth).

The `Program.cs` of `lectio-memorize/functions/` has an empty `ConfigureServices` block:
```csharp
.ConfigureServices(services =>
{
    // No DB needed — just proxies to the Lectio Divina API
})
```
This is where the `IHttpClientFactory` registration should go. The `LectioDivinaClient` already has a constructor that accepts `HttpClient` — but nothing uses it.

**Finding: N+1 fallback queries — either dead code or a mapping bug (MEDIUM)**

`src/LectioDivina.Services/.../VerseService.cs:42-43`:
```csharp
var book = verse.Book ?? await _bookRepository.GetByIdAsync((short)request.Book);
var version = verse.Version ?? await _versionRepository.GetByCodeAsync(request.Version);
```
The repository's SQL JOINs both `books` and `versions` with `splitOn: "Id,Id"`. If the Dapper multi-mapping is correct, `verse.Book` and `verse.Version` should never be null — making these fallback queries dead code. If they are null (because multi-mapping fails on certain result shapes), that's a real bug being masked by the fallback rather than fixed. Either way, this should be resolved definitively: either remove the dead code or fix the mapping.

### Pillar 3: Frontend Tier(s)

**Finding: Raw fetch with no resilience in React (HIGH)**

`lectio-memorize/src/lib/api.ts:5-10`:
```typescript
async function get<T>(path: string): Promise<T> {
  const res = await fetch(`${API_BASE}/api${path}`);
  if (!res.ok) throw new Error(`API error: ${res.status}`);
  const json = await res.json();
  return json.data;
}
```
No timeout, no retry, no request deduplication. If a user navigates to a page that calls `fetchVerse()` three times in rapid succession (common in a memorization UI loading a stack), all three calls hit the wire independently. A single network hiccup becomes a broken UI. Compare this to the Blazor client's `ApiErrorHandler` at `src/LectioDivina.Client/Services/ApiErrorHandler.cs`, which provides centralized error handling with user-facing snackbar messages.

**Finding: Fake VerseRangeResponse adapter hack (MEDIUM)**

`src/LectioDivina.Client/Pages/SingleVerse.razor.cs:114-120`:
```csharp
var fakeRange = new VerseRangeResponse(
    _response.Reference, _response.Translation,
    [new VerseResponseItem((short)_selectedBookId, _selectedChapter, _selectedVerse, _response.Text)],
    _response.Copyright);
return VerseFormatter.Format(fakeRange, ...);
```
A single verse is wrapped into a `VerseRangeResponse` with a one-element list just to call `IVerseFormatter.Format()`. The formatter interface has no method for formatting a single verse. This is a design flaw in the formatting abstraction — it conflates "format verse data" with "format a range." The formatter should accept a common input type, or have a single-verse overload.

### Pillar 4: Operational Resilience & Economics

**Finding: No correlation IDs or structured observability (HIGH)**

There is no trace ID propagating from frontend to backend. The Blazor client's Refit calls go through `CookieCredentialsHandler`, `TokenAuthorizationHandler`, and `ApiErrorHandler` — none add a correlation ID header. The API functions log with structured parameters (good) but don't extract or propagate a request ID. The `ResponseEnvelopeHelper` wraps responses in `{ data, meta: { request_id } }` but the `request_id` appears to be auto-generated per response, not traced through the call chain.

When a user reports "I got an error on the verse lookup page at 2:14 PM," there is no way to find the corresponding server log entry without guessing based on timestamp proximity.

**Finding: API key in query string (LOW)**

`sdk/dotnet/.../LectioDivinaClient.cs:97`:
```csharp
var url = $"{path}?code={_apiKey}";
```
The SDK appends the API key as a query parameter. This means the key appears in:
- Server access logs
- Any proxy logs between client and server
- Browser network tabs (if used from a browser)
- Potentially in `Referer` headers

The middleware at `src/LectioDivina.Api/Middleware/ApiKeyMiddleware.cs` already supports Bearer token authentication (`Authorization: Bearer sk_live_...`), which doesn't have this problem. The SDK just needs to use it instead of the `?code=` approach.

---

## 3. High-Priority Backlog

| # | Task | Category | Impact | Complexity | Justification |
|---|------|----------|--------|------------|---------------|
| 1 | Add retry/circuit-breaker to all outbound calls | Resilience | High | Medium | 14 unguarded outbound calls across 4 layers; one network blip causes user-facing 500 |
| 2 | Replace `BuildServiceProvider()` with factory lambda | Architecture | High | Low | Second DI container is a known anti-pattern; 2-line fix prevents duplicate singletons |
| 3 | Consolidate DTOs to single source of truth with client generation | Architecture | High | High | 4 copies of same types; every API change risks silent deserialization failures |
| 4 | Add `IHttpClientFactory` to lectio-memorize functions | Resilience | High | Low | `new HttpClient()` per invocation will exhaust ports under load |
| 5 | Add correlation ID propagation (frontend → API → DB) | Observability | High | Medium | Currently impossible to trace a user-reported error to a server log |
| 6 | Add timeout and retry to React fetch layer | Resilience | Medium | Low | Raw fetch with no retry; any network blip breaks the memorization UI |
| 7 | Resolve N+1 fallback queries — remove or fix mapping | Backend | Medium | Low | Either dead code inflating every request by 2 optional queries, or a Dapper mapping bug |
| 8 | Consolidate auth to one system (Supabase or Entra, not both) | Architecture | Medium | High | Two parallel auth systems double the attack surface and maintenance burden |
| 9 | Add single-verse overload to IVerseFormatter | Frontend | Low | Low | Eliminates the hack of wrapping a single verse in a VerseRangeResponse |
| 10 | Switch SDK to Bearer token auth instead of query string | Security | Low | Low | API key in query string is visible in logs and proxy traces; Bearer already supported |
