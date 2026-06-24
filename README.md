# cluster-builder — programmable HA Kubernetes lab

Spin up a **highly-available** Kubernetes cluster (2 load balancers with a floating
VIP + N control planes + N workers) on VirtualBox, then build the cluster by running
**interactive, narrated** scripts on each node.

> Vagrant only creates the VMs. Kubernetes is installed by the `setup-*.sh` scripts so
> you can watch and learn every stage. The scripts also work on **physical machines**.

## Files

| File | What it does |
|------|--------------|
| `configure.sh` | Interactive generator — asks counts (LB/CP/WK), subnet, IP ranges → writes `cluster.yaml`. |
| `cluster.yaml` | The cluster definition (edit by hand or via `configure.sh`). |
| `Vagrantfile` | Reads `cluster.yaml` and creates the VMs with hostnames, IPs and `/etc/hosts`. |
| `setup-loadbalancer.sh` | HAProxy (+ Keepalived VIP) on each load balancer. |
| `setup-controlplane.sh` | Prepares the node, then `kubeadm init` (first) or join (others) + Calico. |
| `setup-worker.sh` | Prepares the node and joins it as a worker. |
| `serve.sh` | Optional: serve these scripts over HTTP so nodes can `curl` them. |

## Quick start

```bash
# 1) choose your topology (counts, subnet, IPs) — control planes must be odd
./configure.sh

# 2) create the VMs
vagrant up

```bash
# 3) You can now run the scripts directly from your public GitHub repository!
```

Then, in this order, SSH into each VM (`vagrant ssh <name>`) and run the matching script directly from GitHub:

```bash
# on k8s-lb1 and k8s-lb2:
curl -fsSL https://raw.githubusercontent.com/nisalatp/KubeCluseter/main/setup-loadbalancer.sh | bash

# on k8s-cp1 (answer "yes" to FIRST control plane):
curl -fsSL https://raw.githubusercontent.com/nisalatp/KubeCluseter/main/setup-controlplane.sh | bash
# on k8s-cp2 and k8s-cp3 (answer "no"; it auto-reads the join command on Vagrant):
curl -fsSL https://raw.githubusercontent.com/nisalatp/KubeCluseter/main/setup-controlplane.sh | bash

# on k8s-w1 and k8s-w2:
curl -fsSL https://raw.githubusercontent.com/nisalatp/KubeCluseter/main/setup-worker.sh | bash
```

Verify on any control plane:

```bash
kubectl get nodes -o wide
```

## Hosting the scripts

`curl | bash` needs a URL. By default, you can pull your scripts directly from the GitHub repository you created:

- **GitHub (Primary):** The commands in the Quick Start section above pull securely from `https://raw.githubusercontent.com/nisalatp/KubeCluseter/main/...`
- **Local (Fallback):** If your VMs don't have internet access, you can run `./serve.sh` on your host machine to serve them over your local network using python's built-in HTTP server.

## Footprint (running ~7 VMs at once)

Tuned to be light on a laptop:

- **Minimal base box** — `configure.sh` shows a menu; the default `bento/debian-12` has the
  leanest idle RAM (~120 MB). Pick Ubuntu from the menu if you prefer.
- **Linked clones** — the VMs share one base disk image instead of seven full copies.
- **Trimmed hardware** — tiny video RAM, no USB, KVM paravirtualization.
- **Right-sized RAM** — LB 512 MB, control plane 2048 MB, worker 1536 MB by default
  (editable in `configure.sh` or `cluster.yaml`). The default 2 + 3 + 2 cluster ≈ **10 GB** of VM RAM.
  On a tight laptop, lower the counts (e.g. 1 LB + 1 CP + 1 WK) or the per-VM RAM.

## Notes

- On Vagrant, the first control plane writes the join commands to `/vagrant/join-commands.txt`;
  the other scripts read it automatically, so you usually just press Enter.
- Control planes must be an **odd** number (etcd quorum). With 3 you can lose one at a time.
- Pod CIDR (`10.244.0.0/16`) must not overlap your machine subnet.
- See the accompanying Word document for the full, explained walkthrough.
