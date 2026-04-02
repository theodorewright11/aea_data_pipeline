# CLAUDE.md -- Agent Instructions

Rules and patterns for any Claude Code session working on this project.

---

## Before You Start

- Read `PRD.md` to understand what the pipeline produces and why.
- Read `ARCHITECTURE.md` to understand how the code is structured.
- If your task touches data processing logic, read the relevant section of ARCHITECTURE.md carefully before writing code.
- This rule if very important: If ambiguous about where a change should go or what it should do, ask before implementing.

---

## After Every Change

- If you changed processing logic, function signatures, file I/O, column names, pipeline stages, etc.: update ARCHITECTURE.md.
- If you changed what the pipeline produces, what inputs it uses, or added/removed a dataset version: update PRD.md.
- If you discovered a new pitfall or gotcha during implementation: add it to the Common Pitfalls section of ARCHITECTURE.md.

---

## Code Quality Rules

### Python (Notebooks)
- All processing functions should have clear parameter names and return types in docstrings or comments.
- Validate DataFrame shapes and expected columns at function entry points -- don't let missing columns propagate silently.
- Use `normalize_text()` consistently for all text matching. Never compare raw task strings directly.

### General
- Prefer small, targeted edits over full cell rewrites.
- Be meticulous about making sure we have the correct merge keys and that don't lose data on different merging and data imputation. 

---

## Project Structure

```
aea_data_pipeline/
├── CLAUDE.md              # This file
├── PRD.md                 # What the pipeline produces
├── ARCHITECTURE.md        # How the code works
├── requirements.txt       # Python dependencies
├── .gitignore             # Excludes data/ and venv/
├── scripts/
│   ├── data_merge.ipynb           # Main pipeline
│   ├── code_for_exploring.ipynb   # Exploratory analysis
│   └── code_storage.ipynb         # Utility code storage
└── data/                  # All CSVs (gitignored)
    ├── final/             # Final output datasets
    └── merged_data_files/ # Optional intermediate saves
```

---

## Guardrails

These are common ways this pipeline can break. Check these when your change touches the relevant area:

- **Text normalization**: All task and title matching must go through `normalize_text()`. Raw string comparisons will silently miss matches.
- **SOC code versions**: AEI and ECO 2015 use 2010 SOC codes. MCP, Microsoft, and ECO 2025 use 2019 SOC codes. Never merge across SOC versions without the crosswalk.
- **pct_normalized**: Must sum to 100 across all rows for a given dataset based on unique task and occupation pairs. After any filtering or combining, check this.
- **Employment double-counting**: O*NET decimal SOC codes (e.g., 11-1011.00 vs 11-1011.03) share the same BLS employment number. The `adjust_emp` functions handle this.
- **State columns**: The pipeline processes all 54 US states/territories, not just Utah. State column names follow the pattern `emp_{abbrev}`, `a_median_{abbrev}`, `h_median_{abbrev}`.
- **Cumulative datasets**: Use `build_cumulative()` for combining AEI versions. auto_aug takes max, pct_normalized sums then re-normalizes, date takes latest.
- **Run order**: Part 1 must be run once per dataset individually. Parts 2 and 3 process all datasets together in a single execution.
