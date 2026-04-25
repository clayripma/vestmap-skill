# VestMap Field Catalog

Empirical reliability tiers for `query_gis_field` calls, derived from 1,821 past calls across 44 sessions. Query Tier 1 with confidence, Tier 2 with fallback handling, Tier 3 never (use `get_section_data` instead), Tier X only after `search_real_estate_data` discovery.

Convention: `_CY` suffix = Current Year (2024). `_FY` suffix = Future Year (2029 forecast).

---

## Reliability Tier System

### Tier 1 — Query with confidence (>80% empirical success)

These fields returned non-null data in 80%+ of past calls across MapServer `/9`, `/11`, `/12`. Query freely in Step 3 of the discovery workflow.

**Employment + all 13 Occupations**
- `EMP_CY` — Employed civilian population 16+
- `UNEMP_CY` — Unemployed population
- `FAMHH_CY` — Family households
- `TOTHH_CY` — Total households (HINC denominator)
- `OCCMGMT_CY` — Management / business / financial (87.5%)
- `OCCSALE_CY` — Sales and related (87.5%)
- `OCCCOMP_CY` — Computer / engineering / science
- `OCCEDUC_CY` — Education / legal / arts
- `OCCHLTH_CY` — Healthcare
- `OCCFOOD_CY` — Food preparation / serving
- `OCCPROT_CY` — Protective service
- `OCCPERS_CY` — Personal care
- `OCCBLDG_CY` — Building / grounds
- `OCCCONS_CY` — Construction / extraction (100%)
- `OCCFARM_CY` — Farming / fishing / forestry
- `OCCPROD_CY` — Production (92.9%)
- `OCCTRAN_CY` — Transportation / moving

**Education (5 of 7 reliable — see reduced buckets below)**
- `NOHS_CY` — Less than 9th grade
- `HSGRAD_CY` — High school graduate
- `SMCOLL_CY` — Some college, no degree
- `BACHDEG_CY` — Bachelor's degree
- `GRADDEG_CY` — Graduate / professional degree

**Income Distribution (all 9 HINC buckets)**
- `HINC0_CY`   — <$15,000
- `HINC15_CY`  — $15,000–$24,999
- `HINC25_CY`  — $25,000–$34,999
- `HINC35_CY`  — $35,000–$49,999
- `HINC50_CY`  — $50,000–$74,999
- `HINC75_CY`  — $75,000–$99,999
- `HINC100_CY` — $100,000–$149,999
- `HINC150_CY` — $150,000–$199,999
- `HINC200_CY` — $200,000+

### Tier 2 — Query but expect gaps (~50% empirical success)

These return data about half the time. Query them, but have a fallback ready and don't panic if blank.

| Field | Success | Notes |
|---|---|---|
| `TOTPOP_CY` | 46.7% | Often missing; `get_section_data("expansion")` gives growth trajectory with more levels |
| `UNEMPRT_CY` | 50% | Prefer this for direct read; compute from `UNEMP_CY / (EMP_CY + UNEMP_CY)` as fallback if both Tier 1 fields are present |
| `AVGHINC_CY` | ~50% | Use this over `MEDHINC_CY` if querying directly — but `get_section_data("income")` is better |

### Tier 3 — Do NOT query directly (<35% empirical success)

These look like obvious core metrics but fail 65%+ of the time via `query_gis_field`. **Use `get_section_data("income")` or the relevant bundled section instead** — they return the same concepts more reliably and at more geographic levels.

| Field | Success | Bundled alternative |
|---|---|---|
| `MEDHINC_CY` | **28.6%** | `get_section_data("income")` → `median_household_income` (returns Block/Tract/ZIP/County/State/National in one call) |
| `MEDAGE_CY` | 30.8% | No reliable bundled alternative — accept that median age may be unavailable for some addresses |
| `PCI_CY` | 28.6% | `get_section_data("income")` context |
| `OWNER_CY` | 30.8% | No bundled alternative; if user needs owner/renter mix, query the Tier 1 `TOTHH_CY` and surface the limitation |
| `RENTER_CY` | 30.8% | (same as above) |
| `AVGHHSZ_CY` | 30.8% | No bundled alternative |

### Tier X — Unvalidated (never tested in past logs)

Appear in field catalogs but no empirical success data. **Discover via `search_real_estate_data` first** to confirm they exist on the layer you're targeting. Do not query blind.

