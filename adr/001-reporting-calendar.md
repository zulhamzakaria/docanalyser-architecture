# ADR 001: Anchoring the Reporting Calendar to the Dataset's UTC Offset

**Status:** Accepted (Oct 2026)

## Context
When a user asks, "What were my sales this year?", we must define what "this year" means. The server running the application might be in UTC, but the Shopify store that generated the CSV might be in `-04:00` (New York).

If we use the server's timezone or implicitly use UTC, a transaction that occurred at 10:00 PM on Dec 31st in New York will be bucketed into the wrong year. The LLM does not know the user's timezone.

## Decision
We establish the reporting calendar by calculating a **fixed UTC offset per dataset** based on the latest paid instant found during CSV ingestion.
1. **Ingestion:** The Parquet file stores pure UTC timestamps. However, we calculate the dataset's offset (e.g., `-04:00`) by querying the raw CSV *before* conversion.
2. **Planning:** The `AnalysisPlanner` uses this offset to build the exact `[start, end)` boundaries for the query.
3. **Lowering:** The SQL generated for DuckDB explicitly shifts the UTC timestamps into the store-local timezone before applying `date_trunc` or `EXTRACT(year)`.

## Consequences
*   **Positive:** The calendar matches the business reality of the store. "This month" means the store's month, regardless of where the server runs.
*   **Negative:** The system assumes a single fixed offset per dataset. It does not handle a dataset that spans multiple timezones or Daylight Saving Time transitions (a documented limitation).
*   **Negative:** We had to migrate away from DuckDB's native `TIMESTAMPTZ` bucketing, which relies on the session's timezone, to an explicit `AT TIME ZONE 'UTC' + INTERVAL ? MINUTE` SQL expression.
