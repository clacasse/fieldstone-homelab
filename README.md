# k8s Cluster Homelab

Ephemeral, reproducible-from-git k3s cluster for a small fleet of Ubuntu boxes — optionally with one or more NVIDIA GPU nodes for AI workloads. Clone, edit an inventory, run two commands: you end up with a k3s cluster running Ollama on the GPU, managed by Argo CD, with clean `<app>.apps.localdomain` URLs via Traefik Ingress.

> **LAN-only by design.** Ollama's API has no authentication. Do NOT deploy this on a cloud VM, a box with a public IP, or any network you don't fully trust without adding your own auth layer (reverse proxy with basic auth, Tailscale ACLs, etc.).

## What you get

- **k3s** cluster: 1 server + N agents (no HA)
- **Argo CD** on the control node, reconciling from your fork via the app-of-apps pattern — accessed at `http://argocd.apps.localdomain`
- **Ollama** deployed to the GPU node with a persistent local-path PVC — accessed at `http://ollama.apps.localdomain`
- **NVIDIA device plugin** installed via Helm so pods can request `nvidia.com/gpu: 1`
- **Traefik Ingress** (shipped with k3s) fronted by one wildcard DNS record — adding apps never requires touching the router again
- A single Python CLI (`cluster_manager.py`) that drives the whole lifecycle

## Topology

```
┌─────────────────────────┐  ┌─────────────────────────┐  ┌──────────────────────────┐
│  control node           │  │  worker node            │  │  gpu node                │
│  (k3s server)           │  │  (k3s agent)            │  │  (k3s agent)             │
│  stable, always-on      │  │  stable, always-on      │  │  NVIDIA GPU; may reboot  │
│                         │  │                         │  │  labels: nvidia.com/gpu  │
│                         │  │                         │  │  taints: nvidia.com/gpu  │
└────────────┬────────────┘  └────────────┬────────────┘  └────────────┬─────────────┘
             │                            │                            │
             └────────────────────────────┴────────────────────────────┘
                                    LAN (router DNS, .localdomain)

                   *.apps.localdomain  →  control node IP  (wildcard A record)
                   AI workloads (Ollama) nodeSelector+tolerate nvidia.com/gpu=true
                   → only schedule on GPU node
```

- **One server, no HA.** If it dies, rebuild from git.
- **GPU node is tainted** so random workloads don't steal its resources.
- **Minimum useful cluster** is 1 control + 1 GPU; scale agents as you like.

## Requirements

### Workstation (where you run the CLI)
- macOS or Linux
- `ansible` (`brew install ansible` / `apt install ansible`)
- Python 3.10+ (CLI deps installed into a venv — see walkthrough below)
- SSH key already loaded in your agent, accepted as an authorized key on every node

### Nodes
- **x86_64 Ubuntu** — already installed and booted before you touch this repo. Ubuntu 25.10+ is recommended (newer kernels ship drivers for recent NICs like the Realtek RTL8126).
- **Same sudo password on every node** (the CLI prompts once with `--ask-become-pass`)
- **Router DNS registration** — your router must register DHCP client hostnames into DNS so you can SSH to `<name>.localdomain`. Ubiquiti's built-in DNS does this by default. If yours doesn't, edit `inventory.ini` to use raw IPs instead.
- **One node with an NVIDIA GPU** if you want Ollama (any card supported by `ubuntu-drivers --gpgpu`)

### Network (one-time wildcard DNS)

You'll add one wildcard DNS record to your router, pointing every `*.apps.localdomain` at the control node. After that, every future app just gets a free hostname — no per-app DNS work.

**UniFi / Ubiquiti**:
1. Network app → **Settings** → **Routing** → **DNS** → **DNS Entries**
2. **Create Entry**:
   - **Record Type:** `A`
   - **Hostname:** `*.apps.localdomain`
   - **IP Address:** the control node's IP (check with `ssh k3s-control.localdomain hostname -I`)
   - **TTL:** default is fine
3. Apply. Verify from your workstation:
   ```bash
   dig +short argocd.apps.localdomain
   # should print the control node IP
   ```

If you use a different router, make the equivalent wildcard A record. If you want a different domain, pass `--apps-domain <your.domain>` to `init-fork` below.

## First-run walkthrough

All commands run on your workstation.

