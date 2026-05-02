# Phase 1 — Metrics Core Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Spec:** [`docs/superpowers/specs/2026-04-28-phase-1-metrics-core-design.md`](../specs/2026-04-28-phase-1-metrics-core-design.md)

**Goal:** Deploy `kube-prometheus-stack` (bundled — Prometheus + Alertmanager + Grafana + kube-state-metrics + node-exporter + Prometheus Operator) into a new `observability` namespace, scrape every existing SM/PM cluster-wide, route alerts to one Discord webhook, expose all three UIs LAN-internally via `envoy-internal`.

**Architecture:** Single `HelmRelease` (chart `kube-prometheus-stack` v`83.7.0` from `oci://ghcr.io/prometheus-community/charts/`) with `crds.enabled: false` (CRDs already shipped by `bootstrap/helmfile.d/00-crds.yaml` at the same version). Sibling resources in the same `app/` directory: SOPS-encrypted `Secret` for the Discord webhook URL, three `HTTPRoute` objects attached cross-namespace to `envoy-internal`. Prometheus rule discovery is label-gated on `release: kube-prometheus-stack`; SM/PM/Probe discovery is cluster-wide with empty selectors. Talos-incompatible kps subcharts (kubeControllerManager, kubeScheduler, kubeProxy, kubeEtcd) and their default rule groups are disabled at chart values level.

**Tech Stack:** Flux v2 (kustomize-controller + helm-controller + source-controller), Kustomize, Talos Linux, kube-prometheus-stack chart `83.7.0`, Alertmanager `discord_configs` receiver, Gateway API (envoy-gateway), SOPS+age for secrets.

---

## File map

All new files live under `kubernetes/apps/observability/`. One existing file is modified (`.renovaterc.json5`).

| File | Role |
|---|---|
| `kubernetes/apps/observability/kustomization.yaml` | Namespace-level Kustomize; pulls SOPS component, emits namespace + app Kustomization |
| `kubernetes/apps/observability/namespace.yaml` | `observability` namespace with PSA `privileged` labels |
| `kubernetes/apps/observability/kube-prometheus-stack/ks.yaml` | Flux `Kustomization` CR pointing at `app/`; healthcheck on the HR |
| `kubernetes/apps/observability/kube-prometheus-stack/app/kustomization.yaml` | App-level Kustomize; lists every manifest below |
| `kubernetes/apps/observability/kube-prometheus-stack/app/ocirepository.yaml` | Flux `OCIRepository` pointing at `oci://ghcr.io/prometheus-community/charts/kube-prometheus-stack` |
| `kubernetes/apps/observability/kube-prometheus-stack/app/helmrelease.yaml` | Flux `HelmRelease`, all values inline, `crds.enabled: false` |
| `kubernetes/apps/observability/kube-prometheus-stack/app/secret.sops.yaml` | SOPS-encrypted Secret `discord-webhook` (key `url`) |
| `kubernetes/apps/observability/kube-prometheus-stack/app/httproute-grafana.yaml` | HTTPRoute for `grafana.${SECRET_DOMAIN}` |
| `kubernetes/apps/observability/kube-prometheus-stack/app/httproute-prometheus.yaml` | HTTPRoute for `prometheus.${SECRET_DOMAIN}` |
| `kubernetes/apps/observability/kube-prometheus-stack/app/httproute-alertmanager.yaml` | HTTPRoute for `alertmanager.${SECRET_DOMAIN}` |
| `.renovaterc.json5` | **Modify** — add `packageRules` entry grouping CRD chart + HR chart into a single PR |

Zero changes to `kubernetes/flux/cluster/ks.yaml` — Flux's `cluster-apps` Kustomization reconciles `./kubernetes/apps` recursively and discovers new namespace directories automatically (verified — see Phase 0 plan).

---

## Task 1: Verify prerequisites

**Files:** (none)

- [ ] **Step 1: Confirm cluster access.**

  Run:
  ```bash
  kubectl get nodes
  ```
  Expected: two `Ready` nodes (`ctrl01`, `work01`).

- [ ] **Step 2: Confirm Flux is healthy.**

  Run:
  ```bash
  flux check && flux get ks -A && flux get hr -A
  ```
  Expected: no errors; all existing Kustomizations and HelmReleases `Ready=True`.

- [ ] **Step 3: Confirm storage classes from Phase 0 exist.**

  Run:
  ```bash
  kubectl get sc
  ```
  Expected: `local-path` (default) and `local-path-retain` both present.

- [ ] **Step 4: Confirm kube-prometheus-stack CRDs are installed and at the expected chart version.**

  Run:
  ```bash
  kubectl get crd prometheuses.monitoring.coreos.com alertmanagers.monitoring.coreos.com servicemonitors.monitoring.coreos.com podmonitors.monitoring.coreos.com prometheusrules.monitoring.coreos.com probes.monitoring.coreos.com -o name | sort
  grep -A2 'kube-prometheus-stack' bootstrap/helmfile.d/00-crds.yaml | head -5
  ```
  Expected: all six CRDs printed; helmfile shows `version: 83.7.0` on the kube-prometheus-stack release. If the helmfile version differs (renovate may have bumped it since this plan was written), use that version everywhere `83.7.0` appears in this plan.

