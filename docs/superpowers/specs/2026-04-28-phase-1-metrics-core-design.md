# Phase 1 — Metrics Core (kube-prometheus-stack, bundled)

**Status:** Draft
**Date:** 2026-04-28
**Owner:** @khreez

## Context

Phase 0 delivered persistent storage; the cluster now has a default `local-path` StorageClass and a `local-path-retain` SC for data that must survive accidental PVC deletion. Several apps already in the cluster (Cilium, CoreDNS, envoy-gateway) ship `ServiceMonitor`/`PodMonitor` objects that nothing scrapes, because the kube-prometheus-stack CRDs are installed (via the bootstrap helmfile, chart `83.7.0`) but no Prometheus instance exists yet.

This spec defines Phase 1 of the observability roadmap: deploy the metrics core — Prometheus, Alertmanager, Grafana, kube-state-metrics, node-exporter, and the Prometheus Operator — as a single bundled `kube-prometheus-stack` HelmRelease, plus one Discord receiver so cluster-critical alerts actually reach the operator.

## Goals

- Deploy a working Prometheus that scrapes everything in the cluster that already has a SM/PM, with no per-app onboarding required.
- Deploy Alertmanager with a single Discord receiver wired to the user's webhook; default kps rules fire (minus Talos-incompatible groups).
- Deploy Grafana with Prometheus + Alertmanager auto-wired as datasources, kps default dashboards visible, and sidecar discovery for any future `grafana_dashboard=1` ConfigMap cluster-wide.
- Expose all three UIs LAN-internally via the existing `envoy-internal` gateway and the wildcard cert.
- Stay within the Phase 0 capacity budget (33 GiB PVCs total).

## Non-goals

- **Standalone Grafana.** The split-chart option (kps minus Grafana + separate `grafana/grafana` HelmRelease) was rejected during brainstorm — bundled keeps the spec to one HelmRelease.
- **External exposure of any UI.** Cloudflare Access / Cloudflare Tunnel for Grafana was considered (Q3-B) and deferred. All three UIs are LAN-only via `envoy-internal`. Revisit when the user wants dashboard access from outside the LAN.
- **Curated PrometheusRule library.** Only kps's built-in default rules ship in this phase, with the four Talos-incompatible groups disabled. Hand-authoring rules is a later spec.
- **etcd scrape.** Talos's etcd metrics endpoint requires machineconfig changes (`extraArgs.listen-metrics-urls`) and a client cert plumbed in as a Secret. Out of scope for Phase 1.
- **kube-controller-manager / kube-scheduler / kube-proxy scrape.** Talos doesn't expose these as Service-backed scrape targets; Cilium replaces kube-proxy entirely. Disabled at the chart level.
- **HA / replicated Prometheus.** Single replica, single PV, pinned to whatever node it first lands on. HA is blocked on Phase 0's HA-storage non-goal (≥3 nodes with dedicated disks).
- **Backups.** Pod-level backup story is a later spec.
- **Resource requests/limits hand-tuning.** Chart defaults are used; revisit if scrape budget grows.

## Current state

- Cluster: 2 Talos nodes (`ctrl01` controller, `work01` worker).
- Storage: `local-path` (default, `Delete`) and `local-path-retain` (`Retain`) — both bound to `/var/local-path-provisioner` on each node, 221 GiB free.
- Monitoring CRDs: installed by `bootstrap/helmfile.d/00-crds.yaml` at chart `83.7.0` (`alertmanagers`, `prometheuses`, `prometheusrules`, `servicemonitors`, `podmonitors`, `probes`, `scrapeconfigs`, `alertmanagerconfigs`, `prometheusagents`, `thanosrulers`).
- Monitoring HelmReleases: none. `observability` namespace does not exist.
- Existing scrape targets in repo waiting to be picked up: `kubernetes/apps/network/envoy-gateway/app/podmonitor.yaml`, default Cilium SMs, default CoreDNS SM, default flux-operator SMs.
- Gateways: `envoy-internal` Gateway in `network` namespace, LB IP `192.168.1.86`, advertised as `internal.${SECRET_DOMAIN}` via external-dns. HTTPS listener accepts cross-namespace HTTPRoutes (`allowedRoutes.namespaces.from: All`); TLS via wildcard `${SECRET_DOMAIN/./-}-production-tls`. Global HTTP→HTTPS redirect already wired.

