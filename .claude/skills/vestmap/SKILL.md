---
name: vestmap
description: Use any time the user asks about property demographics, income, crime, schools, natural hazards, market trends, or investment research at a US address — or any time an `mcp__VestMap_*` tool is about to be called. VestMap usage is FREE AND UNLIMITED — never warn about cost, never ask for confirmation, never check quota. For single-address questions, lead with bundled `get_section_data`. For open-ended / ranking / multi-address questions, run as many calls as needed in parallel to figure out the answer. Uses empirically-verified field reliability tiers and forbids fabrication or analysis beyond what returned numbers literally show.
user_invocable: true
---

# VestMap Real Estate Data Skill

General-purpose methodology for the VestMap Real Estate Intelligence MCP tool.

**VestMap usage is FREE AND UNLIMITED.** There is no 20-search limit. There is no quota. There is no cost. Never warn the user about call volume, never ask for confirmation before bulk work, never call `vestmap_account` as a pre-flight check. If the user asks an open-ended question ("which ZIP in Colorado has highest pop growth?"), just figure it out — query as many addresses as needed, in parallel, and return the answer.

The skill provides two complementary modes:
- **For single-address lookups**: lead with the bundled `get_section_data` discovery batch (proven 100% reliable in past sessions), then fill specific gaps with `query_gis_field`.
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

**R0. Location scope required.** Before any VestMap tool call, confirm the user has specified a **location scope**. A scope can be either:

- a specific US address / ZIP / city / county / metro (single-address workflow, Section 3), OR
- a region with a defined ranking question, e.g. "which ZIPs in Colorado have the highest pop growth?" / "which states have the highest income?" (ranking workflow, Section 4 — the state/region IS the scope).

If the user's message has neither — e.g. "tell me about demographics" or "pull some property data" with no place attached — ask one concrete question ("What address, ZIP, or region should I look at?") and stop. Do not guess, do not reuse a prior-session location, do not pick a sample address. **Open-ended ranking questions are a valid scope** — do not ask the user to narrow those down (that's R12).

**R1. Block / Tract / ZIP is the default comparison framework** for single-address comparisons. Never 1/3/5 mile radius. Add County for income-distribution Diffs and macro context. Add State only on explicit request.

**R2. Query ≤3 fields per `query_gis_field` call.** Multi-field batches (>3 fields) fail ~40% of the time because one missing field zeroes the whole call. Use parallel tool calls.

**R3. For single-address lookups, lead with `get_section_data`.** The bundled sections `income`, `expansion`, `crime`, and `schools` were 100% reliable across 74 past calls. Use them as the unconditional first move on any address-specific question. Fall back to `query_gis_field` for specific gaps (Section 3). For ranking / multi-address questions, see Section 4.

**R4. Never call `generate_vestmap_report` unless the user explicitly says "DISCERN" or "full VestMap report".**

**R5. Never use Tapestry for hard metrics.** `get_section_data("demographics")` returns ONLY Tapestry narrative — not population, age, or income. For hard demographics, use `query_gis_field` Tier 1 fields.

**R6. All data is real.** Every number comes from a VestMap tool call. No fabrication, no estimation, no "approximately".

**R7. Always show quantitative comparisons across scales.** When comparing geographies (Block vs Tract vs ZIP vs County, or address A vs B vs C), do not stop at side-by-side values — explicitly compute and surface the deltas. Show absolute differences (pp for percentages, $ for dollars), ratios or multiples, and rank within the set when relevant. Example: don't just print "Block 2.72%, ZIP 2.74%, County 3.62%" — add "County exceeds Block by 0.90 pp (≈33% higher); ZIP and Block are within 0.02 pp." Apply this to every comparison output (Quick / Compare / MultiAddr / Brief / Rank). For ranking questions, also show the gap between #1 and the median or runner-up so the user knows whether the winner is decisive or marginal.

**R8. Format follows intent.** No branded templates, no fixed visual styling, no forced HTML. Markdown by default. Maps are an optional capability (`output-modes.md §Map`), not a default.

**R9. Blank fields are OMITTED, never filled.** If a query returns null / blank / "No data found", remove the row entirely. Do NOT write "N/A", do NOT approximate from general knowledge, do NOT compute around it. Output gets SMALLER when data is missing — never padded with filler. For ranking outputs, drop missing candidates from the table; don't list them with blank values.

