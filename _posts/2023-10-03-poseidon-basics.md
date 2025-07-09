---
title: Poseidon basics
author: colette-kelly
tags: lab, how-to, computing
---
# Using Python on Poseidon
Quick overview of connecting to and running Python scripts on Poseidon.

Before you do anything, you will need to *request an account on Poseidon* with an IS help ticket.

Logging in:
```bash
colette$ ssh <WHOI ID>@poseidon.whoi.edu
```
### Slurm
Slurm is the resource manager on Poseidon. It lets you request computing resources and submit jobs to the cluster. You submit jobs with Sbatch scripts, which tell Slurm what computing resources you need, and then run the commands to activate virtual environments and run Python scripts (example [here](https://github.com/boom-lab/poseidon-data/blob/main/cds/get_era5.sh)). Slurm also lets you start an interactive session from which you can run Jupyter Notebooks (more info [here](https://boom.science/2023/10/02/poseidon-jupyter.html)).

Start a ```bash``` interactive session on a compute node with one core for 20 minutes:
```bash
[colette.kelly@poseidon-l1 ~]$ srun -N 1 -n 1 --mem=500 --time=00:20:00 --pty bash
```
*[colette.kelly@poseidon-l1 ~] indicates that we are on login node 1 of Poseidon's two login nodes.*<br>

Submit a job:
```bash
[colette.kelly@poseidon-l1 ~]$ sbatch <scriptname>.sh
```
See what jobs are currently running:
```bash
[colette.kelly@poseidon-l1 ~]$ squeue -u <WHOI ID>
```
Cancel a job:
```bash
[colette.kelly@poseidon-l1 ~]$ scancel <JOBID>
```
Track how much memory a job is using:
```bash
[colette.kelly@poseidon-l2 era5]$ sstat -j 3190091 --format=JobID,MaxRSS
       JobID     MaxRSS 
------------ ---------- 
3190091.0       254340K
```
Get effective job usage for completed job (this example is terrible, your efficiency should be way better than this!!):
```bash
[colette.kelly@poseidon-l2 cds]$ seff 3190107
Job ID: 3190107
Cluster: slurm_cluster
User/Group: colette.kelly/domain users
State: COMPLETED (exit code 0)
Nodes: 1
Cores per node: 20
CPU Utilized: 00:05:22
CPU Efficiency: 1.21% of 07:25:00 core-walltime
Job Wall-clock time: 00:22:15
Memory Utilized: 903.89 MB
Memory Efficiency: 44.14% of 2.00 GB
```

### Creating a virtual environment with conda
1. Module load miniconda
```bash
[colette.kelly@poseidon-l2 ~]$ module load miniconda/23.11
```
2. Create the virtual environment based on an environment.yml file
```bash
[colette.kelly@poseidon-l2]$ conda env create -f environment.yml
```
(You can also activate the virtual environment and install packages individually)

3. The following command initializes the conda environment:
```bash
[colette.kelly@pn136 ~]$ . $CONDA_PREFIX/etc/profile.d/conda.sh
```
You can also do `conda init` but that modifies your .bashrc file, which creates potential conflicts if you want to try different conda environments or use any Poseidon module beyond the one you used to run `conda init`.

4. Activate the virtual environment
```bash
[colette.kelly@poseidon-l2]$ conda activate <env-name>
```
OR
```bash
[colette.kelly@poseidon-l2]$ source activate <env-name>
```

### Creating a Python virtual environment with Pip
1. Module load Python3.9
```bash
[colette.kelly@poseidon-l1 ~]$ module load python3/3.9.10
```
*Note that while Poseidon has a Python3.10 module, as of writing (8/15/2023), the python3/3.10.2 module does not have the SSL certificate installed for the Python Package Index. This essentially means that trying to install packages with pip in the python3/3.10.2 module will throw an error. So for the time being, use the python3/3.9.10, instead. You CAN use conda to set up a virtual environment in the python3/3.10.2 module and install packages in that conda environment, but updating conda on Poseidon takes an extremely long time (> 3hours) right now so this is not recommended.*
3. Create a virtual environment called "py39-env" with Python3.9:
```bash
[colette.kelly@poseidon-l1 ~]$ python3 -m venv ~/py39-env
```
4. Activate the virtual environment:
```bash
[colette.kelly@poseidon-l1 ~]$ source ~/py39-env/bin/activate
```
5. Install all of the packages in a requirements.txt file ([example](https://github.com/boom-lab/poseidon-data/blob/main/requirements.txt)) in the virtual environment:
```bash
(py39-env) [colette.kelly@poseidon-l1 ~]$ python3 -m pip install -r poseidon-data/requirements.txt
```
6. OR, install one package at a time (for example, Pandas):
```bash
(py39-env) [colette.kelly@poseidon-l1 ~]$ python3 -m pip install Pandas
```
7. Leave the virtual environment:
```bash
(py39-env) [colette.kelly@poseidon-l1 ~]$ deactivate
```

### Run a Python script on Poseidon
Once a virtual environment is activated (either in an interactive session or in a Batch script), you'll have access to the version of Python in that environment, as well as all of the Python packages you've installed in that virtual environment. Now, you can run a script that requires 3rd-party packages:
```bash
[colette.kelly@pn055 ~]$ python3 get_era5.py
```

If you're running the python code with a batch script, include a line to activate the virtual environment before the command to run the Python script, as in this [example](https://github.com/boom-lab/poseidon-data/blob/main/examples/python3rdparty.sh).


### Syncing files to Poseidon
There are lots of ways to sync files between Poseidon and your local machine. My preferred way is to clone Github repositories directly into my home directory on Poseidon. This gives you all the benefits of git (version control, branches) so you can undo changes if you mess things up.

A really simple way to sync files is with secure copy. Instructions are below because the Poseidon documentation is a bit out of date as of writing (10/3/23).

1. Secure copy a file from local machine to Poseidon \\
Open a Terminal on your local machine and run:
```bash
colette$ scp <local-filename> <whoiID>@poseidon.whoi.edu:/vortexfs1/home/<whoiID>/<remote-filename>
```

2. Secure copy a file from Poseidon to local machine \\
Open a Terminal on your local machine and run:
```bash
colette$ scp <whoiID>@poseidon.whoi.edu:/vortexfs1/home/<whoiID>/<remote-filename> <local-filename>
```