# PRD -- AEA Data Pipeline

What this pipeline produces and why.

---

## 1. Overview

This pipeline constructs the datasets used by the Automation Exposure Analysis (AEA) Dashboard. It takes raw data from multiple AI scoring sources, O*NET occupational data, and BLS employment/wage statistics, then merges, normalizes, and enriches them into a unified format where each row represents one task within one occupation.

The final CSVs are consumed by the dashboard backend (`aea_dashboard` repo). This repo is the reference for anyone wanting to understand how the data was built, reproduce it, or extend it with new dataset versions.

---

## 2. Data Sources

### AI Scoring Sources (Inputs)

**Anthropic Economic Index (AEI)** -- Derived from real Claude conversation data. Each O*NET task gets a `pct_normalized` (share of conversations) and `auto_aug_mean` (automatability score, 0--5). Uses O*NET v20.1 (2015) task statements and 2010 SOC codes.

- AEI Conv. v1--v5: Five snapshot versions (Dec 2024 -- present), conversation data only.
- AEI API v3--v5: Three snapshot versions, API/tool-use interactions only.
- Input files: `task_pct_v{1..5}.csv`, `task_pct_api_v{3..5}.csv`
- Auto/aug scores: `automation_vs_augmentation_by_task_v{2..5}.csv`, `automation_vs_augmentation_by_task_api_v{3..5}.csv`

**MCP Server Pipeline** -- AI task classifications from Model Context Protocol server logs. Four cumulative versions (Apr 2025 -- Feb 2026). Uses O*NET v30.1 (2025) task statements and 2019 SOC codes.

- Input files: `task_results_{date}.csv` (one per version date)

**Microsoft Copilot Analysis** -- Assessment of AI automation/augmentation from Copilot usage data (Sep 2024). Uses IWA-level metrics mapped to tasks. Also includes physical task flags.

- Input file: `iwa_metrics.csv`

### Structural & Economic Sources (Inputs)

**O*NET** -- Task statements (v20.1 for AEI, v30.1 for MCP/Microsoft/ECO), task ratings (frequency, importance, relevance), work activity taxonomy (DWA/IWA/GWA).

**BLS OEWS** -- Employment counts and median wages by occupation, national and all 54 US states/territories, for both 2024 and 2015.

**SOC Crosswalk** -- Maps 2010 SOC codes to 2019 SOC codes for AEI data compatibility.

---

## 3. Output Datasets

All final outputs are saved to `data/final/`. Each row is one task within one occupation.

### Per-Version Datasets (15 files)

| Dataset | File | SOC Version |
|---------|------|-------------|
| AEI Conv. v1--v5 | `final_aei_v{1..5}.csv` | 2010 |
| AEI API v3--v5 | `final_aei_api_v{3..5}.csv` | 2010 |
| MCP Cumul. v1--v4 | `final_mcp_v{1..4}.csv` | 2019 |
| Microsoft | `final_microsoft.csv` | 2019 |
| ECO 2025 | `final_eco_2025.csv` | 2019 |
| ECO 2015 | `final_eco_2015.csv` | 2010 |

### Cumulative Datasets (42 files)

Built by combining per-version datasets across sources. For each bucket, a new cumulative version is produced each time a new dataset arrives chronologically. Combining logic: union of task-occupation pairs, max `auto_aug_mean`, max `pct_normalized`.

Output naming: `final_{bucket_name}_{end_date}.csv`

**Usage buckets:**

| Bucket | Description | Sources | Task Set | Versions |
|--------|-------------|---------|----------|----------|
| `all_confirmed_usage` | All confirmed usage | AEI Both + Microsoft | 2025 | 6 (2024-09-30 to 2026-02-12) |
| `confirmed_human_usage` | Confirmed human usage | AEI Conv + Microsoft | 2025 | 6 (2024-09-30 to 2026-02-12) |
| `aei_all_usage` | AEI all confirmed usage | AEI Conv + AEI API | 2015 | 5 (2024-12-23 to 2026-02-12) |
| `aei_human_usage` | AEI confirmed human usage | AEI Conv only | 2015 | 5 (2024-12-23 to 2026-02-12) |
| `aei_agentic_usage` | AEI confirmed agentic usage | AEI API only | 2015 | 3 (2025-08-11 to 2026-02-12) |

**Agentic bucket:**

| Bucket | Description | Sources | Task Set | Versions |
|--------|-------------|---------|----------|----------|
| `all_agentic_usage` | All possible agentic usage | MCP + AEI API | 2025 | 7 (2025-04-24 to 2026-02-18) |

**All bucket:**