- `MEDHINC_FY` — Median HH income 2029 forecast — use `get_section_data("income")` → `annual_forecasted_median_income_growth` instead
- `MEDVAL_CY` — Median home value — use `get_section_data("income")` → `median_home_value` instead
- `MEDVAL_FY`, `MEDRENT_CY`, `MEDCRNT_CY` — Housing value / rent fields
- `POPDENS_CY` — Population density
- `POPGRWCYFY` — Population growth CAGR — use `get_section_data("expansion")` instead
- `RENTER_FY`, `OWNER_FY`, `TOTPOP_FY`, `TOTHH_FY` — Forecast fields
- `SOMEHS_CY` — 9th–12th grade (fold into `NOHS_CY` bucket)
- `ASSCDEG_CY` — Associate degree (fold into `SMCOLL_CY` bucket)
- `DIVINDX_CY` — Diversity index
- `MEDNW_CY` — Median net worth
- `MALES_CY`, `FEMALES_CY` — Gender counts
- `AVGIA25_CY`, `AVGIA35_CY`, `AVGIA45_CY`, `AVGIA55_CY`, `AVGIA65UCY` — Age-segmented income cohorts

---

## "Leave It to get_section_data" Cheatsheet

For these metrics, DO NOT use `query_gis_field`. The bundled sections are both more reliable AND return more comparison levels:

| Concept | Bundled source | Levels returned |
|---|---|---|
| Median household income | `get_section_data("income")` → `median_household_income` | Block, Tract, ZIP, **County, State, National** |
| Median home value | `get_section_data("income")` → `median_home_value` | Block, Tract, ZIP |
| Income growth forecast (CAGR) | `get_section_data("income")` → `annual_forecasted_median_income_growth` | Tract, ZIP |
| Population growth CAGR | `get_section_data("expansion")` | Block, Tract, ZIP, **County, State, National** |
| Crime counts (all offenses) | `get_section_data("crime")` → `CRMCYTOTC`, `CRMCYPROC`, `CRMCYPERC`, `CRMCYMURD`, `CRMCYRAPE`, `CRMCYROBB`, `CRMCYASST`, `CRMCYBURG`, `CRMCYLARC` | Block Group |
| Nearest schools + ratings | `get_section_data("schools")` → `schools_nearest_3`, `district_name` | Point-radius |

---

## Field Details by Category (with tier annotations)

### Demographics (Population & Households)

| Field | Tier | Description | Notes |
|---|---|---|---|
| `TOTPOP_CY` | **2** | Total population | ~47% success; `get_section_data("expansion")` gives growth trend instead |
| `TOTHH_CY` | **1** | Total households | HINC denominator — reliable |
| `FAMHH_CY` | **1** | Family households | Reliable |
| `AVGHHSZ_CY` | **3** | Average household size | ~31% — skip |
| `MEDAGE_CY` | **3** | Median age | ~31% — no reliable alternative |
| `MALES_CY`, `FEMALES_CY` | X | Gender counts | Unvalidated |
| `DIVINDX_CY` | X | Diversity index 0-100 | Unvalidated |
| `POPDENS_CY` | X | Population density | Unvalidated |

### Housing

| Field | Tier | Description | Notes |
|---|---|---|---|
| `OWNER_CY` | **3** | Owner-occupied | 31% — skip |
| `RENTER_CY` | **3** | Renter-occupied | 31% — skip |
| `MEDVAL_CY` | X | Median home value | **Use `get_section_data("income")` → `median_home_value`** |
| `MEDVAL_FY`, `MEDRENT_CY`, `MEDCRNT_CY` | X | Other housing values | Unvalidated |
| `OWNER_FY`, `RENTER_FY` | X | Forecast tenure | Unvalidated |

### Income & Wealth

| Field | Tier | Description | Notes |
|---|---|---|---|
| `MEDHINC_CY` | **3** | Median HH income | **29% — use `get_section_data("income")` → `median_household_income` instead** |
| `AVGHINC_CY` | **2** | Average HH income | ~50% — fallback if section data missing |
| `PCI_CY` | **3** | Per capita income | 29% |
| `MEDHINC_FY` | X | MH income 2029 forecast | `get_section_data("income")` gives you the % trajectory instead |
| `MEDNW_CY` | X | Median net worth | Unvalidated |

### Income Distribution (all Tier 1)

Each `HINC*_CY` field is a raw household count. Percentage = `HINC*_CY / TOTHH_CY × 100`.

| Field | Bucket |
|---|---|
| `HINC0_CY` | <$15,000 |
| `HINC15_CY` | $15,000–$24,999 |
| `HINC25_CY` | $25,000–$34,999 |
| `HINC35_CY` | $35,000–$49,999 |
| `HINC50_CY` | $50,000–$74,999 |
| `HINC75_CY` | $75,000–$99,999 |
| `HINC100_CY` | $100,000–$149,999 |
| `HINC150_CY` | $150,000–$199,999 |
| `HINC200_CY` | $200,000+ |

