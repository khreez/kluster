# Home Network Dashboards Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Scrape the Pi-hole Raspberry Pi's two exporters into the cluster's Prometheus and show them in Grafana via one custom at-a-glance dashboard and two pinned community dashboards.

**Architecture:** A new Flux app `kubernetes/apps/observability/home-network/` (its own `ks.yaml`, `dependsOn` kube-prometheus-stack) holding two `ScrapeConfig` CRs and a kustomize `configMapGenerator` that turns three dashboard JSON files into ConfigMaps labelled `grafana_dashboard: "1"`. Grafana's existing sidecar loads them. The kube-prometheus-stack HelmRelease is not touched.

**Tech Stack:** Flux v2 Kustomization, kustomize 5.7, Prometheus Operator `monitoring.coreos.com/v1alpha1` ScrapeConfig, Grafana 13.2 (kube-prometheus-stack 91.9.0), jq, kubeconform.

**Spec:** `docs/superpowers/specs/2026-10-04-home-network-dashboards-design.md`

## Global Constraints

- **Commits and pushes happen ONLY on explicit user instruction in the current turn.** Approval of content is not authorization to commit. Never commit as part of a task otherwise.
- Pi host: `192.168.1.40`, instance label value `pi-hole.lan`.
- Targets: `192.168.1.40:9100` (node_exporter, `jobName: pihole-node`) and `192.168.1.40:9617` (eko/pihole-exporter v1.2.0, `jobName: pihole-exporter`). Plain HTTP, no auth, path `/metrics`, `scrapeInterval: 30s`.
- Every ScrapeConfig carries label `release: kube-prometheus-stack` (the live Prometheus `scrapeConfigSelector` requires it).
- Namespace: `observability`. Grafana Prometheus datasource UID: `prometheus`.
- Dashboard ConfigMaps: label `grafana_dashboard: "1"`, `disableNameSuffixHash: true`.
- Community dashboards pinned: Node Exporter Full **#1860 revision 45**, Pi-hole Exporter **#10176 revision 3**.
- No `postBuild` substitution in `home-network/ks.yaml` (dashboard JSON contains `${...}` and `$var` strings that Flux would otherwise substitute).
- SoC temperature query: `node_thermal_zone_temp{instance="pi-hole.lan", type="cpu-thermal"}`.
- Pi 3B thresholds: temperature amber 70 / red 80 °C; memory 80 / 90 %; disk 80 / 90 %; up 0 = red DOWN.
- No alert rules. No changes to `kube-prometheus-stack/app/helmrelease.yaml`.

## Review Focus

1. **Exporters not yet deployed:** the overview's status tiles must show red **DOWN** (Prometheus emits `up == 0` for failed scrapes), not "No data". Pinned by the value-mapping assertions in Task 3 Step 1 and the `up` check in Task 4 Step 6.
2. **Grafana 13 has no Angular:** #10176 (built on Grafana 6.2) uses `graph` and `grafana-piechart-panel`. Expected: panels render, auto-migrated to `timeseries`/`piechart`, with no "panel plugin not found". Task 2 Step 3 converts `grafana-piechart-panel` → `piechart` explicitly. Task 2 Step 1 asserts no Angular-only panel types remain, and Task 4 Step 7 visually confirms.
3. **Flux variable substitution mangling dashboards:** `${DS_PROMETHEUS}` / `$job` must reach Grafana intact. Pinned by Task 1 Step 1 (assert no `postBuild` in `ks.yaml`) and Task 4 Step 6 (in-cluster ConfigMap content equals the repo file).
4. **ConfigMap size limit (1 MiB):** #1860 is ~470 KB. Pinned by the byte-size assertion in Task 2 Step 1 and the server-side dry-run in Task 2 Step 5.
5. **Pi-hole API failure while the exporter is up:** pihole-exporter keeps the last gauge values on API error, so the DNS tiles show stale numbers rather than blanks. A reasonable person would expect *some* signal. The overview cannot detect this from v1.2.0 metrics; it is a documented limitation. Task 4 Step 8 records it in the spec's Failure modes section so it is not mistaken for a bug.

---

### Task 1: Flux Kustomization and ScrapeConfigs

**Files:**
- Create: `kubernetes/apps/observability/home-network/ks.yaml`
- Create: `kubernetes/apps/observability/home-network/app/kustomization.yaml`
- Create: `kubernetes/apps/observability/home-network/app/scrapeconfig.yaml`
- Modify: `kubernetes/apps/observability/kustomization.yaml` (add `./home-network/ks.yaml` under `resources`)

