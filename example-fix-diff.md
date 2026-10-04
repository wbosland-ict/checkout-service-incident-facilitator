# Model Answer — Exercise 4 fix diff (FACILITATOR MODEL ANSWER)

This is a reference implementation of RFC `W 2607 012` (options **A + B +
C** from `example-problem-record.md`) applied to
`checkout-service-incident-sourcecode/`. Use it to sanity-check a
participant's AI-assisted diff — not to hand out before the exercise.

It was produced and verified the same way a participant is asked to work:
implement the fix, then confirm the test suite goes from **1 passed** (the
buggy baseline) to **5 passed** (smoke test + 3 regression cases across
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

Adds exponential backoff with jitter, honours a gateway-supplied
`Retry-After` on 429s instead of blind fixed-delay retries, trips a
circuit breaker after repeated failures, and threads an idempotency key
through to the (stubbed) charge request.

```diff
 public record PaymentResult(string ChargeId);

+public class PaymentRateLimitedException : Exception
+{
+    public TimeSpan? RetryAfter { get; }
+
+    public PaymentRateLimitedException(TimeSpan? retryAfter)
+        : base("Payment gateway rate limit exceeded (429).")
+    {
+        RetryAfter = retryAfter;
+    }
+}
+
+public class PaymentGatewayTransientException : Exception
+{
+    public PaymentGatewayTransientException(string message) : base(message)
+    {
+    }
+}
+
+public class PaymentGatewayUnavailableException : Exception
+{
+    public PaymentGatewayUnavailableException(string message) : base(message)
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
-    public async Task<PaymentResult> ChargeAsync(ReservationResult reservation, IReadOnlyList<LineItem> lines)
+    private const int MaxAttempts = 4;
+    private const int CircuitBreakerFailureThreshold = 5;
+    private static readonly TimeSpan BaseDelay = TimeSpan.FromMilliseconds(200);
+    private static readonly TimeSpan CircuitBreakerOpenDuration = TimeSpan.FromSeconds(30);
+
+    private int _consecutiveFailures;
+    private DateTimeOffset _circuitOpenUntil = DateTimeOffset.MinValue;
+
+    public async Task<PaymentResult> ChargeAsync(ReservationResult reservation, IReadOnlyList<LineItem> lines, string idempotencyKey)
+    {
+        if (DateTimeOffset.UtcNow < _circuitOpenUntil)
+        {
+            throw new PaymentGatewayUnavailableException(
+                "Payment gateway circuit breaker is open; refusing to call out until it resets.");
+        }
+
+        for (var attempt = 1; attempt <= MaxAttempts; attempt++)
+        {
+            try
+            {
+                var result = await SendChargeRequestAsync(reservation, lines, idempotencyKey);
+                _consecutiveFailures = 0;
+                return result;
+            }
+            catch (PaymentRateLimitedException ex) when (attempt < MaxAttempts)
+            {
+                RecordFailure();
+                await Task.Delay(ex.RetryAfter ?? ComputeBackoffWithJitter(attempt));
+            }
+            catch (PaymentGatewayTransientException) when (attempt < MaxAttempts)
+            {
+                RecordFailure();
+                await Task.Delay(ComputeBackoffWithJitter(attempt));
+            }
+        }
+
+        RecordFailure();
+        throw new PaymentGatewayUnavailableException(
+            $"Payment gateway did not succeed after {MaxAttempts} attempts.");
+    }
+
+    private void RecordFailure()
+    {
+        _consecutiveFailures++;
+        if (_consecutiveFailures >= CircuitBreakerFailureThreshold)
+        {
+            _circuitOpenUntil = DateTimeOffset.UtcNow.Add(CircuitBreakerOpenDuration);
+        }
+    }
+
+    private static TimeSpan ComputeBackoffWithJitter(int attempt)
+    {
+        var exponential = BaseDelay * Math.Pow(2, attempt - 1);
+        var jitter = Random.Shared.NextDouble() * exponential.TotalMilliseconds * 0.2;
+        return exponential + TimeSpan.FromMilliseconds(jitter);
+    }
+
+    private async Task<PaymentResult> SendChargeRequestAsync(ReservationResult reservation, IReadOnlyList<LineItem> lines, string idempotencyKey)
     {
         await Task.Delay(1);
         return new PaymentResult(Guid.NewGuid().ToString("N"));
```

A stricter answer would stop retrying on **any** 429 unless `Retry-After`
is present and acceptable; this version always honours `Retry-After` when
given and otherwise backs off exponentially with jitter, capped at
`MaxAttempts`. The gateway call itself is still a stub (`SendChargeRequestAsync`
never actually throws these exceptions) — a real implementation would map
the HTTP client's status codes/headers onto them.

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
- After the fix: **5 passed** (1 smoke test + 3 regression cases), 0
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
