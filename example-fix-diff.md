# Model Answer — Exercise 4 fix diff (FACILITATOR MODEL ANSWER)

This is a reference implementation of RFC `W 2607 012` (options **A + B +
C** from `example-problem-record.md`) applied to
`checkout-service-incident-sourcecode/`. Use it to sanity-check a
participant's AI-assisted diff — not to hand out before the exercise.

It was produced and verified the same way a participant is asked to work:
implement the fix, then confirm the test suite goes from **1 passed** (the
buggy baseline) to **4 passed** (smoke test + 3 regression cases across
cart sizes 4/8/11), with the new regression test failing against the
original buggy code first.

## A — Restore efficient fetching (`Cart/CartRepository.cs`)

Replaces `FindAsync` (no related data) with a single eager-loaded query
covering the cart, its items, and each item's product.

```diff
+using Microsoft.EntityFrameworkCore;
+
 namespace ShopFast.Checkout.Cart;

 public class CartRepository : ICartRepository
@@
     public async Task<Cart?> GetByIdAsync(long cartId)
     {
-        return await _dbContext.Carts.FindAsync(cartId);
+        return await _dbContext.Carts
+            .Include(c => c.Items)
+            .ThenInclude(i => i.Product)
+            .FirstOrDefaultAsync(c => c.Id == cartId);
     }
 }
```

A split query (`.AsSplitQuery()`) was also considered; for this shape (one
collection + one reference nav) a single joined query is simpler and
issues one SQL statement. `CartItem.Product` can stay `virtual` — eager
loading marks it as loaded, so lazy loading never fires for carts fetched
through this method.

## A — Narrow the transaction scope (`CheckoutService.cs`)

The transaction no longer wraps the cart read or the inventory/payment
remote calls — only the final order write, which is the only step that
needs DB atomicity. This is also where the `Idempotency-Key` for the
payment call is generated, one per checkout attempt.

```diff
     public async Task<CheckoutResult> StartAsync(long cartId)
     {
-        using var transaction = await _dbContext.Database.BeginTransactionAsync();
-
         var cart = await _cartRepository.GetByIdAsync(cartId)
             ?? throw new CartNotFoundException();

         var lines = cart.Items
             .Select(i => new LineItem(i.Product.Sku, i.Product.Price, i.Quantity))
             .ToList();

+        var idempotencyKey = $"checkout-{cartId}-{Guid.NewGuid():N}";
+
         var reservation = await _inventoryClient.ReserveAsync(lines);
-        var payment = await _paymentClient.ChargeAsync(reservation, lines);
-        var result = await _orderWriter.CreateAsync(cartId, reservation, payment);
+        var payment = await _paymentClient.ChargeAsync(reservation, lines, idempotencyKey);

+        using var transaction = await _dbContext.Database.BeginTransactionAsync();
+        var result = await _orderWriter.CreateAsync(cartId, reservation, payment);
         await transaction.CommitAsync();
+
         return result;
     }
 }
```

## B — Harden the payment client (`Payment/PaymentClient.cs`)

The starter code already retries, but exactly as
`problem-evidence/checkout-service-config.yaml` describes it (bound via
`PaymentGatewayOptions`): 3 attempts, a **fixed** 200 ms delay, **no
jitter**, retrying on 429 as readily as on a 5xx, no circuit breaker, and
no `Idempotency-Key` on `POST /v1/charge`. That is the retry policy that
turned a DB slowdown into a payment-gateway rate-limit breach and 7
duplicate charges, so this is a change to existing behaviour, not new
code on a blank stub.

The fix keeps the same shape and attempt budget, but: backs off
exponentially with jitter, only retries a 429 when the gateway supplied a
short `Retry-After` (and then waits exactly that long), trips a circuit
breaker after repeated failures, and sends an `Idempotency-Key` header on
every charge attempt.

