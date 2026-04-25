# VestMap Computation Cookbook

Formulas, mappings, and category aggregations. All variables refer to field names in `fields.md`.

## Rule R11 — Precondition Gating (applies to every formula below)

Before running ANY computation, verify that every input field returned non-null at the (address, level) you're computing for. If even one component is missing, **SKIP the entire computation** — do not estimate, interpolate, partial-sum, or approximate. The output should simply omit that row/metric.

Each computation below includes a `Required fields` line. If any field in that list returned blank, the metric does not appear in the output.

## Viability Summary

- **Fully viable (all components Tier 1, reliable):** Occupation buckets, Income distribution percentages, Reduced education buckets
- **Viable with care:** Income distribution Diff vs County (needs county query to also succeed), Unemployment rate (two Tier 1 inputs), Owner/Renter mix (both Tier 3 — usually skip)
- **Marginal (unvalidated inputs):** Population CAGR from `_FY` fields — prefer `get_section_data("expansion")`, Home value CAGR — prefer `get_section_data("income")` growth forecast
- **Do not compute from `query_gis_field`:** Median income comparisons — use `get_section_data("income")` directly

## Unemployment Rate

**Required fields:** `EMP_CY` AND `UNEMP_CY` (both Tier 1) at same layer — OR `UNEMPRT_CY` (Tier 2) alone.

**Preferred:** `UNEMPRT_CY` directly if present.

**Fallback formula:**
```
Unemployment Rate = UNEMP_CY / (EMP_CY + UNEMP_CY) × 100
```

**R11 gate:** if `UNEMPRT_CY` is blank AND either `EMP_CY` or `UNEMP_CY` is blank at that layer, SKIP the metric.

## Occupation Buckets

**Required fields:** ALL 13 occupation fields returned at the same layer. All are Tier 1, so this computation is **fully viable** empirically.

Group the 13 occupation fields into three display categories. Values are raw counts; divide by `EMP_CY` for percentages.

**White Collar (Professional / Admin)**
```
WC = OCCMGMT_CY + OCCSALE_CY + OCCCOMP_CY + OCCEDUC_CY + OCCHLTH_CY
```

**Services**
```
SVC = OCCFOOD_CY + OCCPROT_CY + OCCPERS_CY + OCCBLDG_CY
```

**Blue Collar (Production / Trade)**
```
BC = OCCCONS_CY + OCCFARM_CY + OCCPROD_CY + OCCTRAN_CY
```

**Share percentages**
```
WC %  = WC  / EMP_CY × 100
SVC % = SVC / EMP_CY × 100
BC %  = BC  / EMP_CY × 100
```

These three should roughly sum to 100% (small rounding + unclassified residual is expected).

**R11 gate:** if ANY of the 13 occupation fields is blank at the target layer, SKIP the WC/Services/BC breakdown entirely. Do not compute partial buckets.

## Education Buckets (Reduced — Tier 1 Only)

**Required fields:** `NOHS_CY`, `HSGRAD_CY`, `SMCOLL_CY`, `BACHDEG_CY`, `GRADDEG_CY` (all Tier 1). `SOMEHS_CY` and `ASSCDEG_CY` are Tier X (unvalidated) and dropped from the default computation.

Group the 5 reliable education fields (Pop 25+) into four display categories:

```
No HS Diploma       = NOHS_CY                       (NOTE: excludes 9-12 no diploma)
HS Graduate         = HSGRAD_CY
Some College        = SMCOLL_CY                     (NOTE: excludes associate degrees)
Bachelor's & Higher = BACHDEG_CY + GRADDEG_CY
```

**Total Population 25+ (reduced)**
```
Pop25 = NOHS_CY + HSGRAD_CY + SMCOLL_CY + BACHDEG_CY + GRADDEG_CY
```

**Percentages**
```
Bucket % = Bucket / Pop25 × 100
```

**R11 gate:** if ANY of the 5 Tier 1 education fields is blank, SKIP the breakdown.

**Full 7-field formula (only if you verified `SOMEHS_CY` and `ASSCDEG_CY` also returned data):**
```
No HS Diploma       = NOHS_CY + SOMEHS_CY
HS Graduate         = HSGRAD_CY
Some College/Assoc  = SMCOLL_CY + ASSCDEG_CY
Bachelor's & Higher = BACHDEG_CY + GRADDEG_CY
Pop25               = sum of all 7 fields
```

