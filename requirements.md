# Role

You are a senior Python automation architect and production-grade framework engineer.

# Task

Create a robust, production-ready Python script/module for large data comparison in the existing `data-validation-framework`.

The new implementation should support:

1. File-to-file comparison
2. File-to-table comparison
3. SQL Server connection using SQLAlchemy
4. DuckDB SQL as the large-data comparison engine
5. Data-driven execution using the existing framework-style config file
6. Existing-style HTML/Excel reporting similar to the current data-validation-framework
7. Logs and output reports written to the same respective folders as the existing framework
8. DuckDB memory limit, temp/spill directory, threads, and other parameters passed from the config file

The implementation should avoid loading the full source/target data into Pandas memory.

---

# Context

The existing framework is a Python-based Data Validation Framework. It already supports validations like file-to-file, file-to-table, table-to-table, and table-to-file. It uses a config-driven approach, where source, target, join columns, compare columns, and execution flags are provided through a config file or Excel sheet.

The current framework already has reporting logic that generates HTML reports with summary, matched/unmatched records, mismatches, and failure details.

For large files, Pandas/DataComPy causes memory issues. Therefore, create a new large-data comparison implementation using DuckDB SQL.

The new engine should integrate with the existing framework without disturbing the current small-file comparison flow.

---

# High-Level Requirement

Create a new compare engine called:

```text
duckdb_sql_engine.py
```

The engine should compare large datasets using DuckDB SQL.

The engine should support:

```text
file_to_file
file_to_table
```

For file-to-table:

```text
Source: CSV file
Target: SQL Server table/query
Connection: SQLAlchemy
Comparison engine: DuckDB SQL
```

For file-to-file:

```text
Source: CSV file
Target: CSV file
Comparison engine: DuckDB SQL
```

The engine should generate failure files and summary outputs that can be passed to the existing report generator.

---

# Important Design Principle

Do not return full DataFrames from comparison functions.

Bad:

```python
mismatch_df = con.execute(sql).fetchdf()
return mismatch_df
```

Good:

```python
con.execute(\"\"\"
COPY (
    SELECT ...
)
TO 'output/failures/value_mismatches.csv'
(HEADER, DELIMITER ',');
\"\"\")
return summary_counts
```

All large outputs should be written directly to disk using DuckDB `COPY`.

Only small summary counts and sample records should be loaded into Pandas.

---

# Expected Folder Structure

Use the existing framework folders if available. Otherwise create this structure under the configured output base directory:

```text
output/
  <run_id>/
    comparison_report.html
    comparison_summary.xlsx
    summary.json

    failures/
      only_in_source.csv
      only_in_target.csv
      value_mismatches.csv
      source_duplicate_keys.csv
      target_duplicate_keys.csv
      column_mismatch_summary.csv

    samples/
      only_in_source_sample.csv
      only_in_target_sample.csv
      value_mismatches_sample.csv
      source_duplicate_keys_sample.csv
      target_duplicate_keys_sample.csv

    logs/
      <run_id>.log

    duckdb/
      comparison.duckdb

    duckdb_spill/
      temporary spill files
```

After successful execution, if `cleanup_duckdb_spill = true`, delete only the spill directory. Do not delete output reports or failure files.

---

# Config-Driven Design

The script should read all server, file path, output path, and DuckDB parameters from the config.

Support config through YAML first. Keep the code modular so it can later be integrated with Excel-based `Config` and `Test_Data` sheets.

Example file-to-table config:

```yaml
run:
  validation_name: large_csv_vs_sqlserver_validation
  validation_type: file_to_table
  is_active: true
  run_id: auto
  output_base_dir: D:/data_validation_framework/output
  logs_base_dir: D:/data_validation_framework/logs

source:
  type: csv
  name: source_csv
  file_path: D:/data/source/customer_source.csv
  delimiter: ","
  encoding: utf-8
  header: true

target:
  type: sqlserver
  name: sqlserver_target
  server: MY_SERVER_NAME
  database: MY_DATABASE
  driver: ODBC Driver 17 for SQL Server
  authentication: windows
  username:
  password:
  table:
  query: |
    SELECT
      policy_id,
      customer_name,
      status,
      premium_amount,
      effective_date
    FROM dbo.PolicyTable

comparison:
  join_columns:
    - policy_id

  compare_columns:
    - customer_name
    - status
    - premium_amount
    - effective_date

  trim_columns: true

  ignore_case_columns:
    - customer_name
    - status

  date_columns:
    - effective_date

  numeric_columns:
    - premium_amount

  numeric_tolerance:
    premium_amount: 0.01

  null_equivalents:
    - ""
    - "NULL"
    - "None"
    - "nan"

  fail_on_schema_mismatch: true
  fail_on_duplicate_keys: true
  sample_limit: 100

duckdb:
  database_path: auto
  memory_limit: 8GB
  temp_directory: auto
  max_temp_directory_size: 100GB
  threads: 4
  preserve_insertion_order: false
  enable_progress_bar: true
  stage_source_as_table: true
  stage_target_as_table: true
  cleanup_spill_directory: true

sqlalchemy:
  chunksize: 100000
  echo: false
  pool_pre_ping: true
  fast_executemany: true

reporting:
  generate_html: true
  generate_excel: true
  generate_summary_json: true
  open_report_after_run: false
```