`PaymentClient` is registered as a typed `HttpClient`
(`AddHttpClient<IPaymentClient, PaymentClient>()`), which makes it
**transient**: breaker state kept in instance fields would reset on every
checkout and never trip. The breaker therefore lives in its own singleton,
`PaymentGatewayCircuitBreaker` (new file, shown after the diff).

```diff
--- a/src/ShopFast.Checkout/DependencyInjection/CheckoutServiceCollectionExtensions.cs
+++ b/src/ShopFast.Checkout/DependencyInjection/CheckoutServiceCollectionExtensions.cs
@@ -18,6 +18,7 @@ public static class CheckoutServiceCollectionExtensions
         services.Configure<PaymentGatewayOptions>(configuration.GetSection(PaymentGatewayOptions.SectionName));
 
         services.TryAddSingleton(TimeProvider.System);
+        services.AddSingleton<PaymentGatewayCircuitBreaker>();
 
         services.AddHttpClient<IInventoryClient, InventoryClient>();
         services.AddHttpClient<IPaymentClient, PaymentClient>();
--- a/src/ShopFast.Checkout/Payment/PaymentClient.cs
+++ b/src/ShopFast.Checkout/Payment/PaymentClient.cs
@@ -14,11 +14,21 @@ public record PaymentResult(string ChargeId, decimal Amount, string Currency);
 public class PaymentGatewayException : Exception
 {
     public int StatusCode { get; }
+    public TimeSpan? RetryAfter { get; }
 
-    public PaymentGatewayException(int statusCode, string message)
+    public PaymentGatewayException(int statusCode, string message, TimeSpan? retryAfter = null)
         : base(message)
     {
         StatusCode = statusCode;
+        RetryAfter = retryAfter;
+    }
+}
+
+public class PaymentGatewayCircuitOpenException : Exception
+{
+    public PaymentGatewayCircuitOpenException(string message)
+        : base(message)
+    {
     }
 }
 
@@ -43,22 +53,27 @@ public class PaymentDeclinedException : Exception
 
 public interface IPaymentClient
 {
-    Task<PaymentResult> ChargeAsync(ReservationResult reservation, IReadOnlyList<LineItem> lines);
+    Task<PaymentResult> ChargeAsync(ReservationResult reservation, IReadOnlyList<LineItem> lines, string idempotencyKey);
 }
 
 public class PaymentClient : IPaymentClient
 {
+    private static readonly TimeSpan MaxHonouredRetryAfter = TimeSpan.FromSeconds(2);
+
     private readonly HttpClient _httpClient;
     private readonly PaymentGatewayOptions _options;
+    private readonly PaymentGatewayCircuitBreaker _circuitBreaker;
     private readonly ILogger<PaymentClient> _logger;
 
     public PaymentClient(
         HttpClient httpClient,
         IOptions<PaymentGatewayOptions> options,
+        PaymentGatewayCircuitBreaker circuitBreaker,
         ILogger<PaymentClient> logger)
     {
         _httpClient = httpClient;
         _options = options.Value;
+        _circuitBreaker = circuitBreaker;
         _logger = logger;
 
         _httpClient.BaseAddress = _options.BaseUrl.WithTrailingSlash();
@@ -66,43 +81,76 @@ public class PaymentClient : IPaymentClient
         _httpClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", _options.ApiKey);
     }
 
-    public async Task<PaymentResult> ChargeAsync(ReservationResult reservation, IReadOnlyList<LineItem> lines)
+    public async Task<PaymentResult> ChargeAsync(
+        ReservationResult reservation, IReadOnlyList<LineItem> lines, string idempotencyKey)
     {
+        if (_circuitBreaker.IsOpen)
+        {
+            throw new PaymentGatewayCircuitOpenException(
+                "Payment gateway circuit breaker is open; not sending further charge requests yet.");
+        }
+
         var maxAttempts = _options.Retry.MaxAttempts;
 
         for (var attempt = 1; attempt <= maxAttempts; attempt++)
         {
             try
             {
-                return await SendChargeRequestAsync(reservation, lines, attempt);
+                var result = await SendChargeRequestAsync(reservation, lines, idempotencyKey, attempt);
+                _circuitBreaker.RecordSuccess();
+                return result;
             }
-            catch (PaymentGatewayTimeoutException) when (attempt < maxAttempts)
+            catch (PaymentGatewayTimeoutException)
             {
-                await DelayBeforeRetryAsync(attempt + 1);
+                _circuitBreaker.RecordFailure();
+                if (attempt == maxAttempts)
+                {
+                    throw;
+                }
+
+                await DelayBeforeRetryAsync(attempt + 1, ComputeBackoffWithJitter(attempt));
             }
-            catch (PaymentGatewayException ex) when (attempt < maxAttempts && IsRetryable(ex.StatusCode))
+            catch (PaymentGatewayException ex) when (ex.StatusCode == 429 || ex.StatusCode >= 500)
             {
-                await DelayBeforeRetryAsync(attempt + 1);
+                _circuitBreaker.RecordFailure();
+                if (attempt == maxAttempts || !IsRetryable(ex))
+                {
+                    throw;
+                }
+
+                await DelayBeforeRetryAsync(attempt + 1, ex.RetryAfter ?? ComputeBackoffWithJitter(attempt));
             }
         }
 
         throw new PaymentGatewayException(502, $"Payment gateway did not succeed after {maxAttempts} attempts.");
     }
 
-    private static bool IsRetryable(int statusCode)
+    private static bool IsRetryable(PaymentGatewayException ex)
+    {
+        if (ex.StatusCode == 429)
+        {
+            return ex.RetryAfter is { } retryAfter && retryAfter <= MaxHonouredRetryAfter;
+        }
+
+        return ex.StatusCode >= 500;
+    }
+
+    private TimeSpan ComputeBackoffWithJitter(int attempt)
     {
-        return statusCode == 429 || statusCode >= 500;
+        var exponentialMs = _options.Retry.BackoffDelayMs * Math.Pow(2, attempt - 1);
+        var jitterMs = Random.Shared.NextDouble() * exponentialMs * 0.2;
+        return TimeSpan.FromMilliseconds(exponentialMs + jitterMs);
     }
 
-    private async Task DelayBeforeRetryAsync(int nextAttempt)
+    private async Task DelayBeforeRetryAsync(int nextAttempt, TimeSpan delay)
     {
-        var delayMs = _options.Retry.BackoffDelayMs;
-        _logger.LogInformation("retry.scheduled attempt={Attempt} delay_ms={DelayMs}", nextAttempt, delayMs);
-        await Task.Delay(delayMs);
+        _logger.LogInformation(
+            "retry.scheduled attempt={Attempt} delay_ms={DelayMs}", nextAttempt, (long)delay.TotalMilliseconds);
+        await Task.Delay(delay);
     }
 
     private async Task<PaymentResult> SendChargeRequestAsync(
-        ReservationResult reservation, IReadOnlyList<LineItem> lines, int attempt)
+        ReservationResult reservation, IReadOnlyList<LineItem> lines, string idempotencyKey, int attempt)
     {
         var request = new ChargeRequest(
             AmountMinor: lines.Sum(l => Money.ToMinorUnits(l.Price) * l.Quantity),
@@ -119,8 +167,13 @@ public class PaymentClient : IPaymentClient
 
         try
         {
-            using var response = await _httpClient.PostAsJsonAsync(
-                "charge", request, DownstreamJson.Options, timeoutCts.Token);
+            using var message = new HttpRequestMessage(HttpMethod.Post, "charge")
+            {
+                Content = JsonContent.Create(request, options: DownstreamJson.Options),
+            };
+            message.Headers.Add("Idempotency-Key", idempotencyKey);
+
+            using var response = await _httpClient.SendAsync(message, timeoutCts.Token);
 
             var status = (int)response.StatusCode;
 
@@ -151,7 +204,9 @@ public class PaymentClient : IPaymentClient
             }
 
             throw new PaymentGatewayException(
-                status, error?.Message ?? $"Payment gateway returned HTTP {status} ({response.ReasonPhrase}).");
+                status,
+                error?.Message ?? $"Payment gateway returned HTTP {status} ({response.ReasonPhrase}).",
+                response.Headers.RetryAfter?.Delta);
         }
         catch (OperationCanceledException) when (timeoutCts.IsCancellationRequested)
         {
--- a/tests/ShopFast.Checkout.Tests/Fixtures/DownstreamFakes.cs
+++ b/tests/ShopFast.Checkout.Tests/Fixtures/DownstreamFakes.cs
@@ -49,11 +49,14 @@ public static class DownstreamFakes
     }
 
     public static PaymentClient CreatePaymentClient(
-        HttpMessageHandler handler, PaymentGatewayOptions? options = null)
+        HttpMessageHandler handler,
+        PaymentGatewayOptions? options = null,
+        PaymentGatewayCircuitBreaker? circuitBreaker = null)
     {
         return new PaymentClient(
             new HttpClient(handler),
             Options.Create(options ?? new PaymentGatewayOptions { ApiKey = "test-api-key" }),
+            circuitBreaker ?? new PaymentGatewayCircuitBreaker(TimeProvider.System),
             NullLogger<PaymentClient>.Instance);
     }
 
```

