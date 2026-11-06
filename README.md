# Earthenware plugin

Companion [Grok Build](https://x.ai/partners/marketplace) plugin (skills only) for the [Earthenware Kubernetes scheduler](https://github.com/fitzyracing1/earthenware-k8s).

This repository is not the scheduler itself. The Go plugin, agent, manifests, and tests live in [fitzyracing1/earthenware-k8s](https://github.com/fitzyracing1/earthenware-k8s). Related projects: [earthenware-tree](https://github.com/fitzyracing1/earthenware-tree), [heatgate](https://github.com/fitzyracing1/heatgate).

## What this plugin gives an agent

- `earthenware-scheduler` — bin-packing on requested CPU+memory, optional heat filter on measured `earthenware.io/temp-c`, honest status
- `earthenware-install` — build and apply steps from the earthenware-k8s Dockerfile and `deploy/` manifests
- `earthenware-tree` — how `deploy/tree.json` maps host `C` and `VM1`–`VM4` slots onto Kubernetes nodes

There is no MCP server and no shell hooks. Install steps apply only files from an earthenware-k8s checkout (no `curl | bash`, no remote binary exec).

## Honest status

The scheduler compiles and its unit tests pass. It has not been tested on a real cluster, and no container image is published yet. Do not invent benchmark numbers or production-ready claims.

Using these skills does not require a live cluster. Scheduling workloads with Earthenware does: deploy the scheduler from earthenware-k8s and set `schedulerName: earthenware` on those pods. The heat filter is optional (`maxTempC` plus a measured `earthenware.io/temp-c` annotation). heatgate is not required.

## License

Apache-2.0. See [LICENSE](LICENSE).
