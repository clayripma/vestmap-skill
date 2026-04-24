# VestMap Known Gotchas

Things that bite. Consult this file when a query misbehaves or a result looks wrong.

## `get_section_data` Section Quirks (read this first)

**`get_section_data(address, "demographics")` returns ONLY Tapestry narrative.**
- Return shape: `{ group: "NeWest Residents", who_we_are: "...", grade: "13C", nearest_groups: [...] }`
- NO population, NO median age, NO household counts, NO education, NO income.
- This is lifestyle segmentation copy, not hard data. **Never use it as a metric source** (R5 forbids this). If you need population or age, use Tier 1 `query_gis_field` (e.g., `TOTHH_CY`, `FAMHH_CY`) or accept the gap.
- Historical: 21/21 calls succeeded technically, but the content is not useful for hard-data reports.

**`get_section_data(address, "neighborhood")` is BROKEN (0/12 success).**
- Returns "No data available" 100% of the time in past logs.
- Do not call it as part of discovery.
- If the user asks for neighborhood character / walkability / transit scores, use `search_real_estate_data("walkability")` or similar to find specific fields.

**`get_section_data(address, "hpi")` is FRAGILE (~90% success).**
- About 10% of calls return "No data available".
- Only call when the user specifically needs House Price Index.
- Always handle the failure path — do not assume the data returned.
- The response structure for successful calls is inconsistent in logs; verify before building output.

**`get_section_data(address, "income")`, `"expansion"`, `"crime"`, `"schools"` are the RELIABLE four.**
- 100% success across 74 past calls combined.
- These four are your discovery batch. Lead with them.
- `income` returns comparisons at Block/Tract/ZIP/County/State/National — far more than you get from raw field queries.
- `expansion` returns population growth CAGR at all six levels.
- `crime` returns raw counts at Block Group (normalize by `TOTPOP_CY` only if `TOTPOP_CY` actually returned — otherwise present counts as-is).
- `schools` returns top 3 schools with ratings and URLs.

**⚠️ Reliability of the bundled section ≠ correctness of every field inside it.** "100% reliable" means the call returns a payload — it does NOT mean every value inside that payload matches the canonical GIS field. A small number of discovery fields are known to be wrong or incomplete; they're listed under "Known bad/incomplete discovery fields (R13)" below. Skip the section value for those and go direct. For every other field, trust discovery and move on — do NOT cross-check at runtime.

## Known bad/incomplete discovery fields (R13)

This is the **authoritative list** of `get_section_data` fields that are known to be wrong, incomplete, or missing scales. Use it to pick the right tool up front: for any metric below, skip `get_section_data` and query the canonical `_CY` / `_FY` field via `query_gis_field` directly.

**The list is exhaustive.** If a field is not here, trust the discovery value — do NOT cross-check it as a matter of routine. Runtime self-verification is explicitly not how this skill works: we get smarter by growing this list as confirmed mismatches surface, not by double-querying every value.

### `annual_forecasted_median_income_growth` (income section)

**Symptom:** Two distinct problems.
1. **Coverage gap:** the payload only contains `tract` and `zip` keys. Block Group is omitted, even though `MHIGRWCYFY` exists at Block Group `/12`.
2. **Value mismatch at ZIP:** the `zip` value returned by the section has been observed to disagree with the canonical `MHIGRWCYFY` field at layer `/9` by ~6×. (Example: 295 Clementina St, Louisville CO 80027 — section returned `0.46%`, direct query of `MHIGRWCYFY` at `/9` returned `2.74%`. Tract values matched at `2.24%`, so it's a ZIP-specific issue.)

**Workaround:** ignore the section's growth-rate values. Query `MHIGRWCYFY` directly via:
- Block Group: `https://demographics5.arcgis.com/arcgis/rest/services/USA_Demographics_and_Boundaries_2024/MapServer/12`
- Tract: `https://demographics5.arcgis.com/arcgis/rest/services/USA_Demographics_and_Boundaries_2024/MapServer/11`
- ZIP: `https://demographics5.arcgis.com/arcgis/rest/services/USA_Demographics_and_Boundaries_2024/MapServer/9`
- County: `https://demographics5.arcgis.com/arcgis/rest/services/USA_Demographics_and_Boundaries_2024/MapServer/7`

Pair with `PCIGRWCYFY` (Per Capita Income CAGR) on the same layers if needed. Use `MEDCRNT_CY` / `MEDCRNT_FY` on the same layers for forecasted rent (compute CAGR as `(FY/CY)^(1/5) − 1`). Use `HUGRWCYFY` (total housing units CAGR) and `RNTGRWCYFY` (renter-occupied units CAGR) for unit-supply forecasts.

### Growing the list (how we catch new mismatches)

**Do not treat this as a per-query cross-check routine.** These are the triggers for when to *investigate once* in-session and, if confirmed, add a new entry to the list above so the next session skips discovery from the start:

- A user flags that a discovery value contradicts a reliable external benchmark they know.
- Discovery omits a scale (e.g., Block Group, County) that a canonical-layer query reveals actually exists at that scale — fill the missing scale directly; if this happens twice for the same field, add it to the list.
- You're about to call `search_real_estate_data` for a metric and find that the section field and the canonical `_CY` / `_FY` field have noticeably different definitions or vintage.

When a new mismatch is confirmed: run the canonical field via `query_gis_field` once, prefer that value, and **add the field to the "Known bad/incomplete discovery fields" list above with a symptom + workaround entry**. That is the only sanctioned route for the skill to get smarter — not runtime self-verification.

## Flaky Fields That Look Reliable (Tier 3 Trap)

Six core-looking `query_gis_field` fields only succeed ~30% of the time. **Do NOT query these directly**:

- `MEDHINC_CY` (28.6% success) — use `get_section_data("income")` → `median_household_income`
- `MEDAGE_CY` (30.8%) — no reliable alternative; accept as potentially missing
- `PCI_CY` (28.6%) — no reliable alternative
- `OWNER_CY` (30.8%) — no reliable alternative
- `RENTER_CY` (30.8%) — no reliable alternative
- `AVGHHSZ_CY` (30.8%) — no reliable alternative

If your output needs one of these, try to satisfy the user from the bundled income/expansion data. Do NOT write "Median Income: N/A" in a table — per R9, omit the row entirely.

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

## Observed Failure-Rate Baseline

From ~20 past sessions (before this skill existed):
- **292** "No data found" occurrences
- **254** "outside service area" occurrences
- Single address retried up to **4×** with identical failure (don't do this)

These should drop materially when the F1/F2 escalation rules are followed (one strike, then escalate or drop).

## What to Do When Something Is Weird

1. Re-read `computations.md` and verify your formula.
2. Check `fields.md` to confirm the field exists at the level you're querying.
3. Check `layers.md` to confirm the layer URL.
4. Call `search_real_estate_data` with the topic — maybe the field lives on a different layer.
5. If it's a coverage issue, surface it to the user honestly; don't paper over with estimates.
6. Do NOT call `generate_vestmap_report` as a shortcut — it doesn't work around coverage gaps.
