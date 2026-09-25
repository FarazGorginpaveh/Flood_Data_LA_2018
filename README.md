# Louisiana 2018 Flood Characteristics Dataset (Parish and ZCTA level)

A step-by-step pipeline that measures **how often, how strongly, how long and how severely** every Louisiana parish (county) and ZCTA (ZIP Code Tabulation Area) flooded in 2018, per day and per event.

The idea is inspired by Khajehei et al. (2020), *Scientific Reports* 10:448, [doi:10.1038/s41598-019-57349-z](https://www.nature.com/articles/s41598-019-57349-z), which described flash floods by **frequency, magnitude, duration (rise time) and severity**.

---

## Contents

1. [The idea in one minute](#1-the-idea-in-one-minute)
2. [Folder structure](#2-folder-structure)
3. [Setup](#3-setup)
4. [Run order](#4-run-order)
5. [The steps, one by one](#5-the-steps-one-by-one)
6. [Definitions used everywhere](#6-definitions-used-everywhere)
7. [Column guide for the final tables](#7-column-guide-for-the-final-tables)
8. [Key results for 2018](#8-key-results-for-2018)
9. [Limitations](#9-limitations)
10. [Using the pipeline for other states or years](#10-using-the-pipeline-for-other-states-or-years)
11. [Troubleshooting](#11-troubleshooting)
12. [Data sources](#12-data-sources)
13. [Glossary](#13-glossary)

---

## 1. The idea in one minute

- **Measured data (USGS gauges)** is accurate but only exists at about 200 points; 23 of the 64 parishes have no flow gauge at all.
- **A model (NOAA National Water Model, NWM)** gives hourly river flow for all 43,240 stream reaches in Louisiana, so every parish has data.
- For every reach we compare 2018 flow with the reach's own **2-year flow** (a flow reached in about half of all years). Going above it = a flood.
- We **check the model** against USGS gauges and NOAA flood reports, **remove problem areas** (tidal marsh, lakes, pumped bayous), give each parish a **confidence level**, and then **summarize** everything per parish and per ZCTA.
- Final products: daily tables, event tables, yearly summaries, storm IDs, event types, return periods and observed/modeled flags.

```
USGS gauges ─┐
             ├─► flood events ─► checks & clean-up ─► parish tables ─► ZCTA tables ─► storms & types
NWM model ───┘                    (confidence)        (validated)
```

---

## 2. Folder structure

Project root: `D:\Tufts\Wishlist\Sepideh_2\`

```
Sepideh_2\
├── scripts\                         (the Python scripts listed below)
└── data\
    ├── usgs\          step 01-02   gauge data, peaks, thresholds, parish outlines
    │   └── continuous\             15-min data, one file per gauge and parameter
    ├── nwm\           step 03      NWM reaches, 2018 hourly flow, 43-yr maxima, thresholds
    ├── events\        step 05      flood events for every reach and gauge
    ├── evaluation\    step 06      gauge grades, parish confidence, reach flags
    ├── final\         step 07      MAIN PARISH DATASET
    ├── diagnostics\   step 07b-c   long-block checks and the reviewed exclusion list
    ├── stormevents\   step 08      cached NOAA Storm Events file
    ├── validation\    step 08      comparison with NOAA flood reports
    ├── zcta\          step 09-10   ZCTA DATASET, storm table
    │   └── census\                 cached Census ZCTA files
    └── hurdat\        step 10      cached hurricane track file
```

Every folder has a `figures\` subfolder for its plots.

---

## 3. Setup

**Python packages** (Anaconda is fine):

```
pip install requests pandas pyarrow tqdm matplotlib numpy
pip install "xarray[io]" zarr s3fs fsspec dask
pip install geopandas
```

**USGS API key** (free, needed for step 01):

1. Sign up at <https://api.waterdata.usgs.gov/signup/>.
2. In Windows Command Prompt: `setx USGS_API_KEY "your_key"`, then close and reopen the terminal (and Spyder/VS Code).
   Or paste the key into the `API_KEY = ...` line of `01_download_usgs_streamflow.py`.

No account is needed for NWM (public AWS bucket), NOAA Storm Events, Census files or hurricane tracks.

**Paths:** each script has `PROJECT_DIR = Path(r"D:\Tufts\Wishlist\Sepideh_2")` near the top. Change it once if you move the project.

---

## 4. Run order

| Order | Script | Time (approx.) | Needs |
|---|---|---|---|
| 1 | `01_download_usgs_streamflow.py` | 15-30 min | API key |
| 2 | `02_check_usgs_data.py` | 1 min | step 01 |
| 3 | `03_download_nwm_retrospective.py --plan-only` | 1 min | internet |
| 4 | `03_download_nwm_retrospective.py` | hours (overnight) | internet |
| 5 | `05_extract_flood_events.py` | a few min | steps 01, 03 |
| 6 | `06_evaluate_nwm.py` | 1 min | steps 03, 05 |
| 7 | `07_parish_dataset.py` | 1-2 min | step 06 |
| 8 | `07b_diagnose_long_blocks.py` | 1-3 min | step 07 |
| 9 | `07c_make_reviewed_exclusions.py` | seconds | step 07b |
| 10 | `07_parish_dataset.py` (again, now with exclusions) | 1-2 min | step 07c |
| 11 | `08_validate_storm_events.py` | 1 min | step 07 |
| 12 | `09_zcta_dataset.py` | 2-5 min | steps 07, 08 |
| 13 | `10_enrich_zcta_events.py` | 1-3 min | step 09 |

All download steps are **resumable**: if a script stops, run it again and it continues where it left off. Step numbers skip 04 on purpose (see below).

---

## 5. The steps, one by one

Each step has: **Goal** (one line), **What it does** (simple), **Outputs**, and **More detail**.

### Step 01 - Download USGS streamflow
`01_download_usgs_streamflow.py`

**Goal:** get real, measured river data for Louisiana.

**What it does**
1. Finds every Louisiana surface-water monitoring location (streams, canals, tidal streams, estuaries).
2. Reads which data series each location has, plus stored flood thresholds.
3. Downloads **15-minute** discharge (flow, code 00060) and gage height (water level, code 00065) for **1 Dec 2017 - 31 Jan 2019** (2018 plus one month on each side, so floods crossing New Year are complete).
4. Downloads daily mean discharge.
5. Downloads each gauge's **annual peak flows** for its whole record (often decades).
6. Counts how many gauges each parish has.

**Outputs** (`data\usgs\`)

| File | What it is |
|---|---|
| `sites_LA.csv` | all 3,184 Louisiana surface-water locations, with lat/lon and parish FIPS |
| `timeseries_metadata_LA.csv` | which data series exist at each location |
| `thresholds_LA.csv` | reference levels stored by USGS (incl. NWS flood stage for some gauges) |
| `continuous\USGS-xxxx_00060.parquet` | 15-min discharge (cfs) per gauge |
| `continuous\USGS-xxxx_00065.parquet` | 15-min gage height (ft) per gauge |
| `continuous_manifest.csv` | what was downloaded, record length, time step |
| `daily_discharge_LA.parquet` | daily mean discharge |
| `annual_peaks_LA.csv` | yearly peak flows (full record) |
| `parish_coverage.csv` | gauges per parish |
| `download_log.txt` | log of the run |

**More detail**
- Uses USGS's **new Water Data API** (`api.waterdata.usgs.gov`). The old NWIS "waterservices" system is being retired in early 2027, so code built on it would stop working.
- 15-minute data is needed because the paper's **duration is rise time in hours**; daily means cannot show it.
- A free API key avoids the very strict anonymous rate limit (HTTP 429 errors).
- Daily discharge is downloaded in small groups of sites, because one big statewide query timed out on the USGS server.

### Step 02 - Check the USGS data
`02_check_usgs_data.py`

**Goal:** find out what USGS alone can and cannot support.

**What it does:** counts and maps gauges, checks how complete each 2018 record is, checks which gauges have a usable flood threshold, and (with the local 15-min files) shows which gauges went above NWS flood stage in 2018.

**Outputs** (`data\usgs\`): `data_check_report.txt`, `la_parishes.geojson` (parish outlines, reused by later steps), `figures\fig1_gauge_maps.png`, `fig2_data_quality.png`, `fig3_support_matrix.png`, and (if the 15-min files are present) `fig4_2018_flood_stage.png` + `flood_stage_exceedance_2018.csv`.

**More detail - what we learned**
- 203 gauges had 15-min data in 2018, but only **73 measure flow**; 130 measure water level only.
- **23 parishes had no flow gauge**, **15 had no gauge at all**.
- Only 40 of 202 stage gauges had an NWS flood stage; USGS does not store the paper's "action stage" at all.
- 68 of 73 flow gauges had at least 10 years of annual peaks, enough for a statistical flood threshold.
- Conclusion: USGS alone cannot cover every parish, so a model was needed.

### Step 03 - Download the National Water Model (NWM v3.0 retrospective)
`03_download_nwm_retrospective.py`

**Goal:** get river flow for every stream in Louisiana, including streams without gauges.

**What it does**
1. **Step 0 (plan):** prints how many GB each step will read (`--plan-only`).
2. **Step 1:** lists every NWM river reach inside Louisiana (43,240) and gives each one its parish (point-in-polygon), stream order, elevation and USGS gage ID if any.
3. **Step 2:** downloads **hourly flow** for all these reaches, 1 Dec 2017 - 31 Jan 2019.
4. **Step 3:** for every reach, finds the **highest flow of each water year 1980-2022** (43 years) and computes flood thresholds.
5. **Step 4:** matches NWM reaches to USGS gauges and compares 2018 hydrographs.

**Outputs** (`data\nwm\`)

| File | What it is |
|---|---|
| `reaches_LA.csv` | all Louisiana reaches: feature_id, lat/lon, parish, order, elevation, gage_id |
| `nwm_LA_hourly_2018.zarr` | hourly flow (m³/s), Dec 2017 - Jan 2019 |
| `annual_max_parts\wyYYYY.npy` | one file per water year (makes the step resumable) |
| `annual_max_LA.parquet` | yearly maximum flow per reach, 1980-2022 |
| `reach_thresholds_LA.csv` | per reach: 2-yr flow (median of annual maxima), Gumbel 2/5/10/25-yr flows, mean and std of annual maxima |
| `usgs_nwm_crosswalk_LA.csv` | NWM reach ↔ USGS gauge |
| `nwm_vs_usgs_2018_stats.csv` | per gauge: hourly correlation, peak ratio, peak timing |
| `figures\fig5_nwm_reaches.png`, `fig6_nwm_vs_usgs_2018.png` | reaches per parish, 2-yr flow map, hydrographs |

**More detail**
- Source: `s3://noaa-nwm-retrospective-3-0-pds/CONUS/zarr/chrtout.zarr` (public, no account). It covers Feb 1979 - Jan 2023, hourly, about 2.7 million reaches in the US.
- **Why 43 years were downloaded when we only study 2018:** to know what is "normal" for each reach. The **2-year flow = median of the 43 yearly maxima**. It is exceeded in about half of all years (roughly bankfull). Only one number per reach per year was saved, so the files stay small even though the whole record was read.
- Using the model's own history for the threshold means model bias mostly cancels: if NWM runs 20% high at a stream, both the 2018 peak and its 2-yr flow are 20% high.
- The retrospective has **no data assimilation** (it is not corrected with gauge data), which is why steps 06-08 check it.
- Result of the first check (53 gauges): median hourly correlation **0.83**, median peak ratio **0.94**.
- The 43 years include water year 2018 itself; with 43 years this barely changes the median, but it can be removed if a strict baseline is needed.

### (Step 04 - not used)
Planned: download NWS action / flood stages for the USGS stage gauges (the paper's event definition). It was not needed once NWM gave a model-consistent threshold for every reach. It remains an option to compare with the paper's method.

### Step 05 - Find the flood events
`05_extract_flood_events.py`

**Goal:** turn river flow into a list of floods, with the paper's four measures.

**What it does:** for every NWM reach and every USGS gauge with a threshold, it finds each period in 2018 when flow (or stage) went **above the threshold** and records start, peak and end.

| Source | Threshold |
|---|---|
| NWM reach | the reach's own 2-yr flow (from step 03) |
| USGS discharge gauge | median of the gauge's annual peaks (at least 10 years) |
| USGS stage gauge | NWS flood stage (when USGS stores it) |

**Outputs** (`data\events\`)

| File | What it is |
|---|---|
| `events_nwm_2018.parquet` | one row per reach-event (about 34,000 in 2018) |
| `events_usgs_2018.parquet` | one row per gauge-event (discharge and stage) |
| `reach_summary_2018.csv` | every reach, incl. those with **zero** events (needed for frequency) |
| `figures\fig7_events_2018.png` | flooded reaches, share of each parish flooded, month of peaks, rise times |
| `events_log.txt` | log |

**More detail**
- Two exceedances less than **24 h** apart are merged into one event.
- Reaches with a 2-yr flow below **0.1 m³/s** (dry ditches) or without a threshold are skipped.
- Reaches with a 2-yr flow above **3,000 m³/s** are flagged `big_river` (Mississippi, Atchafalaya, lower Red).
- USGS 15-min data is averaged to hourly so it is comparable with NWM.
- Events are kept when their **peak** falls in 2018 (local time, America/Chicago). Events already above threshold at the start or end of the window are flagged as censored.
- Sanity check: 18,686 of 35,504 usable reaches (**53%**) flooded at least once. For a 2-yr threshold about 50% is expected in a normal year, so the result is realistic.

### Step 06 - How much can we trust NWM in each parish?
`06_evaluate_nwm.py`

**Goal:** give every parish a confidence level (high / medium / low / unverified).

**What it does**
1. **Grades each USGS gauge that sits on an NWM reach:**
   - hourly correlation r ≥ 0.75 → good, 0.50-0.75 → fair, < 0.50 → poor;
   - one level lower if NWM and USGS disagree on whether 2018 was a flood year there, or if NWM catches fewer than half of the USGS events;
   - NWM peak < 1/20 or > 20× USGS → **crosswalk error** (gauge linked to the wrong NWM channel), not used.
2. **Rates each parish:** from its own gauges (good = 1, fair = 0.5, poor = 0); if it has none, from the 3 nearest gauges within 75 km, but then at most "medium"; none nearby → "unverified". Score ≥ 0.75 high, ≥ 0.40 medium, else low.
3. **Flags problem reaches:** above threshold for more than 2,000 h (lakes, backwater) → excluded; peak more than 10× the 2-yr flow (tiny streams) → capped later.

**Outputs** (`data\evaluation\`): `gauge_evaluation_2018.csv`, `matched_events_2018.csv`, `parish_confidence_2018.csv`, `reach_flags_2018.csv`, `evaluation_summary.txt`, `figures\fig8_nwm_evaluation.png`.

**More detail - result (preview run)**
- Gauges: 31 good, 9 fair, 7 poor, 6 crosswalk errors.
- Parishes: **21 high** (north and west, Florida Parishes), **24 medium**, **16 low** (delta, coast, parts of the northeast), **3 unverified** (Cameron, Plaquemines, St. Bernard).
- Why the south is weak: NWM does not simulate tides, storm surge, levees, river diversions or pumped drainage.

### Step 07 - Build the parish dataset (main result)
`07_parish_dataset.py`

**Goal:** one clean table of flooding per parish, per day and per event.

**What it does**
- **Flood day:** at least **3** reaches and at least **1%** of the parish's reaches above their 2-yr flow on that local day.
- **Flood event:** flood days separated by at most **1** dry day.
- **Source per parish:** high / medium → NWM; low → USGS gauges if the parish has usable ones (not tidal, regulated or big-river), otherwise NWM (still marked low); unverified → **no data** (left empty, not zero).
- **Clean-up:** removes the reaches and gauges in `data\diagnostics\reviewed_exclusions.csv` (from steps 07b-07c) and tidal/regulated gauges (names with canal, outlet, pump, GIWW, flood gate, estuary site types).
- **Big rivers** (Mississippi, Atchafalaya, lower Red) are reported in a separate column, not mixed with local floods.
- Peak ratios are capped at **10**; events longer than **30 days** are flagged.

**Outputs** (`data\final\`)

| File | Rows |
|---|---|
| `parish_daily_2018.csv` | 64 parishes × 365 days = 23,360 |
| `parish_events_2018.csv` | one per parish flood event (195 in the final run) |
| `parish_annual_2018.csv` | 64, one per parish |
| `figures\fig9_parish_maps_2018.png` | 4 maps: frequency, magnitude, duration, severity |
| `figures\fig10_parish_calendar_2018.png` | every parish × every day of 2018 |

**More detail**
- All settings (`MIN_REACHES`, `MIN_FRAC`, `MERGE_DAYS`, `RATIO_CAP`, `INCLUDE_BIG_RIVERS`, `LONG_EVENT_DAYS`) are at the top of the script.
- Read the daily table with `pd.read_csv("parish_daily_2018.csv", dtype={"flood_day": "boolean"})` so empty values stay empty.
- The calendar reproduces the **late-February 2018** floods (state of emergency for Avoyelles, Beauregard, Bossier, Caddo, Grant, Morehouse, Natchitoches, Ouachita and Rapides) and the **very wet December** in northwest Louisiana.

### Step 07b - Diagnose long flood blocks
`07b_diagnose_long_blocks.py`

**Goal:** tell real slow floods apart from errors.

**What it does**
1. Finds every run of parish flood days lasting **21+ days** (gaps up to 3 dry days allowed).
2. Lists the reaches or gauges that flood during it and marks **suspects**:
   - NWM reach: one event ≥ 21 days or flooding on ≥ 80% of the block's days (**stuck**), or elevation below 2 m (**coastal**);
   - USGS gauge: tidal/regulated name or site type, or big river.
3. **Removes the suspects and recounts:** if at least 50% of the flood days disappear, the block was caused by those few reaches → "likely artifact"; otherwise "widespread (likely real)" or "check hydrographs".

**Outputs** (`data\diagnostics\`): `long_blocks_2018.csv` (one row per block + verdict), `long_block_contributors.csv`, `suggested_exclusions.csv`, `figures\diag_<parish>_<date>.png` (map of contributing reaches + hydrographs of the top 3 with their threshold).

**More detail - what the review found**

| Parish | Cause | Decision |
|---|---|---|
| Cameron, Plaquemines | coastal marsh at ~0 m elevation | exclude |
| St. Tammany (Mar-May) | Pearl River mouth below sea level, tidal | exclude |
| Vernon (Feb-Apr) | 17 reaches at almost the same elevation = a lake surface | exclude |
| Ouachita (Apr-May) | 4 tiny headwater reaches stuck for 38 days | exclude |
| **Caldwell** (Feb-Mar) | the **real Ouachita River flood** (order-7 main river) | **keep** - the rule was wrong |
| **Ascension** | Bayou Lafourche gauges, pump-fed from the Mississippi | **exclude** - the rule missed it |
| St. Mary (Mar-May) | Atchafalaya / Wax Lake high water | move to big-river category |
| Bossier, Caddo, Red River, Webster... | real Feb-Mar and December floods | keep |

### Step 07c - Save the reviewed exclusion list
`07c_make_reviewed_exclusions.py`

**Goal:** turn the hand review above into the file that step 07 reads.

**What it does:** starts from `suggested_exclusions.csv`, keeps the Caldwell reaches, moves the Atchafalaya/Wax Lake gauges to the big-river category, adds the two Bayou Lafourche gauges, and saves the result.

**Output:** `data\diagnostics\reviewed_exclusions.csv` (742 rows: 736 NWM reaches excluded, 3 USGS gauges dropped, 3 moved to big-river).

Then **run step 07 again**. Its first line must say `exclusions: 736 NWM reaches, 3 USGS gauges dropped, 3 USGS gauges moved to big-river category`.

### Step 08 - Check against NOAA flood reports
`08_validate_storm_events.py`

**Goal:** independent proof that the dataset catches real floods.

**What it does**
1. Downloads the 2018 **NOAA NCEI Storm Events** file (newest version found automatically) and keeps Louisiana.
2. Uses "Flash Flood" and "Flood" reports; counts coastal flood / surge and heavy-rain reports separately.
3. Compares in both directions with a ±1 day tolerance:
   - **detection rate:** share of reported parish floods that the dataset also flagged;
   - **confirmation rate:** share of the dataset's events that have a matching report;
   - **day level:** POD, FAR, CSI, frequency bias.
4. Repeats for local floods only and local + big-river floods; splits by flood type and confidence level.

**Outputs** (`data\validation\`): `validation_summary.txt`, `validation_summary_2018.csv`, `storm_events_LA_2018.csv`, `reported_floods_matched_2018.csv`, `our_events_confirmed_2018.csv`, `validation_by_parish_2018.csv`, `validation_by_confidence_2018.csv`, `figures\fig11_validation.png`.

**More detail:** see the results in [section 8](#8-key-results-for-2018). A low confirmation rate is expected: Storm Events only lists floods that caused impacts and were reported, while a 2-yr threshold also catches many small rural floods. **Detection rate is the key number.**

### Step 09 - The ZCTA dataset
`09_zcta_dataset.py`

**Goal:** the same results at ZIP-code-area (ZCTA) level.

**What it does**
1. Downloads (once) the 2020 Census ZCTA boundaries, the ZCTA-to-county relationship file and the urban-area-to-ZCTA file.
2. Places every usable NWM reach in its ZCTA (same clean-up as step 07).
3. Computes flood days, events and yearly values per ZCTA:
   - flood day: at least **2** reaches and at least **2%** of the ZCTA's reaches;
   - fewer than **5** usable reaches, or main parish unverified → **no data**.
4. Confidence: taken from the parish holding most of the ZCTA's land, **one level lower if more than half of the ZCTA is urban**.
5. Validation: flood reports that have coordinates are placed in a ZCTA and checked in the same ZCTA and in the ZCTA or its neighbours.

**Outputs** (`data\zcta\`): `zcta_daily_2018.csv`, `zcta_events_2018.csv`, `zcta_annual_2018.csv`, `zcta_summary_2018.txt`, `zcta_validation_reports_2018.csv`, `zcta_reaches.csv` and `zcta_boundaries_LA.gpkg` (used by step 10), `figures\fig12_zcta_maps_2018.png`, `fig13_zcta_reliability.png`, and `census\` (cached files).

**More detail**
- ZCTAs are smaller than parishes, so results are more detailed but less certain, especially in cities (pipes, canals and pumps are not in NWM).
- Keep `zcta` as **text** when reading files (`dtype={"zcta": str}`) so codes keep all 5 digits.
- ZCTAs approximate ZIP code areas; they are not identical to postal ZIP codes.

### Step 10 - Storm IDs, event types, return periods, observed/modeled flags
`10_enrich_zcta_events.py`

**Goal:** add the extra fields requested for the ZCTA events.

**What it does**
1. **Return period** of every reach peak (e.g. "a 10-year flood"), from a Gumbel fit to the reach's 43 yearly maxima; capped at 100 years.
2. **Observed magnitude:** if a usable USGS gauge inside the ZCTA flooded during the event, its peak ratio is the observed value.
3. **Links NOAA flood reports** (same or neighbouring ZCTA for reports with coordinates, same parish otherwise).
4. **Storm ID:** ZCTA events in **neighbouring ZCTAs with peaks ≤ 2 days apart**, or linked to the same NWS episode, get the same `storm_id`.
5. **Event type** per storm, from NOAA HURDAT2 hurricane tracks (a tropical cyclone within 300 km) and the NWS reports.

**Outputs** (`data\zcta\`): `zcta_events_2018_enriched.csv`, `storms_2018.csv`, `figures\fig14_storms_2018.png`.

**More detail**
- Event types: `tropical cyclone`, `heavy rainfall - flash flooding`, `heavy rainfall - river flooding`, `heavy rainfall - not reported`. The column `event_type_basis` says what each label is based on.
- Coastal flooding and big-river flooding are **flags** only (`coastal_flood_reported`, `big_river_flooding`), because the magnitudes do not include them.
- If storms chain into very long storms, lower `STORM_PEAK_GAP` from 2 to 1.
- Flood **depth** and **flooded area** are not yet included (they need a flood inundation model; planned next stage).

---

## 6. Definitions used everywhere

| Term | Definition | Unit |
|---|---|---|
| **Threshold (2-yr flow)** | median of the 43 yearly maximum flows (1980-2022) of a reach; for USGS gauges, median of the gauge's annual peaks | m³/s |
| **Flood (reach / gauge)** | flow above the threshold; exceedances < 24 h apart are one event | - |
| **Magnitude** | event peak ÷ 2-yr flow (`peak_ratio`, capped at 10); raw peak also kept | ratio (m³/s) |
| **Duration (paper's)** | rise time = time of peak − time the threshold was first exceeded | hours |
| **Length** | number of days of a parish / ZCTA event | days |
| **Severity** | magnitude ÷ rise time (`severity_norm`) | per hour |
| **Frequency** | number of parish (or ZCTA) flood events in the year | count |
| **Return period** | how rare the peak is, e.g. 10 = expected once every 10 years (Gumbel fit, capped at 100) | years |
| **Flood day (parish)** | ≥ 3 reaches and ≥ 1% of reaches above threshold that local day | yes/no/empty |
| **Flood day (ZCTA)** | ≥ 2 reaches and ≥ 2% of reaches above threshold that local day | yes/no/empty |
| **Flood event (parish / ZCTA)** | flood days separated by ≤ 1 dry day | - |
| **Storm** | ZCTA events in neighbouring ZCTAs with peaks ≤ 2 days apart (or same NWS episode) | - |
| **Local time** | all dates are in America/Chicago; raw times are stored in UTC | - |

---

## 7. Column guide for the final tables

Empty cells mean **no reliable data** (not zero). FIPS and ZCTA codes should be read as text.

### 7.1 `data\final\parish_daily_2018.csv` - one row per parish per day

**Main columns (the "best answer")**

| Column | Meaning |
|---|---|
| `parish_fips`, `parish_name` | county FIPS code (e.g. 22033) and name |
| `date` | local calendar day |
| `confidence` | high / medium / low / unverified (from step 06) |
| `source` | `NWM` (modeled), `USGS` (observed gauges), `none` (no reliable data) |
| `flood_day` | True / False / empty |
| `magnitude_peak_ratio` | largest peak ÷ 2-yr flow among events peaking that day |
| `duration_rise_h` | median rise time (h) of events peaking that day |
| `severity_norm` | largest peak ratio ÷ rise time of events peaking that day |
| `events_peaking` | number of reach/gauge events peaking that day |
| `big_river_flood_day` | Mississippi / Atchafalaya / lower Red flooding in the parish that day (separate) |

**Supporting columns**
- `nwm_*`: the NWM values behind the main columns - reaches flooding (`nwm_reaches_flooding`), total usable reaches (`nwm_n_reaches`), share flooding (`nwm_frac_flooding`), max/median peak ratio, max peak (m³/s), rise times, severities, events starting that day, and `nwm_flood_day`.
- `usgs_*`: the gauge values - gauges flooding, events peaking, max peak ratio, rise time, severity, stage above flood stage (ft), usable gauges in the parish (`usgs_n_gauges`), big-river gauges flooding, and `usgs_flood_day`.
- `big_river_reaches_flooding`: number of big-river NWM reaches above threshold.

### 7.2 `data\final\parish_events_2018.csv` - one row per parish flood event

| Column | Meaning |
|---|---|
| `event_id` | `<parish_fips>_2018_<nn>` |
| `start_date`, `end_date`, `peak_day` | event dates; peak day = day with the most reaches/gauges flooding |
| `length_days`, `n_flood_days` | calendar length; number of flood days inside it |
| `max_extent`, `max_frac_reaches` | most reaches (or gauges) flooding on one day; as a share of the parish's reaches |
| `n_reach_or_gauge_events` | reach/gauge events peaking inside the event |
| `magnitude_max_peak_ratio`, `magnitude_max_peak_cms` | largest peak ÷ 2-yr flow; largest peak in m³/s |
| `duration_median_rise_h`, `duration_max_rise_h` | rise time (paper's duration) |
| `severity_max_norm`, `severity_median_norm` | peak ratio ÷ rise time |
| `big_river_days` | days of the event with big-river flooding |
| `long_event` | True if longer than 30 days |
| `source`, `confidence` | as in the daily table |

### 7.3 `data\final\parish_annual_2018.csv` - one row per parish

| Column | Meaning |
|---|---|
| `frequency_events` | number of parish flood events in 2018 (empty for no-data parishes) |
| `flood_days` | number of flood days |
| `magnitude_mean_peak_ratio`, `magnitude_max_peak_ratio` | mean / max event magnitude |
| `duration_median_rise_h` | typical rise time |
| `severity_mean_norm`, `severity_max_norm` | mean / max event severity |
| `n_long_events` | events longer than 30 days |
| `big_river_flood_days` | days with big-river flooding (separate) |
| `nwm_reach_events_per_100_reaches` | reach-level frequency, comparable across parishes of different size |
| `nwm_share_reaches_flooded` | share of the parish's reaches that flooded at least once |

### 7.4 ZCTA tables (`data\zcta\`)

`zcta_daily_2018.csv`, `zcta_events_2018.csv` and `zcta_annual_2018.csv` have the same logic as the parish tables, with these differences:

| Column | Meaning |
|---|---|
| `zcta` | 5-digit ZCTA code (text) |
| `main_parish_fips`, `main_parish_name` | parish holding most of the ZCTA's land |
| `main_parish_share`, `n_parishes` | that parish's share of the land; number of parishes the ZCTA touches (annual table) |
| `n_reaches` | usable NWM reaches inside the ZCTA - **check this before trusting a value** |
| `reaches_flooding`, `frac_flooding` | reaches above threshold that day; as a share |
| `urban`, `urban_share` | more than half urban; urban share of the land |
| `confidence` | high / medium / low / no data |
| `no_data_reason` | "unverified parish" or "< 5 reaches" |
| `share_reaches_flooded` | share of reaches that flooded at least once in 2018 (annual table) |

### 7.5 `data\zcta\zcta_events_2018_enriched.csv` - ZCTA events with the extra fields

| Column | Meaning |
|---|---|
| `storm_id` | storm the event belongs to (links to `storms_2018.csv`) |
| `event_id` | `<zcta>_2018_<nn>` |
| `event_type`, `event_type_basis` | type label and what it is based on |
| `magnitude_best` | observed magnitude if available, else modeled |
| `magnitude_source` | `observed (USGS)`, `modeled (NWM)` or `none`; "imputed" reserved for future use |
| `magnitude_modeled`, `magnitude_observed` | NWM and USGS peak ÷ 2-yr flow |
| `observed_stage_above_flood_ft` | USGS stage above NWS flood stage, if a stage gauge flooded |
| `usgs_gauges` | gauges inside the ZCTA that flooded during the event |
| `return_period_max`, `return_period_median`, `return_period_class` | how rare the peaks were (years; classes <2, 2-5, 5-10, 10-25, 25-50, ≥50) |
| `duration_median_rise_h`, `severity_max_norm` | as above |
| `nws_episode_ids`, `nws_report_types`, `nws_flood_cause` | matching NOAA reports |
| `tropical_cyclone`, `coastal_flood_reported`, `big_river_flooding` | storm-level context |

### 7.6 `data\zcta\storms_2018.csv` - one row per storm

| Column | Meaning |
|---|---|
| `storm_id` | `LA2018_S###`, numbered in time order |
| `start_date`, `end_date`, `length_days` | first and last flood day of any ZCTA in the storm; length |
| `peak_day` | earliest peak day among the storm's ZCTAs |
| `n_zctas`, `zctas` | number of ZCTAs affected; their codes separated by `;` |
| `n_parishes`, `parishes` | number of parishes; their FIPS codes separated by `;` |
| `event_type`, `event_type_basis` | tropical cyclone / heavy rainfall - flash flooding / heavy rainfall - river flooding / heavy rainfall - not reported; and the evidence used |
| `tropical_cyclone` | name and NHC ID of a cyclone within 300 km (e.g. `Gordon (AL072018)`) |
| `nws_flood_cause`, `nws_episode_ids` | from matching NOAA reports |
| `coastal_flood_reported`, `big_river_flooding` | context flags |
| `magnitude_max` | largest peak ÷ 2-yr flow in the storm |
| `return_period_max` | rarest flood in the storm (years, capped at 100) |
| `duration_median_rise_h` | typical rise time |
| `severity_max_norm` | most severe flooding |
| `share_observed` | share of the storm's ZCTA events with an observed (USGS) magnitude |

To get one row per storm and ZCTA:

```python
st = pd.read_csv("storms_2018.csv", dtype={"zctas": str})
long = st.assign(zcta=st.zctas.str.split(";")).explode("zcta")
```

---

## 8. Key results for 2018

**Data coverage**
- USGS: 203 gauges with 15-min data (73 flow, 130 stage only); 23 parishes without a flow gauge.
- NWM: 43,240 reaches in all 64 parishes; 35,504 usable after removing dry ditches and reaches without a threshold.

**NWM vs USGS (53 gauges)**
- Median hourly correlation 0.83; median peak ratio 0.94.
- 42 of 53 gauges agree well; the poor ones are mostly in the delta, tidal and regulated areas.
- In flood years, NWM also flooded at 28 of the 34 gauges where USGS flooded (82%).

**Parish dataset (final run, after clean-up)**
- 979 parish flood days (4.4% of parish-days with data); 195 parish flood events.
- Median 3 events per parish; most: Calcasieu (8).
- Only one event longer than 30 days: Bossier (real late-February Red River flood).
- Big-river flooding: 803 parish-days, reported separately.
- Sources: NWM 51 parishes, USGS 10, no data 3.

**Validation with NOAA Storm Events**

| Measure | Local floods | Local + big-river |
|---|---|---|
| Reported parish floods detected | **38 / 58 (66%)** | 40 / 58 (69%) |
| Dataset events confirmed by a report | 36 / 195 (18%) | 36 / 195 (18%) |
| Day level POD / FAR / CSI | 0.56 / 0.91 / 0.06 | 0.81 / 0.93 / 0.05 |
| Frequency bias (dataset / reported flood days) | 9.0 | 15.6 |

| Group | Detected |
|---|---|
| High-confidence parishes | **81%** (22 / 27) |
| Medium-confidence parishes | 50% (10 / 20) |
| Low-confidence parishes | 55% (6 / 11) |
| Flash floods | 63% (31 / 49) |
| River floods | 78% (7 / 9) |

**How to write it up (example)**
> Compared with NWS flood reports in the NOAA Storm Events Database, the dataset detected 66% of reported parish flood episodes in 2018 (81% in high-confidence parishes; 78% of river floods and 63% of flash floods). Most dataset events had no matching report, which reflects both the moderate 2-year-flow threshold and the known under-reporting of floods in rural areas.

**Known 2018 events reproduced:** the late-February / March floods in north and west Louisiana (Red, Ouachita, Sabine and Calcasieu basins) and the very wet December in northwest Louisiana.

ZCTA and storm results (steps 09-10): see `zcta_summary_2018.txt` and the printed summary of step 10.

---

## 9. Limitations

- **Coast, delta and cities.** NWM does not simulate tides, storm surge, levees, diversions or pumped urban drainage. Cameron, Plaquemines and St. Bernard have no reliable data; 16 parishes are low confidence; urban ZCTAs are downgraded.
- **Streamflow, not flooding on the ground.** No flood depth or flooded area yet; rainfall ponding away from streams is missed (a main reason urban flash floods are missed).
- **Mild threshold.** The 2-yr flow is roughly bankfull; many events are small. Use `return_period_*` to focus on rarer floods, or re-run steps 05-07 with a higher threshold (e.g. 5- or 10-yr flow) as a sensitivity test.
- **Mostly modeled magnitudes.** Only events at USGS gauges have observed values (`magnitude_source`).
- **Big-river and coastal flooding** are flags / separate columns, not part of local magnitudes.
- **Baseline includes 2018.** The 43-yr thresholds include water year 2018; the effect is small.
- **Manual review.** The exclusion list (step 07c) reflects a hand review for Louisiana 2018; it must be redone for other places or years.
- **Return periods** above ~43 years are extrapolations (capped at 100).
- **ZCTAs** approximate ZIP codes; small ZCTAs rest on few reaches (check `n_reaches`).

---

## 10. Using the pipeline for other states or years

- **Years:** NWM v3.0 retrospective covers Feb 1979 - Jan 2023, so **1980-2022** work directly (USGS, Storm Events and HURDAT2 cover these years too). 2023 onward needs another NWM source.
- **States:** the method works anywhere in the continental US, but NWM skill and flood types differ:

| Region | Watch out for |
|---|---|
| Humid East / South | similar to Louisiana; should transfer well |
| Arid West | many dry streams, 2-yr flow near zero, unstable ratios |
| Snowmelt regions | floods rise over days-weeks; rise time and the 30-day flag behave differently |
| Dammed rivers | flow follows releases; more "stuck" reaches |
| Other coasts | same tidal/surge limits as Louisiana |

**What to change / redo for a new state or year**
1. Settings: `YEAR`, state FIPS, bounding box, county boundaries (step 02 downloads them), and the big-river name list (Mississippi / Atchafalaya are hard-coded in steps 07, 07b, 10).
2. Re-run all steps, including the 43-year annual-maximum download (large states take longer).
3. Re-run the evaluation (06) and diagnosis (07b) and **review the exclusions by hand** again.
4. Re-validate with Storm Events (08) and report the detection rate per state and year.

Recommendation: pilot one contrasting case (e.g. Louisiana 2016, or a humid inland state) before scaling up.

---

## 11. Troubleshooting

| Message | Cause | Fix |
|---|---|---|
| `429 Too Many Requests` (step 01) | no API key / rate limit | get a key, set `USGS_API_KEY`, wait 10-15 min |
| `HTTP 400 ... Long running query has been cancelled` | query too large for USGS | already handled (small groups); re-run the step |
| `reviewed_exclusions.csv not found` (step 07) | file not in `data\diagnostics\` | run `07c_make_reviewed_exclusions.py`, then step 07 again |
| `Cannot mask with non-boolean array containing NA` | old script reading empty `flood_day` values | use the updated scripts; read with `dtype={"flood_day": "boolean"}` |
| `Cannot convert input [... days] ... to Timestamp` (step 09) | pandas version date handling | use the updated step 09 |
| `Mean of empty slice` warning (step 03) | reaches with no data in all years | harmless; those reaches get no threshold |
| Automatic download fails (Storm Events, Census, HURDAT2) | network / file renamed | download by hand and pass `--stormevents-file`, `--zcta-file`, `--rel-file`, `--ua-file` or `--hurdat-file` |
| Very long storms in `storms_2018.csv` | storms chaining through neighbours | set `STORM_PEAK_GAP = 1` in step 10 |

---

## 12. Data sources

| Data | Provider | Access |
|---|---|---|
| Streamflow and stage, annual peaks, thresholds | U.S. Geological Survey, Water Data APIs | <https://api.waterdata.usgs.gov> |
| National Water Model v3.0 retrospective (hourly streamflow 1979-2023) | NOAA Office of Water Prediction | `s3://noaa-nwm-retrospective-3-0-pds` (<https://registry.opendata.aws/nwm-archive/>) |
| Storm Events Database (flood reports) | NOAA NCEI | <https://www.ncei.noaa.gov/pub/data/swdi/stormevents/csvfiles/> |
| HURDAT2 Atlantic hurricane tracks | NOAA National Hurricane Center | <https://www.nhc.noaa.gov/data/> |
| 2020 ZCTA boundaries and relationship files | U.S. Census Bureau | <https://www2.census.gov/geo/> |
| County (parish) outlines | Census-based GeoJSON (plotly datasets) | downloaded by step 02 |
| Method inspiration | Khajehei et al. (2020), Sci. Rep. 10:448 | doi:10.1038/s41598-019-57349-z |

---

## 13. Glossary

- **Parish** - Louisiana's name for a county; parish FIPS = county FIPS.
- **ZCTA** - ZIP Code Tabulation Area, the Census approximation of a ZIP code area.
- **Reach** - one stream segment in the NWM river network (NHDPlus v2), identified by `feature_id`.
- **Discharge / flow** - volume of water passing per second (m³/s; USGS reports cfs, 1 cfs = 0.0283 m³/s).
- **Stage / gage height** - water level at a gauge (ft).
- **2-yr flow** - a flow reached or exceeded in about half of all years (roughly bankfull).
- **Return period** - average number of years between floods of a given size.
- **Rise time** - hours from the start of a flood (threshold crossing) to its peak.
- **Retrospective (NWM)** - a long historical model run driven by observed weather, without gauge correction.
- **Crosswalk** - the link between a USGS gauge and its NWM reach.
- **POD / FAR / CSI** - probability of detection, false alarm ratio, critical success index (standard forecast-verification scores).
- **Frequency bias** - dataset flood days ÷ reported flood days (> 1 = more floods in the dataset than reported).
- **Big river** - reaches with a 2-yr flow above 3,000 m³/s (Mississippi, Atchafalaya, lower Red), handled separately.
- **Confidence** - how much the dataset can be trusted in a parish / ZCTA, from step 06 (and urban share for ZCTAs).
