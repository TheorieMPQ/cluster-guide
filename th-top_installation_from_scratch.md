# Table of contents

- [th-top reloaded](#th-top-reloaded)
  - [Preliminary](#preliminary)
  - [Install OpenHPC Components](#install-openhpc-components)
  - [Provisioning setup](#provisioning-setup)
  - [Prepare system images for CPU nodes](#prepare-system-images-for-cpu-nodes)
  - [Prepare system images for GPU nodes](#prepare-system-images-for-gpu-nodes)
  - [Install OpenHPC Development Components](#install-openhpc-development-components)
  - [Booting the nodes](#booting-the-nodes)
  - [Test job](#test-job)
- [How to add new packages (for example a new Julia version) ?](#how-to-add-new-packages-for-example-a-new-julia-version-)
- [What to do after a power cut ?](#what-to-do-after-a-power-cut-)
- [JupyterHub installation](#jupyterhub-installation)
- [GPU tuning](#gpu-tuning)


# th-top reloaded

The software layer of our th-top cluster has been redeployed from scratch thanks to the
OpenHPC recipe https://github.com/openhpc/ohpc/releases/download/v2.5.GA/Install_guide-Rocky8-Warewulf-SLURM-2.5-x86_64.pdf



We summarize below by heavily following this guide the key steps performed during the installation.

## Preliminary 

### How the cluster works

Our cluster consists of a master node and 6 worker nodes (2 of which having GPUS), where a node means a computer with a lot of computational resources (such as CPUs and RAMs), that are connected and coordinated by the master node to perform heavy calculations together. We adopted a modern solution where only the master node has a persistent OS (in our case Rocky Linux 8.6) whereas the worker nodes don't: they don't need hard drives and boot off via the network directly from a image hosted on the master and their OS runs completely in the RAM (just like a live disk session given by a linux installation disk), which is therefore referred to as a "stateless" configuration. This can be easily achieved using the "Warewulf" program.   

The physical architecture is very well sumarized in Figure 1 of the guide linked above. The master node has two network interfaces, one connected to the public network for us to access the cluster, and the other connected to a private internal network for the provision and management of the nodes. Each node also has two network interfaces, one used to receive provision information (such as the boot image and shared folders) from the master and the other dedicated to the power management and monitoring of the nodes via the BMC (baseboard management controllers, they are actually independent single-board computers living on the nodes allowing us to, for example, reboot the nodes or change their bios settings without having physical access to them and regardless of their system state).

### Install OS on the master node
To follow the OpenHPC recipe, we will install the Rocky8.6 (8.5 is outdated) distribution onto the master node. You will need to find the dvd image
(for example download here: https://download.rockylinux.org/pub/rocky/8/isos/x86_64/Rocky-8.6-x86_64-minimal.iso) and then make a bootable USB flash drive from it (for example, Windows users can use balenaEtcher). Then go to the server room (at the 6th floors of the Grands Moulins building), plug in the USB and reboot the master node. Don't forget to plug in a keyboard (and a mouse too, a USB hub will help) and keep pressing F11 until you are prompted to select a boot device. Select the USB and start the installation. 

**Remember to set a STRONG password for root. You need to be logged in as root throughout this guide.**

#### hostname and ip setting

During the installation we can setup the hostname ("th-top") and the network interfaces. You should be able to identify which of the interface is for the public network (it will show a public ip automatically upon connection) and the others one is for the internal network with the other nodes. Take note of the names of the interfaces. During our installation we found the internal interface to be called "ens18f1".   

Set up two static IP addresses for the internal interface:   
1) 192.168.1.250, subnet mask: 255.255.255.0 This subnet will be used for provisioning the nodes  
2) 192.168.3.250, subnet mask: 255.255.255.0 This subnet contains the BMC ip address of the nodes.
 
**Make sure both interfaces are set to automatically connect.**
 
If you forgot to set those during the installation, it is also possible to configure afterwards, for example via the command `nmtui`, which is very straightforward to use.


*After the OS installation you may leave the server room and continue via ssh from a more comfortable place. Log in as root.*

### Definition of a few useful variables

To facilitate the cluster deployment, it is highly recommended to pre-define a text file containing a few useful variables. **This will allow you to copy the commands from the guide and run them directly in the terminal, after sourcing this text file.**

```shell

# ---------------------------
# SMS (system management server, same thing as "master") node settings
# ---------------------------

# Hostname of the master node
sms_name="th-top"

# Static IP address of master node on the the internal network
sms_ip="192.168.1.250"

# Internal ethernet interface name on master node (you need to note this down when installing the OS to the master node. In our case we found the name to be "ens18f1". Adapt it to your case accordingly.)
sms_eth_internal="ens18f1"

# Subnet netmask for internal cluster network
internal_netmask="255.255.255.0"

# server for time synchronization
ntp_server="0.centos.pool.ntp.org"

# BMC user credentials for use by IPMI (we will need these when performing remote hard reboots for the worker nodes)
bmc_username="ADMIN"
bmc_password="ADMIN"

# Derised name of the provisioning interface used by compute hosts
eth_provision="eth0"

# -------------------------
# compute node settings (for the 4 nodes without GPUs)
# -------------------------

# total number of computes
num_computes="4"

# desired compute hostnames
c_name[0]="c1"
c_name[1]="c2"
c_name[2]="c3"
c_name[3]="c4"

# regex and starting prefix that matches defined compute hostnames
compute_regex="c*"
compute_prefix="c"


# desired compute node IP addresses
c_ip[0]="192.168.1.100"
c_ip[1]="192.168.1.101"
c_ip[2]="192.168.1.102"
c_ip[3]="192.168.1.103"

# compute node MAC addreses for provisioning interface
c_mac[0]="00:25:90:f9:73:46"
c_mac[1]="00:25:90:f9:73:42"
c_mac[2]="00:25:90:f9:73:3c"
c_mac[3]="00:25:90:f9:73:3e"

# compute node BMC addresses
c_bmc[0]="192.168.3.100"
c_bmc[1]="192.168.3.101"
c_bmc[2]="192.168.3.102"
c_bmc[3]="192.168.3.103"

# -------------------------
# GPU node settings (for the 2 nodes with GPUs)
# -------------------------

# we have 2 GPU nodes which will be named g1 and g2
num_gpus="2"

gpu_regex="g*"
gpu_prefix="g"

g_name[0]="g1"
g_name[1]="g2"

g_ip[0]="192.168.1.200"
g_ip[1]="192.168.1.201"

g_mac[0]="ac:1f:6b:ba:ec:7a"
g_mac[1]="ac:1f:6b:ba:ec:de"

g_bmc[0]="192.168.3.200"
g_bmc[1]="192.168.3.201"


```

In practice, I put these into a file named `input.local` in my (root) home folder `/root/`, such that when I do `source /root/input.local` these variables will be assigned.

### make sure the hostname is resolvable:
```
echo ${sms_ip} ${sms_name} >> /etc/hosts
```

### disable SELinux:

`nano /etc/sysconfig/selinux `
change the corresponding line in the file to 
```
SELINUX=disbaled
```

reboot the master with the command `reboot`. This will take a few minutes. **When you connect again don't forget to `source /root/input.local` again**. Verify that SELinux has been indeed disabled with the command `sestatus`.

### regarding the firewall
The OpenHPC guide recommends turning off the firewall with 
```
systemctl disable firewalld
systemctl stop firewalld
```

However, this makes our cluster more vulnerable since it is exposed on the public network. We can do instead 
`firewall-cmd --permanent --change-zone=${sms_eth_internal} --zone=trusted` to allow internal network traffic free from firewall, while keeping the firewall service running.

## Install OpenHPC Components  

```
yum -y install http://repos.openhpc.community/OpenHPC/2/EL_8/x86_64/ohpc-release-2-1.el8.x86_64.rpm
```

```
yum -y install dnf-plugins-core
```

```
yum config-manager --set-enabled powertools

```

```
yum -y install ohpc-base
```

```
yum -y install ohpc-warewulf
```

### NTP protocol for time synchronization
```
systemctl enable chronyd.service
```

```
echo "local stratum 10" >> /etc/chrony.conf
```

```
echo "server ${ntp_server}" >> /etc/chrony.conf
```

```
echo "allow all" >> /etc/chrony.conf
```

```
systemctl restart chronyd
```

### SLURM job scheduler

```
yum -y install ohpc-slurm-server
```

```
cp /etc/slurm/slurm.conf.ohpc /etc/slurm/slurm.conf
```

```
perl -pi -e "s/ControlMachine=\S+/ControlMachine=${sms_name}/" /etc/slurm/slurm.conf
```

The file `/etc/slurm/slurm.conf` needs to be modified to match the cluster's resources. The file content we put is as follows:
```shell

ClusterName=mpq-th-top
ControlMachine=th-top
SlurmUser=slurm
SlurmctldPort=6817
SlurmdPort=6818
AuthType=auth/munge
StateSaveLocation=/var/spool/slurm/ctld
SlurmdSpoolDir=/var/spool/slurm/d
SwitchType=switch/none
MpiDefault=none
SlurmctldPidFile=/var/run/slurmctld.pid
SlurmdPidFile=/var/run/slurmd.pid
SlurmctldTimeout=300
SlurmdTimeout=300
InactiveLimit=0
MinJobAge=300
KillWait=30
Waittime=0
SchedulerType=sched/backfill
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory
SlurmctldDebug=info
SlurmctldLogFile=/var/log/slurmctld.log
SlurmdDebug=info
SlurmdLogFile=/var/log/slurmd.log


ProctrackType=proctrack/cgroup
TaskPlugin=task/affinity,task/cgroup
PropagateResourceLimitsExcept=MEMLOCK
JobCompType=jobcomp/filetxt
Epilog=/etc/slurm/slurm.epilog.clean



GresTypes=gpu

NodeName=c[1-4] CPUs=20 CoresPerSocket=10 ThreadsPerCore=1 RealMemory=120000 State=UNKNOWN

NodeName=g[1-2] Gres=gpu:3 CPUS=24 CoresPerSocket=12 ThreadsPerCore=1 RealMemory=90000 State=UNKNOWN

PartitionName=cpu Nodes=c[1-4] Default=YES MaxTime=INFINITE State=UP DefMemPerCPU=512 Oversubscribe=NO
PartitionName=gpu Nodes=g[1-2] MaxTime=INFINITE State=UP DefMemPerCPU=512 Oversubscribe=NO

SlurmctldParameters=enable_configless
ReturnToService=2


```

create another file `gres.conf` in the same folder to specify the gpu resources `nano /etc/slurm/gres.conf`:
```
NodeName=g1 Name=gpu File=/dev/nvidia[0-2]
NodeName=g2 Name=gpu File=/dev/nvidia[0-2]
```

as we specified in `ProctrackType` and `TaskPlugin`, slurm will use `cgroup` (a linux kernel feature) to track and limit the computational resources used by the jobs. Therefore, we should create another file `cgroup.conf` to specify what we want slurm to constrain `nano /etc/slurm/cgroup.conf`:
```
CgroupAutomount=yes

ConstrainCores=yes
ConstrainRAMSpace=yes
ConstrainSwapSpace=yes
ConstrainDevices=yes
```


## Provisioning setup

```
perl -pi -e "s/device = eth1/device = ${sms_eth_internal}/" /etc/warewulf/provision.conf

systemctl enable httpd.service

systemctl restart httpd

systemctl enable dhcpd.service

systemctl enable tftp.socket

systemctl start tftp.socket

```




## Prepare system images for CPU nodes

Create a folder to store the temporary files for the VNFS (virtual node file system) image compilation.
We first do it for the CPU nodes, and later for the GPU nodes.

```
export CHROOT=/opt/ohpc/admin/images/rocky8.6
```

```
wwmkchroot -v rocky-8 $CHROOT
```

Install packages into the image

```
dnf -y --installroot $CHROOT install epel-release
cp -p /etc/yum.repos.d/OpenHPC*.repo $CHROOT/etc/yum.repos.d

yum -y --installroot=$CHROOT install ohpc-base-compute

cp -p /etc/resolv.conf $CHROOT/etc/resolv.conf


```

```
cp /etc/passwd /etc/group $CHROOT/etc
```
(press `y` and enter to confirm overwrite for both files)


```
yum -y --installroot=$CHROOT install ohpc-slurm-client
chroot $CHROOT systemctl enable munge
```
```
chroot $CHROOT systemctl enable slurmd

perl -pi -e "s/network.target/network-online.target/" $CHROOT/usr/lib/systemd/system/slurmd.service
```



```
echo SLURMD_OPTIONS="--conf-server ${sms_ip}" > $CHROOT/etc/sysconfig/slurmd

yum -y --installroot=$CHROOT install chrony
echo "server ${sms_ip} iburst" >> $CHROOT/etc/chrony.conf

yum -y --installroot=$CHROOT install kernel-`uname -r`
yum -y --installroot=$CHROOT install lmod-ohpc

```

### Initialize warewulf database and ssh_keys
```
wwinit database
wwinit ssh_keys
```

### Add NFS client mounts of /home and /opt/ohpc/pub to base image
```
echo "${sms_ip}:/home /home nfs nfsvers=3,nodev,nosuid 0 0" >> $CHROOT/etc/fstab

echo "${sms_ip}:/opt/ohpc/pub /opt/ohpc/pub nfs nfsvers=3,nodev 0 0" >> $CHROOT/etc/fstab

echo "/home *(rw,no_subtree_check,fsid=10,no_root_squash)" >> /etc/exports

echo "/opt/ohpc/pub *(ro,no_subtree_check,fsid=11)" >> /etc/exports


exportfs -a

systemctl restart nfs-server

systemctl enable nfs-server

```

### Import files into warewulf database

```
wwsh file import /etc/passwd

wwsh file import /etc/group

wwsh file import /etc/shadow

wwsh file import /etc/munge/munge.key

```

### build bootstrap image


```
wwbootstrap `uname -r`
```



### compile VNFS (virtual node file system) image for the CPU nodes

```
wwvnfs --chroot $CHROOT
```
### register cpu nodes for provisioning

```
echo "GATEWAYDEV=${eth_provision}" > /tmp/network.$$

wwsh -y file import /tmp/network.$$ --name network

wwsh -y file set network --path /etc/sysconfig/network --mode=0644 --uid=0

for ((i=0; i<$num_computes; i++)) ; do
    wwsh -y node new ${c_name[i]} --ipaddr=${c_ip[i]} --hwaddr=${c_mac[i]} -D ${eth_provision}
done

wwsh -y provision set "${compute_regex}" --vnfs=rocky8.6 --bootstrap=`uname -r` \
--files=dynamic_hosts,passwd,group,shadow,munge.key,network


systemctl restart dhcpd
wwsh pxe update

```

### enable munge and slurm on master
```
systemctl enable munge

systemctl enable slurmctld
 
```
The slurm service script constains a bug that needs to fixed:

```
perl -pi -e "s/network.target/network-online.target/" /usr/lib/systemd/system/slurmctld.service
```

```
 
systemctl start munge
 
systemctl start slurmctld

```



## Prepare system images for GPU nodes

First repeat the steps for the VNFS preparation as for the CPU nodes


```
export CHROOT=/opt/ohpc/admin/images/rocky8.6gpu
```

```
wwmkchroot -v rocky-8 $CHROOT
```

Install packages into the image

```
dnf -y --installroot $CHROOT install epel-release
cp -p /etc/yum.repos.d/OpenHPC*.repo $CHROOT/etc/yum.repos.d

yum -y --installroot=$CHROOT install ohpc-base-compute

cp -p /etc/resolv.conf $CHROOT/etc/resolv.conf


```

```
cp /etc/passwd /etc/group $CHROOT/etc
```
(press `y` and enter to confirm overwrite for both files)


```
yum -y --installroot=$CHROOT install ohpc-slurm-client
chroot $CHROOT systemctl enable munge

```

```
chroot $CHROOT systemctl enable slurmd

perl -pi -e "s/network.target/network-online.target/" $CHROOT/usr/lib/systemd/system/slurmd.service


```


```
echo SLURMD_OPTIONS="--conf-server ${sms_ip}" > $CHROOT/etc/sysconfig/slurmd

yum -y --installroot=$CHROOT install chrony
echo "server ${sms_ip} iburst" >> $CHROOT/etc/chrony.conf

yum -y --installroot=$CHROOT install kernel
yum -y --installroot=$CHROOT install lmod-ohpc

```


### Add NFS client mounts of /home and /opt/ohpc/pub to base image
```
echo "${sms_ip}:/home /home nfs nfsvers=3,nodev,nosuid 0 0" >> $CHROOT/etc/fstab

echo "${sms_ip}:/opt/ohpc/pub /opt/ohpc/pub nfs nfsvers=3,nodev 0 0" >> $CHROOT/etc/fstab


```


### Add Nvidia drivers

install kernel headers

```
dnf -y install kernel-devel-$(uname -r) kernel-headers-$(uname -r)

dnf -y --installroot $CHROOT install kernel-devel-$(uname -r) kernel-headers-$(uname -r)

```

```
dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel8/x86_64/cuda-rhel8.repo

dnf clean all

dnf -y module install nvidia-driver:latest-dkms

dnf -y install cuda

dnf -y install nvidia-gds

dnf -y install freeglut-devel libX11-devel libXi-devel libXmu-devel \
    make mesa-libGLU-devel freeimage-devel


```
same thing for nodes' image:

```
cp -p /etc/yum.repos.d/cuda*.repo $CHROOT/etc/yum.repos.d

dnf -y --installroot $CHROOT module install nvidia-driver:latest-dkms

dnf -y --installroot $CHROOT install cuda

dnf -y --installroot $CHROOT install nvidia-gds

dnf -y --installroot $CHROOT install freeglut-devel libX11-devel libXi-devel libXmu-devel \
    make mesa-libGLU-devel freeimage-devel
```

enable NVIDIA Persistence Daemon on nodes
```

chroot $CHROOT systemctl enable nvidia-persistenced

```


### compile VNFS (virtual node file system) image for the GPU nodes

```
wwvnfs --chroot $CHROOT

```


### register gpu nodes for provisioning

```


for ((i=0; i<$num_gpus; i++)) ; do
    wwsh -y node new ${g_name[i]} --ipaddr=${g_ip[i]} --hwaddr=${g_mac[i]} -D ${eth_provision}
done

wwsh -y provision set "${gpu_regex}" --vnfs=rocky8.6gpu --bootstrap=`uname -r` \
--files=dynamic_hosts,passwd,group,shadow,munge.key,network


systemctl restart dhcpd
wwsh pxe update

```




## Install OpenHPC Development Components

```
yum -y install ohpc-autotools

yum -y install EasyBuild-ohpc

yum -y install hwloc-ohpc

yum -y install spack-ohpc

yum -y install valgrind-ohpc

yum -y install gnu9-compilers-ohpc

yum -y install openmpi4-gnu9-ohpc mpich-ofi-gnu9-ohpc

yum -y install ohpc-gnu9-perf-tools

yum -y install lmod-defaults-gnu9-openmpi4-ohpc


```

### Modulefiles

Note that here we don't need to repeat the installation for CHROOT because they are installed into a path shared with the nodes via NFS `/opt/ohpc/pub/`, and the installation automatically created the modulefiles in `/opt/ohpc/pub/modulefiles` such that their paths can be added into the environment by `module load ` follow by the module name. To see the available module, use `module av`.   

We should therefore create a modulefile for the cuda path (check nvidia documentation https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html#post-installation-actions and lmod documentation https://lmod.readthedocs.io/).

When installing other apps (such as Julia), it is recommended to put them under the shared folder `/opt/ohpc/pub/apps/` and then write corresponding module files for each. See example of installing Julia below.

## Booting the nodes

Reboot the master node with `reboot`.   
wait a few minutes and connect again as root. `source input.local` again.

At this point you should be able to boot up the nodes into the vnfs. The IPMI interface (for BMC) allows us to "push the power button" of the nodes remotely:

```


for ((i=0; i<${num_computes}; i++)) ; do
    ipmitool -I lanplus -H ${c_bmc[$i]} -U ${bmc_username} -P ${bmc_password} chassis power on
    ipmitool -I lanplus -H ${c_bmc[$i]} -U ${bmc_username} -P ${bmc_password} chassis power cycle
done

for ((i=0; i<${num_gpus}; i++)) ; do
    ipmitool -I lanplus -H ${g_bmc[$i]} -U ${bmc_username} -P ${bmc_password} chassis power on
    ipmitool -I lanplus -H ${g_bmc[$i]} -U ${bmc_username} -P ${bmc_password} chassis power cycle
done

```


in another few minutes you should be able to connect to the nodes, which run the provisioned OS. A useful command for parallel ssh is `pdsh`. For example:
`pdsh -w c[1-4],c[1-2] uptime` runs the command `uptime` in all the nodes specified after `-w`.
You will see some output like
```
# pdsh -w c[1-4],g[1-2] uptime
c4:  13:09:14 up 1 day, 41 min,  0 users,  load average: 0.00, 0.00, 0.00
c1:  13:09:14 up 1 day, 41 min,  0 users,  load average: 1.00, 1.01, 1.06
c2:  13:09:14 up 1 day, 43 min,  0 users,  load average: 0.00, 0.00, 0.00
g1:  13:09:14 up 1 day, 42 min,  0 users,  load average: 6.08, 6.81, 6.47
c3:  13:09:14 up 1 day, 41 min,  0 users,  load average: 0.00, 0.00, 0.00
g2:  13:09:14 up 1 day, 41 min,  0 users,  load average: 0.31, 0.16, 0.16

```
(Note that if you issue this command too early the nodes may not be fully booted yet and the connection could be refused. Be patient ...).

Of course, you can also directly ssh into one of the nodes and if everything went correctly going from master to a worker node shouldn't require password. For example, check whether slurmd service is running with

```
ssh c1 systemctl status slurmd
● slurmd.service - Slurm node daemon
   Loaded: loaded (/usr/lib/systemd/system/slurmd.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2022-11-04 14:13:53 EDT; 1min 2s ago
 Main PID: 1363 (slurmd)
    Tasks: 1
   Memory: 1.0M
   CGroup: /system.slice/slurmd.service
           └─1363 /usr/sbin/slurmd -D --conf-server 192.168.1.250

Nov 04 14:13:53 c1 systemd[1]: Started Slurm node daemon.
```
and you should see `active (running)`. 



**Note that when the master reboots, the worker nodes won't automatically reboot. You should issue a manual reboot for the nodes with** `pdsh -w c[1-4],g[1-2] reboot`. If the nodes are not reachable via ssh then use the ipmitool as done above to issue a hard power reset.

Check the slurm status with `sinfo`. If the node `STATE` is `down` then you need to bring them back by 
```
scontrol update nodename=c[1-4],g[1-2] state=resume
```

## Test job

It is never recommended to run MPI jobs as root, so let's create another user with less privilege.

`useradd -m test`

Do the following whenever adding users to propagate updated user credential files to nodes:
```
wwsh file resync passwd shadow group

pdsh -w c[1-4],g[1-2] 'rm -rf /tmp/.wwgetfile*  &&  /warewulf/bin/wwgetfiles'

```

**Alternatively**, I added a cron job that runs every 5 minutes by issueing
```
echo '*/5 * * * * root wwsh file resync passwd shadow group' > /etc/cron.d/resync_ww_files
```
which updates the three files in warewulf database. The nodes themselves also have a cron job (already configured by warewulf) to fetch the files at 5-minute intervals. So if you are not impatient, in at most 10 minutes the user accounts should have been updated across the nodes.

now switch to "test" user:
```
su - test
```
(**Don't forget the `-`**).

Compile a "hello, world" MPI program:
```
mpicc -O3 /opt/ohpc/pub/examples/mpi/hello.c
```

Submit an interactive job to slurm running on 8 cpus from 2 nodes:
```
salloc -n 8 -N 2
```

run it:
```
prun ./a.out
```
and you should see something like 
```
[prun] Master compute host = c1
[prun] Resource manager = slurm
[prun] Launch cmd = mpiexec.hydra -bootstrap slurm ./a.out
Hello, world (8 procs total)
--> Process # 0 of 8 is alive. -> c1
--> Process # 4 of 8 is alive. -> c2
--> Process # 1 of 8 is alive. -> c1
--> Process # 5 of 8 is alive. -> c2
--> Process # 2 of 8 is alive. -> c1
--> Process # 6 of 8 is alive. -> c2
--> Process # 3 of 8 is alive. -> c1
--> Process # 7 of 8 is alive. -> c2
```



# Congratulations! You now have a working cluster!

---

# How to add new packages (for example a new Julia version) ?

As the cluster uses the `Lmod` Environment Module System to modify environment variables such as `PATH` on the fly using `module load ...` commands, when installing new apps one should make use of this system instead of including the installation directly into the `PATH`.   

For example, in order to install julia 1.8.5, one can proceed as follows: 
1. download and extract the Julia Linux Binaries as per their official guidelines
```
wget https://julialang-s3.julialang.org/bin/linux/x64/1.8/julia-1.8.5-linux-x86_64.tar.gz
```

```
tar -xzvf julia-1.8.5-linux-x86_64.tar.gz
```

2. copy the extracted folder to `/opt/ohpc/pub/apps/`(note that this is our conventional path for adding new apps on this cluster)
```
sudo cp -r julia-1.8.5 /opt/ohpc/pub/apps/
```

3. create a modulefile for this app, which basically indicates how the environment variables should be modified in order to "load" this app so that we can call it by its name. In the case of julia, it is simply done by setting the `PATH` variable to include the path where we have put julia.  

To proceed, first create a modulefile for the new app:
```
sudo nano /opt/ohpc/pub/modulefiles/julia/1.8.5
```
and we put the following in this text file:
```
#%Module1.0#####################################################################

proc ModulesHelp { } {

    puts stderr " "
    puts stderr "This module loads the program"
    puts stderr "Julia."
    puts stderr "\nVersion 1.8.5\n"

}
module-whatis "Name: Julia"
module-whatis "Version: 1.8.5"

set     version                     1.8.5

prepend-path    PATH                /opt/ohpc/pub/apps/julia-1.8.5/bin/
```

save and exit, and now we are all set ! To check that it works, issue `module avail` and you will see `julia/1.8.5    (D)` as one of the options. Note that the module system automatically sets the default version (marked by `(D)`) to be the highest one, which can then be loaded by `module load julia`. To specify a version to load, one should use `module load julia/1.8.5` instead, which is the recommended way. 

4. (Optional, for Jupyter users) If a user (not `root`) wants to use the new julia version in Jupyter lab, he/she should first load this julia version with `module load julia/1.8.5`, then launch `julia` from the master node, press `]` to enter the Pkg REPL mode, and issue `add IJulia` to install the kernel, and finally do `build IJulia` to build the kernel. Then, if Jupyter is installed on the system, the new Julia kernel will show up as one of the available options when launching a notebook.


# What to do after a power cut ?

As our university sometimes does sudden power cuts to the server room ~and we don't have the money to invest in a UPS~, the cluster will not function properly in such scenario: the power cut will cause all the nodes (master and workers) to reboot at the same time, while the provisioning service (hosted on master) won't be ready until the master node is fully up. Therefore, the PXE boot on the worker nodes will fail and they will attempt to boot from local hard drives (which contain the old OS left by the previous life of the cluster) and we will lose ssh access to them. There are also chances that the master node's OS be affected by the forced reboot. I would therefore recommend perform another gentle reboot to the master (i.e. `reboot` via ssh) to have a clean start. Once the master is up, we then hard-reboot the worker nodes via IPMI. To automate this second step, I wrote a script `/usr/local/sbin/hard-reboot-nodes.sh` that will power cycle the worker nodes and created a service `/etc/systemd/system/reboot-nodes.service` which will call this script once the master is up.


`nano /usr/local/sbin/hard-reboot-nodes.sh`  :
```
#!/bin/bash

num_computes="4"
num_gpus="2"

# compute node BMC addresses
c_bmc[0]="192.168.3.100"
c_bmc[1]="192.168.3.101"
c_bmc[2]="192.168.3.102"
c_bmc[3]="192.168.3.103"

g_bmc[0]="192.168.3.200"
g_bmc[1]="192.168.3.201"


bmc_username="ADMIN"
bmc_password="ADMIN"


for ((i=0; i<${num_computes}; i++)) ; do
    ipmitool -I lanplus -H ${c_bmc[$i]} -U ${bmc_username} -P ${bmc_password} chassis power on
    ipmitool -I lanplus -H ${c_bmc[$i]} -U ${bmc_username} -P ${bmc_password} chassis power cycle
done

for ((i=0; i<${num_gpus}; i++)) ; do
    ipmitool -I lanplus -H ${g_bmc[$i]} -U ${bmc_username} -P ${bmc_password} chassis power on
    ipmitool -I lanplus -H ${g_bmc[$i]} -U ${bmc_username} -P ${bmc_password} chassis power cycle
done
```

Don't forget to `chmod +x /usr/local/sbin/hard-reboot-nodes.sh`.  

`nano /etc/systemd/system/reboot-nodes.service`:  

```
[Unit]
Description=Run script at startup after all systemd services are loaded
After=default.target

[Service]
Type=simple
User=root
RemainAfterExit=yes
ExecStart=/usr/local/sbin/hard-reboot-nodes.sh
TimeoutStartSec=0

[Install]
WantedBy=default.target

```

`systemctl enable reboot-nodes.service`  

Wait a few minutes and verify that the nodes are all up with `pdsh -w c[1-4],g[1-2] uptime`.  
After the reboot, verify `systemctl is-active slurmctld` on master and `pdsh -w c[1-4],g[1-2] systemctl is-active slurmd` for the nodes, and that `sinfo` reports `idle` for the nodes' states.
  
In the worst-case scenario, the power cut can destroy the master's hard drive and you would need to buy new server-grade harddisks and start over from the very beginning of this guide (which is why it was written). We therefore recommend performing a backup of the master hard drive once a fully-working state of the cluster is established, such that in case of disk failure it suffices to reflash the backup image into the new disk and everything else whould work as usual (the advantage of having a stateless node configuration).
  
---

# JupyterHub installation

(reference: https://github.com/jupyterhub/jupyterhub-the-hard-way/blob/HEAD/docs/installation-guide-hard.md)

use the system python (version 3.6.8 at the moment of writing this document) to create a virtual environment private to the to-be-installed jupyterhub:  
`/usr/bin/python3 -m venv /opt/ohpc/pub/apps/jupyterhub`   
Note that we are installing into a path shared to the nodes via NFS so that the nodes will have access to it upon installation on the master.
upgrade pip in this environment with  
`/opt/ohpc/pub/apps/jupyterhub/bin/python3 -m pip install --upgrade pip`  
Then install packages for jupyterhub  
`/opt/ohpc/pub/apps/jupyterhub/bin/python3 -m pip install wheel jupyterhub jupyterlab ipywidgets`   

`yum -y install nodejs npm`    

`npm install -g configurable-http-proxy`

Install packages allowing us to spawn jupyterhub jobs on worker nodes via slurm:  
`/opt/ohpc/pub/apps/jupyterhub/bin/python3 -m pip install batchspawner wrapspawner`  

and add the following to the end of `/opt/ohpc/pub/apps/jupyterhub/etc/jupyterhub/jupyterhub_config.py`:  

```python

c.JupyterHub.db_url = "sqlite:////opt/ohpc/pub/apps/jupyterhub/bin/jupyterhub.sqlite"

c.Spawner.default_url = '/lab'

import wrapspawner
import batchspawner
c.JupyterHub.spawner_class = 'wrapspawner.ProfilesSpawner'
c.Spawner.http_timeout = 30
#------------------------------------------------------------------------------
# ProfilesSpawner configuration

c.SlurmSpawner.batch_script = """#!/bin/bash
#SBATCH --output={{homedir}}/jupyterhub_slurmspawner_%j.log
#SBATCH -e {{homedir}}/jupyterhub_slurmspawner_%j.err
#SBATCH --job-name=jupyterhub
#SBATCH --chdir={{homedir}}
#SBATCH --export={{keepvars}}
#SBATCH --get-user-env=L
#SBATCH -N 1
{% if runtime    %}#SBATCH --time={{runtime}}
{% endif %}{% if nprocs     %}#SBATCH --ntasks={{nprocs}}
{% endif %}{% if memory     %}#SBATCH --mem={{memory}}
{% endif %}{% if partition  %}#SBATCH --partition={{partition}}
{% endif %}{% if ngpus      %}#SBATCH --gres=gpu:{{ngpus}}
{% endif %}{% if options %}#SBATCH {{options}}
{% endif %}


trap 'echo SIGTERM received' TERM

export PATH=/opt/ohpc/pub/apps/jupyterhub/bin:$PATH
which jupyterhub-singleuser
{{cmd}}
echo "jupyterhub-singleuser ended gracefully"
"""


c.ProfilesSpawner.profiles = [
   ('1 core, 1 GB, 1 hours', '1c1g1h', 'batchspawner.SlurmSpawner',
      dict(req_nprocs='1', req_runtime='1:00:00', req_memory='1G')),
   ('5 core, 30 GB, 12 hours', '5c30g12h', 'batchspawner.SlurmSpawner',
      dict(req_nprocs='5', req_runtime='12:00:00', req_memory='30G')),
   ('10 core,  60GB, 24 hours', '10c60g24h', 'batchspawner.SlurmSpawner',
      dict(req_nprocs='10', req_runtime='24:00:00', req_memory='60G')),
   ('20 core, 115 GB, 48 hours', '20c115g48h', 'batchspawner.SlurmSpawner',
      dict(req_nprocs='20', req_runtime='48:00:00', req_memory='115G')),
   ('1 GPU 24 core, 85 GB, 72 hours', '1gpu24c85g72h', 'batchspawner.SlurmSpawner',
      dict(req_nprocs='24', req_runtime='72:00:00', req_memory='85G', req_partition='gpu', req_ngpus='1')),
   ('2 GPU 24 core, 85 GB, 72 hours', '2gpu24c85g72h', 'batchspawner.SlurmSpawner',
      dict(req_nprocs='24', req_runtime='72:00:00', req_memory='85G', req_partition='gpu', req_ngpus='2')),
   ('3 GPU 24 core, 85 GB, 72 hours', '3gpu24c85g72h', 'batchspawner.SlurmSpawner',
      dict(req_nprocs='24', req_runtime='72:00:00', req_memory='85G', req_partition='gpu', req_ngpus='3')),
]

c.JupyterHub.hub_ip = '192.168.1.250'

c.JupyterHub.port = 8090
```

(The `c.ProfilesSpawner.profiles` have been adapted to the resources on our cluster. Modify or create other profiles according to your needs.)

generate config file for jupyterhub  
`mkdir -p /opt/ohpc/pub/apps/jupyterhub/etc/jupyterhub`   
`cd /opt/ohpc/pub/apps/jupyterhub/etc/jupyterhub`  
`/opt/ohpc/pub/apps/jupyterhub/bin/jupyterhub --generate-config`



create a system service such that jupyterhub runs at boot. (Therefore it should never be launched by hand in this installation !)
`mkdir -p /opt/ohpc/pub/apps/jupyterhub/etc/systemd`  

`nano /opt/ohpc/pub/apps/jupyterhub/etc/systemd/jupyterhub.service`    
```
[Unit]
Description=JupyterHub
After=syslog.target network-online.target

[Service]
User=root
Environment="PATH=/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/opt/ohpc/pub/apps/jupyterhub/bin"
ExecStart=/opt/ohpc/pub/apps/jupyterhub/bin/jupyterhub -f /opt/ohpc/pub/apps/jupyterhub/etc/jupyterhub/jupyterhub_config.py

[Install]
WantedBy=multi-user.target
```

Create a symbolic link for the service 

`ln -s /opt/ohpc/pub/apps/jupyterhub/etc/systemd/jupyterhub.service /etc/systemd/system/jupyterhub.service`   

enable and start it  

`systemctl daemon-reload`   

`systemctl enable jupyterhub.service`   

`systemctl start jupyterhub.service`    


check with   
`systemctl status jupyterhub.service`.

Note that We set in the config file that jupyterhub runs on the port 8090, such that you can access it from the browser after connecting via SSH with port forwarding:
(from your personal computer) `ssh -L 8090:localhost:8090 ${YOUR_USERNAME}@th-top.mpq.univ-paris-diderot.fr`   
Then open a browser and go to the address `localhost:8090` and you should be good to go! 


---

Note that during our installation, the spawner only worked for the system python (3.6.8, anaconda python was too recent and didn't work) with certain combinations of packages found with trial and error (due to bugs in the packages), so it is strongly recommended not to upgrade jupyterhub nor any package in its dedicated virtual environment, otherwise things are very likely to break. For reference, we list below the output of `/opt/ohpc/pub/apps/jupyterhub/bin/pip list`.

```
Package              Version
-------------------- ---------
alembic              1.7.7
anyio                3.6.2
argon2-cffi          21.3.0
argon2-cffi-bindings 21.2.0
async-generator      1.10
attrs                22.1.0
Babel                2.11.0
backcall             0.2.0
batchspawner         1.2.0
bleach               4.1.0
certifi              2022.9.24
certipy              0.1.3
cffi                 1.15.1
charset-normalizer   2.0.12
contextvars          2.4
cryptography         38.0.3
dataclasses          0.8
decorator            5.1.1
defusedxml           0.7.1
entrypoints          0.4
greenlet             2.0.1
idna                 3.4
immutables           0.19
importlib-metadata   4.8.3
importlib-resources  5.4.0
ipykernel            5.5.6
ipython              7.16.3
ipython-genutils     0.2.0
ipywidgets           7.7.2
jedi                 0.17.2
Jinja2               3.0.3
json5                0.9.10
jsonschema           3.2.0
jupyter-client       7.1.2
jupyter-core         4.9.2
jupyter-server       1.13.1
jupyter-telemetry    0.1.0
jupyterhub           2.3.1
jupyterlab           3.2.9
jupyterlab-pygments  0.1.2
jupyterlab-server    2.10.3
jupyterlab-widgets   1.1.1
Mako                 1.1.6
MarkupSafe           2.0.1
mistune              0.8.4
nbclassic            0.3.5
nbclient             0.5.9
nbconvert            6.0.7
nbformat             5.1.3
nest-asyncio         1.5.6
notebook             6.4.10
oauthlib             3.2.2
packaging            21.3
pamela               1.0.0
pandocfilters        1.5.0
parso                0.7.1
pexpect              4.8.0
pickleshare          0.7.5
pip                  21.3.1
pipdeptree           2.2.1
prometheus-client    0.15.0
prompt-toolkit       3.0.32
ptyprocess           0.7.0
pycparser            2.21
Pygments             2.13.0
pyOpenSSL            22.1.0
pyparsing            3.0.9
pyrsistent           0.18.0
python-dateutil      2.8.2
python-json-logger   2.0.4
pytz                 2022.6
pyzmq                24.0.1
requests             2.27.1
ruamel.yaml          0.17.21
ruamel.yaml.clib     0.2.7
Send2Trash           1.8.0
setuptools           39.2.0
six                  1.16.0
sniffio              1.2.0
SQLAlchemy           1.4.43
terminado            0.12.1
testpath             0.6.0
tornado              6.1
traitlets            4.3.3
typing_extensions    4.1.1
urllib3              1.26.12
wcwidth              0.2.5
webencodings         0.5.1
websocket-client     1.3.1
wheel                0.37.1
widgetsnbextension   3.6.1
wrapspawner          1.0.1
zipp                 3.6.0
```

# GPU tuning  

To boost the performance of the GPUs, we can add a `systemd` service to the GPU nodes which will call a tuning script at each boot. The script will set the GPU clock frequencies and power usage to the maximum supported values, and set the compute mode to `Exclusive Process`, such that only up to one process (usable from multiple threads at a time) is allowed per GPU device.   

First create the tuning script modified from https://github.com/Microway/MCMS-OpenHPC-Recipe :   
`export CHROOT=/opt/ohpc/admin/images/rocky8.6gpu`  
`nano $CHROOT/etc/init.d/nvidia`  

```bash
#!/bin/bash
#
# nvidia    Set up NVIDIA GPU Compute Accelerators
#
# chkconfig: 2345 55 25
# description:    NVIDIA GPUs provide additional compute capability. \
#    This service sets the GPUs into the desired state.
#
# config: /etc/sysconfig/nvidia

### BEGIN INIT INFO
# Provides: nvidia
# Required-Start: $local_fs $network $syslog
# Required-Stop: $local_fs $syslog
# Should-Start: $syslog
# Should-Stop: $network $syslog
# Default-Start: 2 3 4 5
# Default-Stop: 0 1 6
# Short-Description: Set GPUs into the desired state
# Description:    NVIDIA GPUs provide additional compute capability.
#    This service sets the GPUs into the desired state.
### END INIT INFO


################################################################################
######################## Microway Cluster Management Software (MCMS) for OpenHPC
################################################################################
#
# Copyright (c) 2015-2016 by Microway, Inc.
#
# This file is part of Microway Cluster Management Software (MCMS) for OpenHPC.
#
#    MCMS for OpenHPC is free software: you can redistribute it and/or modify
#    it under the terms of the GNU General Public License as published by
#    the Free Software Foundation, either version 3 of the License, or
#    (at your option) any later version.
#
#    MCMS for OpenHPC is distributed in the hope that it will be useful,
#    but WITHOUT ANY WARRANTY; without even the implied warranty of
#    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#    GNU General Public License for more details.
#
#    You should have received a copy of the GNU General Public License
#    along with MCMS.  If not, see <http://www.gnu.org/licenses/>
#
################################################################################


# source function library
. /etc/rc.d/init.d/functions

# Some definitions to make the below more readable
NVSMI=/usr/bin/nvidia-smi
NVCONFIG=/etc/sysconfig/nvidia
prog="nvidia"

# default settings
NVIDIA_ACCOUNTING=0
NVIDIA_PERSISTENCE_MODE=1
NVIDIA_COMPUTE_MODE=3
NVIDIA_CLOCK_SPEEDS=max
# pull in sysconfig settings
[ -f $NVCONFIG ] && . $NVCONFIG

RETVAL=0


# Determine the maximum graphics and memory clock speeds for each GPU.
# Create an array of clock speed pairs (memory,graphics) to be passed to nvidia-smi
declare -a MAX_CLOCK_SPEEDS
get_max_clocks()
{
    GPU_QUERY="$NVSMI --query-gpu=clocks.max.memory,clocks.max.graphics --format=csv,noheader,nounits"

    MAX_CLOCK_SPEEDS=( $($GPU_QUERY | awk '{print $1 $2}') )
}


start()
{
    /sbin/lspci | grep -qi nvidia
    if [ $? -ne 0 ] ; then
        echo -n $"No NVIDIA GPUs present. Skipping NVIDIA GPU tuning."
        warning
        echo
        exit 0
    fi

    echo -n $"Starting $prog: "

    # If the nvidia-smi utility is missing, this script can't do its job
    [ -x $NVSMI ] || exit 5

    # A configuration file is not required
    if [ ! -f $NVCONFIG ] ; then
        echo -n $"No GPU config file present ($NVCONFIG) - using defaults"
        echo
    fi

    # Set persistence mode first to speed things up
    echo -n "persistence"
    $NVSMI --persistence-mode=$NVIDIA_PERSISTENCE_MODE 1> /dev/null
    RETVAL=$?

    if [ ! $RETVAL -gt 0 ]; then
        echo -n " accounting"
        $NVSMI --accounting-mode=$NVIDIA_ACCOUNTING 1> /dev/null
        RETVAL=$?
    fi

    if [ ! $RETVAL -gt 0 ]; then
        echo -n " compute"
        $NVSMI --compute-mode=$NVIDIA_COMPUTE_MODE 1> /dev/null
        RETVAL=$?
    fi


    if [ ! $RETVAL -gt 0 ]; then
        echo -n " clocks"
        if [ -n "$NVIDIA_CLOCK_SPEEDS" ]; then
            # If the requested clock speed value is "max",
            # work through each GPU and set to max speed.
            if [ "$NVIDIA_CLOCK_SPEEDS" == "max" ]; then
                get_max_clocks

                GPU_COUNTER=0
                GPUS_SKIPPED=0
                while [ "$GPU_COUNTER" -lt ${#MAX_CLOCK_SPEEDS[*]} ] && [ ! $RETVAL -gt 0 ]; do
                    if [[ ${MAX_CLOCK_SPEEDS[$GPU_COUNTER]} =~ Supported ]] ; then
                        if [ $GPUS_SKIPPED -eq 0 ] ; then
                            echo
                            GPUS_SKIPPED=1
                        fi
                        echo "Skipping non-boostable GPU"
                    else
                        $NVSMI -i $GPU_COUNTER --applications-clocks=${MAX_CLOCK_SPEEDS[$GPU_COUNTER]} 1> /dev/null
                        $NVSMI -i $GPU_COUNTER --cuda-clocks=OVERRIDE
                        power=$($NVSMI -i $GPU_COUNTER -q -d POWER | grep -F 'Max Power')
                        power="${power#*: }"
                        power="${power%.00 W}"
                        if [[ "$power" != "N/A" ]]; then
                            $run_or_print $NVSMI -i $GPU_COUNTER -pl "$power"
                        fi
                        
                    fi
                    RETVAL=$?

                    GPU_COUNTER=$(( $GPU_COUNTER + 1 ))
                done
            else
                # This sets all GPUs to the same clock speeds (which only works
                # if the GPUs in this system are all the same).
                $NVSMI --applications-clocks=$NVIDIA_CLOCK_SPEEDS 1> /dev/null
            fi
        else
            $NVSMI --reset-applications-clocks 1> /dev/null
        fi
        RETVAL=$?
    fi

    if [ ! $RETVAL -gt 0 ]; then
        if [ -n "$NVIDIA_POWER_LIMIT" ]; then
            echo -n " power-limit"
            $NVSMI --power-limit=$NVIDIA_POWER_LIMIT 1> /dev/null
            RETVAL=$?
        fi
    fi

    if [ ! $RETVAL -gt 0 ]; then
        success
    else
        failure
    fi
    echo
    return $RETVAL
}

stop()
{
    /sbin/lspci | grep -qi nvidia
    if [ $? -ne 0 ] ; then
        echo -n $"No NVIDIA GPUs present. Skipping NVIDIA GPU tuning."
        warning
        echo
        exit 0
    fi

    echo -n $"Stopping $prog: "
    [ -x $NVSMI ] || exit 5

    $NVSMI --persistence-mode=0 1> /dev/null && success || failure
    RETVAL=$?
    echo
    return $RETVAL
}

restart() {
    stop
    start
}

force_reload() {
    restart
}

status() {
    $NVSMI
}

case "$1" in
    start)
        start
        ;;
    stop)
        stop
        ;;
    restart)
        restart
        ;;
    force-reload)
        force_reload
        ;;
    status)
        status
        RETVAL=$?
        ;;
    *)
        echo $"Usage: $0 {start|stop|restart|force-reload|status}"
        RETVAL=2
esac
exit $RETVAL
```

then `chmod +x $CHROOT/etc/init.d/nvidia` and create the service file:
`nano $CHROOT/lib/systemd/system/nvidia-gpu.service`  
```
[Unit]
Description=NVIDIA GPU Initialization
After=remote-fs.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/etc/init.d/nvidia start
ExecStop=/etc/init.d/nvidia stop

[Install]
WantedBy=multi-user.target
```
and enable the service
`chroot $CHROOT systemctl enable nvidia-gpu.service` 

Finally update the vnfs image with
`wwvnfs --chroot ${CHROOT}`
and then reboot gpu nodes when they are idle to upload the new image.