- [ ] **Step 5: Confirm the chart URL is reachable.**

  Run:
  ```bash
  helm show chart oci://ghcr.io/prometheus-community/charts/kube-prometheus-stack --version 83.7.0 | head -10
  ```
  Expected: chart metadata printed (`name: kube-prometheus-stack`, `version: 83.7.0`, an appVersion).

- [ ] **Step 6: Confirm the Discord webhook URL is in hand.**

  You need a Discord webhook URL of the form `https://discord.com/api/webhooks/<channel-id>/<token>` already created in the target Discord channel (Channel Settings → Integrations → Webhooks → New Webhook → Copy URL). The URL is the only secret in this plan and must be available before Task 4.

  If you don't have one yet, stop and create one — this plan cannot complete without it.

- [ ] **Step 7: Confirm SOPS encryption is configured locally.**

  Run:
  ```bash
  test -f age.key && echo "age.key present" || echo "MISSING age.key"
  cat .sops.yaml | grep -A3 'kubernetes/'
  ```
  Expected: `age.key present`; `.sops.yaml` contains the `(bootstrap|kubernetes)/.*\.sops\.ya?ml` rule encrypting `^(data|stringData)$`.

- [ ] **Step 8: Confirm gateway and wildcard cert are in place.**

  Run:
  ```bash
  kubectl -n network get gateway envoy-internal -o jsonpath='{.status.addresses[0].value}{"\n"}'
  kubectl -n network get secret | grep -- '-production-tls'
  ```
  Expected: gateway address `192.168.1.86`; one Secret matching `*-production-tls`.

- [ ] **Step 9: Resolve `${SECRET_DOMAIN}` for use in this plan.**

  Run:
  ```bash
  kubectl -n flux-system get secret cluster-secrets -o jsonpath='{.data.SECRET_DOMAIN}' | base64 -d; echo
  ```
  Expected: your Cloudflare domain (e.g. `example.com`). Note this value — it is the literal that `${SECRET_DOMAIN}` resolves to via Flux postBuild substitution at reconcile time.

---

## Task 2: Scaffold the `observability` namespace

**Files:**
- Create: `kubernetes/apps/observability/namespace.yaml`
- Create: `kubernetes/apps/observability/kustomization.yaml`

- [ ] **Step 1: Create the directory.**

  Run:
  ```bash
  mkdir -p kubernetes/apps/observability/kube-prometheus-stack/app
  ```

- [ ] **Step 2: Write `namespace.yaml`.**

  Create `kubernetes/apps/observability/namespace.yaml`:
  ```yaml
  ---
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

  `privileged` is required because node-exporter mounts hostPath, hostNetwork, and hostPID. The prune annotation prevents Flux from deleting the namespace itself if the parent Kustomization is removed.

- [ ] **Step 3: Write the namespace-level `kustomization.yaml`.**

  Create `kubernetes/apps/observability/kustomization.yaml`:
  ```yaml
  ---
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  namespace: observability

  components:
    - ../../components/sops

  resources:
    - ./namespace.yaml
    - ./kube-prometheus-stack/ks.yaml
  ```

  This matches the Phase 0 storage namespace exactly (`kubernetes/apps/storage/kustomization.yaml`). The SOPS component is required because the app ships an encrypted Secret.

- [ ] **Step 4: Verify with kustomize build.**

  Run:
  ```bash
  kubectl kustomize kubernetes/apps/observability/ 2>&1 | head -40
  ```
  Expected: emits the Namespace manifest. Will error on `./kube-prometheus-stack/ks.yaml` not existing yet — that's fine for now (next task adds it). To check just the namespace, temporarily comment the `ks.yaml` line and re-run, then uncomment.

---

## Task 3: Scaffold the kube-prometheus-stack app skeleton

**Files:**
- Create: `kubernetes/apps/observability/kube-prometheus-stack/ks.yaml`
- Create: `kubernetes/apps/observability/kube-prometheus-stack/app/ocirepository.yaml`
- Create: `kubernetes/apps/observability/kube-prometheus-stack/app/kustomization.yaml`

- [ ] **Step 1: Write the Flux `Kustomization` CR (`ks.yaml`).**

  Create `kubernetes/apps/observability/kube-prometheus-stack/ks.yaml`:
  ```yaml
  ---
  apiVersion: kustomize.toolkit.fluxcd.io/v1
  kind: Kustomization
  metadata:
    name: kube-prometheus-stack
  spec:
    healthChecks:
      - apiVersion: helm.toolkit.fluxcd.io/v2
        kind: HelmRelease
        name: kube-prometheus-stack
        namespace: observability
    interval: 1h
    path: ./kubernetes/apps/observability/kube-prometheus-stack/app
    postBuild:
      substituteFrom:
        - name: cluster-secrets
          kind: Secret
    prune: true
    sourceRef:
      kind: GitRepository
      name: flux-system
      namespace: flux-system
    targetNamespace: observability
    wait: true
    timeout: 15m
  ```

  Notes:
  - `postBuild.substituteFrom` is what makes `${SECRET_DOMAIN}` in the HTTPRoutes resolve.
  - `timeout: 15m` (not the 5m default) accommodates Prometheus's first-startup TSDB checkpoint.

- [ ] **Step 2: Write the `OCIRepository`.**

  Create `kubernetes/apps/observability/kube-prometheus-stack/app/ocirepository.yaml`:
  ```yaml
  ---
  apiVersion: source.toolkit.fluxcd.io/v1
  kind: OCIRepository
  metadata:
    name: kube-prometheus-stack
  spec:
    interval: 30m
    layerSelector:
      mediaType: application/vnd.cncf.helm.chart.content.v1.tar+gzip
      operation: copy
    url: oci://ghcr.io/prometheus-community/charts/kube-prometheus-stack
    ref:
      tag: 83.7.0
  ```

  Renovate's "Process OCI dependencies" customManager (regex `oci://(?<depName>[^:]+):(?<currentValue>\S+)`) is keyed on the colon between URL and version. With `ref.tag` separate, that regex won't match. To restore renovate tracking, the repo convention is to keep the URL inline; check how `kubernetes/apps/storage/local-path-provisioner/app/helmrepository.yaml` handles this — actually that's an HTTPS HelmRepository which uses `chart.spec.version` in the HelmRelease and is tracked by the `helm` datasource manager, not the OCI regex. For OCI, look at how `kubernetes/apps/network/cloudflare-dns/app/ocirepository.yaml` declares its tag — match its style. If renovate annotation is required, prepend a comment like `# renovate: datasource=docker depName=ghcr.io/prometheus-community/charts/kube-prometheus-stack`.

