---
layout: default
title:  "DeepSeek 3FS: non-RDMA install, faster ecosystem app dev/testing."
date:   2025-04-01 11:00:00 +0800
---

With M3FS, users can easily set up and explore DeepSeek 3FS in a non-RDMA virtual machine environment.

## Preface

Currently, DeepSeek 3FS only supports installation in an RDMA environment, which significantly increases the difficulty of learning and research.  A key question arises: can  open-source contributor research on 3FS using non-RDMA virtual machines?

One of M3FS's primary goals is to simplify 3FS deployment. To achieve this, M3FS supports using RXE to simulate RDMA, enabling 3FS to be installed in a common virtualization environment.


In this guide,, we'll show how to install DeepSeek 3FS in a non-RDMA virtual machine environment and explore 3FS.


## Environmental Planning

We provisioned three virtual machines, each equipped with an 8-core CPU, 16GB of RAM, and a 200GB system disk. These VMs can be deployed on virtualization platforms like VMware and KVM.


The role configuration of the 3FS component in this test is shown in the following figure.

![list-nodes]({{ site.url }}/assets/imgs/20250401-3fs-cluster-overview.png)


## Virtual Machine Configuration

Install Ubuntu 22.04.5 OS on the virtual machine. You can use <https://releases.ubuntu.com/jammy/ubuntu-22.04.5-live-server-amd64.iso> for the installation. During the OS installation process, be careful not to select any options under "Featured server snaps".

After installing the OS，Edit the /etc/hosts file on each node and add the following content.
```
10.16.189.27  open3fs-node01
10.16.189.32  open3fs-node02
10.16.189.34  open3fs-node03
```
Note that you need to remove the line "127.0.0.1  open3fs-node01" from the /etc/hosts file.

Configure passwordless SSH login among the virtual machines.

Install Docker Community Edition (Docker CE) on the virtual machines. Refer to the installation instructions for Ubuntu here: <https://docs.docker.com/engine/install/ubuntu/>.

## Deploy a 3FS Cluster with M3FS

We use the open3fs-node01 node as the main operating node during deployment.
### Installation of M3FS

Install m3fs on the open3fs-node01 node.
```
mkdir m3fs
cd m3fs
curl -sfL https://artifactory.open3fs.com/m3fs/getm3fs | sh 
```

### Create cluster.yml

Execute the following command to generate the cluster.yml configuration file.
```
./m3fs config create
```

### Edit cluster.yml

Edit the cluster.yml file. If you need to modify more configurations for 3FS cluster, please refer to <https://github.com/open3fs/m3fs/blob/main/cluster.yml.sample> .


When installing 3FS in a virtual machine environment and a non-RDMA environment, key settings are required.
- networkType: "RXE"   ## Simulate RDMA using RXE
- diskType: "dir"            ## Use directories to emulate NVMe drives
- logLevel: "INFO"         ## Set the log level of the 3FS component. The default is INFO and can be set to DEBUG

```yaml
name: "open3fs"
workDir: "/opt/3fs"
# networkType configure the network type of the cluster, can be one of the following:
# - RDMA: use RDMA network protocol
# - ERDMA: use aliyun ERDMA as RDMA network protocol
# - RXE: use linux rxe kernel module to mock RDMA network protocol
networkType: "RXE"
logLevel: "INFO"
nodes:
  - name: open3fs-node01
    host: "10.16.189.27"
    username: "root"
  - name: open3fs-node02
    host: "10.16.189.32"
    username: "root"
  - name: open3fs-node03
    host: "10.16.189.34"
    username: "root"
services:
  client:
    nodes:
      - open3fs-node01
    hostMountpoint: /mnt/3fs
  storage:
    nodes:
      - open3fs-node01
      - open3fs-node02
      - open3fs-node03
    # diskType configure the disk type of the storage node to use, can be one of the following:
    # - nvme: NVMe SSD
    # - dir: use a directory on the filesystem
    diskType: "dir"
  mgmtd:
    nodes:
      - open3fs-node01
      - open3fs-node02
      - open3fs-node03
  meta:
    nodes:
      - open3fs-node01
      - open3fs-node02
      - open3fs-node03
  monitor:
    nodes:
      - open3fs-node01
  fdb:
    nodes:
      - open3fs-node01
      - open3fs-node02
      - open3fs-node03
  clickhouse:
    nodes:
      - open3fs-node01
images:
  registry: ""
  3fs:
    repo: "open3fs/3fs"
    tag: "20250327"
  fdb:
    repo: "open3fs/foundationdb"
    tag: "7.3.63"
  clickhouse:
    repo: "open3fs/clickhouse"
    tag: "25.1-jammy"
```

