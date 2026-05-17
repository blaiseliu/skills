# Ruthless Review: API Tier (LectioDivina.Api + Services + Data)

## 1. Executive Summary (CEO Lens)

The API tier's biggest risk in production is the complete absence of transient fault handling from the repository layer outward. Every database call, every outbound HTTP call through `NativeAuthClient`, and every health check hits the wire with no retry, no circuit breaker, no timeout configuration. In Azure Functions' consumption plan where cold starts are normal and connection pools are ephemeral, a single PostgreSQL connection timeout or a 500ms network hiccup to the identity provider will produce a user-facing error — not a retry that succeeds 200ms later. This isn't speculative. This is table stakes for any serverless API that touches a database.

The hidden cost is in the DI configuration. `Program.cs` calls `BuildServiceProvider()` inside `ConfigureServices`, which creates a second dependency injection container — a documented .NET anti-pattern that produces duplicate singleton instances and will generate analyzer warnings in .NET 8+. The `ReferenceCache` takes `IServiceProvider` as a dependency (service locator pattern) to work around the fact that it's a Singleton consuming Scoped services. Both of these are solvable with standard patterns, but left as-is they cause subtle bugs — like the two `ILogger<DatabaseConfiguration>` instances that currently exist, one of which is never disposed.

The API tier is architecturally clean — middleware pipeline is well-ordered, functions are consistently structured, and the v1/v2 API versioning is handled correctly. But the operational gaps (resilience, observability, DI hygiene) make it fragile under load. A well-timed PostgreSQL failover or a surge of cold starts will expose every gap simultaneously.

---

## 2. Architectural Breakdown (Tech Lead Lens)

### Composition Root Analysis

**Finding: `BuildServiceProvider()` anti-pattern (CRITICAL)**

`src/LectioDivina.Api/Program.cs:44-46`:
```csharp
services.AddSingleton(new DatabaseConfiguration(
    connectionString,
    services.BuildServiceProvider().GetRequiredService<ILogger<DatabaseConfiguration>>()
));
```
This builds a second DI container solely to resolve an `ILogger<T>`. The consequences:
- Two `ILogger<DatabaseConfiguration>` instances exist — one from the temp container, one from the real one
- The temp container is never disposed
- Validation errors in modern .NET (the host warns about this pattern)

The factory overload is a one-line fix:
```csharp
services.AddSingleton(sp => new DatabaseConfiguration(
    connectionString, sp.GetRequiredService<ILogger<DatabaseConfiguration>>()));
```

**Finding: Service locator in ReferenceCache (MEDIUM)**

`src/LectioDivina.Api/Caching/ReferenceCache.cs:11`:
```csharp
public ReferenceCache(IServiceProvider serviceProvider, ...)
```
Then in `InitializeAsync()`:
```csharp
using var scope = _serviceProvider.CreateScope();
var bookRepo = scope.ServiceProvider.GetRequiredService<IBookRepository>();
```
`ReferenceCache` is registered as Singleton but needs Scoped repositories. The service locator pattern (`IServiceProvider` injection) works but hides the dependency graph — you can't tell from the constructor what this class actually needs. The standard solution is to register `ReferenceCache` as a hosted service (`IHostedService`) or to accept factory delegates (`Func<IBookRepository>`) instead of the container itself.

**Finding: Empty catch in UpdateLastUsedFireAndForget (LOW)**

`src/LectioDivina.Api/Middleware/ApiKeyMiddleware.cs:81`:
```csharp
private async Task UpdateLastUsedFireAndForget(Guid keyId)
{
    try { await _apiKeyRepo.UpdateLastUsedAsync(keyId); } catch { /* best effort */ }
}
```
This is a legitimate fire-and-forget pattern (the method is called without `await` at call sites), but the empty catch will swallow transient database errors silently. If the database is intermittently unavailable, last-used timestamps will silently fall out of date with no alerting. At minimum, log a warning-level message.

### Repository Layer

**Finding: Log-and-rethrow is noise, not resilience (HIGH)**

`src/LectioDivina.Data/.../VerseRepository.cs:42-47` (repeated in every method):
```csharp
catch (Exception ex)
{
    _logger.LogError(ex, "Error getting verse {VersionCode} {BookId} {Chapter}:{Verse}", ...);
    throw;
}
```
Three problems:
1. Zero retry logic. PostgreSQL transient errors (deadlocks, connection timeouts) are just rethrown as 500s.
2. The same exception is logged here AND in the function layer (`SingleVerseFunction.cs:67` catches and logs again). Two log entries for one failure.
3. The exception is logged before any retry is attempted — so if you add retry later, you'll get error logs for requests that eventually succeed.

