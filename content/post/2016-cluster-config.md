+++
title = "Configuring QLogic Infiniband"
date = 2016-04-14T14:42:42-07:00
draft = false
summary="Intel True Scale Fabric Configuration/Install on [Rubicon](http://cmesim.rd.unr.edu) Rocks 6.2"
# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["hpc","qlogic","infiniband"]
categories = ["HPC"]

# Featured image
# Place your image in the `static/img/` folder and reference its filename below, e.g. `image = "example.jpg"`.
# Use `caption` to display an image caption.
#   Markdown linking is allowed, e.g. `caption = "[Image credit](http://example.org)"`.
# Set `preview` to `false` to disable the thumbnail in listings.
[header]
image = ""
caption = ""
preview = true

+++
## Intel True Scale Fabric Configuration/Install on [Rubicon](http://cmesim.rd.unr.edu) Rocks 6.2

`Author: V.Vasquez`
`Date: Thu 14 Apr 2016 08:47:15 AM PDT `

 
## HPC Roll 

If the HPC roll is installed, it needs to be removed from the headnode and compute network, including 
uninstalling all the RPMS in`noarch` and `x86_64`.

## Installing Host Stack on SM Server

I am using the Frontend or Headnode for the SM Server. Any other Compute node can be used for 
this purpose. 

**Note:** Use `sudo su` to perform all these operations.

```Shell
$ cd
$ tar xf /share/apps/src/IntelIB-Basic.RHEL6-x86_64.7.4.0.0.21.tgz
$ cd IntelIB-Basic.RHEL6-x86_64.7.4.0.0.21/
$ ./INSTALL -a
$ reboot 
```

On the node(s) which will be SM serves:
```Shell
$ chkconfig opensmd on
```
#$ Installing Host Stack on Compute Nodes

For installing in Compute Nodes we will use the distributed shell command `dsh`. There are other choices 
including `tentakel`.

## First, we need to install a bunch of required RPMs and tgz on the compute nodes to meet all dependencies.

The `libevent-devel` rpms has some issues. It is better to use the following procedure:
```Shell
$ cd /share/apps/IntelIB
$ wget "https://github.com/downloads/libevent/libevent/libevent-1.4.14b-stable.tar.gz"
```
Then compile and install this library on each compute node:

```Shell
$ dsh -acM "tar -xzvf /share/apps/IntelIB/libevent-1.4.14b-stable.tar.gz"
```
```Shell
$ dsh -acM "cd libevent-1.4.14b-stable; ./configure"
$ dsh -acM "cd libevent-1.4.14b-stable; make"
$ dsh -acM "cd libevent-1.4.14b-stable; make install"
```
Download the RPM `libgssglue-devel-0.1-11.el6.x86_64.rpm` to `/share/apps/IntelIB` and execute:
```Shell
$ dsh -acM "yum -y install /share/apps/IntelIB/libgssglue-devel-0.1-11.el6.x86_64.rpm"
```
Find the RPM `gcc-gfortran-4.4.7-11.el6.x86_64.rpm` (which it is not easy to find!) and execute 
the command:
```Shell
$ dsh -acM "yum -y install /share/apps/IntelIB/gcc-gfortran-4.4.7-11.el6.x86_64.rpm"
```
**Note:** I found this file at http://ftp.riken.jp/Linux/cern/slc60beta/updates/x86_64/RPMS/repoview/gcc-gfortran.html

Install `binutils-devel`:
```Shell
$ dsh -acM "yum -y install binutils-devel"
```
Then install the dependencies below. I do it in segments. It is easier to identify failed installations this way, 
but at this point it should go smoothly. 

```Shell
$ dsh -acM "yum -y install libsysfs libsysfs-devel pciutils-devel tcl tcl-devel tk tk-devel libstdc++-devel" 
$ dsh -acM "yum -y install gcc-c++ krb5-devel krb5-libs libevent-devel nfs-utils-lib-devel openldap-devel"
$ dsh -acM "yum -y install e2fsprogs-devel zlib-devel kernel-devel"
```
Now we are to start the installation of the **Intel True Scale Fabric Software**.