Example file-to-file config:

```yaml
run:
  validation_name: large_file_to_file_validation
  validation_type: file_to_file
  is_active: true
  run_id: auto
  output_base_dir: D:/data_validation_framework/output
  logs_base_dir: D:/data_validation_framework/logs

source:
  type: csv
  name: source_csv
  file_path: D:/data/source/source.csv
  delimiter: ","
  encoding: utf-8
  header: true

target:
  type: csv
  name: target_csv
  file_path: D:/data/target/target.csv
  delimiter: ","
  encoding: utf-8
  header: true

comparison:
  join_columns:
    - policy_id

  compare_columns:
    - customer_name
    - status
    - premium_amount
    - effective_date

  trim_columns: true
  ignore_case_columns:
    - customer_name
    - status
  date_columns:
    - effective_date
  numeric_columns:
    - premium_amount
  numeric_tolerance:
    premium_amount: 0.01
  sample_limit: 100

duckdb:
  database_path: auto
  memory_limit: 8GB
  temp_directory: auto
  max_temp_directory_size: 100GB
  threads: 4
  preserve_insertion_order: false
  enable_progress_bar: true
  stage_source_as_table: true
  stage_target_as_table: true
  cleanup_spill_directory: true

reporting:
  generate_html: true
  generate_excel: true
  generate_summary_json: true
```

---

# Required Python Files

Create or update the following files:

```text
compare_engines/
  duckdb_sql_engine.py

utils/
  duckdb_manager.py
  sqlalchemy_connection.py
  large_compare_sql_builder.py
  report_context_builder.py
  logger_utils.py
  file_utils.py

reports/
  large_compare_html_report.py
  large_compare_excel_report.py

main_large_compare.py
```

If the existing framework already has similar modules, reuse them instead of duplicating.

---

# Functional Requirements

## 1. DuckDB Manager

Create a reusable `DuckDBManager` class.

Responsibilities:

```text
Create DuckDB database
Configure memory limit
Configure temp/spill directory
Configure max temp directory size
Configure thread count
Configure preserve_insertion_order
Configure progress bar
Create run folders
Close connection safely
Cleanup spill directory if configured
```

Example settings:

```sql
SET memory_limit = '8GB';
SET temp_directory = '<run_dir>/duckdb_spill';
SET max_temp_directory_size = '100GB';
SET threads = 4;
SET preserve_insertion_order = false;
SET enable_progress_bar = true;
```

---

## 2. SQLAlchemy Connection

Create SQLAlchemy connection builder for SQL Server.

Support Windows authentication and SQL authentication.

For Windows authentication, use a connection string similar to:

```python
mssql+pyodbc://@SERVER_NAME/DATABASE_NAME?driver=ODBC+Driver+17+for+SQL+Server&trusted_connection=yes
```

For SQL authentication:

```python
mssql+pyodbc://username:password@SERVER_NAME/DATABASE_NAME?driver=ODBC+Driver+17+for+SQL+Server
```

Use URL encoding for driver names and credentials.

Do not print passwords in logs.

---

## 3. Stage Source CSV into DuckDB

For file source, create either a DuckDB physical table or view based on config.

Recommended for large files:

```sql
CREATE OR REPLACE TABLE source_data AS
SELECT selected columns
FROM read_csv_auto('<source_file_path>', HEADER = true);
```

The selected columns should include:

```text
join_columns + compare_columns
```

Do not use `SELECT *` unless explicitly configured.

---

## 4. Stage Target CSV into DuckDB for file-to-file

For file-to-file target, create:

```sql
CREATE OR REPLACE TABLE target_data AS
SELECT selected columns
FROM read_csv_auto('<target_file_path>', HEADER = true);
```

Again, use only selected columns.

---

## 5. Stage SQL Server Target into DuckDB for file-to-table

Use the existing SQLAlchemy approach.

Read SQL Server query in chunks:

```python
pd.read_sql_query(query, sqlalchemy_engine, chunksize=configured_chunksize)
```

For each chunk:

```text
Register chunk as DuckDB temp relation
Create target_data table from first chunk
Append subsequent chunks using INSERT INTO target_data SELECT * FROM chunk
Unregister chunk
Log chunk number and row count
```

Do not keep all chunks in memory.

Pseudo logic:

```python
first_chunk = True
for chunk_df in pd.read_sql_query(query, engine, chunksize=chunksize):
    con.register("sqlserver_chunk", chunk_df)

    if first_chunk:
        con.execute(\"\"\"
            CREATE OR REPLACE TABLE target_data AS
            SELECT selected columns
            FROM sqlserver_chunk
        \"\"\")
        first_chunk = False
    else:
        con.execute(\"\"\"
            INSERT INTO target_data
            SELECT selected columns
            FROM sqlserver_chunk
        \"\"\")

    con.unregister("sqlserver_chunk")
```

---

## 6. Schema Validation

Before comparison:

```text
Check source columns exist
Check target columns exist
Check join columns exist
Check compare columns exist
Report missing columns
Fail if fail_on_schema_mismatch = true
```

Use DuckDB `DESCRIBE source_data` and `DESCRIBE target_data`.

---

## 7. Normalized Views

Create `source_norm` and `target_norm`.

Normalization rules should come from config.

Rules:

```text
Join columns: cast to VARCHAR and trim
Ignore-case columns: lower(trim(cast(col as varchar)))
Normal text columns: trim(cast(col as varchar))
Date columns: TRY_CAST(col AS DATE)
Numeric columns: TRY_CAST(col AS DOUBLE)
Null equivalents: convert configured values to NULL where practical
```

Example:

```sql
CREATE OR REPLACE VIEW source_norm AS
SELECT
    NULLIF(trim(CAST(policy_id AS VARCHAR)), '') AS policy_id,
    lower(NULLIF(trim(CAST(customer_name AS VARCHAR)), '')) AS customer_name,
    lower(NULLIF(trim(CAST(status AS VARCHAR)), '')) AS status,
    TRY_CAST(premium_amount AS DOUBLE) AS premium_amount,
    TRY_CAST(effective_date AS DATE) AS effective_date
FROM source_data;
```

Generate this SQL dynamically from config.

---

## 8. Duplicate Key Checks

Generate:

```text
source_duplicate_keys.csv
target_duplicate_keys.csv
source_duplicate_keys_sample.csv
target_duplicate_keys_sample.csv
```

For single-column key:

```sql
SELECT policy_id, COUNT(*) AS duplicate_count
FROM source_norm
GROUP BY policy_id
HAVING COUNT(*) > 1
```

For composite key, group by all join columns.

---

## 9. Only-in-Source and Only-in-Target

Use DuckDB `ANTI JOIN`.

For source-only:

```sql
COPY (
    SELECT s.*
    FROM source_norm s
    ANTI JOIN target_norm t
    USING (<join_columns>)
)
TO '<failures>/only_in_source.csv'
(HEADER, DELIMITER ',');
```

For target-only:

```sql
COPY (
    SELECT t.*
    FROM target_norm t
    ANTI JOIN source_norm s
    USING (<join_columns>)
)
TO '<failures>/only_in_target.csv'
(HEADER, DELIMITER ',');
```

Also create sample files with `LIMIT sample_limit`.

---

## 10. Value Mismatch Detection

Generate mismatch SQL dynamically.

For each compare column, output:

```text
source_<column>
target_<column>
<column>_mismatch_flag
```

Example output columns:

```text
policy_id
customer_name_mismatch
source_customer_name
target_customer_name
status_mismatch
source_status
target_status
premium_amount_mismatch
source_premium_amount
target_premium_amount
effective_date_mismatch
source_effective_date
target_effective_date
```

Use:

```sql
IS DISTINCT FROM
```

for null-safe comparison.

For numeric tolerance:

```sql
ABS(s.premium_amount - t.premium_amount) > 0.01
```

Handle nulls carefully:

```sql
(
    s.premium_amount IS DISTINCT FROM t.premium_amount
    AND (
        s.premium_amount IS NULL
        OR t.premium_amount IS NULL
        OR ABS(s.premium_amount - t.premium_amount) > 0.01
    )
)
```

Export:

```text
value_mismatches.csv
value_mismatches_sample.csv
```

---

## 11. Column Mismatch Summary

Create:

```text
column_mismatch_summary.csv
```

Example:

```text
column_name,mismatch_count
customer_name,120
status,45
premium_amount,900
effective_date,12
```

Generate one query per compare column and combine using `UNION ALL`.

---

## 12. Summary Counts

Generate summary metrics:

```text
source_row_count
target_row_count
matched_key_count
only_in_source_count
only_in_target_count
mismatch_row_count
source_duplicate_key_count
target_duplicate_key_count
column_mismatch_count
final_status
```

