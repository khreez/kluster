# Home Network Dashboards (Pi-hole Raspberry Pi)

**Status:** Draft — awaiting review
**Date:** 2026-10-04
**Owner:** @khreez

## Context

The cluster runs `kube-prometheus-stack` (chart `91.9.0`) in `observability`, with Grafana's dashboard sidecar enabled (`grafana_dashboard: "1"`, `searchNamespace: ALL`) and Prometheus retaining 30d. Nothing outside the cluster is scraped today.

The home network has one off-cluster host worth watching: a **Raspberry Pi 3B** at `192.168.1.40` (`pi-hole.lan`, Debian 13 arm64) running **Pi-hole v6** natively. It is managed by Ansible in the separate `home-net` repo. That repo (session `home-net-91`) is adding two exporters to the Pi as systemd services; this spec covers only the kluster side: scraping them and dashboards.

## Goals

- **At a glance:** one custom dashboard that answers "is the Pi healthy and is Pi-hole blocking?" in a few seconds.
- **Trends:** detailed history (up to the existing 30d retention) for the Pi's host metrics and Pi-hole DNS stats.
- Fully GitOps: all scrape config and dashboard JSON lives in this repo; Grafana needs no internet access at start.
- Leave the `kube-prometheus-stack` HelmRelease untouched.

## Non-goals

- **Alerting.** No `PrometheusRule`s for these targets. Revisit in a later spec.
- **Retention beyond 30d.** Long-term storage is out of scope.
- **Installing or configuring exporters.** Owned by `home-net` (Ansible).
- **Other devices** (router, NAS, etc.). The layout allows adding them later; none are built now.
- **Blackbox / ping probes.** The exporters' `up` series is enough for status.
- **Grafana folders.** The sidecar's `folderAnnotation` is not configured; dashboards land in the default folder.

## Interface contract with `home-net`

Agreed with `home-net-91`:

| Exporter | Endpoint | Auth | Metric prefix |
|---|---|---|---|
| node_exporter (default collectors incl. thermal_zone/hwmon; systemd collector disabled) | `http://192.168.1.40:9100/metrics` | none | `node_` |
| eko/pihole-exporter v1.2.0 | `http://192.168.1.40:9617/metrics` | none | `pihole_` |

Until the Ansible playbook runs, both targets report `up == 0`.

## Verified current state

- `scrapeconfigs.monitoring.coreos.com` CRD present, version `v1alpha1`, and its spec supports `jobName`.
- Live `Prometheus` has `scrapeConfigSelector: {matchLabels: {release: kube-prometheus-stack}}` and `scrapeConfigNamespaceSelector: {}`, so ScrapeConfigs **must** carry `release: kube-prometheus-stack`.
- Grafana Prometheus datasource UID is `prometheus`.
- Repo convention for inter-app ordering: `dependsOn` in `ks.yaml` (see `kubernetes/apps/flux-system/flux-instance/ks.yaml`).

## Design

### Layout

```
kubernetes/apps/observability/
  kustomization.yaml                 # add ./home-network/ks.yaml
  home-network/
    ks.yaml
    app/
      kustomization.yaml
      scrapeconfig.yaml
      dashboards/
        home-network-overview.json
        node-exporter-full.json
        pihole-exporter.json
```

### Flux Kustomization (`home-network/ks.yaml`)

Same shape as `kube-prometheus-stack/ks.yaml` (interval 1h, `prune: true`, `sourceRef` flux-system, `targetNamespace: observability`, `wait: true`) plus `dependsOn: [{name: kube-prometheus-stack}]` so the operator and Grafana exist first. No `postBuild` substitution: nothing here is secret. This also means dashboard JSON containing `${...}` is not mangled by Flux.

### Scraping (`app/scrapeconfig.yaml`)

Two `monitoring.coreos.com/v1alpha1` `ScrapeConfig`s, both labelled `release: kube-prometheus-stack`:

| Name / `jobName` | Target | Interval |
|---|---|---|
| `pihole-node` | `192.168.1.40:9100` | 30s |
| `pihole-exporter` | `192.168.1.40:9617` | 30s |

Each uses `staticConfigs` with `labels: {instance: pi-hole.lan}`, plain HTTP, and the default `/metrics` path. If the static label does not override `instance` in practice, fall back to a `relabelings` rule that sets `instance`.

### Dashboards (`app/kustomization.yaml`)

A `configMapGenerator` creates one ConfigMap per JSON file, with `generatorOptions`:
- `labels: {grafana_dashboard: "1"}`
- `disableNameSuffixHash: true`

**Community dashboards.** Downloaded once from grafana.com at a pinned revision, recorded in a comment in `app/kustomization.yaml`:
- Node Exporter Full, #1860
- Pi-hole Exporter, #10176

