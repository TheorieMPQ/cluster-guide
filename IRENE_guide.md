## Guide to using the IRENE supercomputer

Since we are now many adepts of th-top, sometimes we have to wait in the virtual queue for our jobs to launch. Thankfully we gained access to the IRENE supercomputer with virtually unlimited access until march 2022.

*Access to IRENE: you need to complete a form and ask Cristiano and Loic Noel, the IT guy, to sign it.*

### Package installation

IRENE does not have internet access, so you will not be able to add packages to julia in the usual way. 

To do so, follow these steps in the right order (and replace username with your username, don't just copy and run the lines, especially those with `export`):

1) connect to th-top
2) `module unload julia`
3) `module load julia/1.5` (this is because the IRENE version is 1.5.3, it means that on th-top this will be the version to add packages to to transfer them to IRENE).
4) Remove components of your .julia folder `rm -rf .julia`(keep a copy in case you need to)
5) `export JULIA_DEPOT_PATH="/home/username/.julia"`
6) Launch julia, add packages you want and run `]precompile` to precompile everything
8) Exit julia, and transfer your .julia directory to IRENE with `rsync -rvazh .julia/ username@irene-amd-fr.ccc.cea.fr:/ccc/cont003/home/unipdide/username/.julia/` (check that there is a .julia folder in IRENE).
10) `module load julia`
11) `export JULIA_DEPOT_PATH="/ccc/cont003/dsku/blanchet/home/user/unipdide/username/.julia"` (replace username by your username!)
12) Run julia, check with `]status` that you have the desired packages and have fun!

If you're a bit lazy you can ask me (Kaelan) and I can send you a compressed file that contains the following packages:

QuantumOpticsBase, QuantumOptics, LinearAlgebra, SparseArrays, ElasticArrays, Random, Distributions, Statistics, DifferentialEquations, OrdinaryDiffEq, DiffEqBase, DiffEqCallbacks, LightGraphs, Kronecker, IterativeSolvers, LinearMaps, DataStructures, KrylovKit, Interpolations, JLD, JLD2, BSON, Revise, Distributed, LsqFit, Optim, Conda, PyCall, FFTW, AbstractFFTs, ProgressMeter, MLDataUtils, StatsBase, Dates, Flux

Unzip it on your home directory with `tar -xzvf julia_irene.tar.gz -C /ccc/cont003/home/unipdide/username ` and start from step 10.

### Submitting a job

Here is an example of a julia script:
```
#!/bin/bash
#MSUB -q  rome 
#MSUB -A  gen12462
#MSUB -m  scratch,work,store 
#MSUB -T 3600 
#MSUB -o juliaTest.out
#MSUB -e juliaTest.err 
module load julia
echo "running a job" 
julia /ccc/cont003/dsku/blanchet/home/user/unipdide/username/JULIA_FILE_NAME.jl
```

`-T` is the time limit in seconds (if not specified default is 7200). You can run your job with `ccc_msub my_script`, and obtain information about it with `ccc_mpeek job_id`. The command `ccc_mpp`is like `qstat` on th-top, but here it is useless since too many jobs are running. 
For more information, type `machine_info` in IRENE or visit this page https://forge.ipsl.jussieu.fr/igcmg_doc/wiki/Doc/ComputingCenters/TGCC/Irene.