Only fetch this small summary result into Pandas/dict.

Write:

```text
summary.json
```

---

## 13. Report Context

Create a common report context dictionary that looks similar to the existing framework’s reporting object.

Example:

```python
report_context = {
    "run_id": run_id,
    "validation_name": validation_name,
    "validation_type": validation_type,
    "comparison_engine": "DuckDB SQL",
    "status": "PASSED" or "FAILED",
    "source_name": source_name,
    "target_name": target_name,
    "join_columns": join_columns,
    "compare_columns": compare_columns,
    "summary": summary,
    "column_summary": column_summary_records,
    "samples": {
        "only_in_source": only_in_source_sample_records,
        "only_in_target": only_in_target_sample_records,
        "value_mismatches": value_mismatch_sample_records,
        "source_duplicate_keys": source_duplicate_sample_records,
        "target_duplicate_keys": target_duplicate_sample_records
    },
    "output_files": {
        "only_in_source": path,
        "only_in_target": path,
        "value_mismatches": path,
        "column_mismatch_summary": path,
        "source_duplicate_keys": path,
        "target_duplicate_keys": path,
        "summary_json": path,
        "html_report": path,
        "excel_report": path,
        "log_file": path
    },
    "technical_details": {
        "duckdb_database": path,
        "duckdb_temp_directory": path,
        "duckdb_memory_limit": value,
        "duckdb_threads": value,
        "sqlalchemy_chunksize": value
    }
}
```

This object should be passed to the report generator.

---

## 14. HTML Report

Create an HTML report similar to the current data-validation-framework style.

The report should include:

```text
Execution Summary
Reconciliation Summary
Matched On
Column Mismatch Summary
Failure Samples
Output File Links
Technical Details
```

Use Jinja2 if available.

The report should not embed huge CSV files. It should show only sample records and link to full failure CSV files.

---

## 15. Excel Report

Create an Excel report with these sheets:

```text
Summary
Column_Mismatch_Summary
Only_In_Source_Sample
Only_In_Target_Sample
Value_Mismatches_Sample
Source_Duplicate_Keys_Sample
Target_Duplicate_Keys_Sample
Output_Files
Technical_Details
```

Use `openpyxl`.

Freeze headers, apply basic formatting, autofit columns where practical.

---

## 16. Logging

Create timestamped log file under the configured logs folder.

Log:

```text
Run ID
Config file used
Validation type
Source details
Target details
DuckDB settings
SQLAlchemy connection status, without password
CSV staging start/end
SQL Server staging start/end
Chunk number and rows loaded
Schema validation result
Duplicate check result
Only-in-source export result
Only-in-target export result
Value mismatch export result
Summary metrics
Report generation paths
Total execution time
Errors with stack trace
```

---

## 17. Error Handling

Implement strong error handling.

Handle:

```text
Missing config file
Invalid validation_type
Missing source file
SQL Server connection failure
SQL query failure
Missing columns
DuckDB SQL failure
Output folder permission issue
Disk space issue if detectable
Empty source
Empty target
Duplicate keys when fail_on_duplicate_keys = true
```

On failure, write the error to log and generate a minimal failure summary if possible.

---

## 18. CLI Interface

Create a CLI entrypoint:

```bash
python main_large_compare.py --config config/large_compare.yml
```

Optional:

```bash
python main_large_compare.py --config config/large_compare.yml --run-id RUN_20260522_001
```

---

# Implementation Quality Requirements

The code should be:

```text
Production-ready
Modular
Readable
Well documented
Type hinted
Config-driven
Memory efficient
Safe for large files
Easy to integrate into existing framework
```

Use classes where appropriate.

Avoid hardcoded file paths.

Avoid hardcoded columns.

Avoid hardcoded SQL Server details.

Avoid full Pandas loading for source/target data.

Use DuckDB `COPY` for writing large result sets.

---

# Expected Final Output

After execution, the script should print:

```text
Validation completed.

Run ID: RUN_YYYYMMDD_HHMMSS
Status: PASSED/FAILED
HTML Report: <path>
Excel Report: <path>
Summary JSON: <path>
Log File: <path>
```

---

# Additional Notes

Use DuckDB SQL for full comparison.

Do not force DataComPy for large comparisons. The reporting should be similar to the existing DataComPy-style report, but the actual comparison engine should be DuckDB SQL.

The design should allow the existing framework to choose the comparison engine like this:

```text
pandas_datacompy -> small files
duckdb_sql       -> large files
```

Add this parameter to config:

```yaml
comparison_engine: duckdb_sql
```

The implementation should be created in a way that it can later be called from the existing `main.py` based on validation type and comparison engine.
