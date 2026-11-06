---
name: earthenware-tree
description: >
  Explain how deploy/tree.json from earthenware-k8s maps Earthenware Computer
  layout (host C, VM1..VM4 slots) onto Kubernetes nodes and pod slots. Use when
  the user asks about earthenware-tree, tree.json, VM slots, or treePath in the
  Earthenware scheduler.
when-to-use: >
  Questions about tree.json structure, host C vs VM beginnings, block counts,
  earthenware.io/vm labels, fill order, or cmd/earthenware-layout output.
---

# Earthenware tree mapping

Do **not** invent tree structure. Read `deploy/tree.json` in [earthenware-k8s](https://github.com/fitzyracing1/earthenware-k8s) (copied from [earthenware-tree](https://github.com/fitzyracing1/earthenware-tree)). `pkg/tree` validates: one host, ternary (every parent ≤ 3 children), VMs link only to the host.

## Layout summary (from README + deploy/tree.json)

The cluster layout follows tree.json: **5 beginnings, 9 steps, 15 blocks**, ternary.

| tree.json | Kubernetes |
|---|---|
| host subtree `C` (1 beginning → 2 steps → 4 blocks, `kind: computer`) | control plane + Earthenware scheduler; gets **no** workload pods |
| each VM beginning `VM1..VM4` (linked only to `C` via `links`) | a node group: nodes labelled `earthenware.io/vm=VMn` |
| each block under a VM (`VM1.s1.b1` …) | one pod slot (VM1=4, VM2=4, VM3=2, VM4=1 → **11** slots) |

Verified counts from earthenware-k8s `deploy/tree.json`: beginnings `C`, `VM1`, `VM2`, `VM3`, `VM4`; VM block owners VM1:4, VM2:4, VM3:2, VM4:1.

`links` in the file: `VM1`–`VM4` → `C`.

## Scheduler behavior with `treePath`

With `treePath` set (default in `deploy/scheduler.yaml`: `/etc/earthenware-tree/tree.json`):

- Filter rejects nodes outside a VM group and full groups
- Score ranks earlier groups first (and fuller within a band)
- Fill order: VM1's blocks in order, then VM2, VM3, VM4
- Only pods with `schedulerName: earthenware` occupy slots

Without `treePath`, the plugin falls back to plain request-based bin-packing (see `earthenware-scheduler`).

**Assumption** (README): one Kubernetes node per VM group (as in `bench/kind-config.yaml`). With several nodes per group, the slot cap currently applies **per node**, not per group — say that plainly; do not invent a different semantics.

## Layout helper

`cmd/earthenware-layout` emits a tree.json-compatible file (all original keys kept, plus a `"pod"` field on each occupied block) from `kubectl get nodes/pods -o json`. Unit tests use the real tree.json and check counts, fill order, overflow, host exclusion, and round-trip.

## Related skills

- `earthenware-scheduler` — scoring and heat filter
- `earthenware-install` — ConfigMap from `deploy/tree.json` and apply steps
