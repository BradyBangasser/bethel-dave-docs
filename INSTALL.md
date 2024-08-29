# Install Slurm on RPI
These are instructions to setup a cluster using Slurm as a workload manager
Note: These instructions were developed using 8 Raspberry Pi 4Bs and these instruction are likely not adequate for full scale production servers

## Choosing an operating system
There aren't many choices for the RPI OS, I recommended the Raspberry PI OS 64 bit with the Desktop GUI on the master node and the Raspberry Pi OS Lite 64 bit for all of the other nodes

## Makefile
Remember -j is used to specify the amount of threads used to compile
```bash
make -j 8 # Make with 8 threads
```

## Setting up NFS Drive
##### This section is derived heavily from [this medium article](https://glmdev.medium.com/building-a-raspberry-pi-cluster-784f0df9afbd)
#### You will need to decide whether you want to have your NFS server on your Master Node or another Node, the setup process is the same for each
### 1. Find the UUID and dev location of the drive you want to use for your NFS server
### 2. Format the drive to ext4
```bash
sudo mkfs.ext4 /dev/(your dev location here)
```
### 3. Make the mount directory for the drive, we used /clusterfs but it can be anything

```bash
mkdir /clusterfs
```

### 4. Set drive permissions<br>
You will need to decide whether you want to install the Slurm binaries and dependencies on each node or on the cluster NFS drive<br>

#### Here are some advantages and disadvantages of each

| On NFS      | On Node |
| :----:        |    :----:   |
| Slower      | Faster       |
| Fairly easy to maintain   | More difficult and time consuming to maintain |
| Easier to set up | Harder to set up |
| Less Secure | More Secure |

Generally I would recommend initially installing the binaries and libs on the NFS drive and eventually installing the binaries and libs on all of the Nodes.

#### If you choose to install on Each Node
Set the permissions of however you see fit, I recommend setting the drive to 0777 nobody:nogroup and fine tuning the permissions after the cluster is working.
```bash
sudo chmod 777 -R /clusterfs
sudo chown nobody:nogroup /clusterfs
```

#### Else
You will need to set the drive owner as root and permissions as 777
```bash
sudo chmod 777 -R /clusterfs
sudo chown root:root /clusterfs
```
### 5. Set up auto mounting on the NFS device
We need to set up the device to auto mount on boot, because manually mounting it every time the system booted would not be very efficient. All we need to do is to add the device to /etc/fstab.
```bash
echo "UUID=whateveryourUUIDishere /clusterfs ext4 defaults 0 2" | sudo tee >> /etc/fstab
```

### 6. Mount the drive
```bash
sudo mount -a
```

### 7. Setting up the actual NFS server (This is where the fun begins)
We need to install the NFS server package
```bash
sudo apt install nfs-kernel-server
```

After it is installed we must add our cluster directory to /etc/exports, you will need to decide on what IP scheme you will be using, or check what your network is using. If you are able to setup your own network, I recommend a 10. scheme because it will allow for greater organization of your network.
```bash
echo "/clusterfs 10.1.0.0/24(rw,sync,no_squash_root,no_subtree_check)" | sudo tee >> /etc/exports
```
rw means read and write, sync forces all transactions to be written, no_squash_root allows other root users to write files as root, no_subtree_check prevents errors from multiple systems using the drive

### 8. Setting up the clients
This should be done after configuring the network on every node except for the one hosting the NFS server

#### Install nfs-common
```bash
sudo apt install nfs-common
```

#### Set up auto mounting, because manual mounting would be annoying
```bash
echo "IPorHostnameOfNodeWithNFSServerHere:/clusterfs /clusterfs nfs defaults 0 0" | sudo tee >> /etc/fstab
```

#### Mount and Test NFS server
```bash
sudo mount -a
cd /clusterfs
# sanity check
touch $(hostname)
```
You should now be able to see that file on all devices with NFS set up

### Now the NFS server should be set up

## Creating configuration files

## Installing Slurm
You should start by install slurm on the Master Node

### All Nodes
These steps should be followed on all nodes, including the master node

