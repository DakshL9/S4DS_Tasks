# Chicago Crime Dataset — Exploratory Data Analysis

## Dataset Overview

- **120,759 records** across **23 columns** (22 after dropping the redundant `_year` column — confirmed identical to `Year` via `df['Year'].equals(df['_year'])`, which returned `True`)
- Covers reported incidents from **2021–2025**
- 11 numerical columns and 10 text/categorical columns
- No fully duplicate rows found in the dataset

## Data Quality

- **Total missing values: 9,859 cells (0.37% of the dataset)** — a very clean dataset overall
- Missing values are concentrated in location fields:
  - `Latitude`, `Longitude`, `X Coordinate`, `Y Coordinate`, `Location` — each missing **1,877 rows**
  - `Location Description` — missing **463 rows**
  - `Community Area` — missing **11 rows**

### Missing coordinates are not random

Comparing crime types in the missing-coordinate subset vs. the full dataset reveals a clear pattern:

| Crime Type | % of missing-coordinate rows | % of full dataset |
|---|---|---|
| NARCOTICS | 30.7% | 5.7% |
| DECEPTIVE PRACTICE | 24.0% | 5.8% |
| THEFT | 16.0% | 19.6% |
| BATTERY | 5.8% | 18.5% |

**NARCOTICS and DECEPTIVE PRACTICE together account for ~55% of all missing-coordinate rows**, despite making up only ~11.5% of crimes overall — a 4–5x overrepresentation. This suggests the missingness is tied to how these crimes are recorded (e.g. narcotics incidents tied to a stop/arrest rather than a fixed address, and deceptive practice/fraud often lacking a physical crime location) rather than random data loss.

### Placeholder timestamps at exactly 00:00:00

A large number of records had a `Date` time of exactly **00:00:00**. This is very unlikely to reflect genuine crime timing at that volume — it's a common pattern where the true incident time was unknown and defaulted to midnight during data entry, rather than a real behavioral spike in crime at that instant.

- **4,072 rows** had a time of exactly 00:00:00 (**~3.4%** of the dataset)
- These rows were **dropped** before generating the hourly crime distribution (dataset reduced to **116,687 rows**), since including them would have artificially inflated Hour 0 and misrepresented real time-of-day crime patterns
- All other analyses (crime type, arrest rate, location, day-of-week, month) retain these rows, since the date itself (day/month) is still valid — only the *time* portion was a placeholder

## Univariate Analysis

### Crime Type Distribution
Top 5 crime types by volume:
1. **THEFT** — 23,660 incidents
2. **BATTERY** — 22,338 incidents
3. **CRIMINAL DAMAGE** — 11,330 incidents
4. **ASSAULT** — 10,341 incidents
5. **OTHER OFFENSE** — 8,201 incidents

The most common specific descriptions are `SIMPLE` (13,338 — mostly simple assault/battery), `DOMESTIC BATTERY SIMPLE` (10,297), and theft-related descriptions like `$500 AND UNDER` (7,169) and `RETAIL THEFT` (6,204).

### Arrest Distribution
- **False (no arrest): 81,142 (67.2%)**
- **True (arrest made): 39,617 (32.8%)**

About one in three reported incidents results in an arrest.

### Domestic Distribution
- **False: 97,408 (80.7%)**
- **True (domestic incident): 23,351 (19.3%)**

### Location Description Distribution
Top 5 location types where crimes occur:
1. STREET — 33,558
2. APARTMENT — 21,907
3. RESIDENCE — 13,989
4. SIDEWALK — 7,798
5. SMALL RETAIL STORE — 4,070

### District, Ward, and Community Area
- Highest-volume **District**: 11 (8,192 incidents), followed by District 8 (7,509) and District 6 (7,169)
- Highest-volume **Ward**: 28 (5,970 incidents)
- Highest-volume **Community Area**: 25 (6,527 incidents)

### Geographic Distribution
A scatter plot of all incidents by Latitude/Longitude traces the outline of the city of Chicago, with visibly denser clustering along the central and southern portions of the map — confirming that crime incidents are not evenly spread but concentrated in specific corridors of the city.

### Temporal Distribution

**By Hour of Day** (after dropping the 4,072 placeholder 00:00:00 timestamps):
- The chart is now smooth and continuous — the artificial midnight spike is gone, and Hour 0 (~4,300 incidents) sits in line with its neighboring hours
- The **lowest point is around 5 AM** (~1,900 incidents)
- Incidents rise steadily through the morning, peak around **noon (~6,900)**, dip slightly in early afternoon, then hold a plateau of roughly 6,300–6,600 incidents from about 3 PM to 8 PM before declining into the night
- *This confirms the original midnight spike (~8,400 incidents at Hour 0) seen before cleaning was a data artifact from placeholder timestamps, not real crime behavior.*

**By Day of Week:**
- **Friday** is the highest-crime day (~17,200 incidents), followed closely by **Saturday** and **Sunday**
- **Thursday** and **Tuesday** are the lowest (~16,300–16,400 incidents)
- The spread across days is relatively narrow, but there's a mild weekend-leaning pattern (Fri–Sun outpacing the midweek)

**By Month:**
- **July** is the peak month (~10,900 incidents), followed by August and October
- **February** is the lowest month (~8,400 incidents)
- There is a visible summer-to-winter pattern, with warmer months (June–October) consistently higher than winter months (December–February)

## Bivariate / Multivariate Analysis