## Design

### Architecture

One namespace `observability`. One Flux `Kustomization` reconciling one `HelmRelease` (`kube-prometheus-stack` chart `83.7.0`) plus four sibling resources in the same `app/` directory:

1. `OCIRepository` pointing at the prometheus-community OCI registry.
2. `Secret` (SOPS-encrypted) holding the Discord webhook URL.
3. Three `HTTPRoute` objects, one per UI, attaching to `envoy-internal`.

CRDs are *not* shipped from this HelmRelease (`crds.enabled: false`); they continue to be managed by the bootstrap helmfile at the same chart version. See "CRD coordination" risk below.

### Brainstorm decisions (locked)

| # | Question | Choice |
|---|---|---|
| Q1 | Phase 1 scope | Data plane + one Alertmanager receiver |
| Q2 | Architecture shape | Bundled (kps with `grafana.enabled: true`); single HelmRelease |
| Q3 | Ingress/auth | All three UIs internal-only via `envoy-internal`; Grafana built-in admin login; no auth on Prom/AM |
| Q4 | Receiver | Discord webhook |
| Q5 | Dashboards | kps defaults + sidecar discovery (`grafana_dashboard=1`, all namespaces) |
| Q6 | SM/PM/Rule discovery | SM/PM cluster-wide; PrometheusRule label-gated on `release: kube-prometheus-stack` |
| Q7 | PrometheusRule defaults | All kps defaults, with Talos-incompatible groups pre-disabled |

### File layout

```
kubernetes/apps/observability/
├── kustomization.yaml                     # ../../components/sops + namespace + ks
├── namespace.yaml                         # PSA: privileged
└── kube-prometheus-stack/
    ├── ks.yaml                            # Flux Kustomization, healthcheck on the HR
    └── app/
        ├── kustomization.yaml             # lists every yaml below
        ├── ocirepository.yaml             # oci://ghcr.io/prometheus-community/charts/kube-prometheus-stack
        ├── helmrelease.yaml               # version 83.7.0, all values inline, crds.enabled: false
        ├── secret.sops.yaml               # name: discord-webhook, key: url
        ├── httproute-grafana.yaml         # grafana.${SECRET_DOMAIN} → svc/kube-prometheus-stack-grafana:80
        ├── httproute-prometheus.yaml      # prometheus.${SECRET_DOMAIN} → svc/kube-prometheus-stack-prometheus:9090
        └── httproute-alertmanager.yaml    # alertmanager.${SECRET_DOMAIN} → svc/kube-prometheus-stack-alertmanager:9093
```

The new `observability` namespace requires no edit to `kubernetes/flux/cluster/ks.yaml` — the cluster-root `cluster-apps` Kustomization reconciles `./kubernetes/apps` recursively and discovers new namespace directories automatically.

### Component inventory

| Component | Replicas | PVC | Storage class | Notes |
|---|---|---|---|---|
| prometheus-operator | 1 (Deployment) | none | — | Watches all namespaces |
| prometheus | 1 (StatefulSet) | 30 GiB | `local-path-retain` | 30d retention, 25 GiB retentionSize floor |
| alertmanager | 1 (StatefulSet) | 1 GiB | `local-path` | Single Discord receiver |
| grafana | 1 (Deployment) | 2 GiB | `local-path` | Prom + AM datasources auto-wired by chart; sidecar discovery enabled |
| kube-state-metrics | 1 (Deployment) | none | — | |
| node-exporter | 2 (DaemonSet) | none | — | hostNetwork + hostPID; needs `privileged` PSA |

**Disabled at chart level:** `kubeControllerManager`, `kubeScheduler`, `kubeProxy`, `kubeEtcd` (subcharts and their PrometheusRule groups). Cilium handles the kube-proxy role; Talos doesn't expose the others as scrape targets.

### Discovery model

- `prometheus.prometheusSpec.serviceMonitorSelector: {}` — picks up every SM in the cluster.
- `prometheus.prometheusSpec.podMonitorSelector: {}` — picks up every PM in the cluster.
- `prometheus.prometheusSpec.probeSelector: {}` — picks up every Probe in the cluster.
- `prometheus.prometheusSpec.serviceMonitorNamespaceSelector: {}` and the matching pod/probe namespace selectors — empty (all namespaces).
- `prometheus.prometheusSpec.ruleSelector: { matchLabels: { release: kube-prometheus-stack } }` — only PrometheusRules carrying that label fire.
- `prometheus.prometheusSpec.ruleNamespaceSelector: {}` — but the label gate applies cluster-wide.

