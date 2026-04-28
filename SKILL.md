---
name: vestmap
description: Use any time the user asks about property demographics, income, crime, schools, natural hazards, market trends, or investment research at a US address — or any time an `mcp__VestMap_*` tool is about to be called. VestMap usage is FREE AND UNLIMITED — never warn about cost, never ask for confirmation, never check quota. For single-address questions, lead with bundled `get_section_data` for orientation and verify the canonical source for any quantitative output. For open-ended / ranking / multi-address questions, run as many calls as needed in parallel to figure out the answer. Discover the current canonical source per query via `search_real_estate_data` — never rely on memorized layer numbers or historical reliability percentages. Forbids fabrication or analysis beyond what returned numbers literally show.
user-invocable: true
---

# VestMap Real Estate Data Skill

General-purpose methodology for the VestMap Real Estate Intelligence MCP tool.

**VestMap usage is FREE AND UNLIMITED.** There is no 20-search limit. There is no quota. There is no cost. Never warn the user about call volume, never ask for confirmation before bulk work, never call `vestmap_account` as a pre-flight check. If the user asks an open-ended question ("which ZIP in Colorado has highest pop growth?"), just figure it out — query as many addresses as needed, in parallel, and return the answer.

The skill provides two complementary modes:
- **For single-address lookups**: lead with the bundled `get_section_data` discovery batch for orientation, then verify the canonical source for any quantitative output (R13) and fill specific gaps with `query_gis_field`.
- **For open-ended / ranking / multi-address questions**: enumerate the candidate set, run parallel section calls or field queries against all of them, rank or aggregate, return the answer.

Stay grounded — every number comes from a tool result, no fabrication, no qualitative analysis beyond the literal numbers.

## When to use this skill

Trigger on any of:
- Any `mcp__VestMap_*` tool is about to be called
- User mentions a US address + wants demographic / income / crime / school / hazard / market data
- User asks open-ended ranking questions ("which ZIP / city / county has the highest / lowest X in [region]")
- User asks for investment analysis, due diligence, property research, or neighborhood comparison
- User asks to compare multiple addresses or build a property database
- User provides a competitor document using 1/3/5 mile radius data

## Always-Follow Rules (hard constraints, no exceptions)

**R0. Location-scope gate.** Before any `mcp__VestMap_*` tool call, confirm the request has scope: either (a) a specific **address / ZIP / city / county**, or (b) a **region** (state / metro / named list of places) paired with a **defined ranking question** ("highest pop growth in Colorado", "cheapest ZIPs in Texas"). Open-ended ranking questions on a region ARE a valid scope — do not force the user to narrow them down (see R12). If neither (a) nor (b) is present (e.g., "tell me about the market", "how's real estate right now?"), stop and ask exactly:

> **"What address, ZIP, or region should I look at?"**

Do not guess a location, do not pick a default city, do not call any VestMap tool until the user answers.

**R1. Block / Tract / ZIP is the default comparison framework** for single-address comparisons. Never 1/3/5 mile radius. Add County for income-distribution Diffs and macro context. Add State only on explicit request.

**R2. Query ≤3 fields per `query_gis_field` call.** Multi-field batches (>3 fields) fail ~40% of the time because one missing field zeroes the whole call. Use parallel tool calls.

**R3. For single-address lookups, lead with `get_section_data` as orientation.** The bundled sections `income`, `expansion`, `crime`, and `schools` return a payload reliably and are the fastest first move on any address-specific question. Treat the payload as orientation (fine to serve as-is for §Snapshot); for any quantitative output, route through R13's canonical verification. Fall back to `query_gis_field` for specific gaps (Section 3). For ranking / multi-address questions, see Section 4.

**R4. Never call `generate_vestmap_report` unless the user explicitly says "DISCERN" or "full VestMap report".**

**R5. Never use Tapestry for hard metrics.** `get_section_data("demographics")` returns ONLY Tapestry narrative — not population, age, or income. For hard demographics, use `query_gis_field` Tier 1 fields.

