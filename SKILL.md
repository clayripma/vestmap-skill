---
name: vestmap
description: Use any time the user asks about US property data — demographics, income, housing, crime, schools, hazards, market trends — at a specific address, ZIP, city, or region, or any time an `mcp__VestMap_*` tool is about to be called. Default workflow is `search_real_estate_data` → `query_gis_field` at Block Group / Tract / ZIP in parallel, presented as a comparison table with explicit deltas. VestMap is free and unlimited — no quota checks, no cost warnings.
user-invocable: true
---

# VestMap Skill

Block Group / Tract / ZIP comparisons for any US address, queried directly from the canonical GIS layers. No section-first ceremony, no fixed template, no quota gates.

## Scope gate

Before any `mcp__VestMap_*` call, confirm the request has scope: a specific **address / ZIP / city / county**, or a **region paired with a defined ranking question** ("highest income ZIPs in CO"). If neither is present, ask:

> *"What address, ZIP, or region should I look at?"*

Don't guess a default location.

## Default workflow

For a single-address question:

1. **`search_real_estate_data(<metric keywords>)`** — find the canonical field. From the result set, pick the **newest-vintage service URL** (e.g., `*_2024_*` over `*_2021_*`) and prefer Esri `_CY` / `_FY` over the ACS equivalent for the same concept.
2. **`query_gis_field` in parallel at the Block Group + Tract + ZIP layer URLs** returned by that search. Cap each call at 3 fields.
3. **Render a Block Group / Tract / ZIP comparison table with deltas** (see Presentation).

**Do not lead with `get_section_data`.** Section payloads return reliably but are wired to specific (sometimes older) services and silently drift from the canonical layer. For any quantitative comparison, go straight to search → query. `get_section_data` is fine as a fallback when search returns nothing useful, or for the single-scale-only sections below.

For ranking questions across many candidates (ZIPs, cities, counties), run the same search → query pattern in parallel across the candidate set. Hundreds of parallel calls per turn is fine — VestMap is free and unlimited.

## Single-scale-only data

These metrics live on one geography and have no multi-scale comparison. Skip the search and query the supported layer directly:

- **Crime** — Block Group only (`USA_Crime_2024/MapServer/12`). `get_section_data(address, "crime")` is a convenient shortcut.
- **FEMA NRI hazards** — Tract only (`National_Risk_Index_Census_Tracts/FeatureServer/0`).
- **CBSA business counts** (`N01_BUS`, `N01_EMP`) — CBSA only.

## Presentation

Default output for any single-address metric is a Block Group / Tract / ZIP comparison table with explicit deltas — both absolute and percentage:

| Scale       | Median HH Income | vs ZIP              | vs Block            |
|-------------|------------------|---------------------|---------------------|
| Block Group | $130,652         | +$23,621 (+22.1%)   | —                   |
| Tract       | $126,877         | +$19,846 (+18.5%)   | −$3,775 (−2.9%)     |
| ZIP 95060   | $107,031         | —                   | −$23,621 (−18.1%)   |

Cite the source on a single line below the table — field name, service vintage, and the geography IDs hit. Example:

> Source: `MEDHINC_CY` on `USA_Demographics_and_Boundaries_2024` — BG `060871011002`, Tract `06087101100`, ZIP `95060`.

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
