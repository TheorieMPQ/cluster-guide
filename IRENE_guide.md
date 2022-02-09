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

IRENE does not have internet access, so you will not be able to add packages to julia in the usual way. Luckily IRENE runs the same operating system as our th-top (linux x86_64), which means we can transfer installed packages from th-top to IRENE.

In the following we show how to create a minimal fresh installation of Julia and to transfer it to IRENE.

To do so, follow these steps:

1) connect to th-top
2) `module unload julia`
3) Create a temporary directory, for example `mkdir -p temp/.julia`, where we will install the Julia packages (by default it is `.julia/`) and then export the `JULIA_DEPOT_PATH` environment variable, for example   
`export JULIA_DEPOT_PATH="/home/YOUR-USERNAME-ON-TH-TOP/temp/.julia"` (this command only affects the current session and won't overwrite your shell configuration).
4) Download a linux x86_64 build of Julia. ( The version doesn't matter, as julia runs stand-alone-ly. As an example we'll use 1.6 here. ) `wget https://julialang-s3.julialang.org/bin/linux/x64/1.6/julia-1.6.2-linux-x86_64.tar.gz`
5) Decompress: `tar zxvf julia-1.6.2-linux-x86_64.tar.gz`
6) Launch julia with `julia-1.6.2/bin/julia`, add packages you want and run `]precompile` to precompile everything. _Warning: if you need to install Conda, PyCall, or packages that require them, refer to the section below._
7) Exit julia, and transfer your temporary `.julia` directory to IRENE. It is strongly recommended to compress it first: 
`tar -czvf temp/pointjulia.tar.gz -C temp/ .julia` and then transfer with 
with `rsync -rvazh temp/pointjulia.tar.gz USERNAME@irene-amd-fr.ccc.cea.fr:/ccc/cont003/home/unipdide/USERNAME/`. Finally, on IRENE, extract the files with `tar xzvf pointjulia.tar.gz`, which will release the files in your `$HOME/.julia/` folder. As an alternative, one may instead issue `rsync -rvazh username@th-top.mpq.univ-paris-diderot.fr:temp/pointjulia.tar.gz .` (don't forget the final dot in the command) from the home directory on IRENE, and then extract. 
8) Transfer the `julia-1.6.2` directory to IRENE, compress it with `tar` if too slow.
9) Connect to Irene
10) launch julia with `julia-1.6.2/bin/julia`, and precompile everything `]precompile`
11) Check with `using `... that you have your desired packages and have fun!

If later you want to install more packages, just do the following
1) on th-top, `export JULIA_DEPOT_PATH="/home/YOUR-USERNAME-ON-TH-TOP/temp/.julia"`
2) Lauch the julia that you downloaded earlier `julia-1.6.2/bin/julia`, add packages and `]precompile'
3) do step 7) above. (No need to transfer the `julia-1.6.2` folder again, just the `.julia` matters here.)
4) On IRENE, have fun with `julia-1.6.2/bin/julia`!

### Using packages that call `python`
The above procedure works for packages that are purely in Julia, but not for those that require `PyCall` or `Conda`. 
To properly install `PyCall` and `Conda` on IRENE, we need to first install anaconda python, create a conda environment for Julia and then tell Julia where to find them. This can be achieved by the following steps.
1) on th-top, `export JULIA_DEPOT_PATH="/home/YOUR-USERNAME-ON-TH-TOP/temp/.julia"`
2) Lauch the julia that you downloaded earlier `julia-1.6.2/bin/julia`, `]add PyCall Conda` as well as other packages that depend on them.
3) exit Julia and download Anaconda for Linux 64 bit on th-top. In this example we use `wget https://repo.anaconda.com/archive/Anaconda3-2021.05-Linux-x86_64.sh`.
4) transfer the downloaded installer to IRENE with `rsync -rvazh Anaconda3-2021.05-Linux-x86_64.sh USERNAME@irene-amd-fr.ccc.cea.fr:/ccc/cont003/home/unipdide/USERNAME/`
5) do step 7 above, i.e. compress and transfer the temporary `.julia` directory to IRENE and extract over there.
6) on IRENE, run the Anaconda installer with `bash Anaconda3-2021.05-Linux-x86_64.sh`. Note that when the installer asks for the installation directory, it is highly recommended to put `/ccc/work/cont003/gen12462/USERNAME/anaconda3` as the `work` directory has much more disk quota than `home`. Follow the instructions and complete the installation.
7) disconnect from IRENE and reconnect to make the Anaconda installation take effect.
8) check that Anaconda has been properly installed. You can find the path to `conda` (`python`) with `which conda` (resp. `which python`). Note down the full paths.
9) Create a new conda environment for julia with `conda create -n conda_jl --clone root`.
10) lauch your julia `julia-1.6.2/bin/julia` and edit the environment variables according to the paths that you noted down previously. 
 `ENV["CONDA_JL_HOME"] = "/path/to/anaconda/envs/conda_jl"`    
