# Snowbound

Snowbound is the standard migration framework for moving legacy analytics SQL (MySQL, MSSQL 2016, and existing Snowflake views) into a cost-aware Snowflake EAST serving pattern.

## What we are doing

We apply the `RWA-75` pattern to each report/view:

1. Build one normalized source view in `HOLT.PIPELINE`.
2. Materialize historical rows in a transient history table.
3. Run a scheduled rollover task to keep history updated.
4. Publish one user-facing view in `HOLT.PUBLIC` that unions history with recent live rows.

This reduces recurring compute by avoiding full historical scans for every dashboard/report query.

## Standards in this repo

- Pattern name: `RWA-75` (Rolling Window Archive, 75-day hot window).
- Automation warehouse: `WH_AUTOMATION`.
- Pipeline schema: `HOLT.PIPELINE`.
- Public serving schema: `HOLT.PUBLIC`.
- Object naming: all pipeline objects start with the root view name.

Example for root `BR_ALLORDERS`:

- `HOLT.PIPELINE.BR_ALLORDERS_SOURCE_V`
- `HOLT.PIPELINE.BR_ALLORDERS_HIST`
- `HOLT.PIPELINE.BR_ALLORDERS_ROLLOVER`
- `HOLT.PUBLIC.BR_ALLORDERS`

## Source SQL translation layer (mandatory)

Before applying `RWA-75`, detect source SQL dialect and normalize to Snowflake EAST rules:

- If source dialect is MySQL or MSSQL 2016, fully qualify table references to:
  - `WORKWAVE_DATAFACTORY_DB.APP_BLUE_SKY_PEST_CORP.<TableName>`
- Default to current table (`<TableName>`) with no Fivetran state filters.
- Only use Fivetran/history logic when source intent requires non-current semantics.
- If historical rows are required, use `<TableName>_H` and union as needed.
- Map `_FIVETRAN_SYNCED` to `_SYNCED`.
- Translate `_START`/`_END` intent using available EAST metadata (typically `_SYNCED`).

## Inputs and runbook

- Runbook: `docs/RWA-75.md`
- Input template: `docs/INPUT_INSTRUCTIONS.md`