#### Install OpenSSL
You can download and build OpenSSL from source but that is fairly difficult, so I recommend just installing it using apt
```bash
sudo apt install openssl
```

#### Install NTP
This can also be download and build but that is very time consuming so I also recommend just using apt
```bash
sudo apt install ntp
```

### Create Munge and Slurm users
```bash
sudo adduser -r slurm
sudo adduser -r munge
```

### Make the folders required by munge and slurm
```bash
sudo mkdir /var/log/munge /var/lib/munge /var/run/munge
sudo chown munge:munge /var/log/munge /var/run/munge
sudo mkdir /var/spool/slurmd
sudo chown slurm:slurm /var/spool/slurmd
```
## 1. Downloading and Compiling
### If installing Binaries on each node
Note: You may be able to compile these on one node and then just copy them, but for simplicity sake we will compile and install everything locally
#### Download and Install HWLOC and libevent
Download, compile, and install [HWLOC](https://www.open-mpi.org/projects/hwloc/) and [libevent](https://libevent.org/), the default options should be fine
#### Download and install Munge
Download, compile, and install [Munge](https://github.com/dun/munge/releases/tag/munge-0.5.16)
The default options should be fine
#### Download and Install OpenPMIx
Download, compile, and Install [openPMIx](https://docs.openpmix.org)([Release Download](https://github.com/openpmix/openpmix/releases)), The default options should be fine
#### Download and Install Slurm
Download, compile, and Install [Slurm](https://www.schedmd.com/download-slurm/), default options should be fine
### Else
Follow the steps under master node to install the binaries to the NFS server, then follow the steps to set up each node with the NFS installed binaries

## 2. Installing
### If installing Binaries on each node
#### Master Node
These steps are only to be done on the master node<br>

* Copy the slurmctld service file to /lib/systemd/system
* Enable the slurmctld daemon
```bash
sudo systemctl enable slurmctld
```
* Ensure the slurm.conf file is in the correct location (usually it's either /etc/slurm or /etc)
* Start slurmctld
```bash
sudo systemctl start slurmctld
```
* Ensure slurmctld was started
```bash
systemctl status slurmctld
```

### Else
* Download [HWLOC](https://www.open-mpi.org/projects/hwloc/) and [libevent](https://libevent.org/)
* Configure both of them with the prefix /clusterfs and their local, shared, and run state directories at their default locations
```bash
# This should work on both HWLOC and libevent
./configure --prefix=/clusterfs --sharedstatedir=/com --localstatedir=/var --runstatedir=/run
```
* Compile and install both
```bash
make -j 8
sudo make install
```
* 


set up automounting on nfs drive
chown root:root mounted dir

add /clusterfs/bin to path
add /clusterfs/lib to ld_library_path

Install OpenSSL library (libssl-dev), this can be done using apt
Install [NTP](https://support.ntp.org/Support/InstallingNTP), this can be done using apt
Install [Munge](https://github.com/dun/munge), configure prefix to nfs drive, configure state dirs to /
mkdir /var/log/munge /var/lib/munge
chown munge:munge /var/log/munge
sudo systemctl enable $(path to munge)
Create Munge User (munge)
chown munge:munge /clusterfs/etc/munge
mkdir -p /clusterfs/var/run/munge
chown munge:munge !$
sudo systemctl start munge
Install [libdbus]()
Install [HWLOC](https://www.open-mpi.org/projects/hwloc/) and [libevent](https://libevent.org/), this can be done concurrently 
Install [openPMIx](https://docs.openpmix.org) [Release Download](https://github.com/openpmix/openpmix/releases), I believe the default install path for the libs is /usr/local/lib

Install Slurm
create slurm user
cp all service files to /clusterfs/lib/systemd/system
Add After clusterfs.mount to all 
# master
mkdir /var/log/
touch /var/log/slurmctld.log
chown slurm:slurm !$
mkdir /var/spool/slurmctld
chown slurm:slurm !$

# Child Node
configure fstab
configure /etc/hosts and /etc/hostname
copy required .service files (from /clusterfs/lib/systemd/system) to /lib/systemd/system and enable them

start munge daemon
mkdir /var/spool/slurmd
chown slurm:slurm !$
start slurmd 