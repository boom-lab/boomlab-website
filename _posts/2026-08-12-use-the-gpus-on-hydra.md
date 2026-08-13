---
title: Use the GPUs on Hydra
author: colette-kelly
tags: lab, how-to, computing, gpu
---

# Use the GPUs on Hydra

This resource covers what a GPU actually is, when and why you should use one, how to access a GPU node on Hydra, and how to build a conda environment with nvForest so you can run a trained scikit-learn random forest on the GPU.

If you are new to high-performing computing or have never used the Poseidon or Hydra computer clusters before, I recommend checking out my other blog posts on high-performance computing:

🔗 [HPC basics](https://boom.science/2023/10/03/poseidon-basics.html)
🔗 [Getting started with Hydra](https://boom.science/2026/06/10/getting-started-with-hydra.html)


Here are the third party references I used when figuring this out:

🔗 [nvForest docs](https://docs.rapids.ai/api/nvforest/stable/)
🔗 [RAPIDS install guide](https://docs.rapids.ai/install/)
🔗 [RAPIDS platform support](https://docs.rapids.ai/platform-support/)

## Overview

- [Why use a GPU](#why-use-a-gpu)
- [What is a GPU?](#what-is-a-gpu)
- [When (and when not) to use a GPU](#when-and-when-not-to-use-a-gpu)
- [Start an interactive session on a GPU node](#start-an-interactive-session-on-a-gpu-node)
- [Check the GPU you got](#check-the-gpu-you-got)
- [Set up a conda environment with nvForest](#set-up-a-conda-environment-with-nvforest)
- [Add cuDF and CuPy (optional: data on the GPU)](#add-cudf-and-cupy-optional-data-on-the-gpu)
- [A tiny example: scikit-learn → GPUForestInferenceRegressor](#a-tiny-example-scikit-learn--gpuforestinferenceregressor)
- [The CPU/GPU boundary](#the-cpugpu-boundary)
- [Scale to all four GPUs](#scale-to-all-four-gpus)
- [The payoff](#the-payoff)

---

## Why use a GPU?

I have several Random Forest models that predict nitrous oxide (N₂O) from things like temperature, salinity, oxygen, and nitrate. I tried using these models to predict N₂O on a large feature dataset, using traditional CPU resources on Hydra. The result: disaster. Well not quite... but even with the backend parallel processing built into Scikit-Learn's Random Forest Inference Regressor running on 16 parallel cores, it took four minutes for one model (out of four) to do the prediction on a single month of data (out of a 384-month dataset). Why? Each random forest model has 600 decision trees, and each month of data includes 20 million rows. Here's that example:

```python
import numpy as np
import pandas as pd
import xarray as xr
import gsw
from joblib import load

TPATH = "/proj/boom/data/argo_rfromv22_temp"
SPATH = "/proj/boom/data/argo_rfromv22_sal"
O2PATH = "/proj/boom/data/gobai-highres-o2"
NO3PATH = "/proj/boom/data/gobai-highres-no3"

FEATURE_DIR = "/proj/boom/data/highres_features"
PRED_DIR = "/proj/boom/data/highres_n2o_pred/etphighres"
MODEL_DIR = "/user/colette.kelly/ml-argo-n2o"

# lat-lon slices for the eastern tropical Pacific
LAT = slice(-30,25)
LON = slice(360 - 180, 360 - 70)

# pressure window for prediction - set to something small for testing
PMIN, PMAX = 0.0, 5000.0

# model features
# needs to match the feature order each RF was trained on
MODEL_FEATURES = {
    1: ["AOU", "CT", "SA", "no3"],
    2: ["o2", "CT", "SA", "no3"],
    3: ["AOU", "CT", "SA"],
    4: ["o2", "CT", "SA"]}

F32 = np.float32

year = 2024
month = 1

tab = pd.read_parquet(f"{FEATURE_DIR}/features_{year}_{month:02d}.parquet")
sub = tab[(tab["pres"] > PMIN) & (tab["pres"] < PMAX)]

model_id = 1
cols = MODEL_FEATURES[model_id]

# set up Numpy array to do predictions on
# use "sub" -> so the N2O predictions will only be made for the selected pressure range
X = np.ascontiguousarray(sub[cols].to_numpy(dtype=F32))
X.shape

# load RF models
models = {mid: load(f"/user/colette.kelly/ml-argo-n2o/model{mid}_rf_full.joblib") for mid in (1,2,3,4)}

model = models[model_id]
model.n_jobs = -1
print(f"[backend] model {model_id}: sklearn ({type(model).__name__})") # expect RandomForestRegressor

%timeit y = np.asarray(model.predict(X), dtype=F32)

6min 15s ± 2.6 s per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

---

## What is a GPU?

So, time to turn to a GPU. [WHOI's new computer cluster Hydra](https://boom.science/2026/06/04/about-hydra.html) has some amazing resources, including a GPU node. Here's that actually means:

The traditional compute nodes on Hydra have 48 powerful CPUs (central processing units) per node that are great at sequential logic: "do this, then check that, then decide what's next."" The GPU node on Hydra has 4 NVDIA GPUs (graphics processing units), each of which has *thousands* of small, simple cores that are great at doing the *same operation across a huge amount of data at once*.

A commonly-used analogy: a CPU is a kitchen full of expert chefs each cooking a multi-course meal from start to finish. A GPU is a *stadium*-full  (actually though) of line cooks all chopping onions at the same time. So is your problem, cook this fancy multi-course meal? Or chop 10 million onions the same way? If you're doing something highly parallelizable, the GPU is the correct path.

The one catch: a GPU has its *own memory*, separate from the computer's main (host) memory. Your data has to be copied from host memory into GPU (device) memory and the results copied back. So you should avoid shuttling data across that boundary, and only use a GPU if the computational savings from parallelizing on a GPU are greater than the cost of copying data back and forth.

---

## When (and when not) to use a GPU

Good fits for a GPU:

- Training or running machine-learning models on lots of data
- Large array / linear-algebra math (big matrix ops, FFTs)
- DataFrame operations on large tables via [RAPIDS](https://docs.rapids.ai/) (cuDF / CuPy)
- Anything "embarrassingly parallel" — the same simple operation over millions of elements

My use case: applying trained random forest models to *~20 million rows* of gridded ocean data to predict N₂O. On a CPU this was painfully slow and kept running out of memory; on the GPU the same inference finishes in seconds.

Skip the GPU for:

- Small data, because the host↔device copy dominates and you gain nothing
- Serial, branchy logic that can't be parallelized
- I/O-bound work: reading/writing files, downloading data, conda installs
- Plain pandas on a modest table

A GPU can actually be *slower* once you count the transfer overhead.

### GPU etiquette

GPU nodes are a scarce, shared resource. Only request one for work that actually uses the GPU. Do your feature engineering, file wrangling, and environment installs on a regular `compute` node and save the `gpu` partition for the inference itself.

---

## Start an interactive session on a GPU node

Similar to [starting an interactive session on a traditional compute node](https://boom.science/2026/06/10/getting-started-with-hydra.html), but ask SLURM for the `gpu` partition and one GPU with `--gpus-per-node`:
```bash
[colette.kelly@hydra-l2 ~]$ srun --partition=gpu --gpus-per-node=1 --mem=32GB --time=00:30:00 --pty bash
```

Your prompt updates to the GPU node you were assigned:

```bash
[colette.kelly@gpu003 ~]$
```

A note on `--mem`: that's *host* RAM, which is separate from the GPU's own memory. Loading a single large random forest into host memory can take ~14 GB on its own, so make sure you've requested enough memory or the session will time out. You don't need to size it to the GPU's memory.

---

## Check the GPU you got

It's worth checking the GPU hardware to make sure you're installing versions of libraries that are compatible with the hardware. This is a bit more involved on GPU than with a traditional compute node. Here's what you need to do.

Before installing anything, see what hardware you landed on:
```bash
[colette.kelly@gpu003 ~]$ nvidia-smi --query-gpu=name,compute_cap,driver_version --format=csv
name, compute_cap, driver_version
NVIDIA H200, 9.0, 610.43.02
```
- `name` → NVIDIA H200. A current-generation (Hopper) datacenter GPU with ~141 GB of on-board memory.
- `compute_cap` → 9.0. The GPU's *compute capability* — its architecture / feature level. The RAPIDS ecosystem, which includes the *nvforest* library for RF inference, requires 7.0 or higher (Volta and newer).
- `driver_version` → 610.43.02. The NVIDIA driver installed on the node. The driver determines the newest CUDA version the node can run. RAPIDS 26.06 needs driver 525.60.13+ for its CUDA 12 build or 580.65.06+ for CUDA 13. RAPIDS ships separate builds for CUDA 12 and CUDA 13, and you have to install one your driver supports. We have `610` so we can use either.

### Sidebar: what is CUDA?

CUDA is NVIDIA's platform for running general-purpose computation on their GPUs:

- The driver is the system-level CUDA, installed by the cluster admins, that sets the *maximum* CUDA version the node can run.
- The toolkit / runtime are the CUDA libraries your code actually links against. Conda installs this into your environment, so you control it.

You just need to make sure that the driver is new enough for the toolkit you install. On our H200 with driver 610, we're clear for either CUDA 12 or 13.

---

## Set up a conda environment with nvForest

nvForest is the RAPIDS library for fast decision-tree and forest inference on the GPU (it's the successor to the now-deprecated `cuml.fil`). It imports an *already-trained* scikit-learn / XGBoost / LightGBM forest and runs its predictions on the GPU, so you don't retrain anything.

### Step 1 — see what's actually available

Install everything on a regular compute node, not a GPU.

```bash
[colette.kelly@hydra-l1 ~]$ srun -N 1 -n 1 --mem=50GB --time=01:00:00 --partition=compute --pty bash
[colette.kelly@cn059 ~]$ module load miniconda/25.9
[colette.kelly@cn059 ~]$ . $CONDA_PREFIX/etc/profile.d/conda.sh
[colette.kelly@cn059 ~]$ # See what channels conda already has configured — look for `defaults`
conda config --show channels
channels:
  - defaults
[colette.kelly@cn059 ~]$ conda search -c rapidsai -c conda-forge nvforest
2 channel Terms of Service accepted
Loading channels: done
# Name                       Version           Build  Channel             
nvforest                    26.04.00 cuda12_cp311_abi3_260408_e77480fb  rapidsai            
nvforest                    26.04.00 cuda13_cp311_abi3_260408_e77480fb  rapidsai            
nvforest                    26.06.00 cuda12_cp311_abi3_260604_a9216c61  rapidsai            
nvforest                    26.06.00 cuda13_cp311_abi3_260604_a9216c61  rapidsai 
```
- `nvforest` is on the rapidsai channel, with 26.06.00 available
- `cuda12` = the CUDA 12 build;  `cuda13` = the CUDA 13 build
- `cp311` = built against CPython 3.11.
- `abi3` is the key part — these are built against Python's stable ABI, so a single build is forward-compatible across later 3.x. The giveaway is that there's only one build per (version, CUDA) instead of separate cp311/cp312/cp313 rows. That means the package declares roughly python >=3.11, and 26.06 supports Python 3.11 through 3.14, so python=3.12 resolves to that same cuda12_cp311_abi3 build. You're not locked to 3.11.

### Step 2 — create the environment

```bash
[colette.kelly@cn059 ~]$ CONDA_OVERRIDE_CUDA=12.9 conda create -n nvforest \
  --override-channels -c rapidsai -c conda-forge \
  --solver=libmamba \
  nvforest=26.06.* python=3.12 'cuda-version>=12.2,<=12.9'
```

- The CONDA_OVERRIDE_CUDA=12.9 matters because we're solving on cn059, a regular compute node with no GPU driver visible — without it, conda can't detect a CUDA driver and the solve gets confused.

- `--override-channels` is needed because Hydra's base conda has *only the `defaults` (Anaconda) channel* configured and RAPIDS packages are *not* built for `defaults`.

- `-c rapidsai -c conda-forge -c nodefaults` are the channels that RAPIDS needs.

- `--solver=libmamba` is a fast solver that handles big RAPIDS environments better than the classic solver and gives clearer errors.

- Pin nvforest=26.06.00 for reproducibility, so you get the build you just inspected rather than whatever's latest later.

- `python=3.12` — if python=3.12 ever throws a solver complaint, drop to python=3.11 to match the build label.

- The 'cuda-version>=12.2,<=12.9' pin selects the cuda12 build (the cuda12_cp311_abi3 row) over the cuda13 one. Hydra's H200 + driver 610 would run either, but cuda12 keeps the surrounding conda-forge ecosystem on the better-trodden path. 

### Step 3 - add other packages

```bash
[colette.kelly@cn059 ~]$ conda activate nvforest 
(nvforest) [colette.kelly@cn059 ~]$ conda install xarray pandas pyarrow scikit-learn netcdf4 matplotlib dask
```

---

## Add cuDF and CuPy (optional: data on the GPU)

If you want to read your feature files *straight into GPU memory* instead of going through pandas on the host, add cuDF. CuPy almost certainly came in already as an nvForest dependency, so check first:

```bash
(nvforest) [colette.kelly@cn059 ~]$ conda list | grep -E 'cudf|cupy'
cupy                             14.1.1   ...   conda-forge
cupy-core                        14.1.1   ...   conda-forge
```

If only `cupy` shows up, add `cudf`:

```bash
(nvforest) [colette.kelly@cn058 ~]$ CONDA_OVERRIDE_CUDA=12.9 conda install -n nvforest \
  --override-channels -c rapidsai -c conda-forge \
  --solver=libmamba \
  'cudf=26.06.*'
```

Warnings:

- Don't `pip install` cupy or cudf into a conda RAPIDS environment. Mixing Pip's CUDA wheels and conda's CUDA packages breaks things in confusing ways.
- cuDF pins pandas to a supported range (e.g. `pandas < 2.4`), so installing it may downgrade pandas. Ordinary feature-engineering code (building DataFrames, `to_parquet`) uses only stable pandas API and is unaffected.

### Verify

Activate the environment and confirm everything imports together:

```bash
(nvforest) [colette.kelly@cn059 ~]$ python -c "import nvforest, cudf, cupy, sklearn; \
print('nvforest', '+ cudf', cudf.__version__, '+ cupy', cupy.__version__, '— all import OK')"
```

---

## A tiny example: scikit-learn → GPUForestInferenceRegressor

Exit the interactive session on the compute node:
```bash
(nvforest) [colette.kelly@cn059 ~]$ exit
exit
```

Now I start an interactive session on the GPU node and load an RF model:
```bash                          
[colette.kelly@hydra-l2 ~]$ srun --partition=gpu --gpus-per-node=1 --time=00:30:00 --mem=100GB --pty bash
[colette.kelly@gpu003 ~]$ module load miniconda/25.9
[colette.kelly@gpu003 ~]$ . $CONDA_PREFIX/etc/profile.d/conda.sh
[colette.kelly@gpu003 ~]$ conda activate nvforest
(nvforest) [colette.kelly@gpu003 ~]$ ipython
```
```python
import joblib, nvforest

skl = joblib.load("/user/colette.kelly/ml-argo-n2o/model1_rf_full.joblib")

fm = nvforest.load_from_sklearn(skl, device = "gpu")

print(type(fm).__name__)
GPUForestInferenceRegressor

import numpy as np

X = np.random.rand(5, skl.n_features_in_).astype("float32")
print(fm.predict(X))
[[328.24543802]
 [328.62658223]
 [333.47100661]
 [328.17146862]
 [328.19903352]]

```

---

## the CPU/GPU boundary

The tiny example handed nvForest a NumPy array sitting in host memory. That works, but it copies the data across the host→device boundary on every call. But we  want the data to land in GPU memory in the first place. That's what cuDF is for. It's pandas' API on the GPU, so the code barely changes:

```python
import cudf, cupy as cp

cols = ["AOU", "CT", "SA", "no3"]        # the features this forest was trained on
FEATURE_DIR = "/proj/boom/data/highres_features"
year = 2024
month = 1
path = f"{FEATURE_DIR}/features_{year}_{month:02d}.parquet"
# read the parquet straight into GPU memory, only the columns we need
gdf = cudf.read_parquet(path, columns=cols + ["pres", "flat_index"])
gdf = gdf[(gdf["pres"] > 0) & (gdf["pres"] < 5000)]     # filter runs on the GPU

# feature matrix: contiguous float32, still on the GPU
Xg = cp.ascontiguousarray(gdf[cols].astype("float32").to_cupy())

 %timeit yg = fm.predict(Xg) # inference on the GPU; yg is a CuPy array
 344 ms ± 118 μs per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

⚠️ nvForest always returns a *device* array that lives on the GPU. You need to copy it back explicitly before you can write it:

```python
yg = fm.predict(Xg)
y = cp.asnumpy(yg).astype(np.float32).reshape(-1)
```

`cp.asnumpy` is also a synchronization point between where GPU does the math and CPU writes the file. For a fast pipeline: only copy back the small things you'll actually write. Here, we do inference on 20 million rows on the GPU but only bring home the 1-D prediction vector.

Finishing up:

```python
flat = gdf["flat_index"].to_numpy() # move the cudf series to host numpy

TPATH = "/proj/boom/data/argo_rfromv22_temp"

import numpy as np, xarray as xr

# scatter + NetCDF write are host-side, since netCDF is a CPU library
LAT = slice(-30,25)
LON = slice(360 - 180, 360 - 70)
src = xr.open_dataset(
    f"{TPATH}/RFROMV22_TEMP_STABLE_{year}_{month:02d}.nc"
).sel(latitude = LAT, longitude = LON)["ocean_temperature"]

out = np.full(src.size, np.nan, dtype = np.float32)
out[flat] = y

da = xr.DataArray(out.reshape(src.shape), dims = src.dims, coords = src.coords, name = f"pN2O_model1")
out = f"n2o_model1_{year}_{month:02d}.nc"
da.to_netcdf(out)
print(f"[predict {out} predicted={len(y):,}")
```

---

## Scale to all four GPUs

One month of inference takes seconds. I have 384 of them (monthly, 1993–2024), and Hydra's GPU node has four H200s. I could have either set up a SLURM job array or submitted one job that fanned tasks across the four GPUs. Before decided what to do, I checked the partition:

```bash
[colette.kelly@hydra-l1 ~]$ scontrol show partition gpu | grep -i oversubscribe
   ... OverSubscribe=NO
```

`OverSubscribe=NO` means that SLURM will never hand two jobs to the same core. The GPU node is still shareable, each job getting its own slice of cores, memory, and GPUs.

> 🔬 Sydney gave me a way to check this. I opened one interactive GPU session, then requested a second from another terminal, which was assigned resources on the same node immediately, while the first was still running.

So a job array (`--gpus-per-node=1` per task, throttled with `--array=0-383%4`) would run four single-GPU tasks side by side here and work fine. I still chose a single job that used all four GPUs and fanned the work out over one worker per GPU. The advantage of this is that a worker that holds a GPU for the whole run loads the four forests only once and reuses them, instead of reloading and re-converting them on every one-month task.

### CUDA_VISIBLE_DEVICES

Before submitting a job, you can ask CUDA which of the GPUs it can reach:

```bash
python -c "import cupy; print(cupy.cuda.runtime.getDeviceCount())"
```

### SLURM BATCH SCRIPT

Here's a simplified version of the SLURM batch script I used to set up this job:

```bash
#SBATCH --partition=gpu
#SBATCH --nodes=1
#SBATCH --gpus-per-node=4
#SBATCH --cpus-per-task=96          # the whole node — see the thread note below
#SBATCH --time=01:00:00

NWORKERS=4
THREADS=$(( SLURM_CPUS_PER_TASK / NWORKERS ))   # 96 / 4 = 24 cores each

pids=()
for w in 0 1 2 3; do
    CUDA_VISIBLE_DEVICES=${w} \
    OMP_NUM_THREADS=${THREADS} OPENBLAS_NUM_THREADS=${THREADS} MKL_NUM_THREADS=${THREADS} \
    python3 run_worker.py --worker ${w} --nworkers ${NWORKERS} \
        > worker_${SLURM_JOB_ID}_gpu${w}.log 2>&1 &
    pids+=($!)
done

fail=0
for pid in "${pids[@]}"; do wait "${pid}" || fail=1; done   # reap all, remember failures
exit ${fail}
```

Each worker takes a round-robin slice of the months (`months[worker::nworkers]`), loads the four forests onto its GPU once, and reuses them across all its months:

```python
months = all_months()[args.worker::args.nworkers]             # this GPU's ~96 months
backends = {mid: load_backend(mid) for mid in (1, 2, 3, 4)}   # load once, reuse
for (year, month) in months:
    gdf = read_features(year, month)
    for mid in (1, 2, 3, 4):
        predict_and_write(year, month, gdf, mid, backends[mid])
```

### Don't let four workers each grab all 96 cores

The GPUs do the inference, but the parquet decode and NetCDF write still runs on the CPU. We don't want ~384 threads thrashing over 96 cores. The extra lines `OMP_NUM_THREADS` / `OPENBLAS_NUM_THREADS` / `MKL_NUM_THREADS`  gives each CPU worker a clean slice.

---

## The payoff

When the batch job finishes, `seff` summarizes what it actually used:

```bash
[colette.kelly@hydra-l1 ~]$ seff 118272
State: COMPLETED (exit code 0)
Cores per node: 96
CPU Utilized: 00:16:24
CPU Efficiency: 2.39% of 11:26:24 core-walltime
Job Wall-clock time: 00:07:09
Memory Utilized: 256.00 GB (100.00% of 256.00 GB)
```

Two things:

- CPU efficiency of 2.39% is OK for a GPU job. I reserved 96 cores for seven minutes but only spent 16 CPU-minutes, because the GPUs did the work — the cores are just there so host-side I/O (parquet decode, NetCDF writes) doesn't bottleneck. Don't reflexively "fix" the low number by shrinking the CPU request. But since the node is *shared*, not exclusive, good etiquette is to size `--cpus-per-task` to what you need without bottlenecking your code. This takes trial and error; it's helpful to play with job requests on the compute nodes to get a sense of how much computing power different tasks need.
- Memory at exactly 100% is fine here but it does mean that I bumped against my `--mem=256G` ceiling. The node has a full ~1 TB of memory, so I could have requested more. Just don't reserve 1 TB "to be safe" on a shared node, but rather request what you'll actually use. Again, this is usually a matter of trial and error as you get used to high-performance computing.

The payoff is that the same `.predict()` call, ~20 million rows, model already loaded and data already in memory:

| backend | time for one `.predict()` |
|---|---|
| scikit-learn, CPU | 4 min 19 s |
| nvForest, one H200 | 0.34 s |

So that's a roughly 750× speedup on identical work. So for the full job — four forests over 384 months — that's about 31 billion predictions (20,452,736 grid points × 4 models × 384 months). What extrapolates to *days* of CPU inference finished in 7 minutes across the four H200s. That's what makes using GPUs worth it.
