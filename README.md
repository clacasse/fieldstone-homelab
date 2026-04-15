# GPU Workstation Homelab

Ephemeral, reproducible-from-git GPU workstation for running local LLMs on Kubernetes. Drop in any x86_64 Ubuntu box with an NVIDIA GPU, clone, run one script — end up with Ollama on k3s managed by Argo CD.

## Design goals

1. **Ephemeral OS** — bare-metal Ubuntu is throwaway. Clone + one command = running Ollama again.
2. **Everything in git** — no hand-edits on the box. If it's not in this repo (or pullable at runtime), it doesn't exist.
3. **Minimum imperative, maximum declarative** — `bootstrap.sh` does just enough to get k3s + Argo running; Argo reconciles everything else.
4. **Model-agnostic** — no model names in any manifest. Pull/swap/run any model at runtime without redeploying.
5. **Hardware-agnostic** — no driver version, GPU model, or host-specific detail baked in. Works on any modern NVIDIA GPU supported by the current Ubuntu kernel.
6. **Fork-friendly** — repo URL is auto-detected from `git config` at bootstrap; forks work without find-replace.

## Requirements

- **Host**: x86_64 Ubuntu (tested on 25.10+; any release new enough for your GPU's driver should work)
- **GPU**: Any NVIDIA card the recommended Ubuntu driver supports (`ubuntu-drivers` chooses)
- **Network**: LAN with internet access for package + image downloads
- **Storage**: Enough free space for model weights you plan to pull (PVC is 200Gi by default)

## Usage

On a fresh Ubuntu host (any user with sudo):

```bash
git clone https://github.com/<your-fork>/gpu-workstation-homelab.git
cd gpu-workstation-homelab
./bootstrap.sh
```

First run installs the NVIDIA driver and asks you to reboot. After reboot, re-run `./bootstrap.sh`; it's idempotent and continues to k3s → Argo CD → Ollama.

When it finishes, Ollama is reachable on the host at NodePort `31434`. Pull any model:

```bash
./scripts/pull-model.sh llama3.3:70b              # from the box itself
./scripts/pull-model.sh gemma3:27b <host>:31434   # from another LAN machine
```

## What gets installed

| Layer | What | How |
|-------|------|-----|
| Base | apt upgrade, unattended-upgrades, utilities | Ansible `base` role |
| NVIDIA | driver (autodetect) + Container Toolkit | Ansible `nvidia` role |
| Cluster | k3s single node with NVIDIA as default containerd runtime | Ansible `k3s` role |
| GitOps | Argo CD + Applications for this cluster | Ansible `argocd` role |
| Workload | Ollama (Deployment + PVC + NodePort Service) | Argo CD → `clusters/apps/ollama/` |
| Device | nvidia-device-plugin daemonset (`nvidia.com/gpu` resource) | Argo CD → upstream Helm chart |

## Repo layout

```
.
├── README.md
├── bootstrap.sh                    # one-shot entrypoint on fresh Ubuntu
├── ansible/
│   ├── ansible.cfg
│   ├── inventory.ini               # localhost
│   ├── site.yml
│   └── roles/
│       ├── base/
│       ├── nvidia/
│       ├── k3s/
│       └── argocd/                 # renders Applications from templates
├── clusters/
│   └── apps/
│       └── ollama/                 # raw k8s manifests, no repo URL anywhere
└── scripts/
    └── pull-model.sh               # runtime convenience; not part of state
```

## Key decisions

| Decision | Choice | Reason |
|----------|--------|--------|
| GitOps tool | Argo CD | UI is useful for homelab; app-of-apps pattern fits single-cluster |
| Model storage | Persistent local-path PVC | Ephemeral = OS layer. Re-pulling large models on every rebuild is silly. |
| External access | LAN only (NodePort) | No public exposure by default |
| Secrets | None in v1 | Ollama has no auth; trusted LAN. Add Sealed Secrets if/when needed. |
| Model management | Runtime-only via API | No model names in repo. Truly model-agnostic. |
| Argo Application source | Rendered by Ansible at bootstrap | No repo URL committed to the repo; forks work unchanged. |

## Configuration

Most things have sensible defaults. Common overrides:

```bash
./bootstrap.sh -e timezone=America/Los_Angeles
```

Storage size, NodePort, and other knobs live in `clusters/apps/ollama/*.yaml` — edit, commit, Argo reconciles.

## Adding a new app

1. Create `clusters/apps/<name>/` with raw k8s manifests
2. Add an `Application` block to `ansible/roles/argocd/templates/root-app.yaml.j2`
3. Re-run `./bootstrap.sh` (idempotent) OR `kubectl apply` the rendered Application manually

## Non-goals

- Multi-node clusters
- Public internet exposure (add Tailscale/Cloudflare Tunnel separately if needed)
- Model pre-pulling / init containers (models managed at runtime)
- Backup of model weights (re-downloadable)
