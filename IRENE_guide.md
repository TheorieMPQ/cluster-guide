## Guide to using the IRENE supercomputer

Since we are now many adepts of th-top, sometimes we have to wait in the virtual queue for our jobs to launch. Thankfully we gained access to the IRENE supercomputer with virtually unlimited access until march 2022.

*Access to IRENE: you need to complete a form and ask Cristiano and Loïc Noël, our IT responsible, to sign it.*

### Storage systems on IRENE, quota, groups and ownership

There are different accessible data spaces on IRENE, each coming with its own quota and specific rules. For details about this data spaces [see the doc](http://www-hpc.cea.fr/docs/userdoc-tgcc-public.pdf). 

1) Home directory:

- When you connect to IRENE you end up by default on your "home" directory. The exact path to this directory is stored in the variables HOME or CCFRHOME. The quota on this directory is small (5GB per user), but unlike other directories "home" is saved and not purged, so it must be used to store important files such as source code.

2) Work directory

- When working on IRENE it is preferable to use the "work" folder, which can be accessed by `cd $CCCWORKDIR` or `cd $CCFRWORK`. The quota policy on "work" is of 5 TB and 500 000 files/user. This file system is not purged but it is also not saved, so remember to make regular backups. 
- For a file (or a directory) to be correctly taken into account for the quotas on "work", it must be owned by the project group (gen12462 in our case) and be granted the group rights. To check the ownership and permissions of files in a given directory use the command `ls -l`. To change the group of a file: `chown :<your_group> <your_directory_or_file>` (you can add the `-R`option to do this recursively on a given directory content). To grant group permission to a file use: `chmod g+s <your_directory_or_file>` (more details about this special permissions can be found [here](https://learning.lpi.org/en/learning-materials/010-160/5/5.3/5.3_01/)). Especially if you transfer files with rsync, you might need to add the following options to your rsync command: `--chown=:<your_group>`and  `--chmod=g+s`(or `--chmod=Dg+s` if you transfer a directory).

### Package installation

IRENE does not have internet access, so you will not be able to add packages to julia in the usual way. 

To do so, follow these steps:

1) connect to th-top
2) `module unload julia`
3) `module load julia/1.5` (this is because the IRENE version is 1.5.3, it means that on th-top this will be the version to add packages to to transfer them to IRENE).
4) Remove components of your .julia folder belonging to the version 1.5 `rm -rf .julia/environments/v1.5* ; rm -rf .julia/compiled/v1.5*`(keep a copy in case you need to)
5) Download julia-1.5: `wget https://julialang-s3.julialang.org/bin/linux/x64/1.5/julia-1.5.4-linux-x86_64.tar.gz`
6) Decompress: `tar zxvf julia-1.5.4-linux-x86_64.tar.gz`
7) Launch julia with `julia-1.5.4/bin/julia`, add packages you want and run `]precompile` to precompile everything
8) Exit julia, and transfer your .julia directory to IRENE. If your Julia installation is too heavy you adapt the following commands to transfer the different files on you "work" directory. Otherwise you can install Julia directly on your "home" directory with `rsync -rvazh .julia/ username@irene-amd-fr.ccc.cea.fr:/ccc/cont003/home/unipdide/username/.julia/` (check that there is a .julia folder in IRENE). As an alternative, one may instead issue `rsync -rvazh username@th-top.mpq.univ-paris-diderot.fr:.julia/ .julia/` from the home directory on IRENE.
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

`-T` is the time limit in seconds (if not specified default is 7200). You can run your job with `ccc_msub my_script`, and obtain information about it with `ccc_mpeek job_id`. The command `ccc_mpp` shows all jobs currently running. To show only your own jobs you can do `ccc_mpp -u $USER`.
For more information, type `machine_info` in IRENE or visit this page https://forge.ipsl.jussieu.fr/igcmg_doc/wiki/Doc/ComputingCenters/TGCC/Irene.
