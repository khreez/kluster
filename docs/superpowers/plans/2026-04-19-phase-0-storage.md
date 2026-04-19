# Phase 0 — Persistent Storage Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Spec:** [`docs/superpowers/specs/2026-04-19-phase-0-storage-design.md`](../specs/2026-04-19-phase-0-storage-design.md)

**Goal:** Deploy Rancher `local-path-provisioner` via Flux so the cluster has a default StorageClass (`local-path`) and a retain-policy StorageClass (`local-path-retain`), unblocking Phase 1 (metrics core).

**Architecture:** New `storage` namespace with PSA `privileged`. Flux pulls a community Helm chart from the containeroo HTTPS Helm repo (`https://charts.containeroo.ch`) via a `HelmRepository` source. (The OCI equivalent does not exist for this chart — verified in Task 1.) Chart-bundled StorageClass is disabled; two SCs are committed as explicit manifests. PV data lives under `/var/local-path-provisioner/` inside Talos' writable ephemeral partition (top-level under `/var`, **not** under `/var/mnt/` — Talos reserves that subtree for `machineconfig` user volumes) — no Talos `machineconfig` patch required.

**Tech Stack:** Flux v2 (kustomize-controller + helm-controller + source-controller), Kustomize, Talos Linux, `local-path-provisioner` chart `0.0.36` from `https://charts.containeroo.ch`.

---

## File map

All new files live under `kubernetes/apps/storage/`. No existing files are modified.

| File | Role |
|---|---|
| `kubernetes/apps/storage/kustomization.yaml` | Namespace-level Kustomize; emits namespace + app Kustomization CR |
| `kubernetes/apps/storage/namespace.yaml` | `storage` namespace with PSA `privileged` labels |
| `kubernetes/apps/storage/local-path-provisioner/ks.yaml` | Flux `Kustomization` CR pointing at `app/` |
| `kubernetes/apps/storage/local-path-provisioner/app/kustomization.yaml` | App-level Kustomize; lists the 5 manifests below |
| `kubernetes/apps/storage/local-path-provisioner/app/helmrepository.yaml` | Flux `HelmRepository` pointing at containeroo's classic Helm repo |
| `kubernetes/apps/storage/local-path-provisioner/app/helmrelease.yaml` | Flux `HelmRelease` referencing the HelmRepository, with values |
| `kubernetes/apps/storage/local-path-provisioner/app/storageclass-default.yaml` | `local-path` SC (default, reclaim Delete) |
| `kubernetes/apps/storage/local-path-provisioner/app/storageclass-retain.yaml` | `local-path-retain` SC (reclaim Retain) |

Zero changes to `kubernetes/flux/cluster/ks.yaml` — Flux discovers new namespaces under `./kubernetes/apps` automatically (confirmed by existing namespaces that are not registered anywhere else).

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
  Expected: no errors; existing Kustomizations `Ready=True`.

- [ ] **Step 3: Confirm no existing StorageClass.**

  Run:
  ```bash
  kubectl get sc
  ```
  Expected: `No resources found` (if this prints a list, stop — review spec assumptions before proceeding).

- [ ] **Step 4: Confirm the chart URL is reachable.**

  Run:
  ```bash
  helm show chart oci://ghcr.io/containeroo/charts/local-path-provisioner --version 0.0.36 | head -20
  ```
  Expected: chart metadata printed (name, version, appVersion). If this fails with `unauthorized` or `not found`:
  - Fallback A: check a later tag — `helm search repo` is not possible for OCI, so browse `https://github.com/containeroo/charts/releases?q=local-path-provisioner` and use the newest `local-path-provisioner-<ver>` tag (version string without the prefix).
  - Fallback B: switch to raw manifests — out of scope for this plan; stop and flag.

- [ ] **Step 5: Confirm `kustomize` CLI is present.**

  Run:
  ```bash
  kustomize version
  ```
  Expected: version printed (managed by `mise`; installed via `mise install`).

- [ ] **Step 6: Confirm disk headroom on both nodes.**

  `talosctl` has no `df` subcommand; use the kubelet stats API instead:
  ```bash
  for n in ctrl01 work01; do
    kubectl get --raw "/api/v1/nodes/$n/proxy/stats/summary" \
      | jq -r --arg n "$n" '"\($n): \(.node.fs.availableBytes/1024/1024/1024 | floor) GiB free"'
  done
  ```
  Expected: >80 GiB free on each node. Record the values — Phase 1 needs ≥60 GiB for observability PVCs.

