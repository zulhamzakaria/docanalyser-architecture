
# The Reliability Layer: Taming Flaky LLM APIs

Building an LLM application requires assuming the provider will fail. Rate limits, network drops, and hidden SDK retries can turn a simple API call into an outage or a massive billing surprise. 

This document explains how DocAnalyser guarantees safe, bounded LLM execution.

## 1. The "Hidden Retry" Problem
We discovered that the default OpenAI .NET client (via `System.ClientModel`) has an **undocumented hidden retry policy**. If a network request fails, the SDK automatically retries it 3 times with exponential backoff. 
*   **The Impact:** A single user request could trigger 4 API calls, blowing past our RPM quota and hanging the HTTP thread for up to a minute. 
*   **The Fix:** We explicitly disabled the SDK's retry policy (`maxRetries: 0`). Our code is now the *only* layer that retries, ensuring 1 admitted attempt = exactly 1 HTTP request.

## 2. The Provider Call Gate
Before any LLM call is dispatched, it must pass through the `ProviderCallGate`. This acts as an in-memory bouncer.

*   **The Lease (Concurrency):** A `SemaphoreSlim` ensures that only *N* requests (default 1) are in flight at a time. This serializes the workflow and prevents thread-pool exhaustion.
*   **Sliding Window (RPM):** Tracks the exact timestamp of every admitted request. If the 60-second window is full, it calculates the exact time to wait. If the wait exceeds the remaining HTTP deadline, it fails instantly with a `429 Too Many Requests` rather than making the user wait just to fail.
*   **Rolling Window (Daily Cap):** Tracks a 24-hour history to prevent exceeding the provider's daily limit.
*   **Two Clocks:** We use **monotonic time** for the 60-second window (immune to server clock changes) and **wall-clock time** for the 24-hour window (matching the provider's reset logic).

## 3. The Resilient Chat Client
Wrapping the actual SDK call, this client manages the 20-second request deadline.

*   **The AsyncLocal Scope:** We inject a `RequestScope` into the async context, which carries the absolute deadline and a `CancellationToken`.
*   **The Timeout Race:** We wrap the SDK call in `Task.WaitAsync(token)`. If the LLM hangs, the token fires, throwing an exception that the client catches and maps to a clean `DeadlineExceeded` failure.
*   **Exception Classification:** We do not blindly catch all exceptions. A `429` fails immediately. A `500` or `503` triggers our custom exponential backoff. A `TaskCanceledException` (from an underlying HTTP timeout) is mapped to `ProviderUnavailable` rather than a deadline error.

## 4. The "Abandoned Task" Problem
C# cannot kill a running `Task`. If our 20-second deadline fires, we stop waiting for the LLM, but the underlying HTTP request continues in the background.
We use an `Interlocked` holder to track "abandoned" scopes. When an abandoned task eventually completes, it discards its result and safely disposes its memory without corrupting the application state.
