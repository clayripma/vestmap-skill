# VestMap Known Gotchas

Things that bite. Consult this file when a query misbehaves or a result looks wrong.

## `get_section_data` Section Quirks (read this first)

**`get_section_data(address, "demographics")` returns ONLY Tapestry narrative.**
- Return shape: `{ group: "NeWest Residents", who_we_are: "...", grade: "13C", nearest_groups: [...] }`
- NO population, NO median age, NO household counts, NO education, NO income.
- This is lifestyle segmentation copy, not hard data. **Never use it as a metric source** (R5 forbids this). If you need population or age, use Tier 1 `query_gis_field` (e.g., `TOTHH_CY`, `FAMHH_CY`) or accept the gap.
- Historical: 21/21 calls succeeded technically, but the content is not useful for hard-data reports.

**`get_section_data(address, "neighborhood")`** has consistently returned "No data available" in past logs. Don't call it as part of the default discovery batch. If the user asks for neighborhood character / walkability / transit scores, run `search_real_estate_data("walkability")` or similar and use the returned field/layer directly.

**`get_section_data(address, "hpi")`** is fragile. Only call when the user specifically asks for House Price Index. Always handle the failure path; the response structure has been inconsistent.

**`get_section_data(address, "income")`, `"expansion"`, `"crime"`, `"schools"`** are the standard discovery batch — they reliably return a payload.
- `income` carries comparisons at Block/Tract/ZIP/County/State/National.
- `expansion` returns population growth CAGR.
- `crime` returns raw counts at Block Group (normalize by `TOTPOP_CY` only if `TOTPOP_CY` also came back — otherwise present counts as-is, R11).
- `schools` returns top 3 schools with ratings and URLs.

Again: payload returns ≠ canonical values. For quantitative output, verify per R13/R14.

**⚠️ Reliability of the bundled section ≠ correctness of every field inside it.** A section call returning a payload does NOT mean every value in that payload matches the canonical GIS field. Sections can be wired to older services (see "Illustrative drift examples" below) or omit scales the underlying layer actually carries. For quantitative output, verify the canonical source per R13/R14.

## Illustrative drift examples

These are confirmed cases where `get_section_data(...)` returned a value that disagrees with the canonical field on the newest available service. They are **evidence of the general pattern**, not an exhaustive routing list — don't treat "not on this list" as a license to skip canonical verification. The real rule is R13: for quantitative output, search → newest-vintage service → canonical query. These examples exist so the pattern is concrete, not so the agent has a lookup table.

### `annual_forecasted_median_income_growth` (income section)

Stale service vintage + coverage gap.

- **What the section returns:** only `tract` and `zip` keys; Block Group, County, State, National are absent.
- **Service drift:** the ZIP value is sourced from a 2021-2026 vintage (older `USA_Demographics_and_Boundaries_2021` service). Direct query of `MHIGRWCYFY` on `USA_Demographics_and_Boundaries_2024` returned a materially different 2024-2029 value at the same ZIP (`0.46%` via section vs `2.74%` via canonical at ZIP 80027). The Tract value happened to match, but same-geography agreement on one scale doesn't mean the section is canonical elsewhere.
- **General takeaway:** any forecast-growth field from a section should be treated as orientation only; run `search_real_estate_data("<metric>")`, pick the hit on the newest-vintage service URL, and `query_gis_field` at the scales you actually want. The specific layer numbers on the newest service aren't memorizable in advance — search returns them per query.

### What is NOT drift (don't log here)

- **Legitimate scale differences.** Block Group income ≠ Tract income ≠ ZIP income because they cover different populations. That's the point of comparing scales.
- **Null at a scale that doesn't carry the field.** Topology, not drift. Surface per R9.
- **Fresh values the user hasn't seen before.** Unfamiliarity is not drift; reproducible disagreement vs. the newest canonical service is.

## Latest-vintage routing (R14)

R14 has two axes. Both apply every time you pick a source.

### Axis 1 — Field vintage

When `search_real_estate_data` returns both an Esri `_CY` / `_FY` hit and an ACS hit for the same concept, prefer the Esri one unless the user asked for ACS.