On import:
- Remove the `__inputs` / `__requires` blocks.
- Replace `${DS_PROMETHEUS}` (or equivalent) with the datasource UID `prometheus`.
- Set a stable `uid` so links from the overview work.

Check that each one renders against the real metric names, especially #10176 vs pihole-exporter v1.2.0's v6 metrics.

**Custom `home-network-overview.json`.** Stable `uid: home-network-overview`, default range 24h, refresh 1m. It has three rows:

1. **Status** (stat panels)
   - Pi up: `up{job="pihole-node"}`
   - Exporter up: `up{job="pihole-exporter"}`
   - Power: `max(node_hwmon_in_lcrit_alarm_volts{job="pihole-node", chip="soc:firmware_raspberrypi_hwmon"})`, mapped 0 = green OK, 1 = red UNDERVOLTAGE. Added after node_exporter went live; the series was confirmed in the Pi's real `/metrics` output.
   - Pi-hole blocking enabled
   - Uptime
   - SoC temperature
2. **Pi health** (time series)
   - CPU %
   - Memory used %
   - Root filesystem used %
   - SoC temperature
3. **DNS** (stats + time series)
   - Queries today
   - Blocked today
   - % blocked
   - Unique clients
   - Queries vs blocked over time

Thresholds are tuned for the Pi 3B (1 GiB RAM, throttles around 80 °C):

| Metric | Amber | Red |
|---|---|---|
| Temperature | 70 °C | 80 °C |
| Memory | 80% | 90% |
| Disk | 80% | 90% |
| Up | — | 0 = **DOWN** |

Dashboard links point to the two community dashboards for detail.

**pihole metric names:** the exact `pihole_*` metric names must be taken from pihole-exporter v1.2.0 (source or live `/metrics`) before writing queries. They are not assumed here.

**SoC temperature query:** `node_thermal_zone_temp{instance="pi-hole.lan", type="cpu-thermal"}` (Celsius). `home-net-91` confirmed it read-only from sysfs on the Pi: `thermal_zone0` has `type=cpu-thermal`. These labels are stable and don't depend on hwmon chip naming. `node_hwmon_temp_celsius` reports the same reading but its `chip` label is unknown until deploy, so it is not used.

## Failure modes

- **Exporters not deployed yet:** status panels show red DOWN and the other panels show "No data". This is expected until `home-net` runs its playbook.
- **Pi-hole down but Pi up:** the exporter may still be up, but the `pihole_*` status / query metrics go stale or report disabled. The status panel reflects that.
- **Pi-hole API error while exporter is up:** pihole-exporter v1.2.0 keeps the last gauge values when the Pi-hole API call fails, so DNS tiles show stale numbers instead of blanks. Not detectable from its metrics; check the exporter's logs on the Pi (`journalctl -u <exporter unit>`) if numbers stop changing.
- **Default kps `TargetDown` alert:** the built-in rule (`count(up == 0) by job / count(up) by job > 10%`, `for: 10m`, severity warning) has no job filter, and the `discord` AlertmanagerConfig receives all alerts. If either Pi target is down for more than 10 minutes, Discord gets a warning (re-sent every 12h). To avoid noise, push this change only **after** `home-net` has deployed the exporters. Ongoing alerts for a real Pi outage are left as-is. This spec authors no alerts, and silencing a default rule is a separate decision.
- **Pi in cluster CPU/network recording rules:** default kps recording rules such as `cluster:node_cpu:ratio` and `instance:node_network_*` have no `job="node-exporter"` filter, so they include the Pi. Node* alert rules filter on `job="node-exporter"` and are unaffected.
- **Missing release label:** Prometheus silently ignores the ScrapeConfig. Caught by the post-deploy target check below.

## Testing

**Local, before any commit:**
- `kustomize build kubernetes/apps/observability/home-network/app` succeeds.
- The output validates with `kubeconform` (CRD schemas for `ScrapeConfig`).
- `jq empty` passes on all three dashboard JSON files.
- `kustomize build kubernetes/apps/observability` still succeeds.

**Post-deploy** (only after the user approves commit + push):
- `flux get ks -n flux-system home-network` shows Ready.
- In Prometheus, `up{instance="pi-hole.lan"}` returns two series, with jobs `pihole-node` and `pihole-exporter`. Values are 0 until the exporters exist, 1 after.
- All three dashboards appear in Grafana. After the exporters are live, every panel on the overview has data.

## Rollout

1. Implement and pass local tests.
2. Commit and push **only on explicit user instruction**.
3. Flux reconciles. Targets show down until `home-net` deploys the exporters.
4. After the exporters are live (`home-net-91` will send a post-deploy report): confirm the pihole metric names and the temperature series, and adjust queries if needed.