**R6. All data is real.** Every number comes from a VestMap tool call. No fabrication, no estimation, no "approximately".

**R7. Always show quantitative comparisons across scales.** When comparing geographies (Block vs Tract vs ZIP vs County, or address A vs B vs C), do not stop at side-by-side values — explicitly compute and surface the deltas. Show absolute differences (pp for percentages, $ for dollars), ratios or multiples, and rank within the set when relevant. Example: don't just print "Block 2.72%, ZIP 2.74%, County 3.62%" — add "County exceeds Block by 0.90 pp (≈33% higher); ZIP and Block are within 0.02 pp." Apply this to every comparison output (Quick / Compare / MultiAddr / Brief / Rank). For ranking questions, also show the gap between #1 and the median or runner-up so the user knows whether the winner is decisive or marginal.

**R8. Format follows intent.** No branded templates, no fixed visual styling, no forced HTML. Markdown by default. Maps are an optional capability (`output-modes.md §Map`), not a default.

**R9. Blank fields are OMITTED, never filled.** If a query returns null / blank / "No data found", remove the row entirely. Do NOT write "N/A", do NOT approximate from general knowledge, do NOT compute around it. Output gets SMALLER when data is missing — never padded with filler. For ranking outputs, drop missing candidates from the table; don't list them with blank values.

**R10. No qualitative analysis beyond the literal numbers.** Describe, don't interpret. No adjectives without a backing number from the tool results: *growing, affluent, up-and-coming, family-friendly, desirable, stable, diverse, working-class, well-educated, declining*. No trend claims without a growth rate. No recommendations.

**R11. Computed metrics are precondition-gated.** Before computing buckets, Diffs, CAGRs, etc., verify ALL component fields returned non-null. If ANY missing, SKIP the computation entirely.

**R12. Open-ended questions get full exploration.** When the user asks something exploratory ("which X has highest Y", "rank the top 10", "find me the best Z in this state", "explore neighborhoods near…"), do NOT ask the user to narrow it down. Do NOT warn about how many calls it will take. Do NOT check quota. Enumerate the candidate set and run the calls in parallel. VestMap is unlimited — use it freely. The user wants you to figure it out, not to manage call budgets.

**R13. Sections are convenience; canonical fields are truth for quantitative output.** `get_section_data(...)` is fast and usually returns a payload — treat it as **orientation data**, not as the authoritative source. The section payload can be silently stale (wired to an older service) or incomplete (missing scales the underlying layer actually carries). Neither failure mode is rare; neither is detectable just by looking at the payload.

Routing rule:

  - **Orientation / §Snapshot output** → use the section payload directly. Fast, fine.
  - **Any quantitative output** (comparison across scales, ranking, forecast, decision-grade numbers, anything the user will act on) → do not stop at the section value. Run `search_real_estate_data(<metric>)`, pick the newest-vintage hit per R14 (both field vintage AND service vintage), `query_gis_field` the canonical field at the scales the user asked about, and cite vintage inline.
  - **Section value and canonical query disagree at the same geography** → prefer the canonical value and surface the mismatch so the user can see which source is being used.
  - **Canonical query returns null at a requested scale** → fail honestly (R9, omit the row). No silent substitution from the section, another scale, or general knowledge.

  `references/gotchas.md` keeps **illustrative examples** of known section/service drift (e.g., `annual_forecasted_median_income_growth`). Those are evidence of the general pattern — they are NOT an exhaustive routing list, and "not on the list" is never a reason to skip canonical verification for quantitative output. Discover the current canonical source per query; don't trust a precooked routing map that ossifies when services update.

