# Fingertips Data Model – `test-fingertips`

## 1. Purpose & high-level design

The `test-fingertips` database stores curated **indicator data from the Fingertips public health profiles** platform in a star-schema:

- One main **fact table**: `fact_indicator_measure_current`
- Multiple **dimension tables**:
  - `dim_indicator_current`
  - `dim_area_current`
  - `dim_age_current`
  - `dim_sex_current`
  - `dim_category_current`
  - `dim_time_current`
- One **technical log table**: `partition_refresh_log`

### Grain of the model

- **Fact table grain:**  
  1 row ≈ **one indicator value for a given area, time period, age group, sex, category, profile and snapshot**.

  More precisely:  
  `indicator_id + area_code + time_sortable + sex_key + age_key + category_type + category_value + profile_key + snapshot_date` (and `area_type_id`) should be close to a unique record.

- Dimensions hold reference data for common attributes used across Fingertips profiles.

### Typical analytical questions

- “What is the QOF depression incidence in each ICB for 2023/24, by sex and age group?”  
- “Show practice-level prevalence for indicator X vs England average for the latest year.”  
- “Trend over time for an indicator for a specific area (e.g. LA, ICB, region).”

The star schema is designed so that:
- **Measures & CIs** live in `fact_indicator_measure_current`
- **Lookups/text** live in dimension tables
- **Refresh tracking** lives in `partition_refresh_log`

For **LangChain / NLP** this model should be described as:

> There is one main table of indicator values, with lookup tables for area, indicator, time, age, sex and category. Almost any query about rates, percentages, or counts of Fingertips indicators will primarily use the `fact_indicator_measure_current` table joined to these dimensions.

---

## 2. Dimension tables

### 2.1 `dim_age_current`

**Business description:**  
Lookup of age bands used in Fingertips indicators (e.g. `<28 days`, `<75 yrs`, `All ages`, etc.).

**Technical info:**

- **Table name:** `test-fingertips.dim_age_current`  
- **Grain:** 1 row = 1 age band currently in use.

**Columns:**

| Column    | Type   | Description                                             |
|----------|--------|---------------------------------------------------------|
| `age_key` | string | Human-readable label of an age band (e.g. `<28 days`, `<75 yrs`, `0–4 yrs`, etc.). |

**Relationships:**

- Joined from `fact_indicator_measure_current.age_key = dim_age_current.age_key`.

**NLP / LangChain hints:**

- Users will say: “age group”, “age band”, “age category”, “for adults 18+”, “under 75”, etc.
- Map those concepts to `age_key`.
- When a question mentions a specific age band (e.g. “under 75”), filter `fact_indicator_measure_current.age_key` to the matching label in `dim_age_current`.

---

### 2.2 `dim_area_current`

**Business description:**  
Lookup of geographical areas (MSOA, LA, region, England, etc.) and their parent areas.

**Technical info:**

- **Table name:** `test-fingertips.dim_area_current`  
- **Grain:** 1 row = 1 area code (at its defined area_type).

**Columns:**

| Column          | Type   | Description                                                                                          |
|-----------------|--------|------------------------------------------------------------------------------------------------------|
| `area_type_id`  | int    | Numeric identifier for the type of area (e.g. MSOA, LA, ICB, Region, England).                      |
| `area_code`     | string | Official ONS/NHS area code (e.g. `E02000004`, `E06000001`, `E12000001`, `E92000001`).               |
| `area_name`     | string | Area name (e.g. `Eastbrookend`, `North East region (statistical)`, `England`).                      |
| `parent_code`   | string | Code of the parent area (e.g. MSOA’s parent local authority, region, or England).                   |
| `parent_name`   | string | Name of the parent area (e.g. `England`).                                                            |
| `area_type_name`| string | Human-readable area type (e.g. `Middle Super Output Area`, `Region`, `Local Authority` etc.).       |

**Relationships:**

- Joined from `fact_indicator_measure_current.area_code = dim_area_current.area_code`.
- `fact_indicator_measure_current.area_type_id` aligns with `dim_area_current.area_type_id`.
- `parent_code` / `parent_name` can be used for roll-ups (e.g. MSOA → England).

**NLP / LangChain hints:**

- Users will say: “by area”, “by MSOA”, “by local authority”, “region”, “ICB”, “for England overall”.
- Map:
  - “England overall” → filter on `area_code = 'E92000001'` (or `area_type_name = 'Country'`).
  - “North East region” → `area_name LIKE 'North East%' AND area_type_name = 'Region'`.
- For queries like “for each region” or “by local authority”, use this table to group or filter by `area_type_name` and `area_name`.

---

### 2.3 `dim_category_current`