```Shell
$ dsh -acM "tar xf /share/apps/IntelIB/IntelIB-Basic.RHEL6-x86_64.7.4.0.0.21.tgz"
$ dsh -acM "cd IntelIB-Basic.RHEL6-x86_64.7.4.0.0.21/; ./INSTALL -a"
$ dsh -acM "rm -rf IntelIB-Basic.RHEL6-x86_64.7.4.0.0.21/"
$ dsh -acM "reboot" 
```

## Configuring the IPoIB Interface in the SM Server (choice: Frontend)

1. The network need to be configured by using: `rocks add network {name} {subnet} {netmask}`
```Shell
$ rocks add network ib 10.240.20.0 255.255.255.0
```
2. Set network MTU for IPoIB. Either Connected Mode (CM), MTU size 65520 or 
Unreliable Datagram (UD), MTU size 2044. 

```Shell
$ rocks set network mtu ib 65520
```
3. Edit firewall settings as follows:

```Shell
$ rocks add firewall host=compute action=accept service=ssh chain=input network=ib protocol=all flags="-m state --state NEW"
```

4. The firewall must be synced:

```Shell
$ rocks sync host firewall compute
```

5. Tell Rocks to serve DNS for the ipoib network, and then push out the new configuration to DNS.

```Shell
$ rocks set network servedns ib true

$ rocks sync config
```

6. The interface can now be setup by using: `rocks add host interface {host} {iface} [ip=string] [name=string] [subnet=string]`
```Shell
$ rocks add host interface localhost iface=ib0 subnet=ib ip=10.240.20.100 name=localhost-ib 
```
If the error interface `ib0` exists is encountered, you need to remove a previously added IP address:
```Shell
$ rocks remove host interface localhost ib0
```

7. Use more add host interface commands to set the IP address for each compute node (write script to this). For example:

```Shell
$ rocks add host interface compute-1-1 iface=ib0 subnet=ib ip=10.240.20.101 name=compute-1-1-ib 
```

8. If the error interface `ib0` exists is encountered, you need to remove a previously added IP address:

```Shell
$ rocks remove host interface compute-1-1 ib0
```
The above removes the entry for the Infiniband interface ib0 associated with `compute-1-1` .
In general:

```Shell
$ rocks remove host interface frontend_node ib0

$ rocks remove host interface compute_node ib0
```
9. For safety, we are going to remove any `ib0` interfaces if present in the compute nodes:

```Shell
#!/bin/sh
# 
# Script for removing host interfaces in compute nodes. 
# Author: V.Vasquez, Date: 2016-04-07
#
ME=`hostname`
EXECHOSTS=$(qconf -sel|sed 's/.local//')
node=100

if [ $(id -u) -eq 0 ]; then
        for TARGETHOST in $EXECHOSTS; do
                if [ "$ME" == "$TARGETHOST" ]; then
                        echo "Skipping $ME. This is the submission host"
                else
                        node=$((node + 1))
                        echo "Removing host interface ib0 in $TARGETHOST-ib"
                        rocks remove host interface $TARGETHOST ib0
                fi
        done
else
        echo "Only root may add/remove hosts interfaces to compute nodes"
        exit 2
fi
```
10. Sync the network

```Shell
$ rocks sync host network
```

11. Add the host interfaces to compute nodes:

```Shell
#!/bin/sh
# 
# Script for adding host interfaces to compute nodes for InfiniBand Subnet Manager (SM) 
# Author: V.Vasquez, Date: 2016-04-07
#
ME=`hostname`
EXECHOSTS=$(qconf -sel|sed 's/.local//')
node=100

if [ $(id -u) -eq 0 ]; then
        for TARGETHOST in $EXECHOSTS; do
                if [ "$ME" == "$TARGETHOST" ]; then
                        echo "Skipping $ME. This is the submission host"
                else
                        node=$((node + 1))
                        echo "Adding host interface ib0 in $TARGETHOST-ib with IP=10.240.20.$node"
                        rocks add host interface $TARGETHOST iface=ib0 subnet=ib ip=10.240.20.$node name=$TARGETHOST-ib
                fi
        done
else
        echo "Only root may add/remove hosts interfaces to compute nodes"
        exit 2
fi
```