New file `Payment/PaymentGatewayCircuitBreaker.cs`:

```csharp
namespace ShopFast.Checkout.Payment;

public class PaymentGatewayCircuitBreaker
{
    private const int FailureThreshold = 5;
    private static readonly TimeSpan OpenDuration = TimeSpan.FromSeconds(30);

    private readonly TimeProvider _timeProvider;
    private readonly object _lock = new();
    private int _consecutiveFailures;
    private DateTimeOffset _openUntil = DateTimeOffset.MinValue;

    public PaymentGatewayCircuitBreaker(TimeProvider timeProvider)
    {
        _timeProvider = timeProvider;
    }

    public bool IsOpen
    {
        get
        {
            lock (_lock)
            {
                return _timeProvider.GetUtcNow() < _openUntil;
            }
        }
    }

    public void RecordSuccess()
    {
        lock (_lock)
        {
            _consecutiveFailures = 0;
        }
    }

    public void RecordFailure()
    {
        lock (_lock)
        {
            _consecutiveFailures++;
            if (_consecutiveFailures >= FailureThreshold)
            {
                _openUntil = _timeProvider.GetUtcNow().Add(OpenDuration);
                _consecutiveFailures = 0;
            }
        }
    }
}
```

The tests talk to a fake payment-gateway (`FakeHttpMessageHandler` in
`tests/ShopFast.Checkout.Tests/Fixtures/`), so this retry logic *can* be
exercised: a 429 without `Retry-After` should produce exactly one request,
a 429 with `Retry-After: 1` a second request carrying the **same**
`Idempotency-Key`, and five consecutive failures should open the breaker.
The model answer was checked against those three cases. A participant who
writes such tests, or who points out that the hardening is otherwise
untested, is making exactly the right observation; the exercise's "watch
out for" list calls out tests that look safe but prove nothing.

