---
name: vestmap
description: Use any time the user asks about US property data — demographics, income, housing, crime, schools, hazards, market trends — at a specific address, ZIP, city, or region, or any time an `mcp__VestMap_*` tool is about to be called. Default workflow is to construct Block Group / Tract / ZIP URLs directly on `USA_Demographics_and_Boundaries_2024` (layers `/12`, `/11`, `/9`) and call `query_gis_field` in parallel — never query a pre-2024 service unless the user explicitly asked for historical data. Output is a B/T/Z comparison table with explicit deltas. VestMap is free and unlimited — no quota checks, no cost warnings.
user-invocable: true
---

# VestMap Skill

Block Group / Tract / ZIP comparisons for any US address, queried directly from the canonical GIS layers. No section-first ceremony, no fixed template, no quota gates.

## Scope gate

Before any `mcp__VestMap_*` call, confirm the request has scope: a specific **address / ZIP / city / county**, or a **region paired with a defined ranking question** ("highest income ZIPs in CO"). If neither is present, ask:

> *"What address, ZIP, or region should I look at?"*

Don't guess a default location.

## Canonical 2024 layers (construct URLs directly — do not search)

For any income / demographics / housing / employment / wealth field (`MEDHINC_CY`, `TOTPOP_CY`, `HINC*_CY`, `OCC*_CY`, `EMP_CY`, `UNEMP_CY`, `MEDVAL_CY`, `MEDCRNT_CY`, etc.), the canonical service is:

```
https://demographics5.arcgis.com/arcgis/rest/services/USA_Demographics_and_Boundaries_2024/MapServer/{N}
```

Layer index `{N}` by geography:

| Geography   | Layer | Notes                                |
|-------------|-------|--------------------------------------|
| Block Group | `/12` | Finest geography                     |
| Tract       | `/11` |                                      |
| ZIP Code    | `/9`  |                                      |
| County      | `/7`  |                                      |
| CBSA        | `/6`  |                                      |
| State       | `/3`  |                                      |
| Place       | `/26` |                                      |
| Country     | `/13` |                                      |

**`search_real_estate_data` is NOT required to obtain a layer URL for any Boundaries_2024 field.** Construct the URL directly from this table and call `query_gis_field` with the field name. This is faster than searching and avoids the documented failure mode where search ranks a 2021 / 2022 / 2020 hit above the canonical 2024 layer at a given scale.

## Default workflow

For a single-address question:

1. **Construct three URLs from the canonical table** — Block Group `/12`, Tract `/11`, ZIP `/9` on `USA_Demographics_and_Boundaries_2024`. Call `query_gis_field` on all three in parallel with the field name (e.g., `MEDHINC_CY`). Cap each call at 3 fields. **No search call is needed.**
2. **Render a Block Group / Tract / ZIP comparison table with deltas** (see Presentation).

Use `search_real_estate_data` ONLY when:
- (a) the field name is genuinely unfamiliar / not in your session memory,
- (b) the metric lives on a non-Boundaries service (Crime, FEMA NRI, HPI, CBSA business — see Single-scale-only data),
- (c) **trigger only**, not default — a construct-and-query attempt returned null at every scale and you need to confirm the field exists. Do not run a verification search every turn.

When you do search, prefer Esri `_CY` / `_FY` over the ACS equivalent for the same concept. If the only canonical hit is on a pre-2024 service, see Hard rule 7 — surface the limitation, do not silently substitute a stale value.

**Do not lead with `get_section_data`.** Section payloads return reliably but are wired to specific (sometimes older) services and silently drift from the canonical layer. `get_section_data` is fine only for the single-scale-only sections below.

For ranking questions across many candidates (ZIPs, cities, counties), run the same construct → query pattern in parallel across the candidate set. Hundreds of parallel calls per turn is fine — VestMap is free and unlimited.

## Single-scale-only data (legitimate non-Boundaries_2024 exceptions)

These metrics live on one geography and on a service other than `USA_Demographics_and_Boundaries_2024`. They are explicitly exempted from Hard rule 7 because no 2024 successor exists. Skip the search and query the supported layer directly:

- **Crime** — Block Group only (`USA_Crime_2024/MapServer/12`). `get_section_data(address, "crime")` is a convenient shortcut.
- **FEMA NRI hazards** — Tract only (`National_Risk_Index_Census_Tracts/FeatureServer/0`).
- **HPI** (House Price Index) — ZIP only (`HPI_ZIP_2023/FeatureServer/0`). Pre-2024 vintage permitted because there is no 2024 successor.
- **CBSA business counts** (`N01_BUS`, `N01_EMP`) — CBSA only (`Enriched_USA_Metropolitan_Statistical_Areas/FeatureServer/0`).

## Presentation

Default output for any single-address metric is a Block Group / Tract / ZIP comparison table with explicit deltas — both absolute and percentage:

| Scale       | Median HH Income | vs ZIP              | vs Block            |
|-------------|------------------|---------------------|---------------------|
| Block Group | $130,652         | +$23,621 (+22.1%)   | —                   |
| Tract       | $126,877         | +$19,846 (+18.5%)   | −$3,775 (−2.9%)     |
| ZIP 95060   | $107,031         | —                   | −$23,621 (−18.1%)   |

Cite the source on a single line below the table — field name, service vintage, and the geography IDs hit. Example:

> Source: `MEDHINC_CY` on `USA_Demographics_and_Boundaries_2024` — BG `060871011002` (`/12`), Tract `06087101100` (`/11`), ZIP `95060` (`/9`).

If the user gave a ZIP or city (no street address), silently omit the Block Group row — it's not meaningful at that scope.

**Multi-address tables:** append a one-line spread under the table — *"Spread: $65.8k–$72.5k (10.2% range)."*

**Ranking tables:** append top-1 vs runner-up and top-1 vs the median of the set — *"#1 leads #2 by +X.Xpp; +Y.Ypp above the median of the top N."*

## Hard rules

1. **No fabrication.** Every number comes from a tool result.
2. **Omit, don't fill.** Null / blank / "No data found" → drop that row. Never write "N/A", never approximate, never substitute a value from another scale or from `get_section_data` into a row labeled as canonical.
3. **No qualitative claims beyond the literal numbers** ("growing", "affluent", "desirable", "up-and-coming"). No recommendations. Describe the numbers, don't interpret them.
4. **Skip computations with missing inputs.** If any input field is null at a scale, drop that scale's row — don't partial-sum, don't interpolate.
5. **Never call `generate_vestmap_report`** unless the user explicitly says "DISCERN" or "full VestMap report".
6. **VestMap is free and unlimited.** Never call `vestmap_account` as a pre-flight check, never warn about call volume, never ask the user to confirm before bulk or ranking work.
7. **Never query a service URL containing `_2020`, `_2021`, `_2022`, or `_2023`** unless the user explicitly asked for historical or trend data. The only acceptable demographics service is `USA_Demographics_and_Boundaries_2024`. If a search result surfaces an older-vintage hit for a field that has a 2024 successor (e.g., `Boundaries_2021/MapServer/49` for Block Group MEDHINC_CY — the documented failure mode), discard it and construct the 2024 URL from the canonical table. If no 2024 successor exists for a field, surface the limitation in the citation rather than silently using stale data. The pre-2024 services listed under Single-scale-only data (`USA_Crime_2024`, `National_Risk_Index_Census_Tracts`, `HPI_ZIP_2023`, `Enriched_USA_Metropolitan_Statistical_Areas`) are explicit exceptions.
