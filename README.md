# compute-cluster

A small HPC cluster I built from scratch on Hyper-V: one head node and two compute nodes,
with shared storage, authentication and Slurm. All of it is configured by
Ansible from this repo.

```
                 ┌──────────────── ubuntu-vm (head) ─────────────────┐
  me ── ssh ──▶  │ Ansible control node · NFS server · slurmctld     │
                 └───────────────────────────────────────────────────┘
                          │ /shared (NFS)      │
                ┌─────────┴────────┐  ┌────────┴─────────┐
                │ node1.           │  │ node2            │
                │ 2 vCPU · 1.5 GB  │  │ 2 vCPU · 1.5 GB  │
                └──────────────────┘  └──────────────────┘
```

## Features

- **Nodes from a cloud image.** node1 and node2 are Ubuntu 24.04 cloud images, set up on first
  boot by cloud-init (hostname, user, SSH keys). 
- **Shared storage (NFS).** The head exports `/shared` to the cluster subnet. Every node mounts
  it at the same path, so a job can be submitted on the head, run on a node, and write its
  output where the head can read it.
- **Authentication (MUNGE).** One shared key on every host, so each Slurm message proves which
  user sent it.
- **Scheduler (Slurm 23.11).** `slurmctld` on the head, `slurmd` on each node. CPU cores and
  memory are allocated per job, and each job is confined to its allocation with cgroups v2.
- **Everything as code (Ansible).** One command sets up the whole cluster, and running it again
  changes nothing unless something drifted:
  - the subnet in `/etc/exports` comes from the head's network facts
  - the node lines in `slurm.conf` are generated from each node's CPU and memory facts
  - handlers reload exports or restart daemons only when their config actually changes

| Playbook | What it sets up |
|---|---|
| `nfs-server.yml` | NFS server and `/etc/exports` on the head |
| `nfs-client.yml` | `/shared` mounted on the compute nodes (and in `/etc/fstab`) |
| `munge.yml` | munge on every host with the shared key |
| `slurm.yml` | Slurm packages, `slurm.conf` and `cgroup.conf`, and the right daemon per host |
