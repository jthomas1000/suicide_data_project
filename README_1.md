# US Suicide Rates, 1950-2018

A data cleaning pipeline and Power BI report built on NCHS/CDC suicide mortality statistics for the United States, broken down by sex, age, and race/Hispanic origin.

## Data source

`Death_rates_for_suicide__by_sex__race__Hispanic_origin__and_age__United_States.csv`, the public NCHS "Death rates for suicide" table: crude suicide rates per 100,000 population, 1950 through 2018, split out by demographic group.

## Pipeline (`SuicideMix.ipynb`, Google Colab)

The notebook takes the raw NCHS export and turns it into a clean, analysis-ready table.

1. **Load and rename.** Drop the `FLAG` and `INDICATOR` columns, rename the NCHS field names to something readable (`STUB_LABEL` to `demo_type`, `ESTIMATE` to `suicide_rate`, and so on).
2. **Keep crude rates only.** The raw file reports both crude and age-adjusted rates for the same demo_type_id/YEAR pair. Age-adjusted is only calculated for aggregate levels (totals, sex, race) and never for a specific age bracket, so filtering to age-adjusted would silently drop every age-bracket row. Crude is the one that keeps all the data, so that's what the pipeline standardizes on.
3. **Drop the 2018 "Single race" rows.** 2018 was re-tabulated under newer single-race categories as a one-time comparison against the standard bridged-race categories used for the rest of the 1950-2018 series. Those comparison rows duplicate demo_type_id codes already covered by the bridged-race series, so they're dropped.
4. **Resolve remaining duplicates.** A handful of (demo_type_id, YEAR) pairs are genuine duplicate publications in the raw NCHS file with two different reported values. There's no principled way to pick a "correct" one, so the pipeline averages `suicide_rate` within each duplicated pair.
5. **Impute missing values.** Interior gaps (missing years with real data on both sides) are filled by linear interpolation. Leading or trailing gaps are extrapolated with a per-group linear regression on YEAR, but only when a group has at least 4 observed points to fit a trend on. Predictions are clamped at 0, and every filled row is flagged in `suicide_rate_imputed` so it stays distinguishable from an observed value. About 2.3% of rows end up imputed.
6. **Map to demographic groups.** Numeric `demo_type_id` codes are mapped to labels (age bands, sex, Hispanic origin) via a lookup dictionary. Race groups are matched by keyword against the `demo_type` text, restricted to demo_id 4 so it can never collide with the Hispanic-origin classification done elsewhere.
7. **Validate and export.** Before export, the pipeline asserts no negative rates and no group value outside the known set. The cleaned table is exported three ways:
   - `suicide_rates_richdata.csv`: a long-format table (YEAR, Rate, Imputed, Breakdown, Sex, AgeGroup, Race) covering every breakdown (total, age, sex, sex+age, sex+race, Hispanic origin).
   - `suicide_data_export.json`: the same corrected data rounded to 2 decimal places.
   - A MySQL table (`suicide_data`, hosted on Aiven) with indexes on `YEAR` and `demographic_group`, connected over TLS with the CA certificate pinned via `DB_SSL_CA`.

## Power BI report (`SuicideMix.pbix`)

Six pages, built from the cleaned data (`suicide_rates_powerbi.xlsx`). "All" aggregate rows are excluded from age-, sex-, and race-specific visuals to avoid double-counting alongside their component categories.

| Page | What it shows |
|---|---|
| National Overview | The overall U.S. rate from 1950 to 2018, plus a year slicer and a card showing the imputed-row percentage. |
| Sex Comparison | Male vs. female rate trends and the male:female ratio over time. |
| Age Analysis | Rate and percent change (2000-2018) by age band. |
| Sex x Age Matrix | A pivot table and matrix crossing sex and age group. |
| Race & Ethnicity | Rate trends and percent change by race/ethnicity. |
| Findings & Notes | Narrative summary of the key patterns, plus a data-notes card documenting scope and exclusions. |

### Headline findings

- The national rate fell from 11.4 (1950) to a series low of 10.4 (2000), then rose to 14.8 by 2018, a 42.3% increase over that span.
- Men have died by suicide at a consistently higher rate than women throughout the period. As of 2018 the male rate was 3.83 times the female rate, and that gap has widened rather than narrowed.
- Rates rise with age overall (the 85+ band is the highest in the dataset), but since 2000 the sharpest percentage increases have been among younger and middle-aged groups, not the elderly.
- Every racial and ethnic group tracked has seen its rate climb since 2000. White and Native American or Alaska Native populations have the highest absolute rates, while White (+45.7%) and Hispanic or Latino (+44.4%) populations saw the largest percentage increases.

## Repo contents

- `SuicideMix.ipynb`: the Colab data cleaning and export pipeline described above.
- `SuicideMix.pbix`: the Power BI report.
- `suicide_rates_richdata.csv` / `suicide_data_export.json`: cleaned data exports produced by the notebook.

## Requirements

- Python: pandas, numpy, scikit-learn (for the notebook)
- A MySQL-compatible database (optional, only needed for the DB export cell) with connection details supplied via a `.env` file (`DB_USER`, `DB_PASS`, `DB_HOST`, `DB_PORT`, `DB_NAME`, and optionally `DB_SSL_CA`)
- Power BI Desktop to open and edit the `.pbix` report
