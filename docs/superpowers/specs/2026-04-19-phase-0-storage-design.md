# Phase 0 — Persistent Storage (local-path-provisioner)

**Status:** Implemented 2026-04-26 — ctrl01=221GiB free, work01=221GiB free
**Date:** 2026-04-19
**Owner:** @khreez

## Context

The cluster has no StorageClass, no CSI driver, and no PersistentVolumes. Every ServiceMonitor / PodMonitor in the repo is ready to be scraped, but Prometheus, Alertmanager, Grafana, Victoria-Logs, and Gatus — the observability stack coming in Phases 1–4 — all need persistent storage. Without it, they either fail to schedule or lose state on every restart.

This spec defines Phase 0 of the observability roadmap: a minimal, node-local StorageClass that lets stateful workloads run on the existing hardware. HA replication is explicitly out of scope; it is blocked on hardware growth (a third node and/or dedicated disks).

## Goals

- Provide a cluster-default StorageClass that the observability stack (and other stateful apps) can consume.
- Provide a second StorageClass with `Retain` reclaim policy for data that must survive accidental PVC deletion.
- Follow the repo's existing Flux + OCIRepository + HelmRelease pattern.
- Keep Talos machine-config changes to zero if possible.

## Non-goals

- HA / replicated storage (rook-ceph, longhorn, mayastor). Deferred until the cluster has ≥3 nodes with dedicated raw disks.
- Snapshots / CSI VolumeSnapshots. Backups will be addressed pod-level in a later spec.
- Enforced volume quotas. Size on PVC will be advisory; workloads must self-limit (e.g. Prometheus `retentionSize`).
- Topology awareness beyond `WaitForFirstConsumer`. A PVC on an offline node remains unavailable until the node returns.

## Current state

- 2 Talos nodes:
  - `ctrl01` — controller, `/dev/nvme0n1` (OS disk)
  - `work01` — worker, `/dev/nvme0n1` (OS disk)
- Both disks are fully consumed by the Talos default partition layout.
- `kubectl get sc|pv|csidrivers` → `No resources found`.
- `kube-prometheus-stack` CRDs are installed (via bootstrap helmfile) but no HelmRelease deploys the stack itself yet.

## Design

### Component

**Rancher `local-path-provisioner`** — filesystem-backed, node-local dynamic provisioner. One Deployment, one ServiceAccount, one ClusterRole, one ConfigMap, plus the StorageClasses. On PVC-bound events, a short-lived helper Pod mounts the node's hostPath and creates or deletes a subdirectory that becomes the PV.

### Alternatives considered

| Option | Verdict |
|---|---|
| OpenEBS LocalPV-hostpath | Functionally equivalent to local-path-provisioner, but adds a CRD-based control plane and feature surface (mayastor/zfs/lvm) we will not use. More moving parts for the same outcome. |
| TopoLVM | Correct long-term choice for enforced volume sizes. Requires carving an LVM volume group per node, which means repartitioning the OS disk (Talos reinstall). Too heavy for Phase 0. |
| Rook-ceph / Longhorn / Mayastor | Replicated storage, the right target. Blocked on hardware: rook-ceph needs ≥3 dedicated raw disks on ≥3 nodes; longhorn with 1 worker offers no HA. Revisit when hardware grows. |
| Rancher local-path-provisioner (chosen) | Smallest-footprint option that matches current hardware. Easy migration to a replicated backend later (recreate PVCs on new SC, rsync data, recreate workloads). |

### Hostpath layout

- Backing path: `/var/local-path-provisioner/` on each node.
- The path lives directly under Talos' ephemeral partition (mounted at `/var`), which is writable and persistent across reboots.
- The helper Pod will `mkdir -p` on first use — **no Talos `machineconfig` patch required**.
- **Why not `/var/mnt/...`?** Talos reserves `/var/mnt/` exclusively for user volumes declared in `machineconfig` — the overlay rejects runtime `mkdir` there even though `/var` itself is RW. Top-level paths under `/var` (e.g. `/var/local-path-provisioner`, `/var/openebs/local`) work without a machineconfig patch.
- Trade-off: storage shares space with kubelet state and container image cache. Acceptable for Phase 0 given expected footprint (see "Capacity" below).

### StorageClasses

Two SCs, both provisioner `rancher.io/local-path`:

**`local-path` (cluster default)**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: false
```

**`local-path-retain`**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path-retain
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
allowVolumeExpansion: false
```