**Business description:**  
Lookup of non-demographic categories used in Fingertips indicators (e.g. deprivation deciles, care settings, ethnic groups – depending on profile). The example you gave is deprivation deciles.

**Technical info:**

- **Table name:** `test-fingertips.dim_category_current`  
- **Grain:** 1 row = 1 category label within a category type.

**Columns:**

| Column          | Type   | Description                                                                                                    |
|-----------------|--------|----------------------------------------------------------------------------------------------------------------|
| `category_type` | string | Category dimension name, e.g. `County & UA deprivation deciles in England (IMD2019, 4/21 geography)`.         |
| `category_value`| string | Category value label, e.g. `Second most deprived decile (IMD2019)`.                                            |

**Relationships:**

- Joined via `fact_indicator_measure_current.category_type` and `category_value`.

**NLP / LangChain hints:**

- Users may say: “by deprivation decile”, “most deprived areas”, “least deprived decile”.
- Map that to:
  - `category_type` including “deprivation deciles”.
  - `category_value` including phrases like “most deprived” or “least deprived”.
- When a question doesn’t mention categories at all, it may be valid to ignore `category_type/category_value` (or filter to a default such as “All categories” where appropriate).

---

### 2.4 `dim_indicator_current`

**Business description:**  
Master list of indicators, with ID, long definition and display label.

**Technical info:**

- **Table name:** `test-fingertips.dim_indicator_current`  
- **Grain:** 1 row = 1 Fingertips indicator.

**Columns:**

| Column                | Type   | Description                                                                                                                              |
|-----------------------|--------|------------------------------------------------------------------------------------------------------------------------------------------|
| `indicator_id`        | int    | Numeric Fingertips indicator ID (e.g. `90646`, `92590`).                                                                                |
| `indicator_name`      | string | Full indicator name (e.g. `Depression: QOF incidence - new diagnosis`).                                                                 |
| `indicator_definition`| string | Detailed definition/description explaining what the indicator measures and how it is calculated.                                        |
| `display_label`       | string | Short label intended for charts/dashboards (often same as `indicator_name` but may be shortened).                                       |

**Relationships:**

- Joined from `fact_indicator_measure_current.indicator_id = dim_indicator_current.indicator_id`.

**NLP / LangChain hints:**

- Users will mention indicators by:
  - Full name (“Depression QOF incidence – new diagnosis”)
  - Partial phrase (“QOF depression incidence”, “PAD prevalence”, “smoking prevalence”)
  - Concept (“depression”, “stroke admissions”, “life expectancy”)
- LangChain/LLM should:
  - Prefer matching user text to `display_label` and `indicator_name`.
  - Possibly embed both `indicator_name` and `indicator_definition` for semantic search of indicators.
- When user doesn’t know the exact indicator, you can:
  - Run a similarity search over `indicator_name + indicator_definition`.
  - Return suggested indicators and their IDs.

---

### 2.5 `dim_sex_current`

**Business description:**  
Lookup of sex categories used in the indicators.

**Technical info:**

- **Table name:** `test-fingertips.dim_sex_current`  
- **Grain:** 1 row = 1 sex category.

**Columns:**

| Column    | Type   | Description                              |
|----------|--------|------------------------------------------|
| `sex_key`| string | Sex label (e.g. `Male`, `Female`, `Persons`, `Not applicable`). |

**Relationships:**

- Joined from `fact_indicator_measure_current.sex_key = dim_sex_current.sex_key`.

**NLP / LangChain hints:**

- Users may say “for men”, “for women”, “both sexes combined”, “all persons”.
- Map:
  - “men / male” → `sex_key = 'Male'`
  - “women / female” → `sex_key = 'Female'`
  - “all / overall / both sexes” → `sex_key` = `'Persons'` or equivalent.
- If sex is not mentioned, it might be fine to:
  - Either default to the `Persons` row if available, or
  - Return all sex categories and let visualisation handle it.

---

### 2.6 `dim_time_current`

**Business description:**  
Lookup table for time periods and a sortable numeric representation.

**Technical info:**

- **Table name:** `test-fingertips.dim_time_current`  
- **Grain:** 1 row = 1 time period label.

**Columns:**

| Column         | Type   | Description                                                                                               |
|----------------|--------|-----------------------------------------------------------------------------------------------------------|
| `time_sortable`| int    | Numeric time key used for sorting and joins (e.g. `20160000` for `2016/17`, `20160100` for `2016 Q1`).   |
| `time_label`   | string | Human-readable time period label (e.g. `2016/17`, `2016 Q1`, `2015`, `2020-22`).                         |

**Relationships:**

- Joined from `fact_indicator_measure_current.time_sortable = dim_time_current.time_sortable`  
  or via `time_label` if needed.

**NLP / LangChain hints:**