---

## Task 2: Scaffold the `storage` namespace

**Files:**
- Create: `kubernetes/apps/storage/kustomization.yaml`
- Create: `kubernetes/apps/storage/namespace.yaml`

- [ ] **Step 1: Create the namespace manifest.**

  Write `kubernetes/apps/storage/namespace.yaml`:
  ```yaml
  ---
  apiVersion: v1
  kind: Namespace
  metadata:
    name: storage
    annotations:
      kustomize.toolkit.fluxcd.io/prune: disabled
    labels:
      pod-security.kubernetes.io/enforce: privileged
      pod-security.kubernetes.io/audit: privileged
      pod-security.kubernetes.io/warn: privileged
  ```

  Notes:
  - The `prune: disabled` annotation matches every other namespace in this repo (`cert-manager/namespace.yaml`, etc.) — prevents Flux from deleting the namespace on accidental manifest removal.
  - The `privileged` PSA labels are required because `local-path-provisioner`'s helper Pod mounts a `hostPath`, which is forbidden under the `baseline` profile Talos applies.

- [ ] **Step 2: Create the namespace-level Kustomize.**

  Write `kubernetes/apps/storage/kustomization.yaml`:
  ```yaml
  ---
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  namespace: storage

  components:
    - ../../components/sops

  resources:
    - ./namespace.yaml
    - ./local-path-provisioner/ks.yaml
  ```

  Notes:
  - Matches the shape of `kubernetes/apps/cert-manager/kustomization.yaml`.
  - The `sops` component is harmless here (no secrets) but keeps the pattern uniform with every other namespace.
  - The `./local-path-provisioner/ks.yaml` entry is a forward-reference; the file is created in Task 3. `kustomize build` will fail at this point — that's expected until Task 3 completes.

- [ ] **Step 3: Commit.**

  ```bash
  git add kubernetes/apps/storage/namespace.yaml kubernetes/apps/storage/kustomization.yaml
  git commit -m "feat(storage): scaffold storage namespace with PSA privileged"
  ```

---

## Task 3: Add the Flux `Kustomization` CR for local-path-provisioner

**Files:**
- Create: `kubernetes/apps/storage/local-path-provisioner/ks.yaml`

- [ ] **Step 1: Write the Flux Kustomization CR.**

  Write `kubernetes/apps/storage/local-path-provisioner/ks.yaml`:
  ```yaml
  ---
  apiVersion: kustomize.toolkit.fluxcd.io/v1
  kind: Kustomization
  metadata:
    name: local-path-provisioner
  spec:
    healthChecks:
      - apiVersion: helm.toolkit.fluxcd.io/v2
        kind: HelmRelease
        name: local-path-provisioner
        namespace: storage
    interval: 1h
    path: ./kubernetes/apps/storage/local-path-provisioner/app
    postBuild:
      substituteFrom:
        - name: cluster-secrets
          kind: Secret
    prune: true
    sourceRef:
      kind: GitRepository
      name: flux-system
      namespace: flux-system
    targetNamespace: storage
    wait: true
  ```

  Notes:
  - `wait: true` ensures downstream Kustomizations (future Phase 1 workloads) block until this one reports Ready — matters because Prometheus PVCs depend on the StorageClass existing.
  - `substituteFrom: cluster-secrets` is boilerplate — no `${VAR}` substitutions are used in this app today, but including it keeps the manifest identical to existing apps so future `${SECRET_DOMAIN}`-style additions "just work".
  - `healthChecks` only asserts the HelmRelease is Ready — the StorageClasses are cluster-scoped and don't have a `.status.conditions` we could assert on here.

- [ ] **Step 2: Validate the YAML parses.**

  ```bash
  yq '.' kubernetes/apps/storage/local-path-provisioner/ks.yaml >/dev/null
  ```
  Expected: no output, exit 0.

- [ ] **Step 3: Commit.**

  ```bash
  git add kubernetes/apps/storage/local-path-provisioner/ks.yaml
  git commit -m "feat(storage): add local-path-provisioner Flux Kustomization"
  ```

---

## Task 4: Add the chart source (HelmRepository)

**Files:**
- Create: `kubernetes/apps/storage/local-path-provisioner/app/helmrepository.yaml`

