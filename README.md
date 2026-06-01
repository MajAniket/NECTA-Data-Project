# NECTA CSEE 2025 — Data Pipeline & Analysis
### Tanzania Secondary Education Performance Study
*A Peace Corps research project | Applied Math & Statistics*

---

## Project Overview

This project scrapes, cleans, and analyzes Tanzania's 2025 Certificate of Secondary Education Examination (CSEE) results published by the National Examinations Council of Tanzania (NECTA). It covers **5,821 secondary schools** across all **31 regions** of Tanzania.

Three research questions guide the analysis:

- **RQ1**: Does exam failure cluster geographically in ways that suggest infrastructure or teacher deployment explains outcomes, rather than student ability?
- **RQ2**: What is the gender gap in Division I attainment, and does it vary by region?
- **RQ3**: Are there schools with zero Division I or II passes — and where are they concentrated?

A fourth question — whether school ownership type (government vs. private vs. religious mission) predicts performance after controlling for region — is scoped for a future stage requiring manual school classification.

---

## Understanding the NECTA Grading System

Tanzania's CSEE uses a Division system where **lower is better**.

Students earn letter grades per subject based on exam percentage:

| Grade | Range | Points |
|-------|-------|--------|
| A | 75–100% | 1 |
| B | 65–74% | 2 |
| C | 45–64% | 3 |
| D | 30–44% | 4 |
| F | 0–29% | 5 |

A student's aggregate score is the sum of their best subject points. Division is assigned from that aggregate:

| Division | Meaning | Aggregate |
|----------|---------|-----------|
| I | Excellent | 7–17 |
| II | Very Good | 18–21 |
| III | Good | 22–25 |
| IV | Satisfactory | 26–33 |
| 0 | Fail | 34+ or incomplete |

**School GPA** is the mean aggregate across all students. Lower = better. A school GPA of 1.0 is exceptional; 5.0 is near-total failure. Unlike a US GPA, higher numbers are worse.

---

## Data Source

**Base URL**: `https://matokeo.necta.go.tz/results/2025/csee/`

Individual school pages follow the pattern:
`https://matokeo.necta.go.tz/results/2025/csee/results/sXXXX.htm`

Each page contains:
- A division summary table broken down by sex (Female / Male / Total)
- School metadata: region, GPA, registered students, absent, sat

The school index is split across 26 alphabetical sub-pages:
`indexfiles/index_a.htm` through `index_z.htm`

---

## Repository Structure

```
necta_scrape/
├── raw_html_schools/         # Cached HTML pages (~5,800 files, one per school)
├── schools.csv               # School list: code, name, URL
├── necta_2025_schools.csv    # Raw scraped data (one row per school, 26 columns)
├── necta_2025_regions.csv    # Aggregated data by region (31 rows)
└── checkpoint_*.csv          # Intermediate saves during scraping (every 200 schools)
```

---

## Notebooks

### Notebook 1: Scraper

Run in **Google Colab**. All output saves directly to Google Drive to survive runtime resets — Colab periodically wipes local storage, so writing to Drive is essential for a multi-hour scrape. I also wanted to stick to online work, as it gets easier.

**Libraries used:**
- `requests` — fetches web pages programmatically
- `beautifulsoup4` — parses HTML into navigable structure
- `pandas` — DataFrame construction and CSV output
- `time` — polite delays between requests (1.5s) to avoid overloading NECTA's server
- `re` — regular expressions for text pattern matching

**Key implementation notes:**

*Index page quirks:* NECTA's alphabetical index pages use Windows-style backslashes in hrefs (e.g. `..\results\s0101.htm`). These must be normalized to forward slashes before URL construction. Link text format is `SCHOOL NAME - S1234` (name first, code at end) — the opposite of what you might expect.

*Parser design:* Each school page has two data sources: (1) an HTML table for division counts by sex, and (2) plain text for region, GPA, and enrollment. The enrollment regex matches the full header row `REGIST ABSENT SAT WITHHELD NO-CA CLEAN DIV I...` before capturing numbers — an earlier version using positional indexing produced shifted values (registered=0, absent=121, sat=6 instead of registered=121, absent=6, sat=115).

*Caching:* Raw HTML is saved to `raw_html_schools/` before parsing. Re-runs skip already-cached pages, so a reset mid-scrape resumes from where it left off rather than re-fetching everything.

*Checkpointing:* The results DataFrame is saved to Drive every 200 schools during the main loop. A runtime reset loses at most 200 schools of parsed data.

**Runtime:** ~3-4 hours for 5,800+ schools at 1.5s delay.

---

### Notebook 2: Analysis & Visualization

**Derived variables computed at school level:**

