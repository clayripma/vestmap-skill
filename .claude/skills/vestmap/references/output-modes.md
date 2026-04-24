# VestMap Output Modes

Shape-by-shape guidance. The skill is format-agnostic — these are defaults, not templates. Match the shape to the user's intent and ignore anything that doesn't help answer the question.

## R7 Universal Add-On — Quantitative Cross-Scale Comparison

**Every output that shows the same metric at multiple scales (Block / Tract / ZIP / County) or at multiple addresses must include explicit quantitative comparisons, not just side-by-side values.** This applies to every mode below.

For a single metric across N scales, after the values, append at minimum one of:
- **Absolute delta** between extremes: "County exceeds Block by 0.90 pp" (for percentages) or "ZIP $874k vs Block $955k → −$81k (−8.5%)" (for dollars).
- **Ratio / multiple:** "ZIP rent ($1,967) is 1.19× the Block Group rent ($1,654)."
- **Pairwise relationship the user cares about:** "Forecasted income growth (2.74%/yr at ZIP) outpaces forecasted rent growth (1.65%/yr at ZIP) by 1.09 pp."

For multi-address tables, after the table append a "Spread" or "Range" line: "Median income spread across the 3 addresses: $65.8k–$72.5k (10.2% range)."

For ranking tables, after the ranked list append: "Top 1 vs median of the set: +X.X pp / X×. Top 1 vs runner-up: +X.X pp." This tells the user whether the winner is decisive or marginal.

Skip this only when literally one value is shown (no comparison possible) or the user explicitly asked for raw numbers only.

## §Snapshot — default output mode

**This is the default shape for any bare or vague single-location request** ("tell me about 123 Main St", "what's going on at this ZIP?", "is this a good area?", or just an address / ZIP pasted with no instruction). Markdown only — no HTML, no branded header, no fixed container. Cite every field's vintage inline per R14.

### Build from the standard discovery batch

```
get_section_data(address, "income")      → income table + home value
get_section_data(address, "expansion")   → population growth CAGR
get_section_data(address, "crime")       → Block Group offense counts
```
Skip `schools` unless the user mentioned schools. Route around bad-list fields per R13 (e.g., swap in a `query_gis_field` on `MHIGRWCYFY` if you need forecasted income growth).

### Layout (in order)

1. **Header** — one line: `address · locality · county · ZIP`
2. **Income table** — median household income at **Block Group / Tract / ZIP / County** (cite `(Esri 2024)` or whatever vintage the section reports).
3. **Population growth CAGR** — **Tract / ZIP / County**, forecast window noted (e.g., `2024 → 2029`).
4. **Home value** — **Block Group / Tract / ZIP**, from `income.median_home_value`.
5. **Crime** — **top 3-4 offense categories** by Block Group count, raw counts only (no per-capita unless `TOTPOP_CY` also returned at Block Group — R11).
6. **Closing line** — render exactly:

   > *Want to go deeper? I can add: schools · natural hazards · income distribution · occupation mix · education mix · rental market · full OM page.*

### Scope behavior

- **User gave a street address** → include all four scales where available (Block Group / Tract / ZIP / County).
- **User gave a ZIP or city (no street address)** → **silently omit Block Group rows** from the income and home-value tables; they are not meaningful at that scope. Do not call them out, do not leave blanks.
- Any individual scale that returns null → omit that row per R9. Do not write "N/A".
- R7 cross-scale deltas still apply — add a short delta line under each multi-scale table.

### What §Snapshot is NOT

- Not a brief. Do not expand into prose sections for occupations, education, hazards, tenure. Those are gated behind the closer and only appear if the user asks.
- Not a template. If a section has no data, delete it — do not pad.

## §Quick — one-liner with comparison

For "What's X at this address?" style questions. One sentence + inline comparison across Block / Tract / ZIP.

Example structure:
> Median household income at **{address}** is **$X** (Block Group) vs **$Y** (Tract) vs **$Z** (ZIP Code).

If only one level returns data, state that explicitly:
> Block Group data unavailable for this address. Median household income is **$Y** (Tract) vs **$Z** (ZIP Code).

Keep it to 1–2 sentences. No table unless the user asks for one.

## §Compare — 3-column markdown table (Block / Tract / ZIP)

For "Give me the full demographics at this address" or "Compare Block / Tract / ZIP".

Plain markdown table. One metric per row. Column order: Metric · Block Group · Tract · ZIP.

```
| Metric              | Block Group | Tract    | ZIP      |
|---------------------|-------------|----------|----------|
| Population          | 1,234       | 5,678    | 12,345   |
| Median Age          | 34.2        | 36.1     | 38.4     |
| Median HH Income    | $72,500     | $68,200  | $65,800  |
| Owner-Occupied %    | 58%         | 62%      | 65%      |
```

Format rules:
- Currency: `$` prefix, comma-separated, no decimals for income
- Percentages: one decimal place
- Counts: comma-separated
- Numeric columns are right-aligned naturally in rendered markdown

