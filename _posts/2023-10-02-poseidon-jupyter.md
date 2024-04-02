---
title: Running Jupyter Notebooks on Poseidon
author: colette-kelly
tags: lab, how-to, computing
---
# Running Jupyter Notebooks Remotely on Poseidon
Based on [this blog post](https://alexanderlabwhoi.github.io/post/2019-03-08_jpn_slurm/) from the Alexander Lab.

### 1: Connect to Poseidon
Open a Terminal window on your local machine and type:
```bash
colette$ ssh <whoi.ID>@poseidon.whoi.edu
```

### 2. Start interactive session on a compute node
Replace `--mem=1000` with how much memory you're requesting. You need at least 1 GB memory to work with data from one float. The session will time out if you go over the requested memory. Replace `--time=00:30:00` with how much time you need.
```bash
[colette.kelly@poseidon-l1 ~]$ srun -N 1 -n 1 --mem=1000 --time=00:30:00 --pty bash
```
To list how much memory is available on each node, run:
```bash
sinfo -N -l
```

### 3. Module load miniconda
To use conda or any other software on Poseidon, you need to load the associated module first. 
```bash
[colette.kelly@pn023 ~]$ module load miniconda/23.11
```

### 4. Initialize conda environment
```bash
[colette.kelly@pn023 ~]$ . $CONDA_PREFIX/etc/profile.d/conda.sh
```

### 5. Activate conda environment
Replace `envname` with the name of your environment:
```bash
[colette.kelly@pn023 ~]$ conda activate envname
```

### 6. Launch Jupyter Notebook without a browser window
```bash
(envname) [colette.kelly@pn023 ~]$ jupyter notebook --no-browser --port=8888
```
You'll need to use a port number that isn't already in use. To check which ports are already in use, run:
```bash
ss -tuln
```

### 7. Create an SSH tunnel
Open a new Terminal window on your local machine and type the following. Replace `colette.kelly` with your username. Replace `pn023` with whatever node the interactive session is running on.
```bash
(base) CowardlyLion:~ colette$ ssh -t -t colette.kelly@poseidon.whoi.edu -L 8888:localhost:8888 ssh pn023 -L 8888:localhost:8888
```

### 8. Open Jupyter Notebook in a browser window
Copy-past the url that begins with "http://localhost:8888/tree?token=" into a browser window. Or,
if Jupyter passwork is configured, just enter `localhost:8888` in a browser window.

You can set configure a Jupyter password by entering the following in a terminal:
```bash
jupyter notebook password
```

You'll need to restart your Jupyter Notebook session to be able to use the new password.

### 9. Shut down Jupyter Notebook
Make sure you shut down the Jupyter Notebook (either control-C in Terminal or with the Jupyter GUI), otherwise you might run into issues re-opening a port.

# Troubleshooting

If, in creating the SSH tunnel, you get the error:
```bash
bind: Address already in use
```
### 1. Shut down the existing notebook (either ctrl-C. in Terminal or with the Jupyter GUI). Disconnect the ssh tunnel with 'logout' and ctrl-C.

### 2. In the first terminal window, start a new notebook with a different port
You can do this by replacing `8888` with a different number. E.g., 
```bash
(envname) [colette.kelly@pn023 ~]$ jupyter notebook --no-browser --port=8889
```

### 3. In the second terminal window, start a tunnel to the new port
Replace `8888` with the new number, e.g.:
```bash
(base) CowardlyLion:~ colette$ ssh -t -t colette.kelly@poseidon.whoi.edu -L 8889:localhost:8889 ssh pn023 -L 8889:localhost:8889
```

### 4. Open Jupyter Notebook in a browser window
You can either copy the notebook URL or enter `localhost:8889` in a browser window, making sure the number matches the port number from above.

You may need to iterate through 1-4 above a few times until you find an open port.