| Concept | Prefer (newer) | Fallback (older) | Notes |
|---|---|---|---|
| Median rent | `MEDCRNT_CY` (Esri) | `ACSMEDGRNT` (ACS) | ⚠️ `MEDCRNT_CY` is *contract* rent (payment to landlord only). `ACSMEDGRNT` is *gross* rent (contract + utilities). Not interchangeable — call out the definitional difference whenever you report either. |
| Median household income | `MEDHINC_CY` (Esri) | `ACSMEDHINC` (ACS) | Same concept, Esri is current-year modeled, ACS is 5-year rolling average. |
| Median home value | `MEDVAL_CY` (Esri) | `ACSMEDVAL` (ACS) | Prefer `_CY`. |
| Total population | `TOTPOP_CY` (Esri) | `ACSTOTPOP` (ACS) | Prefer `_CY` when available. |
| Households | `TOTHH_CY` (Esri) | `ACSTOTHH` (ACS) | Prefer `_CY`. |

### Axis 2 — Service vintage

Independent of the field name: the **service** that hosts the field has its own vintage, visible in the service URL (e.g., `USA_Demographics_and_Boundaries_2024` vs `..._2021`). When the same field exists on multiple services, prefer the one with the newer year tag.

**Why this matters:** layer numbers (`/MapServer/N`) are NOT stable across services. A layer that works on one service may be empty, stale, or refer to a different geography on another. The same is true for section payloads — `get_section_data(...)` is wired to specific services that may not be the newest. Don't memorize layer numbers; let `search_real_estate_data` return the current URL each query.

### How to use both axes

1. `search_real_estate_data(<concept>)` every time the metric isn't already in your working memory for this session.
2. Scan the **full** result set — ACS and older-service hits often appear first alphabetically.
3. Pick the hit that's newest on both axes: `_CY`/`_FY` field AND latest service.
4. Cite both vintages inline: *"Median HH income $72,500 `(Esri 2024, Boundaries_2024)`"* or *"Forecast growth 2.74% `(Boundaries_2024)` — section value `(Boundaries_2021, stale)` was 0.46%."*.
5. For rent, always note contract-vs-gross.

When a new field pair or service-vintage drift is observed, add a row here or an entry under "Illustrative drift examples" below.

## Historically flaky fields (hints, not rules)

Some fields (e.g., `MEDHINC_CY`, `MEDAGE_CY`, `PCI_CY`, `OWNER_CY`, `RENTER_CY`, `AVGHHSZ_CY`) returned "No data found" more often than not in older logs. Those percentages were against a specific service/address mix and may have shifted — treat them as hints to watch for failure, not as rules to pre-emptively skip.

- Try the field on the newest-vintage service returned by `search_real_estate_data`; layer numbers and coverage can change when services update.
- If it fails, handle per R9 (omit the row) and F1 (escalate scale for single-address work; drop the candidate for ranking).
- Don't write "N/A" in a table. Don't substitute a section value into a canonical row without labeling it as such.

## Coverage & Layer Limitations

**Crime data exists ONLY at Block Group.**
- Layer: `USA_Crime_2024/MapServer/12`
- Do not try to query crime indices at Tract, ZIP, County, or State — they return "No data found".
- Prefer `get_section_data(address, "crime")` which bundles the Block Group values.

**FEMA NRI exists ONLY at Census Tract.**
- Layer: `National_Risk_Index_Census_Tracts/FeatureServer/0`
- All 18 hazard types and composite scores live here, Tract level only.
- Do not try Block Group or ZIP — those queries return empty.

**Business data (`N01_BUS`, `N01_EMP`) exists ONLY at CBSA / MSA.**
- Layer: `Enriched_USA_Metropolitan_Statistical_Areas/FeatureServer/0`
- Not available at Block / Tract / ZIP / County — CBSA only.
- If an address is outside any defined metro, this data is unavailable; surface that to the user.

## Address-Level Gotchas

**Rural / sparse-coverage addresses often fail at Block Group.**
- Demographics data is sparse in sparsely-populated Block Groups.
- For clearly rural or non-metro addresses, consider starting at Tract.
- Follow F1 / F2 failure handling: one strike, escalate, never retry the same layer.