**R14. Prefer latest-vintage on both field AND service axes.** `search_real_estate_data` results carry two independent vintages. Scan the full result set and prefer the newest on each axis before routing:

  - **Field vintage.** Esri `_CY` / `_FY` > ACS when both hit for the same concept.
    - **Rent:** `MEDCRNT_CY` (Esri) > `ACSMEDGRNT` (ACS). ⚠️ Definitional difference: `MEDCRNT_CY` is *contract* rent (tenant payment only); `ACSMEDGRNT` is *gross* rent (contract + utilities). Cite the difference when reporting.
    - **Median household income:** `MEDHINC_CY` (Esri) > `ACSMEDHINC` (ACS).
    - **Home value / total population / households:** prefer the `_CY` variant over the ACS equivalent.
  - **Service vintage.** When the same field appears on multiple services, prefer the one with the newer year tag in the service URL — e.g., `USA_Demographics_and_Boundaries_2024` > `..._2021` > older. **Layer numbers are NOT stable across services**; the same `/MapServer/N` on an older service can return a stale value or no value. Trust the service URL returned by `search_real_estate_data`, not a memorized layer map.

  Cite both axes inline: `(Esri 2024, Boundaries_2024)` / `(ACS 2022 5-Yr)` / `(Boundaries_2021 — stale, newer service available)` so the user can reason about freshness.

## Section 3 — Single-Address Discovery-First Workflow

For single-address questions, follow this sequence.

### Step 1 — Discovery Batch (parallel)

```
get_section_data(address, "income")     → median HH income @ Block/Tract/ZIP/County/State/National,
                                           median home value @ Block/Tract/ZIP,
                                           annual_forecasted_median_income_growth @ Tract/ZIP only
                                           (orientation only — for quantitative output, verify canonical
                                            per R13/R14; forecast growth has a documented drift example
                                            in gotchas.md)
get_section_data(address, "expansion")  → population growth CAGR @ Block/Tract/ZIP/County/State/National
get_section_data(address, "crime")      → raw crime counts @ Block Group
get_section_data(address, "schools")    → top 3 nearest schools + district_name (only if user touches schools)
```

**Do NOT include in discovery:**
- `demographics` — Tapestry narrative only, no hard metrics (R5).
- `neighborhood` — has consistently returned "No data available"; don't call it as part of the default batch. If the user asks for walkability / transit / neighborhood character specifically, discover via `search_real_estate_data` first.
- `hpi` — fragile; only call when the user specifically asks for House Price Index, and handle the failure case.

### Step 2 — Route by output intent (R13)

What the user is going to do with the numbers decides which source to trust:

- **§Snapshot / orientation** → use the section payload directly. Proceed to Step 4.
- **Quantitative output** (§Compare, §MultiAddr, §Brief, §Rank, §Bulk — any output that invites comparison, ranking, or decision-making) → Step 3 runs canonical verification via `search_real_estate_data` → newest-vintage service (R14) → `query_gis_field`. Do not ship quantitative output sourced only from a section.
- **Discovery omits a scale the user asked about** → Step 3: `search_real_estate_data` the metric, find the layer that carries it, query directly. Don't silently drop the scale.
- **Discovery returns null and canonical also returns null at the requested scale** → omit the row per R9. Do not substitute.

### Step 3 — Canonical verification via `query_gis_field`

For any metric going into quantitative output: `search_real_estate_data(<metric>)` → scan the full result set for the newest-vintage service URL (per R14, e.g., `*_2024_*` > `*_2021_*`) → pick the canonical `_CY` / `_FY` field on that service → `query_gis_field` at every scale the user asked about, in parallel (≤3 fields per call).

Do not assume layer numbers are stable across services — the same layer number on an older service may carry a stale value or no value at all. Trust the service URL returned by `search_real_estate_data`, not a baked-in layer map.

If the canonical value disagrees with a non-null section value at the same geography, prefer canonical and note the mismatch (one short line, not a paragraph).

`references/fields.md` catalogs fields observed in past logs, with rough historical success hints. Treat those hints as starting points, not hard rules — services update and success rates shift. Never pre-emptively skip a field because of a historical percentage; try it, and handle failure per R9/F1. Always run `search_real_estate_data` first when the metric isn't already in your working memory for this session.

