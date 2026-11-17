# Temporal Calendars: Solving Timezone Hallucinations

When a user asks, "What were my sales this month?", the answer depends entirely on *where* the store is located. 

If a transaction happened at `11:30 PM on Dec 31st` in New York (`-05:00`), it must be counted in that year. If the server processing the query is in UTC, it will see the transaction as `04:30 AM on Jan 1st` of the *next* year.

## The Implicit UTC Trap
Most systems store all dates in UTC and assume that is enough. But if you run a `date_trunc('year', timestamp)` function on a UTC timestamp, you will misclassify millions of dollars of revenue at the year boundary.

## Our Solution: Anchor to the Store's Offset
During CSV ingestion, DocAnalyser scans the raw file to determine the store's UTC offset (e.g., `+08:00`). It stores this offset as metadata alongside the Parquet file.

When generating SQL, we do not use native timezone functions (which depend on the database's session timezone). Instead, we explicitly shift the timestamps before bucketing them:

```sql
-- We convert the raw timestamp to UTC, then apply the offset via a bound parameter
SELECT date_trunc('month', (paid_at AT TIME ZONE 'UTC') + INTERVAL (?) MINUTE)
