# IS announcement: Hydra — WHOI's New HPC Computing Cluster

This is a repeat of the IS announcement that went out via WHOI Headlines about the new Hydra computer cluster. We have copied it here for a quick reference.

> **WHOI Information Services** is excited to announce the launch of **Hydra**, a new high-performance computing (HPC) cluster that will replace the current Poseidon cluster.

---

## Overview

Hydra represents a significant upgrade in compute power, memory capacity, networking speed, and overall system architecture to better support current and future research needs — from bioinformatics to ocean modeling to state-of-the-art AI/ML.

As part of this transition, all new users will be onboarded on Hydra, and Poseidon will be formally retired as current workloads migrate to the new system.

---

## Hardware

Hydra is currently comprised of **8,644 Intel Xeon Gold cores** with a minimum base frequency of **2.4 GHz**. The WHOI core compute resources include:

| Resource | Specs |
|---|---|
| 92 traditional compute nodes | 256 GB RAM · 48 cores/node |
| 2 shared memory nodes | 2 TB RAM · 48 cores/node |
| 1 GPU node | 4× NVIDIA H200 GPUs (NVLink) · 1 TB RAM · 564 GB vRAM (141 GB/GPU) |
| 2 login nodes | User access via 10 Gb Ethernet |

All primary nodes and VAST storage are connected via **high-speed, low-latency NDR InfiniBand (200 Gb/sec)**.

---

## System Details

- **OS:** Rocky Linux 9
- **Job scheduler:** SLURM (same as Poseidon)
- All compute jobs must be submitted via a SLURM submit script or an interactive SLURM job
- **Login nodes** are intended only for editing, compiling, and submitting jobs

---

## Migration

WHOI IS HPC staff are reaching out to current Poseidon users to begin their migration to Hydra. Migration will continue over the next several months.

If you would like to move your workloads to Hydra **before IS contacts you**, please submit a ServiceNow ticket:

🔗 **[Submit a migration request](https://whoi.service-now.com/whoi_sp)**

We will reach out as soon as possible.