+++
title = "Rubicon Cluster Basic Usage Guide"
date = 2016-06-22T16:05:47-07:00
draft = false

summary="Chemical & Materials Engineering Department's Rubicon Cluster Basic Usage Guide"
# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = []
categories = []

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
## Adding new admin users
User creation is handled via a predistributed script in the /opt directory.
Both scripts require root privileges to run, and cannot be modified.

Two scripts are installed:

    1. `create_admin_user.sh`
    2. `create_normal_user.sh`

Usage syntax is as follows:

```shell
sudo ./create_admin_user.sh USERNAME
```

OR
 
```shell
sudo ./create_normal_user.sh USERNAME
```

The script performs the following actions:

 - create the user on the head node
 - set a temporary password
 - expire that password
 - generate SSH keys for the user
 - copy SSH keys to user home (for compute node access)
 - create user folder in /scratch/USERNAME and link it to /home/USERNAME/scratch
 - create user on compute node NFS root filesystem with same UID and GID as the user on head node
 - expire the password of the compute node account as access is controlled strictly by SSH key

Once complete, the user should log in to the head node with the temporary password,
at which point they will be asked to set the new account password.

Note: admin users are added to the group 'wheel' for sudo access. Normal users are not added to this group. 

## Running an MPI job in Slurm
Currently, SLURM is configured for non-direct usage of the OpenMPI interface. Simply put, SLURM will not automatically configure ```mpirun``` for you. However, this is not a big deal as we will see.

We can run slurm directly or in batch mode. For this example we will demonstrate a direct method that can easily be converted into a batch script.

Lets say we want to run a simple SCF calculation using Quantum Espresso. This requires the ```pw.x``` executable and an MPI implementation, and the input files.


Lets check on the status of the cluster and see if other jobs are running:


```shell
[monkey@cmesim ~]$ sinfo

PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
compute      up   infinite     64   idle compute-1-[1-16],compute-2-[1-16],compute-3-[1-16],compute-4-[1-16]
chassis1*    up   infinite     16   idle compute-1-[1-16]
chassis2     up   infinite     16   idle compute-2-[1-16]
chassis3     up   infinite     16   idle compute-3-[1-16]
chassis4     up   infinite     16   idle compute-4-[1-16]
```


```shell
[monkey@cmesim ~]$ squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)

```

Everything looks clear!

Now lets allocate 8 nodes for our run:

```shell
[monkey@cmesim ~]$ salloc -N 8
salloc: Granted job allocation 76
```

If we check the status again, we will see some changes:

```shell
[monkey@cmesim ~]$ sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
compute      up   infinite      8  alloc compute-1-[1-8]
compute      up   infinite     56   idle compute-1-[9-16],compute-2-[1-16],compute-3-[1-16],compute-4-[1-16]
chassis1*    up   infinite      8  alloc compute-1-[1-8]
chassis1*    up   infinite      8   idle compute-1-[9-16]
chassis2     up   infinite     16   idle compute-2-[1-16]
chassis3     up   infinite     16   idle compute-3-[1-16]
chassis4     up   infinite     16   idle compute-4-[1-16]
```

```shell
[monkey@cmesim ~]$ squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
                78  chassis1     bash   monkey  R       0:19      8 compute-1-[1-8]
```

Lets make sure our hostfile is set for the allocation:

```shell
[monkey@cmesim run_folder]$ cat ../../hostfile 
compute-1-1
compute-1-2
compute-1-3
compute-1-4
compute-1-5
compute-1-6
compute-1-7
compute-1-8
```

> NOTE: The hostfile needs to reflect the current nodes in the allocation, e.g. if you are allocated nodes ```compute-1-[1-8]```, then the ```hostfile``` should contain an expanded list of those compute nodes.


We can now run our ```mpirun``` command from within the allocation:


```shell
[monkey@cmesim run_folder]$ mpirun --mca btl_openib_verbose 1 --mca btl ^tcp --hostfile ../../hostfile -N 8 -n 24 pw.x < si.scf.cg.in 
```

Command breakdown:

| command | Reason| 
|---------|-------|
|```mpirun``` | OpenMPi Binary|
|```--mca btl_openib_verbose 1```| Enable verbose output of IB info|
|```--mca btl ^tcp``` | Disable TCP (use IB instead)|
|```--hostfile /PATH/TO/HOSTFILE``` | path to host file|
|```-N 8``` | Use 8 total nodes (allocated)|
|```-n 24``` | use 24 procs on EACH node (total procs = N*n)|
|``` pw.x < filename.in``` | the binary we want to run|



Once we are done running our job, we exit the allocation and give the reservation back to the scheduler:

```shell
[monkey@cmesim ~]$ exit
exit
salloc: Relinquishing job allocation 78
```




*For more information and help running jobs in SLURM, go to*:

 > https://slurm.schedmd.com/quickstart.html


