# State Wildlife Action Plans Explorer

An AI-powered interactive map for comparing **U.S. State Wildlife Action Plans** — how different
states plan for wildlife conservation. It currently federates three deliberately different plans:

- **California — SWAP 2025 (CDFW):** a relational plan — provinces → conservation units, SGCN species
  + ranges, and Conservation Targets & Strategies planning tables.
- **Nevada — SWAP 2022 (NDOW):** a flat, species-centric plan — 258 species distributions + 17 key
  habitat classes.
- **Missouri — CCS 2022 (MDC):** a habitat/place-based *Comprehensive Conservation Strategy* (not
  called a "SWAP") — Conservation Opportunity Areas by habitat system + priority geographies, with no
  species layer.

Each state's layers are kept in their **native structure** and labeled by state (`CA ·`, `NV ·`,
`MO ·`) so users can see and manually compare the different approaches. US-wide context layers
(PAD-US protected areas, NLCD land cover, SVI social vulnerability, Census boundaries) overlay any state.

Ask questions in plain language; the app uses an LLM agent with map tools and SQL access. Built on the
[geo-agent](https://github.com/boettiger-lab/geo-agent) framework — the map, chat, and agent modules
are loaded from CDN, so there is no JavaScript to write here.

## Configuration

| File | Purpose |
|---|---|
| `index.html` | HTML shell — loads core JS/CSS from the geo-agent CDN |
| `layers-input.json` | Which SWAP collections to show + map/LLM settings |
| `system-prompt.md` | LLM persona, domain context, and guardrails |
| `k8s/` | Kubernetes deployment manifests (app slug `swap`, namespace `biodiversity`) |

## Data & attribution

State plan data © the respective agencies, reprocessed to cloud-native formats by the
[Boettiger Lab](https://boettigerlab.berkeley.edu), UC Berkeley:

- **California — SWAP 2025**, California Department of Fish and Wildlife — **CC-BY-4.0** ([wildlife.ca.gov/SWAP](https://wildlife.ca.gov/SWAP)).
- **Nevada — SWAP 2022**, Nevada Department of Wildlife — license **pending confirmation**; source [NDOW ArcGIS Data Hub](https://nevada-department-of-wildlife-data-hub-ndow.hub.arcgis.com/).
- **Missouri — CCS 2022**, Missouri Department of Conservation — license **pending confirmation**; source [2022 Missouri CCS](https://mdc.mo.gov/sites/default/files/2022-04/2022-Missouri-CCS.pdf) (MDC ArcGIS COA services).

Ingestion is tracked in data-workflows issues [#381](https://github.com/boettiger-lab/data-workflows/issues/381) (CA), [#383](https://github.com/boettiger-lab/data-workflows/issues/383) (NV), and [#384](https://github.com/boettiger-lab/data-workflows/issues/384) (MO). NV/MO are held out of source.coop until licenses are resolved.

## Deployment

See [`AGENTS.md`](./AGENTS.md) for the full deployment workflow. In short: edit the source files,
push to `main`, then `kubectl rollout restart -f k8s/deployment.yaml` — the pod's initContainer
re-clones `main` on restart.
