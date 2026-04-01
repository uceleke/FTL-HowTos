# Talos & Kubernetes Upgrade Runbook

---

## Important Rules

- **Always upgrade Talos one minor version at a time** — no skipping (e.g. 1.9 → 1.10 → 1.11 → 1.12)
- **Always upgrade control plane nodes before workers**
- **Always upgrade Talos OS before upgrading Kubernetes**
- **Wait for each node to return to Ready before moving to the next**
- **Check the [Talos upgrade notes](https://www.talos.dev/latest/introduction/what-is-new/) for breaking changes before each minor version jump**

---

## Pre-Upgrade Checklist

```bash
# 1. Verify all nodes are healthy before starting
kubectl get nodes -o wide
talosctl get members

# 2. Check etcd health on all control plane nodes
talosctl service etcd --nodes <CP1_IP>
talosctl service etcd --nodes <CP2_IP>
talosctl service etcd --nodes <CP3_IP>

# 3. Back up your configs
cp ~/talos/secrets.yaml ~/talos/secrets.yaml.bak
cp ~/talos/kubeconfig ~/talos/kubeconfig.bak

# 4. Confirm current versions
talosctl version --nodes <CP1_IP>
kubectl get nodes -o wide
```

---

## Part 1 — Upgrade Talos OS

### Version Path Example
```
v1.9.x → v1.10.x → v1.11.x → v1.12.x
```

Check the latest patch release for each minor version at:  
https://github.com/siderolabs/talos/releases

### Upgrade Command (per node)
```bash
talosctl upgrade \
  --nodes <NODE_IP> \
  --image ghcr.io/siderolabs/installer:<VERSION>
```

### Step-by-Step Example

```bash
# Upgrade CP-1 first
talosctl upgrade \
  --nodes <CP1_IP> \
  --image ghcr.io/siderolabs/installer:v1.10.4

# Watch the node reboot and rejoin from another CP node
talosctl get members --nodes <CP2_IP>

# Wait until CP-1 shows Ready
kubectl get node <CP1_HOSTNAME>

# Upgrade CP-2
talosctl upgrade \
  --nodes <CP2_IP> \
  --image ghcr.io/siderolabs/installer:v1.10.4

# Wait for Ready, then upgrade CP-3
talosctl upgrade \
  --nodes <CP3_IP> \
  --image ghcr.io/siderolabs/installer:v1.10.4

# Wait for Ready, then upgrade workers one at a time
talosctl upgrade \
  --nodes <WORKER1_IP> \
  --image ghcr.io/siderolabs/installer:v1.10.4

talosctl upgrade \
  --nodes <WORKER2_IP> \
  --image ghcr.io/siderolabs/installer:v1.10.4
```

Repeat this same process for each subsequent minor version (e.g. v1.11.x, v1.12.x). Always verify all nodes are healthy between each minor version jump before proceeding.

### Verify Talos Upgrade Complete
```bash
talosctl version --nodes <CP1_IP>,<CP2_IP>,<CP3_IP>,<WORKER1_IP>,<WORKER2_IP>
```

---

## Part 2 — Upgrade Kubernetes

Only proceed after ALL nodes are on the target Talos version.

### Check Compatible Kubernetes Versions
Each Talos version supports a specific range of Kubernetes versions. Check the compatibility matrix before choosing a target version:  
https://www.talos.dev/latest/introduction/support-matrix/

### Upgrade Command
Kubernetes upgrades in Talos are handled via `talosctl` — do **not** use `kubeadm`.  
The command only needs to run once against one control plane node. Talos handles rolling the upgrade across all nodes automatically.

```bash
talosctl upgrade-k8s \
  --nodes <CP1_IP> \
  --to <K8S_VERSION>

# Example
talosctl upgrade-k8s \
  --nodes <CP1_IP> \
  --to 1.33.0
```

### Check Available Kubernetes Version for Your Talos Release
```bash
talosctl upgrade-k8s --nodes <CP1_IP> --help
```

### Verify Kubernetes Upgrade Complete
```bash
kubectl get nodes -o wide
# All nodes should show the new Kubernetes version
```

---

## Troubleshooting

**Node stuck in NotReady after upgrade:**
```bash
talosctl service kubelet --nodes <NODE_IP>
talosctl logs kubelet --nodes <NODE_IP> | tail -30
```

**etcd unhealthy after upgrade:**
```bash
talosctl service etcd --nodes <CP1_IP>
talosctl logs etcd --nodes <CP1_IP> | tail -30
talosctl etcd members --nodes <CP1_IP>
```

**Roll back a node if upgrade fails:**
```bash
# Talos dual-partition scheme allows automatic rollback
talosctl rollback --nodes <NODE_IP>
```

**Check upgrade status:**
```bash
talosctl get upgradestatus --nodes <NODE_IP>
```

---

## References
- Talos Releases: https://github.com/siderolabs/talos/releases
- Talos Support Matrix: https://www.talos.dev/latest/introduction/support-matrix/
- Talos Upgrade Guide: https://www.talos.dev/latest/talos-guides/upgrading-talos/
- Kubernetes Upgrade via Talos: https://www.talos.dev/latest/kubernetes-guides/upgrading-kubernetes/