### Arrest Rate by Crime Type
Arrest likelihood varies dramatically by crime type — from near-certain to near-zero:

**Highest arrest rates:**
| Crime Type | Arrest Rate |
|---|---|
| GAMBLING | 100.0% |
| CONCEALED CARRY LICENSE VIOLATION | 99.2% |
| NARCOTICS | 98.9% |
| PROSTITUTION | 98.9% |
| LIQUOR LAW VIOLATION | 97.8% |

**Lowest arrest rates:**
| Crime Type | Arrest Rate |
|---|---|
| MOTOR VEHICLE THEFT | 9.1% |
| DECEPTIVE PRACTICE | 9.0% |
| KIDNAPPING | 7.1% |
| INTIMIDATION | 3.0% |
| HUMAN TRAFFICKING | 0.0% |

The pattern makes sense structurally: the highest-arrest crimes (gambling, narcotics, prostitution, liquor violations) are typically **caught in the act** during a stop, raid, or sting — the "crime" and the "arrest" often happen at the same moment. The lowest-arrest crimes (motor vehicle theft, deceptive practice, kidnapping, human trafficking) are usually **discovered after the fact**, with the offender long gone by the time it's reported, making arrest far less likely.

### Correlation Matrix (numeric variables)
Computed across `Latitude`, `Longitude`, `X Coordinate`, `Y Coordinate`, `Year`, `Arrest`, and `Domestic`:

- `Latitude` ↔ `Y Coordinate` and `Longitude` ↔ `X Coordinate` correlate at **~1.0** — expected and not a real finding, since X/Y Coordinate is just a projected version of the same Latitude/Longitude values
- `Latitude`/`Longitude` correlate at **~-0.53 to -0.54** with each other (a mild geometric relationship from Chicago's diagonal city shape, not a behavioral insight)
- `Arrest` and `Domestic` both show **near-zero correlation** (roughly -0.05 to 0.01) with location and year — meaning neither where a crime happens nor when (by year) has a meaningful linear relationship with whether an arrest occurs or whether an incident is domestic. This tells us arrest likelihood is driven much more by **crime type** (see above) than by geography or time trend.

### Crime Type Trends by Year (2021–2025)
Tracking the top 6 crime types year over year reveals different trajectories hidden by the overall totals:

- **MOTOR VEHICLE THEFT** shows the most dramatic movement — rising sharply from ~830 incidents (2021) to a peak of ~2,300 (2023), then declining to ~1,370 by 2025
- **THEFT** and **BATTERY** both trend upward across the period, from ~3,300/~4,000 in 2021 to ~5,000/~4,600 by 2025, with THEFT overtaking BATTERY as the top crime by 2024–2025
- **CRIMINAL DAMAGE**, **ASSAULT**, and **OTHER OFFENSE** stay relatively flat across the 5 years, showing no strong directional trend

This shows that the city-wide "crime is up/down" framing hides very different underlying stories per crime type — motor vehicle theft's post-pandemic-era spike and subsequent decline is a standout pattern that a simple total-crime trend line would miss.

### Crime Type by District
A District × Crime Type heatmap (top 8 crime types) highlights how crime *composition* — not just volume — varies sharply by district:

- **District 11** has a strikingly disproportionate concentration of **NARCOTICS** incidents (~2,300) — more than double any other district, and far exceeding what its overall crime volume would suggest
- **Districts 1, 18, and 19** stand out for unusually high **THEFT** counts (~2,000–2,200 each) relative to other crime types in those same districts — consistent with these being downtown/high-foot-traffic areas
- **District 31** shows almost zero incidents across every crime category, a clear anomaly suggesting it may be a non-patrol, administrative, or otherwise inactive designation rather than a genuine low-crime area — worth flagging as a data caveat rather than an operational finding

## Key Takeaways

1. The dataset is largely clean and complete, with under 0.4% of cells missing overall.
2. Missing location coordinates are **not random** — they are concentrated in NARCOTICS and DECEPTIVE PRACTICE incidents, pointing to a structural reason (nature of how these crimes are reported) rather than a data-collection error.
3. Theft and battery dominate the crime-type distribution, together accounting for well over a third of all reported incidents.
4. Only about **1 in 3** reported incidents leads to an arrest overall — but this hides huge variation by crime type: near-100% for gambling/narcotics/prostitution vs. near-0% for human trafficking/intimidation/kidnapping, largely explained by whether the crime is typically caught in the act or discovered after the fact.
5. Crime is geographically concentrated in specific corridors of Chicago rather than evenly distributed, and crime *composition* varies by district — District 11 is a clear narcotics hotspot, while Districts 1, 18, and 19 are theft-heavy.
6. A large number of records (4,072 rows, ~3.4%) had a placeholder timestamp of exactly 00:00:00, which artificially inflated the midnight hour in the raw data. After removing these, the true hourly pattern shows a smooth curve with an early-morning low around 5 AM and sustained afternoon/evening activity — no genuine midnight spike.
7. Friday is the highest-crime day of the week (with the weekend generally outpacing midweek); July is the highest-crime month, pointing to a seasonal summer increase.
8. Numeric correlations confirm that geography and year have little to no linear relationship with arrest or domestic status — arrest outcomes are driven far more by crime type than by where or when the crime occurred.
9. Year-over-year trends reveal diverging patterns by crime type: motor vehicle theft spiked sharply through 2023 before declining, while theft and battery show steady upward trends — a pattern masked by looking at total crime volume alone.

