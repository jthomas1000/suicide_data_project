# Data Cleaning & Analysis Report: CDC/NCHS Suicide Mortality Dataset

**Project:** SuicideMix — data pipeline and MySQL export
**Source notebook:** [Google Colab — SuicideMix](https://colab.research.google.com/drive/1coR6Ij-R3b4due4Aie844_ZQRaKNOBvm)
**Date:** August 2026

## Overview

This report documents the cleaning and preparation of a CDC/NCHS "Death rates for suicide, by sex, race, Hispanic origin, and age" dataset, and summarizes what the cleaned data shows. The raw file contains 6,390 rows covering the United States from 1950 through 2018 (data points are decennial before 1980 — 1950, 1960, 1970 — and annual from 1980 onward), broken out by sex, age group, race, and Hispanic origin. The pipeline was built in a Google Colab notebook using pandas, and the corrected output was exported to a MySQL database hosted on Aiven for downstream use.

The raw data had five distinct quality problems that needed to be resolved before it could be trusted: a mix of two incompatible rate types in the same table, a block of duplicate demographic categories, a small number of rows that collided on the same demographic/year key, several hundred missing rate values, and (on the engineering side) a couple of pipeline bugs that had nothing to do with the data itself but would have broken the export. Each is described below along with how it was fixed and how the fix was verified.

Since the first version of this report, the project has moved from a single flat demographic grouping to a properly dimensioned extract — Sex, Age Group, and Race/Ethnicity as separate columns rather than folded into one label — built specifically to support ranked comparisons, a sex-by-age matrix, and race/ethnicity trends in Power BI. That rebuild also fixed, at the source, a bug that the original flat grouping had introduced (see "Fixing the race/ethnicity sex-collapse bug at the source" below). The findings section has been expanded accordingly with results only visible once sex and age can be crossed against each other and against race.

## Deliverables

This project now produces three companion artifacts, each suited to a different use:

- **`suicide_rates_powerbi.xlsx`** — the primary analytical deliverable. A three-sheet workbook (Data, Data Dictionary, Key Insights) built from the enriched, properly-dimensioned extract described below, meant to be imported directly into Power BI.
- **`SuicideMix_PowerBI_Build_Guide.docx`** — a step-by-step guide specifying the exact Power BI report pages, visuals, filters, and DAX measures needed to turn that workbook into a multi-page analyst report (national trend, sex comparison, age analysis, a sex-by-age matrix, and race/ethnicity trends).
- **An interactive HTML dashboard** — a lighter-weight, browser-based companion view built earlier in the project, reading from the original `suicide_data` MySQL table. It remains a useful quick-look tool for browsing the same demographic groups, but the Power BI workbook above is now the more complete and more rigorously dimensioned dataset, and is the one to use for any deeper analysis.

## Data Quality Issues and Corrections

The dataset originally mixed two different measurement types in a single `UNIT` column: "crude" rates and "age-adjusted" rates, at 5,578 and 812 rows respectively. These are not comparable numbers — age-adjusted rates are statistically reweighted to control for a population's age distribution, while crude rates are not — so keeping both in one table would silently corrupt any aggregation across the full dataset. The pipeline filters down to the 5,578 crude-rate rows and drops the age-adjusted rows, since crude rates are the more granular and widely comparable series across every demographic breakdown in the file.

A second problem was a set of 96 rows tagged "Single race" that applied only to the year 2018. These existed to let analysts compare an older "bridged-race" classification method against a newer "single race" one, but every demographic code they used was already covered by the bridged-race rows elsewhere in the file. Left in place, they would have produced duplicate entries for the same demographic group and year. These 96 comparison rows were identified and dropped.

Even after those two fixes, 8 rows still shared the same demographic-group/year key — mostly cases where the same population subgroup had been recorded twice with slightly different values (for example, Black male populations aged 45–64 and 65+ each appeared twice for the same year). Rather than arbitrarily keeping one and discarding the other, the pipeline averages the `suicide_rate` values within each duplicated pair, which keeps all of the underlying information rather than throwing part of it away. A verification pass after this step confirmed that every (demographic group, year) pair is unique in the resulting table.

The dataset also carried a `FLAG` field on 906 of the 6,390 rows (roughly 14%), used by NCHS to mark estimates that don't meet reliability standards (small sample sizes, suppressed values, etc.). Checking this field against the `suicide_rate` column showed a perfect 1.0 correlation between having a flag and having a missing rate — in other words, every flagged row was exactly the set of rows where NCHS had suppressed or withheld the actual estimate. That confirmed the flag was a reliable signal for "value not reported," which set up the final data-quality step: 803 rows were missing a `suicide_rate` value. Of those, 700 were recoverable through interpolation and trend-based regression against neighboring years within the same demographic group, and the remaining 103 were left as missing because there wasn't enough surrounding data to support a trustworthy estimate — those rows were not guessed at.

Separately from the data itself, two engineering bugs in the notebook were found and fixed during this project. The pipeline cell had accumulated roughly 90 lines of dead code — an older, unused approach to extracting demographic categories from dictionaries, plus a duplicate, disconnected block of SQL/SSL setup code — left over from earlier iterations of the notebook. This was removed and the cleanup was verified with a full "Restart and run all," confirming the simplified cell produces identical output to before. Separately, the `pip install` step for the MySQL driver (`pymysql`) had ended up positioned after the export cell that depended on it — because Colab's "Run all" stops at the first uncaught exception, this meant a fresh run of the notebook would have failed before the install ever executed. That cell was moved to run immediately after the main pipeline cell, well before it's needed.

## Final Dataset

After all corrections, the cleaned dataset contains 1,092 rows spanning 21 demographic groups (10 age bands, 5 race/ethnicity categories, and male/female) across years 1950–2018. It was exported to the `suicide_data` table in a MySQL database on Aiven, with indexes on `YEAR` and `demographic_group` to support the kinds of time-series and group-comparison queries a dashboard would need. The connection uses a TLS-verified link authenticated against Aiven's own CA certificate, rather than falling back to an unverified connection.

### The enriched Power BI extract

The `demographic_group` scheme above folds sex, age, and race into a single text label per row, which is convenient for a flat lookup table but breaks down for cross-tabbing — you can't ask "what's the female rate for ages 10–14" from a table that only ever gives you one dimension per row. For the Power BI workbook, a second extract (`suicide_rates_richdata.csv`, 2,310 rows) was built directly from the pipeline's post-imputation dataframe, keeping Sex, Age Group, and Race/Ethnicity as separate columns rather than collapsing them into one label.

The underlying CDC source table isn't a single full cross-tab of sex × age × race — it stacks several independent breakdowns of the same mortality counts, distinguished by a `Breakdown` field: `Total` (42 rows, the national all-persons series), `Age` (588 rows, age bands with sex and race held at "All"), `Sex` (84 rows), `Sex and Age` (1,176 rows, previously unused), and `Sex and Race/Ethnicity` (420 rows). A row's rate is only comparable to another row's rate within the same `Breakdown` value — this is documented prominently in both the workbook's Data Dictionary sheet and the Power BI build guide, since it's the one detail that would otherwise be easy to get wrong when building a chart.

This extract also recovers the true national "All persons" trend line (the `Total` breakdown), which the flat `demographic_group` scheme above does not carry as its own row, and adds a Sex-by-Age breakdown that wasn't available in any form in the earlier dataset.

## Key Findings: Demographic Group Averages

These findings come from the original `demographic_group` scheme (full-period averages, 1950–2018) and remain valid; the deeper, trend-level findings below use the enriched, separately-dimensioned extract instead.

Looking at the cleaned data, the population-level ("All persons") crude suicide rate moved from about 13.3 per 100,000 people in 1950 to about 14.1 per 100,000 in 2018 — a modest net increase over nearly seven decades, though the year-by-year path in between is not covered by this summary and is not necessarily a straight line.

Two patterns stand out much more clearly than the overall trend. The first is sex: averaged across the full 1950–2018 period, the male rate (19.3 per 100,000) is roughly 3.8 times the female rate (5.1 per 100,000), and that gap widened rather than narrowed — the male rate rose from 17.8 in 1950 to 23.4 in 2018, while the female rate rose from 5.1 to 6.4 over the same span. The second is age: rates climb steadily through adulthood and peak among the oldest groups in the data, averaging 20.0 per 100,000 for ages 75–84 and 19.9 for ages 85+, compared to just 1.4 per 100,000 for ages 10–14.

| Demographic group | Avg. rate (per 100,000, 1950–2018) |
|---|---|
| Ages 75–84 | 20.00 |
| Ages 85+ | 19.86 |
| Male | 19.34 |
| Ages 65+ | 17.79 |
| Ages 45–54 | 16.96 |
| Ages 45–64 | 16.70 |
| Ages 55–64 | 16.31 |
| Ages 65–74 | 16.05 |
| Ages 35–44 | 15.52 |
| Ages 25–44 | 14.95 |
| Ages 25–34 | 14.34 |
| Ages 20–24 | 13.85 |
| White | 13.46 |
| Native American or Pacific Islander | 11.57 |
| Ages 15–24 | 11.32 |
| Ages 15–19 | 8.74 |
| Asian | 6.09 |
| Black | 5.99 |
| Hispanic or Latino | 5.85 |
| Female | 5.05 |
| Ages 10–14 | 1.41 |

Race and Hispanic-origin categories show a wider spread than might be expected: White (13.46) and Native American or Pacific Islander (11.57) populations have the highest average rates among these groups, roughly double the rates for Asian (6.09), Black (5.99), and Hispanic or Latino (5.85) populations.

## Deeper Findings: Trends, Not Just Averages

The findings above are single-number averages over the full 1950–2018 span. They flatten out exactly the kind of pattern a data analyst would actually want to see — whether a group's rate is rising or falling, and how fast, relative to the others. The enriched extract makes that possible by keeping Year, Sex, Age Group, and Race/Ethnicity as independent, crossable columns. Six findings stood out, all computed directly from the data (not estimated):

1. **A 2000 trough, then an 18-year climb.** The national rate fell fairly steadily from the mid-1980s to a series low of 10.4 per 100,000 in 2000, then rose without interruption through 2018, reaching 14.8 per 100,000 — the highest point in the entire 1950–2018 series. Averaged across the full period the change looks modest (13.3 → 14.1); the 2000–2018 window alone accounts for a 42% increase.
2. **Absolute risk and relative growth point in different directions across age groups.** The oldest groups (ages 75–84 and 85+) still carry the highest absolute rates, but grew the least since 2000 — +6.2% and −2.6% respectively. The fastest *relative* growth since 2000 is concentrated at the youngest end (ages 10–14, +93%) and in late-middle-age (ages 55–64, +67%), a pattern that's invisible if you only rank groups by their absolute level.
3. **Young women show the sharpest relative increases of any group in the dataset.** Female ages 10–14 rose from 0.6 to 2.0 per 100,000 between 2000 and 2018 — a +233% change. Female ages 20–24 (+100%) and 15–19 (+93%) follow close behind. No male age band comes close to this rate of relative increase over the same period.
4. **The male:female ratio peaked around 2000, then narrowed.** Comparing only the two endpoints (1950: 3.5x, 2018: 3.7x) makes the ratio look roughly flat, or even slightly wider — but the path between them isn't flat: it climbed from 3.5x in 1950 to a peak of about 4.3x around 2000, then narrowed back down to 3.7x by 2018, mainly because female rates (especially at younger ages) rose faster than male rates over that later window. The endpoint comparison and the mid-series peak are both real; neither alone tells the full story.
5. **White and Native American or Alaska Native populations have the highest race/ethnicity rates throughout the series**, a pattern that holds consistently from 1950 through 2018, not just as a full-period average.
6. **Every race/ethnicity group rose after 2000, but not at the same pace.** White (+45.7%) and Hispanic or Latino (+44.4%) grew fastest in relative terms between 2000 and 2018, followed by Native American or Alaska Native (+41.7%), Asian (+37.7%), and Black (+29.7%).

These figures, along with the headline metrics behind them, are also available as live formulas in the Key Insights sheet of `suicide_rates_powerbi.xlsx`, so they recalculate automatically if the underlying data changes.

## Fixing the Race/Ethnicity Sex-Collapse Bug at the Source

An earlier version of this project surfaced a data-quality problem while building the first companion dashboard: the five race/ethnicity groups (White, Black, Asian, Hispanic or Latino, Native American or Pacific Islander) each carried two rows per year in the exported `suicide_data` table instead of one. Comparing the values showed why — each pair was actually that race's male and female rate, but the pipeline's `demographic_group` construction step had collapsed both into the same label without keeping sex distinct. Age and sex groups weren't affected; only the race/ethnicity breakdown had lost that dimension. It didn't change any of the full-period averages reported above (an average of the two sub-rates equals the combined-sex rate either way), but it meant a query expecting one row per (race, year) would silently double-count, and a time-series chart built directly from the table would zig-zag between the male and female series instead of showing a clean trend.

At the time, the dashboard worked around this by averaging same-year duplicates per group at display time — a reasonable patch, but not a fix. The enriched extract described above fixes it properly, at the source: the pipeline's `Sex and race` breakdown reports each race once per sex (for example, separate "Male: White" and "Female: White" rows), and the new extraction step parses that label into two explicit columns — `Sex` and `Race/Ethnicity` — instead of discarding the sex half. The `suicide_rates_powerbi.xlsx` workbook and the Power BI report built from it now carry race and sex as genuinely independent dimensions, so no display-layer averaging is needed to get a clean per-(race, year) series.

## Limitations

A few caveats are worth keeping in mind when interpreting this data. The pipeline deliberately kept only crude rates, not age-adjusted ones, so any trend over time in a broad category (like "All persons" or "Male") is partly a function of how the U.S. population's age distribution shifted over the same 68 years, not purely a change in underlying risk. The 103 rows with no recoverable value are simply absent from analysis, which could understate or overstate certain group/year combinations if the missingness wasn't random. And because the pre-1980 data is decennial (1950, 1960, 1970) while post-1980 data is annual, any chart spanning the full period will show far more year-to-year detail after 1980 than before it. Finally, this is aggregate, population-level surveillance data — it describes rates within groups, not individuals, and shouldn't be read as saying anything about any specific person's risk.

The enriched extract's `Age` and `Sex and Age` breakdowns also mix non-overlapping 10-year bands (10–14, 15–19, 20–24, 25–34, 35–44, 45–54, 55–64, 65–74, 75–84, 85+) with broader bands that re-aggregate them (15–24, 25–44, 45–64, 65+) — this reflects how the CDC source table itself reports the data, for different downstream uses, not an error introduced by this pipeline. Charting all fourteen age labels side by side would double-count the population; every age-based visual in the Power BI build guide filters down to the ten non-overlapping bands for exactly this reason.

## Technical Summary

The full pipeline — source-file loading, the crude/age-adjusted filter, the single-race duplicate removal, the cross-year duplicate averaging, the interpolation/regression imputation, and the demographic-group construction — runs end-to-end in the Colab notebook, followed by the MySQL export with both indexes and a certificate-verified TLS connection. This has been confirmed working via a full restart-and-run-all of the notebook, with the export cell completing successfully and no SSL warnings.

A second, later step builds the enriched Power BI extract from the same post-imputation dataframe: it parses each of the CDC source table's demographic-breakdown labels (`Total`, `Age`, `Sex`, `Sex and age`, `Sex and race`, plus the Hispanic-origin variant of the race breakdown) into explicit `Sex`, `Age Group`, and `Race/Ethnicity` columns, verified to produce zero null rates and zero duplicate (Breakdown, Sex, Age Group, Race/Ethnicity, Year) keys across all 2,310 rows. That extract is exported to `suicide_rates_powerbi.xlsx` — a workbook with a live-formula Key Insights sheet (every headline figure recalculates from the Data sheet rather than being hardcoded) — verified with zero formula errors, and cross-checked cell by cell against the pandas-computed figures above.

---

*A note on this dataset's subject matter: this project uses aggregate, publicly reported CDC/NCHS mortality statistics for data-engineering and analysis practice. If you or someone you know is struggling, the 988 Suicide & Crisis Lifeline (call or text 988 in the U.S.) is available 24/7.*