12. Restart the compute network

```Shell
$ dsh -acM "reboot" 
```

13. Once the IP addresses have been set, use the `sync` command to sync the fabric:

```Shell
$ rocks sync host network localhost
$ rocks sync host network compute
```

## Checking Status of HCA fabric (Up/Active)

```Shell
$ dsh -acM "ibstat | grep Sleep"
```
If you see an output like this:

```Shell
compute-1-8: 		Physical state: Sleep
compute-1-12: 		Physical state: Sleep
compute-1-14: 		Physical state: Sleep
```
then do the following:

```Shell
$ pdsh -w compute-1-[8,12,14] ibportstate -D 0 1 enable
```

Check that all fabric components are visible:

```Shell
$ fabric_info
```
Typical output:

```Shell
Fabric Information:
SM: cmesim HCA-1 Guid: 0x0011750000707ac4 State: Master
Number of CAs: 60
Number of CA Ports: 60
Number of Switch Chips: 7
Number of Links: 107
Number of 1x Ports: 0
-------------------------------------------------------------------------------
```

Check that all fabric nodes hosts are visible:

```Shell
$ ibhosts
```
Typical output:

```shell
Ca	: 0x001175000079dafe ports 1 "compute-4-16 HCA-1"
Ca	: 0x0011750000791080 ports 1 "compute-4-15 HCA-1"
Ca	: 0x0011750000792154 ports 1 "compute-4-14 HCA-1"
Ca	: 0x0011750000791bd2 ports 1 "compute-4-13 HCA-1"
Ca	: 0x0011750000791a64 ports 1 "compute-4-12 HCA-1"
Ca	: 0x00117500007997cc ports 1 "compute-4-11 HCA-1"
Ca	: 0x0011750000792140 ports 1 "compute-4-10 HCA-1"
Ca	: 0x0011750000790e68 ports 1 "compute-4-9 HCA-1"
Ca	: 0x0011750000799776 ports 1 "compute-4-7 HCA-1"
Ca	: 0x0011750000791c9e ports 1 "compute-4-6 HCA-1"
Ca	: 0x0011750000792100 ports 1 "compute-4-5 HCA-1"
Ca	: 0x0011750000791c34 ports 1 "compute-4-4 HCA-1"
Ca	: 0x00117500007921d8 ports 1 "compute-4-3 HCA-1"
Ca	: 0x001175000079933e ports 1 "compute-4-1 HCA-1"
Ca	: 0x00117500007919be ports 1 "compute-2-16 HCA-1"
Ca	: 0x0011750000790e6c ports 1 "compute-2-15 HCA-1"
Ca	: 0x0011750000790e84 ports 1 "compute-2-14 HCA-1"
Ca	: 0x0011750000799954 ports 1 "compute-2-13 HCA-1"
Ca	: 0x0011750000792020 ports 1 "compute-2-12 HCA-1"
Ca	: 0x0011750000791a5c ports 1 "compute-2-11 HCA-1"
Ca	: 0x0011750000790e62 ports 1 "compute-2-10 HCA-1"
Ca	: 0x00117500007969d2 ports 1 "compute-2-9 HCA-1"
Ca	: 0x001175000079216e ports 1 "compute-2-8 HCA-1"
Ca	: 0x0011750000791c78 ports 1 "compute-2-7 HCA-1"
Ca	: 0x0011750000790e6a ports 1 "compute-2-6 HCA-1"
Ca	: 0x0011750000791c5c ports 1 "compute-2-5 HCA-1"
Ca	: 0x0011750000795f5c ports 1 "compute-2-4 HCA-1"
Ca	: 0x0011750000799672 ports 1 "compute-2-3 HCA-1"
Ca	: 0x00117500007920b8 ports 1 "compute-2-2 HCA-1"
Ca	: 0x0011750000792192 ports 1 "compute-2-1 HCA-1"
Ca	: 0x0011750000791a44 ports 1 "compute-3-16 HCA-1"
Ca	: 0x001175000079938c ports 1 "compute-3-15 HCA-1"
Ca	: 0x00117500007919d0 ports 1 "compute-3-14 HCA-1"
Ca	: 0x0011750000799a92 ports 1 "compute-3-13 HCA-1"
Ca	: 0x0011750000790ef2 ports 1 "compute-3-12 HCA-1"
Ca	: 0x001175000078cd94 ports 1 "compute-3-11 HCA-1"
Ca	: 0x0011750000792056 ports 1 "compute-3-9 HCA-1"
Ca	: 0x0011750000791a46 ports 1 "compute-3-10 HCA-1"
Ca	: 0x0011750000791064 ports 1 "compute-3-8 HCA-1"
Ca	: 0x0011750000792064 ports 1 "compute-3-7 HCA-1"
Ca	: 0x0011750000790f62 ports 1 "compute-3-6 HCA-1"
Ca	: 0x0011750000799b74 ports 1 "compute-3-5 HCA-1"
Ca	: 0x00117500007920be ports 1 "compute-3-3 HCA-1"
Ca	: 0x0011750000796d68 ports 1 "compute-3-1 HCA-1"
Ca	: 0x001175000079939e ports 1 "compute-3-2 HCA-1"
Ca	: 0x0011750000790f2e ports 1 "compute-1-16 HCA-1"
Ca	: 0x00117500007910ee ports 1 "compute-1-15 HCA-1"
Ca	: 0x0011750000792024 ports 1 "compute-1-12 HCA-1"
Ca	: 0x0011750000791c60 ports 1 "compute-1-13 HCA-1"
Ca	: 0x0011750000790f3c ports 1 "compute-1-11 HCA-1"
Ca	: 0x001175000079212a ports 1 "compute-1-10 HCA-1"
Ca	: 0x0011750000791a22 ports 1 "compute-1-9 HCA-1"
Ca	: 0x0011750000792168 ports 1 "compute-1-8 HCA-1"
Ca	: 0x0011750000792128 ports 1 "compute-1-7 HCA-1"
Ca	: 0x00117500007996dc ports 1 "compute-1-6 HCA-1"
Ca	: 0x00117500007919c6 ports 1 "compute-1-3 HCA-1"
Ca	: 0x001175000079207a ports 1 "compute-1-5 HCA-1"
Ca	: 0x0011750000796dd8 ports 1 "compute-1-2 HCA-1"
Ca	: 0x0011750000799378 ports 1 "compute-1-1 HCA-1"
Ca	: 0x0011750000707ac4 ports 1 "cmesim HCA-1"
```