- [ ] **Step 3: Cross-reference an existing OCIRepository in the repo.**

  Run:
  ```bash
  cat kubernetes/apps/network/cloudflare-dns/app/ocirepository.yaml
  ```
  Expected: shows the existing OCI pattern (likely `ref.tag: 1.20.0` style, possibly with renovate annotation). Update Step 2's manifest to match style exactly (annotation placement, `layerSelector` presence, etc.). The `external-dns` chart is renovate-tracked successfully today, so its file is the canonical template.

- [ ] **Step 4: Write a stub app-level `kustomization.yaml`.**

  Create `kubernetes/apps/observability/kube-prometheus-stack/app/kustomization.yaml`:
  ```yaml
  ---
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
    - ./ocirepository.yaml
    - ./helmrelease.yaml
    - ./secret.sops.yaml
    - ./httproute-grafana.yaml
    - ./httproute-prometheus.yaml
    - ./httproute-alertmanager.yaml
  ```

  The remaining files are added in Tasks 4–6. `kustomize build` will fail until those exist — that's expected.

---

## Task 4: Encrypt and add the Discord webhook Secret

**Files:**
- Create: `kubernetes/apps/observability/kube-prometheus-stack/app/secret.sops.yaml`

- [ ] **Step 1: Write the plain-text Secret manifest temporarily.**

  Create `kubernetes/apps/observability/kube-prometheus-stack/app/secret.sops.yaml` with the **literal** webhook URL (from Task 1 Step 6):
  ```yaml
  ---
  apiVersion: v1
  kind: Secret
  metadata:
    name: discord-webhook
    namespace: observability
  type: Opaque
  stringData:
    url: https://discord.com/api/webhooks/REPLACE_WITH_REAL_CHANNEL_ID/REPLACE_WITH_REAL_TOKEN
  ```

  Do **not** commit this file in plain text. The next step encrypts it in place.

- [ ] **Step 2: Encrypt in place with SOPS.**

  Run:
  ```bash
  sops --encrypt --in-place kubernetes/apps/observability/kube-prometheus-stack/app/secret.sops.yaml
  ```
  Expected: command exits 0 silently. The file is rewritten in place; only the `stringData.url` value is encrypted (per the `.sops.yaml` `encrypted_regex: ^(data|stringData)$` rule).

- [ ] **Step 3: Verify encryption.**

  Run:
  ```bash
  grep -E '^\s+url:\s+' kubernetes/apps/observability/kube-prometheus-stack/app/secret.sops.yaml
  ```
  Expected: a line beginning with `    url: ENC[AES256_GCM,...` — if it still shows `url: https://discord...`, **stop and re-encrypt**. Do NOT proceed to commit a plain-text webhook.

- [ ] **Step 4: Verify decryption round-trip.**

  Run:
  ```bash
  SOPS_AGE_KEY_FILE=./age.key sops --decrypt kubernetes/apps/observability/kube-prometheus-stack/app/secret.sops.yaml | grep -E '^\s+url:\s+https://discord'
  ```
  Expected: prints the original webhook URL line. Confirms the file decrypts correctly.

---

