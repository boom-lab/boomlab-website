---
title: Use the GPUs on Hydra
author: colette-kelly
tags: lab, how-to, computing, gpu
---

# Use the GPUs on Hydra

This resource covers what a GPU actually is, when and why you should use one, how to access a GPU node on Hydra, and how to build a conda environment with **nvForest** so you can run a trained scikit-learn random forest on the GPU.

If you are new to high-performing computing or have never used the Poseidon or Hydra computer clusters before, I recommend checking out my other blog posts on high-performance computing:

🔗 **[HPC basics](https://boom.science/2023/10/03/poseidon-basics.html)**
🔗 **[Getting started with Hydra](https://boom.science/2026/06/10/getting-started-with-hydra.html)**


## Overview

- [Why use a GPU](#why-use-a-gpu)
- [What is a GPU?](#what-is-a-gpu)
- [When (and when not) to use a GPU](#when-and-when-not-to-use-a-gpu)
- [Start an interactive session on a GPU node](#start-an-interactive-session-on-a-gpu-node)
- [Check the GPU you got](#check-the-gpu-you-got)
- [Set up a conda environment with nvForest](#set-up-a-conda-environment-with-nvforest)
- [Add cuDF and CuPy (optional: data on the GPU)](#add-cudf-and-cupy-optional-data-on-the-gpu)
- [A tiny example: scikit-learn → GPUForestInferenceRegressor](#a-tiny-example-scikit-learn--gpuforestinferenceregressor)
- [From one forest to millions of rows: the CPU/GPU boundary](#from-one-forest-to-millions-of-rows-the-cpugpu-boundary)
- [Scale it up: all four GPUs on one node](#scale-it-up-all-four-gpus-on-one-node)
- [Reading the results: seff and the payoff](#reading-the-results-seff-and-the-payoff)

---

## Why use a GPU?

I have several Random Forest models that predict nitrous oxide (N2O) from thigns like temperature, salinity, oxygen, and nitrate. I tried using these models to predict N2O on a large feature dataset, using traditional CPU resources on Hydra. The result: disaster. Well not quite... but even with the backend parallel processing built into Scikit-Learn's Random Forest Inference Regressor running on 16 parallel cores, it took four minutes for one model (out of four) to do the prediction on a single month of data (out of a 384-month dataset). Why? Each random forest model has 600 decision trees, and each month of data includes 20 million rows. Here's that example:

```Python
tab = pd.read_parquet(f"{FEATURE_DIR}/features_{year}_{month:02d}.parquet")
sub = tab[(tab["pres"] > PMIN) & (tab["pres"] < PMAX)]
sub
	flat_index	pres	SA	CT	o2	no3	AOU
0	0	2.5	35.926041	21.980213	220.591110	0.237462	-4.066206
1	1	2.5	35.921921	21.993288	220.561722	0.236455	-4.079823
2	2	2.5	35.915161	21.993517	220.585419	0.238403	-4.093950
3	3	2.5	35.912014	21.846651	221.111893	0.268081	-4.059767
4	4	2.5	35.904461	21.846907	221.126190	0.270777	-4.063333
...	...	...	...	...	...	...	...
20452731	22457595	1975.0	35.313015	3.343683	258.761871	18.643993	60.655472
20452732	22457596	1975.0	35.313972	3.336914	258.803680	18.642286	60.664089
20452733	22457597	1975.0	35.314999	3.336901	258.716064	18.648146	60.749294
20452734	22457598	1975.0	35.315697	3.338968	258.554932	18.659927	60.892616
20452735	22457599	1975.0	35.316238	3.344537	258.351166	18.674530	61.051662
20452736 rows × 7 columns

model_id = 1
cols = MODEL_FEATURES[model_id]
# set up Numpy array to do predictions on
# use "sub" -> so the N2O predictions will only be made for the selected pressure range
X = np.ascontiguousarray(sub[cols].to_numpy(dtype=F32))
# load RF models
models = {mid: load(f"/user/colette.kelly/ml-argo-n2o/model{mid}_rf_full.joblib") for mid in (1,2,3,4)}
model = models[model_id]
model.n_jobs = 16
print(f"[backend] model {model_id}: sklearn ({type(model).__name__})") # expect RandomForestRegressor
[backend] model 1: sklearn (RandomForestRegressor)
%%timeit
# let sklearn's backend parallelization do the work
y = np.asarray(model.predict(X), dtype = F32)
4min 19s ± 7.01 s per loop (mean ± std. dev. of 7 runs, 1 loop each)
```