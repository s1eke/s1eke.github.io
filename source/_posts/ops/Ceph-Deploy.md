---
layout: post
title: 使用 ceph-deploy 部署 Ceph 集群
date: 2020-07-03 02:16:33
tags: 
    - Linux
    - Ceph
categories:
  - [运维]
---

# 部署 Ceph 存储集群

## 前置条件

1. 管理节点下需要将所有节点对应的主机名和 IP 写入 hosts 方便批量操作（所有节点需要用不同的主机名）
2. 所有节点配置免密登录方便管理节点使用 ceph-deploy 部署(用户需要有sudo权限，并且是NOPASSWD)
   
## 重置环境（仅在原环境上操作需要使用）

该操作需要在集群的管理节点的 ceph-deploy 目录下操作。

1. 删除所有节点的ceph软件 

        $ ceph-deploy purge {ceph-node} [{ceph-node}]

2. 删除所有节点的ceph软件数据

        $ ceph-deploy purgedata {ceph-node} [{ceph-node}]

3. 删除所有节点的keys

        $ ceph-deploy forgetkeys

4. 删除ceph配置文件

        $ rm ceph.*

## 自动部署（ceph-deploy）

### 安装 ceph-deploy

    # pip install ceph-deploy

---

### 存储集群

由于官方的源再国内访问所以需要使用第国内供的软件源，通过配置环境变量使用 aliyun 提供的 ceph-mimic 版本的仓库。

```shell
# export CEPH_DEPLOY_REPO_URL=https://mirrors.aliyun.com/ceph/debian-mimic
```

#### 创建集群

先在管理节点上创建一个目录，用于保存 ceph-deploy 生成的配置文件和密钥对。

    # mkdir my-cluster
    # cd my-cluster

然后开始创建监控节点：

    # ceph-deploy new 监控节点主机名

这里需要把所有设备配置好免密登录，以及把所有设备的主机名和IP的对应关系写入管理节点的 hosts ，防止网络发现没有生效。另外因为 ceph-deploy 是直接调用 ssh 远程操作节点，而且会使用 sudo ，而且不支持 root 登录（官方的口径是 root 不支持 sudo ，但是实际上是可以用的。），所以要配置好用户的 sudo 免密操作。

#### 配置文件

ceph 自带的容灾配置，默认是保存三份副本，可以把 `osd pool default size = 3` 写入 Ceph 配置文件的 `[global]` 段下。更改为其他数量。

另外如果你有多个网卡，也可以可以把 public network 写入 Ceph 配置文件的 `[global]` 段下。



以下是我的配置文件：

```shell
[global]
fsid = aa7852b8-dd0b-40d4-a2f4-bd1f38c6dc69
mon_initial_members = node3, node4
mon_host = 192.168.1.3,192.168.1.4
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
osd pool default pg num = 4096
osd pool default pgp num = 4096
osd pool default size = 3
mon_allow_pool_delete = true
mon clock drift allowed = 0.1
mon data avail warn = 10
osd pool default min size = 1
```


#### 为所有节点安装 Ceph

    # ceph-deploy install 所有节点主机名（包括管理节点）

#### 配置初始 monitor(s)、并收集所有密钥

    # ceph-deploy mon create-initial

如有报错，请检查管理节点的 /etc/hosts、~/.ssh/config 里的所有主机名和 IP 与监控节点的主机名和 IP 名一致。另外还有多个节点

---
### 创建集群

#### 分发管理节点的配置以及密钥

    # ceph-deploy admin  所有节点名

#### 为监控节点部署管理器守护程序

    # ceph-deploy mgr create 所有监控节点

#### 创建 OSD

    # ceph-deploy osd create --data {device} {ceph-node}
    
    示例：ceph-deploy osd create --data /dev/sdb node30

OSD 创建完毕后再检查一些集群的健康情况(当使用ceph时需要使用管理员权限)：

    # ceph health

也可以用 `ceph -s` 查看集群的详细信息。

```shell
vm@Lotus-Master ~/my-cluster % sudo ceph -s
  cluster:
    id:     8d24cae1-07a9-4c3c-a74c-d0b56e4ad243
    health: HEALTH_OK

  services:
    mon: 2 daemons, quorum node3,node4 (age 2m)
    mgr: node7(active, since 43m), standbys: node8, node9, node10, node11, node12, node15, node16, node6, node14, node18, Lotus-Master, node3, node5, node4
    osd: 14 osds: 14 up (since 2m), 14 in (since 15m)

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   14 GiB used, 114 TiB / 114 TiB avail
    pgs:
```

---




## 创建 Cephfs

### 创建 mds 节点

    # ceph-deploy mds create {ceph-node}

CephFS 中的所有元数据操作都通过 mds 进行，因此至少需要一台 mds 节点。为了防止单点故障，在这里可以配置多个mds 服务器。默认的

### 创建 pool

    # ceph osd pool create cephfs_data 2048
    # ceph osd pool create cephfs_meta 512

