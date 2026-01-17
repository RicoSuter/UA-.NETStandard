# PR Title

Fix Subscription.Dispose() deadlock when called while holding a lock that callbacks need

# Description

## Summary

Fixes a deadlock in `Subscription.Dispose()` that occurs when called from code holding a lock that subscription callbacks also need to acquire.

## Root Cause

After PR #3133 (commit 876cf28c), `ResetPublishTimerAndWorkerState()` uses `.GetAwaiter().GetResult()` to synchronously wait for the message worker task. This creates a classic deadlock scenario:

1. Caller holds a lock and calls `Dispose()`
2. `Dispose()` blocks waiting for the worker task
3. Worker invokes `FastDataChangeCallback`
4. Callback tries to acquire the same lock
5. **Deadlock**: Dispose waits for worker, worker waits for lock

## Fix

The synchronous `Dispose()` no longer blocks waiting for the worker task:

1. Signal the worker to stop (cancel CTS, set the event)
2. Use `Task.ContinueWith` to dispose the CTS when the worker completes
3. Worker exits on its next loop iteration when it checks cancellation

This preserves the cleanup guarantees while avoiding the deadlock. An async version `ResetPublishTimerAndWorkerStateAsync()` is provided for async callers (like `DeleteAsync`) that can properly await.

## Testing

Added `DisposeDoesNotDeadlockDuringActiveDataChanges` test that:
- Creates a subscription with active data changes
- Uses a lock pattern that reproduces the deadlock scenario
- Verifies `Dispose()` completes within a reasonable timeout

## Changes

- `Subscription.cs`:
  - `ResetPublishTimerAndWorkerState()` - no longer blocks, uses continuation for cleanup
  - Added `ResetPublishTimerAndWorkerStateAsync()` for async callers

- `SubscriptionTest.cs`:
  - Added deadlock reproduction test

## Related

- Fixes regression introduced in #3133 / commit 876cf28c325701a90a3fcb2466f5ba2ddca7533f
