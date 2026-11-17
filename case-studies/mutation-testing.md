# Case Study: Proving the Test Suite Works with Mutation Testing

It is easy to write tests that pass. It is much harder to write tests that actually catch bugs. To ensure the reliability of the `ProviderCallGate` and the `RequestScope`, I used **Mutation Testing**—deliberately breaking the code to prove the test suite goes red.

## The Scoreboard

| The Attack (Mutation) | Did the tests catch it? | What happened? |
| :--- | :--- | :--- |
| **Remove the Concurrency Lock** | 🟢 YES | The test launched 200 parallel threads. Without the lock, **65 requests** bypassed the limit of 50. |
| **Remove the Attempt Counter** | 🟢 YES | **23 tests** instantly failed with `Assert.Equal(Expected: 1, Actual: 0)`, proving the tests are deeply tied to actual API usage. |
| **Remove the Scope Restore** | 🟢 YES | **4 tests** failed with a `NullReferenceException`. The `AsyncLocal` trap test proved that an async helper drops the ambient context. |
| **Break the Clock Logic (UTC only)** | 🟢 YES | **1 test** failed. By ignoring the monotonic clock, a forward wall-clock jump let the rate limiter admit requests too early. |

## The "Trap Test"
The most critical mutation was testing the `AsyncLocal` behavior. C# `AsyncLocal` values flow *down* into awaited methods, but do not flow *back up* to the caller. If a developer accidentally creates the request scope inside an `async` helper method, the 20-second deadline will silently disappear.

To prevent this, we wrote a test that specifically creates the scope in an async helper and asserts that the caller sees `null`. If anyone ever accidentally refactors the code to hide the scope creation, this test will fail instantly.