创建了一个名为 cephfs_data 和一个名为 cephfs_meta 的 pool 。关于 pg 数量的配置公式如下。当使用公式计算时，pg 数一定要是贴近计算值的2的指数值。譬如有两百个 OSD ，两个 pool ，三个副本，则计算值为200*100/2/3=4000，pg 值应设置为最接近 4000 的 2 的指数值 4096。其中 cephfs 的 data pool 和 metadata pool 的 pg 数按 [ceph官方邮件](http://lists.ceph.com/pipermail/ceph-users-ceph.com/2017-March/016965.html) 讨论中来说，建议是 4：1 的比例。

```
                         (OSDs * 100)
Total PGs =  ---------------------------------------
              (pool count) * (max replication count)
```

另外可通过 `sudo ceph osd lspools` 来查看已有的 pool，如果要删除 pool 可以使用以下命令：

    #  ceph osd pool delete  cephfs_data cephfs_data --yes-i-really-really-mean-it


### 创建 cephfs

    # ceph fs new mycephfs cephfs_meta cephfs_data

### 挂载 cephfs

监控节点的密钥可以通过在监控节点执行 “sudo cat /etc/ceph/ceph.client.admin.keyring|grep key|cut -d" " -f3” 获取。

    # mount -t ceph 监控节点IP:6789:/ 目录挂载点 -o name=admin,secret="监控节点密钥",acl,async,rw,noexec,nodev,noatime,nodiratime

示例：

    # mount -t ceph 192.168.1.30:6789,192.168.1.32:6789,192.168.1.34:6789,192.168.1.35:6789,192.168.1.37:6789:/ /lotus -o name=admin,secret="AQCyVdde8Y5EDBAAQCrFCJYVsmXZHAZC+4mAJQ==",acl,async,rw,noexec,nodev,noatime,nodiratime

---


## 实施中遇到的其他问题

1. 用parted分区后的分区表不会立即生效，需要重启。如果不想重启，可以使用 `partprobe` 命令，此命令可以让系统内核重新读取分区表信息，就不用重新启动电脑。

2. 如果硬盘在使用之前已经有了 GPT 分区创建 OSD 的时候就会报报 `error: GPT headers found` ，这时可以直接用 dd 将 GPT 分区删掉：

    ```shell
    # dd if=/dev/zero of=/dev/sdb bs=512K count=1
    ```
    
    如果需要添加的硬盘在已存在的lvm卷组里，譬如重置环境后需要加入新的集群的 osd ，可以用以下命令先脱离 lvm ，再删除GPT分区：

    ```shell
    # lvremove -vf `lvdisplay|grep "LV Path"|awk '{print $3}'`
    # vgremove -vf `vgdisplay|grep "VG Name"|awk '{print $3}'`
    # pvremove -vf `pvdisplay|grep "PV Name"|awk '{print $3}'`
    ```


3. 修改 ceph.conf后要使用 `ceph-deploy --overwrite-conf config push 所有节点名称` 来把新配置推送到所有节点上。


4. 如果删除 pool 报错 `Error EPERM: pool deletion is disabled; you must first set the mon_allow_pool_delete config option to true before you can destroy a pool` ，则可以添加 `mon_allow_pool_delete = true` 到 ceph.conf ，然后执行 `ceph osd pool delete  lotus lotus --yes-i-really-really-mean-it` ，或者用下面的方法

    ```shell
    # ceph tell mon.\* injectargs '--mon-allow-pool-delete=true'
    # ceph osd pool delete  lotus lotus --yes-i-really-really-mean-it
    ```

5. 错误：
   
        # rbd: create error: (33) Numerical argument out of domain
        # 2020-03-09 22:38:32.177 7fe1a39ecf40 -1 librbd::image::CreateRequest: validate_order: order must be in the range [12, 25]

    原因：`rbd_default_order` 配置过高，这个配置项决定了集群内切块的大小，默认是22，范围是[12, 25]，不在这个范围都会报错。

6. 错误：集群大小与实际存储容量不符合
   
   原因：RBD 进行数据删除的时候实际上并不会直接删除对象，而是给对象打上已删除的标记，所以删除内容还是在集群内占用空间。这里需要通过文件系统去主动释放被标记为已删除的对象 `sudo fstrim rbd挂载目录` 。

7. 错误：pg 数设置错误，需要调整
   
   解决方法（另外这里需要注意，pg 数不能一次性扩大太多，每次调整的大小尽量不要超过100%）：

        # ceph osd pool set cephfs_data pg_num 4096
        # ceph osd pool set cephfs_data pgp_num 4096

8. 错误：监控节点数量添加少了，需要调整
   
   解决方法：
   
        # ceph-deploy mon add 新增监视器主机名

    这里有一点要注意，所有监视器一定要设置 NTP 同步，尽量选择同一个 NTP 服务器进行同步。

    而一旦添加了新的Ceph监视器，Ceph将开始同步监视器并形成仲裁。可以通过执行以下操作检查仲裁状态：

        # ceph quorum_status --format json-pretty

9. 错误：cephfs出现错误需要重建，删除pool却显示pool正在被cephfs使用中
   
   解决方法：

    首先停止mds的服务

        # systemctl stop ceph-mds@$HOSTNAME

    然后 设置mds为失败

        # ceph mds fail 0    
   
    接着删除cephfs
   
        # ceph fs rm mycephfs --yes-i-really-mean-it  
   
    然后删除cephfs使用的pool
   
        # ceph osd pool delete  lotus lotus --yes-i-really-really-mean-it
---

