# Data Pipeline & ETL Rules for Claude Code

# Data Pipeline & ETL Rules

## Pipeline Design
- Idempotent pipelines — re-running must produce the same result
- Incremental over full loads — never reprocess 5 years of data when you have deltas
- Checkpointing: resume from last successful point, not from the beginning
- Data lineage: track where every column came from — required for debugging and compliance
- Separate raw, staging, and transformed layers (medallion architecture: bronze/silver/gold)

## Data Quality
- Validate at every stage transition — fail fast, not at the end
- Schema enforcement: explicitly define and validate schemas (not "infer from data")
- Null handling policy defined per column — nulls are not errors unless they should be
- Row count checks: input rows vs output rows — unexplained drops are bugs
- Referential integrity checks before loading to final tables

```python
# Great Expectations or Soda for data quality
def validate_users_data(df: pd.DataFrame) -> None:
    assert df['user_id'].notna().all(), "user_id cannot be null"
    assert df['user_id'].is_unique, "user_id must be unique"
    assert df['email'].str.contains('@').all(), "invalid email format"
    assert (df['created_at'] <= pd.Timestamp.now()).all(), "future dates not allowed"
```

## Orchestration (Airflow/Prefect/Dagster)
- DAGs are code — version controlled, tested, reviewed like application code
- Task retries with exponential backoff for transient failures
- SLAs on critical pipelines — alert before business users notice
- Backfill strategy defined upfront — not figured out during an incident
- Dependencies between pipelines explicit in the orchestrator — not via cron timing

## Performance
- Partition tables by date for all time-series data — queries scan less data
- Columnar formats (Parquet, ORC) for analytical workloads — not CSV
- Cluster on join keys after partitioning — Spark/BigQuery/Snowflake specific
- Broadcast joins for small lookup tables — avoid shuffle for large tables
- Sample before aggregating when developing — don't run full dataset locally