`WaitForFirstConsumer` is mandatory: the PV must pin to the node where the consuming Pod is scheduled, because the data physically lives on that node's disk.

### Chart source

- **Primary:** `oci://ghcr.io/containeroo/charts/local-path-provisioner` (community Helm chart), consumed via an `OCIRepository` + `HelmRelease` pair — identical to every other app in the repo.
- **Fallback:** if the chart is unavailable or unreliable, vendor the raw YAML from `rancher/local-path-provisioner` and apply via pure Kustomize. Decision deferred to rollout; not architecturally load-bearing.

### HelmRelease values (key overrides)

```yaml
storageClass:
  create: false                            # we ship our own SCs explicitly
  provisionerName: rancher.io/local-path   # must match the SC manifests; chart auto-generates a name otherwise, breaking SC↔provisioner binding
nodePathMap:
  - node: DEFAULT_PATH_FOR_NON_LISTED_NODES
    paths:
      - /var/local-path-provisioner
```

### Namespace + PodSecurity

New namespace `storage`. Talos enforces PodSecurityAdmission at cluster level; the helper Pod's `hostPath` mount is forbidden under the `baseline` profile. The namespace is labelled `privileged` for all three PSA modes:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: storage
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
```

### Repo layout

Follows the existing 3-level Kustomization hierarchy (namespace → app → `app/`):

```
kubernetes/apps/storage/
├── kustomization.yaml
├── namespace.yaml
└── local-path-provisioner/
    ├── ks.yaml
    └── app/
        ├── kustomization.yaml
        ├── ocirepository.yaml
        ├── helmrelease.yaml
        ├── storageclass-default.yaml
        └── storageclass-retain.yaml
```

The new `storage` namespace is registered in `kubernetes/flux/cluster/ks.yaml` next to `cert-manager`, `kube-system`, `network`, etc.

## Capacity planning (forward-looking)

Phase 1–4 stateful workloads and rough size targets, to size `/var` headroom before continuing:

| Workload | PVC size (target) | StorageClass |
|---|---|---|
| Prometheus (30d retention) | 30 GiB | `local-path-retain` |
| Alertmanager | 1 GiB | `local-path` |
| Grafana | 2 GiB | `local-path` |
| Victoria-Logs (Phase 3, 7d) | ~20 GiB | `local-path-retain` |
| Gatus (Phase 4) | 1 GiB | `local-path` |
| **Total** | **~55 GiB** | |

Pre-Phase 1 gate: `df -h /var` on each node must show ≥60 GiB free before deploying kube-prometheus-stack.

## Validation plan

1. `kubectl get sc` → `local-path` marked `(default)`, `local-path-retain` present.
2. `kubectl -n storage get pods` → provisioner pod `Running`.
3. Smoke test — create a PVC + Pod that writes a file:
   - PV binds only after Pod scheduling (confirms `WaitForFirstConsumer`).
   - On the scheduled node: `/var/local-path-provisioner/pvc-<uuid>_<ns>_<pvc>` exists and contains the written file.
4. Delete PVC bound to `local-path` → directory removed.
5. Create a second PVC bound to `local-path-retain`, delete it → directory survives.
6. Reboot the worker (`talosctl reboot -n <worker-ip>`) → Pod reschedules on return, data intact.
7. `df -h /var` on both nodes captured and recorded; must be ≥60 GiB free on any node that will host Phase 1 workloads.

## Rollout

1. Merge PR creating `kubernetes/apps/storage/…` and the cluster-level registration.
2. Flux reconciles. Watch `Kustomization/local-path-provisioner` go `Ready`.
3. Run validation steps 1–6.
4. Tag spec as implemented in the spec header.
5. Proceed to Phase 1 spec (metrics core).

## Risks & mitigations

- **Ephemeral partition exhaustion.** Capacity table above sets the budget; size each PVC deliberately. If headroom is tight, skip Victoria-Logs in Phase 3 or reduce retention.
- **Helper Pod fails to schedule.** PSA labels on the namespace are the most common cause — verified in §Namespace. Chart-default `tolerations` may also need extending if nodes gain taints later; not an issue today.
- **Chart unavailability.** Fallback to vendored raw manifests documented above.
- **Silent data loss on node rebuild.** Reinstalling Talos wipes `/var`. Any PVC on `local-path` is gone. This is accepted for Phase 0; the backup story is a later spec.

## Open questions

None blocking. Chart-source fallback decision is made at rollout time if the primary OCI chart misbehaves.
