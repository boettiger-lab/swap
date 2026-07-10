# AI Agent Guide — swap (State Wildlife Action Plans Explorer)

A geo-agent map app comparing **U.S. State Wildlife Action Plans** across states — currently
**California SWAP 2025** (CDFW), **Nevada SWAP 2022** (NDOW), and **Missouri CCS 2022** (MDC) —
plus US-wide context layers. Created from
[boettiger-lab/geo-agent-template](https://github.com/boettiger-lab/geo-agent-template).

> **Federated, not unified.** The three state plans have deliberately different structures
> (CA is relational with species + planning tables; NV is species-ranges + key-habitats;
> MO is habitat-only Conservation Opportunity Areas with *no* species data and is a "CCS,"
> not a "SWAP"). Layers are grouped and labeled by state (`display_name` prefixed `CA ·`/`NV ·`/`MO ·`)
> and kept in their native schemas — do **not** merge them into one schema. See `system-prompt.md`
> for the per-state vocabulary map the agent relies on. Note Missouri's inconsistent name-column
> casing (`NAME` vs `Name`) across COA layers — tooltips are matched per-layer.

## Deployment model: public repo, git-clone initContainer

The pod's `git-clone` initContainer (`k8s/deployment.yaml`) runs
`git clone --depth 1 https://github.com/boettiger-lab/swap.git` on each pod start and copies
`index.html`, `layers-input.json`, and `system-prompt.md` into the nginx html dir. Pod content
tracks `main`. `k8s/configmap.yaml` holds only the LLM model list and the nginx reverse-proxy
template — **not** website content.

## Repo relationship

| Repo | Purpose |
|---|---|
| `geo-agent` | Core library (map, chat, agent, tools) loaded from CDN. Source of truth for all functionality. |
| `swap` | This application repo. Configure `layers-input.json`, `system-prompt.md`, and `k8s/` for the SWAP dataset. |

**You do not write JavaScript.** Core modules are loaded from the CDN pin in `index.html`.

**Full docs:** [boettiger-lab.github.io/geo-agent/docs](https://boettiger-lab.github.io/geo-agent/docs/)

## The data

All state plans are public agency releases reprocessed by the Boettiger Lab. Each state's collections
are children of a state parent collection (`cdfw-datasets`, `nevada-datasets`, `missouri-datasets`) —
**not** the root catalog — so each entry in `layers-input.json` sets an explicit `collection_url` to the
sub-collection JSON to bypass root-catalog traversal. Ingestion + authoritative sources are documented in
data-workflows issues **#381** (CA SWAP 2025), **#383** (NV SWAP 2022), **#384** (MO CCS 2022).

> **Licenses:** California is **CC-BY-4.0**; the STAC declares **Nevada and Missouri as `"other"` —
> redistribution license pending confirmation** (both held out of source.coop). Do not describe NV/MO
> data as CC-BY. Sources: NV → [NDOW ArcGIS Data Hub](https://nevada-department-of-wildlife-data-hub-ndow.hub.arcgis.com/);
> MO → MDC ArcGIS `gisblue.mdc.mo.gov/.../CCS/MapServer` + the [2022 CCS PDF](https://mdc.mo.gov/sites/default/files/2022-04/2022-Missouri-CCS.pdf).

**California — SWAP 2025 (CDFW)**, bucket `public-cdfw/swap-2025/`:

| Collection | Map layer | Notes |
|---|---|---|
| `swap-2025-provinces` | ✅ | 8 analysis regions |
| `swap-2025-conservation-units` | ✅ | 46 units; `ParentProvinceID` → Provinces `ProvinceID` |
| `swap-2025-bay-delta-conservation-unit` | ✅ | single Bay-Delta polygon |
| `swap-2025-sgcn-ranges` | ✅ | 422 vertebrate range polygons |
| `swap-2025-noaa-esu-dps` | ✅ | 37 salmonid ESU/DPS units |
| `swap-2025-marine-bioregions` | ✅ | 3 marine polygons |
| `swap-2025-sgcn-species` | — (SQL) | 1,437 SGCN species; `SpeciesKey` ← ranges `ParentSpeciesKey` |
| `swap-2025-sgcn-species-units` | — (SQL) | 4,268-row species × unit crosswalk (`ParentProvSpeciesKey`) |
| `swap-2025-targets` | — (SQL) | 460 Conservation Targets (join units by name `ConsUnit`) |
| `swap-2025-strategies` | — (SQL) | 525 Strategies (`ParentTargetID` → `TargetID`) |

**Nevada — SWAP 2022 (NDOW)**, bucket `public-nevada/swap-2022/`:

| Collection | Map layer | Notes |
|---|---|---|
| `nevada-swap-2022-species-distributions` | ✅ | 258 SGCN range polygons; `Major_Group` = Bird/Reptile/Mammal/Aquatic |
| `nevada-swap-2022-key-habitats` | ✅ | 17 `KEYHABCLAS` habitat classes |

**Missouri — CCS 2022 (MDC)**, bucket `public-missouri/ccs-2022/` — *habitat only, no species data*:

| Collection | Map layer | Notes |
|---|---|---|
| `missouri-ccs-2022-{grassland-prairie-savanna,forest-woodland,glade,cave-karst,wetland}-coa` | ✅ | COAs by habitat system; name col `NAME` **or** `Name` (varies) |
| `missouri-ccs-2022-stream-reach-coa` | ✅ | aquatic COAs — **line** features (`layer_type: "line"`) |
| `missouri-ccs-2022-priority-geographies` | ✅ | 11 landscape priorities; `NAME`, `GIS_Acres` |

**National context** (reused, proven): `pad-us-4.1-fee`, `pad-us-4.1-easement` (GAP-status colored),
`nlcd-2024` (categorical COG), `svi-2022` (tract-pmtiles), and `census-2024-{state,county,cd}` /
`census-2025-{sldu,sldl}` (collapsed "Administrative Boundaries" group).

> **Hex caveat:** each polygon layer has an H3 hex variant where one row = one (feature, cell) pair.
> `Shape__Area` / `Shape__Length` (Missouri: `Shape.STArea()`) repeat on every cell — never `SUM` them
> on hex. `_cng_fid` / `h8` / `h0` are the safe keys. Documented in each STAC description (`get_schema`).

## Deployment

> **If you lack credentials or permissions** to run `kubectl` or `git push`, do not attempt to
> discover or work around credentials. Instead, provide the user with the exact commands to run.

Deployment name + namespace come from `k8s/deployment.yaml` (`swap` / `biodiversity`) — the commands
below reference the manifest file so you cannot restart the wrong app.

```bash
# 1. Edit source files (index.html, layers-input.json, system-prompt.md)
# 2. Commit and push to main — the initContainer clones from GitHub
git add <source-files> && git commit -m "<message>" && git push
# 3. Restart the deployment so a new pod re-clones the latest main
kubectl rollout restart -f k8s/deployment.yaml
kubectl rollout status  -f k8s/deployment.yaml
```

The git push does **not** update running pods — step 3 does.

If you change `k8s/configmap.yaml` (LLM model list or nginx template), apply it before the rollout:
`kubectl apply -f k8s/configmap.yaml`. Routine content edits don't need this.

First-time setup: create the GitHub repo `boettiger-lab/swap`, push `main`, then `kubectl apply -f k8s/`.
The `open-llm-proxy-secrets` secret (key `proxy-key`) must already exist in the `biodiversity` namespace.

### CDN versioning

`index.html` pins the geo-agent library version. Verify jsDelivr serves a new tag before deploying:

```bash
curl -sI https://cdn.jsdelivr.net/gh/boettiger-lab/geo-agent@vX.Y.Z/app/style.css | grep HTTP
# Must return HTTP/2 200 — a 404 means jsDelivr hasn't indexed the tag yet
```

---

For the full `layers-input.json` schema, troubleshooting, and configuration reference see the
[geo-agent-template AGENTS.md](https://github.com/boettiger-lab/geo-agent-template/blob/main/AGENTS.md)
or the [docs](https://boettiger-lab.github.io/geo-agent/docs/).