```bash
# 1. Fork this repo on GitHub, then clone your fork.
git clone https://github.com/<you>/k8s-cluster-homelab.git
cd k8s-cluster-homelab

# 2. Create a Python venv and install CLI dependencies.
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# 3. One-time: rewrite the REPO_URL and APPS_DOMAIN placeholders in cluster
#    manifests. REPO_URL is auto-detected from `git remote`; apps domain
#    defaults to apps.localdomain. Commit + push — Argo reconciles from git.
./scripts/cluster_manager.py init-fork
git commit -am "Initialize fork"
git push

# 4. Configure your nodes. Fill in hostnames (or IPs) matching your network.
cp ansible/inventory.ini.example ansible/inventory.ini
$EDITOR ansible/inventory.ini

# 5. Pre-authorize each node's SSH host key (Ansible won't connect otherwise).
for host in $(awk '/^\[/{g=$0} g!~/vars|children/ && /\./ {print $1}' ansible/inventory.ini); do
    ssh-keyscan -H "$host" >> ~/.ssh/known_hosts
done

# 6. Prep every node (apt upgrade, hostname, NVIDIA driver on GPU nodes).
#    Run once per host. Idempotent — safe to re-run anytime.
./scripts/cluster_manager.py prep-node k3s-control.localdomain
./scripts/cluster_manager.py prep-node k3s-worker.localdomain
./scripts/cluster_manager.py prep-node k3s-gpu.localdomain

# 7. Bootstrap the cluster: k3s on every node + Argo CD on control.
./scripts/cluster_manager.py bootstrap

# 8. Verify.
./scripts/cluster_manager.py status
```

In future sessions, reactivate the venv before running the CLI: `source .venv/bin/activate`.

After `bootstrap` finishes, Argo CD reconciles Ollama, the NVIDIA device plugin, and its own Ingress on its own — typically in under a minute. Watch progress:

```bash
ssh k3s-control.localdomain sudo k3s kubectl -n argocd get applications
```

Then open **`http://argocd.apps.localdomain`** in a browser. The initial admin password:

```bash
ssh k3s-control.localdomain sudo k3s kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
```

Change it immediately after first login.

## The CLI

`scripts/cluster_manager.py` is the single entrypoint:

| Command | Purpose |
|---|---|
| `init-fork [URL] [--apps-domain D]` | Rewrite `REPO_URL` + `APPS_DOMAIN` placeholders in `clusters/**/*.yaml`. |
| `prep-node <host>` | Run `ansible/prep.yml` against one host. Apt upgrade, hostname, NVIDIA if in `[gpu]` group. |
| `bootstrap` | Run `ansible/cluster.yml` against the whole inventory. Installs k3s + Argo CD. |
| `pull-model <tag>` | Pull a model into the running Ollama server. Defaults to `ollama.apps.localdomain`. |
| `status` | `kubectl get nodes,pods -A` via SSH to the control node. |

Run `./scripts/cluster_manager.py --help` (or `<cmd> --help`) for full options.

### What `prep.yml` does
1. `base` on every targeted host — apt upgrade, utilities, unattended-upgrades, set hostname (then DHCP renew so the router registers `<name>.localdomain` in its DNS).
2. `nvidia` on hosts in `[gpu]` — install driver (autodetected via `ubuntu-drivers`) + NVIDIA Container Toolkit. Auto-reboots if a new driver was installed.

### What `cluster.yml` does
1. `k3s-server` on control — install k3s (pinned), capture join token.
2. `k3s-agent` on every agent — join the cluster. GPU nodes also get the `nvidia.com/gpu=true` label + `NoSchedule` taint and containerd NVIDIA runtime config.
3. `argocd` on control — install Argo CD (pinned), set `server.insecure=true` so it serves HTTP for Ingress, and apply the root Application that owns everything under `clusters/homelab/applications/children/`.

## Pulling models

Ollama is reachable at `http://ollama.apps.localdomain`:

```bash
./scripts/cluster_manager.py pull-model llama3.3:70b
# or against a different host:
./scripts/cluster_manager.py pull-model gemma3:27b --host ollama.apps.localdomain
```

## Adding a new app

Pure git workflow — no Ansible, no DNS changes:

1. Create `clusters/homelab/apps/<name>/` with raw Kubernetes manifests (Deployment, Service, optionally an Ingress for `<name>.apps.localdomain`).
2. For AI workloads, include `nodeSelector: nvidia.com/gpu: "true"` and the matching toleration.
3. Create `clusters/homelab/applications/children/<name>.yaml` — an Argo `Application` pointing at that path.
4. Commit + push. Argo picks it up automatically via `selfHeal: true`, Traefik routes the hostname, wildcard DNS does the rest.

