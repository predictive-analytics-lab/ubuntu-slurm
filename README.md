# Installing SLURM

Instructions for setting up a SLURM cluster using Ubuntu 18.04.3 with GPUs. Go from a pile of hardware to a functional GPU cluster with job queueing.

OS used: Ubuntu 18.04.3 LTS

1. [Overview](#overview)
2. [OS preparations](#os-preparations)
3. [Preparing for SLURM installation](#preparing-for-slurm-installation)
4. [Install SLURM](#install-slurm)
5. [Troubleshooting](#troubleshooting)

# Overview

This guide will help you create and install a GPU HPC cluster with a job queue and user management. The idea is to have a GPU cluster which allows use of a few GPUs by many people. Using multiple GPUs at once is not the point here, and hasn’t been tested. This guide demonstrates how to create a GPU cluster for neural networks (deep learning) which uses Python and related neural network libraries (Tensorflow, Keras, Pytorch), CUDA, and NVIDIA GPU cards. You can expect this to take you a few days up to a week.

## Acknowledgements

This wouldn’t have been possible without this [github repo](https://github.com/mknoxnv/ubuntu-slurm) from mknoxnv. I don’t know who that person is, but they saved me weeks of work trying to figure out all the conf files and services, etc.

# OS preparations

## Synchronizing time

You can see if times are synced with the `date` command on the various machines.

It’s not a bad idea to sync the time across the servers. [Here’s how](https://knowm.org/how-to-synchronize-time-across-a-linux-cluster/). One time when I set it up, it was ok, but another time the slurmctld service wouldn’t start and it was because the times weren’t synced.

## Set up munge and slurm users and groups

Immediately after installing OS’s, you want to create the munge and slurm users and groups on all machines. The GID and UID (group and user IDs) must match for munge and slurm across all machines. If you have a lot of machines, you can use the parallel SSH utilities mentioned before. There are also other options like NIS and NISplus. One other option is to use FreeIPA to create users and groups.

On all machines we need the munge authentication service and slurm installed. First, we want to have the munge and slurm users/groups with the same UIDs and GIDs. In my experience, these are the only GID and UIDs that need synchronization for the cluster to work. On all machines:

```
sudo adduser -u 511 munge --disabled-password --gecos ""
sudo adduser -u 521 slurm --disabled-password --gecos ""
```

### You shouldn’t need to do this, but just in case, you could create the groups first, then create the users

```
sudo addgroup -gid 511 munge
sudo addgroup -gid 521 slurm
sudo adduser -u 511 munge --disabled-password --gecos "" -gid 511
sudo adduser -u 521 slurm --disabled-password --gecos "" -gid 521
```

When a user is created, a group with the same name is created as well.

The numbers don’t matter as long as they are available for the user and group IDs. These numbers seemed to work with a default Ubuntu 18.04.3 installation. It seems like by default ubuntu sets up a new user with a UID and GID of UID + 1 if the GID already exists, so this follows that pattern.

# Preparing for SLURM installation

## Install munge on the master:

```
sudo apt-get install libmunge-dev libmunge2 munge -y
sudo systemctl enable munge
sudo systemctl start munge
```

Test munge if you like: `munge -n | unmunge | grep STATUS`

Copy the munge key to /storage

```
sudo cp /etc/munge/munge.key /storage/
sudo chown munge /storage/munge.key
sudo chmod 400 /storage/munge.key
```

## Install munge on worker nodes:

```
sudo apt-get install libmunge-dev libmunge2 munge
sudo cp /storage/munge.key /etc/munge/munge.key
sudo systemctl enable munge
sudo systemctl start munge
```

If you want, you can test munge: `munge -n | unmunge | grep STATUS`

## Prepare DB for SLURM

These instructions more or less follow this github repo: https://github.com/mknoxnv/ubuntu-slurm

First we want to clone the repo: `cd /storage` `git clone https://github.com/mknoxnv/ubuntu-slurm.git`

Install prereqs:

```
sudo apt-get install git gcc make ruby ruby-dev libpam0g-dev libmariadb-client-lgpl-dev libmysqlclient-dev mariadb-server build-essential libssl-dev -y
sudo gem install fpm
```

Next we set up MariaDB for storing SLURM data:

```
sudo systemctl enable mariadb
sudo systemctl start mariadb
sudo mysql -u root
```

Within mysql:

```
create database slurm_acct_db;
create user 'slurm'@'localhost';
set password for 'slurm'@'localhost' = password('slurmdbpass');
grant usage on *.* to 'slurm'@'localhost';
grant all privileges on slurm_acct_db.* to 'slurm'@'localhost';
flush privileges;
exit
```

Copy the default db config file: `cp /storage/ubuntu-slurm/slurmdbd.conf /storage`

Ideally you want to change the password to something different than `slurmdbpass`. This must also be set in the config file `/storage/slurmdbd.conf`.

# Install SLURM

## Download and install SLURM on Master

### Build the SLURM .deb install file

It’s best to check the downloads page and use the latest version (right click link for download and use in the wget command). Ideally we’d have a script to scrape the latest version and use that dynamically.

You can use the -j option to specify the number of CPU cores to use for ‘make’, like `make -j12`. `htop` is a nice package that will show usage stats and quickly show how many cores you have.

```
cd /storage
wget https://download.schedmd.com/slurm/slurm-20.11.5.tar.bz2
tar xvjf slurm-20.11.5.tar.bz2
cd slurm-20.11.5
./configure --prefix=/tmp/slurm-build --sysconfdir=/etc/slurm --enable-pam --with-pam_dir=/lib/x86_64-linux-gnu/security/ --without-shared-libslurm
make
make contrib
make install
cd ..
```

### Install SLURM

```
sudo fpm -s dir -t deb -v 1.0 -n slurm-20.11.5 --prefix=/usr -C /tmp/slurm-build .
sudo dpkg -i slurm-20.11.5_1.0_amd64.deb
```

Make all the directories we need:

```
sudo mkdir -p /etc/slurm /etc/slurm/prolog.d /etc/slurm/epilog.d /var/spool/slurm/ctld /var/spool/slurm/d /var/log/slurm
sudo chown slurm /var/spool/slurm/ctld /var/spool/slurm/d /var/log/slurm
```

Copy slurm control and db services:

```
sudo cp /storage/ubuntu-slurm/slurmdbd.service /etc/systemd/system/
sudo cp /storage/ubuntu-slurm/slurmctld.service /etc/systemd/system/
```

The slurmdbd.conf file should be copied before starting the slurm services: `sudo cp /storage/slurmdbd.conf /etc/slurm/`

Start the slurm services:

```
sudo systemctl daemon-reload
sudo systemctl enable slurmdbd
sudo systemctl start slurmdbd
sudo systemctl enable slurmctld
sudo systemctl start slurmctld
```

If the master is also going to be a worker/compute node, you should do:

```
sudo cp /storage/ubuntu-slurm/slurmd.service /etc/systemd/system/
sudo systemctl enable slurmd
sudo systemctl start slurmd
```

## Worker nodes

Now install SLURM on worker nodes:

```
cd /storage
sudo dpkg -i slurm-19.05.2_1.0_amd64.deb
sudo cp /storage/ubuntu-slurm/slurmd.service /etc/systemd/system/
sudo systemctl enable slurmd
sudo systemctl start slurmd
```

## Configuring SLURM

Next we need to set up the configuration file. Copy the default config from the github repo:

`cp /storage/ubuntu-slurm/slurm.conf /storage/slurm.conf`

Note: for job limits for users, you should add the [AccountingStorageEnforce=limits](https://slurm.schedmd.com/resource_limits.html) line to the config file.

Once SLURM is installed on all nodes, we can use the command

`sudo slurmd -C`

to print out the machine specs. Then we can copy this line into the config file and modify it slightly. To modify it, we need to add the number of GPUs we have in the system (and remove the last part which show UpTime). Here is an example of a config line:

`NodeName=worker1 Gres=gpu:2 CPUs=12 Boards=1 SocketsPerBoard=1 CoresPerSocket=6 ThreadsPerCore=2 RealMemory=128846`

Take this line and put it at the bottom of `slurm.conf`.

Next, setup the `gres.conf` file. Lines in `gres.conf` should look like:

```
NodeName=master Name=gpu File=/dev/nvidia0
NodeName=master Name=gpu File=/dev/nvidia1
```

If you have multiple GPUs, keep adding lines for each node and increment the last number after nvidia.

To make GPUs shareable by jobs, we need to use MPS ([https://slurm.schedmd.com/gres.html#MPS_Management](https://slurm.schedmd.com/gres.html#MPS_Management)):

```
# Example 2 of gres.conf
# Configure support for four different GPU types (with MPS)
AutoDetect=nvml
Name=gpu Type=gtx1080 File=/dev/nvidia0 Cores=0,1
Name=gpu Type=gtx1070 File=/dev/nvidia1 Cores=0,1
Name=gpu Type=gtx1060 File=/dev/nvidia2 Cores=2,3
Name=gpu Type=gtx1050 File=/dev/nvidia3 Cores=2,3
Name=mps Count=1300   File=/dev/nvidia0
Name=mps Count=1200   File=/dev/nvidia1
Name=mps Count=1100   File=/dev/nvidia2
Name=mps Count=1000   File=/dev/nvidia3
```

(I think `Cores` binds the GPU to specific CPUs.)

As far as I can tell, `MPS` refers to an arbitrary number. In the examples it's kind of like percent, but it doesn't have to be that way. I propose to use GPU memory in GB.
**Note:** while the number is basically arbitrary, there is still an upper limit. I suspect this has to do with how fine-grained a GPU can be partitioned. The limit likely depends on the GPU, but I've encountered 1024 as one upper limit.

`slurm.conf` then needs to contain something like

```
NodeName=tux[1-16] Gres=gpu:2,mps:200
```

> For example, a job request for "--gres=mps:50" will not be satisfied by using 20 percent of one GPU and 30 percent of a second GPU on a single node. Multiple jobs from different users can use MPS on a node at the same time. Note that GRES types of GPU and MPS can not be requested within a single job.

Another example for `slurm.conf`:

```
# Configure support for our four GPUs (with MPS), plus bandwidth
GresTypes=gpu,mps
NodeName=tux[0-7] Gres=gpu:tesla:2,gpu:kepler:2,mps:400
```

(Gres has more options detailed in the docs: https://slurm.schedmd.com/slurm.conf.html (near the bottom).)

Another thing that needs to be configured in `slurm.conf` are *ports*. Specify ports that are not blocked.
`slurmctl` and `slurmdb` only run on the master node, so there is no need to open communication between those two for the firewall.
`SlurmctldPort` definitely needs to be open; and `SlurmdPort` likely as well.

This should probably be removed: `DefMemPerNode=64000`.

### Scheduling jobs by CPU or by memory
The `SelectTypeParameters` parameter determines whether jobs are primarily scheduled by requested number of CPU or by requested memory.
The two values are `CR_Core` and `CR_Core_Memory`.
If you choose `CR_Core_Memory`, then, by default, all jobs request *all* memory on a node.
This means, in order to run multiple jobs per node at all, you need to then always specify `--mem=xxx` (where `xxx` is the requested memory in MB).

However, we can use `DefMemPerCPU` and `DefMemPerGPU` to avoid having to specify `--mem` all the time.

### Distribute config

Finally, we need to copy .conf files on **all** machines. This includes the `slurm.conf` file, `gres.conf`, `cgroup.conf` , and `cgroup_allowed_devices_file.conf`. Without these files it seems like things don’t work.

```
sudo cp /storage/ubuntu-slurm/cgroup* /etc/slurm/
sudo cp /storage/slurm.conf /etc/slurm/
sudo cp /storage/gres.conf /etc/slurm/
```

This directory should also be created on workers:

```
sudo mkdir -p /var/spool/slurm/d
sudo chown slurm /var/spool/slurm/d
```

After the conf files have been copied to all workers and the master node, you may want to reboot the computers, or at least restart the slurm services:

Workers: `sudo systemctl restart slurmd` Master:

```
sudo systemctl restart slurmctld
sudo systemctl restart slurmdbd
sudo systemctl restart slurmd
```

Next we just create a cluster: `sudo sacctmgr add cluster compute-cluster`

## Configure cgroups

I think cgroups allows memory limitations from SLURM jobs and users to be implemented. Set memory cgroups on all workers with:

```
sudo nano /etc/default/grub
And change the following variable to:
GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1"
sudo update-grub
```

Finally at the end, I did one last `sudo apt update`, `sudo apt upgrade`, and `sudo apt autoremove`, then rebooted the computers: `sudo reboot`

# Troubleshooting

When in doubt, first try updating software with `sudo apt update; sudo apt upgrade -y` and rebooting (`sudo reboot`).

## Log files

When in doubt, you can check the log files. The locations are set in the `slurm.conf` file, and are `/var/log/slurmd.log` and `/var/log/slurmctld.log` by default. Open them with `sudo nano /var/log/slurmctld.log`. To go to the bottom of the file, use ctrl+_ and ctrl+v. I also changed the log paths to `/var/log/slurm/slurmd.log` and so on, and changed the permissions of the folder to be owner by slurm: `sudo chown slurm:slurm /var/log/slurm`.

## Checking SLURM states

Some helpful commands:

`scontrol ping` – this checks if the controller node can be reached. If this isn’t working (i.e. the command returns ‘DOWN’ and not ‘UP’), you might need to allow connections to the slurmctrlport (in the slurm.conf file). This is set to 6817 in the config file. To allow connections with the firewall, execute:

`sudo ufw allow from any to any port 6817`

and

`sudo ufw reload`

## Error codes 1:0 and 2:0

If trying to run a job with `sbatch` and the exit code is 1:0, this is usually a file writing error. The first thing to check is that your output and error file paths in the .job file are correct. Also check the .py file you want to run has the correct filepath in your .job file. Then you should go to the logs (`/var/log/slurm/slurmctld.log`) and see which node the job was trying to run on. Then go to that node and open the logs (`/var/log/slurm/slurmd.log`) to see what it says. It may say something about the path for the output/error files, or the path to the .py file is incorrect.

It could also mean your common storage location is not r/w accessible to all nodes. In the logs, this would show up as something about permissions and unable to write to the filesystem. Double-check that you can create files on the /storage location on all workers with something like `touch testing.txt`. If you can’t create a file from the worker nodes, you probably have some sort of NFS issue. Go back to the NFS section and make sure everything looks ok. You should be able to create directories/files in /storage from any node with the admin account and they should show up as owned by the admin user. If not, you may have some issue in your /etc/exports or with your GID/UIDs not matching.

If the exit code is 2:0, this can mean there is some problem with either the location of the python executable, or some other error when running the python script. Double check that the srun or python script is working as expected with the python executable specified in the sbatch job file.

## Node is stuck draining (drng from `sinfo`)

This has happened due to the memory size in slurm.conf being higher than actual memory size.
(This can also happen with the MPS count.)
Double check the memory (e.g., with `free -m` or `sudo slurmd -C`) and update slurm.conf on all machines in the cluster.
Then run:
```
sudo scontrol update nodename=<nodename> state=resume
```

## Nodes are not visible upon restart

After restarting the master node, sometimes the workers aren’t there. I’ve found I often have to do `sudo scontrol update NodeName=worker1 State=RESUME` to get them working/available.

## Taking a node offline

The best way to take a node offline for maintenance is to drain it: `sudo scontrol update NodeName=worker1 State=DRAIN Reason='Maintenance'`

Users can see the reason with `sinfo -R`

## Testing GPU load

Using `watch -n 0.1 nvidia-smi` will show the GPU load in real-time. You can use this to monitor jobs as they are scheduled to make sure all the GPUs are being utilized.

## Setting account options

You may want to limit jobs or submissions. Here is how to set attributes (-1 means no limit):

```
sudo sacctmgr modify account students set GrpJobs=-1sudo sacctmgr modify account students set GrpSubmitJobs=-1sudo sacctmgr modify account students set MaxJobs=-1sudo sacctmgr modify account students set MaxSubmitJobs=-1
```

# Better sacct

This shows all running jobs with the user who is running them.

`sacct --format=jobid,jobname,state,exitcode,user,account`

More on sacct [here](https://slurm.schedmd.com/sacct.html).

# Changing IPs

If the IP addresses of your machines change, you will need to update these in the file `/etc/hosts` on all machines and `/etc/exports` on the master node. It’s best to restart after making these changes.

# Node not able to connect to slurmctld

If a node isn’t able to connnect to the controller (server/master), first check that time is properly synced. Try using the `date` command to see if the times are synced across the servers.

# Running a demo file

To test the cluster, it’s useful to run some demo code. Since Keras is within TensorFlow from version 2.0 onward, there are two demo files under the folder ‘demo_files’. tf1_test.py is for TensorFlow 1.x, and tf2_test.py is for TensorFlow 2.x.

To run the demo file, it’s best to first just run it with the system Python to see if it works. You can run `python tf2_test.py`, and if it works then you can proceed to the next step. If not, check the Python path (`which python`) to make sure it’s using the correct Python executable.

To ensure you’re using the GPU and not CPU, you can run `nvidia-smi` to watch and make sure the GPU memory is getting used while running the file. `watch -n 0.1 nvidia-smi` will show the GPU memory updated every 0.1 second.

Once the TensorFlow demo file works, you can try submitting it as a SLURM job. This uses the test.job file. Run `sbatch test.job`. Then you can check the status of the job with `sacct`. This should show ‘running’, and you should see GPU being used on one of the worker nodes. To specify the exact worker node being used, add a line to the .job file: `#SBATCH --nodelist=worker_name` where ‘worker_name’ is the name of the node. You should also be able to use `sinfo` to check which nodes are running jobs.
