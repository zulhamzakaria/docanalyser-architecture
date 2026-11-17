# DocAnalyser: A Reliability-First NL-to-SQL Engine

## The Problem
Letting an LLM generate SQL for a CSV is easy. Letting it do so **reliably, safely, and deterministically** for a production system is incredibly hard. LLMs hallucinate schemas, struggle with time zones, and are subject to strict rate limits. 

DocAnalyser is a backend-first Natural Language to Analysis tool. You upload a Shopify CSV, ask a question in plain English, and get a bounded, exact, SQL-backed answer.

## The Architecture
This project decouples the "intelligence" from the "execution":
1. **LLM (The Intent):** The model only translates natural language into a strict, validated JSON schema (`AnalysisIntent`). It never sees the raw data, and it never writes SQL.
2. **Compiler (The Plan):** A deterministic C# planner resolves the JSON intent against a semantic manifest.
3. **DuckDB (The Execution):** The query is lowered to parameterized SQL and executed in a sandboxed, in-memory DuckDB instance.

## The Engineering Moat (Why this repo exists)
I am keeping the core source code private because the LLM prompts and semantic mappings are proprietary. However, this repository documents the **architecture, reliability constraints, and trade-offs** required to make LLM pipelines production-ready.

Read the Architecture Decision Records (ADRs) to see how I solved:
*   **Clock Drift:** How to enforce daily quotas across wall-clock NTP jumps and laptop sleep states (`ADRs/002-quota-and-concurrency.md`).
*   **Hidden API Limits:** How to discover and disable the OpenAI SDK's silent retry loops to prevent quota exhaustion (`ADRs/002-quota-and-concurrency.md`).
*   **Timezone Hallucinations:** How to anchor temporal queries to the dataset's original UTC offset rather than the server's local timezone (`ADRs/001-reporting-calendar.md`).
*   **Async Lifetimes:** How to prevent `AsyncLocal` scope leaks when a deadline cancels a background task (`ADRs/003-async-lifetime.md`).

## The Testing Philosophy
This project does not just rely on "happy path" unit tests. It uses **Mutation Testing** to prove the test suite works. For example, if I remove the concurrency lock, exactly 17 tests fail. If I break the timezone logic, 1 test fails. You can read the full mutation scoreboard in the ADRs.