## Income Distribution Percentages

**Required fields:** 9 HINC buckets + `TOTHH_CY` (all Tier 1). **Fully viable empirically.**

`HINC*_CY` fields are raw household counts. Convert to percentages using `TOTHH_CY` at the same geographic level:

```
HINC_Bucket % = HINC_Bucket / TOTHH_CY × 100
```

**R11 gate:** if `TOTHH_CY` OR any HINC bucket is blank, skip that bucket. If fewer than 6 buckets returned, skip the whole distribution.

## Income Distribution Diff (Block vs County)

**Required fields:** All 9 HINC buckets + `TOTHH_CY` at BOTH Block Group AND County levels. Viable if both layers return — but you need TWO successful query_gis_field batches.

For each income bucket, compute the Block-level percentage minus the County-level percentage. Result is signed percentage points.

```
For each bucket in (HINC0, HINC15, HINC25, HINC35, HINC50, HINC75, HINC100, HINC150, HINC200):
    Block_%  = HINC{bucket}_CY_at_Block  / TOTHH_CY_at_Block  × 100
    County_% = HINC{bucket}_CY_at_County / TOTHH_CY_at_County × 100
    Diff     = Block_% − County_%   # signed percentage points (e.g. +8.2, −3.1)
```

To get the County values, query the same `HINC*_CY` + `TOTHH_CY` fields at MapServer `/7`.

**R11 gate:** if EITHER the Block or County query returns blank on any bucket, skip that bucket's Diff row. If fewer than 6 buckets have valid Diffs, skip the whole Diff column.

## Population Growth CAGR (5-Year)

**DO NOT compute from `query_gis_field` — use `get_section_data("expansion")` instead.**

`get_section_data("expansion")` returns CAGR at Block/Tract/ZIP/County/State/National in one reliable call (20/20 success in past logs). That's your go-to.

Only fall back to a CAGR formula if `get_section_data("expansion")` fails AND both `TOTPOP_FY` and `TOTPOP_CY` returned at the same layer:
```
CAGR = (TOTPOP_FY / TOTPOP_CY)^(1/5) − 1
```

`POPGRWCYFY` is Tier X (unvalidated). Do not query it directly without `search_real_estate_data` discovery first.

## Diversity Index

`DIVINDX_CY` is Tier X (unvalidated via `query_gis_field`). Do not include in default output. If `search_real_estate_data` confirms it exists on a layer and it returns, display the raw 0–100 value as-is. Never describe the area as "diverse" or "homogeneous" without this number (R10).

## Home Value / Rent Growth

**DO NOT compute from `query_gis_field`.** `MEDVAL_CY` and `MEDVAL_FY` are Tier X. `get_section_data("income")` returns `median_home_value` at Block/Tract/ZIP reliably; use that.

If the user specifically wants a 5-year home value CAGR and it's not in the income section, acknowledge the gap — do not fabricate from general knowledge.

## Owner / Renter Mix

**Do NOT compute.** Both `OWNER_CY` and `RENTER_CY` are Tier 3 (~31% success). If both happen to return, the formula is:

```
Owner %  = OWNER_CY  / (OWNER_CY + RENTER_CY) × 100
Renter % = RENTER_CY / (OWNER_CY + RENTER_CY) × 100
```

**R11 gate:** if either field is blank (usually the case), skip the metric entirely. Do not write "Owner-occupied: not available".

## Competitor OM Radius Translation

When a source document uses 1-mile / 3-mile / 5-mile radius data:

| Source radius | Default replacement |
|---|---|
| 1 mile | Block Group |
| 3 mile | Tract |
| 5 mile | ZIP Code |

This is a rough mapping — actual Census geometries don't match concentric circles. When translating, **state explicitly in the output** that Block / Tract / ZIP replaces the radius columns.

## Notes

- Never estimate a value by blending multiple sources — query each level directly.
- Never use Tapestry segment names (e.g. "Suburban Comfort") as a source for `MEDAGE_CY` or `MEDHINC_CY`. Tapestry is lifestyle segmentation only.
- When any term in a formula is missing, report the gap rather than substituting a default.
- Round display values only — keep full precision in intermediate calculations.