## C — Query-count regression test (new file:
`tests/ShopFast.Checkout.Tests/QueryCountRegressionTests.cs`)

```csharp
using ShopFast.Checkout.Cart;
using ShopFast.Checkout.Tests.Fixtures;

namespace ShopFast.Checkout.Tests;

public class QueryCountRegressionTests
{
    [Theory]
    [InlineData(4)]
    [InlineData(8)]
    [InlineData(11)]
    public async Task GetByIdAsync_DoesNotIssueOneQueryPerCartItem(int itemCount)
    {
        var (setupContext, connection) = TestDbContextFactory.CreateContext();
        await using var disposableConnection = connection;
        await using var disposableSetupContext = setupContext;

        var cart = CartFixtures.CreateCartWithItems(setupContext, itemCount);

        var interceptor = new QueryCountingInterceptor();
        await using var dbContext = TestDbContextFactory.CreateContextOnConnection(connection, interceptor);

        var repository = new CartRepository(dbContext);
        var loaded = await repository.GetByIdAsync(cart.Id)
            ?? throw new InvalidOperationException("Cart not found.");

        var skus = loaded.Items.Select(i => i.Product.Sku).ToList();
        Assert.Equal(itemCount, skus.Count);

        Assert.True(
            interceptor.QueryCount <= 2,
            $"Expected at most 2 queries regardless of cart size, but saw {interceptor.QueryCount} " +
            $"for a cart with {itemCount} items - this indicates an N+1 query pattern.");
    }
}
```