**Interfaces:**
- Consumes: nothing from earlier tasks. Needs `KUBECONFIG=./kubeconfig` for the dry-run.
- Produces: `app/kustomization.yaml` with a `resources:` list. Tasks 2 and 3 append a `configMapGenerator` to this same file. Flux Kustomization name `home-network`. ScrapeConfig job names `pihole-node` and `pihole-exporter`, which the dashboards in Task 3 query.

- [ ] **Step 1: Write the failing checks**

Run from the repo root:
```bash
test -f kubernetes/apps/observability/home-network/ks.yaml && ! grep -q postBuild kubernetes/apps/observability/home-network/ks.yaml && echo OK-ks
kustomize build kubernetes/apps/observability/home-network/app | yq -e 'select(.kind=="ScrapeConfig") | .metadata.labels.release == "kube-prometheus-stack"'
```
Expected: FAIL. The first prints nothing (no file). The second errors with `must build at directory: not a valid directory`.

- [ ] **Step 2: Create `kubernetes/apps/observability/home-network/ks.yaml`**

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: home-network
spec:
  dependsOn:
    - name: kube-prometheus-stack
  interval: 1h
  path: ./kubernetes/apps/observability/home-network/app
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
  targetNamespace: observability
  wait: true
  timeout: 5m
```

`dependsOn` uses the name only. Both Flux Kustomizations are created in the same namespace by `kubernetes/apps/observability/kustomization.yaml` (`namespace: observability`). This matches `kubernetes/apps/flux-system/flux-instance/ks.yaml`. Deliberately there is **no** `postBuild` (see Global Constraints).

- [ ] **Step 3: Create `kubernetes/apps/observability/home-network/app/scrapeconfig.yaml`**

```yaml
---
apiVersion: monitoring.coreos.com/v1alpha1
kind: ScrapeConfig
metadata:
  name: pihole-node
  labels:
    release: kube-prometheus-stack
spec:
  jobName: pihole-node
  scrapeInterval: 30s
  metricsPath: /metrics
  scheme: HTTP
  staticConfigs:
    - targets:
        - 192.168.1.40:9100
      labels:
        instance: pi-hole.lan
---
apiVersion: monitoring.coreos.com/v1alpha1
kind: ScrapeConfig
metadata:
  name: pihole-exporter
  labels:
    release: kube-prometheus-stack
spec:
  jobName: pihole-exporter
  scrapeInterval: 30s
  metricsPath: /metrics
  scheme: HTTP
  staticConfigs:
    - targets:
        - 192.168.1.40:9617
      labels:
        instance: pi-hole.lan
```

- [ ] **Step 4: Create `kubernetes/apps/observability/home-network/app/kustomization.yaml`**

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ./scrapeconfig.yaml
```

- [ ] **Step 5: Register the app in `kubernetes/apps/observability/kustomization.yaml`**

Change the `resources:` block to:
```yaml
resources:
  - ./namespace.yaml
  - ./kube-prometheus-stack/ks.yaml
  - ./home-network/ks.yaml
```

- [ ] **Step 6: Run the checks and validators**

```bash
test -f kubernetes/apps/observability/home-network/ks.yaml && ! grep -q postBuild kubernetes/apps/observability/home-network/ks.yaml && echo OK-ks
kustomize build kubernetes/apps/observability/home-network/app | yq -e 'select(.kind=="ScrapeConfig") | .metadata.labels.release == "kube-prometheus-stack"'
KC='kubeconform -strict -summary -schema-location default -schema-location https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json'
kustomize build kubernetes/apps/observability/home-network/app | $KC
kustomize build kubernetes/apps/observability | $KC -skip Secret
kustomize build kubernetes/apps/observability/home-network/app | KUBECONFIG=./kubeconfig kubectl apply --dry-run=server -n observability -f -
```
Expected:
- `OK-ks`
- `true` printed twice
- kubeconform `Valid: 2, Invalid: 0` for the app, and `Invalid: 0` for the namespace directory (the Secret is skipped because SOPS adds a `sops` field the strict schema rejects)
- `scrapeconfig.monitoring.coreos.com/pihole-node created (server dry run)`, and the same for `pihole-exporter`

The server dry-run validates against the live CRD and persists nothing.

- [ ] **Step 7: Do NOT commit.** Leave changes in the working tree; the commit is gated in Task 4.

---

### Task 2: Community dashboards (pinned, imported for Grafana 13)