**Note**:  If not using IFS software on the cluster and need to start OpenSM on a node to manage the IB fabric, then start OpenSM as shown below:
```Shell
service opensmd status
opensm is stopped
service opensmd start
Starting IB Subnet Manager. [  OK  ]
service opensmd status
opensm (pid 7366) is running...
```
**Note**: If using Host Based SM, then by default it is tied to Port1 of the HCA.

Use the following command to determine which version of Intel software is installed:
```Shell
$ iba_config -V

7.4.0.0.21
```
**Note**: Verify the link is up using `ibstat` command and state is `active`. If state is initializing; then it means there is no subnet manager running on the fabric.

## Build OpenMPI with gcc and Intel Compilers on Intel HCAs

The appropriate scripts to build OpenMPI with Infiniband support are located 
at `/opt/iba/src/MPI`. Here, we are going to use `do_openmpi_buid` to compile 
OpenMPI version 1.8.4 with SGE support. Before proceding, we need to modify 
this script to change the line:
```Shell
--define 'configure_options $CONFIG_OPTIONS $openmpi_ldflags --with-openib=$STACK_PREFIX \ 
--with-openib-libdir=$STACK_PREFIX/$openmpi_lib $openmpi_comp_env  \ 
$openmpi_conf_psm --with-contrib-vt-flags=--disable-iotrace' \
```
to
```Shell
--define 'configure_options $CONFIG_OPTIONS $openmpi_ldflags --with-verbs=$STACK_PREFIX \ 
--with-verbs-libdir=$STACK_PREFIX/$openmpi_lib $openmpi_comp_env \
$openmpi_conf_psm --with-contrib-vt-flags=--disable-iotrace' \
```
The change is easily done with the command:
```Shell
sed -i 's/openib/verbs/g' do_openmpi_build
```
**Note**: The change is necessary because the option the option `--with-openib` is deprecated in OpenMPI version 1.8.4.

