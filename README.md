# Pi Cluster Ansible - Air-Gapped CI/CD Platform

Ansible playbooks for deploying a fully air-gapped CI/CD platform on a Raspberry Pi cluster running upstream Kubernetes via kubeadm.

## Architecture

- **13 Nodes**: rpi-1 through rpi-12 + cm3588-plus
- **OS**: Debian 13 Trixie (ARM64)
- **Kubernetes**: kubeadm v1.32
- **Container Runtime**: containerd
- **CNI**: Flannel

## Components

| Component | Port | Description |
|-----------|------|-------------|
| Container Registry | 30500 | registry:2 for internal images |
| Gitea | 30300 (HTTP), 30022 (SSH) | Self-hosted Git server |
| Tekton Dashboard | 30900 | Pipeline visualization |
| Webhook Listener | 30800 | Gitea webhook receiver |

## Quick Start

```bash
# Full bootstrap (all playbooks in order)
ansible-playbook site.yml

# Or run individual playbooks
ansible-playbook playbooks/00-prerequisites.yaml
ansible-playbook playbooks/01-containerd.yml
ansible-playbook playbooks/01-containerd-registry.yaml
ansible-playbook playbooks/02-kubeadm-install.yml
ansible-playbook playbooks/03-init-control-plane.yml
ansible-playbook playbooks/04-install-cni.yml
ansible-playbook playbooks/06-join-workers.yml
ansible-playbook playbooks/03-registry.yaml
ansible-playbook playbooks/04-gitea.yaml
ansible-playbook playbooks/05-tekton.yaml
ansible-playbook playbooks/06-tekton-tasks.yaml
```

## Directory Structure

```
pi-cluster-ansible/
├── ansible.cfg
├── site.yml                    # Master playbook
├── kubeconfig                  # Cluster access config
├── inventory/
│   ├── hosts.yml              # Node inventory
│   └── group_vars/
│       └── all/
│           └── vault.yml      # Encrypted secrets
├── group_vars/
│   ├── all.yaml               # Global variables
│   └── vault.yaml             # Secrets (encrypt with ansible-vault)
├── playbooks/
│   ├── 00-prerequisites.yaml  # Node preparation
│   ├── 01-containerd.yml      # Container runtime
│   ├── 01-containerd-registry.yaml # Registry mirror config
│   ├── 02-kubeadm-install.yml # Kubernetes tools
│   ├── 03-init-control-plane.yml # Cluster init
│   ├── 03-registry.yaml       # Deploy registry
│   ├── 04-install-cni.yml     # Flannel CNI
│   ├── 04-gitea.yaml          # Deploy Gitea
│   ├── 05-tekton.yaml         # Deploy Tekton
│   ├── 06-join-workers.yml    # Join worker nodes
│   ├── 06-tekton-tasks.yaml   # ClusterTasks
│   └── 99-teardown.yaml       # Destroy cluster
├── files/
│   └── manifests/
│       ├── registry/          # Registry k8s manifests
│       ├── gitea/             # Gitea k8s manifests
│       └── tekton/            # Tekton ClusterTasks
└── roles/                     # Ansible roles (future)
```

## ClusterTasks

| Task | Description |
|------|-------------|
| `git-clone` | Clone from internal Gitea |
| `buildah-build-push` | Build & push images (ARM64, privileged) |
| `kustomize-deploy` | Deploy with kustomize overlays |

## Application Repository Standard

All applications should follow this structure:

```
app-name/
├── src/
├── Dockerfile
├── k8s/
│   ├── base/
│   │   ├── kustomization.yaml
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── overlays/
│       └── prod/
│           └── kustomization.yaml
└── tekton/
    ├── pipeline.yaml
    └── trigger.yaml
```

## Pre-pull Images for Air-Gap

Before disconnecting from internet, pull these images:

```bash
# Core images
docker pull registry:2
docker pull gitea/gitea:1.21-linux-arm64
docker pull postgres:15-alpine
docker pull quay.io/buildah/buildah:latest
docker pull alpine/git:latest
docker pull bitnami/kubectl:latest

# Tekton images (see group_vars/all.yaml for versions)
docker pull gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/controller:v0.56.0
docker pull gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/webhook:v0.56.0
# ... additional tekton images
```

## Configuration

### group_vars/all.yaml
Main configuration file with registry, gitea, and tekton settings.

### group_vars/vault.yaml
Secrets file - encrypt with:
```bash
ansible-vault encrypt group_vars/vault.yaml
```

## Usage Examples

### Run a Pipeline Manually

```bash
kubectl create -f - <<EOF
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: myapp-
  namespace: tekton-pipelines
spec:
  serviceAccountName: tekton-pipeline-sa
  pipelineRef:
    name: build-and-deploy
  params:
    - name: git-url
      value: "http://gitea.local:30300/myorg/myapp.git"
    - name: image-name
      value: "registry.local:30500/myapp:v1"
  workspaces:
    - name: shared-workspace
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 1Gi
EOF
```

### Configure Gitea Webhook

1. Go to Repository → Settings → Webhooks
2. Add webhook:
   - URL: `http://<node-ip>:30800`
   - Content-Type: `application/json`
   - Trigger: Push events

## Teardown

```bash
# Interactive (requires confirmation)
ansible-playbook playbooks/99-teardown.yaml

# Force (no confirmation)
ansible-playbook playbooks/99-teardown.yaml -e force_teardown=true
```

## Nodes

| Node | IP | Role | User |
|------|-----|------|------|
| rpi-1 | 192.168.8.197 | control-plane | rpi1 |
| rpi-2 | 192.168.8.196 | worker | rpi2 |
| rpi-3 | 192.168.8.195 | worker | rpi3 |
| rpi-4 | 192.168.8.194 | worker | rpi4 |
| rpi-5 | 192.168.8.108 | worker | pi |
| rpi-6 | 192.168.8.235 | worker | pi |
| rpi-7 | 192.168.8.209 | worker | pi |
| rpi-8 | 192.168.8.202 | worker | pi |
| rpi-9 | 192.168.8.187 | worker | pi |
| rpi-10 | 192.168.8.210 | worker | pi |
| rpi-11 | 192.168.8.231 | worker | pi |
| rpi-12 | 192.168.8.105 | worker | pi |
| cm3588-plus | 192.168.8.152 | worker | root |

## License

MIT