- [ ] **Step 1: Write the HelmRepository.**

  Write `kubernetes/apps/storage/local-path-provisioner/app/helmrepository.yaml`:
  ```yaml
  ---
  apiVersion: source.toolkit.fluxcd.io/v1
  kind: HelmRepository
  metadata:
    name: containeroo
  spec:
    interval: 30m
    url: https://charts.containeroo.ch
    type: default
  ```

  Notes:
  - Named `containeroo` (the publisher) rather than `local-path-provisioner` so the same `HelmRepository` can be reused if we adopt another containeroo chart later.
  - `type: default` = classic HTTPS Helm repo (as opposed to `oci`). `containeroo` does not publish an OCI chart — verified in Task 1 Step 4.
  - No auth needed; the repo is public.

- [ ] **Step 2: Commit.**

  ```bash
  git add kubernetes/apps/storage/local-path-provisioner/app/helmrepository.yaml
  git commit -m "feat(storage): add containeroo HelmRepository source"
  ```

---

## Task 5: Add the HelmRelease

**Files:**
- Create: `kubernetes/apps/storage/local-path-provisioner/app/helmrelease.yaml`

- [ ] **Step 1: Write the HelmRelease.**

  Write `kubernetes/apps/storage/local-path-provisioner/app/helmrelease.yaml`:
  ```yaml
  ---
  apiVersion: helm.toolkit.fluxcd.io/v2
  kind: HelmRelease
  metadata:
    name: local-path-provisioner
  spec:
    interval: 1h
    chart:
      spec:
        chart: local-path-provisioner
        version: 0.0.36
        sourceRef:
          kind: HelmRepository
          name: containeroo
    values:
      storageClass:
        create: false
        provisionerName: rancher.io/local-path
      configmap:
        setup: |-
          #!/bin/sh
          set -eu
          mkdir -m 0777 -p "$VOL_DIR"
        teardown: |-
          #!/bin/sh
          set -eu
          rm -rf "$VOL_DIR"
        helperPod:
          name: helper-pod
          priorityClassName: system-node-critical
          tolerations:
            - operator: Exists
      nodePathMap:
        - node: DEFAULT_PATH_FOR_NON_LISTED_NODES
          paths:
            - /var/local-path-provisioner
      resources:
        requests:
          cpu: 10m
          memory: 32Mi
        limits:
          memory: 128Mi
  ```

  Notes:
  - `storageClass.create: false` — chart does not emit its own `StorageClass`; we ship explicit ones in Task 6.
  - `storageClass.provisionerName: rancher.io/local-path` — **required**. The chart auto-generates a name (`cluster.local/local-path-provisioner`) if unset; that name won't match the SC manifests in Task 6 (which use `provisioner: rancher.io/local-path`), and every PVC will sit Pending forever because the provisioner ignores PVCs that don't reference its own name.
  - `nodePathMap` with `DEFAULT_PATH_FOR_NON_LISTED_NODES` applies the single hostpath to every node. This is a special value defined by local-path-provisioner. Note `nodePathMap` is at the **root** of the values block, not nested under `configmap` — the chart reads it from there.
  - Path is `/var/local-path-provisioner` (top-level under `/var`). **Do not** use `/var/mnt/...` — Talos reserves that subtree for user volumes declared in `machineconfig` and the overlay rejects runtime `mkdir` even though `/var` is RW.
  - `configmap.helperPod` is a **typed object** in this chart (`name`, `priorityClassName`, `tolerations`, `annotations`, `namespaceOverride`), not a raw Pod manifest YAML string. The chart constructs the helper Pod itself from these fields. `priorityClassName: system-node-critical` lets it preempt ordinary workloads during PV provisioning; `tolerations: [{operator: Exists}]` lets it run on any node regardless of taints.
  - `resources` are conservative — provisioner itself is tiny.
  - The parent Flux Kustomization's global patches (see `kubernetes/flux/cluster/ks.yaml`) already inject `install.crds: CreateReplace`, `rollback.cleanupOnFail`, and `upgrade.remediation.retries` — no need to repeat here.
  - `chart.spec.sourceRef` (namespace omitted) implicitly references the `HelmRepository` in the same namespace as this HelmRelease (`storage`).