## Task 5: Add the HelmRelease

**Files:**
- Create: `kubernetes/apps/observability/kube-prometheus-stack/app/helmrelease.yaml`

- [ ] **Step 1: Write `helmrelease.yaml` with full values inline.**

  Create `kubernetes/apps/observability/kube-prometheus-stack/app/helmrelease.yaml`:
  ```yaml
  ---
  apiVersion: helm.toolkit.fluxcd.io/v2
  kind: HelmRelease
  metadata:
    name: kube-prometheus-stack
  spec:
    interval: 1h
    chartRef:
      kind: OCIRepository
      name: kube-prometheus-stack
    values:
      crds:
        enabled: false
      defaultRules:
        create: true
        disabled:
          kubeControllerManager: true
          kubeScheduler: true
          kubeProxy: true
          etcd: true
      kubeControllerManager:
        enabled: false
      kubeScheduler:
        enabled: false
      kubeProxy:
        enabled: false
      kubeEtcd:
        enabled: false
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
                accessModes: ["ReadWriteOnce"]
                resources:
                  requests:
                    storage: 30Gi
      alertmanager:
        alertmanagerSpec:
          storage:
            volumeClaimTemplate:
              spec:
                storageClassName: local-path
                accessModes: ["ReadWriteOnce"]
                resources:
                  requests:
                    storage: 1Gi
          secrets:
            - discord-webhook
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

  Notes:
  - `chartRef` (not `chart.spec`) is used for OCI sources in helm-controller v1+.
  - `defaultRules.disabled` keys are **group names** (e.g. `kubeControllerManager`), not alert names. The chart's `_helpers.tpl` iterates over rule group names to apply the disabled map — alert-name keys are silently ignored. The four entries (`kubeControllerManager`, `kubeScheduler`, `kubeProxy`, `etcd`) match the groups whose subcharts are also disabled at the subchart level.
  - `accessModes: ["ReadWriteOnce"]` is explicit because some kps versions default to a list including RWX, which the local-path provisioner does not support.
  - Grafana's `sidecar.dashboards.searchNamespace: ALL` is the Helm chart's literal value for "all namespaces" — case-sensitive.

- [ ] **Step 2: Local kustomize build.**

  Run:
  ```bash
  kubectl kustomize kubernetes/apps/observability/kube-prometheus-stack/app/ 2>&1 | head -60
  ```
  Expected: emits OCIRepository, HelmRelease, Secret (encrypted), and references three HTTPRoutes that don't exist yet. The build will fail with `accumulating resources: ... no such file or directory` for the three HTTPRoute files — that's expected; Task 6 adds them.

---

## Task 6: Add the three HTTPRoutes

**Files:**
- Create: `kubernetes/apps/observability/kube-prometheus-stack/app/httproute-grafana.yaml`
- Create: `kubernetes/apps/observability/kube-prometheus-stack/app/httproute-prometheus.yaml`
- Create: `kubernetes/apps/observability/kube-prometheus-stack/app/httproute-alertmanager.yaml`

- [ ] **Step 1: Write `httproute-grafana.yaml`.**

  Create `kubernetes/apps/observability/kube-prometheus-stack/app/httproute-grafana.yaml`:
  ```yaml
  ---
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: grafana
  spec:
    hostnames: ["grafana.${SECRET_DOMAIN}"]
    parentRefs:
      - name: envoy-internal
        namespace: network
        sectionName: https
    rules:
      - backendRefs:
          - name: kube-prometheus-stack-grafana
            port: 80
  ```

- [ ] **Step 2: Write `httproute-prometheus.yaml`.**

  Create `kubernetes/apps/observability/kube-prometheus-stack/app/httproute-prometheus.yaml`:
  ```yaml
  ---
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: prometheus
  spec:
    hostnames: ["prometheus.${SECRET_DOMAIN}"]
    parentRefs:
      - name: envoy-internal
        namespace: network
        sectionName: https
    rules:
      - backendRefs:
          - name: kube-prometheus-stack-prometheus
            port: 9090
  ```

- [ ] **Step 3: Write `httproute-alertmanager.yaml`.**

  Create `kubernetes/apps/observability/kube-prometheus-stack/app/httproute-alertmanager.yaml`:
  ```yaml
  ---
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: alertmanager
  spec:
    hostnames: ["alertmanager.${SECRET_DOMAIN}"]
    parentRefs:
      - name: envoy-internal
        namespace: network
        sectionName: https
    rules:
      - backendRefs:
          - name: kube-prometheus-stack-alertmanager
            port: 9093
  ```

- [ ] **Step 4: Local kustomize build (full app dir).**

  Run:
  ```bash
  kubectl kustomize kubernetes/apps/observability/kube-prometheus-stack/app/ > /tmp/kps-build.yaml && wc -l /tmp/kps-build.yaml
  ```
  Expected: builds without error; output is several hundred lines (OCIRepository + HelmRelease + Secret + 3 HTTPRoutes). The HelmRelease is rendered as one CR — chart contents are NOT rendered locally (helm-controller does that in-cluster).

- [ ] **Step 5: Local kustomize build (full namespace dir).**

  Run:
  ```bash
  kubectl kustomize kubernetes/apps/observability/ | head -20
  ```
  Expected: emits Namespace + Flux Kustomization CR. (The app-level resources are not pulled here — they're materialized only when Flux reconciles `ks.yaml`.)

---

## Task 7: Add the Renovate group rule

**Files:**
- Modify: `.renovaterc.json5` (insert one new entry into `packageRules`)

- [ ] **Step 1: Locate the `packageRules` array.**

  Open `.renovaterc.json5`. The `packageRules` array starts around line 31 with `packageRules: [`. Find the existing "Flux Operator Group" block (around line 39) — the new entry mirrors its shape.

- [ ] **Step 2: Insert the new group rule.**

  Add this entry to `packageRules`, immediately after the "Flux Operator Group" block:
  ```json5
      {
        description: "kube-prometheus-stack chart and CRD lockstep",
        groupName: "kube-prometheus-stack",
        matchDatasources: ["docker"],
        matchPackageNames: ["/kube-prometheus-stack/"],
        group: {
          commitMessageTopic: "{{{groupName}}} group",
        },
      },
  ```

  This matches both the bootstrap helmfile entry (`bootstrap/helmfile.d/00-crds.yaml`) and the new HelmRelease's OCIRepository, grouping any kps version bump into a single PR.

- [ ] **Step 3: Validate JSON5 syntax.**

  Run:
  ```bash
  command -v json5 >/dev/null && json5 --validate .renovaterc.json5 \
    || python3 -c "import json5,sys; json5.load(open('.renovaterc.json5')); print('OK')"
  ```
  Expected: `OK` (or json5's silent success). If neither tool is available, install `pip install json5` first.

---

## Task 8: Local validation

**Files:** (none)

- [ ] **Step 1: Run repo's kubeconform task.**

  Run:
  ```bash
  task template:kubeconform 2>&1 | tee /tmp/kubeconform.log | tail -40
  ```
  Expected: exit 0; no failures. If `task` reports the `template:kubeconform` task isn't present (e.g. `task template:tidy` was already run, archiving the templates Taskfile), fall back to running the script directly:
  ```bash
  bash .taskfiles/template/resources/kubeconform.sh kubernetes 2>&1 | tail -40
  ```

  Common failures and fixes:
  - **"could not find schema for kind X"** for `HTTPRoute`, `OCIRepository`, `HelmRelease`, `Kustomization` (Flux), `EnvoyProxy`, etc.: schemas live in additional registries; the script already loads them via `--schema-location`. If still missing, the kubeconform script has been changed since this plan was written — check its `--schema-location` arguments.
  - **"failed schema validation"** with field-level error: likely a typo in the YAML; fix and re-run.

- [ ] **Step 2: Decrypt-test the secret one more time.**

  Run:
  ```bash
  SOPS_AGE_KEY_FILE=./age.key sops --decrypt kubernetes/apps/observability/kube-prometheus-stack/app/secret.sops.yaml | yq '.stringData.url' | grep -E '^https://discord\.com/api/webhooks/[0-9]+/[A-Za-z0-9_-]+$'
  ```
  Expected: prints the URL on stdout (regex match confirms shape). If empty or non-matching, the encrypted file is wrong — re-do Task 4.

- [ ] **Step 3: Confirm git sees only the expected new files.**

  Run:
  ```bash
  git status --porcelain
  ```
  Expected exactly these lines (order may vary):
  ```
  ?? kubernetes/apps/observability/
   M .renovaterc.json5
  ```

- [ ] **Step 4: Confirm the Secret is encrypted in the working tree.**

  Run:
  ```bash
  grep -E '^\s+url:' kubernetes/apps/observability/kube-prometheus-stack/app/secret.sops.yaml
  ```
  Expected: line beginning `    url: ENC[AES256_GCM,`. If not, **STOP** — do not commit a plain webhook URL.

---

## Task 9: Commit and push

**Files:** (commits new files + `.renovaterc.json5` modification)

- [ ] **Step 1: Confirm push authorization.**

  This task pushes to `origin`. Per the user's standing rule, do not run `git push` unless the user has explicitly authorized push in the current turn. If unsure, ask: "Phase 1 files are ready to commit and push. Push to origin/main, push to a feature branch, or hold the commit local?"

- [ ] **Step 2: Stage explicit paths.**

  Run:
  ```bash
  git add kubernetes/apps/observability/ .renovaterc.json5
  git status
  ```
  Expected: only the Phase 1 paths are staged. If anything else appears, unstage it.

- [ ] **Step 3: Commit.**

  Run:
  ```bash
  git commit -m "$(cat <<'EOF'
  feat(observability): add Phase 1 metrics core (kube-prometheus-stack)

  Deploy bundled kube-prometheus-stack 83.7.0 into a new observability namespace:
  Prometheus (30 GiB / 30d retention on local-path-retain), Alertmanager (1 GiB
  on local-path) with a single Discord receiver, Grafana (2 GiB on local-path)
  with sidecar dashboard discovery (label grafana_dashboard=1, all namespaces),
  plus kube-state-metrics and node-exporter. SM/PM/Probe discovery is
  cluster-wide; PrometheusRule discovery is label-gated on
  release=kube-prometheus-stack. Talos-incompatible subcharts disabled
  (kubeControllerManager, kubeScheduler, kubeProxy, kubeEtcd). CRDs continue
  to come from bootstrap/helmfile.d/00-crds.yaml.

  Three HTTPRoutes attach Grafana/Prometheus/Alertmanager to envoy-internal
  using the wildcard cert; LAN-only access per Q3-A of the spec.

  Renovate packageRule added to group the bootstrap CRD chart and the new
  HelmRelease into a single PR on future bumps.

  Spec: docs/superpowers/specs/2026-04-28-phase-1-metrics-core-design.md

  Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
  EOF
  )"
  ```

- [ ] **Step 4: Push (only if authorized in Step 1).**

  Run (assuming push approved):
  ```bash
  git push origin HEAD
  ```
  Expected: push succeeds. If the current branch isn't tracked, `git push -u origin <branch>` instead.

---

## Task 10: Watch Flux reconcile

**Files:** (none)

- [ ] **Step 1: Force a reconciliation.**

  Run:
  ```bash
  flux reconcile source git flux-system
  flux reconcile kustomization cluster-apps
  ```
  Expected: both report `applied revision`. The cluster-apps reconciliation triggers the new `Kustomization/kube-prometheus-stack` discovery.

- [ ] **Step 2: Wait for the new Kustomization to appear.**

  Run:
  ```bash
  for i in $(seq 1 30); do
    kubectl -n flux-system get kustomization kube-prometheus-stack -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}{"\n"}' 2>/dev/null && break
    sleep 5
  done
  ```
  Expected: prints `Unknown` then `True` within ~2 minutes. If it stays `Unknown` past 5 minutes or prints `False`, inspect:
  ```bash
  kubectl -n flux-system describe kustomization kube-prometheus-stack | tail -30
  ```

- [ ] **Step 3: Wait for the HelmRelease to install.**

  Run:
  ```bash
  flux -n observability get hr kube-prometheus-stack
  ```
  Initial state: `Reconciling` → `Ready=False (Reason: InstallFailed?)` is possible during first install while pods schedule. Re-check every ~30s. The `timeout: 15m` from Task 3 Step 1 governs how long Flux waits for `wait: true` to succeed.

- [ ] **Step 4: Stream events if it stalls.**

  If `Ready=True` doesn't appear within 15 minutes:
  ```bash
  kubectl -n observability get events --sort-by='.metadata.creationTimestamp' | tail -40
  kubectl -n observability get hr kube-prometheus-stack -o yaml | yq '.status'
  ```
  Most likely first-time failures:
  - **PVC pending** — confirm storage class names exactly match `local-path` / `local-path-retain` (Task 5 Step 1 values).
  - **Secret `discord-webhook` not found** — confirm Task 4 ran and the SOPS component is in the kustomization (Task 2 Step 3).
  - **Chart values rejected** — schema mismatch; check the Helm chart version's `values.schema.json` field names.

---

## Task 11: Validate — Flux / Pods / PVCs (spec steps 1–3)

**Files:** (none)

- [ ] **Step 1: Flux green.**

  Run:
  ```bash
  flux -n flux-system get ks kube-prometheus-stack
  flux -n observability get hr kube-prometheus-stack
  ```
  Expected: both `Ready=True`.

- [ ] **Step 2: Pods Running.**

  Run:
  ```bash
  kubectl -n observability get pods -o wide
  ```
  Expected (all `Running`, all `READY` denominators matched):
  ```
  alertmanager-kube-prometheus-stack-alertmanager-0
  prometheus-kube-prometheus-stack-prometheus-0
  kube-prometheus-stack-operator-<hash>
  kube-prometheus-stack-grafana-<hash>
  kube-prometheus-stack-kube-state-metrics-<hash>
  kube-prometheus-stack-prometheus-node-exporter-<hash> (×2; one per node)
  ```

- [ ] **Step 3: PVCs bound to the right storage classes.**

  Run:
  ```bash
  kubectl -n observability get pvc -o wide
  ```
  Expected: 3 `Bound`. Verify:
  - Prometheus PVC → `STORAGECLASS=local-path-retain`, `CAPACITY=30Gi`
  - Alertmanager PVC → `STORAGECLASS=local-path`, `CAPACITY=1Gi`
  - Grafana PVC → `STORAGECLASS=local-path`, `CAPACITY=2Gi`

  If a PVC is `Pending`: `kubectl -n observability describe pvc <name>` will explain why. Most common cause: PVC requested a SC that doesn't exist (typo in HR values).

- [ ] **Step 4: Disabled-component check (spec validation step 10, run early).**

  Run:
  ```bash
  kubectl get servicemonitor -A | grep -E 'kube-controller-manager|kube-scheduler|kube-proxy|etcd' || echo "none (expected)"
  kubectl -n observability get prometheusrule -o yaml | grep -E 'KubeControllerManagerDown|KubeSchedulerDown|KubeProxyDown' || echo "none (expected)"
  ```
  Expected: both print `none (expected)`. If the second command shows hits, the `defaultRules.disabled` block in the HR is using the wrong key form (alert-name vs group-name); switch to group-name form (`kubeControllerManager: true`, etc.) and re-reconcile.

---

## Task 12: Validate — Targets / Watchdog / Discord (spec steps 4–6)

**Files:** (none)

- [ ] **Step 1: DNS sanity check from your workstation.**

  Run:
  ```bash
  dig +short prometheus.${SECRET_DOMAIN} grafana.${SECRET_DOMAIN} alertmanager.${SECRET_DOMAIN}
  ```
  (Substitute the literal domain from Task 1 Step 9.)
  Expected: each resolves to `192.168.1.86`. If they don't, k8s_gateway forwarding from your home DNS isn't set up (see AGENTS.md "Networking and DNS" section) — fix at the home-DNS layer before continuing.

- [ ] **Step 2: Targets healthy.**

  Browse to `https://prometheus.<your-domain>/targets`. Required UP (one row per labelled instance):
  - `serviceMonitor/observability/kube-prometheus-stack-prometheus`
  - `serviceMonitor/observability/kube-prometheus-stack-alertmanager`
  - `serviceMonitor/observability/kube-prometheus-stack-operator`
  - `serviceMonitor/observability/kube-prometheus-stack-grafana`
  - `serviceMonitor/observability/kube-prometheus-stack-kube-state-metrics`
  - `serviceMonitor/observability/kube-prometheus-stack-kubelet` (3 endpoint paths × 2 nodes)
  - `podMonitor/observability/kube-prometheus-stack-prometheus-node-exporter` (2 pods)

  Pre-existing SMs/PMs from elsewhere in the repo (envoy-gateway PodMonitor, Cilium SMs, CoreDNS SM, flux-system SMs) should also appear. Note any DOWN ones in a follow-up — they are not Phase 1 blockers.

