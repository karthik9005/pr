Read the full requirements from `prompt.md` in the repository root and follow them exactly.

Before making any code changes:
1. Inspect the existing project structure.
2. Identify the current Polars implementation.
3. Identify reusable utility functions, logging patterns, config handling, and output generation logic.
4. Provide a short implementation plan.

After the plan, implement the DuckDB solution as instructed in `prompt.md`.

Important:
- Do not modify or remove the existing Polars implementation.
- Create a new DuckDB-based script/module.
- Preserve the existing output files, folder structure, reports, and logs.
- Use modern DuckDB streaming-first processing.
- Use DuckDB disk spillover with `threads = 3`, `memory_limit = '10GB'`, and a temporary spill folder under the framework.
- Create a timestamped log file under the `logs` folder.
- Clean up temporary spill/staging files after completion or failure.
- Keep the implementation production-ready, efficient, documented, and compatible with the existing framework.