## Adding a new node later

1. Install Ubuntu on the new machine (any method — this repo doesn't care how).
2. Add it to `ansible/inventory.ini` in the appropriate group.
3. Authorize its SSH host key: `ssh-keyscan -H <new-host> >> ~/.ssh/known_hosts`.
4. `./scripts/cluster_manager.py prep-node <new-host>`
5. `./scripts/cluster_manager.py bootstrap` — idempotent; only the new node actually changes.

## Repo layout

```
.
├── README.md
├── requirements.txt                    # Python deps for the CLI
├── scripts/
│   └── cluster_manager.py              # typer CLI
├── ansible/
│   ├── ansible.cfg
│   ├── inventory.ini.example           # committed template
│   ├── inventory.ini                   # gitignored (site-specific)
│   ├── prep.yml                        # per-node: base + nvidia
│   ├── cluster.yml                     # cluster-wide: k3s + argocd
│   ├── group_vars/all.yml              # pinned versions, apps_domain
│   └── roles/
│       ├── base/                       # apt, hostname, unattended-upgrades
│       ├── nvidia/                     # GPU only; auto-reboots
│       ├── k3s-server/
│       ├── k3s-agent/                  # GPU variant adds label/taint/containerd
│       └── argocd/                     # installs Argo CD, applies root Application
└── clusters/
    └── homelab/
        ├── applications/
        │   ├── root.yaml               # app-of-apps, applied by Ansible
        │   └── children/               # reconciled by root
        │       ├── ollama.yaml
        │       ├── nvidia-device-plugin.yaml
        │       └── argocd-ingress.yaml
        └── apps/                       # raw k8s manifests, reconciled by Argo
            ├── argocd-ingress/         # Ingress for the Argo CD UI itself
            └── ollama/
```

## Version pinning

All pinned in `ansible/group_vars/all.yml`:

| Component | Version |
|---|---|
| k3s | `v1.32.3+k3s1` |
| Argo CD | `v2.14.3` |
| Ollama | `0.6.5` |
| NVIDIA device plugin Helm chart | `0.17.0` |

Bump deliberately; re-run `./scripts/cluster_manager.py bootstrap` to apply.

## Known sharp edges

- **Unattended-upgrades on the GPU node can break `nvidia-smi`.** A kernel upgrade without a DKMS rebuild silently breaks GPU access. If it happens, re-run `prep-node <gpu-host>` — the `nvidia` role will reinstall drivers.
- **local-path PVCs don't survive OS reinstall.** Reinstalling the GPU node's OS means re-pulling models. By design — treat the OS as ephemeral.
- **Ingress is plain HTTP.** TLS is terminated nowhere on the LAN. Fine for a trusted network; add cert-manager + a CA if you need it.
- **Ollama has no auth.** LAN only. See top-of-readme warning.

## Key decisions

| Decision | Choice | Reason |
|---|---|---|
| Cluster topology | 1 server + N agents, no HA | Simple; rebuild on failure |
| Control plane placement | Non-GPU node | Stable; GPU node can reboot freely |
| GPU scheduling | Label + taint + toleration on `nvidia.com/gpu` | Matches NVIDIA GPU Operator convention |
| Bootstrap driver | Ansible behind a typer CLI | Idempotent roles, one operator entrypoint |
| Node addressability | Router DNS (`<name>.localdomain`) | No DHCP-reservation bookkeeping |
| App addressability | Wildcard DNS (`*.apps.localdomain`) + Traefik Ingress | One-time DNS setup; new apps add no manual steps |
| GitOps tool | Argo CD | UI is useful; app-of-apps pattern |
| App delivery | Committed Application manifests + `init-fork` | Fork-friendly; adding apps is pure git |
| Model storage | Persistent local-path PVC on GPU node | Ephemeral = OS; don't re-pull large models |
| External access | LAN only (HTTP Ingress) | No public exposure |
| Secrets | None in v1 | Trusted LAN. Add Sealed Secrets later if needed. |
| Model management | Runtime-only via API | No model names in repo |
| Version pinning | All in `group_vars/all.yml` | Reproducible re-runs |

## Non-goals

- HA control plane
- Public internet exposure
- TLS on the LAN
- Model pre-pulling / init containers
- Backup of model weights
