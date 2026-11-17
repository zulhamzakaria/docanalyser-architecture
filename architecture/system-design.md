# System Design: The NL-to-SQL Pipeline

## The Core Problem
A standard "AI wrapper" simply takes a user's question, pastes it into a prompt alongside the raw CSV data, and asks the LLM to output an answer. This fails for three reasons:
1. **Privacy & Scale:** LLMs have context limits and you should not send entire databases to a third party.
2. **Math & Accuracy:** LLMs are notoriously bad at exact arithmetic and complex aggregations.
3. **Security:** Letting an LLM write raw SQL opens the door to SQL injection and destructive queries.

## The Solution: Compile, Don't Generate
DocAnalyser treats the LLM as a **compiler front-end**, not a database engine. The LLM is only responsible for translating Natural Language into a strict, validated JSON schema (`AnalysisIntent`). The actual data processing is handled deterministically by DuckDB.

### The Pipeline

```mermaid
graph TD
    A[User Question] -->|LLM API| B(Intent Extraction)
    B -->|Strict JSON| C{Semantic Manifest}
    C -->|C# Planner| D[Analysis Plan]
    D -->|C# Lowerer| E[Parameterized SQL]
    E -->|Sandboxed Executor| F[(DuckDB Parquet)]
    F -->|Result Set| G[JSON Response]