The correct layering: repository catches Npgsql transient exceptions (08XXX class, 40001 deadlock, 57P03 cannot connect), retries with exponential backoff, and only logs if all retries fail. Permanent errors (23505 unique violation, etc.) should not be retried and should not be swallowed.

**Finding: Connection opened in factory method, not in using block (MEDIUM)**

`src/LectioDivina.Data/.../DatabaseConfiguration.cs:22`:
```csharp
public virtual NpgsqlConnection CreateConnection()
{
    var connection = new NpgsqlConnection(_connectionString);
    connection.Open();
    return connection;
}
```
The connection is opened before returning from the factory. The caller then does:
```csharp
using var connection = _dbConfig.CreateConnection();
// ... Dapper query ...
```
This means the connection is opened in `CreateConnection()` and disposed by `using`. It works, but the `Open()` call itself has no retry or timeout — if the database is momentarily unavailable, the exception surfaces to the caller immediately. Dapper can open connections lazily; you can let it handle the open inside the query execution where retry logic can wrap the entire operation.

### Request Pipeline

**Finding: Two different error response formats (MEDIUM)**

The v1 functions use `ResponseEnvelopeHelper.WrapError()` which returns:
```json
{ "data": null, "meta": { "request_id": "..." }, "error": { "code": "...", "message": "..." } }
```
The v2 functions (`SingleVerseFunction.cs`) return raw strings:
```
"Invalid request body"
```
The Blazor client's `IApiService` (Refit) expects deserializable response types. A Refit client hitting a v2 endpoint that returns a raw string on error will get a deserialization exception instead of the actual error message. The `ApiErrorHandler` catches this and shows the raw HTTP status, but the structured error detail is lost.

### DI Scope Analysis

**Finding: Singleton holding Scoped dependency — handled correctly (GOOD)**

`ReferenceCache` is Singleton but needs `IBookRepository` (Scoped). The code handles this correctly by creating a scope at initialization time:
```csharp
using var scope = _serviceProvider.CreateScope();
```
This is the correct pattern. The service locator approach is questionable but the scope management is correct.

**Finding: HttpClient registration is correct (GOOD)**

`Program.cs` registers `services.AddHttpClient("EntraAuth")` — using the `IHttpClientFactory` pattern which handles socket lifetime. This is the correct pattern for serverless. The `NativeAuthClient` uses this named client. Contrast with the `lectio-memorize/functions/` which uses `new HttpClient()` directly.

### Middleware Pipeline

The middleware ordering is correct: `ApiKeyMiddleware` → `JwtValidationMiddleware` → `RateLimitingMiddleware` → `CorsMiddleware`. The `ApiKeyMiddleware` correctly scopes its enforcement to v1 data routes only, passing through JWT-authenticated routes. The `RateLimitingMiddleware` correctly partitions by user `sub` claim with an "anonymous" fallback. Both are well-structured.

One concern: the `RateLimitingMiddleware` creates a `PartitionedRateLimiter` in its constructor and never disposes it. `PartitionedRateLimiter` implements `IDisposable` — in a long-running process this is a resource leak, though in Azure Functions' ephemeral execution model the impact is minimal.

---

## 3. High-Priority Backlog

| # | Task | Category | Impact | Complexity | Justification |
|---|------|----------|--------|------------|---------------|
| 1 | Add retry/circuit-breaker to repository layer (Npgsql transient errors) | Resilience | High | Medium | No retry on any DB call; a 200ms blip becomes a user-facing 500 |
| 2 | Replace `BuildServiceProvider()` with factory lambda in Program.cs | DI Hygiene | High | Low | Second DI container is a documented anti-pattern; 2-line fix |
| 3 | Unify error response format across v1 and v2 endpoints | API Design | Medium | Medium | v2 returns raw strings, v1 returns structured envelopes; Refit clients break on v2 errors |
| 4 | Add retry/timeout to `NativeAuthClient` outbound calls | Resilience | Medium | Medium | Auth flow has 3 sequential HTTP calls with no retry on any of them |
| 5 | Log warning in `UpdateLastUsedFireAndForget` empty catch | Observability | Low | Low | Silently dropping DB errors means last-used timestamps can be stale with no alert |
| 6 | Dispose `PartitionedRateLimiter` when middleware is disposed | Resource | Low | Low | Rate limiter implements IDisposable but is never disposed |
