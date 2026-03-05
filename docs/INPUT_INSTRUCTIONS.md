# Input Instructions for Snowbound Migrations

Provide the following for each migration request.

## Required

1. `ROOT_NAME`
   - Example: `BR_ALLORDERS`
2. `SOURCE_DIALECT`
   - One of: `MYSQL`, `MSSQL2016`, `SNOWFLAKE`
3. `SOURCE_SQL`
   - Full view/query text to migrate
4. `DATE_COLUMN`
   - Column used for hot/cold split (example: `WORKDATE`)
5. `HOT_DAYS`
   - Default `75` unless specified
6. `ROLLOVER_START_DAYS`
   - Default `120` unless specified

## Optional

1. Keying guidance
   - Preferred business keys for `RECORD_KEY` derivation
2. Late-arrival behavior
   - Any source-specific correction window beyond defaults
3. Task schedule/timezone
   - If different from daily schedule in local timezone
4. Validation checks
   - Row-level parity checks or aggregate checks to preserve

## Conversion commitments

For `MYSQL` and `MSSQL2016` sources, migration output will:

1. Detect and normalize SQL into Snowflake syntax.
2. Fully qualify source tables to:
   - `WORKWAVE_DATAFACTORY_DB.APP_BLUE_SKY_PEST_CORP.<TableName>`
3. Avoid Fivetran metadata/state logic unless source intent explicitly requires non-current semantics.
4. Use `<TableName>_H` or current+history union only when non-current logic is required.
5. Map `_FIVETRAN_SYNCED` to `_SYNCED` and emulate `_START`/`_END` intent with available EAST metadata.