`ENV["PYTHON"] = "/path/to/anaconda/bin/python"`.   
If you followed all the steps so far, the paths should be   
 `ENV["CONDA_JL_HOME"] = "/ccc/work/cont003/gen12462/USERNAME/anaconda3/envs/conda_jl"`    
`ENV["PYTHON"] = "/ccc/work/cont003/gen12462/USERNAME/anaconda3/bin/python"`.   
(these modifications to the environment variables only affect the current session.)
11) `]build Conda`, `]build Pycall` and finally `]precompile`.
12) If there are no errors so far, have fun !

NOTE: after installing Anaconda, the default `env python` will become Anaconda python. This may cause certain `ccc...` scripts that requires `python2` to fail, such as `ccc_quota`. To fix this, we have to remove Anaconda python from the `$PATH` variable.  This can be done (for the current session) by running
```
export PATH=`echo $PATH | tr ":" "\n" | grep -v "anaconda" | tr "\n" ":"`
```
This will temporarily revert the default `python` and `ccc_quota` should work as expected. Note that to show the quota for all filesystems, we need to load them with `module switch dfldatadir/gen12462` before calling `ccc_quota`.

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
#MSUB -n 1

cd && module switch dfldatadir/gen12462 && cd
echo "running a job" 
$HOME/julia-1.6.2/bin/julia $CCCWORKDIR/JULIA_FILE_NAME.jl
```

`$HOME` is the environment variable that stores the path of your home directory. `module switch dfldatadir/gen12462` changes the data space and allows access to the work directory by `$CCCWORKDIR` (which has more space, so it is highly recommanded to put the julia script and its outputs in this directory). Note that there is no need to load Julia as we are using our own downloaded version, just  call the path to the executable (`$HOME/julia-1.6.2/bin/julia` in this example). `-T` is the time limit in seconds (if not specified default is 7200), and `-n` is the number of processors. You can run your job with `ccc_msub my_script`, and obtain information about it with `ccc_mpeek job_id`, kill it with `ccc_mdel job_id`, or see the status of all your jobs with `ccc_mpp -u $(whoami)`. The command `ccc_mpp`is like `qstat` on th-top, but here it is useless without specifying the user since too many jobs are running. 

`-T` is the time limit in seconds (if not specified default is 7200). Note that the maximum wall-time (`-T` value) we can put is `86400` (1 day) by default. To trade in priority for more time we can add a line in the script `#MSUB -Q long` and this extends the maximum wall-time to `259200` (3 days).
 You can run your job with `ccc_msub my_script`, and obtain information about it with `ccc_mpeek job_id`. The command `ccc_mpp` shows all jobs currently running. To show only your own jobs you can do `ccc_mpp -u $USER`.
For more information, type `machine_info` in IRENE or visit this page https://forge.ipsl.jussieu.fr/igcmg_doc/wiki/Doc/ComputingCenters/TGCC/Irene.

