# HA Docker Cluster with Swarm and Ceph

    2020 Ondrej Sika <ondrej@ondrejsika.com>
    https://github.com/ondrejsika/ha-docker-cluster-with-swarm-and-ceph


## HW Requirements

- 3 nodes
- 2 hdd (one for system, one for ceph)

## Software Requirements

- Debian 10
- Docker
- Ceph (Nautilus)
- Ceph Deploy

## Install Docker

On all 3 nodes install Docker

```
curl -fsSL https://ins.oxs.cz/docker.sh | sudo sh
```

## Install Ceph Nautulus on Debian 10

Ceph does'n have official repositories for Debian 10 Buster yet (29-01-2020), we use [Croit](https://croit.io) mirror. See their blog post <https://croit.io/2019/07/07/2019-07-07-debian-mirror>

```
# Add repositories
curl https://mirror.croit.io/keys/release.gpg > /etc/apt/trusted.gpg.d/croit-mirror.gpg
echo 'deb [signed-by=/etc/apt/trusted.gpg.d/croit-mirror.gpg] https://mirror.croit.io/debian-nautilus/ buster main' > /etc/apt/sources.list.d/croit-ceph.list
printf 'Package: *\nPin: origin mirror.croit.io\nPin-Priority: 990\n' > /etc/apt/preferences.d/pin-mirror-croit-io
apt update

# Check Apt Pin
apt-cache policy ceph

# Install CEPH
apt install -y ceph

# Install NTP
apt install -y ntpsec
```

## Install Ceph Deploy

Install ceph-deploy on first node

```
# Ensure Python 3
apt install -y python3 python3-pip

# Install Ceph Deploy
pip3 install ceph-deploy
```

## Setup hostnames

Set fqdn hostname `node{1,2,3}.sikademo.com` to every node (in `/etc/hostname`).

## Setup /etc/hosts

Add those lines to `/etc/hosts` in every node

```
192.168.0.11 node1 node1.sikademo.com
192.168.0.12 node2 node2.sikademo.com
192.168.0.13 node3 node3.sikademo.com
```

## Setup Ceph Cluster

See more: <https://docs.ceph.com/docs/master/start/quick-ceph-deploy>


Create directory `ceph-deploy` and checkout there.

```
mkdir ~/ceph-deploy
cd ~/ceph-deploy
```

```
ceph-deploy new node1 node2 node3
```

Add Ceph network to `ceph.conf`

```
echo public network = 192.168.0.0/24 >> ceph.conf
```

If you follow tutorial on Ceph, skip ceph-deploy install - we have ceph already installed and ceph-deploy install does't work on Debian 10 yet (29-01-2020).

Setup monitors

```
ceph-deploy mon create-initial
```

Setup admin nodes (you can manage ceph on those nodes)

```
ceph-deploy admin node1 node2 node3
```

Create OSDs

```
ceph-deploy osd create --data /dev/sdb node1
ceph-deploy osd create --data /dev/sdb node2
ceph-deploy osd create --data /dev/sdb node3
```

Check Ceph health

```
ceph health
```

Check Ceph status

```
ceph -s
```

## Setup CephFS

See more: <https://docs.ceph.com/docs/master/start/quick-cephfs/>


Create metadata servers on each node

```
ceph-deploy mds create node1 node2 node3
```

Create Filesystem

```
ceph osd pool create cephfs_data 32
ceph osd pool create cephfs_meta 32
ceph fs new cephfs cephfs_meta cephfs_data
```

Mount CephFS

```
mount -t ceph :/ /cephfs -o name=admin
```