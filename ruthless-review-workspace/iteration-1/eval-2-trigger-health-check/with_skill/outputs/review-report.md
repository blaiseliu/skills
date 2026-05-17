# Ruthless Review: System Health Check — Lectio Divina API

## 1. Executive Summary (CEO Lens)

The single biggest risk in this architecture is the absence of transient fault handling at every layer. The API has 14+ outbound calls (PostgreSQL queries, Entra identity provider HTTP calls, SDK proxy calls) and not one of them has a retry policy, circuit breaker, or configurable timeout. In a serverless architecture where cold starts are normal and the underlying infrastructure is constantly cycling, this means every deployment, every scale-out event, and every network hiccup will produce user-facing 500 errors. A single PostgreSQL connection timeout or a 200ms identity provider blip during the auth flow is indistinguishable from a critical outage to the end user.

The second risk — quieter but more expensive over time — is contract fragmentation. The same API response types are maintained in four separate code locations with slightly different shapes and naming conventions. The Blazor client, the .NET SDK, the TypeScript types, and the Core DTOs all define their own versions of `VerseResponse`, `TranslationInfo`, and `BookInfo`. A field added to the API response must be manually synchronized across all four, and the only way to catch a miss is a runtime deserialization failure. If you plan to add another client SDK (Python, Swift), each new SDK multiplies this maintenance cost.

The third risk is the two-function-app split. The main API (`LectioDivina.Api`) and the React proxy layer (`lectio-memorize/functions`) are deployed as separate Azure Function Apps that share no code, no configuration, and no deployment pipeline. The proxy layer creates `new HttpClient()` per invocation — a known serverless anti-pattern that exhausts ephemeral ports under load. When the main API's health check endpoint changes, the proxy layer's health assumptions become stale with no automated detection.

These three risks interact: a cold start triggers the port exhaustion bug, which triggers the missing retry logic, which produces an error the user can't diagnose because there are no correlation IDs.

---

## 2. Architectural Breakdown (Tech Lead Lens)

### Risk 1: Zero Transient Fault Handling (Critical)

The call chain for a single verse lookup crosses four layers, none of which retry:

| Layer | File | Transient handling |
|-------|------|--------------------|
| Function | `SingleVerseFunction.cs:16` | Catches `Exception`, returns 500. No retry. |
| Service | `VerseService.cs:21` | No try/catch at all — lets exceptions propagate |
| Repository | `VerseRepository.cs:42` | Catches, logs, rethrows. No retry. |
| Connection | `DatabaseConfiguration.cs:22` | `connection.Open()` with no retry, no timeout |

The same pattern exists in the auth flow: `AuthFunction.cs` makes 3 sequential calls to the identity provider (`InitiateAsync`, `ChallengeOobAsync`, `TokenWithOtpAsync`) — if any one fails, the user gets an error page and starts over. The `ApiErrorHandler` in the Blazor client shows a snackbar with the raw error message; there is no retry button, no exponential backoff, no graceful degradation.

**What would fix it:** Polly retry policies at the repository layer (for Npgsql transient exceptions: class 08XXX, 40001, 57P03) and at the `NativeAuthClient` layer (for HTTP 5xx and `HttpRequestException`). Three retries with exponential backoff (100ms, 200ms, 400ms) is usually sufficient for cloud database transient failures.

### Risk 2: Contract Fragmentation (High)

The `VerseResponse` concept exists in four places:

| Location | Type | Naming |
|----------|------|--------|
| `Core/.../SingleVerseResponse.cs` | `class` | `TranslationInfo` (nested), `Text`, `Copyright` |
| `Client/Models/.../SingleVerseResponse.cs` | `record` | `TranslationInfo` (separate record), `Text`, `Copyright` |
| `sdk/dotnet/.../VerseInfo.cs` | `class` | `TranslationDetails` (different name!), `Text`, `Copyright` |
| `lectio-memorize/.../types/index.ts` | TS interface | `VerseInfo` with `id: number` (different shape!) |

The SDK names the translation sub-object `TranslationDetails` while the Core and Client call it `TranslationInfo`. The TypeScript types use `id: number` while the .NET types use `Code: string` for version identification. These are silent deserialization failures waiting to happen — they won't produce compile errors, only runtime `null` fields or missing data.

### Risk 3: Proxy Layer Socket Exhaustion (High)

`lectio-memorize/functions/GetVerse.cs:23`:
```csharp
var client = new LectioDivinaClient(baseUrl, apiKey);
```
The `LectioDivinaClient` creates `new HttpClient { BaseAddress = ... }` internally. In Azure Functions:
- Each invocation creates a new `HttpClient`
- Each `HttpClient` creates a new socket
- The socket enters `TIME_WAIT` for 30-120 seconds after disposal
- Under concurrent load (even 10 req/s), ephemeral ports are exhausted within minutes

The main API's `Program.cs` uses the correct pattern (`services.AddHttpClient("EntraAuth")`) — but the proxy layer's `Program.cs` has an empty `ConfigureServices` block. The fix is to register a typed client there.

---

## 3. High-Priority Backlog

| # | Task | Category | Impact | Complexity | Justification |
|---|------|----------|--------|------------|---------------|
| 1 | Add Polly retry policies to all repository methods | Resilience | High | Medium | Zero retry on any DB call across the entire codebase |
| 2 | Consolidate API contracts to single source of truth | Architecture | High | High | 4 copies of same types; field rename silently breaks 3 clients |
| 3 | Add `IHttpClientFactory` to lectio-memorize functions | Resilience | High | Low | `new HttpClient()` per invocation exhausts ports under load |
| 4 | Add retry to NativeAuthClient outbound HTTP calls | Resilience | Medium | Medium | 3-step auth flow fails entirely on any single network blip |
| 5 | Add correlation ID propagation end-to-end | Observability | Medium | Medium | Can't trace a user error to a server log entry today |
| 6 | Replace `BuildServiceProvider()` anti-pattern | DI Hygiene | Medium | Low | Second DI container is a documented anti-pattern |