Add a County column only if the user asks or if you're computing Diff values. Add a State column only on explicit request.

## §MultiAddr — multi-address comparison table

For "Compare these 3 neighborhoods" or "Show me these 5 addresses side-by-side".

Rows = addresses, Columns = metrics. Include a `Level` column indicating the smallest-working geographic level per address (from F3 cache) so the user knows what they're comparing.

```
| Address              | Level | Population | Med Age | Med HH Inc | Owner% |
|----------------------|-------|------------|---------|------------|--------|
| 123 Main St, KC MO   | Block | 1,234      | 34.2    | $72,500    | 58%    |
| 456 Oak Ave, STL MO  | Tract | 5,678      | 36.1    | $68,200    | 62%    |
| 789 Elm Rd, Omaha NE | ZIP   | 12,345     | 38.4    | $65,800    | 65%    |
```

If address coverage differs significantly (e.g. some rows at Block, others at Tract), note that above the table. Do not silently mix levels without calling it out.

## §Brief — discovery-driven markdown brief (opt-in only)

**Opt-in only.** The default shape for address requests is §Snapshot. Switch to §Brief only when the user explicitly asks for one of:

- "due diligence"
- "full brief"
- "everything you have"
- a named research artifact ("neighborhood analysis", "investment memo", "full workup")

If the user just said "tell me about this address" or pasted an address with no instructions, use §Snapshot and let them opt into a Brief via the §Snapshot closer line. Do NOT pre-emptively run a Brief because the user sounds serious.

### This is NOT a fixed template

The Brief is built from what the discovery batch actually returned. Sections appear ONLY if their underlying data came back. There is no "always include" section list. Apply R9 strictly: missing section → omit whole section.

### Build order

**Step 1** — Run the discovery batch (from SKILL.md §Discovery-First Workflow):
```
get_section_data(address, "income")
get_section_data(address, "expansion")
get_section_data(address, "crime")
get_section_data(address, "schools")    [only if user asked about schools]
```

**Step 2** — Check which sections returned data. Plan the brief around only those sections.

**Step 3** — Targeted `query_gis_field` fill-in for Tier 1 fields the user explicitly needs (occupations, education, income distribution buckets). Skip any section that depends on Tier 3 / Tier X fields unless data actually came back.

**Step 4** — Assemble. Use only sections where data is present.

### Available sections (each conditional)

- **Summary** — 2-3 sentences restating the most relevant numbers from discovery. Only include metrics that actually returned. No adjectives that aren't tied to a specific number (R10).
- **Income** — from `get_section_data("income")`. Include only the fields that returned: `median_household_income` (table showing Block/Tract/ZIP/County/State/National if all six returned; otherwise only the levels that returned), `median_home_value`, `annual_forecasted_median_income_growth`. Omit the section entirely if the income call failed.
- **Growth** — from `get_section_data("expansion")`. Population growth CAGR at the levels that returned. Omit the section entirely if the expansion call failed.
- **Crime** — from `get_section_data("crime")`. Raw offense counts at Block Group. Do NOT compute per-capita rates unless `TOTPOP_CY` also returned at Block Group (R11). Omit section if crime call failed.
- **Schools** — from `get_section_data("schools")`. Top 3 schools with ratings and district name. Omit if schools call failed or if user didn't ask about schools.
- **Occupation mix** — only if a Step 3 query_gis_field batch for all 13 occupation fields returned at the same layer. Present as a small table of WC / Services / BC percentages. Omit if any component missing (R11).
- **Education mix** — only if all 5 Tier 1 education fields (NOHS, HSGRAD, SMCOLL, BACHDEG, GRADDEG) returned at the same layer. Use the reduced 4-bucket formula from `computations.md`. Omit if any component missing.
- **Income distribution** — only if all 9 HINC buckets + `TOTHH_CY` returned at the same layer. Percentages table. Omit if any missing.
- **Natural Hazards** — only if the user explicitly asked for hazards AND the FEMA NRI tract query returned. List only hazards that came back with a non-null `_RISKR` rating.

### Format rules

- No branded header
- No fixed container width, color palette, fonts, CSS
- No forced HTML — markdown only unless the user asks for HTML
- Prose paragraphs ≤ 3 sentences each
- Every number appears alongside a comparison (another level, County Diff, or trend)
- Per R9: blank rows/sections are OMITTED, not filled with "N/A"
- Per R10: no qualitative language beyond what numbers literally show

### If nothing returned

If the entire discovery batch failed (rare — these sections are 100% reliable in past logs), tell the user honestly:
> "I couldn't pull reliable data for this address. Here's what I tried: [list of calls] and here's what failed: [failures]. Is there a different address, or a specific metric you're targeting?"

Do NOT invent a brief from general knowledge.

## §Bulk — CSV for many addresses

For "Build a database with these N addresses".