**Files:**
- Create: `kubernetes/apps/observability/home-network/app/dashboards/node-exporter-full.json`
- Create: `kubernetes/apps/observability/home-network/app/dashboards/pihole-exporter.json`
- Modify: `kubernetes/apps/observability/home-network/app/kustomization.yaml` (add `generatorOptions` and `configMapGenerator`)

**Interfaces:**
- Consumes: `app/kustomization.yaml` from Task 1.
- Produces: dashboards with UIDs `rYdddlPWk` (Node Exporter Full) and `Pi-hole-Exporter` (Pi-hole Exporter). Task 3's overview links to these exact UIDs. ConfigMap names `dashboard-node-exporter-full` and `dashboard-pihole-exporter`.

Facts established during planning (do not re-derive):
- **#1860 rev 45:**
  - No `__inputs`.
  - Uses its own `ds_prometheus` datasource variable (`query: prometheus`), so it binds to the default Prometheus datasource unmodified.
  - The `job` variable is `label_values(node_uname_info, job)`, so `pihole-node` appears alongside the cluster's `node-exporter`.
  - UID `rYdddlPWk`, ~468 KB.
- **#10176 rev 3:**
  - Built on Grafana 6.2.
  - All 18 datasource references are the string `"${DS_PROMETHEUS}"`.
  - Uses `grafana-piechart-panel` (Angular; Grafana 13 has no Angular).
  - Every `pihole_*` metric it queries exists in pihole-exporter v1.2.0.
  - UID `Pi-hole-Exporter`.

- [ ] **Step 1: Write the failing checks**

