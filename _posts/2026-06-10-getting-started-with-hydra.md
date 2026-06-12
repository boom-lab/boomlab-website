---
title: Getting started with Hydra
author: colette-kelly
tags: lab, how-to, computing
---
# Getting started with Hydra 🐉

A quick guide to using Python and Jupyter Notebooks on WHOI's new computer cluster, Hydra. For more info about Hydra, see the IS department's announcement:

🔗 **[About Hydra](https://boom.science/2026/06/04/about-hydra.html)**

For a broader intro to HPC with SLURM and copying files to/from the cluster:

🔗 **[HPC Basics](https://boom.science/2023/10/03/poseidon-basics.html)**

---

## Overview

- [Get an account on Hydra](#get-an-account-on-hydra)
- [Log onto Hydra](#log-onto-hydra)
- [Set up passwordless SSH (one-time)](#set-up-passwordless-ssh-one-time)
- [Create a conda virtual environment](#create-a-conda-virtual-environment)
- [Activate a conda virtual environment](#activate-a-conda-virtual-environment)
- [Install new packages](#install-new-packages)
- [Start an interactive session](#start-an-interactive-session)
- [Start a Jupyter Notebook](#start-a-jupyter-notebook)
- [Shut down a Jupyter Notebook](#shut-down-a-jupyter-notebook)

---

## Get an account on Hydra

Submit a ServiceNow ticket to request an account and migrate your existing workflows:

🔗 **[Submit a migration request](https://whoi.service-now.com/whoi_sp)**

IS will set up your new account with all of your existing home directories from Poseidon. The directories on Hydra mirror those on Poseidon, so files you continue working on in Poseidon will automatically update on Hydra.

---

## Log onto Hydra

```bash
colette$ ssh <WHOI ID>@hydra.whoi.edu
```

Use the same password as your other WHOI accounts. This will open something like:

```
      ___           ___           ___           ___           ___     
     /\__\         |\__\         /\  \         /\  \         /\  \    
    /:/  /         |:|  |       /::\  \       /::\  \       /::\  \   
   /:/__/          |:|  |      /:/\:\  \     /:/\:\  \     /:/\:\  \  
  /::\  \ ___      |:|__|__   /:/  \:\__\   /::\~\:\  \   /::\~\:\  \ 
 /:/\:\  /\__\     /::::\__\ /:/__/ \:|__| /:/\:\ \:\__\ /:/\:\ \:\__\
 \/__\:\/:/  /    /:/~~/~    \:\  \ /:/  / \/_|::\/:/  / \/__\:\/:/  /
      \::/  /    /:/  /       \:\  /:/  /     |:|::/  /       \::/  / 
      /:/  /     \/__/         \:\/:/  /      |:|\/__/        /:/  /  
     /:/  /                     \::/__/       |:|  |         /:/  /   
     \/__/                       ~~            \|__|         \/__/    

Welcome to the Hydra Cluster at WHOI!

Please do not run anything on the login nodes and submit all jobs to SLURM. All running
jobs/processes on the login nodes may be terminated without notice.

===================================================================================
Apptainer (Singularity) is now in the default path - no 'module load singularity' is needed.

$SCRATCH no longer provides any performance advantage. 

Paths and base environment have changed from Poseidon - please review modules and your .bashrc file.
===================================================================================

Last login: Wed Jun 10 16:38:14 2026 from 10.130.128.135
[colette.kelly@hydra-l2 ~]$
```

---

## Set up passwordless SSH (one-time)

The recommended way to tunnel into a Jupyter Notebook (see [Start a Jupyter Notebook](#start-a-jupyter-notebook)) uses SSH **ProxyJump**, which needs key-based authentication to the compute nodes. The compute nodes don't accept your password from your laptop directly — only the login nodes do — so set up an SSH key once on your **local machine** to authenticate everywhere on the cluster.

First, check whether you already have a key:

```bash
colette$ ls ~/.ssh/*.pub
```

If you see a `.pub` file (e.g., `id_ed25519.pub`), skip ahead to installing it. Otherwise, generate one. Press Enter to accept the default file location. The passphrase is optional — it encrypts the private key on your laptop so it's useless if copied; leave it empty for fully passwordless logins:

```bash
colette$ ssh-keygen -t ed25519
```

This creates two files in `~/.ssh`: `id_ed25519` (the **private** key — stays on your laptop, never moves) and `id_ed25519.pub` (the **public** key — goes to Hydra).

Install the public key into your Hydra home directory. Because your home directory is shared across the login and compute nodes, this single key works on all of them:

```bash
colette$ cat ~/.ssh/id_ed25519.pub | ssh colette.kelly@hydra.whoi.edu \
  'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys'
```

Enter your WHOI password when prompted — this is the last time you'll need it for tunneling.

---

## Create a conda virtual environment

Like Poseidon, Hydra provides Python through conda. Check which versions are available:

```bash
[colette.kelly@hydra-l2 ~]$ module spider conda
```

Load miniconda (recommended: 25.9):

```bash
[colette.kelly@hydra-l2 ~]$ module load miniconda/25.9
```

The quickest way to set up an environment is from an existing `environment.yml` file. You can find an example [here](https://github.com/ckelly314/ml-argo-n2o/blob/main/environment.yml).

```bash
[colette.kelly@hydra-l2 ml-argo-n2o]$ conda env create -f environment.yml
```

Accept the terms of service when prompted:

```
Do you accept the Terms of Service (ToS) for https://repo.anaconda.com/pkgs/main? 
[(a)ccept/(r)eject/(v)iew]: a
Do you accept the Terms of Service (ToS) for https://repo.anaconda.com/pkgs/r? 
[(a)ccept/(r)eject/(v)iew]: a
```

Complex environments can take a few minutes to solve. When it finishes successfully, you'll see:

```
# To activate this environment, use
#
#     $ conda activate ml-argo-n2o
#
# To deactivate an active environment, use
#
#     $ conda deactivate
```

You only need to create the environment once.

---

## Activate a conda virtual environment

Do NOT run `conda init` — it modifies your `.bashrc` in ways that cause problems. Instead, source the conda shell script each time you log in:

```bash
[colette.kelly@hydra-l2 ~]$ . $CONDA_PREFIX/etc/profile.d/conda.sh
```

Then activate the environment:

```bash
[colette.kelly@hydra-l2 ~]$ conda activate ml-argo-n2o
```

Your prompt will now show the environment name as a prefix:

```bash
(ml-argo-n2o) [colette.kelly@hydra-l2 ~]$
```

---

## Install new packages

Activate the virtual environment first, then run `conda install`:

```bash
(ml-argo-n2o) [colette.kelly@hydra-l2 ~]$ conda install -c conda-forge joblib
```

---

## Start an interactive session

Before running any code or starting a Jupyter Notebook, request an interactive session on a compute node:

```bash
[colette.kelly@hydra-l2 ~]$ srun -N 1 -n 16 --mem=50GB --partition=compute --time=01:00:00 --pty bash
```

Your prompt will update to show which compute node you've been assigned (e.g., `cn056`):

```bash
[colette.kelly@cn056 ~]$ 
```

---

## Start a Jupyter Notebook

On the compute node, load miniconda and activate your environment:

```bash
[colette.kelly@cn056 ~]$ module load miniconda/25.9
[colette.kelly@cn056 ~]$ . $CONDA_PREFIX/etc/profile.d/conda.sh
[colette.kelly@cn056 ~]$ conda activate ml-argo-n2o
```

Start Jupyter with `--no-browser`. Pick a port number (e.g., `8890`, `8891`, `8892`, ...) — if a port is ever busy, just pick another:

```bash
(ml-argo-n2o) [colette.kelly@cn056 ~]$ jupyter notebook --no-browser --port=8895
```

In a **separate terminal window**, open an SSH tunnel from your local machine to the notebook. The recommended method uses **ProxyJump** (`-J`) to connect straight through the login node to your compute node (this requires the [one-time SSH key setup](#set-up-passwordless-ssh-one-time)):

```bash
colette$ ssh -L 8895:localhost:8895 -J colette.kelly@hydra.whoi.edu colette.kelly@cn056
```

Replace `cn056` with your assigned compute node and `8895` with your port. With your SSH key in place, this connects without a password. A successful tunnel looks like:

```
Last login: Wed Jun 10 20:49:11 2026 from 10.151.0.4
[colette.kelly@cn056 ~]$ 
```

**Why ProxyJump?** It opens a listening port only on your laptop — the login node just relays the connection. The older nested method (below) *also* opens a port on the shared login node, which can fail with `Address already in use` even on your first session of the day, because another user or a leftover process is holding that port on the login node. Restarting your laptop doesn't free it, so the error is confusing. ProxyJump avoids this entirely.

Open a browser and go to `http://localhost:8895/tree` (replace `8895` with your port). Enter your Jupyter password if prompted.

### Backup: nested SSH tunnel

If ProxyJump doesn't work — for example, if key-based authentication to the compute nodes isn't available — fall back to the original nested tunnel. Each hop authenticates from inside the cluster, so it works without an SSH key, but it does open a port on the login node:

```bash
colette$ ssh -t -t colette.kelly@hydra.whoi.edu -L 8895:localhost:8895 ssh cn056 -L 8895:localhost:8895
```

Replace `cn056` with your assigned compute node. If you see `bind: Address already in use`, the login node's copy of that port is taken — pick a different, higher port (e.g., `8917`), restart Jupyter on that port, and retry the tunnel. This applies to both Poseidon and Hydra sessions.

---

## Shut down a Jupyter Notebook

From the Jupyter interface, go to **File → Shut Down**.

Check for active SLURM sessions and cancel them so they're not running idle:

```bash
(ml-argo-n2o) [colette.kelly@cn056 ~]$ squeue -u colette.kelly
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
             43336   compute     bash colette.  R      24:21      1 cn056
```

```bash
(ml-argo-n2o) [colette.kelly@cn056 ~]$ scancel 43336
```

The tunnel window will close on its own:

```
[colette.kelly@cn056 ~]$ Connection to cn056 closed by remote host.
Connection to cn056 closed.
Connection to hydra.whoi.edu closed.
```

If you used the ProxyJump tunnel, just close that terminal window (or type `logout`) to tear it down:

```bash
[colette.kelly@cn056 ~]$ logout
Connection to cn056 closed.
```
