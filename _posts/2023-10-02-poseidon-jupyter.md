---
title: Running Jupyter Notebooks on Poseidon
author: colette-kelly
tags: lab, how-to, computing
---
# Running Jupyter Notebooks Remotely on Poseidon
Based on [this blog post](https://alexanderlabwhoi.github.io/post/2019-03-08_jpn_slurm/) from the Alexander Lab.

## Connect to Posiedon and launch a Jupyter Notebook
### 1: Connect to Poseidon
```bash
colette$ ssh <whoi.ID>@poseidon.whoi.edu
```

### 2. Module load Anaconda
The anaconda3/2021.11 module contains a recent version of conda. The anaconda/5.1 module contains a much older version of conda, which will make solving environments and adding packages extremely slow.
```bash
[colette.kelly@poseidon-l2 ~]$ module load anaconda3/2021.11
```

### 3. Activate conda environment
```bash
[colette.kelly@pn135 ~]$ source activate <env-name>
```

### 4. Start interactive session on a compute node
You need at least 1 GB memory to work with data from one float; more (2 GB+) for working with a region's worth. The session will time out if you go over the requested memory.
```bash
(conda-env) srun -N 1 -n 1 --mem=1000 --time=00:30:00 --pty bash
```

### 5. run "export XDG_RUNTIME_DIR=" - only seem to need to do this once
You should only need to run this command once:
```bash
[colette.kelly@pn026 ~]$ export XDG_RUNTIME_DIR=""
```

### 6. Launch Jupyter Notebook:
```bash
[colette.kelly@pn026 ~]$ jupyter notebook --no-browser --port=8888
```
If you get "Debugger warning: It seems that frozen modules are being used", try this instead:
```bash
[colette.kelly@pn026 ~]$ python3 -Xfrozen_modules=off -m jupyter notebook --no-browser 
```

### 7. Open a new Terminal window and create a tunnel from local machine to Poseidon
Here, replace "pn026" with whatever node the interactive session is running on.
```bash
colette$ ssh -t -t <whoi.ID>@poseidon.whoi.edu -L 8888:localhost:8888 ssh pn026 -L 8888:localhost:8888
```

### 8. Open Jupyter Notebook in a browser window
Copy-past the url that begins with "http://localhost:8888/tree?token=" into a browser window. Or,
if Jupyter passwork is configured, just enter "localhost:8888" in a browser window.

You can set configure a Jupyter password by entering the following in a terminal:
```bash
jupyter notebook password
```

You'll need to restart your Jupyter Notebook session to be able to use the new password.

### 9. Shut down Jupyter Notebook
Make sure you shut down the Jupyter Notebook (either control-C in Terminal or with the Jupyter GUI), otherwise you might run into issues re-opening a port.
