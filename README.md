# GPU Workstation Homelab

Ephemeral, reproducible-from-git k3s cluster for a GPU workstation + worker nodes. Drop in any set of x86_64 Ubuntu boxes, one of them with an NVIDIA GPU, clone, run one script — end up with a 3-node k3s cluster running Ollama on the GPU node managed by Argo CD.

## Topology

```
┌─────────────────────────┐  ┌─────────────────────────┐  ┌──────────────────────────┐
│  control node           │  │  worker node            │  │  gpu node                │
│  (k3s server)           │  │  (k3s agent)            │  │  (k3s agent)             │
│  stable, always-on      │  │  stable, always-on      │  │  NVIDIA GPU; may reboot  │
│                         │  │                         │  │  labels: gpu=true        │
│                         │  │                         │  │  taints: gpu=true:NoSch. │
└────────────┬────────────┘  └────────────┬────────────┘  └────────────┬─────────────┘
             │                            │                            │
             └────────────────────────────┴────────────────────────────┘
                                    LAN (DHCP reservations)

                   AI workloads (Ollama) nodeSelector+tolerate gpu=true
                   → only schedule on GPU node
```

- **One server** (no HA). If it dies, PXE + re-bootstrap.
- **GPU node is tainted** so random workloads don't steal its resources.
- **Other workloads** (future) run on the two non-GPU nodes.

## Design goals

1. **Ephemeral OS** — nodes are throwaway; PXE + bootstrap recreates.
2. **Everything in git** — no hand-edits on any node.
3. **Minimum imperative, maximum declarative** — Ansible installs k3s + Argo; Argo reconciles apps.
4. **Model-agnostic** — no model names in any manifest.
5. **Hardware-agnostic** — NVIDIA driver autodetected; no driver version baked in.
6. **Fork-friendly** — repo URL auto-detected from `git config` at bootstrap.

## Requirements

- **Nodes**: 3× x86_64 Ubuntu hosts (25.10+ recommended for recent GPU support)
- **Network**: shared LAN, DHCP reservations on router (stable IPs)
- **GPU**: NVIDIA card on one node (any model supported by `ubuntu-drivers`)
- **SSH**: key-based access from your workstation to all 3 nodes with sudo
- **Workstation**: Ansible installed locally (`brew install ansible` / `apt install ansible`)

## Usage

**On your workstation (Mac/Linux):**

```bash
git clone https://github.com/<your-fork>/gpu-workstation-homelab.git
cd gpu-workstation-homelab

# Configure your nodes
cp ansible/inventory.ini.example ansible/inventory.ini
$EDITOR ansible/inventory.ini   # fill in real hostnames/IPs

# Run it
./bootstrap.sh
```

The playbook orchestrates:

1. `base` on all nodes — apt upgrade, utilities, unattended-upgrades
2. `nvidia` on GPU node — driver (autodetected) + Container Toolkit; auto-reboots if driver newly installed
3. `k3s-server` on control node — installs k3s, captures join token
4. `k3s-agent` on workers + GPU node — joins cluster; GPU node additionally gets `gpu=true` label, `gpu=true:NoSchedule` taint, and containerd NVIDIA runtime
5. `argocd` on control node — installs Argo CD, applies Applications for nvidia-device-plugin (Helm) and Ollama

Final output prints the Argo CD admin password. Port-forward from the control node and open the UI.

## Pulling models

Ollama is reachable at NodePort `31434` on any node (services span the cluster, but the pod runs on the GPU node).

```bash
./scripts/pull-model.sh llama3.3:70b                 # pulls from localhost
./scripts/pull-model.sh gemma3:27b <any-node>:31434  # from your workstation
```

## Repo layout

```
.
├── README.md
├── bootstrap.sh                      # runs from workstation; NOT on a node
├── ansible/
│   ├── ansible.cfg
│   ├── inventory.ini.example         # committed template
│   ├── inventory.ini                 # gitignored (site-specific)
│   ├── site.yml                      # multi-play orchestration
│   └── roles/
│       ├── base/
│       ├── nvidia/                   # GPU node only; auto-reboots
│       ├── k3s-server/
│       ├── k3s-agent/                # GPU variant adds label/taint/containerd
│       └── argocd/                   # renders Applications from templates
├── clusters/
│   └── apps/
│       └── ollama/                   # raw k8s manifests, nodeSelector + toleration
└── scripts/
    └── pull-model.sh                 # runtime convenience, not declarative state
```

## Key decisions

| Decision | Choice | Reason |
|----------|--------|--------|
| Cluster topology | 1 server + 2 agents, no HA | Simple; rebuild on failure |
| Control plane placement | Non-GPU worker node | Stable; GPU node can reboot freely |
| GPU scheduling | Node label + taint + toleration | AI workloads pin to GPU node; random workloads can't land there |
| GitOps tool | Argo CD | UI useful; first-class declarative Apps |
| Model storage | Persistent local-path PVC on GPU node | Ephemeral = OS layer; don't re-pull 17GB models |
| External access | LAN only (NodePort) | No public exposure by default |
| Secrets | None in v1 | Trusted LAN. Add Sealed Secrets later if needed. |
| Argo Application source | Rendered by Ansible at bootstrap | No repo URL committed; forks work unchanged |

## Adding a new app

1. Create `clusters/apps/<name>/` with raw k8s manifests
2. If it's an AI workload, add nodeSelector `gpu: "true"` + toleration for `gpu=true:NoSchedule`
3. Add an `Application` block to `ansible/roles/argocd/templates/root-app.yaml.j2`
4. Commit + re-run `./bootstrap.sh` (idempotent)

## Non-goals

- HA control plane
- Multi-GPU node
- Public internet exposure
- Model pre-pulling / init containers
- Backup of model weights
