# ARCHITECTURE.md -- AEA Data Pipeline

Technical reference for how the pipeline works. Read this before making changes.
Paired with [PRD.md](PRD.md) (what it produces) and [CLAUDE.md](CLAUDE.md) (agent rules).

---

## 1. Repository Structure

```
aea_data_pipeline/
├── scripts/
│   ├── data_merge.ipynb           # Main pipeline (108 cells, 3 parts)
│   ├── code_for_exploring.ipynb   # Exploratory analysis (32 cells)
│   └── code_storage.ipynb         # Utility code storage (12 cells)
├── data/
│   ├── (root)                     # Raw inputs + intermediate files
│   ├── final/                     # Final output CSVs (25 files)
│   └── merged_data_files/         # Optional intermediate saves
├── CLAUDE.md
├── PRD.md
├── ARCHITECTURE.md
├── requirements.txt               # pandas, numpy, matplotlib, seaborn, scipy, langdetect
└── .gitignore                     # Excludes data/ and venv/
```

---

## 2. Pipeline Overview

The main pipeline lives in `scripts/data_merge.ipynb`. It has three parts:

| Part | Scope | Input | Output | Run Pattern |
|------|-------|-------|--------|-------------|
| Part 1 | Pcts, wages, emp | Raw CSVs | `first_pass_*.csv` | Once per dataset (set `run_name`) |
| Part 2 | Dates, taxonomy, auto/aug | `first_pass_*.csv` | `second_pass_*.csv` | Once for all datasets |
| Part 3 | Ratings, cumulative, reorder | `second_pass_*.csv` | `final_*.csv` | Once for all datasets |

---

## 3. Configuration

### Run Names

The `runs` dictionary maps output filenames to their raw input config. Valid `run_name` values:

```
first_pass_aei_v1.csv          first_pass_aei_api_v3.csv
first_pass_aei_v2.csv          first_pass_aei_api_v4.csv
first_pass_aei_v3.csv          first_pass_aei_api_v5.csv
first_pass_aei_v4.csv          first_pass_mcp_v1.csv
first_pass_aei_v5.csv          first_pass_mcp_v2.csv
first_pass_microsoft.csv       first_pass_mcp_v3.csv
first_pass_eco_tasks_2015.csv  first_pass_mcp_v4.csv
first_pass_eco_tasks_2025.csv
```

Dataset type is auto-detected from the run name:
- `aei_run`: `"aei" in run_name`
- `mcp_run`: `"mcp" in run_name`
- `microsoft_run`: `"microsoft" in run_name`
- `eco_run_2015` / `eco_run_2025`: exact match
- `aei_v3_and_up`: AEI runs with v3, v4, or v5 (have auto/aug data from Anthropic)

### Constants

```python
frequency_weights = {1: 1/260, 2: 3/260, 3: 48/260, 4: 130/260, 5: 1, 6: 3, 7: 12}
jan_2020_inflation_factor = 1.24   # scraped wages to present value
may_2015_inflation_factor = 1.36   # 2015 BLS wages to present value
```

### States

All 54 US states/territories are processed (not just Utah):
`al, ak, az, ar, ca, co, ct, de, dc, fl, ga, hi, id, il, in, ia, ks, ky, la, me, md, ma, mi, mn, ms, mo, mt, ne, nv, nh, nj, nm, ny, nc, nd, oh, ok, or, pa, ri, sc, sd, tn, tx, ut, vt, va, wa, wv, wi, wy, gu, pr, vi`

---

## 4. Part 1: Core Processing (Pcts, wages, emp)

### Helpers

`normalize_text(text)` -- Lowercases, removes punctuation, collapses whitespace. Used for all task and title matching throughout the pipeline.

### Step 1: Map Raw Data to O*NET Tasks

Four separate code paths depending on dataset type:

**AEI path** (`pct_to_onet_tasks()`):
- Loads `task_pct_v*.csv` (Anthropic's task percentage mappings)
- For v1--v2: merges with O*NET v20.1 task statements on normalized task text
- For v3+: filters raw AEI data for `GLOBAL/onet_task` entries, extracts pct values
- Counts `n_occurrences` per task as a total of duplicate task names that map to different occupations, divides pct by occurrence count for `pct_normalized`
- Normalizes so all rows sum to 100

**MCP path**:
- Loads `task_results_{date}.csv`
- Winsorizes ratings at 75th percentile + 1.5*IQR
- Maps to O*NET v30.1 task statements
- Crosswalks SOC codes to 2019 where needed

**Microsoft path**:
- Loads `iwa_metrics.csv` (IWA-level share_user, share_ai metrics)
- Distributes IWA weights across task-occupation pairs
- Maps to O*NET task statements
- Normalizes globally to sum to 100

**ECO path**:
- Loads pre-built `task_pct_eco_{2015,2025}.csv`
- Maps to appropriate O*NET task statements (v20.1 for 2015, v30.1 for 2025)
- Crosswalks SOC codes for 2025 version

### Step 2: Add SOC Structure

`add_soc_structure()` merges occupation hierarchy from `soc_structure_2019.csv`:
- `major_occ_category` (23 categories, matched on first 2 digits of SOC)
- `minor_occ_category`
- `broad_occ`
- `broad_counts` (number of detailed occupations per broad category)

Hard-coded SOC anomaly fixes:
- 15-1200 -> 15-1000
- 31-1100 -> 31-1000
- 51-5100 -> 51-5000
- 29-122x/123x -> 29-1210

### Step 3: Add 2024 Wage & Employment

Six sub-functions, run sequentially:

1. `add_updated_soc_code()` -- Maps titles to 2019 SOC codes via crosswalk (needed for BLS merge)
2. `add_nat_wage_2024()` -- National wages from OEWS 2024. Fallback chain: detailed SOC -> broad SOC -> scraped O*NET wages (with 1.24x inflation) -> hourly/annual conversion
3. `add_state_wage_2024()` -- State wages from OEWS 2024. Fallback: hourly/annual conversion -> national wages
4. `add_nat_emp_2024()` -- National employment from OEWS 2024. Fallback: broad category divided by `broad_counts`
5. `add_state_emp_2024()` -- State employment from OEWS 2024. Fallback: national employment * (state total / national total)
6. `wage_emp_to_tasks_2024()` -- Merges all wage/emp columns into the task-level DataFrame

For titles that map to multiple 2019 SOC codes: wages are averaged, employment is divided by duplicate count then summed.

### Step 4: Add 2015 Wage & Employment

Same structure as Step 3 but with 2015 BLS data and 1.36x inflation factor. No crosswalk needed (merges on 2010 SOC directly). Produces both nominal and real (inflation-adjusted) wage columns.

### Step 5: Adjust Employment for Decimal SOC Codes

O*NET uses decimal SOC codes (e.g., 11-1011.00 vs 11-1011.03) but BLS only has 6-digit codes. Without adjustment, employment would be counted multiple times for occupations with multiple decimal variants.

Two functions that share the same logic but key on different columns:
- `adjust_emp_old()` -- For AEI/ECO: keys on `soc_code_2010` and `title`
- `adjust_emp_new()` -- For MCP/Microsoft: keys on `soc_code_2019_full` and `title_current`

Both work the same way:
1. Group rows by 6-digit SOC code (trim the decimal)
2. Use `master_pct_normalized` as the proportioning key, with **square root dampening** to reduce the influence of extreme values
3. For each SOC6 group where multiple titles share the same employment number: split the total proportionally by each title's dampened pct share
4. If pct data is all zeros for a group, fall back to equal split across titles

### Step 6: Final Cleanup

`final_cleanup()`:
- Data validation (column presence, missing values)
- Missing value imputation for task_type (Core if relevance >= 67 and importance >= 3, else Supplemental)
- Column ordering
- Saves `first_pass_*.csv`

---

## 5. Part 2: Taxonomy, Dates, Auto/Aug

Runs once across all datasets (loads all `first_pass_*.csv` files).

### Dates & Taxonomy

- Maps each dataset to its snapshot date
- Merges O*NET DWA/IWA/GWA taxonomy from `tasks_dwa_iwa_gwa_v{20.1,30.1}.csv`
- Merges physical task flags from `physical_tasks.csv` (sourced from Microsoft's analysis, with DWA majority-rule imputation for unmapped tasks)

### Auto/Aug Standardization

Different processing per source:

**Microsoft**: The source data (`iwa_metrics.csv`) has `impact_scope_ai` and `impact_scope_user` at the IWA level, plus `share_ai` and `share_user`. These are combined into a weighted `impact_scope_avg`. That value is then mapped through MCP conditional distributions (binned by scope) to produce a comparable `auto_aug_mean` on the 1--5 scale. Can optionally compute extra statistics (max, min, p75, p25).

**AEI v2+**: Loads `automation_vs_augmentation_by_task_*.csv` from Anthropic's data release (v2--v5 conversation, api_v3--api_v5). Standardizes across versions into a single `auto_aug_mean` column.

**AEI v1**: No auto/aug data available (column stays null).

**MCP**: Already has `auto_aug_mean` and `auto_aug_mean_adj` (adjusted, preferred) from the classification pipeline.

Output: `second_pass_*.csv`

---

## 6. Part 3: Ratings, Cumulative, Final

Runs once across all datasets.

### Task Ratings

- Loads O*NET task ratings (`task_ratings_v30.1.csv`, `task_ratings_may_2025.csv`, `task_ratings_oct_2015.csv`)
- Applies frequency weights to convert survey responses to daily frequency values
- Merges `freq_mean`, `importance`, `relevance` into each dataset
- Imputes missing ratings with an 8-level fallback chain (min 5 values at each level to use it):
  1. (title, task) direct merge
  2. Task-only average (across all occupations sharing that task)
  3. Occupation-level average
  4. DWA (Detailed Work Activity) average
  5. IWA (Intermediate Work Activity) average
  6. GWA (General Work Activity) average
  7. Major occupation category average
  8. Global mean
- **Penalty**: if an occupation has >50% of its tasks imputed below occupation level, those imputed `freq_mean` values are halved
- For ECO 2025 only: computes `task_prop = count_2015_tasks / count_2025_tasks` per occupation (2015 tasks mapped to 2025 occupations via SOC crosswalk). Used by the dashboard backend as a deflation factor when crosswalking AEI data.

### Cumulative Datasets

Seven buckets of cumulative datasets are built, each with multiple time-point versions (one per new dataset arrival). Output naming: `final_{bucket_name}_{end_date}.csv`.

**Buckets:**

| Bucket | Sources | Task Set | Versions |
|--------|---------|----------|----------|
| `all_confirmed_usage` | AEI Both + Microsoft | 2025 | 6 |
| `confirmed_human_usage` | AEI Conv + Microsoft | 2025 | 6 |
| `aei_all_usage` | AEI Conv + AEI API | 2015 | 5 |
| `aei_human_usage` | AEI Conv only | 2015 | 5 |
| `aei_agentic_usage` | AEI API only | 2015 | 3 |
| `all_agentic_usage` | MCP + AEI API | 2025 | 7 |
| `all_usage` | AEI Both + MCP + Microsoft | 2025 | 10 |

**2015 task set builds** (`build_cumulative_2015`):
- Merge key: `["title", "task_normalized", "dwa_title", "iwa_title", "gwa_title"]`
- For matched rows: `auto_aug_mean` takes **max**, `pct_normalized` takes **max** (highest observed usage share), `date` takes **latest**, other columns from latest version
- No pct renormalization — values preserve the scale of whichever source had the higher value

**2025 task set builds** (`build_cumulative_2025`):
- Extracts scores per unique `(title_current, task_normalized)` from each source
- AEI/API sources match their `title` (2010 SOC) against ECO 2025's `title_current` (2019 SOC); MCP/Microsoft match `title_current` directly (~74% of AEI pairs match)
- Combines scores: **max** auto_aug, **max** pct (no renormalization), **latest** date
- Joins combined scores to ECO 2025 backbone on `(title_current, task_normalized)` for structural columns (DWA/IWA/GWA, SOC 2019 codes, etc.)

Total output: 42 cumulative CSV files across all buckets.

### Job Zones

Merges O*NET Job Zone data (`job_zones_v30.1.csv`, sourced from `Job Zones.xlsx`) into ECO 2025 only. Job Zones classify occupations into one of five zones (1--5) based on education, experience, and training requirements.

- Normalizes `title_current` for matching against Job Zone titles
- Deduplicates occupations before merging to get one `job_zone` value per occupation
- Merges back into the full ECO 2025 task-level DataFrame on `title_current`
- `job_zone` is stored as `Int64` (nullable integer)

### DWS Ratings

Merges `dws_ratings.csv` into ECO 2025 (star ratings for tasks).

### Column Reorder

Final column ordering applied. Dataset-specific columns handled

Output: `third_pass_*.csv` -> copied to `final/final_*.csv`

---

## 7. Input Data Files

### Raw AI Data

| File Pattern | Source | Notes |
|-------------|--------|-------|
| `task_pct_v{1..5}.csv` | AEI conversation snapshots | v1--v2 use original format, v3+ use GLOBAL/onet_task format |
| `task_pct_api_v{3..5}.csv` | AEI API snapshots | Same format as v3+ conversation |
| `task_pct_eco_{2015,2025}.csv` | ECO baselines | Pre-built task percentages (all zeros for pct/auto_aug) |
| `task_results_{date}.csv` | MCP server pipeline | Dates: 2025-04-24, 2025-05-24, 2025-07-23, 2026-02-18 |
| `iwa_metrics.csv` | Microsoft Copilot analysis | IWA-level metrics |
| `automation_vs_augmentation_by_task_*.csv` | AEI auto/aug scores | Separate files for v2, v3, v4, v5, api_v3, api_v4, api_v5 |

### O*NET Reference

| File | Version | Used By |
|------|---------|---------|
| `task_statements_v20.1.csv` | Oct 2015 | AEI v1--v2, ECO 2015 |
| `task_statements_v30.1.csv` | May 2025 | MCP, Microsoft, ECO 2025 |
| `tasks_dwa_iwa_gwa_v20.1.csv` | Oct 2015 | AEI taxonomy |
| `tasks_dwa_iwa_gwa_v30.1.csv` | May 2025 | MCP/Microsoft/ECO taxonomy |
| `tasks_dwa_iwa_gwa_v30.1_physical.csv` | May 2025 | With physical flags merged |
| `task_ratings_v30.1.csv` | May 2025 | Frequency/importance/relevance |
| `physical_tasks.csv` | Microsoft | Physical task boolean flags |
| `dws_ratings.csv` | O*NET | Star ratings for ECO |
| `job_zones_v30.1.csv` | O*NET | Job Zone classifications (1--5) for ECO 2025 |

### Economic Data

| File | Description |
|------|-------------|
| `oews_national_2024.csv` | BLS OEWS national wages & employment, May 2024 |
| `oews_states_2024.csv` | BLS OEWS state-level wages & employment, May 2024 |
| `oews_national_2015.csv` | BLS OEWS national wages & employment, May 2015 |
| `oews_states_2015.csv` | BLS OEWS state-level wages & employment, May 2015 |
| `scraped_wage_data.csv` | O*NET scraped wages (Jan 2020), used as fallback |
| `soc_structure_2019.csv` | SOC hierarchy (major/minor/broad categories) |
| `2010_to_2019_soc_crosswalk.csv` | SOC code mapping between versions |
| `detailed_occ_2010.csv` | Detailed occupation listings (2010 SOC) |
| `detailed_occ_2019.csv` | Detailed occupation listings (2019 SOC) |

---

## 8. Intermediate File Stages

| Stage | Pattern | When Created |
|-------|---------|-------------|
| First pass | `first_pass_*.csv` | After Part 1 (Pct, emp, wage) |
| Second pass | `second_pass_*.csv` | After Part 2 (taxonomy + auto/aug) |
| Third pass | `third_pass_*.csv` | After Part 3 (ratings) |
| Final | `final/final_*.csv` | After cumulative build, dws ratings, column reorder (15 per-version + 10 cumulative) |

All intermediate files are saved to `data/` root. Final files go to `data/final/`.

---

## 9. Secondary Scripts

### `code_for_exploring.ipynb`

Exploratory analysis notebook. Not part of the pipeline. Key analyses:

- MCP physical task bias investigation (why some MCPs rate physical tasks highly)
- Rating distribution analysis across datasets
- AEI-to-ECO 2025 alignment analysis (task coverage across versions)
- State imputation diagnostics
- DWS ratings imputation numbers
- Frequency/importance/relevance correlation (Gini coefficient)

### `code_storage.ipynb`

Utility code storage. Not part of the pipeline. Contains one-off scripts:

- Physical metric extraction: merges Microsoft physical flags into O*NET taxonomy file
- Collaboration pattern extraction: pivots AEI raw data into collaboration type columns (directive, feedback loop, learning, task iteration, validation)
- ECO percentage creation: builds `task_pct_eco_{2015,2025}.csv` from O*NET task statements
- Master title normalization: aggregates pct_normalized across AEI versions into `master_pct_normalized.csv`

---

## 10. Common Pitfalls

1. **O*NET version mismatch.** AEI uses v20.1 task statements (2015). MCP/Microsoft/ECO 2025 use v30.1 (2025). Tasks that exist in one version may not exist in the other. The pipeline handles this by using the correct version per dataset type.

2. **SOC code anomalies.** Several SOC codes don't follow standard hierarchy rules. Hard-coded fixes in `add_soc_structure()` handle these. If adding new data, check that SOC codes resolve correctly.

3. **Employment reallocation.** BLS reports employment at the 6-digit SOC level, but O*NET splits some occupations into decimal variants (e.g., 11-1011.00, 11-1011.03). Without the `adjust_emp` step, employment would be double/triple counted.

4. **AEI v3+ format change.** AEI v1--v2 provide task percentages directly. v3+ provide raw conversation data that needs filtering for `GLOBAL/onet_task` entries before percentage extraction. The `pct_to_onet_tasks()` function handles both formats.

5. **Cumulative pct re-normalization.** When building cumulative AEI datasets, pct_normalized values are summed across versions then divided by the new total of unique task x title pairs and scaled back. This preserves relative proportions while accounting for newly added tasks.

6. **State wage fallback chain.** State OEWS data has no broad O-GROUP category, so the fallback chain is shorter than national: hourly/annual conversion -> national wages. Don't add a broad-category fallback for states.

7. **Inflation factors are hard-coded.** The 1.24x (Jan 2020) and 1.36x (May 2015) factors need manual updates if the base year changes.

8. **MCP rating winsorization.** MCP ratings are winsorized at 75th percentile + 1.5*IQR before mapping. This reduces the impact of extreme outlier ratings from the LLM classification pipeline.