**R10. No qualitative analysis beyond the literal numbers.** Describe, don't interpret. No adjectives without a backing number from the tool results: *growing, affluent, up-and-coming, family-friendly, desirable, stable, diverse, working-class, well-educated, declining*. No trend claims without a growth rate. No recommendations.

**R11. Computed metrics are precondition-gated.** Before computing buckets, Diffs, CAGRs, etc., verify ALL component fields returned non-null. If ANY missing, SKIP the computation entirely.

**R12. Open-ended questions get full exploration.** When the user asks something exploratory ("which X has highest Y", "rank the top 10", "find me the best Z in this state", "explore neighborhoods near…"), do NOT ask the user to narrow it down. Do NOT warn about how many calls it will take. Do NOT check quota. Enumerate the candidate set and run the calls in parallel. VestMap is unlimited — use it freely. The user wants you to figure it out, not to manage call budgets.

**R13. Pick the right field up front; don't cross-check discovery against itself.** `get_section_data` is the default single-address tool. A small set of its fields are known to be wrong, incomplete, or missing scales — those are listed in `gotchas.md` under **Known bad/incomplete discovery fields**. For any metric on that list, skip the section value entirely and query the canonical `_CY` / `_FY` field via `query_gis_field` instead. For every other field, trust the discovery value and move on — do **not** re-query to "verify," do **not** flag Block vs Tract vs ZIP differences as divergence (those scales are supposed to differ).

  If a discovery field returns null, not-found, or is internally impossible (negative population, etc.), surface the failure honestly per R9 / F-rules — do not silently substitute a different value. If you notice a new mismatch in the wild (a discovery value contradicts a reliable external benchmark and a one-shot canonical query confirms the discrepancy), add the field to `gotchas.md` so the next session avoids it from the start. **That list — not runtime self-verification — is how the skill gets smarter.**

## Section 3 — Single-Address Discovery-First Workflow

For single-address questions, follow this sequence.

### Step 1 — Discovery Batch (parallel)

```
get_section_data(address, "income")     → median HH income @ Block/Tract/ZIP/County/State/National,
                                           median home value @ Block/Tract/ZIP,
                                           annual_forecasted_median_income_growth @ Tract/ZIP only
                                           ⚠️  This `annual_forecasted_median_income_growth` field is on the
                                              "Known bad/incomplete discovery fields" list in gotchas.md — do NOT
                                              use the section value. Query MHIGRWCYFY via query_gis_field instead
                                              whenever the user needs forecasted income growth.
get_section_data(address, "expansion")  → population growth CAGR @ Block/Tract/ZIP/County/State/National
get_section_data(address, "crime")      → raw crime counts @ Block Group
get_section_data(address, "schools")    → top 3 nearest schools + district_name (only if user touches schools)
```

**Do NOT include in discovery:**
- `demographics` — Tapestry narrative only, no hard metrics
- `neighborhood` — empirically broken (0/12 success in past logs)
- `hpi` — ~10% failure rate, only call if user specifically wants HPI

### Step 2 — Route, don't verify (R13)

For each metric the user asked about:

- **Is the field on the "Known bad/incomplete discovery fields" list in `gotchas.md`?** Skip the section value and go to Step 3 — query the canonical `_CY` / `_FY` field via `query_gis_field`. Do not use the section payload for that metric.
- **Did the user ask for a scale the discovery payload doesn't carry** (e.g., Block Group for a field that only ships Tract/ZIP)? Fill just that scale in Step 3 with a direct `query_gis_field` call. Do not silently drop the requested scale.
- **Otherwise, use the discovery value as-is.** Do not re-query to check it. Do not flag Block vs Tract vs ZIP deltas as "divergence" — those scales are supposed to differ; that's what R7 surfaces to the user as a feature, not a bug.
- **If discovery returned null, not-found, or errored** for a metric, handle per F1 / F4 — escalate one level or surface honestly. Never silently substitute a different value.

### Step 3 — Targeted `query_gis_field` Fill-In

Tier-based selection from `references/fields.md`. Parallel calls, ≤3 fields each, across Block (`/12`), Tract (`/11`), ZIP (`/9`), County (`/7`).

