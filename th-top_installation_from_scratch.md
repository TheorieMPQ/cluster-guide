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

#
# Example slurm.conf file. Please run configurator.html
# (in doc/html) to build a configuration file customized
# for your environment.
#
#
# slurm.conf file generated by configurator.html.
#
# See the slurm.conf man page for more information.
#
ClusterName=mpq-th-top
ControlMachine=th-top
#ControlAddr=
#BackupController=
#BackupAddr=
#
SlurmUser=slurm
#SlurmdUser=root
SlurmctldPort=6817
SlurmdPort=6818
AuthType=auth/munge
#JobCredentialPrivateKey=
#JobCredentialPublicCertificate=
StateSaveLocation=/var/spool/slurm/ctld
SlurmdSpoolDir=/var/spool/slurm/d
SwitchType=switch/none
MpiDefault=none
SlurmctldPidFile=/var/run/slurmctld.pid
SlurmdPidFile=/var/run/slurmd.pid
ProctrackType=proctrack/pgid
#PluginDir=
#FirstJobId=
#MaxJobCount=
#PlugStackConfig=
#PropagatePrioProcess=
#PropagateResourceLimits=
#PropagateResourceLimitsExcept=
#Prolog=
#Epilog=
#SrunProlog=
#SrunEpilog=
#TaskProlog=
#TaskEpilog=
#TaskPlugin=
#TrackWCKey=no
#TreeWidth=50
#TmpFS=
#UsePAM=
#
# TIMERS
SlurmctldTimeout=300
SlurmdTimeout=300
InactiveLimit=0
MinJobAge=300
KillWait=30
Waittime=0
#
# SCHEDULING
SchedulerType=sched/backfill
#SchedulerAuth=
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory
#PriorityType=priority/multifactor
#PriorityDecayHalfLife=14-0
#PriorityUsageResetPeriod=14-0
#PriorityWeightFairshare=100000
#PriorityWeightAge=1000
#PriorityWeightPartition=10000
#PriorityWeightJobSize=1000
#PriorityMaxAge=1-0
#
# LOGGING
SlurmctldDebug=info
SlurmctldLogFile=/var/log/slurmctld.log
SlurmdDebug=info
SlurmdLogFile=/var/log/slurmd.log
#JobCompType=jobcomp/none
#JobCompLoc=
#
# ACCOUNTING
#JobAcctGatherType=jobacct_gather/linux
#JobAcctGatherFrequency=30
#
#AccountingStorageType=accounting_storage/slurmdbd
#AccountingStorageHost=
#AccountingStorageLoc=
#AccountingStoragePass=
#AccountingStorageUser=
#
# COMPUTE NODES
# OpenHPC default configuration
TaskPlugin=task/affinity
PropagateResourceLimitsExcept=MEMLOCK
JobCompType=jobcomp/filetxt
Epilog=/etc/slurm/slurm.epilog.clean



GresTypes=gpu

#NODES
NodeName=c[1-4] CPUs=20 CoresPerSocket=10 ThreadsPerCore=1 RealMemory=120000 State=UNKNOWN

NodeName=g[1-2] Gres=gpu:3 CPUS=24 CoresPerSocket=12 ThreadsPerCore=1 RealMemory=90000 State=UNKNOWN

#PARTITIONS
PartitionName=cpu Nodes=c[1-4] Default=YES MaxTime=INFINITE State=UP DefMemPerCPU=512 Oversubscribe=NO
PartitionName=gpu Nodes=g[1-2] MaxTime=INFINITE State=UP DefMemPerCPU=512 Oversubscribe=NO

SlurmctldParameters=enable_configless
ReturnToService=1


```

create another file `gres.conf` in the same folder to specify the gpu resources `nano /etc/slurm/gres.conf`:
```
NodeName=g1 Name=gpu File=/dev/nvidia[0-2]
NodeName=g2 Name=gpu File=/dev/nvidia[0-2]
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

When installing other apps (such as Julia), it is recommended to put them under the shared folder `/opt/ohpc/pub/apps/` and then write corresponding module files for each.

## Booting the nodes

Reboot the master node with `reboot`.   
wait a few minutes and connect again as root. `source input.local` again.

At this point you should be able to boot up the nodes into the vnfs. The IPMI interface (for BMC) allows us to "push the power button" of the nodes remotely:

```
export IPMI_PASSWORD=ADMIN


for ((i=0; i<${num_computes}; i++)) ; do
    ipmitool -E -I lanplus -H ${c_bmc[$i]} -U ${bmc_username} -P ${bmc_password} chassis power reset
done

for ((i=0; i<${num_gpus}; i++)) ; do
    ipmitool -E -I lanplus -H ${g_bmc[$i]} -U ${bmc_username} -P ${bmc_password} chassis power reset
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

Check the slurm status with `sinfo`. If the node `STATE` is `down` (typically what happens after nodes reboot) then you need to bring them back by 
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