### Download the 3FS container image

Execute the following command to download the container image of 3FS, with a total size of about 7GiB.

```
./m3fs artifact download -o 3fs_images  -c cluster.yml
```


### Prepare the 3FS environment，Set the RXE module

The next command will distribute the 3FS container images to all nodes and automatically perform environment configuration. In a non-RDMA environment, RXE will be configured.
```
./m3fs cluster prepare -a 3fs_images -c cluster.yml
```

Check whether the RXE module has been loaded.

![list-nodes]({{ site.url }}/assets/imgs/2025-04-01-rxe.png)


### Create the 3FS cluster
Execute the following command to deploy the 3FS cluster within 30 seconds.
```
./m3fs cluster create -c cluster.yml
```

### Check the 3FS cluster
After the 3FS cluster is deployed, we can execute the following command to check the list of 3FS roles.
```
/opt/3fs/admin_cli.sh list-nodes
```

![list-nodes]({{ site.url }}/assets/imgs/2025-04-01-list-nodes.png)
You can check the capacity statistics of 3FS file storage.

![list-nodes]({{ site.url }}/assets/imgs/2025-04-01-df.png)

### Simple FIO Test

Execute the following command on the terminal of the open3fs-node01 node. This fio test is a 1MB sequential read test of two 256M files. 
```
fio  -name=2file_8depth_1M_direct_read_bw  -numjobs=2  -iodepth=8 -ioengine=libaio -direct=1 -rw=read -bs=4M --group_reporting -size=256M -time_based  -runtime=30  -directory=/mnt/3fs/   --eta-newline=1
```

![list-nodes]({{ site.url }}/assets/imgs/2025-04-01-fio.png)

## Explore DeepSeek 3FS

We can explore DeepSeek 3FS in a virtual machine environment. For example, you can explore the underlying storage structure of 3FS, metadata stored in fdb, and so on.

### Check the logs of the 3FS component

View the logs of the 3FS component.
```
 tail -n 20 /opt/3fs/storage/log/storage_main.log
 ```

![list-nodes]({{ site.url }}/assets/imgs/2025-04-01-log.png)

### Check the underlying storage structure of 3FS

We can check the specific definition of 3FS Target.

![list-nodes]({{ site.url }}/assets/imgs/2025-04-01-target1.png)

There are 32 targets on this storage node.


![list-nodes]({{ site.url }}/assets/imgs/2025-04-01-target2.png)

We can see that ChunkEngine manages files of various ChunkSizes for space allocation.

![list-nodes]({{ site.url }}/assets/imgs/2025-04-01-chunk-dirs.png)
![list-nodes]({{ site.url }}/assets/imgs/2025-04-01-chunk-dirs-file.png)

ChunkEngine uses RocksDB to store metadata.


![list-nodes]({{ site.url }}/assets/imgs/2025-04-01-chunkengine-meta.png)

### Get the chunksize and stripesize parameters of the 3FS directory

With the following command, we can get the values of the chunksize and stripesize parameters for a specified directory.  
```
docker exec -it 3fs-mgmtd /opt/3fs/bin/admin_cli -cfg /opt/3fs/etc/admin_cli.toml "stat --display-layout --display-chain-list --display-chunks --display-all-chunks /"
```


![list-nodes]({{ site.url }}/assets/imgs/2025-04-01-get-chunksize.png)

### Modify the chunksize and stripesize of the 3FS directory

Create a new directory /mycs_128k and specify the chunksize and stripesize parameters using the following command.
```
docker exec -it 3fs-mgmtd /opt/3fs/bin/admin_cli -cfg /opt/3fs/etc/admin_cli.toml  "mkdir --recursive --perm 0755 --chain-table-id 1 --chain-table-ver 1 --chunk-size 131072 --stripe-size 32 mycs_128k "
```
![list-nodes]({{ site.url }}/assets/imgs/2025-04-01-set-chunksize.png)

### Use fdbcli to get keys in FDB

We know that 3FS uses fdb to store the keys of metadata, where the prefix of directory entries is DENT and the prefix of file inodes is INOD.
docker exec -it 3fs-fdb fdbcli --exec 'getrangekeys \x00 \xff 10000'  | grep "[DENT|INOD]"


![list-nodes]({{ site.url }}/assets/imgs/2025-04-01-get-fdb-key.png)

## Summary

With M3FS, users can easily set up and explore DeepSeek 3FS in a non-RDMA virtual machine environment.