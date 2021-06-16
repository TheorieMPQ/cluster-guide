## Guide to using the IRENE supercomputer

Since we are now many adepts of th-top, sometimes we have to wait in the virtual queue for our jobs to launch. Thankfully we gained access to the IRENE supercomputer with virtually unlimited access until march 2022.

*Access to IRENE: you need to complete a form and ask Cristiano and Loïc Noël, our IT responsible, to sign it.*

### Package installation

IRENE does not have internet access, so you will not be able to add packages to julia in the usual way. 

To do so, follow these steps:

1) connect to th-top
2) `module unload julia`
3) `module load julia/1.5` (this is because the IRENE version is 1.5.3, it means that on th-top this will be the version to add packages to to transfer them to IRENE).
4) Remove components of your .julia folder belonging to the version 1.5 `rm -rf .julia/environments/v1.5* ; rm -rf .julia/compiled/v1.5*`(keep a copy in case you need to)
5) Download julia-1.5:w `get https://julialang-s3.julialang.org/bin/linux/x64/1.5/julia-1.5.4-linux-x86_64.tar.gz`
6) Decompress: `tar zxvf julia-1.5.4-linux-x86_64.tar.gz`
7) Launch julia with `julia-1.5.4/bin/julia`, add packages you want and run `]precompile` to precompile everything
8) Exit julia, and transfer your .julia directory to IRENE with `rsync -rvazh .julia/ username@irene-amd-fr.ccc.cea.fr:/ccc/cont003/home/unipdide/username/.julia/` (check that there is a .julia folder in IRENE). As an alternative, one may instead issue `rsync -rvazh username@th-top.mpq.univ-paris-diderot.fr:.julia/ .julia/` from the home directory on IRENE.
9) Transfer the `julia-1.5.4` directory to IRENE, compress it with `tar` if too slow.
10) Connect to Irene
11) launch julia with `julia-1.5.4/bin/julia`, and precompile everything `]precompile`
14) Check with `using ...` that you have your desired packages and have fun!

### Submitting a job

Here is an example of a script:
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