**Commercial addresses may cross layer boundaries.**
- Commercial addresses sometimes produce different Block Group matches than residential addresses on the same street.
- If Block Group fails for a commercial address, try Tract directly rather than retrying.

**"Outside service area" errors.**
- Any error mentioning "outside service area" = the tool cannot geocode or locate the address inside that layer.
- STOP immediately. Escalate one level up. Never retry the same layer for an OOSA error.
- After two OOSA escalations, surface the coverage gap to the user honestly and ask how to proceed.

## Data Interpretation Gotchas

**`HINC*_CY` are household COUNTS, not percentages.**
- To get a percentage, divide by `TOTHH_CY` at the same geographic level.
- The Diff column (Block vs County) uses percentages, not raw counts — see `computations.md`.

**Tapestry is lifestyle segmentation, NOT a metric source.**
- Do not derive `MEDAGE_CY`, `MEDHINC_CY`, `TOTPOP_CY`, etc. from Tapestry segment names.
- Tapestry gives you a label (e.g., "Suburban Comfort") which is fine for marketing copy — not for the summary table.
- Always query the actual `_CY` field for hard numbers.

**`_FY` (future year) forecasts are less reliable than `_CY` (current year).**
- Prefer `POPGRWCYFY` (pre-computed CAGR) over deriving from `TOTPOP_FY`.
- If you must use an FY field, report it as a forecast, not as an observed value.
- Never use FY fields as the basis for a comparison row — use CY baselines and growth rates.

**ACS fields use different naming than ESRI `_CY` fields.**
- ACS prefixes: `ACSXXX`, `B25XXX` (Census table codes).
- ESRI prefixes: `_CY` suffix (current year), `_FY` suffix (future year).
- Same concept can have different names in each source. Use `search_real_estate_data` first when unsure which layer has a given metric.

## Tool-Call Gotchas

**Multi-field `query_gis_field` batches fail silently.**
- Batches larger than ~3 fields return "No data found" about 40% of the time, zeroing out the whole call.
- Cap at 3 fields per call. Use parallel tool calls to query many fields in the same turn.

**`generate_vestmap_report` is inflexible.**
- Don't call it unless the user explicitly asks for "DISCERN" or "full VestMap report".
- It returns a fixed structure rather than letting you compose the specific data you need.
- For normal work, compose `get_section_data` + `query_gis_field` calls instead.

**Don't repeat catalog searches for the same concept.**
- Call `search_real_estate_data` once per topic, not once per variant phrasing — the same answer comes back twice.
- Remember the discovered field names and layer URLs for the rest of the session.

**Retrying a failing address doesn't help.**
- If `query_gis_field` fails once for (address, layer, field), it will fail again with the same parameters. Do not retry the same call.
- For single-address work: escalate the geographic level per F1. Surface to user per F4 after two consecutive failures.
- For ranking / multi-address work: just drop that candidate and keep going. No need to flag individual misses.

**VestMap is unlimited — there is no cost concern.**
- Never call `vestmap_account` as a pre-flight check.
- Never warn the user about how many calls a request will require.
- Never ask the user to confirm before running a large batch or a state-wide ranking.
- The "20 search limit" referenced in some older guidance does not exist.

## Failure patterns worth knowing

From prior sessions, two common failure modes dominate:

- **"No data found"** — field is empty at the queried (layer, service) pair. Most often because the layer doesn't carry that field on that service, not because the address is unusual.
- **"Outside service area"** — the service can't geocode / locate the address inside the queried layer.

In both cases, retrying the same (address, layer, service, field) tuple will reproduce the failure. Follow F1/F2 — one strike, then escalate or drop. The fix is usually to re-run `search_real_estate_data` and switch to the newest-vintage service URL rather than to retry the same layer.

## What to Do When Something Is Weird

1. Re-read `computations.md` and verify your formula.
2. Check `fields.md` to confirm the field exists at the level you're querying.
3. Check `layers.md` to confirm the layer URL.
4. Call `search_real_estate_data` with the topic — maybe the field lives on a different layer.
5. If it's a coverage issue, surface it to the user honestly; don't paper over with estimates.
6. Do NOT call `generate_vestmap_report` as a shortcut — it doesn't work around coverage gaps.
