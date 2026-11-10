# Medical & Pharmacy Claims Experience — AtScale SML (converted from Power BI model.bim)

## Inputs

- `context/model.bim` — Power BI / Tabular semantic model (compatibility level 1705, DirectLake over the `adb_mirror_altprod` mirror, schema `analytics_mee_uat_silver`). 71 tables, 173 relationships, 1,213 DAX measures. Sole content input.
- `context/convert_bim_to_sml.py` — the deterministic emitter / DAX→MDX translator used for this build.
- Use case doc: (not provided — measure display folders and names in the .bim used instead)
- NLQs: (not provided)
- DDL: (not provided — column names and types taken from the .bim `sourceColumn` / `dataType`)
- ERD: (not provided — relationships taken from the .bim)
- Data profile: (not provided)
- Target warehouse: Databricks (inferred from the Fabric "Azure Databricks mirror" DirectLake source)
- build.yaml: (not provided)

## Build parameters

| Parameter | Value |
|---|---|
| unrelated_dimension_handling | repeat |
| warehouse | Databricks |
| database | adb_mirror_altprod |
| schema | analytics_mee_uat_silver |
| model_unique_name | medical_pharmacy_claims_experience |
| catalog_unique_name | medical_pharmacy_claims_experience_catalog |
| currency | USD (from DAX format strings) |
| time_window | Calendar, month-end grain (bridge_date_ca) |
| use_cases_covered | Medical IP/OP/Prof, Pharmacy, Behavioral Health, Membership, Claimants, Continuous Enrollment/Cohort |
| use_cases_excluded | Report-layer UI measures (toggles, titles, colours, field-parameter switches) |
| semi_additive_default | none |

## Assumptions and decisions

- **Warehouse = Databricks; database = `adb_mirror_altprod`** — derived from the DirectLake expression name. That is the Fabric mirror item name; the real Unity Catalog name may differ. **Override `database:` in the connection file before deploy.**
- **Identifiers lower-cased** — physical names come from each column's `sourceColumn`, lower-cased per Rule 1 (Databricks resolves case-insensitively).
- **Table selection** — 13 fact tables, 25 dimension tables and the month bridge were converted. Not converted (no fact relationship, or report-only): `Dim_Period`, `Dim_MonthYear`, `Dim_Benchmark`, `Dim_Role`, `dim_iiqrole`, `Dim_DrugHandlingProfileView`, `dim_drg` (its only relationship, DRG id → DrugId, looks wrong), `Dim_ReportingSegment1Account` / `Dim_ReportingSegment2Account` (inactive), the 21 `ST_*` slicer/parameter tables, and the `MetricStore` calculated table (it only held measures).
- **Date dimension** — `bridge_date_ca` became a `type: time` dimension (Year → Quarter → Month, keyed on `asofmonthenddayid`) with calculated columns for year, quarter and month labels. Every fact's single month-end key joins to this same conformed dimension (Rule 21): incurred month for medical/BH, filled month for Rx, month-end for membership and continuous enrollment. `bridge_anchordate_ca` became a separate **Anchor Date Dimension** for the two claimant facts, matching the .bim.
- **Behavioral-health facts** — the four `fact_claim_medical_bh_*` tables have no relationships in the .bim (the DAX reached them with TREATAS). 91 relationships were inferred by matching FK column names used by the other facts.
- **Stale table names** — DAX refers to `Fact_MedClaim_BH_*_Agg`, which don't exist in the .bim. These were mapped to the matching `fact_claim_medical_bh_*_ca_agg` tables.
- **Dimension shape** — each dimension has one key level named with the human-readable column. Other columns are secondary attributes keyed by their own column (Rule 8b). Any attribute that a converted DAX filter uses (for example `MailOrderTypeName = "Retail"`) was promoted to its own single-level hierarchy, so MDX tuples can address it.
- **DAX calculated columns** (`ENA`, `ENABenchMark`, `Order By`, `HCCType_Label`, `Month`, etc.) were not converted.
- **`is_unique_key`** is omitted everywhere because no data profile was provided (Rule 16).
- **Base metrics** — every summable numeric fact column (`summarizeBy: sum`, excluding ID/flag/security columns) became a `sum` metric named `<Fact prefix> <Column>`. When a DAX measure was a plain aggregation (`SUM`, `DISTINCTCOUNT`, `COUNT`, `MIN`, `MAX`, `AVERAGE`), the base metric took the DAX measure name instead.
- **DAX `CALCULATE` translation:**
  - A filter on a dimension column (`=`) became an MDX tuple member `&[value]`.
  - `IN {…}` became a sum of tuples.
  - `<>` became `(All, M) − (member, M)`.
  - `ALL` / `REMOVEFILTERS` became `[All]` members.
  - A filter on a fact column (for example `RelationshipId = 2` for subscribers, or `Ccsrl1_Flag = 1`) became a `CASE WHEN … THEN col END` calculated column on the fact dataset, with its own base metric.
  - `DIVIDE` became MDX `Divide`.
- **`unrelated_dimensions_handling: repeat`** is set on every base metric (the skill default).
- **Period Range / Runout Months** — the aggregate facts hold one row set per `PeriodRange` × `Runout`. Metrics will double-count unless you filter on **Period Range Dimension** and **Runout Months Dimension**, exactly as the Power BI report did. You could add default members or a perspective later.

## Generation summary

- Datasets: 39 (13 fact, 25 dimension, 1 date bridge)
- Dimensions: 26 (1 time + 25 standard)
- Base metrics: 140 (including filtered-column metrics built from DAX fact filters)
- Calculated metrics (`metric_calc`): 253
- Model relationships: 250 (159 from the .bim, 91 inferred for the BH facts)
- Role-play prefixes: none (one date per fact, conformed)
- DAX measures converted: 318 of 1,213. The remaining 895 are listed by reason in `CONVERSION_REPORT.md`. About 520 are report-interaction logic (slicer, toggle and title measures, field-parameter `SWITCH`, YoY/CAGR driven by `ST_MatrixYear`). 77 are the ENA benchmark pattern ("all accounts except the selected one", built with TREATAS). 224 depend on those.
- Caveats:
  - The tuple / `[All]` member syntax follows the bundled AtScale tuple docs but hasn't been tested against a live AtScale instance.
  - YoY can be rebuilt with AtScale time-calculation groups or `ParallelPeriod` calcs on the Date Dimension.
  - The ENA benchmark needs a design decision. One option is a benchmark-group query dataset; another is AtScale row security.

## Reproducing this build

`context/` holds verbatim copies of every input: the original `model.bim` and the converter script (`convert_bim_to_sml.py`). Run `python3 context/convert_bim_to_sml.py context/model.bim <out_dir>` with the same inputs and parameters above to produce an equivalent model. Then run the skill's `validate_sml.py`.
