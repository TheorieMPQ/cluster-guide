# cluster-guide
A guide to using th-top.

## Loading modules

Th-Top uses modules to load python and Julia. By default users have only anaconda and julia.
These can be loaded using the following command:
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
conda activate env_name
```
and install packages as you would usually do. 

## Syncronization of directories from th-top to local computer
rsync -avzhe ssh username@th-top.mpq.univ-paris-diderot.fr:/home/username/dir/ /Users/local_username/local_dir/
where username is your name on th-top, local_username is your local username and local_dir is the directory on your local computer.
This command will copy the directory from the remote server (in this case th-top) to the local computer.


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
$ module load julia/1.6.1
```
Load `julia` v1.6.1.

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

To launch a python script with a local conda environment `env_name`, replace 
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
A more complete set of examples and precisions can be found here: https://hpc-uit.readthedocs.io/en/latest/jobs/examples.html.

## Getting information about resources available on each node

One can use the command `sinfo` to gain information about resources. For more info, use `sinfo --help` for a list of commands that will modify the output of `sinfo`.

#### Example
```shell
$ sinfo 

PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
cpu*         up   infinite      4   idle c[1-4]
gpu          up   infinite      2   idle g[1-2]
```

### Jupiter interacting notebook (NOT ACTIVATED FOR NOW)

From your local computer, connect to th-top with the present command:
```
ssh -L 8090:localhost:8090 YOURLOGIN@th-top.mpq.univ-paris-diderot.fr
```
where YOURLOGIN must be replaced by your personal th-top login.

Then, open Chrome/Firefox (not Safari) on your local computer.
Connect via the web browser to `localhost:8090`

Enter login and password for th-top on the login window.

Now you are on Jupyter. 
First, you have to choose how many cores you want to use for your job.

To see the directory tree: `http://localhost:8090/user/YOURLOGIN/tree?`
Note that you can copy files from your local computer just by dragging the icon from your Directory Finder to the Notebook tree.

To see the menu with the Applications (Julia, Python, ...): `http://localhost:8090/user/YOURLOGIN/lab`

**IMPORTANT: When you are done, you should click on Control Panel and then STOP MY SERVER.
Otherwise, the cores will stay occupied !!!!!**  

### Launching interactive jobs

Interactive jobs are specified by the command-line flag `-I`. Some examples can be found hereafter.

#### Examples
```shell
$ qsub -I -l select=1:ncpus=1
```
Start a job in interactive mode (`-I`) on one CPU.

```shell
$ qsub -I -l select=1:ncpus=5:ngpus=1 -l memory
```
Start a job in interactive mode (`-I`) on 5 CPUs and 1 GPU with default memory bounds (4GB).

```shell
$ qsub -I -l select=1:ncpus=10:ngpus=1:mem=10G
```
Start a job in interactive mode (`-I`) on 10 CPUs and 1 GPU with specific maximum memory (10GB).

Unavailable ressources cannot be allocated. e.g. asking for 30 CPUs will cause the job to never start because no node has 30 CPUs.
This is particularily tricky with RAM, you can check the ressources available on any node by refering to the relevant section of this guide.

