# AI Agent Guide — swap (California SWAP 2025)

A geo-agent map app for **California's State Wildlife Action Plan (SWAP) 2025** (CDFW).
Created from [boettiger-lab/geo-agent-template](https://github.com/boettiger-lab/geo-agent-template).

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

California SWAP 2025 is published by CDFW under CC-BY-4.0 and reprocessed by the Boettiger Lab.
The suite lives under the `swap-2025-*` STAC collections (bucket `public-cdfw/swap-2025/`):

| Collection | Map layer | Notes |
|---|---|---|
| `swap-2025-provinces` | ✅ Provinces (fill) | 8 analysis regions |
| `swap-2025-conservation-units` | ✅ Conservation Units | 46 units; `ParentProvinceID` → Provinces `ProvinceID` |
| `swap-2025-bay-delta-conservation-unit` | ✅ | single Bay-Delta polygon |
| `swap-2025-sgcn-ranges` | ✅ SGCN ranges | 422 species range polygons |
| `swap-2025-noaa-esu-dps` | ✅ Salmonid ESU/DPS | 37 anadromous units |
| `swap-2025-marine-bioregions` | ✅ | 3 marine polygons |
| `swap-2025-sgcn-species` | — (SQL) | SGCN species list; `SpeciesKey` ← ranges `ParentSpeciesKey` |
| `swap-2025-sgcn-species-units` | — (SQL) | SGCN × Conservation Unit crosswalk |
| `swap-2025-targets` | — (SQL) | Conservation Targets (reference units by `Name`) |
| `swap-2025-strategies` | — (SQL) | Conservation Strategies (reference units by `Name`) |

These are children of the `cdfw-datasets` parent collection, not of the root catalog, so each entry
in `layers-input.json` sets an explicit `collection_url` (pointing at the sub-collection JSON) to
bypass root-catalog traversal.

> **Hex caveat:** each polygon layer has an H3 hex variant where one row = one (feature, cell) pair.
> `Shape__Area` / `Shape__Length` repeat on every cell — never `SUM` them on hex. `_cng_fid` / `h8` / `h0`
> are the safe keys. This is documented in each STAC collection description (`get_schema`).

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