The OpenMPI choices in `cmesim.rd.unr.edu` are the following:
```Shell
mvapich2_gcc-2.0.1
mvapich2_gcc_qlc-2.0.1
mvapich2_intel_qlc-2.0.1
mvapich2_pgi_qlc-2.0.1
mvapich_gcc-1.2.0
mvapich_gcc_qlc-1.2.0
mvapich_intel_qlc-1.2.0
mvapich_pgi_qlc-1.2.0
openmpi_gcc-1.8.4
openmpi_gcc_qlc-1.8.4
openmpi_intel_qlc-1.8.4
openmpi_pgi_qlc-1.8.4
```
We need to re-compile the `gcc` and `intel` compilers versions of `openmpi_*-1.8.4`. 
The first choice for `openmpi` will use verbs by default, and any with the `_qlc` string 
will use `PSM` by default. Based on this, we are going to re-compile only the following:
```Shell
openmpi_gcc-1.8.4
openmpi_gcc_qlc-1.8.4
openmpi_intel_qlc-1.8.4
```
First install `flex` in the Frontend, before compilation:
```Shell
$ yum -y install flex
```

The RPMs built during this process will be installed on the system and 
they can also be found in `/opt/iba/src/MPI`, which we need to deploy these new RPMS 
to the compute nodes. 

Compilation of `openmpi_gcc-1.8.4`:
```Shell
$ module load gnu
$ export CONFIG_OPTIONS=--with-sge
$ ./do_openmpi_build -d gcc 
```
After the package builds successfully, then do:

```Shell
$ cp /opt/iba/src/MPI/openmpi_gcc-1.8.4-1.x86_64.rpm /share/apps/IntelIB

$ dsh -acM "yum -y reinstall /share/apps/IntelIB/openmpi_gcc-1.8.4-1.x86_64.rpm"
```
Compilation of `openmpi_gcc_qlc-1.8.4`:
```Shell
$ module load gnu
$ export CONFIG_OPTIONS=--with-sge
$ ./do_openmpi_build -d -Q gcc 
```
After the package builds successfully, then do:

```Shell
$ cp /opt/iba/src/MPI/openmpi_gcc_qlc-1.8.4-1.x86_64.rpm /share/apps/IntelIB

$ dsh -acM "yum -y reinstall /share/apps/IntelIB/openmpi_gcc_qlc-1.8.4-1.x86_64.rpm"
```

Compilation of `openmpi_intel_qlc-1.8.4`:
```Shell
$ module load intel
$ export CONFIG_OPTIONS=--with-sge
$ ./do_openmpi_build -d -Q intel 
```
After the package builds successfully, then do:

```Shell
$ cp /opt/iba/src/MPI/openmpi_intel_qlc-1.8.4-1.x86_64.rpm /share/apps/IntelIB

$ dsh -acM "yum -y reinstall /share/apps/IntelIB/openmpi_intel_qlc-1.8.4-1.x86_64.rpm"
```