```bash
D=kubernetes/apps/observability/home-network/app/dashboards
jq -e '.uid=="rYdddlPWk"' $D/node-exporter-full.json
jq -e '.uid=="Pi-hole-Exporter" and (has("__inputs")|not) and (has("__requires")|not)' $D/pihole-exporter.json
! grep -q 'DS_PROMETHEUS' $D/pihole-exporter.json && echo OK-no-ds-input
jq -e '[.. | objects | select(has("type")) | .type] | (index("grafana-piechart-panel") == null)' $D/pihole-exporter.json
test "$(wc -c < $D/node-exporter-full.json)" -lt 900000 && echo OK-size
kustomize build kubernetes/apps/observability/home-network/app | yq -e 'select(.kind=="ConfigMap") | .metadata.labels.grafana_dashboard == "1"'
```
Expected: FAIL (files don't exist; the last command prints nothing / exits non-zero).

- [ ] **Step 2: Download the pinned revisions**

```bash
D=kubernetes/apps/observability/home-network/app/dashboards
mkdir -p $D
curl -sSfL https://grafana.com/api/dashboards/1860/revisions/45/download | jq '.' > $D/node-exporter-full.json
curl -sSfL https://grafana.com/api/dashboards/10176/revisions/3/download -o "${TMPDIR:-/tmp}/pihole-10176-raw.json"
```

- [ ] **Step 3: Import #10176 for this Grafana**

The transform does three things:
- Drop `__inputs` / `__requires`.
- Bind datasource refs to UID `prometheus`.
- Convert the Angular pie chart type to the core `piechart` panel.

```bash
D=kubernetes/apps/observability/home-network/app/dashboards
jq 'del(.__inputs, .__requires, .id)
    | walk(if type == "object" and .datasource == "${DS_PROMETHEUS}" then .datasource = {"type":"prometheus","uid":"prometheus"} else . end)
    | walk(if type == "object" and .type == "grafana-piechart-panel" then .type = "piechart" else . end)' \
  "${TMPDIR:-/tmp}/pihole-10176-raw.json" > $D/pihole-exporter.json
rm "${TMPDIR:-/tmp}/pihole-10176-raw.json"
```

Leave `graph` panels as-is. Grafana 13 auto-migrates `graph` → `timeseries` on load; Task 4 Step 7 confirms visually.

- [ ] **Step 4: Add the generator to `app/kustomization.yaml`**

Replace the file with:
```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ./scrapeconfig.yaml
generatorOptions:
  disableNameSuffixHash: true
  labels:
    grafana_dashboard: "1"
configMapGenerator:
  # grafana.com #1860 "Node Exporter Full", revision 45 (unmodified)
  - name: dashboard-node-exporter-full
    files:
      - node-exporter-full.json=./dashboards/node-exporter-full.json
  # grafana.com #10176 "Pi-hole Exporter", revision 3
  # imported: dropped __inputs/__requires, datasource -> uid prometheus, grafana-piechart-panel -> piechart
  - name: dashboard-pihole-exporter
    files:
      - pihole-exporter.json=./dashboards/pihole-exporter.json
```

- [ ] **Step 5: Run the checks and validators**

Re-run every command from Step 1, then:
```bash
KC='kubeconform -strict -summary -schema-location default -schema-location https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json'
kustomize build kubernetes/apps/observability/home-network/app | $KC
kustomize build kubernetes/apps/observability/home-network/app | KUBECONFIG=./kubeconfig kubectl apply --dry-run=server --server-side --field-manager=dry-run-check -n observability -f -
```
Expected:
- Step 1's checks: `true`, `true`, `OK-no-ds-input`, `true`, `OK-size`, and then `true` twice (two ConfigMaps)
- kubeconform `Valid: 4, Invalid: 0`
- Dry-run lists 2 ScrapeConfigs and 2 ConfigMaps as `serverside-applied (server dry run)`

Server-side apply avoids the 256 KB `last-applied-configuration` annotation limit that client-side apply would hit with #1860. Flux uses server-side apply too.

- [ ] **Step 6: Do NOT commit.**

---

### Task 3: Custom "Home Network Overview" dashboard

**Files:**
- Create: `kubernetes/apps/observability/home-network/app/dashboards/home-network-overview.json`
- Modify: `kubernetes/apps/observability/home-network/app/kustomization.yaml` (one more `configMapGenerator` entry)

**Interfaces:**
- Consumes:
  - Job names `pihole-node` and `pihole-exporter` (Task 1)
  - Dashboard UIDs `rYdddlPWk` and `Pi-hole-Exporter` (Task 2)
  - pihole-exporter v1.2.0 metrics (namespace `pihole`, label `hostname`): `pihole_status` (1 = blocking enabled), `pihole_dns_queries_today`, `pihole_ads_blocked_today`, `pihole_ads_percentage_today`, `pihole_unique_clients`
- Produces: dashboard UID `home-network-overview`, ConfigMap `dashboard-home-network-overview`.

- [ ] **Step 1: Write the failing checks**

```bash
F=kubernetes/apps/observability/home-network/app/dashboards/home-network-overview.json
# identity + defaults
jq -e '.uid=="home-network-overview" and .refresh=="1m" and .time.from=="now-24h"' $F
# 3 rows, 14 data panels
jq -e '([.panels[]|select(.type=="row")]|length)==3 and ([.panels[]|select(.type!="row")]|length)==14' $F
# every query uses only known metrics
jq -e '[.panels[].targets[]?.expr | scan("(?:node|pihole)_[A-Za-z_]+|\\bup\\b")] | unique - ["up","node_boot_time_seconds","node_cpu_seconds_total","node_filesystem_avail_bytes","node_filesystem_size_bytes","node_memory_MemAvailable_bytes","node_memory_MemTotal_bytes","node_thermal_zone_temp","pihole_ads_blocked_today","pihole_ads_percentage_today","pihole_dns_queries_today","pihole_status","pihole_unique_clients"] | length == 0' $F
# every panel uses datasource uid prometheus
jq -e '[.panels[]|select(.type!="row")|.datasource.uid]|unique==["prometheus"]' $F
# Review Focus 1: up tiles map 0 -> DOWN (red)
jq -e '[.panels[]|select(.title=="Pi" or .title=="Exporter")|.fieldConfig.defaults.mappings[0].options["0"]|.text=="DOWN" and .color=="red"]|all and length==2' $F
# temperature thresholds 70/80 on the exact query
jq -e '[.panels[]|select(.targets[]?.expr=="node_thermal_zone_temp{instance=\"pi-hole.lan\", type=\"cpu-thermal\"}")|[.fieldConfig.defaults.thresholds.steps[].value]==[null,70,80]]|all and length==2' $F
# memory/disk thresholds 80/90
jq -e '[.panels[]|select(.title=="Memory used" or .title=="Root filesystem used")|[.fieldConfig.defaults.thresholds.steps[].value]==[null,80,90]]|all and length==2' $F
# links to community dashboards
jq -e '[.links[].url]|(any(test("/d/rYdddlPWk")) and any(test("/d/Pi-hole-Exporter")))' $F
```
Expected: FAIL (file does not exist).

- [ ] **Step 2: Create `home-network-overview.json`**

```json
{
  "uid": "home-network-overview",
  "title": "Home Network Overview",
  "tags": ["home-network"],
  "timezone": "browser",
  "editable": true,
  "schemaVersion": 39,
  "time": { "from": "now-24h", "to": "now" },
  "refresh": "1m",
  "links": [
    { "type": "link", "title": "Pi host detail (Node Exporter Full)", "url": "/d/rYdddlPWk?var-job=pihole-node", "icon": "dashboard" },
    { "type": "link", "title": "Pi-hole detail", "url": "/d/Pi-hole-Exporter", "icon": "dashboard" }
  ],
  "templating": { "list": [] },
  "annotations": { "list": [] },
  "panels": [
    { "type": "row", "title": "Status", "collapsed": false, "gridPos": { "x": 0, "y": 0, "w": 24, "h": 1 }, "panels": [] },
    {
      "type": "stat", "title": "Pi", "gridPos": { "x": 0, "y": 1, "w": 5, "h": 4 },
      "datasource": { "type": "prometheus", "uid": "prometheus" },
      "targets": [{ "refId": "A", "expr": "up{job=\"pihole-node\"}", "instant": true }],
      "fieldConfig": { "defaults": {
        "color": { "mode": "thresholds" },
        "mappings": [{ "type": "value", "options": { "0": { "text": "DOWN", "color": "red" }, "1": { "text": "UP", "color": "green" } } }],
        "thresholds": { "mode": "absolute", "steps": [{ "color": "red", "value": null }, { "color": "green", "value": 1 }] }
      }, "overrides": [] },
      "options": { "colorMode": "background", "graphMode": "none", "textMode": "value", "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false } }
    },
    {
      "type": "stat", "title": "Exporter", "gridPos": { "x": 5, "y": 1, "w": 5, "h": 4 },
      "datasource": { "type": "prometheus", "uid": "prometheus" },
      "targets": [{ "refId": "A", "expr": "up{job=\"pihole-exporter\"}", "instant": true }],
      "fieldConfig": { "defaults": {
        "color": { "mode": "thresholds" },
        "mappings": [{ "type": "value", "options": { "0": { "text": "DOWN", "color": "red" }, "1": { "text": "UP", "color": "green" } } }],
        "thresholds": { "mode": "absolute", "steps": [{ "color": "red", "value": null }, { "color": "green", "value": 1 }] }
      }, "overrides": [] },
      "options": { "colorMode": "background", "graphMode": "none", "textMode": "value", "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false } }
    },
    {
      "type": "stat", "title": "Blocking", "gridPos": { "x": 10, "y": 1, "w": 5, "h": 4 },
      "datasource": { "type": "prometheus", "uid": "prometheus" },
      "targets": [{ "refId": "A", "expr": "pihole_status{job=\"pihole-exporter\"}", "instant": true }],
      "fieldConfig": { "defaults": {
        "color": { "mode": "thresholds" },
        "mappings": [{ "type": "value", "options": { "0": { "text": "Disabled", "color": "red" }, "1": { "text": "Enabled", "color": "green" } } }],
        "thresholds": { "mode": "absolute", "steps": [{ "color": "red", "value": null }, { "color": "green", "value": 1 }] }
      }, "overrides": [] },
      "options": { "colorMode": "background", "graphMode": "none", "textMode": "value", "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false } }
    },
    {
      "type": "stat", "title": "Uptime", "gridPos": { "x": 15, "y": 1, "w": 5, "h": 4 },
      "datasource": { "type": "prometheus", "uid": "prometheus" },
      "targets": [{ "refId": "A", "expr": "time() - node_boot_time_seconds{job=\"pihole-node\"}", "instant": true }],
      "fieldConfig": { "defaults": { "unit": "s", "decimals": 0, "color": { "mode": "fixed", "fixedColor": "text" } }, "overrides": [] },
      "options": { "colorMode": "none", "graphMode": "none", "textMode": "value", "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false } }
    },
    {
      "type": "stat", "title": "SoC temperature", "gridPos": { "x": 20, "y": 1, "w": 4, "h": 4 },
      "datasource": { "type": "prometheus", "uid": "prometheus" },
      "targets": [{ "refId": "A", "expr": "node_thermal_zone_temp{instance=\"pi-hole.lan\", type=\"cpu-thermal\"}", "instant": true }],
      "fieldConfig": { "defaults": {
        "unit": "celsius", "decimals": 1, "color": { "mode": "thresholds" },
        "thresholds": { "mode": "absolute", "steps": [{ "color": "green", "value": null }, { "color": "orange", "value": 70 }, { "color": "red", "value": 80 }] }
      }, "overrides": [] },
      "options": { "colorMode": "value", "graphMode": "area", "textMode": "value", "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false } }
    },

    { "type": "row", "title": "Pi health", "collapsed": false, "gridPos": { "x": 0, "y": 5, "w": 24, "h": 1 }, "panels": [] },
    {
      "type": "timeseries", "title": "CPU used", "gridPos": { "x": 0, "y": 6, "w": 6, "h": 8 },
      "datasource": { "type": "prometheus", "uid": "prometheus" },
      "targets": [{ "refId": "A", "expr": "100 * (1 - avg(rate(node_cpu_seconds_total{job=\"pihole-node\", mode=\"idle\"}[$__rate_interval])))", "legendFormat": "CPU" }],
      "fieldConfig": { "defaults": { "unit": "percent", "min": 0, "max": 100 }, "overrides": [] },
      "options": { "legend": { "showLegend": false }, "tooltip": { "mode": "single" } }
    },
    {
      "type": "timeseries", "title": "Memory used", "gridPos": { "x": 6, "y": 6, "w": 6, "h": 8 },
      "datasource": { "type": "prometheus", "uid": "prometheus" },
      "targets": [{ "refId": "A", "expr": "100 * (1 - node_memory_MemAvailable_bytes{job=\"pihole-node\"} / node_memory_MemTotal_bytes{job=\"pihole-node\"})", "legendFormat": "Memory" }],
      "fieldConfig": { "defaults": {
        "unit": "percent", "min": 0, "max": 100,
        "custom": { "thresholdsStyle": { "mode": "line" } },
        "thresholds": { "mode": "absolute", "steps": [{ "color": "green", "value": null }, { "color": "orange", "value": 80 }, { "color": "red", "value": 90 }] }
      }, "overrides": [] },
      "options": { "legend": { "showLegend": false }, "tooltip": { "mode": "single" } }
    },
    {
      "type": "timeseries", "title": "Root filesystem used", "gridPos": { "x": 12, "y": 6, "w": 6, "h": 8 },
      "datasource": { "type": "prometheus", "uid": "prometheus" },
      "targets": [{ "refId": "A", "expr": "100 * (1 - node_filesystem_avail_bytes{job=\"pihole-node\", mountpoint=\"/\"} / node_filesystem_size_bytes{job=\"pihole-node\", mountpoint=\"/\"})", "legendFormat": "/" }],
      "fieldConfig": { "defaults": {
        "unit": "percent", "min": 0, "max": 100,
        "custom": { "thresholdsStyle": { "mode": "line" } },
        "thresholds": { "mode": "absolute", "steps": [{ "color": "green", "value": null }, { "color": "orange", "value": 80 }, { "color": "red", "value": 90 }] }
      }, "overrides": [] },
      "options": { "legend": { "showLegend": false }, "tooltip": { "mode": "single" } }
    },
    {
      "type": "timeseries", "title": "SoC temperature over time", "gridPos": { "x": 18, "y": 6, "w": 6, "h": 8 },
      "datasource": { "type": "prometheus", "uid": "prometheus" },
      "targets": [{ "refId": "A", "expr": "node_thermal_zone_temp{instance=\"pi-hole.lan\", type=\"cpu-thermal\"}", "legendFormat": "SoC" }],
      "fieldConfig": { "defaults": {
        "unit": "celsius",
        "custom": { "thresholdsStyle": { "mode": "line" } },
        "thresholds": { "mode": "absolute", "steps": [{ "color": "green", "value": null }, { "color": "orange", "value": 70 }, { "color": "red", "value": 80 }] }
      }, "overrides": [] },
      "options": { "legend": { "showLegend": false }, "tooltip": { "mode": "single" } }
    },

    { "type": "row", "title": "DNS", "collapsed": false, "gridPos": { "x": 0, "y": 14, "w": 24, "h": 1 }, "panels": [] },
    {
      "type": "stat", "title": "Queries today", "gridPos": { "x": 0, "y": 15, "w": 6, "h": 4 },
      "datasource": { "type": "prometheus", "uid": "prometheus" },
      "targets": [{ "refId": "A", "expr": "pihole_dns_queries_today{job=\"pihole-exporter\"}", "instant": true }],
      "fieldConfig": { "defaults": { "unit": "short", "decimals": 0, "color": { "mode": "fixed", "fixedColor": "blue" } }, "overrides": [] },
      "options": { "colorMode": "value", "graphMode": "none", "textMode": "value", "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false } }
    },
    {
      "type": "stat", "title": "Blocked today", "gridPos": { "x": 6, "y": 15, "w": 6, "h": 4 },
      "datasource": { "type": "prometheus", "uid": "prometheus" },
      "targets": [{ "refId": "A", "expr": "pihole_ads_blocked_today{job=\"pihole-exporter\"}", "instant": true }],
      "fieldConfig": { "defaults": { "unit": "short", "decimals": 0, "color": { "mode": "fixed", "fixedColor": "red" } }, "overrides": [] },
      "options": { "colorMode": "value", "graphMode": "none", "textMode": "value", "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false } }
    },
    {
      "type": "stat", "title": "% blocked today", "gridPos": { "x": 12, "y": 15, "w": 6, "h": 4 },
      "datasource": { "type": "prometheus", "uid": "prometheus" },
      "targets": [{ "refId": "A", "expr": "pihole_ads_percentage_today{job=\"pihole-exporter\"}", "instant": true }],
      "fieldConfig": { "defaults": { "unit": "percent", "decimals": 1, "color": { "mode": "fixed", "fixedColor": "orange" } }, "overrides": [] },
      "options": { "colorMode": "value", "graphMode": "none", "textMode": "value", "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false } }
    },
    {
      "type": "stat", "title": "Unique clients (24h)", "gridPos": { "x": 18, "y": 15, "w": 6, "h": 4 },
      "datasource": { "type": "prometheus", "uid": "prometheus" },
      "targets": [{ "refId": "A", "expr": "pihole_unique_clients{job=\"pihole-exporter\"}", "instant": true }],
      "fieldConfig": { "defaults": { "unit": "short", "decimals": 0, "color": { "mode": "fixed", "fixedColor": "text" } }, "overrides": [] },
      "options": { "colorMode": "none", "graphMode": "none", "textMode": "value", "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false } }
    },
    {
      "type": "timeseries", "title": "Queries vs blocked (cumulative per day)", "gridPos": { "x": 0, "y": 19, "w": 24, "h": 8 },
      "datasource": { "type": "prometheus", "uid": "prometheus" },
      "targets": [
        { "refId": "A", "expr": "pihole_dns_queries_today{job=\"pihole-exporter\"}", "legendFormat": "Queries" },
        { "refId": "B", "expr": "pihole_ads_blocked_today{job=\"pihole-exporter\"}", "legendFormat": "Blocked" }
      ],
      "fieldConfig": { "defaults": { "unit": "short", "min": 0 }, "overrides": [] },
      "options": { "legend": { "showLegend": true, "displayMode": "list", "placement": "bottom" }, "tooltip": { "mode": "multi" } }
    }
  ]
}
```

Notes for the implementer:
- `up` returns 0 (not absent) when a scrape fails, so the DOWN mapping fires before the exporters exist.
- The two "today" counters reset at Pi-hole's day rollover, so the bottom chart is a daily sawtooth by design.

- [ ] **Step 3: Add the ConfigMap entry**

Append to the `configMapGenerator:` list in `app/kustomization.yaml`:
```yaml
  # custom at-a-glance overview (spec: docs/superpowers/specs/2026-10-04-home-network-dashboards-design.md)
  - name: dashboard-home-network-overview
    files:
      - home-network-overview.json=./dashboards/home-network-overview.json
```

- [ ] **Step 4: Run the checks and validators**

Re-run every command from Step 1, then:
```bash
KC='kubeconform -strict -summary -schema-location default -schema-location https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json'
kustomize build kubernetes/apps/observability/home-network/app | $KC
kustomize build kubernetes/apps/observability/home-network/app | KUBECONFIG=./kubeconfig kubectl apply --dry-run=server --server-side --field-manager=dry-run-check -n observability -f -
```
Expected:
- Step 1's checks print `true` 8 times
- kubeconform `Valid: 5, Invalid: 0`
- Dry-run shows 2 ScrapeConfigs and 3 ConfigMaps `serverside-applied (server dry run)`

- [ ] **Step 5: Do NOT commit.**

---

### Task 4: Full validation, gated commit, and post-deploy verification

**Files:**
- Modify (Step 8 only): `docs/superpowers/specs/2026-10-04-home-network-dashboards-design.md`

**Interfaces:**
- Consumes: everything from Tasks 1–3, plus the spec and this plan (both uncommitted so far).
- Produces: commit(s) on `main` **only after explicit user instruction**, and a verified deployment.

- [ ] **Step 1: Whole-tree validation**

```bash
KC='kubeconform -strict -summary -schema-location default -schema-location https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json'
kustomize build kubernetes/apps/observability | $KC -skip Secret
for f in kubernetes/apps/observability/home-network/app/dashboards/*.json; do jq empty "$f" && echo "OK $f"; done
git status --short
```
Expected:
- kubeconform `Invalid: 0`
- Three `OK` lines
- `git status` shows only: the spec, this plan, `kubernetes/apps/observability/kustomization.yaml`, and the new `home-network/` tree

- [ ] **Step 2: STOP and ask the user for commit authorization**

Show the `git status --short` output and the proposed commit message:
```
feat(observability): scrape Pi-hole Pi and add home network dashboards

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_018dVP2ANV4dBYKrebeEgWEd
```
Proceed to Step 3 only when the user explicitly says to commit. Ask separately about pushing.

- [ ] **Step 3: Commit (only after explicit instruction)**

```bash
git add docs/superpowers/specs/2026-10-04-home-network-dashboards-design.md \
        docs/superpowers/plans/2026-10-04-home-network-dashboards.md \
        kubernetes/apps/observability/kustomization.yaml \
        kubernetes/apps/observability/home-network
git commit -F- <<'EOF'
feat(observability): scrape Pi-hole Pi and add home network dashboards

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_018dVP2ANV4dBYKrebeEgWEd
EOF
```

- [ ] **Step 4: Push (only after explicit instruction), then reconcile**

```bash
git push origin main
task reconcile
```

- [ ] **Step 5: Flux status**

```bash
KUBECONFIG=./kubeconfig flux get ks -n observability home-network
KUBECONFIG=./kubeconfig kubectl -n observability get scrapeconfig pihole-node pihole-exporter -o name
KUBECONFIG=./kubeconfig kubectl -n observability get cm dashboard-node-exporter-full dashboard-pihole-exporter dashboard-home-network-overview
```
Expected:
- Kustomization `Ready True`
- Both ScrapeConfigs exist
- All three ConfigMaps exist

- [ ] **Step 6: Prometheus targets and content integrity (Review Focus 1, 3)**

```bash
KUBECONFIG=./kubeconfig kubectl -n observability exec sts/prometheus-kube-prometheus-stack-prometheus -c prometheus -- \
  wget -qO- 'http://localhost:9090/api/v1/query?query=up%7Binstance%3D%22pi-hole.lan%22%7D' | jq -c '.data.result[] | {job: .metric.job, value: .value[1]}'
KUBECONFIG=./kubeconfig kubectl -n observability get cm dashboard-pihole-exporter -o jsonpath='{.data.pihole-exporter\.json}' | diff -q - kubernetes/apps/observability/home-network/app/dashboards/pihole-exporter.json && echo OK-intact
```
Expected:
- Two results, jobs `pihole-node` and `pihole-exporter`. Value is `"0"` before `home-net` deploys the exporters and `"1"` after.
- `OK-intact`

If the `up` query returns nothing, the ScrapeConfig was not selected. Check the `release` label and `kubectl -n observability logs deploy/kube-prometheus-stack-operator | grep -i scrapeconfig`.

- [ ] **Step 7: Grafana visual check (Review Focus 1, 2)**

Open Grafana (the existing `httproute-grafana.yaml` hostname) and check:
- **Home Network Overview:** before the exporters exist, the Pi and Exporter tiles show red DOWN. After they exist, every panel has data, and the header links open the two detail dashboards.
- **Pi-hole Exporter:** no panel shows "Panel plugin not found" or an Angular deprecation notice. If a pie chart panel is blank but has data, set its legend/values options in the UI, export, and update the JSON.
- **Node Exporter Full:** selecting job `pihole-node` shows the Pi.

Before the exporters exist, only the overview DOWN state is checkable. Repeat this step after `home-net-91` reports the exporters live.

- [ ] **Step 8: Record the stale-data limitation (Review Focus 5)**

Add this bullet to the spec's "Failure modes" section:
```markdown
- **Pi-hole API error while exporter is up:** pihole-exporter v1.2.0 keeps the last gauge values when the Pi-hole API call fails, so DNS tiles show stale numbers instead of blanks. Not detectable from its metrics; check the exporter's logs on the Pi (`journalctl -u <exporter unit>`) if numbers stop changing.
```
Including this in a commit follows the same explicit-instruction gate as Step 2.
