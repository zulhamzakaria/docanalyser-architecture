# ADR 003: Managing Lifetimes for Abandoned Async Tasks

**Status:** Accepted (Oct 2026)

## Context
We enforce a strict 20-second timeout on LLM extraction. If the LLM takes 21 seconds, we throw a `DeadlineExceeded` error back to the user.

However, C# cannot forcibly abort a running `Task`. When the 20-second timeout fires, the workflow stops waiting, but the underlying HTTP request to the LLM provider keeps running in the background. 

This creates two severe problems:
1. **The Abandoned Scope:** We use an `AsyncLocal` to carry the request's deadline and cancellation tokens. If the background task later finishes, it might accidentally touch a disposed token or alter our rate-limiting state.
2. **Disposal:** We cannot dispose the cancellation tokens while the background task is still running, or it will crash.

## Decision
We implemented a reference-counted "Holder" pattern using `Interlocked`.
1. We cannot kill the background task, so we allow it to run.
2. We wrap the scope's cancellation sources in a holder object with a thread-safe `IsAbandoned` flag.
3. When the workflow times out, it marks the holder as `Abandoned`.
4. When the scope is disposed, it checks the flag. If `Abandoned`, it deliberately **does not dispose** the underlying CancellationTokenSources, leaving them to the Garbage Collector once the background task finally completes. 
5. We use a "Trap Test" to prove that the scope cannot leak upward into the caller's thread.

## Consequences
*   **Positive:** The application does not crash from `ObjectDisposedException` or `NullReferenceException` when a race condition occurs between a timeout and a delayed network response.
*   **Positive:** The `ProviderCallGate` remains completely unaware of abandoned tasks; the lifetime management is entirely encapsulated within the request scope.
*   **Negative:** We temporarily leak memory (the abandoned token sources) until the Garbage Collector runs. This is an acceptable trade-off for avoiding the massive complexity of a distributed task-tracking system. 
