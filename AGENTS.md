# AGENTS.md

This file provides guidance to coding agents (Claude Code, Codex, etc.) when working with code in this repository.

## What this repo is

A personal fork of [onedr0p/cluster-template](https://github.com/onedr0p/cluster-template) — a GitOps-managed Kubernetes cluster running on Talos Linux. Secrets are encrypted with SOPS+age, and the cluster is reconciled by Flux.

## General guidelines
- Do not hallucinate, avoid making things up.
- Ask when in doubt, request clarification if ambiguity is present.
- Always ask for permission to perform material changes or perform desctructive actions.
- Ask for permission before commit or push.
- Cluster changes go through Git/Flux.
- Never read, parse, or decrypt secrets: *.sops.*, age.key, .env, tokens, deploy keys, kubeconfig, talosconfig. Ask the user instead.
- Never commit, push, or make material or destructive changes without explicit permission.

## Tool management

All CLI tools are managed by [mise](https://mise.jdx.dev/). Install them with:

```sh
mise trust && pip install pipx && mise install
```

Key tools: `task`, `talhelper`, `talosctl`, `kubectl`, `flux`, `sops`, `age`, `helm`, `helmfile`, `kustomize`, `kubeconform`, `yq`, `jq`.

> **Note:** Tool versions are pinned in `.mise.toml`. Keep `kubectl` within one minor version of `kubernetesVersion` in `talos/talenv.yaml`.

## Common task commands

```sh
task                          # List all available tasks

# Cluster lifecycle
task bootstrap:talos          # Bootstrap Talos OS on nodes (generates secrets, applies config, bootstraps etcd)
task bootstrap:apps           # Deploy Flux and core apps into the running cluster

# Day-to-day operations
task reconcile                # Force Flux to pull and apply changes from Git

# Talos node management
task talos:generate-config    # Regenerate talos/clusterconfig/ from talconfig.yaml
task talos:apply-node IP=<x>  # Apply Talos config to a specific node (MODE=auto|no-reboot|reboot)
task talos:upgrade-node IP=<x> # Upgrade Talos on a single node
task talos:upgrade-k8s        # Upgrade Kubernetes version cluster-wide

task talos:reset              # Wipe nodes back to maintenance mode (DESTRUCTIVE)
```

## Environment variables (set by Taskfile.yaml and .mise.toml)

- `KUBECONFIG` → `./kubeconfig`
- `TALOSCONFIG` → `./talos/clusterconfig/talosconfig`
- `SOPS_AGE_KEY_FILE` → `./age.key`

These are exported by task, and by mise when it is activated in the shell. Set them manually otherwise.

## Architecture

### Hardware guidance (from upstream)

Bare metal is strongly recommended over Proxmox/VMs. Enterprise NVMe or SATA SSDs are preferred; consumer NVMe can cause latency spikes, corruption, and fsync delays in multi-node setups. Any replicated storage (Rook-Ceph, Longhorn) must use **dedicated disks separate from control plane / etcd nodes**.

### Configuration flow

```
kubernetes/ ──► Git push to main ──► Flux ──► cluster
talos/talconfig.yaml + talenv.yaml + patches/ ──► task talos:generate-config ──► talos/clusterconfig/ ──► task talos:apply-node
```

Flux reconciles only `kubernetes/`. Changes under `talos/` must be applied to nodes with the `talos:*` tasks, and `bootstrap/` is only used by `task bootstrap:apps`.

### Secret management

SOPS rules (`.sops.yaml`):
- `talos/*.sops.yaml` — fully encrypted
- `bootstrap/**/*.sops.yaml` and `kubernetes/**/*.sops.yaml` — only `data`/`stringData` fields encrypted

The age key at `./age.key` is required for all encrypt/decrypt operations. Never commit it. Ensure `~/.config/sops/age/keys.txt` does **not** exist, as it interferes with the repo's age key.

### GitOps flow

Flux watches this Git repo and reconciles changes. `kubernetes/flux/cluster/ks.yaml` defines the root `cluster-apps` Kustomization, which reconciles `kubernetes/apps`. Changes pushed to `main` are automatically applied by Flux. Force a reconcile with `task reconcile`.

For push-triggered reconciliation (instead of polling), set up a GitHub webhook pointing to `https://flux-webhook.${domain}/hook/<id>`. Get the hook path with:

```sh
kubectl -n flux-system get receiver github-webhook --output=jsonpath='{.status.webhookPath}'
```

The token is in `github-push-token.txt`.

### Networking and DNS

Two Envoy gateways handle ingress:
- `envoy-external` — public internet access (also reachable on LAN with split DNS)
- `envoy-internal` — private LAN-only access

`k8s_gateway` provides internal DNS resolution. For it to work, your home DNS server must forward queries for the cluster domain to the `k8s-gateway` LoadBalancer IP (split-horizon DNS / conditional forwarding).

`external-dns` (the `cloudflare-dns` app in the `network` namespace) creates public Cloudflare DNS records for services using the `envoy-external` gateway.

### Directory layout

| Path | Purpose |
|---|---|
| `kubernetes/apps/` | HelmReleases and Kustomizations for all cluster apps |
| `kubernetes/flux/` | Flux root Kustomization |
| `kubernetes/components/` | Low-level K8s components (e.g. SOPS decryption setup) |
| `talos/talconfig.yaml` | Talos cluster definition (nodes, VIPs, extensions) |
| `talos/patches/` | Talos machine config patches (global / controller / worker / per-node) |
| `talos/clusterconfig/` | Generated machine configs and talosconfig (gitignored) |
| `bootstrap/helmfile.d/` | Helm releases applied during `task bootstrap:apps` |
| `.taskfiles/` | Task runner includes (bootstrap, talos subtasks) |
| `docs/superpowers/` | Specs and plans for past work |
| `openspec/` | OpenSpec config, specs and changes |

### Included components

Core: Cilium (CNI), CoreDNS, Flux + Flux Operator, cert-manager, Reloader, Spegel (registry mirror), metrics-server.
Networking: Envoy Gateway, external-dns, cloudflared (Cloudflare Tunnel), k8s_gateway.
Storage: local-path-provisioner (`local-path` default, `local-path-retain`); node-local, not replicated.
Observability: kube-prometheus-stack, plus scraping and dashboards for the home network (`home-network`).

### Adding a node

No re-bootstrap needed. Boot the node into maintenance mode, then:

```sh
# Gather disk and MAC info
talosctl get disks -n <ip> --insecure
talosctl get links -n <ip> --insecure
```

Add the node to `talos/talconfig.yaml`, then:

```sh
task talos:generate-config
task talos:apply-node IP=<new-node-ip>
```

Use an odd number of control plane nodes for etcd quorum.

### Debugging

```sh
# Check Flux sync status
flux get sources git -A && flux get ks -A && flux get hr -A

# Check pods / logs / events
kubectl -n <ns> get pods -o wide
kubectl -n <ns> logs <pod> -f
kubectl -n <ns> describe <resource> <name>
kubectl -n <ns> get events --sort-by='.metadata.creationTimestamp'
```

### Bootstrap caveats

- `task bootstrap:talos` takes 10+ minutes; errors like "couldn't get current server API group list" are normal during CNI bring-up.
- If interrupted with Ctrl+C, run `task talos:reset` before retrying.
- Resetting repeatedly in a short window may trigger DockerHub or Let's Encrypt rate limits.

### Dependency automation

Renovate (`.renovaterc.json5`) proposes updates for container images, Helm charts, GitHub Actions, and mise tools. Minor/patch updates to GitHub Actions (also digests) and mise tools auto-merge after 3 days; everything else has a 5-day minimum release age and is merged manually. Updates run on weekends. The kube-prometheus-stack chart and its bootstrap CRDs are grouped so they update together. Semantic commit messages are enforced (`feat()`, `fix()`, `ci()`, `chore()`). A "Dependency Dashboard" issue is created in the GitHub repo for an overview of pending updates.