**Why it measures on a second `DbContext` over the same connection:** the
starter repo's `TestDbContextFactory.CreateContext()` already keeps schema
creation off the interceptor, but fixture setup (`CartFixtures`, via
`SaveChanges`) also issues reader commands on Sqlite (generated keys use
`RETURNING`, which goes through `ExecuteReader`). Building the cart on one
context and then attaching the interceptor only to a second, fresh context
over the same connection isolates the count to exactly what
`GetByIdAsync` issues. A participant who reuses one interceptor-attached
context for both setup and the call under test will see inflated,
cart-size-dependent counts on *both* the buggy and the fixed code — worth
flagging if it comes up, since "a weak/misleading regression test is worse
than none" is explicitly called out in the exercise's "Watch out for"
section.

## Verifying this model answer

Applied against `checkout-service-incident-sourcecode/`:

```powershell
dotnet test
```

- Before the fix: `QueryCountRegressionTests` fails for all three cart
  sizes (buggy code shows unbounded growth, e.g. 6 queries for 4 items, 10
  for 8 items — still the N+1 shape even though the absolute numbers
  differ from a first impression, because fixture-setup commands are
  excluded here unlike in earlier drafts of this test).
- After the fix: **4 passed** (1 smoke test + 3 regression cases), 0
  failed, 0 skipped.

## What a participant's AI-assisted diff might reasonably differ on

- `.Include()`/`.ThenInclude()` vs. a split query vs. a DTO projection —
  all three are legitimate; ask them to explain the trade-off (joined
  query = 1 round trip but a wider result set with duplicate parent
  columns; split query = N round trips but no column duplication;
  projection = smallest payload but no tracked entities).
- Where exactly the transaction boundary should sit — some valid answers
  narrow it to just the `OrderWriter` call (as here), others might argue
  for no transaction at all if `OrderWriter` becomes a single atomic
  write. Either is defensible; no transaction around the remote calls is
  the non-negotiable part.
- How strictly the payment client refuses to retry on 429 without
  `Retry-After` — anywhere from "never retry a 429 without it" to "retry
  with backoff anyway, just honour it when present" is reasonable; it
  should be a deliberate choice they can explain, not an accident.
- Exact circuit-breaker thresholds/durations and backoff constants — there
  is no single correct number here; the RFC doesn't specify a precise
  value, so look for *a* bounded retry count, backoff that grows, and a
  breaker that stops hammering a failing gateway, not an exact match to
  this file's constants.
- Whether they keep `MaxAttempts` at 3 — the config's attempt count was
  never the problem; the fixed delay, the blind 429 retry and the missing
  idempotency key were. A diff that drops to 1 attempt "to be safe" is
  over-correcting and should be challenged.
- Where the `Idempotency-Key` is generated — here `CheckoutService` makes
  one per checkout attempt and passes it down, so every retry inside
  `PaymentClient` reuses it. That is what stops the duplicate charges
  from this incident. Generating a fresh key per attempt inside the
  retry loop would be wrong and is worth catching. Note the limit: a
  retry by the web/app front-end (on 502/504) starts a new checkout
  attempt and gets a new key, so it is not covered. Closing that gap
  needs a key supplied by the front-end (one per "Pay" click) and passed
  through — a good follow-up to discuss, but out of scope for this fix.
