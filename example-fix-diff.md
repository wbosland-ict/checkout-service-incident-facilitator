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
`problem-evidence/checkout-service-config.yaml` describes it: 3 attempts,
**fixed** 200 ms delay, **no jitter**, retrying on 429 as readily as on a
5xx, no circuit breaker, and no idempotency key. That is the retry policy
that turned a DB slowdown into a payment-gateway rate-limit breach and 7
duplicate charges, so this is a change to existing behaviour, not new
code on a blank stub.

The fix keeps the same shape and attempt budget, but: backs off
exponentially with jitter, only retries a 429 when the gateway actually
supplied a `Retry-After` (and then waits exactly that long), trips a
circuit breaker after repeated failures, and threads an
`Idempotency-Key` through to the charge request.

```diff
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
     }
 }

@@
+public class PaymentGatewayCircuitOpenException : Exception
+{
+    public PaymentGatewayCircuitOpenException(string message)
+        : base(message)
+    {
+    }
+}
+
 public interface IPaymentClient
 {
-    Task<PaymentResult> ChargeAsync(ReservationResult reservation, IReadOnlyList<LineItem> lines);
+    Task<PaymentResult> ChargeAsync(ReservationResult reservation, IReadOnlyList<LineItem> lines, string idempotencyKey);
 }

 public class PaymentClient : IPaymentClient
 {
     private const int MaxAttempts = 3;
-    private const int BackoffDelayMs = 200;
+    private const int CircuitBreakerFailureThreshold = 5;
+    private static readonly TimeSpan BaseDelay = TimeSpan.FromMilliseconds(200);
+    private static readonly TimeSpan CircuitOpenDuration = TimeSpan.FromSeconds(30);

-    public async Task<PaymentResult> ChargeAsync(ReservationResult reservation, IReadOnlyList<LineItem> lines)
+    private int _consecutiveFailures;
+    private DateTimeOffset _circuitOpenUntil = DateTimeOffset.MinValue;
+
+    public async Task<PaymentResult> ChargeAsync(ReservationResult reservation, IReadOnlyList<LineItem> lines, string idempotencyKey)
     {
+        if (DateTimeOffset.UtcNow < _circuitOpenUntil)
+        {
+            throw new PaymentGatewayCircuitOpenException(
+                "Payment gateway circuit breaker is open; not sending further charge requests yet.");
+        }
+
         for (var attempt = 1; attempt <= MaxAttempts; attempt++)
         {
             try
             {
-                return await SendChargeRequestAsync(reservation, lines);
+                var result = await SendChargeRequestAsync(reservation, lines, idempotencyKey);
+                _consecutiveFailures = 0;
+                return result;
             }
             catch (PaymentGatewayTimeoutException) when (attempt < MaxAttempts)
             {
-                await Task.Delay(BackoffDelayMs);
+                RecordFailure();
+                await Task.Delay(ComputeBackoffWithJitter(attempt));
             }
-            catch (PaymentGatewayException ex) when (attempt < MaxAttempts && IsRetryable(ex.StatusCode))
+            catch (PaymentGatewayException ex) when (attempt < MaxAttempts && IsRetryable(ex))
             {
-                await Task.Delay(BackoffDelayMs);
+                RecordFailure();
+                await Task.Delay(ex.RetryAfter ?? ComputeBackoffWithJitter(attempt));
             }
         }

+        RecordFailure();
         throw new PaymentGatewayException(502, $"Payment gateway did not succeed after {MaxAttempts} attempts.");
     }

-    private static bool IsRetryable(int statusCode)
+    private static bool IsRetryable(PaymentGatewayException ex)
+    {
+        if (ex.StatusCode == 429)
+        {
+            return ex.RetryAfter is not null;
+        }
+
+        return ex.StatusCode >= 500;
+    }
+
+    private void RecordFailure()
+    {
+        _consecutiveFailures++;
+        if (_consecutiveFailures >= CircuitBreakerFailureThreshold)
+        {
+            _circuitOpenUntil = DateTimeOffset.UtcNow.Add(CircuitOpenDuration);
+        }
+    }
+
+    private static TimeSpan ComputeBackoffWithJitter(int attempt)
     {
-        return statusCode == 429 || statusCode >= 500;
+        var exponential = BaseDelay * Math.Pow(2, attempt - 1);
+        var jitter = Random.Shared.NextDouble() * exponential.TotalMilliseconds * 0.2;
+        return exponential + TimeSpan.FromMilliseconds(jitter);
     }

-    private async Task<PaymentResult> SendChargeRequestAsync(ReservationResult reservation, IReadOnlyList<LineItem> lines)
+    private async Task<PaymentResult> SendChargeRequestAsync(ReservationResult reservation, IReadOnlyList<LineItem> lines, string idempotencyKey)
     {
         await Task.Delay(1);
         return new PaymentResult(Guid.NewGuid().ToString("N"));
     }
 }
```

`SendChargeRequestAsync` is still a stub that never throws, so none of
this retry logic actually fires in the sample repo — the point is the
policy, not the transport. A participant who notices that and says "this
is untested behaviour, I'd want a fake gateway that returns 429 with and
without `Retry-After`" is making exactly the right observation; the
exercise's "watch out for" list calls out tests that look safe but prove
nothing.

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
  one per checkout attempt and passes it down. Generating it inside
  `PaymentClient` would be wrong (a retry at a higher level would produce
  a fresh key and re-enable double charges); that's worth catching if a
  group does it.