### Step 4 — Build Output

Apply R9 (omit blanks), R10 (no ungrounded prose), R11 (skip incomplete computations).

## Section 4 — Open-Ended / Ranking / Multi-Address Workflow

For questions like "which Colorado ZIP has highest pop growth?", "rank top 10 cities in TX by median income", "compare these 50 neighborhoods", "find the best counties in Florida for X".

**No quota concerns. No confirmation needed. No "this might take a while" hedging. Just run it.**

### Step A — Identify the candidate set

- **State-wide ZIP ranking:** enumerate all ZIPs in the state from your own knowledge.
  - Colorado: ~526 ZIPs in the 80000-81658 range (with gaps)
  - Texas: ~1,800 ZIPs in the 75000-79999, 77xxx, etc. ranges
  - Wyoming: ~180 ZIPs in 82000-83128
  - For any state, your training data has the canonical ZIP list — emit it directly
- **City-wide neighborhood ranking:** enumerate ZIPs or census tracts in the city
- **List of named places:** convert each to a representative address
- **County-level ranking:** enumerate counties in the state
- **Unsure of enumeration?** Use `search_real_estate_data` to find a layer that lists candidates

For ZIPs, a representative address is just the ZIP code itself (e.g., `"80027"` or `"ZIP 80027, CO"` — geocodes to that ZIP region).

### Step B — Run the parallel batch

For each candidate, call `get_section_data(address, <relevant_section>)` in parallel. Use the section that contains the target metric:

| Target metric | Section | Extract from response |
|---|---|---|
| Population growth (CAGR 2024-2029) | `expansion` | `.zip` (or `.county`, etc.) |
| Median household income | `income` | `median_household_income.zip` |
| Median home value | `income` | `median_home_value.zip` |
| Forecasted income growth | `income` | `annual_forecasted_median_income_growth.zip` |
| Crime counts | `crime` | `CRMCYTOTC` etc. (Block Group only — for ZIP-level crime ranking, use `query_gis_field` on the crime layer per ZIP) |

For metrics not in `get_section_data`, use `query_gis_field` Tier 1 fields against the appropriate layer (typically `/9` for ZIP-level, `/7` for county-level).

**Run all candidates in parallel.** Hundreds of calls in a single turn is fine. Use multiple parallel tool calls per turn — don't serialize.

### Step C — Rank