| Bucket | Description | Sources | Task Set | Versions |
|--------|-------------|---------|----------|----------|
| `all_usage` | All usage potential | AEI Both + MCP + Microsoft | 2025 | 10 (2024-09-30 to 2026-02-18) |

2025 task set buckets use ECO 2025 as structural backbone. AEI/API sources are filtered to task-occupation pairs that exist in ECO 2025 before combining. 2015 task set buckets use native AEI row structure.

### ECO Baseline Datasets (2 files)

Economy-wide task baseline (denominator for exposure calculations). `pct_normalized` and `auto_aug_mean` are all zero/null -- values come from AI datasets only.

- `final_eco_2025.csv` -- Primary baseline for MCP/Microsoft (2019 SOC).
- `final_eco_2015.csv` -- Baseline for AEI work-activity analysis (2010 SOC).

---

## 4. Shared Output Columns

Every final dataset includes these columns (some may be null depending on dataset type):

| Column | Description |
|--------|-------------|
| `task`, `task_normalized` | Raw and normalized O*NET task text |
| `title` | Occupation title (2010 SOC for AEI/ECO 2015) |
| `title_current` | Occupation title (2019 SOC, MCP/Microsoft/ECO 2025 only) |
| `soc_code_2010` | O*NET SOC code (2010 system) |
| `dwa_title`, `iwa_title`, `gwa_title` | Work activity hierarchy |
| `broad_occ`, `minor_occ_category`, `major_occ_category` | SOC occupation hierarchy |
| `pct_normalized` | Share of AI usage for this task (sums to ~100 per dataset on unique title x task pairs) |
| `auto_aug_mean` | Automatability score (0--5) |
| `physical` | Boolean: is this a physical task |
| `freq_mean` | Task frequency (daily occurrence rate from O*NET survey) |
| `importance` | Task importance (1--5 from O*NET survey) |
| `relevance` | Task relevance (0--100 from O*NET survey) |
| `emp_tot_nat_2024` | National employment (BLS OEWS 2024) |
| `a_med_nat_2024` | National median annual wage (BLS OEWS 2024) |
| `job_zone` | O*NET Job Zone (1--5), ECO 2025 only |
| `date` | Dataset snapshot date |

Plus state-level wage and employment columns for all 54 US states/territories (`emp_{abbrev}`, `a_median_{abbrev}`, `h_median_{abbrev}` for both 2024 and 2015), and 2015 wage/employment columns (nominal and inflation-adjusted).

---

## 5. Pipeline Workflow

The pipeline runs in three parts from a single notebook (`scripts/data_merge.ipynb`).

**Part 1** runs once per dataset (set `run_name` and re-execute). It maps raw AI data to O*NET tasks, adds SOC structure, merges BLS wage/employment for 2024 and 2015, adjusts employment for SOC decimal codes, and does initial cleanup. Output: `first_pass_*.csv`.

**Part 2** runs once across all datasets. It adds snapshot dates, O*NET taxonomy (DWA/IWA/GWA), physical task flags, and standardized auto_aug scores. Output: `second_pass_*.csv`.

**Part 3** runs once across all datasets. It merges task ratings (frequency, importance, relevance), adds task_prop for ECO 2025, builds cumulative AEI datasets, adds DWS ratings, and does final column reordering. Output: `third_pass_*.csv` then `final_*.csv`.

---

## 6. Dataset Dates

| Dataset | Snapshot Date |
|---------|--------------|
| AEI Conv. v1 | 2024-12-23 |
| AEI Conv. v2 | 2025-03-06 |
| AEI Conv. v3 | 2025-08-11 |
| AEI Conv. v4 | 2025-11-13 |
| AEI Conv. v5 | 2026-02-12 |
| AEI API v3 | 2025-08-11 |
| AEI API v4 | 2025-11-13 |
| AEI API v5 | 2026-02-12 |
| MCP Cumul. v1 | 2025-04-24 |
| MCP Cumul. v2 | 2025-05-24 |
| MCP Cumul. v3 | 2025-07-23 |
| MCP Cumul. v4 | 2026-02-18 |
| Microsoft | 2024-09-30 |

---

## 7. Other Scripts

### `code_for_exploring.ipynb`

Exploratory analysis notebook (not part of the pipeline). Contains data quality checks, rating distribution analysis, MCP physical task bias investigation, AEI-to-ECO alignment analysis, and state imputation diagnostics. For internal use.

### `code_storage.ipynb`

Utility code storage (not part of the pipeline). Contains one-off data transformation scripts: physical metric extraction, AEI collaboration pattern processing, eco percentage creation, and master title normalization. For internal use.
