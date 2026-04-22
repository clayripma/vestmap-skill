# VestMap ESRI Layer Catalog

All layer URLs used by the VestMap MCP tool, with coverage notes and known limitations.

## Primary Demographics Service

Base URL: `https://demographics5.arcgis.com/arcgis/rest/services/USA_Demographics_and_Boundaries_2024/MapServer`

Empirical field-success rates from past logs:

| Level | Layer URL | Field success | Preferred For |
|---|---|---|---|
| ZIP Code | `…/MapServer/9` | **100%** | **Most reliable single-level**. Default choice when you only need one level. Broadest standard geography. |
| Block Group | `…/MapServer/12` | 97.5% | Finest geography, nearly as reliable as ZIP. Use for neighborhood-scale comparisons. |
| Tract | `…/MapServer/11` | 97.5% | Middle tier — FEMA NRI natively uses tract boundaries. |
| County | `…/MapServer/7` | 88% | Rural fallback + baseline for income-distribution Diff. |
| State | `…/MapServer/3` | 0% (FAILING) | **Do NOT use.** Nothing returns here. |
| Place / City | `…/MapServer/20` | 28% | **Avoid.** Very unreliable. |

**Full URLs:**
- Block Group: `https://demographics5.arcgis.com/arcgis/rest/services/USA_Demographics_and_Boundaries_2024/MapServer/12`
- Tract: `https://demographics5.arcgis.com/arcgis/rest/services/USA_Demographics_and_Boundaries_2024/MapServer/11`
- ZIP: `https://demographics5.arcgis.com/arcgis/rest/services/USA_Demographics_and_Boundaries_2024/MapServer/9`
- County: `https://demographics5.arcgis.com/arcgis/rest/services/USA_Demographics_and_Boundaries_2024/MapServer/7`
- State: `https://demographics5.arcgis.com/arcgis/rest/services/USA_Demographics_and_Boundaries_2024/MapServer/3`

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

## Legacy / Alternative Layers (Use With Caution)

Prior-year snapshots exist (2020, 2021, 2022). Do NOT use these unless the user explicitly asks for historical data. The 2024 layers above are canonical.

ACS alternative layers at `USA_ACS_2024/MapServer/{6,7,10,12,23,25}` — different field naming (`ACSXXX` prefix, `B25XXX` Census table codes). Use only for fields not available in the primary Demographics 2024 service. **Always discover via `search_real_estate_data` first** — do not guess ACS field names or layer indices.

## Layer Strategy (NOT a strict fallback chain)

**Default strategy: query Block Group, Tract, and ZIP in parallel.** All three are 97.5-100% reliable. Use whichever return data — no need to retry failures sequentially.

**For single-level lookups** (e.g., user only wants one number): query ZIP (/9). It has the highest empirical success rate (100%) and is the safest single choice.

**For rural or sparse-coverage addresses:** if the parallel Block/Tract/ZIP batch returns all-null, escalate to County (/7). Do not try State — it has 0% success.

**Fallback chain (only for addresses where the parallel batch failed):**

**Block Group → Tract → ZIP → County → *(stop)*** 

Never attempt State (/3 or /19) or Place/City (/20) — empirical success rates are 0% and 28% respectively. Never re-query a failed layer (F1). Cache the smallest-working-level per address (F3).