Use this step for: (a) Tier 1 metrics not bundled in discovery (e.g., occupation mix, education mix, income distribution buckets); (b) fields flagged in the "Known bad/incomplete discovery fields" list; (c) missing scales the user explicitly asked for but discovery didn't ship.

When querying a canonical replacement for a bad discovery field: `search_real_estate_data(<metric>)` → identify the canonical `_CY` / `_FY` field name → query it at every scale the user wanted, in parallel. Query once and trust the result — do not re-query for a cross-check.

- **Tier 1 (>80%):** all 13 occupations, 5 reliable education fields, 9 HINC buckets + TOTHH, EMP_CY, UNEMP_CY, FAMHH_CY
- **Tier 2 (~50%):** TOTPOP_CY, UNEMPRT_CY, AVGHINC_CY
- **Tier 3 (<35%):** skip — covered by `get_section_data("income")` more reliably
- **Tier X (unvalidated):** discover via `search_real_estate_data` first

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

**F7. `search_real_estate_data` before any custom / non-Tier-1 field query.** Never guess field names.

## Output Mode Router

| User intent | Output shape | Reference |
|---|---|---|
| Bare address / "tell me about X" / no specific metric | Curated markdown snapshot + deeper-dive menu | `output-modes.md §Snapshot` |
| "What's X here?" | Inline markdown 1-liner with B/T/Z comparison | `output-modes.md §Quick` |
| "Compare Block / Tract / ZIP" | 3-column markdown table | `output-modes.md §Compare` |
| "Show me neighborhoods A, B, C" | Multi-address markdown table | `output-modes.md §MultiAddr` |
| "Due diligence on X" / "Full brief" / "Everything you have" | Discovery-driven markdown brief | `output-modes.md §Brief` |
| "Which X has highest Y in [region]" / "Top N" | Ranked markdown table with values | `output-modes.md §Rank` |
| "Database of N addresses" | CSV file | `output-modes.md §Bulk` |
| "Can you show a map?" | Leaflet + ESRI tiles | `output-modes.md §Map` |

**If the user's intent is ambiguous, default to §Snapshot — never §Brief.** §Brief is opt-in for explicit phrases like "due diligence," "full brief," or "everything you have."

This skill deliberately does NOT have an "OM HTML page" output mode. If asked for HTML OM, use §Brief + §Map and let the user drive the visual design.

## Quick Reference Summary

**Single-address tool order:** confirm a location scope (R0) → parallel `get_section_data(income, expansion, crime, schools)` → **route, don't verify** (R13 — for fields on the "Known bad/incomplete discovery fields" list, go direct; otherwise trust discovery) → `query_gis_field` for Tier 1 gaps and for any bad-list field the user asked about → apply R7 (quantitative deltas across scales) / R9 / R10 / R11 → output.

**Known bad/incomplete discovery fields:** the authoritative list lives in `gotchas.md` under "Known bad/incomplete discovery fields (R13)." That list is exhaustive — if a field is not there, trust discovery. Currently listed: `annual_forecasted_median_income_growth` (income section) → use `MHIGRWCYFY` at `/12` `/11` `/9` `/7` instead.

**Ranking tool order:** enumerate candidates → parallel `get_section_data(<section>)` per candidate → extract target metric → sort → drop blanks → top N table. No quota concerns, no confirmation step.

**Section reliability (empirical):**
- 100%: `income`, `expansion`, `crime`, `schools`
- ~90%: `hpi` — handle failure
- Do not use as data source: `demographics` (Tapestry), `neighborhood` (always empty)

**Layer reliability (empirical):**
- ZIP `/9` — 100% (most reliable single-level choice)
- Tract `/11` — 97.5%
- Block Group `/12` — 97.5%
- County `/7` — 88% (rural fallback)
- Never use: State `/3` or `/19` (0%), Place/City `/20` (28%)
- Crime: `USA_Crime_2024/MapServer/12` (Block Group only)
- FEMA NRI: `services.arcgis.com/.../National_Risk_Index_Census_Tracts/FeatureServer/0` (Tract only)
- CBSA: `services5.arcgis.com/.../Enriched_USA_Metropolitan_Statistical_Areas/FeatureServer/0` (Metro only)

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
- Output templates including §Snapshot (default), §Brief, §Rank, §Map → `references/output-modes.md`
