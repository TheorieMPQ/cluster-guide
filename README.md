# cluster-guide
A guide to using th-top.


## connecting to the server
From your local terminal (linux, macOS or Windows 10/11), connect to the cluster via ssh:
```shell
ssh ${username}@th-top.mpq.univ-paris-diderot.fr
```
where `${username}` is your login for the cluster. (If you don't have one, go ask the IT admin to create an acount for you.)

## transfer of directories from th-top to local computer
Linux and mac users can use the command `rsync` (not included in Windows) or `scp` to transfer files. Note that there exists also graphical apps for file transfer.   

Example:
```shell
[from your local terminal]# rsync -avzhe ssh ${username}@th-top.mpq.univ-paris-diderot.fr:/home/${username}/dir/ /${local_dir}/ 
```
where `${username}` is your login on th-top and `${local_dir}` is the directory on your local computer. This command will copy the directory from the remote server (in this case th-top) to the local computer.


## Loading modules

Th-Top uses the environment module system `Lmod`. It basically sets environment variables (such as `PATH`) in real time to control whether a software installed on the cluster can be loaded in the user's environment.  

To see the available modules, use the command 
```shell
module av
```
These can be loaded using

```shell
$ module load <module_name>
```

In order to avoid having to do that at each login, one can append these commands to the `.bash_profile` file in the home directory so that they get executed every time you log in. For instance by executing:
```shell
$ echo "module load <module_name>" >> ~/.bash_profile
```

## Using `python`

To use python in your `/home` directory, the best thing to do is to install a virtual environment with conda, whose location will be in your `home`. If you want to do so, the first time you log in, run 
```
module load anaconda
conda init
```
log out, and log back in to initialize `conda`. To create a local environment, execute the following lines of code:

```
conda create -p /home/username/env_name
```
with `env_name` the name of your `conda` environment (you can of course specify the path you want). This allows you to manage your packages locally and install anything you want without perturbing anaconda module installed for everyone. Then, you can enter your environment using

```
conda activate /home/username/env_name
```
and install packages as you would usually do. 

#### Examples
```shell
$ module load anaconda
```
Loads anaconda, which contains python and conda.

```shell
$ module load julia
```
Load default version of `julia`.

```shell
$ module load julia/1.6.2
```
Load `julia` v1.6.2.

## Submitting jobs via `sbatch`
th-top is not like a standard desktop computer, but it is a system composed by a master server managing 6 nodes, 4 nodes with 20 CPU cores and 2 nodes with 20 CPU cores and 3 GPUs.
Running computations on such systems is a bit different than usual, and it will be explained below.

All nodes and the master share the same folders and filesystem.
When you log in via ssh you will be connected to the master node.

A computation to be executed on th-top is called a job. A job can be standard, if it does not require user's intervention and is completely specified by a series of commands that the computer can run itself, or it can be interactive, letting you execute operations through the terminal like a standard terminal session on your desktop computer or th-top. When one job is submitted through master, it gets queued till ressources are available and may not start instantly if all nodes are busy.


### Launching batch jobs

A batch job is a job that runs autonomously without requiring any human intervention. To submit a batch job, a small bash file containing all the commands to be executed together with information about the ressources to be allocated by master is to be created.

This file, here called e.g. `myjob`, should look somehow like the following example using julia:
```
#!/bin/bash

#SBATCH --job-name=name 
#SBATCH -o name.out #file in which the output of code.jl is written
#SBATCH -e name.err #file in which the error log is written
#SBATCH -N 1 #number of compute nodes used 
#SBATCH -n 2 #number of processes used (don't specify if you don't want to parallelize)
#SBATCH --mem=10G #total memory allocated

echo "running a job"
module load julia
julia /home/username/code.jl
```
To use the GPU queue, add a line with `#SBATCH --partition=gpu` and if for example you need 2 gpus, add another line `#SBATCH --gres=gpu:2`. To launch a python script with a local conda environment `env_name`, replace 
```
module load julia
julia /home/username/code.jl
```
with
```
source /opt/ohpc/pub/apps/anaconda3/etc/profile.d/conda.sh
conda activate env_name
python /home/username/code.py
```

The job can then be launched by running
```shell
$ sbatch myjob
```
Its status can be checked with
```shell
$ squeue
```

which gives something like
```shell
JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
  424       cpu jupyterh username  R       9:44      1 c1
```

The example job above has JOBID `424`, which is used in case you need to cancel your job: `scancel 424` (replace the number with your own JOBID).

A more complete set of examples and precisions can be found here: https://slurm.schedmd.com/ (Slurm documentation) and https://hpc-uit.readthedocs.io/en/latest/jobs/examples.html (Documentation of UiT's HPC).

## Getting information about resources available on each node

One can use the command `sinfo` to gain information about the state of the nodes. For more info, use `sinfo --help` for a list of commands that will modify the output of `sinfo`.

#### Example
```shell
$ sinfo 

PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
cpu*         up   infinite      4   idle c[1-4]
gpu          up   infinite      2   idle g[1-2]
```
To show the detailed resources of all nodes, one can issue `scontrol show nodes`.   
To see a specific node, for example `c1`, use `scontrol show node c1`. This will print something like
```
NodeName=c1 Arch=x86_64 CoresPerSocket=10
   CPUAlloc=0 CPUTot=20 CPULoad=0.00
   AvailableFeatures=(null)
   ActiveFeatures=(null)
   Gres=(null)
   NodeAddr=c1 NodeHostName=c1 Version=20.11.9
   OS=Linux 4.18.0-372.26.1.el8_6.x86_64 #1 SMP Tue Sep 13 18:09:48 UTC 2022
   RealMemory=120000 AllocMem=0 FreeMem=126907 Sockets=2 Boards=1
   State=IDLE ThreadsPerCore=1 TmpDisk=0 Weight=1 Owner=N/A MCS_label=N/A
   Partitions=cpu
   BootTime=2022-11-09T15:57:17 SlurmdStartTime=2022-11-09T15:59:01
   CfgTRES=cpu=20,mem=120000M,billing=20
   AllocTRES=
   CapWatts=n/a
   CurrentWatts=0 AveWatts=0
   ExtSensorsJoules=n/s ExtSensorsWatts=0 ExtSensorsTemp=n/s
   Comment=(null)
```

### Jupiter interacting notebook

From your local computer, connect to th-top with the present command:
```
ssh -L 8090:localhost:8090 ${username}@th-top.mpq.univ-paris-diderot.fr
```
where `${username}` must be replaced by your personal th-top login.

Then, open a browser on your local computer.
Connect via the web browser to `localhost:8090`

Enter login and password for th-top on the login window.

Now you are on Jupyter. 
First, you have to choose how many resources you want to use for your job.

To see the directory tree: `http://localhost:8090/user/username/tree?`
Note that you can copy files from your local computer just by dragging the icon from your Directory Finder to the Notebook tree.

To see the menu with the Applications (Julia, Python, ...): `http://localhost:8090/user/username/lab`

**IMPORTANT: When you are done, you should click on Control Panel and then STOP MY SERVER.
Otherwise, the cores will stay occupied !!!!!**  

### Launching interactive jobs

Interactive jobs are specified by the command `srun` and typically in combination with `bash -i`. `srun` accepts the same parameters as `sbatch`. For example, to run an interactive bash session with 1 cpu and 1G memory for a maximum of 1 hour, you can issue

```shell
srun -n 1 --mem=1G --time=01:00:00 --pty bash -i
```
and if resources permit you will be logged onto a worker node where the session runs. The job can be terminated by the command `exit` or the key combination `ctrl`+`D` from the interactive shell.

Finally, note that unavailable ressources cannot be allocated. e.g. asking for 30 CPUs will cause the job to never start because no node has 30 CPUs.
This is particularily tricky with RAM, you can check the ressources available on any node by refering to the relevant section of this guide.