- Users talk about time as:
  - “2015”, “2016/17”, “2016 Q1”, “latest year”, “over time”.
- Model strategies:
  - Map user’s text to `time_label` for filters.
  - Use `time_sortable` for ordering (e.g. time series charts).
  - “Latest year” → max(`time_sortable`) for the relevant indicator/area.

---

## 3. Fact & technical tables

### 3.1 `fact_indicator_measure_current`

**Business description:**  
Core table of Fingertips indicator values, one row per combination of:

- indicator
- area
- time period
- sex
- age group
- category (e.g. deprivation decile, where used)
- profile
- snapshot date (when this data was ingested)

Includes value, confidence intervals, counts, denominators, and comparison flags.

**Technical info:**

- **Table name:** `test-fingertips.fact_indicator_measure_current`  
- **Grain:** 1 row ≈ 1 indicator measure for one area/time/sex/age/category/profile/snapshot.
- **Partitioning:**  
  The table is partitioned. In most implementations this will be by `profile_key`, `area_type_id` and `snapshot_date` (adjust documentation if your actual partition keys differ).

**Columns:**

| Column           | Type    | Description                                                                                                                       |
|------------------|---------|-----------------------------------------------------------------------------------------------------------------------------------|
| `indicator_id`   | int     | ID of the indicator (FK to `dim_indicator_current.indicator_id`).                                                                |
| `area_code`      | string  | Geographic area code (FK to `dim_area_current.area_code`).                                                                       |
| `area_name`      | string  | Area name, repeated from source for convenience (denormalised from area dimension).                                             |
| `parent_code`    | string  | Parent geography code (e.g. region or England), as per source data.                                                             |
| `parent_name`    | string  | Name of the parent geography.                                                                                                    |
| `sex_key`        | string  | Sex category (FK to `dim_sex_current.sex_key`).                                                                                  |
| `age_key`        | string  | Age band (FK to `dim_age_current.age_key`).                                                                                      |
| `category_type`  | string  | Category dimension name (FK to `dim_category_current.category_type` together with `category_value`).                             |
| `category_value` | string  | Specific category label (e.g. deprivation decile, FK to `dim_category_current.category_value`).                                  |
| `time_label`     | string  | Human-readable time period label.                                                                                                |
| `time_sortable`  | int     | Numeric key for time (FK to `dim_time_current.time_sortable`).                                                                  |
| `value`          | double  | Main measure value (percentage, rate, etc.) as published in Fingertips.                                                          |
| `lower_ci_95`    | double  | Lower bound of the 95% confidence interval for the value (nullable where not provided).                                          |
| `upper_ci_95`    | double  | Upper bound of the 95% confidence interval for the value (nullable where not provided).                                          |
| `count`          | double  | Numerator count (e.g. number of events/cases), where provided.                                                                   |
| `denominator`    | double  | Denominator (e.g. population at risk, total practice population) used to compute the indicator.                                  |
| `recent_trend`   | string  | Qualitative recent trend descriptor (e.g. “Increasing”, “Decreasing”, “No significant change”, or “Cannot be calculated”).       |
| `comp_to_eng`    | string  | Comparison to England overall (e.g. “Better”, “Worse”, “Same”, “Not compared”).                                                  |
| `comp_to_region` | string  | Comparison to region (similar values to `comp_to_eng`).                                                                           |
| `value_note`     | string  | Additional notes about the value (e.g. suppression notes, data quality warnings).                                                |
| `snapshot_date`  | date    | Date when this version of the data was ingested into the lake (e.g. `2025-09-17`).                                              |
| `profile_key`    | string  | Profile identifier from Fingertips (e.g. `populations`, `mortality-profile`) representing the thematic profile for the indicator.|
| `area_type_id`   | int     | Area type ID, matching `dim_area_current.area_type_id`.                                                                          |

**Relationships:**

- `indicator_id` → `dim_indicator_current.indicator_id`  
- `area_code`, `area_type_id` → `dim_area_current.area_code`, `dim_area_current.area_type_id`  
- `sex_key` → `dim_sex_current.sex_key`  
- `age_key` → `dim_age_current.age_key`  
- `category_type`, `category_value` → `dim_category_current.category_type`, `dim_category_current.category_value`  
- `time_sortable` → `dim_time_current.time_sortable`  
- `profile_key`, `area_type_id`, `snapshot_date` used for partitioning and refresh behaviour.

**NLP / LangChain hints:**

