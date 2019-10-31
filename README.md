# cluster-guide
A guide to using clusters in general (and th-top in particular)

## Loading modules

Th-Top uses modules to load python and Julia. By default users have only python2 and no julia.
These can be loaded using the following command:
```shell
$ module load <module_name>
```

In order to avoid having to do that at each login, one can append these commands to the `.bash_profile` file in the home directory so that they get executed every time you log in. For instance by executing:
```shell
$ echo "module load <module_name>" >> ~/.bash_profile
```

#### Examples
```shell
$ module load anaconda-python
```
Loads the last `python` distribution (`anaconda-python`).

```shell
$ module load julia
```
Load default version of `julia`.

```shell
$ module load julia/1.2.0
```
Load `julia` v1.2.0.

## Submitting jobs via `qsub`
Th-top is not like a standard desktop computer, but it is a system composed by a master server managing 6 nodes, 4 nodes with 20 CPU cores and 2 nodes with 20 CPU cores and 3 GPUs.
Running computations on such systems is a bit different than usual, and it will be explained below.

All nodes and the master share the same folders and filesystem.
When you log in via ssh you will be connected to the master node.

A computation to be executed on th-top is called a job. A job can be standard, if it does not require user's intervention and is completely specified by a series of commands that the computer can run itself, or it can be interactive, letting you execute operations through the terminal like a standard terminal session on your desktop computer or `thX`. When one job is submitted through master, it gets queued till ressources are available and may not start instantly if all nodes are busy.

A complete guide can be found [here](https://wikis.nyu.edu/display/NYUHPC/Copy+of+Tutorial+-+Submitting+a+job+using+qsub).

### Jupiter interacting notebook 

From your local computer, connect to th-top with the present command:
ssh -L 8090:localhost:8090 YOURLOGIN@th-top.mpq.univ-paris-diderot.fr

where YOURLOGIN must be replaced by your personal th-top login.

Then, open Chrome (not Safari) on your local computer.
Connect via the web browser to localhost:8090/hub/login

Enter login and password for th-top on the login window.

Now you are on Jupiter. 
First, you have to choose how many cores you want to use for your job.

To see the directory tree:
http://localhost:8090/user/YOURLOGIN/tree?
Note that you can copy files from your local computer just by dragging the icon from your Directory Finder to the Notebook tree.

To see the menu with the Applications (Julia, Python, ...):
http://localhost:8090/user/YOURLOGIN/lab

** IMPORTANT: When you are done, you should click on Control Panel and then STOP MY SERVER.
Otherwise, the cores will stay occupied !!!!! ** 

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

### Launching batch jobs

A batch job is a job that runs autonomously without requiring any human intervention. To submit a batch job, a small bash file containing all the commands to be executed together with information about the ressources to be allocated by master is to be created.

This file, here called e.g. `myjob`, should look somehow like the following example:
```
#!/bin/bash
#PBS -N juliaTest
#PBS -o juliaTest.out
#PBS -e juliaTest.err
#PBS -l nodes=1:ncpus=10:mem=XXXG

echo "running a job!"
    
julia  ~/myscript.jl
```

The job can then be launched by running
```shell
$ qsub myjob
```
Its status can be checked with
```shell
$ qstat
```

## Getting information about ressources available on each node

```shell
$ pbsnodes <nodename>
```
for `<nodename>` in `{node0, node1… node3, gpunode0, gpunode1}` (in th-top).

#### Example
```shell
$ pbsnodes gpunode1

gpunode1
     Mom = gpunode1.alineos.net
     ntype = PBS
     state = free
     pcpus = 24
     Priority = 100
     resources_available.arch = linux
     resources_available.host = gpunode1
     resources_available.mem = 97586176kb
     resources_available.ncpus = 24
     resources_available.ngpus = 3
     resources_available.vmem = 105908224kb
     resources_available.vnode = gpunode1
     resources_available.vntype = gpu_node
     resources_assigned.accelerator_memory = 0kb
     resources_assigned.hbmem = 0kb
     resources_assigned.mem = 0kb
     resources_assigned.naccelerators = 0
     resources_assigned.ncpus = 0
     resources_assigned.ngpus = 0
     resources_assigned.vmem = 0kb
     resv_enable = True
     sharing = default_shared
     last_state_change_time = Fri Aug  9 12:58:31 2019
     last_used_time = Wed Sep 25 18:33:42 2019
```
