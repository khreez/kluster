# AGENTS.md

This file provides guidance to coding agents (Claude Code, Codex, etc.) when working with code in this repository.

## What this repo is

A personal fork of [onedr0p/cluster-template](https://github.com/onedr0p/cluster-template) — a GitOps-managed Kubernetes cluster running on Talos Linux. Configuration is rendered from Jinja2 templates via `makejinja`, secrets are encrypted with SOPS+age, and the cluster is reconciled by Flux.

## General guidelines
- Do not hallucinate, avoid making things up.
- Ask when in doubt, request clarification if ambiguity is present.
- Always ask for permission to perform material changes or perform desctructive actions.
- Ask for permission before commit or push.

## Tool management

All CLI tools are managed by [mise](https://mise.jdx.dev/). Install them with:

```sh
mise trust && pip install pipx && mise install
```

Key tools: `task`, `talhelper`, `talosctl`, `kubectl`, `flux`, `sops`, `age`, `makejinja`, `helm`, `helmfile`, `kustomize`, `kubeconform`, `yq`, `jq`.

> **Note:** The upstream template (`onedr0p/cluster-template`) may carry newer tool versions than this fork — particularly Helm (upstream is on 4.x), Talos, talhelper, and sops. Check `.mise.toml` versions against upstream before upgrading nodes or running tooling-sensitive tasks.

## Common task commands

```sh
task                          # List all available tasks

# Initial setup (run once, in order)
task init                     # Rename sample configs → cluster.yaml and nodes.yaml, generate age key
task configure                # Validate schemas, render templates, encrypt secrets, validate configs

# Cluster lifecycle
task bootstrap:talos          # Bootstrap Talos OS on nodes (generates secrets, applies config, bootstraps etcd)
task bootstrap:apps           # Deploy Flux and core apps into the running cluster

# Day-to-day operations
task reconcile                # Force Flux to pull and apply changes from Git
task template:debug           # kubectl get on common resources across all namespaces

# Talos node management
task talos:generate-config    # Regenerate talos/clusterconfig/ from talconfig.yaml
task talos:apply-node IP=<x>  # Apply Talos config to a specific node (MODE=auto|no-reboot|reboot)
task talos:upgrade-node IP=<x> # Upgrade Talos on a single node
task talos:upgrade-k8s        # Upgrade Kubernetes version cluster-wide

# Template/cleanup
task template:tidy            # Archive template files after initial setup is complete
task template:reset           # Nuke all rendered files (bootstrap/, kubernetes/, talos/, .sops.yaml)
task talos:reset              # Wipe nodes back to maintenance mode (DESTRUCTIVE)
```

## Prerequisites for `task configure`

`cloudflare-tunnel.json` must exist in the repo root before running `task configure`. Create it with:

```sh
cloudflared tunnel login
cloudflared tunnel create --credentials-file cloudflare-tunnel.json kubernetes
```

Also ensure `~/.config/sops/age/keys.txt` does **not** exist — it interferes with the repo's age key.

## Environment variables (set automatically by Taskfile.yaml)

- `KUBECONFIG` → `./kubeconfig`
- `TALOSCONFIG` → `./talos/clusterconfig/talosconfig`
- `SOPS_AGE_KEY_FILE` → `./age.key`

These are auto-exported by task. Set them manually if running commands outside of task.

## Architecture

### Hardware guidance (from upstream)

Bare metal is strongly recommended over Proxmox/VMs. Enterprise NVMe or SATA SSDs are preferred; consumer NVMe can cause latency spikes, corruption, and fsync delays in multi-node setups. Any replicated storage (Rook-Ceph, Longhorn) must use **dedicated disks separate from control plane / etcd nodes**.

### Configuration flow

```
cluster.yaml + nodes.yaml  ──► makejinja ──► kubernetes/ + talos/clusterconfig/
                                              (rendered from templates/)
```

`task configure` drives this: it validates `cluster.yaml`/`nodes.yaml` with CUE schemas, runs `makejinja`, encrypts all `*.sops.*` files, then validates rendered manifests with `kubeconform` and `talhelper`.

### cluster.yaml key fields

Required: `node_cidr`, `cluster_api_addr`, `cluster_dns_gateway_addr`, `cluster_gateway_addr`, `cloudflare_gateway_addr`, `cloudflare_domain`, `cloudflare_token`, `repository_name`.

Notable optional fields:
- `node_vlan_tag` — VLAN tagging for Talos nodes
- `node_default_gateway` — override default gateway (defaults to first IP in `node_cidr`)
- `cluster_api_tls_sans` — extra SANs for the Kube API cert (e.g. a hostname alias)
- `cluster_pod_cidr` / `cluster_svc_cidr` — default `10.42.0.0/16` / `10.43.0.0/16`
- `cilium_loadbalancer_mode` — `dsr` (default) or `snat`
- `cilium_bgp_router_addr` / `cilium_bgp_router_asn` / `cilium_bgp_node_asn` — BGP peering

### nodes.yaml key fields

Required per node: `name`, `address`, `controller`, `disk`, `mac_addr`, `schematic_id`.

Advanced optional fields:
- `mtu` — NIC MTU (default 1500)
- `secureboot` — UEFI SecureBoot (requires matching schematic)
- `encrypt_disk` — TPM-based disk encryption
- `kernel_modules` — list of modules for system extensions (e.g. `["nvidia", "nvidia_uvm", "nvidia_drm", "nvidia_modeset"]`, `["zfs"]`)

### Secret management

SOPS rules (`.sops.yaml`):
- `talos/*.sops.yaml` — fully encrypted
- `bootstrap/**/*.sops.yaml` and `kubernetes/**/*.sops.yaml` — only `data`/`stringData` fields encrypted

The age key at `./age.key` is required for all encrypt/decrypt operations. Never commit it.

### GitOps flow

Flux watches this Git repo and reconciles changes. `kubernetes/flux/` contains the root Kustomizations. Changes pushed to `main` are automatically applied by Flux. Force a reconcile with `task reconcile`.

For push-triggered reconciliation (instead of polling), set up a GitHub webhook pointing to `https://flux-webhook.${domain}/hook/<id>`. Get the hook path with:

```sh
kubectl -n flux-system get receiver github-webhook --output=jsonpath='{.status.webhookPath}'
```

The token is in `github-push-token.txt`.

### Networking and DNS

Two Envoy gateways handle ingress:
- `envoy-external` — public internet access (also reachable on LAN with split DNS)
- `envoy-internal` — private LAN-only access

`k8s_gateway` provides internal DNS resolution. For it to work, your home DNS server must forward queries for `${cloudflare_domain}` to `${cluster_dns_gateway_addr}` (split-horizon DNS / conditional forwarding).

`external-dns` (in the `network` namespace) creates public Cloudflare DNS records for services using the `envoy-external` gateway.

### Directory layout

| Path | Purpose |
|---|---|
| `cluster.yaml` / `nodes.yaml` | Primary user config (generated from `*.sample.yaml`) |
| `templates/` | Jinja2 templates that render kubernetes/ and talos/ |
| `kubernetes/apps/` | HelmReleases and Kustomizations for all cluster apps |
| `kubernetes/flux/` | Flux root Kustomizations and GitRepository config |
| `kubernetes/components/` | Low-level K8s components (e.g. SOPS decryption setup) |
| `talos/talconfig.yaml` | Talos cluster definition (nodes, VIPs, extensions) |
| `talos/patches/` | Talos machine config patches (global / controller / worker / per-node) |
| `talos/clusterconfig/` | Generated machine configs and talosconfig (gitignored) |
| `bootstrap/helmfile.d/` | Helm releases applied during `task bootstrap:apps` |
| `.taskfiles/` | Task runner includes (bootstrap, talos, template subtasks) |

### Included components

Core: Cilium (CNI), CoreDNS, Flux + Flux Operator, cert-manager, Reloader, Spegel (registry mirror), metrics-server.
Networking: Envoy Gateway, external-dns, cloudflared (Cloudflare Tunnel), k8s_gateway.

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

### Post-setup tidy

After initial setup is stable, run `task template:tidy` to archive `templates/`, `makejinja.toml`, `cluster.yaml`, `nodes.yaml`, and template-related GitHub Actions. This resolves Renovate "duplicate registry" warnings.

### Dependency automation

Renovate (`renovaterc.json5`) auto-updates container images, Helm charts, and GitHub Actions. It auto-merges minor/patch updates after 3 days. Updates run on weekends. Semantic commit messages are enforced (`feat()`, `fix()`, `ci()`, `chore()`). A "Dependency Dashboard" issue is created in the GitHub repo for an overview of pending updates.
