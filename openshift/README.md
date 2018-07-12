# Pre-requisites

Steps to be done before installing the cluster.

## Basic OS

Install RHEL 7 and subscribe each machine:

    subscription-manager register
    subscription-manager attach --pool=<your pool id>
    subscription-manager repos --disable='*'
    subscription-manager repos \
        --enable=rhel-7-server-rpms \
        --enable=rhel-7-server-optional-rpms \
        --enable=rhel-7-server-extras-rpms \
        --enable=rhel-7-server-ose-3.9-rpms \
        --enable=rhel-7-fast-datapath-rpms \
        --enable=rhel-7-server-ansible-2.4-rpms \
        --enable=rh-gluster-3-client-for-rhel-7-server-rpms

Update and install packages:

    yum -y update
    yum -y install wget git net-tools bind-utils yum-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct
    yum -y install glusterfs-fuse
    yum -y install NetworkManager
    yum -y install atomic-openshift-utils

Add fancy monitoring:

    yum -y install ftp://ftp.tu-chemnitz.de/pub/linux/epel/7/x86_64/Packages/p/python2-psutil-2.2.1-3.el7.x86_64.rpm ftp://ftp.tu-chemnitz.de/pub/linux/epel/7/x86_64/Packages/g/glances-2.5.1-1.el7.noarch.rpm

Reboot the machine:

    reboot

## Docker

Then install docker:

    yum -y install docker-1.13.1

**Note:** Do not start Docker yet!

Create the docker volume group:

    parted /dev/sdb mktable msdos
    parted /dev/sdb mkpart primary 1 100GB
    parted /dev/sdb set 1 lvm on
    pvcreate /dev/sdb1
    vgcreate docker-vg /dev/sdb1

Configure the thin pool storage options:

    cat <<EOF > /etc/sysconfig/docker-storage-setup
    VG=docker-vg
    EOF

Then start it:

    systemctl enable docker
    systemctl start docker

Check if docker is successfully initialized:

    systemctl is-active docker

# Install OpenShift

First log on to the master node and clone this git repository:

    git clone https://github.com/redhat-iot/hono-scale-test

## IoT cluster

    ansible-playbook -i hono-scale-test/openshift/iot-inventory.txt /usr/share/ansible/openshift-ansible/playbooks/prerequisites.yml
    ansible-playbook -i hono-scale-test/openshift/iot-inventory.txt /usr/share/ansible/openshift-ansible/playbooks/deploy_cluster.yml

## Simulation cluster

    ansible-playbook -i hono-scale-test/openshift/sim-inventory.txt /usr/share/ansible/openshift-ansible/playbooks/prerequisites.yml
    ansible-playbook -i hono-scale-test/openshift/sim-inventory.txt /usr/share/ansible/openshift-ansible/playbooks/deploy_cluster.yml

## Post-Installation

Afterwards create the credentials file on each master:

    htpasswd -c /etc/origin/master/htpasswd iot

# Some extras

## InfluxDB storage

For the Hono simulator we used an extra InflxuDB instance. This will be located
on an additional SSD on the Simulator cluster master.

    parted /dev/sdf mktable msdos
    parted /dev/sdf mkpart primary 1 100GB
    parted /dev/sdf set 1 lvm on
    pvcreate /dev/sdf1
    vgcreate ssd-vg /dev/sdf1
    lvcreate -L 80GB ssd-vg --name influxdb-storage
    mkfs -t xfs /dev/mapper/ssd--vg-influxdb--storage
    mkdir /exports/influxdb-storage
    echo "/dev/mapper/ssd--vg-influxdb--storage /exports/influxdb-storage   xfs    defaults 0 0" >> /etc/fstab
    mount /exports/influxdb-storage
    
    echo "/exports/influxdb-storage *(rw,root_squash)" > /etc/exports.d/simulation.exports
    exportfs -r 