- [ ] **Step 2: Commit.**

  ```bash
  git add kubernetes/apps/storage/local-path-provisioner/app/helmrelease.yaml
  git commit -m "feat(storage): add local-path-provisioner HelmRelease"
  ```

---

## Task 6: Add the StorageClass manifests

**Files:**
- Create: `kubernetes/apps/storage/local-path-provisioner/app/storageclass-default.yaml`
- Create: `kubernetes/apps/storage/local-path-provisioner/app/storageclass-retain.yaml`

- [ ] **Step 1: Write the default StorageClass.**

  Write `kubernetes/apps/storage/local-path-provisioner/app/storageclass-default.yaml`:
  ```yaml
  ---
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

- [ ] **Step 2: Write the retain StorageClass.**

  Write `kubernetes/apps/storage/local-path-provisioner/app/storageclass-retain.yaml`:
  ```yaml
  ---
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: local-path-retain
  provisioner: rancher.io/local-path
  volumeBindingMode: WaitForFirstConsumer
  reclaimPolicy: Retain
  allowVolumeExpansion: false
  ```

- [ ] **Step 3: Commit.**

  ```bash
  git add kubernetes/apps/storage/local-path-provisioner/app/storageclass-default.yaml \
          kubernetes/apps/storage/local-path-provisioner/app/storageclass-retain.yaml
  git commit -m "feat(storage): add local-path default and retain StorageClasses"
  ```

---

## Task 7: Wire up the app-level Kustomize

**Files:**
- Create: `kubernetes/apps/storage/local-path-provisioner/app/kustomization.yaml`

- [ ] **Step 1: Write the app Kustomize.**

  Write `kubernetes/apps/storage/local-path-provisioner/app/kustomization.yaml`:
  ```yaml
  ---
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
    - ./helmrepository.yaml
    - ./helmrelease.yaml
    - ./storageclass-default.yaml
    - ./storageclass-retain.yaml
  ```

- [ ] **Step 2: Validate the whole namespace builds.**

  ```bash
  kustomize build kubernetes/apps/storage > /tmp/storage-rendered.yaml
  ```
  Expected: exit 0, no stderr. Inspect `/tmp/storage-rendered.yaml` and confirm it contains:
  - 1 `Namespace/storage`
  - 1 `Kustomization` (kustomize.toolkit.fluxcd.io) named `local-path-provisioner`

  Then validate the app directory in isolation:
  ```bash
  kustomize build kubernetes/apps/storage/local-path-provisioner/app > /tmp/lpp-rendered.yaml
  ```
  Expected: `/tmp/lpp-rendered.yaml` contains:
  - 1 `HelmRelease/local-path-provisioner`
  - 1 `HelmRepository/containeroo`
  - 2 `StorageClass` (`local-path`, `local-path-retain`)

- [ ] **Step 3: Commit.**

  ```bash
  git add kubernetes/apps/storage/local-path-provisioner/app/kustomization.yaml
  git commit -m "feat(storage): wire up local-path-provisioner app kustomization"
  ```

---

## Task 8: Deploy via Flux

**Files:** (none)

- [ ] **Step 1: Push to main.**

  ```bash
  git push origin main
  ```

- [ ] **Step 2: Trigger reconciliation (optional, saves waiting up to 1h).**

  ```bash
  flux reconcile source git flux-system
  flux reconcile kustomization cluster-apps
  ```

- [ ] **Step 3: Watch the new Kustomization reach Ready.**

  ```bash
  flux get ks local-path-provisioner -A --watch
  ```
  Expected (within ~2 minutes): `Ready=True`, message `Applied revision: main@sha1:<sha>`.

  If stuck `Not Ready`:
  - Check: `flux logs --kind=Kustomization --name=local-path-provisioner --namespace=flux-system`
  - Most likely cause: OCI tag typo or PSA label missing. Re-check Task 2 Step 1 and Task 4 Step 1.

- [ ] **Step 4: Watch the HelmRelease reach Ready.**

  ```bash
  flux get hr local-path-provisioner -A --watch
  ```
  Expected: `Ready=True`, `Status: Helm install succeeded for release storage/local-path-provisioner`.

---

## Task 9: Post-deploy verification

**Files:** (none)

- [ ] **Step 1: Confirm StorageClasses exist and default is set.**

  ```bash
  kubectl get sc
  ```
  Expected output (order may vary):
  ```
  NAME                PROVISIONER              RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION
  local-path (default) rancher.io/local-path   Delete          WaitForFirstConsumer   false
  local-path-retain   rancher.io/local-path    Retain          WaitForFirstConsumer   false
  ```

- [ ] **Step 2: Confirm the provisioner pod is Running.**

  ```bash
  kubectl -n storage get pods
  ```
  Expected: one `local-path-provisioner-*` Pod in `Running` state, `1/1 Ready`.

- [ ] **Step 3: Confirm ConfigMap was generated.**

  ```bash
  kubectl -n storage get cm local-path-config -o yaml | yq '.data.config\.json'
  ```
  Expected: JSON with `nodePathMap` containing `/var/local-path-provisioner`.

---

## Task 10: Smoke test — default StorageClass (`Delete` policy)

**Files:** (none, ephemeral test resources only)

- [ ] **Step 1: Create a test PVC.**

  ```bash
  kubectl apply -f - <<'EOF'
  ---
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: smoke-default
    namespace: default
  spec:
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 1Gi
    storageClassName: local-path
  EOF
  ```

- [ ] **Step 2: Confirm PVC is Pending (WaitForFirstConsumer).**

  ```bash
  kubectl -n default get pvc smoke-default
  ```
  Expected: `STATUS=Pending`, `VOLUME` empty. The `WaitForFirstConsumer` binding mode is correct behavior — no PV is created until a Pod consumes the PVC.

- [ ] **Step 3: Create a consuming Pod.**

  ```bash
  kubectl apply -f - <<'EOF'
  ---
  apiVersion: v1
  kind: Pod
  metadata:
    name: smoke-default
    namespace: default
  spec:
    restartPolicy: Never
    containers:
      - name: w
        image: busybox:1.36
        command: ["sh", "-c", "echo hello > /data/hello && sleep 30"]
        volumeMounts:
          - name: data
            mountPath: /data
    volumes:
      - name: data
        persistentVolumeClaim:
          claimName: smoke-default
  EOF
  ```

- [ ] **Step 4: Confirm PVC is Bound and PV exists.**

  ```bash
  kubectl -n default get pvc smoke-default
  kubectl get pv | grep smoke-default
  ```
  Expected: PVC `STATUS=Bound` within ~15s; PV `STATUS=Bound`, `CLAIM=default/smoke-default`.

- [ ] **Step 5: Verify data on node disk via a debug Pod.**

  Identify the node the Pod landed on:
  ```bash
  NODE=$(kubectl -n default get pod smoke-default -o jsonpath='{.spec.nodeName}')
  echo "Pod scheduled on $NODE"
  ```

  Launch a short-lived debug Pod in the `storage` namespace (PSA `privileged` — required for hostPath) and list the provisioner directory:
  ```bash
  kubectl -n storage run fs-inspect --rm -i --restart=Never \
    --image=busybox:1.36 \
    --overrides='{"spec":{"nodeName":"'"$NODE"'","containers":[{"name":"c","image":"busybox:1.36","stdin":true,"tty":true,"command":["sh","-c","ls -l /host && cat /host/*/hello 2>/dev/null || true"],"volumeMounts":[{"name":"h","mountPath":"/host"}]}],"volumes":[{"name":"h","hostPath":{"path":"/var/local-path-provisioner","type":"Directory"}}]}}' \
    -- sh
  ```
  Expected output: one `pvc-<uuid>_default_smoke-default` directory, and the `cat` prints `hello`.

- [ ] **Step 6: Delete the Pod + PVC, confirm directory is removed.**

  ```bash
  kubectl -n default delete pod smoke-default --wait=true
  kubectl -n default delete pvc smoke-default --wait=true
  ```

  Re-run the debug Pod to confirm cleanup:
  ```bash
  kubectl -n storage run fs-inspect --rm -i --restart=Never \
    --image=busybox:1.36 \
    --overrides='{"spec":{"nodeName":"'"$NODE"'","containers":[{"name":"c","image":"busybox:1.36","stdin":true,"tty":true,"command":["sh","-c","ls /host"],"volumeMounts":[{"name":"h","mountPath":"/host"}]}],"volumes":[{"name":"h","hostPath":{"path":"/var/local-path-provisioner","type":"DirectoryOrCreate"}}]}}' \
    -- sh
  ```
  Expected: directory is empty (or contains only directories from other PVCs, if any).

---

## Task 11: Smoke test — retain StorageClass (`Retain` policy)

**Files:** (none)

- [ ] **Step 1: Create PVC + Pod on `local-path-retain`.**

  ```bash
  kubectl apply -f - <<'EOF'
  ---
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: smoke-retain
    namespace: default
  spec:
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 1Gi
    storageClassName: local-path-retain
  ---
  apiVersion: v1
  kind: Pod
  metadata:
    name: smoke-retain
    namespace: default
  spec:
    restartPolicy: Never
    containers:
      - name: w
        image: busybox:1.36
        command: ["sh", "-c", "echo retained > /data/marker && sleep 30"]
        volumeMounts:
          - name: data
            mountPath: /data
    volumes:
      - name: data
        persistentVolumeClaim:
          claimName: smoke-retain
  EOF
  ```

- [ ] **Step 2: Wait for Bound, capture PV name + node.**

  ```bash
  kubectl -n default wait --for=jsonpath='{.status.phase}'=Bound pvc/smoke-retain --timeout=60s
  PV=$(kubectl -n default get pvc smoke-retain -o jsonpath='{.spec.volumeName}')
  NODE=$(kubectl -n default get pod smoke-retain -o jsonpath='{.spec.nodeName}')
  NODE_IP=$(kubectl get node "$NODE" -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')
  echo "PV=$PV NODE=$NODE IP=$NODE_IP"
  ```

- [ ] **Step 3: Delete Pod + PVC.**

  ```bash
  kubectl -n default delete pod smoke-retain --wait=true
  kubectl -n default delete pvc smoke-retain --wait=true
  ```

- [ ] **Step 4: Confirm PV remains in `Released` phase with data intact.**

  ```bash
  kubectl get pv "$PV"
  ```
  Expected: `STATUS=Released`, `RECLAIM POLICY=Retain`.

  Verify the on-disk marker via a debug Pod:
  ```bash
  kubectl -n storage run fs-inspect --rm -i --restart=Never \
    --image=busybox:1.36 \
    --overrides='{"spec":{"nodeName":"'"$NODE"'","containers":[{"name":"c","image":"busybox:1.36","stdin":true,"tty":true,"command":["sh","-c","find /host -name marker -exec cat {} \\;"],"volumeMounts":[{"name":"h","mountPath":"/host"}]}],"volumes":[{"name":"h","hostPath":{"path":"/var/local-path-provisioner","type":"Directory"}}]}}' \
    -- sh
  ```
  Expected: prints `retained`.

- [ ] **Step 5: Clean up the orphaned PV and on-disk directory.**

  The directory is *not* removed automatically under `Retain` — that is the whole point. In production use we leave it; for this smoke test, clean up explicitly:

  ```bash
  kubectl delete pv "$PV"
  kubectl -n storage run fs-cleanup --rm -i --restart=Never \
    --image=busybox:1.36 \
    --overrides='{"spec":{"nodeName":"'"$NODE"'","containers":[{"name":"c","image":"busybox:1.36","stdin":true,"tty":true,"command":["sh","-c","rm -rf /host/*; ls /host"],"volumeMounts":[{"name":"h","mountPath":"/host"}]}],"volumes":[{"name":"h","hostPath":{"path":"/var/local-path-provisioner","type":"Directory"}}]}}' \
    -- sh
  ```
  Expected: the final `ls /host` prints nothing — directory is empty.

---

## Task 12: Reboot persistence test

**Files:** (none)

- [ ] **Step 1: Create a long-lived Pod + PVC that writes a marker.**

  ```bash
  kubectl apply -f - <<'EOF'
  ---
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: reboot-test
    namespace: default
  spec:
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 1Gi
    storageClassName: local-path
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: reboot-test
    namespace: default
  spec:
    replicas: 1
    selector: { matchLabels: { app: reboot-test } }
    template:
      metadata: { labels: { app: reboot-test } }
      spec:
        nodeSelector:
          kubernetes.io/hostname: work01
        containers:
          - name: w
            image: busybox:1.36
            command: ["sh", "-c", "sleep infinity"]
            volumeMounts:
              - name: data
                mountPath: /data
        volumes:
          - name: data
            persistentVolumeClaim:
              claimName: reboot-test
  EOF
  ```

  Notes:
  - `nodeSelector: work01` pins the Pod so we know which node to reboot.
  - **Important:** the container command does **not** write `/data/marker` itself. Step 2 writes the marker via `kubectl exec`. If the container wrote the marker on every start (e.g. `echo persisted > /data/marker; sleep infinity`), the post-reboot replacement Pod would re-create the file and the test would pass even with `emptyDir` — defeating the purpose.

- [ ] **Step 2: Wait for Pod Ready, then write a unique marker.**

  ```bash
  kubectl -n default wait deploy/reboot-test --for=condition=Available --timeout=120s
  MARKER="persisted-$(date +%s)"
  kubectl -n default exec deploy/reboot-test -- sh -c "echo $MARKER > /data/marker && cat /data/marker"
  echo "$MARKER" > /tmp/reboot-test-marker
  ```
  The unique value (Unix timestamp) makes it impossible for the marker to be coincidentally re-created post-reboot.

- [ ] **Step 3: Reboot the worker.**

  ```bash
  talosctl -n 192.168.1.31 reboot --wait=false
  ```
  `--wait=false` returns immediately so we control the wait loop ourselves.

- [ ] **Step 4: Wait for the node to leave Ready, then return.**

  ```bash
  echo "waiting for work01 to go NotReady..."
  until ! kubectl get node work01 -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' 2>/dev/null | grep -q True; do sleep 3; done
  echo "waiting for work01 to become Ready again..."
  kubectl wait --for=condition=Ready node/work01 --timeout=300s
  kubectl -n default wait deploy/reboot-test --for=condition=Available --timeout=180s
  ```
  The first `until` loop is essential — if you skip it and call `kubectl wait Ready` straight away, it returns immediately because the node is still Ready in the API view (kubelet hasn't disconnected yet) and the rest of the test races the reboot.

- [ ] **Step 5: Confirm the marker survived.**

  ```bash
  EXPECTED=$(cat /tmp/reboot-test-marker)
  GOT=$(kubectl -n default exec deploy/reboot-test -- cat /data/marker)
  echo "expected=$EXPECTED  got=$GOT"
  test "$EXPECTED" = "$GOT" && echo OK || { echo FAIL; exit 1; }
  ```
  Expected: marker file contents match the value written in Step 2 — proves the per-PVC directory at `/var/local-path-provisioner/pvc-<uuid>_default_reboot-test/` survived the cold boot.

- [ ] **Step 6: Clean up.**

  ```bash
  kubectl -n default delete deploy reboot-test
  kubectl -n default delete pvc reboot-test
  rm -f /tmp/reboot-test-marker
  ```

---

## Task 13: Capacity gate check

**Files:** (none)

- [ ] **Step 1: Record free space on `/var` on both nodes.**

  ```bash
  for n in ctrl01 work01; do
    kubectl get --raw "/api/v1/nodes/$n/proxy/stats/summary" \
      | jq -r --arg n "$n" '"\($n): \(.node.fs.availableBytes/1024/1024/1024 | floor) GiB free"'
  done
  ```
  Expected: both nodes report at least 60 GiB free. If less, stop — Phase 1 deployment will risk disk pressure. Options before proceeding:
  - Reduce planned Prometheus retention (see spec §Capacity table)
  - Add disks / nodes
  - Prune unused container images: `talosctl -n <ip> service containerd restart` (images re-pull on demand)

- [ ] **Step 2: Append the measurements to the spec header.**

  Edit `docs/superpowers/specs/2026-04-19-phase-0-storage-design.md` and change:
  ```markdown
  **Status:** Draft
  ```
  to:
  ```markdown
  **Status:** Implemented 2026-MM-DD — ctrl01=<X>GiB free, work01=<Y>GiB free
  ```

- [ ] **Step 3: Commit the spec update.**

  ```bash
  git add docs/superpowers/specs/2026-04-19-phase-0-storage-design.md
  git commit -m "docs(specs): mark phase 0 storage as implemented"
  git push origin main
  ```

---

## Done condition

- `kubectl get sc` shows `local-path (default)` and `local-path-retain`.
- `flux get hr local-path-provisioner -n storage` reports `Ready=True`.
- All 4 smoke tests in Tasks 10–12 pass.
- Capacity gate (Task 13) records ≥60 GiB free on each node.
- Spec header updated to `Implemented` with measured capacity values.

Next step after done: start Phase 1 — write a new spec for the metrics core (kube-prometheus-stack + standalone Grafana).