The label `release: kube-prometheus-stack` is what kps's own `defaultRules.labels` already applies, so the bundled rules are discovered automatically. Future custom rules in any namespace must carry the same label or they stay silent — this is the deliberate Q6-C on-ramp.

### HelmRelease values (key overrides)

```yaml
crds:
  enabled: false                     # CRDs come from bootstrap helmfile
defaultRules:
  create: true
  disabled:
    kubeControllerManager: true
    kubeScheduler: true
    kubeProxy: true
    etcd: true
kubeControllerManager: { enabled: false }
kubeScheduler:         { enabled: false }
kubeProxy:             { enabled: false }
kubeEtcd:              { enabled: false }
prometheus:
  prometheusSpec:
    retention: 30d
    retentionSize: 25GiB
    serviceMonitorSelector: {}
    podMonitorSelector: {}
    probeSelector: {}
    serviceMonitorNamespaceSelector: {}
    podMonitorNamespaceSelector: {}
    probeNamespaceSelector: {}
    ruleSelector:
      matchLabels:
        release: kube-prometheus-stack
    ruleNamespaceSelector: {}
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path-retain
          resources: { requests: { storage: 30Gi } }
alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path
          resources: { requests: { storage: 1Gi } }
    secrets: [discord-webhook]      # mounts at /etc/alertmanager/secrets/discord-webhook/
  config:
    route:
      group_by: [alertname, severity]
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: discord
    inhibit_rules:
      - source_matchers: [severity = critical]
        target_matchers: [severity = warning]
        equal: [alertname, namespace]
      - source_matchers: [alertname = InfoInhibitor]
        target_matchers: [severity = info]
        equal: [namespace]
    receivers:
      - name: discord
        discord_configs:
          - webhook_url_file: /etc/alertmanager/secrets/discord-webhook/url
            send_resolved: true
grafana:
  enabled: true
  persistence:
    enabled: true
    storageClassName: local-path
    size: 2Gi
  sidecar:
    dashboards:
      enabled: true
      searchNamespace: ALL
      label: grafana_dashboard
      labelValue: "1"
    datasources:
      enabled: true
      searchNamespace: ALL
```

### Discord webhook secret

```yaml
# kubernetes/apps/observability/kube-prometheus-stack/app/secret.sops.yaml
apiVersion: v1
kind: Secret
metadata:
  name: discord-webhook
  namespace: observability
type: Opaque
stringData:
  url: https://discord.com/api/webhooks/...   # SOPS-encrypted at rest
```

Encrypted under the existing `(bootstrap|kubernetes)/.*\.sops\.ya?ml` rule; only `data`/`stringData` is encrypted, so the rest of the YAML stays readable. Secret is mounted via `alertmanager.alertmanagerSpec.secrets: [discord-webhook]`, which lands the file at `/etc/alertmanager/secrets/discord-webhook/url` inside the Alertmanager pod.

### HTTPRoutes (one per UI)

All three follow the identical shape:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: <ui>
  namespace: observability
spec:
  hostnames: ["<ui>.${SECRET_DOMAIN}"]
  parentRefs:
    - name: envoy-internal
      namespace: network
      sectionName: https
  rules:
    - backendRefs:
        - name: kube-prometheus-stack-<svc>
          port: <port>
```

Hostnames: `grafana.${SECRET_DOMAIN}` → svc `kube-prometheus-stack-grafana:80`; `prometheus.${SECRET_DOMAIN}` → svc `kube-prometheus-stack-prometheus:9090`; `alertmanager.${SECRET_DOMAIN}` → svc `kube-prometheus-stack-alertmanager:9093`. TLS terminates at envoy via the wildcard `${SECRET_DOMAIN/./-}-production-tls`. external-dns ignores HTTPRoutes; k8s_gateway resolves `*.${SECRET_DOMAIN}` LAN-side to `192.168.1.86`. The global `https-redirect` HTTPRoute in `network` already covers HTTP→HTTPS for all hostnames.

### Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: observability
  annotations:
    kustomize.toolkit.fluxcd.io/prune: disabled
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
```

`privileged` is required for node-exporter (hostNetwork + hostPID + hostPath mounts of `/`, `/proc`, `/sys`).

