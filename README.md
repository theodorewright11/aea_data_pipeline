# AEA Data Pipeline

> **Work in Progress** -- The pipeline runs and produces correct output, but the code is not yet modularized or user-friendly. Adding new datasets currently requires manual edits in multiple places. Cleanup to make the pipeline more configurable and easier to extend is planned.

This pipeline constructs the datasets used by the Automation Exposure Analysis (AEA) Dashboard. It takes raw data from multiple AI scoring sources (Anthropic Economic Index, MCP Server Pipeline, Microsoft Copilot), O\*NET occupational data, and BLS employment/wage statistics, then merges, normalizes, and enriches them into a unified format where each row represents one task within one occupation.

All raw input data is included in this repository except for one file that exceeds GitHub's 100MB file size limit: **`mcp_results_2026-02-18.csv`** (186MB). Download it from Hugging Face and place it in the `data/` directory before running the pipeline:

https://huggingface.co/datasets/theodorewright11/mcp_to_onet_classification

---

## Documentation

This project has detailed documentation split across three files:

- **[PRD.md](PRD.md)** -- What the pipeline produces and why. Covers data sources, output datasets, shared columns, pipeline workflow, and dataset dates.
- **[ARCHITECTURE.md](ARCHITECTURE.md)** -- How the code works. Technical reference for the three pipeline parts, all input files, processing logic, helper functions, and common pitfalls.
- **[CLAUDE.md](CLAUDE.md)** -- Agent instructions for Claude Code sessions working on this project. Contains code quality rules, guardrails, and change protocols.

---

## Project Structure

```
aea_data_pipeline/
├── CLAUDE.md                          # Agent instructions
├── PRD.md                             # What the pipeline produces
├── ARCHITECTURE.md                    # How the code works
├── requirements.txt                   # Python dependencies
├── .gitignore                         # Excludes data/ and venv/
├── scripts/
│   ├── data_merge.ipynb               # Main pipeline (3 parts)
│   ├── code_for_exploring.ipynb       # Exploratory analysis
│   └── code_storage.ipynb             # Utility code storage
└── data/                              # All CSVs (gitignored)
    ├── final/                         # Final output datasets
    └── merged_data_files/             # Optional intermediate saves
```

---

## Input Data

### AI Scoring Sources

| Source | Files |
|--------|-------|
| Anthropic Economic Index (AEI) | `task_pct_v{1..5}.csv`, `task_pct_api_v{3..5}.csv`, `automation_vs_augmentation_by_task_*.csv` |
| MCP Server Pipeline | `task_results_{date}.csv` (one per version date) |
| Microsoft Copilot | `iwa_metrics.csv` |
| ECO Baselines | `task_pct_eco_{2015,2025}.csv` |

### O\*NET Reference

| File | Description |
|------|-------------|
| `task_statements_v20.1.csv` | Task statements (Oct 2015), used by AEI |
| `task_statements_v30.1.csv` | Task statements (May 2025), used by MCP/Microsoft/ECO 2025 |
| `tasks_dwa_iwa_gwa_v{20.1,30.1}.csv` | Work activity taxonomy (DWA/IWA/GWA) |
| `task_ratings_v30.1.csv` | Frequency, importance, relevance ratings |
| `physical_tasks.csv` | Physical task boolean flags |
| `dws_ratings.csv` | Star ratings for ECO |
| `job_zones_v30.1.csv` | Job Zone classifications (1--5) for ECO 2025 |

### Economic Data

| File | Description |
|------|-------------|
| `oews_national_2024.csv`, `oews_states_2024.csv` | BLS OEWS wages & employment (May 2024) |
| `oews_national_2015.csv`, `oews_states_2015.csv` | BLS OEWS wages & employment (May 2015) |
| `soc_structure_2019.csv` | SOC hierarchy (major/minor/broad categories) |
| `2010_to_2019_soc_crosswalk.csv` | SOC code mapping between versions |
| `scraped_wage_data.csv` | O\*NET scraped wages (Jan 2020), used as fallback |

---

## Output Datasets

All final outputs are saved to `data/final/`. There are 15 per-version datasets and 10 cumulative AEI datasets (25 total). Each row is one task within one occupation.

### Per-Version (15 files)

| Dataset | File |
|---------|------|
| AEI Conv. v1--v5 | `final_aei_v{1..5}.csv` |
| AEI API v3--v5 | `final_aei_api_v{3..5}.csv` |
| MCP Cumul. v1--v4 | `final_mcp_v{1..4}.csv` |
| Microsoft | `final_microsoft.csv` |
| ECO 2025 | `final_eco_2025.csv` |
| ECO 2015 | `final_eco_2015.csv` |

### Cumulative AEI (10 files)

Cumulative datasets combine per-version AEI datasets (union of tasks, max auto\_aug, summed & re-normalized pct). See [PRD.md](PRD.md) for the full list of cumulative variants.

---

## Key Output Columns

Every final dataset includes these columns (some may be null depending on dataset type). There are additional columns not listed here -- see [PRD.md](PRD.md) for the complete specification.

| Column | Description |
|--------|-------------|
| `task` | Raw O\*NET task text |
| `task_normalized` | Normalized task text (lowercased, punctuation removed) |
| `title` | Occupation title (2010 SOC for AEI/ECO 2015) |
| `title_current` | Occupation title (2019 SOC, for MCP/Microsoft/ECO 2025) |
| `soc_code_2010` | O\*NET SOC code (2010 system) |
| `dwa_title`, `iwa_title`, `gwa_title` | Work activity hierarchy (Detailed, Intermediate, General) |
| `broad_occ`, `minor_occ_category`, `major_occ_category` | SOC occupation hierarchy |
| `pct_normalized` | Share of AI usage for this task (sums to ~100 per dataset on unique title x task pairs) |
| `auto_aug_mean` | Automatability score (0--5 scale) |
| `physical` | Boolean flag for physical tasks |
| `freq_mean` | Task frequency (daily occurrence rate from O\*NET survey) |
| `importance` | Task importance (1--5 from O\*NET survey) |
| `relevance` | Task relevance (0--100 from O\*NET survey) |
| `emp_tot_nat_2024` | National employment (BLS OEWS 2024) |
| `a_med_nat_2024` | National median annual wage (BLS OEWS 2024) |
| `job_zone` | O\*NET Job Zone (1--5), ECO 2025 only |
| `date` | Dataset snapshot date |

---

## Pipeline Workflow

The pipeline runs in three parts from a single notebook (`scripts/data_merge.ipynb`):

1. **Part 1** -- Runs once per dataset (set `run_name`). Maps raw AI data to O\*NET tasks, adds SOC structure, merges BLS wage/employment, and adjusts employment for decimal SOC codes.
2. **Part 2** -- Runs once across all datasets. Adds snapshot dates, O\*NET taxonomy (DWA/IWA/GWA), physical task flags, and standardized auto\_aug scores.
3. **Part 3** -- Runs once across all datasets. Merges task ratings, add Utah DWS outlook numbers, adds Job Zones (ECO 2025), builds cumulative AEI datasets, and does final column reordering.
