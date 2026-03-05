# RWA-75 Standard (Snowbound)

## Purpose

Apply a repeatable, low-cost serving pattern for analytics views in Snowflake by splitting historical and recent data.

## Parameters

- `ROOT_NAME`: report/view root name (example: `BR_ALLORDERS`)
- `HOT_DAYS`: recent live window (default `75`)
- `ROLLOVER_START_DAYS`: late-arrival buffer scan start (default `120`)
- `WAREHOUSE`: always `WH_AUTOMATION`
- `PIPELINE_SCHEMA`: `HOLT.PIPELINE`
- `PUBLIC_SCHEMA`: `HOLT.PUBLIC`

## Step 0: Dialect detection and EAST normalization

Classify source SQL as `MYSQL`, `MSSQL2016`, or `SNOWFLAKE`.

If dialect is `MYSQL` or `MSSQL2016`:

1. Rewrite every table reference to:
   - `WORKWAVE_DATAFACTORY_DB.APP_BLUE_SKY_PEST_CORP.<TableName>`
2. Use current table by default (`<TableName>`), no active-state metadata filters.
3. Add history/Fivetran logic only when source asks for non-current behavior.
4. For non-current requirements:
   - use `<TableName>_H`, or
   - `UNION` `<TableName>` with `<TableName>_H` when both are needed.
5. Metadata mapping:
   - `_FIVETRAN_SYNCED` -> `_SYNCED`
   - `_START` and `_END` have no direct EAST equivalent; emulate intent with available metadata, usually `_SYNCED`.

Output of this step is a Snowflake-safe query body for `<ROOT_NAME>_SOURCE_V`.

## Step 1: Create normalized source view

Create `HOLT.PIPELINE.<ROOT_NAME>_SOURCE_V` from the normalized query body.

## Step 2: Backfill history table

Create `HOLT.PIPELINE.<ROOT_NAME>_HIST` as `TRANSIENT` with:

- all output columns from source view
- computed `RECORD_KEY` for merge idempotency
- rows where `WORKDATE < CURRENT_DATE - HOT_DAYS`

Recommended `RECORD_KEY` pattern:

1. `INV|<InvoiceID>` when invoice id exists
2. `SO|<LocationID>|<OrderID>` when service-order id exists
3. hash fallback from stable fields when both are null

## Step 3: Create rollover task

Create `HOLT.PIPELINE.<ROOT_NAME>_ROLLOVER`:

- warehouse: `WH_AUTOMATION`
- schedule: daily cron (local timezone)
- logic: `MERGE` records from source view where
  - `WORKDATE >= CURRENT_DATE - ROLLOVER_START_DAYS`
  - `WORKDATE < CURRENT_DATE - HOT_DAYS`

Use `SUSPEND_TASK_AFTER_NUM_FAILURES = 3`.

## Step 4: Create public serving view

Create `HOLT.PUBLIC.<ROOT_NAME>` as:

- `SELECT ... FROM <ROOT_NAME>_HIST`
- `UNION ALL`
- `SELECT ... FROM <ROOT_NAME>_SOURCE_V WHERE WORKDATE >= CURRENT_DATE - HOT_DAYS`

Use `COPY GRANTS` on the public view.

## Step 5: Cleanup temporary objects

After cutover, drop any temporary migration views from `HOLT.PUBLIC`.

## Monitoring

Task health query:

```sql
SELECT
  SCHEDULED_TIME,
  STATE,
  QUERY_ID,
  ERROR_CODE,
  ERROR_MESSAGE,
  COMPLETED_TIME
FROM TABLE(
  INFORMATION_SCHEMA.TASK_HISTORY(
    TASK_NAME => 'HOLT.PIPELINE.<ROOT_NAME>_ROLLOVER',
    SCHEDULED_TIME_RANGE_START => DATEADD('day', -14, CURRENT_TIMESTAMP())
  )
)
ORDER BY SCHEDULED_TIME DESC;
```

Data checks:

```sql
SELECT
  (SELECT COUNT(*) FROM HOLT.PUBLIC.<ROOT_NAME>
    WHERE WORKDATE::DATE >= DATEADD(DAY, -<HOT_DAYS>, CURRENT_DATE())) AS FINAL_RECENT_ROWS,
  (SELECT COUNT(*) FROM HOLT.PIPELINE.<ROOT_NAME>_SOURCE_V
    WHERE WORKDATE::DATE >= DATEADD(DAY, -<HOT_DAYS>, CURRENT_DATE())) AS SOURCE_RECENT_ROWS,
  (SELECT COUNT(*) - COUNT(DISTINCT RECORD_KEY)
    FROM HOLT.PIPELINE.<ROOT_NAME>_HIST) AS HIST_DUP_KEYS;
```