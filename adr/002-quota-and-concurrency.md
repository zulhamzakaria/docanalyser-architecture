# ADR 002: Rate Limiting and Concurrency for LLM Providers

**Status:** Accepted (Oct 2026)

## Context
Our LLM provider (Google Gemini) enforces strict rate limits: 15 Requests Per Minute (RPM) and 500 Requests Per Day (RPD). 

During implementation, we observed that the default OpenAI .NET SDK has a hidden retry policy: if a request fails (e.g., due to a network drop), it silently retries 3 times with exponential backoff. This meant a single user request could consume 4 units of our quota and hang the HTTP connection for up to a minute. Furthermore, we have no way to enforce a daily cap if we rely on the SDK's defaults.

## Decision
We implemented a custom `ProviderCallGate` and disabled the SDK's hidden retries.
1. **Disable Retries:** We set `ClientRetryPolicy(maxRetries: 0)` so our code is the only layer that retries.
2. **Custom Gate:** Before any LLM call is made, it must pass through the `ProviderCallGate`. 
3. **The Lock:** The Gate uses a `SemaphoreSlim` (with a default of 1 concurrent call) to serialize execution and prevent thread-pool exhaustion.
4. **The Quotas:** It tracks exact timestamps in memory. It enforces a sliding 60-second window (RPM) and a rolling 24-hour window (Daily Cap).
5. **Two Clocks:** It uses **monotonic time** for the 60-second window (immune to server clock changes) and **wall-clock time** for the 24-hour window (matching the provider's own reset logic).

## Consequences
*   **Positive:** We have hard guarantees on our API spend. We will never hit a `429 Too Many Requests` from the provider, because our local gate stops us first.
*   **Positive:** Network failures now fail in milliseconds (or retry exactly twice with a controlled backoff), rather than hanging for 60 seconds.
*   **Negative:** Because the gate is in-memory, restarting the server resets the daily counter. This is accepted for a local, single-instance deployment.
*   **Negative:** If a laptop sleeps for 25 hours, the monotonic clock may pause. The daily quota might falsely report `DailyCapReached` upon waking. (Accepted local limitation).
