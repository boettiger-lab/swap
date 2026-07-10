# California SWAP 2025

An AI-powered interactive map for **California's State Wildlife Action Plan (SWAP) 2025** — the
California Department of Fish and Wildlife (CDFW) conservation framework of provinces, conservation
units, Species of Greatest Conservation Need (SGCN), and the strategies and targets that guide their
recovery.

Ask questions in plain language; the app uses an LLM agent with map tools and SQL access to visualize
and analyze the data.

Built on the [geo-agent](https://github.com/boettiger-lab/geo-agent) framework — the map, chat, and
agent modules are loaded from CDN, so there is no JavaScript to write here.

## Layers

- **SWAP Regions** — Provinces (8), Conservation Units (46), Bay-Delta Conservation Unit
- **Species of Greatest Conservation Need** — SGCN species ranges by taxonomic group
- **Marine & Anadromous** — Marine Bioregions, salmonid ESU/DPS units

Tabular planning data (SGCN species list, Conservation Targets, Conservation Strategies, and the
SGCN × Conservation Unit crosswalk) is available to the agent via SQL.

## Configuration

| File | Purpose |
|---|---|
| `index.html` | HTML shell — loads core JS/CSS from the geo-agent CDN |
| `layers-input.json` | Which SWAP collections to show + map/LLM settings |
| `system-prompt.md` | LLM persona, domain context, and guardrails |
| `k8s/` | Kubernetes deployment manifests (app slug `swap`, namespace `biodiversity`) |

## Data & attribution

California SWAP 2025, © California Department of Fish and Wildlife, licensed CC-BY-4.0
([wildlife.ca.gov/SWAP](https://wildlife.ca.gov/SWAP)). Cloud-native reprocessing by the
[Boettiger Lab](https://boettigerlab.berkeley.edu), UC Berkeley.

## Deployment

See [`AGENTS.md`](./AGENTS.md) for the full deployment workflow. In short: edit the source files,
push to `main`, then `kubectl rollout restart -f k8s/deployment.yaml` — the pod's initContainer
re-clones `main` on restart.
