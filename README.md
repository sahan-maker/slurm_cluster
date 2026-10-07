# Setting Up a Slurm Cluster: Complete Beginners Guide including Network SetUp

> A practical, from-scratch guide to building a small Slurm cluster, including the mistakes I made along the way.

<!-- Short intro: why you built it, what you wanted to achieve. 2-3 sentences. -->

## Table of Contents

1. [Overview](#overview)
2. [Hardware and Network](#hardware-and-network)
3. [Step 0: Prepare network](#step-0-prepare)
4. [Step 1: Prepare All Nodes](#step-1-prepare-all-nodes)
5. [Step 2: Shared Storage (NFS)](#step-2-shared-storage-nfs)
6. [Step 3: Munge Authentication](#step-3-munge-authentication)
7. [Step 4: Install and Configure Slurm](#step-4-install-and-configure-slurm)
8. [Step 5: Start the Services](#step-5-start-the-services)
9. [Step 6: Test the Cluster](#step-6-test-the-cluster)
10. [Troubleshooting](#troubleshooting)
11. [Lessons Learned](#lessons-learned)
12. [References](#references)

---

## Overview

<!-- What is Slurm? One paragraph. What will the reader have at the end? -->

**What you'll build:**

- 1 head (control) node running `slurmctld`
- N worker nodes running `slurmd`
- A shared filesystem exported over NFS
- Munge for authentication between nodes

**Who this is for:** <!-- e.g. students, lab admins, hobbyists -->

---

## Hardware and Network

| Role | Hostname | IP Address | OS | Notes |
|------|----------|------------|----|-------|
| Head | `head` | `10.x.x.x` | Ubuntu 22.04.5 LTS (GUI version) | Runs slurmctld, NFS server |
| Worker | `node0` | `10.x.x.x` | Ubuntu 22.04.5 LTS Server version(No GUI) | |
| Worker | `node1` | `10.x.x.x` | Ubuntu 22.04.5 LTS Server version(No GUI) | |
| Worker | `node2` | `10.x.x.x` | Ubuntu 22.04.5 LTS Server version(No GUI) | |

- **Subnet:** `10.x.x.0/24`
- **Admin user:** `<username>`
- **Slurm version:** `<version>`

<!-- Optional: add a network diagram or photo of your setup -->
<!-- ![Cluster photo](images/cluster.jpg) -->


## Step 0: Prepare

### 0.1 Install Ubuntu Jammy Jellyfish GUI version on head node 

GUI version is pretty easy to setup.
1. Download iso image
2. Make a bootable drive using rufus
3. Reboot and install using bootable drive
4. Give same name to user on every computer including head. Like,
> user: elec_cluster
> 
> pc_name: head
> 
> password: same on each machine

   

### 0.2 Install Ubuntu Jammy Jellyfish Server version on worker nodes

1. Download server iso image
2. Make a bootable drive using rufus
3. Reboot and install using bootable drive
4. Uncheck lvm group when prompted during install
5. Give same name to user but change pc names(node0, node1). Like,
> user: elec_cluster
> 
> pc_name: node0
> 
> password: same on each machine
6. Connect everything to network switch via ethernet cables
7. Connect a network cable from wall internet to switch
### 0.3 On Head node
Change wired network to manual ip and give an ip address.

### 0.4 On Worker nodes

```bash
ip a

```
This will show network state. Could be down. Note down the network interface name. Something like enp0s31f6 
1. Bring the network state UP if down,
```bash
sudo ip link set enp0s31f6 up
ip a
```
2. Tell cloud-init to stop managing network config.
```bash
sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```
Add this content: 
> network: {config: disabled}
>
And save (ctrl+o to save then ctrl+x to exit)

3. Now edit the netplan file with your actual config
 ```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```
(or whatever the filename is under /etc/netplan/ — use ls /etc/netplan/ to confirm)
4. Replace the content with, using your own ip address, check network admin to figure out what to use instead of 10.20.52 and at the end add a free ip like 175 :
```bash
network:
  version: 2
  ethernets:
    enp0s31f6:
      dhcp4: no
      addresses:
        - 10.20.52.175/24
      routes:
        - to: default
          via: 10.20.52.254
      nameservers:
        addresses: [10.20.2.1]
```
save and exit. 
5. sudo netplan apply
6. Repeat on all nodes. Change ips like, 175,176,177
7. Do "ip a" and confirm the ip address is static after reboots

## Step 1: Prepare All Nodes

### 1.1 Set up `/etc/hosts`
1. On head node terminal
```bash
sudo nano /etc/hosts
```
2. edit following according to your ip setup

```text
127.0.0.1 localhost
127.0.1.1 head

10.20.52.183 head
10.20.52.175 node0
10.20.52.176 node1
10.20.52.177 node2

```
save and exit
3. Install openssh
```bash
sudo apt update 
sudo apt install openssh-server -y
```
4. Gotta copy the hosts file to worker nodes
   
   use the destination pc ip address
```bash
scp /etc/hosts elec_cluster@10.20.52.176:/tmp/hosts
```
5. SSH into that machine and then copy hosts from tmp to etc
```bash
ssh elec_cluster@10.20.52.176
sudo cp /tmp/hosts /etc/hosts
```
6.Repeat this file copying for all nodes until each have an identical hosts file in their etc. But then edit all those files using nano since first two lines should read 127.0.0.1 localhost and 127.0.1.1 nodename. Nodename is it's own name since 1.1 is home. 

7. Check whether you can now ssh and back into each node using just name. Replace elec_cluster with your on cluster/user name
```bash
ssh elec_cluster@node1
ssh elec_cluster@head
```

### 1.2 Set up passwordless SSH

```bash
ssh-keygen -t ed25519
ssh-copy-id elec_cluster@node0
# repeat for each node
```

### 1.3 Sync time

```bash
sudo apt install -y chrony
sudo systemctl enable --now chrony
```

### 1.4 Run commands on all nodes from head

```bash
for n in node0 node1 node2; do
  ssh "$n" "hostname && uptime"
done
```

<!-- Add anything that tripped you up here -->

---

## Step 2: Shared Storage (NFS)

### 2.1 On the head node (server)

```bash
sudo apt install -y nfs-kernel-server
sudo mkdir -p /mnt/shared
sudo chown <user>:<user> /mnt/shared
```

```text
# /etc/exports
/mnt/shared 10.x.x.0/24(rw,sync,no_subtree_check)
```

```bash
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
```

### 2.2 On each worker node (client)

```bash
sudo apt install -y nfs-common
sudo mkdir -p /mnt/shared
sudo mount head:/mnt/shared /mnt/shared
```

Make it persistent:

```text
# /etc/fstab
head:/mnt/shared  /mnt/shared  nfs  defaults,_netdev  0  0
```

### 2.3 Verify

```bash
df -h /mnt/shared
touch /mnt/shared/test && ls -l /mnt/shared
```

### 2.4 Rebuilding NFS from scratch (my script)

<!-- Optional: include your teardown/rebuild script and explain what went wrong that made you need it -->

```bash
#!/usr/bin/env bash
# rebuild_nfs.sh
# TODO: paste your script here
```

---

## Step 3: Munge Authentication

Munge must use the **same key** on every node, and UIDs/GIDs for `munge` must match.

### 3.1 Install

```bash
sudo apt install -y munge libmunge-dev
```

### 3.2 Create the key on head and distribute it

```bash
sudo /usr/sbin/mungekey --create    # or: sudo create-munge-key
sudo scp /etc/munge/munge.key <user>@node0:/tmp/munge.key
# on node0:
sudo mv /tmp/munge.key /etc/munge/munge.key
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key
```

### 3.3 Start and test

```bash
sudo systemctl enable --now munge
munge -n | unmunge                  # local test
munge -n | ssh node0 unmunge        # cross-node test
```

<!-- Common issue: clock skew or mismatched key -> "Rejected credential" -->

---

## Step 4: Install and Configure Slurm

### 4.1 Install packages

**Head node:**

```bash
sudo apt install -y slurm-wlm
```

**Worker nodes:**

```bash
sudo apt install -y slurmd slurm-client
```

<!-- Or describe building from source if that's what you did -->

### 4.2 Create `slurm.conf`

Create on head, then copy the **identical** file to every node (or put it on the NFS share).

```text
# /etc/slurm/slurm.conf
ClusterName=mycluster
SlurmctldHost=head

MpiDefault=none
ProctrackType=proctrack/linuxproc
ReturnToService=1
SlurmctldPidFile=/var/run/slurmctld.pid
SlurmdPidFile=/var/run/slurmd.pid
SlurmdSpoolDir=/var/lib/slurm/slurmd
StateSaveLocation=/var/lib/slurm/slurmctld
SlurmUser=slurm

SchedulerType=sched/backfill
SelectType=select/cons_tres
SelectTypeParameters=CR_Core

# COMPUTE NODES (check with: slurmd -C)
NodeName=node[0-2] CPUs=4 RealMemory=7900 State=UNKNOWN
PartitionName=main Nodes=node[0-2] Default=YES MaxTime=INFINITE State=UP
```

Find the correct hardware values on each node:

```bash
slurmd -C
```

### 4.3 Distribute the config

```bash
for n in node0 node1 node2; do
  scp /etc/slurm/slurm.conf "$n":/tmp/slurm.conf
  ssh "$n" "sudo mv /tmp/slurm.conf /etc/slurm/slurm.conf"
done
```

### 4.4 Create required directories

```bash
sudo mkdir -p /var/lib/slurm/slurmctld /var/lib/slurm/slurmd /var/log/slurm
sudo chown -R slurm:slurm /var/lib/slurm /var/log/slurm
```

---

## Step 5: Start the Services

**Head:**

```bash
sudo systemctl enable --now slurmctld
```

**Workers:**

```bash
sudo systemctl enable --now slurmd
```

Check status:

```bash
sinfo
scontrol show nodes
```

---

## Step 6: Test the Cluster

### 6.1 Simple command across nodes

```bash
srun -N3 hostname
```

### 6.2 Batch job

```bash
#!/bin/bash
#SBATCH --job-name=test
#SBATCH --output=/mnt/shared/test_%j.out
#SBATCH --nodes=2
#SBATCH --ntasks=2

srun hostname
```

```bash
sbatch test.sh
squeue
cat /mnt/shared/test_*.out
```

### 6.3 Expected output

```text
<!-- paste your actual sinfo / squeue output here -->
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Node shows `down` or `unk*` | `slurmd` not running, or config mismatch | `sudo systemctl status slurmd`, compare `slurm.conf` on both sides |
| Node shows `inval` | `CPUs`/`RealMemory` in config exceeds the real hardware | Re-run `slurmd -C` and fix `slurm.conf` |
| `Authentication failure` / `Invalid credential` | Munge key mismatch or clock skew | Re-copy the key, check `date` on all nodes |
| `Unable to contact slurm controller` | Firewall, wrong `SlurmctldHost`, or `slurmctld` down | Check `ufw status`, hostname resolution, `journalctl -u slurmctld` |
| NFS mount hangs | Export or firewall problem, stale handle | `exportfs -v`, `showmount -e head`, remount |
| Jobs stuck in `PD` | No available resources or node drained | `scontrol show job <id>`, then `scontrol update nodename=nodeX state=resume` |

Useful logs:

```bash
sudo journalctl -u slurmctld -f
sudo journalctl -u slurmd -f
sudo tail -f /var/log/slurm/slurmctld.log
```

---

## Lessons Learned

<!-- This is the heart of your guide. Be honest and specific. -->

- **What went wrong:** 
- **What I'd do differently:** 
- **What took the longest:** 
- **Things I wish I knew earlier:** 

---

## Next Steps

- [ ] Slurm accounting (`slurmdbd` + MariaDB)
- [ ] Cgroups for resource limits
- [ ] GPU / GRES support
- [ ] Monitoring (Prometheus, Grafana)
- [ ] Environment modules and MPI

---

## References

- [Slurm Documentation](https://slurm.schedmd.com/documentation.html)
- [Slurm Configuration Tool](https://slurm.schedmd.com/configurator.html)
- [Munge](https://github.com/dun/munge)
- <!-- add the tutorials and forum posts that helped you -->

---

## Contributing

Found a mistake or have a better approach? Open an issue or pull request.

## License

<!-- e.g. MIT, or CC BY 4.0 for documentation -->

---

*Written by [Your Name](https://github.com/your-username). Last updated: YYYY-MM-DD.*
