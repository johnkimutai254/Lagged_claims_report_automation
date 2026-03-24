# Lagged_claims_report_automation
This document summarizes updates to **SCRAPPY** (`project_scrappy`) that automate quarterly **claims-lag** (abbreviated) reports: no manual template ID swaps, and optional **batch** runs that pull aggregation IDs from `queries/claims/claims_dependent_aggregation_ids.sql`.


---

## Table of contents

1. [High-level summary](#changelog-toc-1)
2. [Part A — `--lagged-claims` flag (template, slides, charts, QC)](#changelog-toc-2)
3. [Part B — Batch mode wired to claims-dependent aggregation query](#changelog-toc-3)
4. [How to run](#changelog-toc-4)
5. [Files touched](#changelog-toc-5)

---

<a id="changelog-toc-1"></a>

## 1. High-level summary

| Before | After |
|--------|--------|
| Analysts manually changed the quarterly Google Slides **template ID** in `generate_reports.py` and often renamed slide dicts / YAML keys between full QBR and abbreviated claims-lag runs. | **`--lagged-claims`** sets `LAGGED_CLAIMS_RUN=true`; code picks **abbreviated vs full** template, **slide deletion map**, and **chart keys** automatically. Chart keys: **`Quarterly`** (full, default) and **`Quarterly_Claims_Lagged`** (abbreviated) — automated cloud runs keep using `Quarterly`. |
| Batch quarterly lag runs required manually pasting aggregation IDs from `claims_dependent_aggregation_ids.sql`. | **Batch + quarterly + `--lagged-claims`** runs that SQL first and **replaces `report_specs`** with the returned `aggregation_id` list before `prep_reports_df`. |

---

<a id="changelog-toc-2"></a>

## 2. Part A — `--lagged-claims` flag (template, slides, charts, QC)

### 2.1 `run_reports.py`

**Purpose:** Expose a CLI flag and pass it into the subprocess environment.

**Previous (conceptual):** No `--lagged-claims`; no `LAGGED_CLAIMS_RUN` env var.

**Current:**

```python
# Quarterly variant: abbreviated (claims lag) vs full QBR template
parser.add_argument("--lagged-claims", action="store_true",
                   help="Use abbreviated quarterly template for claims-lag slides (run with --quarterly)")
```

```python
env["LAGGED_CLAIMS_RUN"] = str(getattr(args, "lagged_claims", False)).lower()
```

```python
if args.quarterly:
    print(f"  Lagged claims (abbreviated template): {getattr(args, 'lagged_claims', False)}")
```

---

### 2.2 `analyst_selections.py`

**Purpose:** Read the env var and enforce it is only used with quarterly.

**Previous (conceptual):** No `lagged_claims` variable.

**Current:**

```python
# Abbreviated quarterly template for claims-lag reports (use with --quarterly --lagged-claims)
lagged_claims = os.getenv('LAGGED_CLAIMS_RUN', 'false').lower() == 'true'

if lagged_claims and not quarterly:
    raise Exception("ANALYST_SELECTIONS ERROR: --lagged-claims can only be used with quarterly reports")
```

---

### 2.3 `generate_reports.py` — `get_workspace_config()`

**Purpose:** Choose **full** vs **abbreviated** quarterly Google Slides template ID from constants.

**Previous (conceptual):** Single `quarterly_template_id` for full QBR; lag runs required manually swapping the ID. A later interim used `quarterly_template_full_id` + a computed `quarterly_template_id`; **current** uses **`quarterly_template_id`** as the full QBR constant and uses **`quarterly_template_id_effective`** for the ID passed to the Slides API.

**Current:**

```python
# `quarterly_template_id` matches the original script name (standard full QBR for automated runs).
# `quarterly_template_id_effective` is what the API uses (abbreviated when --lagged-claims).
quarterly_template_id = '1V3ZJpaSw92NycKlKMPdl1R8XeVh82Otb-9nB2FSDzX8'  
quarterly_template_abbreviated_id = '1bNmy7aJr3oPnB6ABojA29BDrr1KYD9eWeErHDrEfQFQ'  # abbreviated template for claims lag slides
quarterly_template_id_effective = (
    quarterly_template_abbreviated_id if getattr(ans, 'lagged_claims', False)
    else quarterly_template_id
)
# template_ids['quarterly'] = quarterly_template_id_effective
```

**Template copy log line:**

```python
if ans.quarterly:
    PRESENTATION_ID = config['template_ids']['quarterly']
    print('** QBR slides based on this template' + (' (abbreviated claims lag)' if ans.lagged_claims else ''))
```

---

### 2.4 `generate_reports.py` — slide deletion map

**Purpose:** Match slide `objectId` filtering to the deck actually copied (abbreviated vs full).

**Previous (conceptual):** Often `quarterly_slide_template_dict` was manually swapped with the abbreviated dict (or vice versa).

**Current:**

```python
elif ans.quarterly:
    slide_template_dict = (
        ref.quarterly_claims_lag_slide_template_dict if ans.lagged_claims
        else ref.quarterly_slide_template_dict
    )
```

---

### 2.5 `reference.py`

**Purpose:** Stable names: abbreviated dict vs full QBR dict.

**Previous (conceptual):** For abbreviated QBR lag template, it required renaming `quarterly_claims_lag_slide_template_dict` to `quarterly_slide_template_dict` and the full deck to `quarterly_slide_template_dict_xx`.

**Current:**

- **`quarterly_claims_lag_slide_template_dict`** — abbreviated claims-lag template slide IDs + service flags.
- **`quarterly_slide_template_dict`** — full QBR template.

```python
# Abbreviated QBR claims lag template (use with --quarterly --lagged-claims)
quarterly_claims_lag_slide_template_dict = {
    'g2f6c70f8821_0_3955': ['aic_base', 'ic_base', 'nav_base', 'cemo_base'],
    # ... abbreviated template slides ...
}

# Full QBR template (default quarterly)
quarterly_slide_template_dict = {
    'g2f6c70f8821_0_3949': ['aic_base', 'ic_base', 'nav_base', 'cemo_base'],
    # ... full template slides ...
}
```

---

### 2.6 `report_keys.yaml`

**Purpose:** Separate chart placement keys for abbreviated vs full quarterly decks.

**Previous (conceptual):** It involved renaming `Quarterly_Claims_Lag` (line 7) to Quarterly and change name of Quarterly (line 39) to whatever you want.

**Current (stable names — automated cloud runs unchanged):**

- **`Quarterly`** — chart positions for the **full** QBR template. Used for default quarterly runs and **automated cloud/cron runs** (when `LAGGED_CLAIMS_RUN` is not set). No code changes needed for existing quarterly automation.
- **`Quarterly_Claims_Lagged`** — chart positions for the **abbreviated** claims-lag template. Used only when `--lagged-claims` is set.

```yaml
Charts:
  Quarterly:  # standard full QBR template (default quarterly / automated cloud runs)
      age_dist_employees:
          slide_index: 148
          # ...
  Quarterly_Claims_Lagged:  # abbreviated claims-lag template (--quarterly --lagged-claims)
      high_opp_clinical_services_chart:
          slide_index: 3
          # ...
```

---

### 2.7 `global_functions.py` — `create_image_requests`

**Purpose:** Insert chart images at the correct slide indices for the active quarterly template.

**Previous:**

```python
if ans.quarterly == True:
    report_type = 'Quarterly'
```

**Current:**

```python
if ans.quarterly:
    report_type = 'Quarterly_Claims_Lagged' if getattr(ans, 'lagged_claims', False) else 'Quarterly'
elif ans.monthly:
    report_type = 'Monthly'
```

---

### 2.8 `qc_reports.py` — `get_list_of_images_for_each_slide_index`

**Purpose:** QC slide→chart mapping respects the same quarterly chart key as production.

**Previous:** Used `chart_keys['Charts'][report_type.title()]` (i.e. `'Quarterly'`) for all quarterly runs.

**Current:**

```python
import analyst_selections as ans
# ...
if report_type.lower() == 'quarterly':
    chart_key = 'Quarterly_Claims_Lagged' if getattr(ans, 'lagged_claims', False) else 'Quarterly'
elif report_type.lower() == 'monthly':
    chart_key = 'Monthly'
else:
    chart_key = report_type.title()

image_dicts = chart_keys['Charts'][chart_key].items()
```

---

<a id="changelog-toc-3"></a>

## 3. Part B — Batch mode wired to claims-dependent aggregation query

### 3.1 `global_functions.py` — `get_claims_dependent_aggregation_ids`

**New function:** Loads `queries/claims/claims_dependent_aggregation_ids.sql`, substitutes dates and `final_cte='agg_list'`, executes via laaso, returns a DataFrame with `aggregation_id` and `aggregation_name`.

```python
def get_claims_dependent_aggregation_ids(report_start_date, report_end_date):
    query_path = f'{ref.query_dir}claims/claims_dependent_aggregation_ids.sql'
    with open(query_path) as f:
        q = f.read()
    query = q.format(
        report_start_date=report_start_date,
        report_end_date=report_end_date,
        final_cte='agg_list'
    )
    df = client.pandas.query(query, dialect, opts=laaso.QueryOptions(format=DataLocation.CSV_FORMAT))
    return df
```

---

### 3.2 `global_functions.py` — `initialize_params`

**Previous:** Batch mode only ever built params from `(report_start_date, report_end_date, rolling12_start_date, prod_cycle)` with no `aggregation_id`.

**Current:** If `report_specs` already contains **`aggregation_id`** (after lagged batch prep in `execute_reports.py`), build the same **VALUES** shape as **manual** mode (one row per aggregation) and return early:

```python
# Lagged-claims batch: report_specs has aggregation_id list; use same params shape as manual
if 'aggregation_id' in ans.report_specs:
    aggregation_id_list = ans.report_specs['aggregation_id']
    lookback_start_date_list = ans.report_specs['lookback_start_date']
    for idx in range(len(prod_cycle_list)):
        specs_list += f"('{aggregation_id_list[idx]}', DATE('{report_start_date_list[idx]}'), ..."
    params = f"""
      SELECT *
        FROM (
          VALUES {specs_list}
          ) AS t (aggregation_id, report_start_date, report_end_date, lookback_start_date, rolling12_start_date, prod_cycle)
    """
    return params
```

---

### 3.3 `global_functions.py` — `initialize_report_config_cte`

**Previous:** With `ans.batch`, always used the “pull all reports from global map + CROSS JOIN params” CTE.

**Current:** If **`aggregation_id` in `report_specs`**, use the **manual-style** CTE (join `global_client_reporting_map` to `params` on `aggregation_id`) so `report_configuration/reports_df.sql` sees only those aggregations:

```python
if 'aggregation_id' in ans.report_specs:
    report_config_cte = f"""
         WITH params AS (
         {params} )
          SELECT DISTINCT
                 CONCAT(report_id_base, '_', CAST(report_start_date AS VARCHAR), '_', CAST(report_end_date AS VARCHAR)) report_id,
                 aggregation_id,
                 aggregation_name,
                 params.{start_date} AS report_start_date,
                 report_end_date,
                 prod_cycle,
                 scheme_name,
                 scheme_type
            FROM client_reporting.global_client_reporting_map:v1
            JOIN params
           USING (aggregation_id)
           WHERE ({secondary_filter})
  """
    return report_config_cte
```

---

### 3.4 `execute_reports.py`

**Purpose:** Before `prep_reports_df(run=True)`, for **batch + quarterly + lagged_claims**, run the claims-dependent query and **overwrite `ans.report_specs`** with one row per returned aggregation.

**Previous:** No such block; batch `report_specs` stayed as single-element date lists only.

**Current:**

```python
import pandas as pd
# ...

# Lagged-claims batch: get aggregation IDs from claims-dependent query and set report_specs before prep_reports_df
if ans.batch and ans.quarterly and ans.lagged_claims:
    report_start_date = ans.report_specs['report_start_date'][0]
    report_end_date = ans.report_specs['report_end_date'][0]
    prod_cycle = ans.report_specs['prod_cycle'][0]
    agg_df = gf.get_claims_dependent_aggregation_ids(report_start_date, report_end_date)
    if agg_df is None or len(agg_df) == 0:
        raise Exception(
            '❌ Claims-dependent aggregation query returned no rows. '
            'Check report dates and that the claims_dependent_aggregation_ids query runs successfully.'
        )
    lookback_start_date = pd.to_datetime(report_start_date).replace(month=1, day=1).strftime('%Y-%m-%d')
    n = len(agg_df)
    ans.report_specs = {
        'prod_cycle': [prod_cycle] * n,
        'aggregation_id': agg_df['aggregation_id'].astype(str).tolist(),
        'report_start_date': [report_start_date] * n,
        'report_end_date': [report_end_date] * n,
        'lookback_start_date': [lookback_start_date] * n,
    }
    print(gf.format_style(f'Lagged-claims batch: using {n} aggregation(s) from claims-dependent query', 'ok'))

reports_df = gf.prep_reports_df(run=True)
```

**Note:** `ans.batch` remains `True`; only `report_specs` shape changes so downstream SQL uses the explicit aggregation list.

---

<a id="changelog-toc-4"></a>

## 4. How to run

### Manual quarterly lagged (abbreviated template + chart keys)

```bash
python run_reports.py --mode manual --quarterly --lagged-claims \
  --aggregation-ids "SALESCLOUD_ACCOUNT_ID:...,..." \
  --report-start-date YYYY-MM-DD --report-end-date YYYY-MM-DD
```

### Batch quarterly lagged (aggregation list from SQL + abbreviated template)

```bash
python run_reports.py --mode batch --quarterly --lagged-claims \
  --report-start-date YYYY-MM-DD --report-end-date YYYY-MM-DD
```

### Regular full quarterly (no lag flag)

```bash
python run_reports.py --mode batch --quarterly
```

Ensure `AWS_ENVIRONMENT` and working directory are set as required by existing SCRAPPY docs (run from `project_scrappy`).

---

<a id="changelog-toc-5"></a>

## 5. Files touched

| File | Change type |
|------|-------------|
| `run_reports.py` | `--lagged-claims`, `LAGGED_CLAIMS_RUN`, config print |
| `analyst_selections.py` | `lagged_claims`, validation |
| `generate_reports.py` | Template IDs, slide dict selection, log |
| `reference.py` | `quarterly_claims_lag_slide_template_dict` vs `quarterly_slide_template_dict` |
| `report_keys.yaml` | `Quarterly` (full), `Quarterly_Claims_Lagged` (abbreviated) |
| `global_functions.py` | `get_claims_dependent_aggregation_ids`, `initialize_params` / `initialize_report_config_cte` branches, `create_image_requests` chart key |
| `qc_reports.py` | Quarterly chart key resolution |
| `execute_reports.py` | Lagged batch `report_specs` population, `pandas` import |

**SQL (unchanged in this work, but required for batch lag list):**  
`queries/claims/claims_dependent_aggregation_ids.sql`

---
