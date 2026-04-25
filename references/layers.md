# VestMap ESRI Layer Catalog

All layer URLs used by the VestMap MCP tool, with coverage notes and known limitations.

## Primary Demographics Service

Current URL (as of this writing): `https://demographics5.arcgis.com/arcgis/rest/services/USA_Demographics_and_Boundaries_2024/MapServer`

The `_2024` suffix is the vintage; newer vintages will eventually supersede it. **Do not memorize service URLs or layer numbers from this file** — always pass the metric through `search_real_estate_data` and use the URL it returns. What follows is a historical snapshot to orient you, not a routing authority.

Observed layer IDs on the 2024 service:

| Level | Layer URL | Notes |
|---|---|---|
| Block Group | `…/MapServer/12` | Finest geography. |
| Tract | `…/MapServer/11` | FEMA NRI natively uses tract boundaries. |
| ZIP Code | `…/MapServer/9` | Broadest standard geography below county. |
| County | `…/MapServer/7` | Used for Diff baselines and rural fallback. May lack some forecast-growth fields (those have been observed at `/22` or `/23` on the 2024 service — verify per query). |
| State | `…/MapServer/3` | Historical logs against older services marked this "0% — never use"; that was wrong for the 2024 service, which returns forecast-growth fields cleanly here. Verify per query. |
| Place / City | `…/MapServer/20` | Historically unreliable; may or may not hold data for a given metric. Verify per query. |
| National | `…/MapServer/2` | Observed for forecast-growth fields on the 2024 service. Verify per query. |

Reliability of any (layer, field) pair is service-specific and can change when the service updates. Don't prune layers based on old percentages. Try the canonical source returned by `search_real_estate_data`; if it returns null, handle per R9/F1.

## Crime Data (Block Group Only)

URL: `https://demographics5.arcgis.com/arcgis/rest/services/USA_Crime_2024/MapServer/12`

- Coverage: Block Group level ONLY
- No Tract / ZIP / County crime layer exists in this service
- Contains total, personal, and property crime indices + specific offense counts per 100k
- **Prefer `get_section_data(address, "crime")`** — it bundles these values into a single call

## FEMA National Risk Index (Tract Only)

URL: `https://services.arcgis.com/XG15cJAlne2vxtgt/arcgis/rest/services/National_Risk_Index_Census_Tracts/FeatureServer/0`

- Coverage: Census Tract level ONLY
- All 18 hazard types (earthquake, hurricane, tornado, drought, wildfire, flooding, etc.) live on this one layer
- Composite scores (`SOVI_SCORE`, `RESL_SCORE`, `RISK_SCORE`, `RISK_RATNG`) also on this layer
- See `fields.md` §FEMA for full hazard field list
- Do NOT try to query hazards at Block Group or ZIP — they don't exist there

## CBSA / MSA Business Data (Metro Area Only)

URL: `https://services5.arcgis.com/9fQmObndozAJu9f5/arcgis/rest/services/Enriched_USA_Metropolitan_Statistical_Areas/FeatureServer/0`

- Coverage: Core Based Statistical Area (metro) level ONLY
- Fields: `N01_BUS` (total businesses), `N01_EMP` (total employees)
- Do NOT try to query business counts at Block / Tract / ZIP / County — they don't exist at finer levels
- If an address is outside any defined metro, business data is unavailable

## Layer Selection by Use Case

| Use case | Layer to query |
|---|---|
| "Median income at address" comparison | `/12`, `/11`, `/9` in parallel (Block, Tract, ZIP) |
| Income distribution Diff vs County | `/12` for buckets + `/7` for County baseline |
| Crime index at address | `get_section_data("crime")` — bundles Block Group crime |
| Natural hazard risk | FEMA NRI tract layer only |
| Business density (metro) | CBSA layer only |
| Schools nearby | `get_section_data("schools")` — bundled |
| House price trends | `get_section_data("hpi")` — bundled |
| Population growth | `POPGRWCYFY` at `/9`, `/11`, `/12` OR `get_section_data("expansion")` |
| Neighborhood character | `get_section_data("neighborhood")` — bundled |
| Demographics overview | `get_section_data("demographics")` — bundled |
| Income overview | `get_section_data("income")` — bundled |

## Service vintage (R14 Axis 2)

When `search_real_estate_data` returns hits on multiple services for the same field, prefer the one with the newest year tag in its URL: `..._2024` > `..._2021` > `..._2020`. Section payloads in `get_section_data(...)` are wired to specific services and may still serve older vintages (see `gotchas.md` → Illustrative drift examples). For quantitative output, route through the newest-service URL the search tool returns rather than the section.

ACS alternative layers at `USA_ACS_2024/MapServer/{6,7,10,12,23,25}` — different field naming (`ACSXXX` prefix, `B25XXX` Census table codes). Per R14 Axis 1, prefer Esri `_CY` over ACS when both hit. **Always discover via `search_real_estate_data` first** — do not guess ACS field names or layer indices.

## Layer Strategy

**Default for multi-scale comparison:** query Block Group, Tract, and ZIP in parallel on the newest-vintage service. Use whichever return data — no need to retry failures sequentially.

**Single-level lookups:** start with the scale the user asked about. ZIP is usually the broadest safe single choice. Don't pre-exclude layers based on historical percentages — try the canonical source and handle failure per R9/F1.

**Sparse-coverage addresses:** if the parallel Block/Tract/ZIP batch returns all-null, escalate outward (County, then State/National) — topology permitting, per whatever the metric's search result shows. Never re-query a failed layer (F1). Cache the smallest-working-level per address (F3).