- [ ] **Step 3: Watchdog firing.**

  Browse to `https://prometheus.<your-domain>/alerts`. Filter: `Watchdog`.
  Expected: one alert in state `firing`. This is kps's heartbeat — it must always be firing.

  If not firing: the rule discovery label is mismatched. Check `kubectl -n observability get prometheusrule -o yaml | grep -A2 'name: Watchdog'` — the rule should carry label `release: kube-prometheus-stack` matching the HR's `prometheus.prometheusSpec.ruleSelector`.

- [ ] **Step 4: Discord delivery.**

  Within ~30 seconds of Watchdog being firing, a message appears in the configured Discord channel. The message will be from "Captain Hook" (Discord's default webhook bot name unless customized) and contain the alertname `Watchdog`.

  If nothing arrives within 2 minutes:
  ```bash
  kubectl -n observability logs -l app.kubernetes.io/name=alertmanager --tail=200 | grep -iE 'discord|notify|error'
  ```
  Common causes: webhook URL wrong (test with `curl -H "Content-Type: application/json" -d '{"content":"test"}' "<url>"`), file path typo (must be `/etc/alertmanager/secrets/discord-webhook/url`), `alertmanager.config` syntax error (will show in logs as config validation failure).

---

## Task 13: Validate — Grafana / sidecar (spec steps 7–8)

**Files:** (none)

- [ ] **Step 1: Read the generated Grafana admin password.**

  Run:
  ```bash
  kubectl -n observability get secret kube-prometheus-stack-grafana \
    -o jsonpath='{.data.admin-password}' | base64 -d; echo
  ```
  Expected: a random 12+ character string.

- [ ] **Step 2: Grafana login + datasources.**

  Browse to `https://grafana.<your-domain>`. Log in as `admin` with the password from Step 1.
  Open Connections → Data sources. Expected: `Prometheus` (default) and `Alertmanager` both present and showing "Working" on the Test button.

- [ ] **Step 3: Default dashboards populate.**

  Open Dashboards → Browse → `General/Kubernetes/...`. Open "Kubernetes / Compute Resources / Cluster".
  Expected: graphs render with non-zero data within ~30 seconds of opening (Prometheus needs a couple of scrape cycles to have data).

  Open "Node Exporter / Nodes". Expected: both nodes (`ctrl01`, `work01`) appear in the dropdown; metrics non-zero.

- [ ] **Step 4: Sidecar smoke test — create.**

  Apply a labelled ConfigMap in `default`:
  ```bash
  cat <<'EOF' | kubectl apply -f -
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: test-dashboard
    namespace: default
    labels:
      grafana_dashboard: "1"
  data:
    test.json: |
      {
        "title": "sidecar-smoke-test",
        "uid": "sidecar-smoke",
        "schemaVersion": 39,
        "panels": []
      }
  EOF
  ```

- [ ] **Step 5: Confirm sidecar picks it up.**

  Run:
  ```bash
  kubectl -n observability logs -l app.kubernetes.io/name=grafana -c grafana-sc-dashboard --tail=20 | grep test-dashboard
  ```
  Expected: line indicating the sidecar wrote `test.json` to disk and reloaded Grafana provisioning.

  Then in Grafana UI: Dashboards → Browse — `sidecar-smoke-test` (uid `sidecar-smoke`) appears within ~60s.

- [ ] **Step 6: Sidecar smoke test — delete.**

  Run:
  ```bash
  kubectl -n default delete configmap test-dashboard
  ```
  Expected: dashboard disappears from Grafana UI within ~60s. Sidecar log shows the file was removed.

---

## Task 14: Validate — Restart resilience / silence-the-flood follow-up

**Files:** (none — may produce a follow-up commit)

- [ ] **Step 1: Restart resilience — delete Prometheus pod.**

  Capture the bound node, then delete:
  ```bash
  ORIG_NODE=$(kubectl -n observability get pod prometheus-kube-prometheus-stack-prometheus-0 -o jsonpath='{.spec.nodeName}')
  echo "originally on: $ORIG_NODE"
  kubectl -n observability delete pod prometheus-kube-prometheus-stack-prometheus-0
  ```

- [ ] **Step 2: Confirm reschedule on the same node + history intact.**

  Run:
  ```bash
  kubectl -n observability wait pod prometheus-kube-prometheus-stack-prometheus-0 --for=condition=Ready --timeout=300s
  NEW_NODE=$(kubectl -n observability get pod prometheus-kube-prometheus-stack-prometheus-0 -o jsonpath='{.spec.nodeName}')
  echo "now on: $NEW_NODE"
  test "$ORIG_NODE" = "$NEW_NODE" && echo "OK: same node (PV pinning works)" || echo "FAIL: scheduled to different node — PV should have prevented this"
  ```
  Expected: `OK: same node`. (PV pinning is what `WaitForFirstConsumer` + node-local volume guarantees.)

  Then in Grafana, re-open "Kubernetes / Compute Resources / Cluster" and zoom out to 1 hour. Expected: timeseries shows a small (< 1 min) gap exactly at the pod restart, then continues — no full data loss.

- [ ] **Step 3: 1-hour Discord noise budget.**

  Wait at least 1 hour after Task 12 Step 4 (first Watchdog delivery). Count non-Watchdog messages received in Discord during that window.

  - **0–2 non-Watchdog pages:** acceptable; proceed to Task 15.
  - **3+ non-Watchdog pages:** identify the noisy alerts (each Discord message has the alertname). For each one:
    1. Decide if it's signal or noise (e.g. `KubeAPIErrorBudgetBurn` on a 2-node cluster is almost certainly noise).
    2. Add it to the HR's `defaultRules.disabled` block under its rule-name or group-name (whichever form the chart expects — see Task 5 Step 1 note).
    3. Commit the change with message `fix(observability): silence noisy <alertname>` and a one-line rationale.
    4. Wait another hour, re-evaluate.

  This step may produce one or more follow-up commits before Task 15 is allowed.

---

## Task 15: Mark spec implemented

**Files:**
- Modify: `docs/superpowers/specs/2026-04-28-phase-1-metrics-core-design.md` (status header)

- [ ] **Step 1: Update the spec status line.**

  Open `docs/superpowers/specs/2026-04-28-phase-1-metrics-core-design.md`. Replace:
  ```
  **Status:** Draft
  ```
  with:
  ```
  **Status:** Implemented YYYY-MM-DD — Watchdog stable in Discord; <N> follow-up rules silenced (see commits)
  ```
  Substitute today's date (`date +%Y-%m-%d`) and the actual count of silence commits from Task 14 Step 3 (`0` is fine).

- [ ] **Step 2: Commit the status update.**

  Run:
  ```bash
  git add docs/superpowers/specs/2026-04-28-phase-1-metrics-core-design.md
  git commit -m "$(cat <<'EOF'
  docs(superpowers): mark Phase 1 metrics-core spec implemented

  Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
  EOF
  )"
  ```

- [ ] **Step 3: Push (only if authorized).**

  Same authorization rule as Task 9 Step 1 / 4. Ask before pushing if not explicitly authorized.

  Run (if approved):
  ```bash
  git push origin HEAD
  ```

- [ ] **Step 4: Done condition.**

  Confirm:
  - All 10 spec validation steps green (Tasks 11–13 cover them all).
  - Watchdog firing in Discord continuously for ≥1h with ≤2 non-Watchdog pages, OR all noisy rules silenced via follow-up commits in Task 14 Step 3.
  - Spec marked Implemented.

  If all true, Phase 1 is done. The next phase per Phase 0's roadmap is Victoria-Logs (logs); rule curation or etcd scrape are smaller sub-phase candidates if either felt load-bearing during this rollout. Topic selection is the user's next brainstorm.