**No quota gates.** VestMap is unlimited (F6). Do NOT call `vestmap_account` as a pre-flight. Do NOT estimate call costs. Do NOT ask the user to confirm before starting. Just run it.

**Execution:**
- Run addresses in parallel batches of 10-20 (multiple parallel tool calls per turn)
- Emit CSV rows incrementally (append after each address succeeds) so a mid-batch failure doesn't lose completed work
- Filename pattern: `vestmap-bulk-{YYYYMMDD-HHMMSS}.csv` in the current working directory

**CSV header:**
```
address,level,metric_1,metric_2,...
```

Where `level` is the smallest-working geographic level for that row (per F3). For mixed-level outputs, either use one row per (address, level) combo OR per-level columns — document the convention in the output message.

**On individual address failure:** drop that address from the CSV. Do not retry. Do not interrupt the batch.

## §Rank — ranked list for superlative questions

For "which ZIP / city / county has the highest / lowest X in [region]?" or "top 10 [units] by [metric]".

**Workflow** (canonical version in `SKILL.md` Section 4):

1. **Enumerate the candidate set.** State ZIPs from your own training-data knowledge:
   - Colorado: ~526 ZIPs in the 80000-81658 range (with gaps)
   - Texas: ~1,800 ZIPs across 75000-79999 + 77xxx ranges
   - Wyoming: ~180 ZIPs in 82000-83128
   - For unfamiliar regions, use `search_real_estate_data` to find an enumeration layer
2. **Run parallel `get_section_data` calls** — one per candidate, one section per call. Use the section containing the target metric:
   - Population growth → `expansion` (extract `.zip` or appropriate level)
   - Median income → `income` → `median_household_income.zip`
   - Median home value → `income` → `median_home_value.zip`
   - Forecasted income growth → `income` → `annual_forecasted_median_income_growth.zip`
   - Crime → `crime` (extract counts; for ZIP-level rankings note that crime is Block-Group-only at the source — use `query_gis_field` on the crime layer per representative ZIP address if you need ZIP-level)
3. **Drop blanks** (R9). Candidates that returned null are excluded — no retry, no "N/A" entry.
4. **Sort** by target metric, descending for "highest" / ascending for "lowest".
5. **Trim** to top N (default 10 unless user specified a number).

**No quota concerns. No confirmation. No "this might take a while" hedging. Run all the candidates in parallel and rank them.**

**Output shape:**

```
**Top 10 Colorado ZIPs by 5-year population growth forecast** *(`get_section_data("expansion").zip`)*

| Rank | ZIP   | Pop Growth CAGR (2024-2029) |
|------|-------|------------------------------|
| 1    | 80027 | +2.41%                       |
| 2    | 80111 | +1.98%                       |
| ...  | ...   | ...                          |
```

Always include the data-source note (which tool call + which response key) so the user can verify.

**Rules:**
- No call-count summary at the end (the user doesn't care)
- No quota or cost language anywhere
- R9/R10 still apply: blank entries are dropped, no qualitative narrative around the table
- If the entire ranking returned empty (e.g., the section call universally failed across the candidate set — very rare), THEN tell the user honestly. Otherwise just present the ranking.

## §Map — optional neutral Leaflet + ESRI recipe

ONLY when the user explicitly asks for a map, visualization, or rendered location view. Do not add maps proactively.

Minimal neutral recipe. No branding, no color palettes, no specific container sizing.

**Dependencies (CDN):**
```html
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
```

**Geocoding (if lat/lng not supplied):**
```javascript
fetch(`https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(address)}`)
  .then(r => r.json())
  .then(data => {
    if (data.length > 0) {
      const lat = parseFloat(data[0].lat);
      const lng = parseFloat(data[0].lon);
      initMap(lat, lng);
    }
  });
```

**Map initialization:**
```javascript
const map = L.map('map', {
  scrollWheelZoom: false,
  dragging: false
}).setView([lat, lng], 13);

L.tileLayer(
  'https://server.arcgisonline.com/ArcGIS/rest/services/World_Street_Map/MapServer/tile/{z}/{y}/{x}',
  { attribution: 'Tiles © Esri', maxZoom: 19 }
).addTo(map);

L.marker([lat, lng]).addTo(map);  // Default Leaflet marker — no custom SVG
```

**What NOT to add:**
- No custom pin SVG
- No brand colors (no teal, no fixed palette)
- No container `<div>` width (let the user's layout decide)
- No fonts (`@import`), no typography scale
- No surrounding header / footer
- No panel grid

If the user wants a specific design, they'll supply it. The skill provides the capability, not the aesthetic.

## Reminder: No OM Template

This skill deliberately does NOT have an "OM HTML page" output mode. If the user asks for an HTML OM:
1. Clarify what shape / format they want (or use §Brief + §Map as a neutral starting point)
2. Do NOT reach for the archived `om-page` skill
3. Do NOT reintroduce VestMap color palettes, fonts, or fixed layouts

Methodology and output style are decoupled on purpose.
