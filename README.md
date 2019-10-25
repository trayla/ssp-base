# SSP - Single Server Platform

## Prerequisits

### Hardware requirements

- A running bare metal machine with a plain Ubuntu 18.04 Server installation and root access (virtual machines are not supported)
- Minimum 16 GB RAM (minimum 8 GB with slightly modified parameters)
- Minimum 1 TB storage (minimum 150 GB with slightly modified parameters)

### Knowledge

- A understanding of Linux based system management and command line tools
- A understanding of virtualization with KVM
- Knowledge about operating a Kubernetes platform
- A basic understanding about Gluster

### Necessary preliminaries

#### Storage

Your system has to provide three directories which will be used as KVM storage pools and therefore populated with disk images for the upcoming virtual machines. The following directories have to be create prior to the start of the installation procedure:

A sample setup could be to use two free partition on independent disks managed by the Logical Volume Manager (LVM). In the following example we are using XFS as the file system. This is not necessary but a good choice. In order to use it on a Ubuntu server you have to install the appropriate package:

~~~~ShellSession
apt install -y xfsprogs
~~~~

Initialize the LVM on two disks. Please replace /dev/sda4 and /dev/sdb4 with the device of your choice.

Create two physical volumes, each on one disk:
~~~~ShellSession
pvcreate /dev/sda4
pvcreate /dev/sdb4
~~~~

Create two volume groups, each for one disk:
~~~~ShellSession
vgcreate vga /dev/sda4
vgcreate vgb /dev/sdb4
~~~~

##### /vmpool

This directory is going to be used as the default storage pool for all virtual machine images except the data images. For redundancy reasons it is recommended to store this directory at least on a RAID 1 device.

Create one logical volume on each volume group as a base of the default storage pool:
```ShellSession
lvcreate --size 150g -n lvp vga
lvcreate --size 150g -n lvp vgb
```

Create a RAID1 array of both newly created logical volumes to ensure redundancy:
```ShellSession
mdadm --create /dev/md10 --level=mirror --raid-devices=2 /dev/vga/lvp /dev/vgb/lvp
```

Verify the RAID disk with the following command:
```ShellSession
cat /proc/mdstat
```

Format the RAID disk with a XFS filesystem:
```ShellSession
mkfs.xfs /dev/md10
```

Create the directory of the default storage pool:
```ShellSession
mkdir -p /vmpool
```

Add the newly created RAID disk to the /etc/fstab file to be automatically mounted after system restarts:
```ShellSession
echo "/dev/md10 /vmpool xfs defaults 0 0" >> /etc/fstab
```

Mount the default storage pool:
```ShellSession
mount /vmpool
```

##### /data1 and /data2

These directories are going to be used as redundancy nodes for the storage cluster and should be on seperate storage disks. Using a high available disk setup like RAID 1 or 5 is not necessary here due to the redenundany of the upcoming storage cluster.

Create the first data disk (replace /dev/sda4 by your partition device and "200g" by a storage size of your choice in giga bytes):

Create the directories of the data storage pools:
```ShellSession
mkdir -p /data1
mkdir -p /data2
```

Create logical volumes for the data storage pools:
```ShellSession
lvcreate --size 500g -n lvd vga
lvcreate --size 500g -n lvd vgb
```

Format the disks with the XFS filesystem:
```ShellSession
mkfs.xfs /dev/vga/lvd
mkfs.xfs /dev/vgb/lvd
```

Add the newly created disk to the /etc/fstab file to be automatically mounted after system restarts:
```ShellSession
echo "/dev/vga/lvd /data1 xfs defaults 0 0" >> /etc/fstab
echo "/dev/vgb/lvd /data2 xfs defaults 0 0" >> /etc/fstab
```

Mount the first data storage disk:
```ShellSession
mount /data1
mount /data2
```

#### A sudo user for system administration

Create a system administrator user and disable root
```ShellSession
adduser sysadm
usermod -aG sudo sysadm
passwd -l root
```

#### Install scripts

In order to execute the scripts you have to clone this GitHub repository to your server into the directory /opt/mgmt/ssp-base by issuing the following commands:
```ShellSession
mkdir -p /opt/mgmt/ssp
git clone https://github.com/trayla/ssp.git /opt/mgmt/ssp
chown -R sysadm:sysadm /opt/mgmt
```

The values file defines specific customizations of your own topology. A sample file is included in this repository. It should be copied to /opt/mgmt and customized before further installation.
```ShellSession
cp /opt/mgmt/ssp/values-default.yaml /opt/mgmt/values.yaml
```

#### Domain

The Single Server Platform provides a lot of services to the outside world. In order to access these services we are registering them as sub domains of a configurable main domain. The most comfortable way is to have a main domain like 'example.com' which points to your platform IP address by a wildcard DNS entry like this '*.example.com > 88.77.66.55'. In this case the platform can route any subdomain to the desired service by itself.

## Usage

Prepare your host with the following command. This is necessary only once while you can install and remove the platform from your host as much as you like. During this process you have to accept the licence agreement of the Ansible product suite.
```ShellSession
sudo /opt/mgmt/ssp/platform.sh prepare
```

Install the platform with the following command:
```ShellSession
sudo /opt/mgmt/ssp/platform.sh install
```

This command removes the whole plattform from your host:
```ShellSession
sudo /opt/mgmt/ssp/platform.sh remove
```

After completion the system will be restarted. It takes a couple of until all virtual machines and services are up an running.

## Result

If everything worked as expected you should have the following setting on your machine.

### Architectural Overview

This picture shows an architectural overview of the desired platform:

![system landscape](/docs/systemlandscape.svg)

### Virtual machines:

The virtual machines are available through the KVM standard command line tools:
- `virsh list --all` lists all virtual machines
- `virsh pool-list --all` lists all storage pools
- `virsh pool-info <poolname>` shows the details of the desired storage pool
- `virsh --help` shows all available commands of the virtual machine management

You can gain shell access to the desired virtual machine either by opening a KVM console
```ShellSession
virsh console <vmname>
```
or by connection to the virtual machine over SSH (password is "pw")
```ShellSession
ssh sysadm@<ipaddr>
```

#### kubemaster

Purpose: Kubernetes master

IP Address: 10.88.20.109

#### kubenode0

Purpose: Kubernetes worker node with storage arbiter

IP Address: 10.88.20.110

#### kubenode1

Purpose: First Kubernetes worker node with data storage leg 1

IP Address: 10.88.20.111

#### kubenode2

Purpose: Second Kubernetes worker node with data storage leg 2

IP Address: 10.88.20.112
