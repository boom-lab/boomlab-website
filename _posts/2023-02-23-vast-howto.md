---
title: Shared HPC network drive
author: david-nicholson
tags: lab, how-to, computing
---
# Storage on the WHOI VAST drive
Our lab has 100 Tb dedicated space on the WHOI VAST SSD storage drive. Permissions for current lab members should be in place, but contact Roo if you need access

## Connecting to VAST
### Option 1: smb share
You can directly mount the shared drive to your laptop. MacOS instructions below

1. With Finder open, <kbd>Ctrl</kbd>+<kbd>k</kbd> to "Connect to Server"
2. server address: ```smb://vast:/proj/boom```
3. Once mounted, the volume is accessible directly at ```/Volumes/boom/```

### Option 2: from Poseidon
1. ssh to ```yourusername@poseidon.whoi.edu```
2. ```cd /proj/boom```
*probably need to be inside WHOI firewall

### Globus

[Overview of Globus @ WHOI](https://whoi-it.whoi.edu/globus-file-transfer-at-whoi/) - Basically you log into Globus via [](https://app.globus.org) with your WHOI account, then search for and connect to the "WHOI Vast Proj Space" collection - you'll see the boom directory and other Vast proj directories within and can connect and transfer between any that you have permissions for. 


## Organization
We are just setting this up, so the below may change...
```
.
├── data
│   ├── gdac
│   │   ├── aoml  [8570 entries exceeds filelimit, not opening dir]
│   │   ├── bodc
│   │   ├── coriolis
│   │   ├── csio
│   │   ├── csiro
│   │   ├── incois
│   │   ├── jma
│   │   ├── kma
│   │   ├── kordi
│   │   ├── meds
│   │   └── nmdis
│   └── osu
│       └── orca.science.oregonstate.edu
│           └── data
│               └── 1x2
│                   └── monthly
│                       └── cafe.modis.r2018
│                           └── hdf  [20 entries exceeds filelimit, not opening dir]
|    └── **add new datasets here...**
└── users
    └── dnicholson
    └── **create your dir here...**

```