1. **Core measure field**
   - When user says “value”, “rate”, “percentage”, “indicator value”, “prevalence”, “incidence”, etc., they usually mean the `value` column.
   - If they say “number of cases”, “numerator”, “count of events”, use `count`.
   - If they say “population” or “denominator`, use `denominator`.

2. **Time**
   - User references to years/quarters must be matched to `time_label` and sorted using `time_sortable`.
   - “Latest data” → filter to max(`time_sortable`) for that indicator & area combination.

3. **Geography**
   - For queries “for England”, filter `area_code` or `area_name` appropriately (England code).
   - For queries “by [area type]”, use `dim_area_current.area_type_name` to decide which area codes to include.

4. **Subgroups**
   - “By sex” → group by `sex_key`.
   - “By age group” → group by `age_key`.
   - “By deprivation decile” → group by `category_value` where `category_type` contains “deprivation”.

5. **Comparisons**
   - Questions like “Is this better or worse than England?” use `comp_to_eng`.
   - “Compared to region” uses `comp_to_region`.
   - “Trend” uses `recent_trend`.

6. **Profiles**
   - `profile_key` allows you to scope queries to a particular Fingertips profile (e.g. `populations`, `mortality-profile`).  
   - For example, “within the mortality profile” → filter `profile_key = 'mortality-profile'`.

**Example NL → SQL mapping (for training docs)**

_Question_: “What is the QOF depression incidence for females under 75 in the North East region in 2015?”  

Use:

- `dim_indicator_current` to find `indicator_id` for “Depression: QOF incidence - new diagnosis”.  
- `dim_area_current` to find `area_code` for “North East region (statistical)”.  
- Filter `fact_indicator_measure_current` on:
  - `indicator_id = <found id>`  
  - `area_name = 'North East region (statistical)'` (or matching `area_code`)  
  - `sex_key = 'Female'`  
  - `age_key = '<75 yrs'`  
  - `time_label = '2015'`  

Select the `value` (and optionally CI, count, denominator).

---

### 3.2 `partition_refresh_log`

**Business description:**  
Technical table that records how partitions in the fact table were refreshed for each profile and area type. Useful for data lineage, auditing and debugging incremental loads.

**Technical info:**

- **Table name:** `test-fingertips.partition_refresh_log`  
- **Grain:** 1 row = 1 refresh action for a given (`profile_key`, `area_type_id`, `snapshot_date`).

**Columns:**

| Column        | Type   | Description                                                                                                 |
|---------------|--------|-------------------------------------------------------------------------------------------------------------|
| `profile_key` | string | Profile identifier (e.g. `mortality-profile`).                                                             |
| `area_type_id`| int    | Area type ID whose partitions were affected.                                                                |
| `action`      | string | Action taken, e.g. `drop`, `refresh`, `add`, etc. (from your loader logic).                                |
| `prev_sha`    | string | Hash of the previous snapshot of the raw data for this profile + area_type (may be null on first load).   |
| `curr_sha`    | string | Hash of the current snapshot of the raw data (used to detect changes).                                     |
| `snapshot_date`| date  | Date when this action was executed (links logically to `fact_indicator_measure_current.snapshot_date`).    |

**Usage:**

- To see when specific profile/area_type partitions were updated or dropped.
- To support idempotent loads: if `prev_sha == curr_sha` you might skip reloads.

**NLP / LangChain hints:**

- This is primarily **technical**, not for end-user analytics.
- It can be used if you build meta-queries like:
  - “When was the mortality profile last refreshed?”
  - “Which area types were updated on 17 Sept 2025?”
- For most public-health questions (“rates, trends, etc.”), **ignore this table**.

---

## 4. Suggested context text for LLMs (LangChain)

The `test-fingertips` database is a star schema containing indicator data from the Fingertips public health profiles platform. The main table is `fact_indicator_measure_current`, which stores the indicator values and confidence intervals. It is linked to dimension tables for indicators (`dim_indicator_current`), geography (`dim_area_current`), age bands (`dim_age_current`), sex (`dim_sex_current`), other categories such as deprivation (`dim_category_current`), and time periods (`dim_time_current`).

Almost all analytical questions about public-health indicators (rates, percentages, counts, trends, comparisons) should use `fact_indicator_measure_current` joined to one or more of the dimensions.

- Filter or group by `indicator_id` and look up human-friendly names in `dim_indicator_current`.  
- Filter or group by area using `area_code`/`area_name` and `area_type_name` in `dim_area_current`.  
- Filter or group by `sex_key`, `age_key` and `category_value` to look at subgroups.  
- Use `time_label` and `time_sortable` to filter by year or quarter and to build time series.  
- Use `value` for the main indicator value, `count` and `denominator` for numerators and denominators, and `lower_ci_95` / `upper_ci_95` for confidence intervals.  
- Use `recent_trend`, `comp_to_eng`, and `comp_to_region` for questions about trends and comparisons to England/region.  

The `partition_refresh_log` table is a technical log for monitoring how data was refreshed and should only be used for questions about data lineage or refresh history, not for public health statistics.