| Variable | Formula | Research Question |
|----------|---------|-------------------|
| `pass_rate` | (Div1+2+3+4) / total_students | RQ1 |
| `fail_rate` | Div0 / total_students | RQ1 |
| `div1_rate` | Div1 / total_students | RQ2 |
| `top_rate` | (Div1+Div2) / total_students | RQ3 |
| `is_zero_top` | 1 if Div1+Div2 == 0 | RQ3 |
| `f_div1_rate` | f_div1 / f_total | RQ2 |
| `m_div1_rate` | m_div1 / m_total | RQ2 |
| `div1_gender_gap` | m_div1_rate − f_div1_rate | RQ2 (positive = boys outperform) |
| `fail_gender_gap` | m_fail_rate − f_fail_rate | RQ2 supplementary |
| `absent_rate` | absent / registered | Supplementary |
| `is_anomaly` | 1 if div1_rate > 0.95 | Data quality flag |

**Regional aggregation:** Schools are grouped by region using `pandas.groupby`. The region field is scraped directly from each school's results page (matching `EXAMINATION CENTRE REGION` in the page text), so no external lookup table is required.

**Visualization:** A 2×2 heatmap grid built with `matplotlib` and `seaborn` displays all four metrics — GPA, pass rate, zero-top rate, and gender gap — sorted by GPA and color-coded with directionally appropriate scales (`RdYlGn_r` for GPA where lower is better, `RdYlGn` for pass rate where higher is better).

---

## Key Findings (2025 Data)

**RQ1 — Geography:** The best regional GPA is 3.112 (Arusha); the worst is 3.833 (Kusini Pemba). The entire country sits between a C and D average. Geographic clustering is weak on the mainland — better and worse regions are interspersed — suggesting the binding constraint varies within regions rather than tracking cleanly to geography or climate. Zanzibar is the clear exception, clustering at the bottom across all metrics. Kilimanjaro's strong performance likely reflects its century-long history of mission school investment rather than geographic advantage.

**RQ2 — Gender gap:** A consistent male advantage in Division I attainment exists across all regions, with larger gaps in Simiyu (8.6pp), Geita (7.6pp), Mbeya (7.3pp), and Manyara (6.5pp). Zanzibar shows near-zero gender gaps despite conservative cultural context — possibly a selection effect, where fewer girls advance past Standard VII, making the CSEE cohort more able on average.

**RQ3 — Zero-top schools:** Kusini Pemba leads at 34.7% — more than 1 in 3 schools produced no Division I or II students in 2025. All five highest-rate regions are Zanzibari. On the mainland, Tanga (7.4%) and Morogoro (5.4%) are the highest, but the Zanzibar-mainland gap is large.

---

## Known Issues & Limitations

**Anomalous schools:** A small number of schools have Division I rates above 95%, which is implausible for a normal school population. These are flagged with `is_anomaly = 1`. Most are likely elite national schools (e.g. seminaries with highly selected intake) rather than data errors, but should be verified before inclusion in regression models.

**Enrollment mismatch:** `total_students` (sum of division counts) sometimes differs slightly from `sat` (enrollment table). Students who sat but were disqualified or had results withheld appear in `sat` but not in division counts. Use `total_students` as the denominator for all performance rate calculations.

**SONGWE region:** Songwe was split from Mbeya in 2016 and is absent from most GeoJSON boundary files. It is kept separate in all school-level analysis but merged with Mbeya for geographic mapping.

**Cross-sectional limitation:** All findings are from 2025 only. Regional differences could reflect decades of differential investment, demographic composition, or recent policy changes. Causal interpretation requires multi-year panel data.

**Ownership type uncoded:** School names contain strong signals — "ST.", "LUTHERAN", "SEMINARY", "ISLAMIC", "ADVENTIST" indicate mission/religious affiliation; plain geographic names suggest government schools. Coding ~5,800 names for ownership is required before running the ownership-type analysis (planned for Stage 2).

---

## Geographic Name Reconciliation

The geoBoundaries GeoJSON uses English names in title case; NECTA data uses Swahili names in uppercase. Mapping applied:

| NECTA (Swahili) | GeoJSON (English) |
|----------------|------------------|
| KASKAZINI UNGUJA | Zanzibar North |
| KUSINI UNGUJA | Zanzibar South & Central |
| MJINI MAGHARIBI | Zanzibar Urban/West |
| KASKAZINI PEMBA | North Pemba |
| KUSINI PEMBA | South Pemba |
| SONGWE | Mbeya (merged — not in GeoJSON) |

---

## Next Steps

- [ ] Stage 2: Code school ownership type from school names; run ownership × region regression
- [ ] Stage 3: Add 2023 and 2024 data using the same pipeline with year parameter
- [ ] Stage 4: Student-level scrape for subject-specific analysis (math vs. other subjects)
- [ ] Visualization: Finalize interactive choropleth map for Substack embed
- [ ] Writing: Publish findings in *The Quest for Agency*

---

## Dependencies

```
requests
beautifulsoup4
pandas
numpy
matplotlib
seaborn
plotly
```

All pre-installed in Google Colab. No local setup required.

---

## Data License & Ethics

NECTA publishes these results publicly. This project does not collect or publish student-level data (candidate numbers are not retained). All analysis is at school and region level only. No Peace Corps contacts, relationships, or institutional access were used in any part of data collection or analysis.

---

*Last updated: June 2026*
