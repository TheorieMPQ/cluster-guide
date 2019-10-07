# cluster-guide
A guide to using clusters in general (and th-top in particular)

## Loading modules

### Examples
```shell
$ module load anaconda-python
```
Load `anaconda-python`.

```shell
$ module load julia
```
Load default version of `julia`.

```shell
$ module load julia/1.2.0
```
Load `julia` v1.2.0.

In order to avoid having to do that at each log in, one can append these commands to the `.bash_profile` file in the home directory. For instance:
```shell
$ echo "module load <module_name>" >> ~/.bash_profile
```

## Submitting jobs via `qsub`
Th-top is not like a standard desktop computer, but it is a system composed by a master server managing several computing nodes. 
Running computations on such systems is a bit different than usual, and it will be explained below.
### Examples
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

## Getting information about ressources available on each node

```shell
$ pbsnodes <nodename>
```
for `<nodename>` in `{node0, node1… node3, gpunode0, gpunode1}` (in th-top).

### Example
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
