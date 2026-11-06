---
name: earthenware-scheduler
description: >
  Explain the Earthenware Kubernetes scheduler plugin (fitzyracing1/earthenware-k8s):
  MostAllocated-style bin-packing on requested CPU+memory, optional heat filter on
  measured annotation earthenware.io/temp-c, heatgate no-guessing rule, honest status.
  Use when the user mentions Earthenware, earthenware-k8s, earthenware scheduler,
  or heatgate in a scheduling context.
when-to-use: >
  User asks what Earthenware does, how scoring/filtering works, how it relates to
  earthenware-tree or heatgate, or whether it is production-ready / has benchmark
  numbers. Prefer earthenware-install for apply steps; earthenware-tree for tree.json
  slot mapping.
---

# Earthenware scheduler (overview)

Source of truth for the scheduler: https://github.com/fitzyracing1/earthenware-k8s  
Related: [earthenware-tree](https://github.com/fitzyracing1/earthenware-tree), [heatgate](https://github.com/fitzyracing1/heatgate).

Earthenware is an **out-of-tree Kubernetes scheduler plugin** (wrapped `kube-scheduler`, pinned to Kubernetes **v1.31.4** modules). It implements the Earthenware Computer idea: pack small jobs into isolated slots on as few hosts as possible, acting only on **measured** data (heatgate principle) — never estimates.

## What it does

### Score (bin-packing)

Nodes that are already busier on **requested** CPU + memory (fraction after placing the pod) score higher, so pods fill fewer nodes and idle nodes can be scaled away / powered down. This is MostAllocated-style scoring on scheduler **requests**.

**Important:** Score uses scheduler *requests*, not live usage. The agent records live CPU (`earthenware.io/cpu-pct`) but that is **not** used for scoring yet. Do not claim otherwise.

### Filter (optional heat gate)

- If `maxTempC > 0`, nodes whose annotation `earthenware.io/temp-c` is **above** the threshold are Unschedulable.
- If the annotation is **missing or unparseable**, the node is **not** filtered (no guessing). That is logged at `--v=2`.
- Never invent a temperature. Never treat absent sensors as hot or cold.

### Agent (`agent/`)

DaemonSet that writes:

- `earthenware.io/temp-c` — max of `/sys/class/thermal/*/temp`, or heatgate if installed
- `earthenware.io/cpu-pct` — from `/proc/stat`

With no sensor (typical VMs) it writes **no** temperature, removes any stale one, and logs that.

### Tree-aware mode

When `treePath` is set in plugin args (see `deploy/scheduler.yaml`), the scheduler fills slots from `deploy/tree.json`. Without `treePath`, it falls back to plain request-based bin-packing. Details: skill `earthenware-tree`.

Only pods with `schedulerName: earthenware` use this scheduler / occupy tree slots.

## Honest status (do not over-claim)

Preserve these facts from the repo README — do **not** invent metrics, dollar/watt figures, or production-ready claims:

- Compiles and unit tests pass (scoring and heat filter logic). CI: vet, build, test, agent syntax check, kubeconform on manifests.
- **Not yet tested on a real cluster.**
- **No container image is published yet** (build locally / kind-load the `:dev` tag).
- Benchmark: point at `bench/run.sh`. **There are no results to treat as authoritative until you run it** on a host with Docker + kind + kubectl. Results (when produced) go to `bench/results-*.txt`. Do not derive dollar or watt figures.
- Score uses requests, not live agent CPU.

## Path to upstream (context only)

README outlines: publish + run kind/hardware benchmarks with raw measured results → integration tests / tagged image → propose to kubernetes-sigs/scheduler-plugins (upstream already has `NodeResourcesFit` MostAllocated; measured-temperature filter is the main new idea).

## Security

- Prefer the install skill's kubectl/apply flow from **this repo's** manifests. No `curl | bash`, no remote binary download+exec, no eval of remote content.
- Do not exfiltrate cluster secrets or kubeconfigs.
- This plugin pack ships **skills only** (no shell-exec MCP).

## Related skills

- `earthenware-install` — build image tag, ConfigMaps, apply manifests, `schedulerName`, `maxTempC`
- `earthenware-tree` — `tree.json` host `C` vs `VM1..VM4` slots