Sort candidates by the target metric (descending for "highest", ascending for "lowest"). Drop candidates that returned blank (R9 — they don't appear in the ranking, no "N/A" rows).

### Step D — Output

Return a ranked markdown table with the value column. Top N (default 10 unless user specified). Include a one-line note saying which tool call and which response key the values came from, so the user can verify.

If the user asked for "the highest" (singular), still show the top 5-10 so they see context (R7).

## Failure Handling

**F1. One strike, then escalate (single-address only).** A failure at a geographic level → escalate Block → Tract → ZIP → County. For ranking/multi-address, just drop failed candidates and keep going — do not retry.

**F2. "Outside service area" = stop immediately for that (address, layer).** Escalate one level for single-address; drop and continue for ranking.

**F3. Cache per-address "smallest working level"** for single-address work. Not relevant for ranking (each candidate is independent).

**F4. Two consecutive failures on the same address = surface to user (single-address only).** For ranking, just exclude the address from results.

**F5. Batch cap: ≤3 fields per `query_gis_field` call.** Parallel tool calls for more fields.

**F6. VestMap is FREE AND UNLIMITED.** Never call `vestmap_account` as a pre-flight check. Never warn about cost, quota, credits, or call volume. Never ask the user to confirm before bulk or ranking work. Never hedge with "this might take a while" or "this will use ~N calls". The "20 search limit" mentioned in any earlier guidance does not exist — ignore it. Just run the queries.

**F7. `search_real_estate_data` before any field query you're not fully sure of.** Never guess field names. Never assume the layer you used last session is still the canonical one — services update. When the result set contains multiple hits for the same concept, pick the newest on both axes per R14 (`_CY`/`_FY` over ACS; newer service over older) and cite both vintages inline.

## Output Mode Router

| User intent | Output shape | Reference |
|---|---|---|
| **Default — bare or vague address request** ("tell me about 123 Main St", "what's this area like?", "is this a good ZIP?") | **Snapshot card (markdown)** | `output-modes.md §Snapshot` |
| "What's X here?" | Inline markdown 1-liner with B/T/Z comparison | `output-modes.md §Quick` |
| "Compare Block / Tract / ZIP" | 3-column markdown table | `output-modes.md §Compare` |
| "Show me neighborhoods A, B, C" | Multi-address markdown table | `output-modes.md §MultiAddr` |
| Explicit opt-in: "due diligence", "full brief", "everything you have" | Discovery-driven markdown brief | `output-modes.md §Brief` |
| "Which X has highest Y in [region]" / "Top N" | Ranked markdown table with values | `output-modes.md §Rank` |
| "Database of N addresses" | CSV file | `output-modes.md §Bulk` |
| "Can you show a map?" | Leaflet + ESRI tiles | `output-modes.md §Map` |

This skill deliberately does NOT have an "OM HTML page" output mode. If asked for HTML OM, use §Brief + §Map and let the user drive the visual design.

## Quick Reference Summary

**Single-address tool order:** R0 scope-gate → parallel `get_section_data(income, expansion, crime)` (+ `schools` only if asked) for orientation → route by output intent (R13): §Snapshot uses section directly; any quantitative output triggers `search_real_estate_data` → newest-vintage service (R14) → `query_gis_field` at the user's scales → cite vintage on both axes → apply R7 / R9 / R10 / R11 → default output is §Snapshot unless the user asked for a specific shape.

**Ranking tool order:** enumerate candidates → canonical verification per R13/R14 (search → newest service → `query_gis_field`) for the target metric across candidates → sort → drop blanks → top N table with vintage cited in the source note. No quota concerns, no confirmation step.

**Source-trust note.** Section payloads return reliably, but "reliable payload" ≠ "canonical value." Sections can be wired to older services or ship fewer scales than the underlying layer actually carries. Do not treat a returned payload as authoritative for quantitative output. Single-level-only topology facts are stable:
- Crime → Block Group only (`USA_Crime_2024/MapServer/12`)
- FEMA NRI → Tract only (`National_Risk_Index_Census_Tracts/FeatureServer/0`)
- Business → CBSA only (`Enriched_USA_Metropolitan_Statistical_Areas/FeatureServer/0`)

For every other field and layer, `search_real_estate_data` returns the current canonical URL. Use it — don't memorize layer numbers or historical reliability percentages; they go stale when services update.

**Tier 1 fields:** `OCCMGMT_CY`, `OCCSALE_CY`, `OCCCOMP_CY`, `OCCEDUC_CY`, `OCCHLTH_CY`, `OCCFOOD_CY`, `OCCPROT_CY`, `OCCPERS_CY`, `OCCBLDG_CY`, `OCCCONS_CY`, `OCCFARM_CY`, `OCCPROD_CY`, `OCCTRAN_CY`, `EMP_CY`, `UNEMP_CY`, `FAMHH_CY`, `NOHS_CY`, `HSGRAD_CY`, `SMCOLL_CY`, `BACHDEG_CY`, `GRADDEG_CY`, `HINC0_CY`–`HINC200_CY`, `TOTHH_CY`

**Single-level-only data:**
- Crime → Block Group only
- FEMA NRI → Tract only
- Business → CBSA only

## Pointers to Reference Files

- Field tier / reliability / catalog → `references/fields.md`
- Layer URL / coverage / priority → `references/layers.md`
- Formulas / computation viability gates → `references/computations.md`
- Section quirks and failure modes → `references/gotchas.md`
- Output templates including §Brief, §Rank, §Map → `references/output-modes.md`
