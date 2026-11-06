---
name: earthenware-install
description: >
  Install the Earthenware Kubernetes scheduler from fitzyracing1/earthenware-k8s
  using the Dockerfile and deploy/ manifests in fitzyracing1/earthenware-k8s. Use when the user asks to
  install, deploy, or set up Earthenware / earthenware-k8s / the earthenware
  scheduler on a cluster.
when-to-use: >
  Building the scheduler image, applying deploy/scheduler.yaml and deploy/agent.yaml,
  creating tree/agent ConfigMaps, setting schedulerName: earthenware, or configuring
  maxTempC / treePath in earthenware-scheduler-config.
---

# Install Earthenware from this repo

Work from a checkout of https://github.com/fitzyracing1/earthenware-k8s. Use the paths and commands below as written in the README — do not substitute remote install scripts.

## Prerequisites

- A Kubernetes cluster and `kubectl` context you control
- Docker (or compatible) to build the scheduler image
- For the kind benchmark only: Docker + kind + kubectl, then `bench/run.sh`

Honest status: **not yet tested on a real cluster**; **no image is published** to GHCR yet. You must build (and load/push) the `:dev` tag yourself.

## 1. Build the scheduler image

```sh
docker build -t ghcr.io/fitzyracing1/earthenware-scheduler:dev .
```

Then either push that tag to a registry your cluster can pull, or `kind load docker-image ghcr.io/fitzyracing1/earthenware-scheduler:dev` (or the equivalent for your cluster). `deploy/scheduler.yaml` references exactly that image name/tag.

## 2. Tree ConfigMap

```sh
kubectl -n kube-system create configmap earthenware-tree --from-file=deploy/tree.json
```

The scheduler Deployment mounts this at `/etc/earthenware-tree` (see `deploy/scheduler.yaml`). Plugin args set `treePath: /etc/earthenware-tree/tree.json`.

## 3. Apply scheduler RBAC, config, and Deployment

```sh
kubectl apply -f deploy/scheduler.yaml
```

This creates:

- ServiceAccount `earthenware-scheduler` + scheduler RBAC bindings
- ConfigMap `earthenware-scheduler-config` with `KubeSchedulerConfiguration`
  - `schedulerName: earthenware`
  - Earthenware on filter + score (weight 10); disables `NodeResourcesBalancedAllocation` and `NodeResourcesFit` in that profile
  - Plugin args: `maxTempC: 85`, `treePath: /etc/earthenware-tree/tree.json`
- Deployment in `kube-system` on the control-plane node (nodeSelector + toleration)

To change the heat threshold later, edit the `earthenware-scheduler-config` ConfigMap (`maxTempC`). Set `maxTempC` to `0` (or omit heat gating per code/docs) only if you intend to disable the heat filter — follow the plugin args semantics in this repo; do not invent flags.

## 4. Agent DaemonSet + agent ConfigMap

```sh
kubectl apply -f deploy/agent.yaml
kubectl create configmap earthenware-agent -n kube-system --from-file=agent/agent.py --dry-run=client -o yaml | kubectl apply -f -
```

`deploy/agent.yaml` ships an empty ConfigMap placeholder; the second command populates it from `agent/agent.py`. The DaemonSet runs `python:3.12-slim` with that script, patches node annotations, and mounts `/sys/class/thermal` read-only.

## 5. Schedule pods with Earthenware

Set on each pod (or pod template) that should use this scheduler:

```yaml
schedulerName: earthenware
```

Only those pods use the Earthenware profile / occupy tree slots when `treePath` is set.

## 6. Labels for tree VM groups (when using tree mode)

Per README / tree mapping: label worker nodes `earthenware.io/vm=VM1` … `VM4` to match `deploy/tree.json` (see skill `earthenware-tree`). `bench/kind-config.yaml` shows the kind layout assumption (one node per VM group).

## Benchmark (optional)

```sh
bench/run.sh
```

Creates a 5-node kind cluster (control plane = host `C`, workers = VM1..VM4), schedules the same pod set with `default-scheduler` and `earthenware`, and records measured nodes used, schedule latency, pods per node, and a `docker stats` CPU% sample. **There are no results until you run it.** Do not invent dollar/watt figures. Output path: `bench/results-*.txt`.

## Security / agent rules

- Apply only manifests from an **earthenware-k8s** checkout (`deploy/`, `agent/agent.py`). No `curl | bash`, no downloading and executing remote binaries, no `eval` of remote content.
- Do not broaden RBAC beyond what these manifests request.
- This plugin pack has no MCP server — do not add a shell-exec MCP to “help” install.
- Remind the user the project is experimental: unit-tested, not production-certified.

## Related skills

- `earthenware-scheduler` — scoring, heat gate, honest status
- `earthenware-tree` — tree.json ↔ Kubernetes mapping