## Capacity planning

| Item | Source | Value |
|---|---|---|
| Prometheus PVC | this spec | 30 GiB on `local-path-retain` |
| Alertmanager PVC | this spec | 1 GiB on `local-path` |
| Grafana PVC | this spec | 2 GiB on `local-path` |
| **PV total** | | **33 GiB** |
| Phase 0 budget | Phase 0 spec | 33 GiB earmarked |
| Free `/var` per node | Phase 0 status | 221 GiB |
| Memory steady-state (rough) | estimate | <1.5 GiB total (~600 MiB Prom · ~150 MiB Grafana · ~50 MiB AM · ~80 MiB KSM · ~30 MiB × 2 node-exporter · ~80 MiB Operator) |
| CPU steady-state | estimate | <500m total; bursts during compaction |

No resource requests/limits set. Chart defaults are appropriate for a small cluster. Revisit if Prometheus active-target count exceeds ~50.

## Validation plan

1. **Flux green.** `flux -n observability get hr kube-prometheus-stack` → `Ready=True`. `flux -n flux-system get ks kube-prometheus-stack` → `Ready=True`.
2. **Pods up.** `kubectl -n observability get pods -o wide` → `prometheus-kube-prometheus-stack-prometheus-0`, `alertmanager-kube-prometheus-stack-alertmanager-0`, `kube-prometheus-stack-operator-*`, `kube-prometheus-stack-grafana-*`, `kube-prometheus-stack-kube-state-metrics-*` all `Running`. `node-exporter` DaemonSet has 2 pods (one per node).
3. **PVCs bound.** `kubectl -n observability get pvc` → 3 bound. Storage classes match plan: prometheus → `local-path-retain` (30 GiB), alertmanager → `local-path` (1 GiB), grafana → `local-path` (2 GiB).
4. **Targets healthy.** Browse `https://prometheus.${SECRET_DOMAIN}/targets`. Required UP: `kube-prometheus-stack-{prometheus,alertmanager,operator,grafana,kube-state-metrics}` SMs, `kube-prometheus-stack-kubelet` SM (kubelet, cAdvisor, probes), `kube-prometheus-stack-prometheus-node-exporter` PM. Pre-existing SMs/PMs from the rest of the repo (envoy-gateway, cilium, coredns) should appear too — note any DOWN ones as a follow-up, not a Phase 1 blocker.
5. **Watchdog firing.** `https://prometheus.${SECRET_DOMAIN}/alerts` shows `Watchdog` in `firing` state. This is the heartbeat — must always be firing.
6. **Discord delivery.** Within ~30s of step 5, a Watchdog message lands in the configured Discord channel. If not, check `kubectl -n observability logs -l app.kubernetes.io/name=alertmanager` for `notify error` lines pointing at the webhook.
7. **Grafana login + dashboards.** Browse `https://grafana.${SECRET_DOMAIN}`. Read admin password: `kubectl -n observability get secret kube-prometheus-stack-grafana -o jsonpath='{.data.admin-password}' | base64 -d`. Open "Kubernetes / Compute Resources / Cluster" — graphs populate. Open "Node Exporter / Nodes" — both nodes appear with non-zero metrics.
8. **Sidecar smoke test.** Apply a labelled ConfigMap in `default`:
   ```sh
   kubectl create configmap test-dashboard -n default \
     --from-literal=test.json='{"title":"sidecar-smoke-test","panels":[]}' \
     --dry-run=client -o yaml \
     | kubectl label --local -f - grafana_dashboard=1 -o yaml \
     | kubectl apply -f -
   ```
   Within ~60s the dashboard `sidecar-smoke-test` appears in Grafana. Delete the ConfigMap; dashboard disappears.
9. **Restart resilience.** `kubectl -n observability delete pod prometheus-kube-prometheus-stack-prometheus-0`. New pod schedules on the same node (PV pinning), waits for PV remount. After Ready, query `up{job=~"kube-prometheus-stack-prometheus"}` in Grafana — series gap < 1 min, history before deletion intact.
10. **Disabled-component check.** Cluster-wide grep: `kubectl get servicemonitor -A` shows no SM for kube-controller-manager, kube-scheduler, kube-proxy, etcd. `kubectl -n observability get prometheusrule -o yaml | grep -E 'KubeControllerManagerDown|KubeSchedulerDown|KubeProxyDown|etcd'` returns nothing.