All 9 buckets reliable. Income distribution is one of the most computable metrics in the catalog.

### Education (Population 25+ — 5 of 7 fields reliable)

| Field | Tier | Description |
|---|---|---|
| `NOHS_CY` | **1** | Less than 9th grade |
| `SOMEHS_CY` | X | 9th–12th, no diploma → **fold into `NOHS_CY` bucket** |
| `HSGRAD_CY` | **1** | High school graduate |
| `SMCOLL_CY` | **1** | Some college, no degree |
| `ASSCDEG_CY` | X | Associate degree → **fold into `SMCOLL_CY` bucket** |
| `BACHDEG_CY` | **1** | Bachelor's degree |
| `GRADDEG_CY` | **1** | Graduate / professional |

**Reduced education buckets (Tier 1 only):**
- No HS Diploma = `NOHS_CY`
- HS Graduate = `HSGRAD_CY`
- Some College = `SMCOLL_CY`
- Bachelor+ = `BACHDEG_CY + GRADDEG_CY`

Total Pop 25+ = sum of these 5 fields. Full 7-field formula lives in `computations.md` under the viability gate.

### Employment

| Field | Tier | Description |
|---|---|---|
| `EMP_CY` | **1** | Employed civilian population 16+ |
| `UNEMP_CY` | **1** | Unemployed population |
| `UNEMPRT_CY` | **2** | Unemployment rate (direct) — prefer but compute from `EMP_CY + UNEMP_CY` as fallback |

### Occupations (all 13 fields Tier 1)

**White Collar**
- `OCCMGMT_CY` — Management / business / financial
- `OCCSALE_CY` — Sales and related
- `OCCCOMP_CY` — Computer / engineering / science
- `OCCEDUC_CY` — Education / legal / arts
- `OCCHLTH_CY` — Healthcare

**Services**
- `OCCFOOD_CY` — Food preparation / serving
- `OCCPROT_CY` — Protective service
- `OCCPERS_CY` — Personal care
- `OCCBLDG_CY` — Building / grounds

**Blue Collar**
- `OCCCONS_CY` — Construction / extraction
- `OCCFARM_CY` — Farming / fishing / forestry
- `OCCPROD_CY` — Production
- `OCCTRAN_CY` — Transportation / moving

All 13 reliable — occupation buckets are the **most computable** aggregation in the catalog. See `computations.md`.

### Growth & Projections

| Field | Tier | Description |
|---|---|---|
| `POPGRWCYFY` | X | Population growth CAGR | **Use `get_section_data("expansion")` instead** |
| `TOTPOP_FY`, `TOTHH_FY` | X | 2029 forecasts | Unvalidated |

### FEMA National Risk Index (Tract Only)

Layer: `https://services.arcgis.com/XG15cJAlne2vxtgt/arcgis/rest/services/National_Risk_Index_Census_Tracts/FeatureServer/0`

18 hazard types × 3 variants each (`_RISKR` rating string, `_RISKV` value, `_RISKS` score). No empirical reliability data for individual hazards; treat as Tier X and verify via query result. Do NOT list hazards that came back null.

Hazard prefixes: `ERQK`, `HRCN`, `TRND`, `DRGT`, `WFIR`, `RFLD`, `CFLD`, `AVLN`, `LNDS`, `TSUN`, `HWAV`, `CWAV`, `ISTM`, `LTNG`, `HAIL`, `SWND`, `VLCN`, `WNTW`.

Composite scores (same layer): `SOVI_SCORE`, `RESL_SCORE`, `RISK_SCORE`, `RISK_RATNG`.

### Business (CBSA Only)

Layer: `https://services5.arcgis.com/9fQmObndozAJu9f5/arcgis/rest/services/Enriched_USA_Metropolitan_Statistical_Areas/FeatureServer/0`

- `N01_BUS` — Total businesses
- `N01_EMP` — Total employees

Not available at Block / Tract / ZIP / County.

---

## Field Discovery for Anything Not Listed

For any field not in this catalog, call `search_real_estate_data(query="<topic keywords>", max_results=15)` first. Use the returned field name + layer URL directly — do not guess.

## No More "19-Field Locked Schema"

An earlier version of this skill claimed a "reliable 19-field locked schema". Empirical testing showed 6 of those 19 fields return blank ~70% of the time (`MEDAGE_CY`, `MEDHINC_CY`, `PCI_CY`, `OWNER_CY`, `RENTER_CY`, `AVGHHSZ_CY`). That schema is discarded. There are only the empirical tiers above. **Lead with `get_section_data` (Step 1 discovery), supplement with Tier 1 fields (Step 3), expect gaps in Tier 2, skip Tier 3 entirely.**
