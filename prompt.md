Review the existing Polars-based large file validation logic in this repository, but do not modify or remove the current Polars implementation.

Create a new DuckDB-based script/module for large file validation because the current Polars path works for sample files but fails with Out Of Memory errors on large files.

Requirements:

1. Preserve existing implementation
- Do not modify, remove, or refactor the existing Polars script unless absolutely required for import/reference only.
- Create a new DuckDB implementation as a separate script/module.
- The DuckDB script should follow the same input configuration approach used by the existing Polars logic.
- The DuckDB implementation should produce the same output files, output folder structure, report structure, naming conventions, and report artifacts as the current Polars implementation.
- Existing small-file/sample-file Polars execution should continue to work unchanged.

2. New DuckDB script/module
- Add a new script, for example:
  - `duckdb_large_file_validation.py`
  - or a similarly named module that fits the current project structure.
- Reuse existing utility functions where appropriate.
- Use DuckDB SQL-based processing wherever possible.
- Avoid loading full large files into pandas or Polars.
- Keep the code modular so it can later be called from the main framework if needed.

3. Modern DuckDB streaming-first implementation
- Use modern DuckDB features with a streaming-first design.
- Prefer DuckDB scans and SQL pipelines over Python DataFrame materialization.
- Use DuckDB functions such as:
  - `read_csv`
  - `read_csv_auto`
  - `read_parquet`
  - `read_json_auto`
  - `COPY (...) TO`
  - views
  - temporary tables only when they improve performance or avoid repeated scans
- Use streaming execution wherever DuckDB supports it.
- Do not call `.df()`, `.fetchdf()`, `.pl()`, `.fetchall()`, or similar methods on large result sets.
- For large outputs, always use `COPY (SELECT ...) TO '<output_file>'` so DuckDB writes directly to disk.
- Push filtering, joining, aggregation, sorting, casting, hashing, lookup checks, and validation rules into DuckDB SQL.
- Avoid row-by-row Python processing.
- Avoid unnecessary intermediate Python objects.
- Avoid repeated full-file scans where possible.
- Use projection pushdown: read only required columns where possible.
- Apply filters as early as possible.
- Avoid `ORDER BY` unless the existing output requires deterministic ordering.
- For lookup joins, project only the required lookup columns.
- For repeated validations on the same large source, consider creating a temporary DuckDB table or temporary Parquet staging file if it reduces repeated scans.
- Prefer Parquet staging for very large intermediate datasets when it improves performance and stability.
- Use DuckDB `COPY` to write staged Parquet files if needed.
- Ensure any staging files are created only inside the DuckDB temp/spill folder or a controlled temporary folder and are cleaned up after execution.

4. DuckDB performance and resource configuration
- Configure DuckDB for stable execution on a VDI with limited resources.
- Use:
  - `threads = 3`
  - `memory_limit = '10GB'`
  - `preserve_insertion_order = false`
  - `temp_directory = '<framework_root>/temp/duckdb_spill_<run_id>'`
- Enable disk spillover through DuckDB temp directory configuration.
- Make these values configurable at the top of the script or through existing config.
- Log all DuckDB configuration values at runtime.

5. Temporary spill/staging folder
- Create a temporary DuckDB spill/staging folder under the framework directory, for example:
  - `framework_root/temp/duckdb_spill_<run_id>`
- Configure DuckDB `temp_directory` to use this folder.
- Ensure DuckDB spill files and any temporary staged Parquet/intermediate files are written only under this folder.
- Remove the temporary folder automatically after successful completion.
- Also remove it on failure using `try/finally` or a safe cleanup handler.
- Do not delete any user input files or framework output files.

6. Logging
- Create a timestamped log file under the framework `logs` folder for every DuckDB run.
- Example log filename:
  - `logs/duckdb_large_file_validation_YYYYMMDD_HHMMSS.log`
- Ensure the `logs` folder is created if it does not already exist.
- All logs from the DuckDB script must be written to this timestamped log file.
- Also print important high-level messages to console if the existing framework style does that.
- Log the following at minimum:
  - script start time and end time,
  - total runtime,
  - input file paths,
  - output folder paths,
  - DuckDB version,
  - DuckDB configuration values,
  - temp spill/staging folder path,
  - number of records processed where available,
  - validation step names,
  - output file creation,
  - warnings,
  - errors with stack trace,
  - temp folder cleanup status.
- Use Python `logging` module, not only `print`.

7. Production readiness
- Add proper error handling and meaningful log messages.
- Avoid hardcoded absolute paths.
- Use `pathlib` for path handling.
- Add docstrings for all important functions.
- Add comments explaining the DuckDB streaming design, memory limit, threading, spillover, and cleanup.
- Keep configuration values such as `threads`, `memory_limit`, logs folder, and temp folder location easy to change.
- Use clear function names and keep the script maintainable.
- Fail fast with clear errors for missing files, invalid config, unsupported file types, or missing required columns.
- Use safe SQL construction. Quote identifiers properly and avoid unsafe string concatenation where values come from config.

8. Output compatibility
- Ensure the DuckDB script produces the same validation results as the existing Polars script.
- Keep the same output columns, mismatch files, summary files, and report artifacts.
- Add a comparison test or sample execution using the existing sample file to confirm DuckDB output matches Polars output.
- If any output difference is unavoidable, document the reason clearly in code comments and logs.

9. Deliverables
- Add the new DuckDB script/module without disturbing the existing Polars implementation.
- Include a short README section or inline documentation explaining:
  - why DuckDB is used,
  - how the streaming-first SQL-based design works,
  - how disk spillover is configured,
  - where temporary files are created,
  - how cleanup works,
  - how `threads = 3` and `memory_limit = '10GB'` are applied,
  - why `preserve_insertion_order = false` is used,
  - where timestamped logs are created,
  - how to run the new DuckDB validation script.

Before coding, inspect the existing project structure, utility functions, output generation logic, logging style, and current Polars implementation. Then create the separate DuckDB script with minimum clean integration while preserving the current framework behavior and output structure.