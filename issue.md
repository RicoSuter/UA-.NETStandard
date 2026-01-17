# Subscription.Dispose() can deadlock when called while holding a lock that callbacks need

## Description

`Subscription.Dispose()` can deadlock when called from code that holds a lock which the `FastDataChangeCallback` (or other subscription callbacks) also needs to acquire.

The root cause is that `ResetPublishTimerAndWorkerState()` uses `.GetAwaiter().GetResult()` to synchronously wait for the message worker task to complete:

```csharp
private void ResetPublishTimerAndWorkerState()
{
    ResetPublishTimerAndWorkerStateAsync().GetAwaiter().GetResult(); // Blocks indefinitely
}
```

This creates a deadlock when:
1. Caller holds a lock (e.g., `_subscriptionsLock` in a SubscriptionsManager)
2. Caller calls `Subscription.Dispose()`
3. `Dispose()` blocks waiting for the message worker task via `.GetAwaiter().GetResult()`
4. The message worker invokes `FastDataChangeCallback`
5. The callback (or downstream observers) tries to acquire the same lock
6. **Deadlock**: Dispose waits for worker, worker waits for lock held by Dispose caller

## Reproduction

This is a common pattern when managing multiple subscriptions:

```csharp
// SubscriptionsManager.ClearSubscriptions()
lock (_subscriptionsLock)  // Step 1: Acquire lock
{
    while (_subscriptions.TryPop(out var subscription))
    {
        subscription.Dispose();  // Step 2-3: Blocks waiting for worker
    }
}

// Meanwhile, on the worker thread:
// FastDataChangeCallback is invoked, observer tries to call GetMonitoredItems()
lock (_subscriptionsLock)  // Step 5: BLOCKED - deadlock!
{
    return _subscriptions.SelectMany(...);
}
```

### Observations
- When there are **no active data changes**, `Dispose()` completes quickly (worker exits fast)
- When there are **active data changes**, `Dispose()` deadlocks (worker is processing callbacks)

## Environment

- **Library version**: 1.5.x (introduced in commit 876cf28c325701a90a3fcb2466f5ba2ddca7533f)
- **Platforms affected**: All (.NET Framework, .NET 6+)

## Suggested Fix

The synchronous `Dispose()` should not block waiting for the worker task. Instead:
1. Signal the worker to stop (cancel CTS, set the event)
2. Use `Task.ContinueWith` to dispose the CTS when the worker completes
3. The worker will exit on its next loop iteration when it checks `ct.IsCancellationRequested`

This avoids the deadlock without requiring timeouts or `IAsyncDisposable`.

See related PR for the implementation.

## Related

- Introduced in: https://github.com/OPCFoundation/UA-.NETStandard/commit/876cf28c325701a90a3fcb2466f5ba2ddca7533f
- PR #3133: "Fix subscription leaking due to not completed PublishResponseMessageWorkerAsync tasks"
