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

# 3) host the scripts so the VMs can curl them (run on your host or k8s-lb1)
./serve.sh           # prints the curl URLs (uses http://<ip>:8000/...)
```

Then, in this order, SSH into each VM (`vagrant ssh <name>`) and run the matching script:

```bash
# on k8s-lb1 and k8s-lb2:
curl -fsSL http://<serve-ip>:8000/setup-loadbalancer.sh | bash

# on k8s-cp1 (answer "yes" to FIRST control plane):
curl -fsSL http://<serve-ip>:8000/setup-controlplane.sh | bash
# on k8s-cp2 and k8s-cp3 (answer "no"; it auto-reads the join command on Vagrant):
curl -fsSL http://<serve-ip>:8000/setup-controlplane.sh | bash

# on k8s-w1 and k8s-w2:
curl -fsSL http://<serve-ip>:8000/setup-worker.sh | bash
```

Verify on any control plane:

```bash
kubectl get nodes -o wide
```

## Hosting the scripts

`curl | bash` needs a URL. Two easy options:

- **Local:** run `./serve.sh` on a machine the nodes can reach (it runs `python3 -m http.server`).
- **GitHub:** push this folder to a repo and use the raw URLs, e.g.
  `curl -fsSL https://raw.githubusercontent.com/<you>/<repo>/main/cluster-builder/setup-worker.sh | bash`.

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