**Done condition:** all 10 steps green. Watchdog firing in Discord for ≥1h with no other unexpected pages, OR follow-up issue filed listing the noisy rules to disable. Spec header updated to `Status: Implemented YYYY-MM-DD`. Phase 2 spec begins.

## Rollout

1. Merge PR creating `kubernetes/apps/observability/…` plus the renovate group rule (see Risks). No edit to `kubernetes/flux/cluster/ks.yaml` needed — the namespace is auto-discovered.
2. Flux reconciles. Watch `Kustomization/kube-prometheus-stack` go `Ready`, then the HR.
3. Run validation steps 1–10.
4. Tag spec as implemented.
5. Proceed to Phase 2 spec.

## Risks & mitigations

- **Day-1 alert noise.** Despite Q7-B group disables, individual rules outside those groups (e.g. `KubeAPIErrorBudgetBurn`, `CPUThrottlingHigh`, `KubeMemoryOvercommit`) may misfire on a 2-node cluster with thresholds tuned for 100+ nodes. **Mitigation:** triage Discord traffic for the first 24h; disable noisy rules via `defaultRules.disabled.<group>: true` or `prometheus.prometheusSpec.additionalPrometheusRulesMap` patches in a follow-up PR. Document each disable with the misfire reason. Noise > ~3 pages/day in 24h ⇒ block the Done condition.

- **Single-node PV pinning (no HA).** Prometheus PV binds to the first node it lands on. If that node dies or reboots, Prometheus is unavailable until the node returns. **Mitigation:** accepted (single-worker cluster — HA storage is blocked on Phase 0's HA-storage non-goal). On reboot, Watchdog stops in Discord → user notices → wait for node back. Documented above.

- **CRD vs. HR version drift.** Bootstrap helmfile installs CRDs at chart `83.7.0`; HR pins `83.7.0`. Renovate will eventually bump them out of lockstep (different files, different PRs). The Operator may emit CRs that older CRDs don't fully understand, or vice versa. **Mitigation:** add a Renovate `packageRules` entry that groups both kps occurrences (`bootstrap/helmfile.d/00-crds.yaml` chart + the new HR's chart) into a single PR. One-line preventive fix; far better than a manual gate.

- **Webhook URL leak via Git.** Discord webhook is the only secret. Stored SOPS-encrypted at rest; decrypted by Flux at reconcile time. Standard repo pattern (cloudflare-tunnel uses identical mechanism). **Mitigation:** verify `git show HEAD:kubernetes/apps/observability/kube-prometheus-stack/app/secret.sops.yaml | head` before pushing the first commit — `stringData.url` must read as ENC[...].

- **Grafana admin password rotation.** kps generates a random admin password at first install (stored in `Secret/kube-prometheus-stack-grafana`). Subsequent helm upgrades do *not* rotate it. **Mitigation:** read it once after first deploy, log in, change via Grafana UI; or set a stable `grafana.adminPassword` value sourced from a SOPS Secret. Treat as a follow-up; not a Phase 1 blocker.

- **PrometheusRule label-gate misconfiguration.** If `prometheus.prometheusSpec.ruleSelector` and `defaultRules.labels` disagree on the label, all rules silently disappear and no alerts ever fire. **Mitigation:** validation step 5 catches this — Watchdog is a default rule, so no Watchdog firing = label mismatch.

- **node-exporter on Talos.** Talos restricts host filesystem access; node-exporter needs hostNetwork + hostPID + hostPath mounts of `/`, `/proc`, `/sys`. The `observability` namespace's `privileged` PSA labels permit it; no Talos machineconfig change needed. **Mitigation:** validation step 2 — node-exporter pods stuck in `CreateContainerConfigError` or `CrashLoopBackOff` here usually means the namespace isn't actually `privileged`.

- **Unauthenticated Prometheus / Alertmanager UIs on the LAN.** Per Q3-A, neither has any auth — anyone on the LAN with the URL can browse Prometheus, run PromQL, or silence alerts. **Mitigation:** none for Phase 1 (LAN trust is the explicit choice). Revisit if/when LAN trust assumption no longer holds (untrusted devices, guest WiFi merge, etc.).

## Open questions

None blocking. The renovate group rule for CRD/HR lockstep is the only choice the spec leaves to the implementer; recommended above, manual gate is the documented fallback.
