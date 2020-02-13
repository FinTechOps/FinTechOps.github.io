---
layout: default
title: "使用ceph-deploy安装ceph集群"
date: 2020-02-07 17:38:00
published: 2020-02-07 17:38:00
comments: true
categories: CICD
tags: [cicd]
author: ykfq
---

<p>
Ceph 是一个开源的分布式存储系统，设计初衷是提供较好的性能、可靠性和可扩展性。它支持三种存储类型：块存储、文件存储和对象存储。在 OpenStack 还如日中天的时候（2015-2017），Ceph 是其后端存储的最佳选择。
</p>

<!--more-->


## 方案选型

ceph 目前有2种比较成熟的部署方案：
- `ceph-deploy`: 官方方案
- `ceph-ansible`：redhat 维护的 ansible 项目

以前一直使用 ceph-deploy，但觉得它略微繁琐，因此本次部署之前先尝试了 ceph-ansible，但发现支持 ceph v12.0(luminous) 的 ceph-ansible 版本 3.2 存在bug，虽然在 4.0 中修复了，但 4.0 只支持 ceph v14（nautilus），目前在生产环境使用 ceph 14 还很冒险，因此放弃了这一方案。

**为何选择 ceph v12（luminous）?**

从以往的经验以及与同行交流得出了这样一个结论：  
ceph 作为开源项目，bug 相对较多，补丁版越多的主流版本稳定性更高。综合考量，当前 mimic 为 13.2.6, nautilus 为 14.2.2, luminous 为 ~~12.2.12~~(整理本文时已更新至12.2.13), luminous 的版本中包含更多的 bug 修复和性能优化，因此选择使用 luminous。



## 必要条件

- 生产环境需要至少 3 个 ceph 节点
- 每个 ceph 节点至少 1 块独立磁盘，2 张网卡（公网和私网分离）；
- 防火墙和安全组需放通端口：6789, 6800 
- 安全起见使用普通用户并添加免密 sudo 权限
  ```bash
  echo "worker ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/worker
  sudo chmod 0440 /etc/sudoers.d/worker
  ```
- 设置 SELinux 为 Permissive 
- 设置 ntp 时钟服务器
- 设置节点主机名相互解析（添加hosts或者内部DNS）



## 集群部署

我们这里因为在隔离的内网，直接使用具有 ssh 免密权限的主机作为 ceph-deploy 节点，并使用 root 账户部署。

请勿随意修改和删除部署节点上的目录：`/root/ceph-cluster`


### PG 速算  

PG(Placement Groups) 在 ceph 中扮演着重要的角色。它将存储池中的对象进行聚合，使得 ceph 定位对象变得简单。  
因为基于单个对象去跟踪对象位置及其元数据需要大量计算（计算昂贵）。设置合适的 pg 数量有利于提高集群的性能。  
PG 值也不是一成不变的，会随着 osd 和 pool 的增加而调整。

创建 ceph 存储池时，我们必须指定 pg 的值，因为它不能被自动计算。由于 pg 只能调大不能减小，一开始我们可以设置小一点，随着 osd 数量的增加，存储池的 pg 数量也应随之调整（还受存储池数量的影响）。

下面是一些通用的选择：
- Less than 5 OSDs set pg_num to 128
- Between 5 and 10 OSDs set pg_num to 512
- Between 10 and 50 OSDs set pg_num to 1024
- If you have more than 50 OSDs, you need to understand the tradeoffs and how to calculate the pg_num value by yourself
- For calculating pg_num value by yourself please take help of pgcalc tool


### 开始部署

- 设置 ceph-deploy 节点免密登录其它节点
  ```bash
  ssh-keygen
  
  ssh-copy-id ceph01
  ssh-copy-id ceph02
  ssh-copy-id ceph03
  ```


- 创建集群
  
  创建集群时，需要指定集群中的 monitor 主机列表，这一步会生成 ceph 配置文件以及 mon 的 keyring:
  ```bash
  ceph-deploy new ceph01 ceph02 ceph03
  
  # 生成如下几个文件
  -rw-r--r-- 1 root root    476 Feb 06 10:22 ceph.conf
  -rw-r--r-- 1 root root 121358 Feb 06 10:22 ceph-deploy-ceph.log
  -rw------- 1 root root     73 Feb 06 10:22 ceph.mon.keyring
  ```


- 修改 `ceph.conf`, 在 `[global]` 下添加如下自定义配置：
  ```bash
  public network = 192.168.16.0/24
  cluster network = 192.168.10.0/24
  
  osd pool default pg num = 64
  osd pool default pgp num = 64
  osd pool default size = 3
  osd pool default min size = 2
  ```


- 在集群节点上安装 ceph 服务

  这里指定的主机名要和ceph节点的 hostname 一致。每个节点上的 ceph 服务默认会创建一个仅包含主机名（非fqdn）的 socket 文件，ceph-deploy 在下一步初始化集群时会使用这个 socket 文件名，如果不一致，将会提示 `file not found` 错误。  
  因此，我们有两个办法来规避这个不一致的问题：  
  a.) 主机名不带点分隔符  
  b.) 安装ceph 时使用 `HOST:FQDN` 名称对（需要添加 hosts 或者自定义 dns）  
  ```bash
  ceph-deploy install \
    --repo-url https://mirrors.cloud.tencent.com/ceph/rpm-luminous/el7 \
    --gpg-url https://mirrors.cloud.tencent.com/ceph/keys/release.asc \
    ceph01 ceph02 ceph03
  ```
  使用 host:fqdn 名称对时，则主机名列表为：
  ```bash
  ceph01:ceph01.prod ceph02:ceph02.prod ceph03:ceph03.prod
  ```


- 创建初始化集群
  ```bash
  ceph-deploy mon create-initial
  ```
  这一步完成，ceph 集群就创建好了，只是还没有 osd 存储设备，无法对外提供服务。


- 分发配置文件到各个节点
  ```bash
  ceph-deploy --overwrite-conf admin ceph01 ceph02 ceph03 
  ```
  

- 创建 manager 服务  
  这个功能在 luminous 引入，也就是要 ceph 12+ 版本需要单独部署该服务。  
  ```bash
  ceph-deploy mgr create ceph01 ceph02
  ```


- 创建 ceph 元数据服务  
  这个功能只有使用 cephfs 时需要，我们规划使用 cephfs 向 CVM 提供共享存储服务，需要部署该服务。cephfs 创建见后文。
  ```bash
  ceph-deploy mds create ceph01 ceph02
  ```


- 添加 osd 存储
  ```bash
  ceph-deploy osd create --bluestore --data /dev/vdb ceph01
  ceph-deploy osd create --bluestore --data /dev/vdb ceph02
  ceph-deploy osd create --bluestore --data /dev/vdb ceph03
  ```
  若这步没问题，则我们已经创建了含有 3个 osd 的ceph 集群。  
  **登录ceph mon 节点**，使用如下命令查看 ceph 集群及 osd 健康状况：
  ```bash
  ceph -s
  ceph osd tree
  ```
  
  结果样例：
  ```bash
  [root@ceph01:~]# ceph -s
  cluster:
      id:     7a620901-ad33-4d2e-9ca9-c8204671e08a
      health: HEALTH_OK
  services:
      mon: 3 daemons, quorum ceph01,ceph02,ceph03
      mgr: ceph01(active), standbys: ceph02
      mds: data-1/1/1 up  {0=ceph01=up:active}, 1 up:standby
      osd: 3 osds: 3 up, 3 in
  data:
      pools:   4 pools, 160 pgs
      objects: 6.20k objects, 21.7GiB
      usage:   68.3GiB used, 1.40TiB / 1.46TiB avail
      pgs:     160 active+clean
  io:
      client:   0B/s rd, 79.3KiB/s wr, 0op/s rd, 1op/s wr
  [root@ceph01:~]# ceph osd tree
  ID CLASS WEIGHT  TYPE NAME       STATUS REWEIGHT PRI-AFF
  -1       1.46489 root default
  -3       0.48830     host ceph01
  0   hdd 0.48830         osd.0       up  1.00000 1.00000
  -5       0.48830     host ceph02
  1   hdd 0.48830         osd.1       up  1.00000 1.00000
  -7       0.48830     host ceph03
  2   hdd 0.48830         osd.2       up  1.00000 1.00000
  [root@ceph01:~]#
  ```

## 使用 ceph 服务

以下操作需要登录任意一台 ceph monitor 节点执行。

### 创建 ceph 存储池  
  ceph 存储池是ceph 对象的逻辑组（logical group）, 使用 ceph 前我们需要创建存储池，并将存储池初始化为 cephfs 或者 ceph rbd。  
  
  按照规划，我们需要使用 cephfs 对 cvm 提供共享存储服务，使用 rbd 对 k8s 提供动态卷服务，因此一次性创建4个 存储池。

  创建存储池时，需要指定 pg 和 pgp 数量。pg 的粗略计算公式为：`(100 * OSD数量) / 副本数`。  

  我们当前有 3个OSD，副本数为 3，则 pg 数量为 (100 * 3) / 3 = 100 。pgp 数量一般等于 pg 数量。  

  我们这里将 pg 设置为 32，这样就可以多创建几个 pool 了，而且初始时设置小一点，后续可以往大了调整。

  ```bash
  # 创建存储池
  ceph osd pool create kube 32 32  # k8s 应用
  ceph osd pool create monitor 32 32  # k8s 中的 prometheus
  ceph osd pool create data_data 32 32  # 用于共享存储的 cephsfs 数据存储池
  ceph osd pool create data_metadata 32 32  # 用于共享存储的 cephsfs 元数据存储池
  
  # 设置存储池配额
  ceph osd pool set-quota data_data max_bytes 10737418240 # 10G
  ceph osd pool set-quota kube max_bytes 429496729600 # 400G
  ceph osd pool set-quota monitor max_bytes 53687091200 # 50G
  
  # 查看配额
  ceph osd pool get-quota kube
  ceph osd pool get-quota monitor
  ceph osd pool get-quota data_data
  ceph osd pool get-quota data_metadata
  
  # 更改 pg/pgp 数量
  ceph osd pool set data_data pg_num 64
  ceph osd pool set data_metadata pgp_num 64
  ```


### 创建 ceph 用户  
  有了 ceph 存储池，虽然可以使用 admin 用户去读写，但我们还是建议为不同用途的存储池创建独立的用户并添加相应的授权。  

  ceph 有多种用户类型，比如 `osd、mds、mgr` 等，而我们要用的类型是 `client`，在创建时指定用户类型即可：
  ```bash
  # 查看已有的用户
  ceph auth ls
  
  # 以创建 data 用户为例
  ceph auth get-or-create client.data mds 'allow rw' mon 'allow r' osd 'allow rwx pool=data_metadata,allow rwx pool=data_data'
  # 用于 cephfs 的用户，需要添加对 mds 服务的读写权限
  ceph auth caps client.data mds 'allow rw' mon 'allow r' osd 'allow rwx pool=data_metadata,allow rwx pool=data_data'
  
  # 将密钥保存为文件，其它 cvm 挂载cephfs 时需要用到 
  # ceph auth get-or-create client.data > /etc/ceph/ceph.client.data.keyring
  ceph auth get-key client.data > /etc/ceph/ceph.client.data.secret
  ```


### 创建 cephfs 文件存储  
  创建 cephfs 时，需要指定元数据存储池（在前）和数据存储池（在后）：
  ```bash
  # 创建 cephfs
  ceph fs new data data_metadata data_data
  ceph fs ls
  ```


### 挂载 cephfs  

挂载 cephfs 需要 ceph 防火墙开放以下端口给客户端节点：
- TCP:6789：ceph-mon 进程端口
- TCP:6800：ceph-mds 进程端口

手动挂载：
```bash
mount -t ceph 192.168.16.10:6789,192.168.16.11:6789,192.168.16.12:6789:/data /data1 -o name=data,secretfile=/etc/ceph/ceph.client.data.secret
```

将挂载信息写入 fstab：
```bash
echo $(mount |grep ceph |awk '{print $1" "$3" "$5" "$6}' | awk -F'[()]' '{print $1 $2}') >> /etc/fstab
mount -a
```


### 创建 ceph rbd 块存储
  RBD (Rados Block Device) 是在 RADOS 集群中存储**块设备**的一种方式。以 RBD 方式存储的块设备通常被称为 RBD images 或者 RBD devices。  

  要存储 RBD 设备我们需要一个专用的存储池。我们这里示例性的初始化一个 RBD 存储池，而实际提供给 k8s 用的 kube 存储池是使用 `rbd-provisioner` 初始化的。
  ```bash
  # 创建存储池
  ceph osd pool create rbd-test 32 32
  # 初始化专用存储池
  rbd pool init rbd-test
  # 在rbd 存储池内创建一个镜像
  rbd create --pool rbd-test --image test-image --size 100M
  # 查看镜像
  rbd ls rbd-test
  rbd info rbd-test/test-image
  rbd image 'test-image':
          size 100MiB in 25 objects
          order 22 (4MiB objects)
          block_name_prefix: rbd_data.233366b8b4567
          format: 2
          features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
          flags:
          create_timestamp: Mon Feb 06 15:06:23 2019
  ```
  其它操作还有增加或压缩镜像尺寸，更多操作参考：https://www.marksei.com/rbd-images-ceph-beginner


## 失败重试与重建
#
安装过程中，难免会因为一些准备工作不完善或者环境配置不正确导致安装失败，如果重试不成功，则建议清理集群重新开始。
```bash
# 清除#集群
c#eph-deploy purge ceph01 ceph02 ceph03
ceph-deploy purgedata ceph01 ceph02 ceph03
ceph-deploy forgetkeys

## 使用 device-mapper driver 移除 ceph 创建的逻辑设备
dmsetup remove $(lsblk /dev/mapper/ceph* |# awk '/ceph/{print $1}')

## 移除逻辑卷组和物理分区
vgremove $(vgs# | awk '/ceph/{print $1}')
pvremove /dev/vdb1

## 清除磁盘
c#eph-deploy disk zap ceph01 /dev/vdb1
ceph-deploy disk zap ceph02 /dev/vdb1
ceph-deploy disk zap ceph03 /dev/vdb1
```



## 扩容集群

扩容集群包括两个方面：
- 在已有节点增加磁盘并创建为 osd
- 新增ceph 节点并新增磁盘

### 新增集群节点

假如新增节点为：ceph04、ceph05、ceph06

- 配置 ceph-deploy 节点能够免密登录新节点


- 使用 ceph-deploy 在新节点安装 ceph 服务
  ```bash
  ceph-deploy install \
    --repo-url https://nexus.fintecer.com/repository/yum/ceph/rpm-luminous/el7 \
    --gpg-url https://mirrors.cloud.tencent.com/ceph/keys/release.asc \
    ceph04 ceph05 ceph06
  ```


- 分发配置文件到新的节点
  ```bash
  ceph-deploy --overwrite-conf admin ceph04 ceph05 ceph06 
  ```


- 将新节点磁盘创建为 ceph osd 磁盘
  ```bash
  ceph-deploy osd create --bluestore --data /dev/vdb ceph04
  ceph-deploy osd create --bluestore --data /dev/vdb ceph05
  ceph-deploy osd create --bluestore --data /dev/vdb ceph06
  ```


- 检查集群健康状况
  ```bash
  ssh ceph01 "ceph -s"
  ssh ceph01 "ceph osd tree"
  ```


### 新增 OSD 

有时候可能也会直接在已有主机上新增硬盘，并将其创建为新的 OSD，因此只需要按照 __新增集群节点__ 后两步操作即可。

假如新增磁盘为：/dev/vdc

- 将新磁盘创建为 ceph osd 磁盘
  ```bash
  ceph-deploy osd create --bluestore --data /dev/vdc ceph04
  ceph-deploy osd create --bluestore --data /dev/vdc ceph05
  ceph-deploy osd create --bluestore --data /dev/vdc ceph06
  ```


- 检查集群健康状况
  ```bash
  ssh ceph01 "ceph -s"
  ssh ceph01 "ceph osd tree"
  ```



## ceph dashboard
#
ceph dashboard 是 ceph luminous 开始新增的模块，用于展示 ceph 集群的各个服务和组件状态:

- dashboard 安装  
  ```bash
  ceph mgr module enable dashboard
  ```

  访问 public-ip:7000  
  [http://192.168.16.20:7000](http://192.168.16.20:7000)



## 监控添加

待完善。



## 错误处理

- ceph-deploy mon create-initial 失败 

```bash 
[ceph01][INFO  ] Running command: /usr/sbin/service ceph -c /etc/ceph/ceph.conf start mon.ceph01
[ceph01][WARNIN] The service command supports only basic LSB actions (start, stop, restart, try-restart, reload, force-reload, status). For other actions, please try to use systemctl.
[ceph01][ERROR ] RuntimeError: command returned non-zero exit status: 2
[ceph_deploy.mon][ERROR ] Failed to execute command: /usr/sbin/service ceph -c /etc/ceph/ceph.conf start mon.ceph01
[ceph_deploy.mon][DEBUG ] detecting platform for host ceph02 ...
```

ceph-deploy 版本太老，升级后解决：
```bash
yum upgrade ceph-deploy
```



-  ceph-deploy disk zap ceph01 /dev/vdb 失败
```bash
[ceph01][DEBUG ] --> --destroy was not specified, but zapping a whole device will remove the partition table
[ceph01][DEBUG ] Running command: wipefs --all /dev/vdb
[ceph01][DEBUG ]  stderr: wipefs: error: /dev/vdb: probing initialization failed: Device or resource busy
[ceph01][ERROR ] RuntimeError: command returned non-zero exit status: 1
[ceph_deploy][ERROR ] RuntimeError: Failed to execute command: /usr/sbin/ceph-volume lvm zap /dev/vdb
```

这是 ceph 创建了残留的磁盘分区，需要手动清理：
```bash
[root@ceph02:~]# lsblk
NAME                                                                                                 MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sr0                                                                                                   11:0    1  3.5M  0 rom
vda                                                                                                  253:0    0   50G  0 disk
└─vda1                                                                                               253:1    0   50G  0 part /
vdb                                                                                                  253:16   0  500G  0 disk
└─ceph--2f905283--4cd9--4be1--9b6b--dced6a402c78-osd--data--f07d75bd--d2f9--4745--a764--efadda39de64 252:0    0  500G  0 lvm
[root@ceph02:~]#  dmsetup remove $(lsblk /dev/mapper/ceph* | awk '/ceph/{print $1}')
```


- 总是出现 failed 的 systemd 服务
```bash
[root@ceph02:~]# systemctl list-units | grep ceph
  var-lib-ceph-osd-ceph\x2d1.mount                                                    loaded active mounted   /var/lib/ceph/osd/ceph-1
  ceph-mds@ceph02.service                                                             loaded active running   Ceph metadata server daemon
  ceph-mgr@ceph02.service                                                             loaded active running   Ceph cluster manager daemon
  ceph-mon@ceph02.service                                                             loaded active running   Ceph cluster monitor daemon
● ceph-mon@jn.service                                                                 loaded failed failed    Ceph cluster monitor daemon
  ceph-osd@1.service                                                                  loaded active running   Ceph object storage daemon osd.1
  system-ceph\x2dmds.slice                                                            loaded active active    system-ceph\x2dmds.slice
  system-ceph\x2dmgr.slice                                                            loaded active active    system-ceph\x2dmgr.slice
  system-ceph\x2dmon.slice                                                            loaded active active    system-ceph\x2dmon.slice
  system-ceph\x2dosd.slice                                                            loaded active active    system-ceph\x2dosd.slice
  ceph-mds.target                                                                     loaded active active    ceph target allowing to start/stop all ceph-mds@.service instances at once
  ceph-mgr.target                                                                     loaded active active    ceph target allowing to start/stop all ceph-mgr@.service instances at once
  ceph-mon.target                                                                     loaded active active    ceph target allowing to start/stop all ceph-mon@.service instances at once
  ceph-osd.target                                                                     loaded active active    ceph target allowing to start/stop all ceph-osd@.service instances at once
  ceph-radosgw.target                                                                 loaded active active    ceph target allowing to start/stop all ceph-radosgw@.service instances at once
```

因为之前主机名命名带有点，不能被 ceph 完整读取，因此改了主机名，这个失败的服务是残留的配置，可以删除“
```bash
systemctl disable ceph-mon@jn.service
systemctl reset-failed ceph-mon@jn.service
systemctl daemon-reload
```

- 创建存储池时指定了超额的 pg 数量

```bash
[root@ceph01:~]# ceph osd pool create schedu 256 256
Error ERANGE:  pg_num 256 size 3 would mean 768 total pgs, which exceeds max 750 (mon_max_pg_per_osd 250 * num_in_osds 3)
```

- cephfs 授权子目录出错
```bash
ceph fs authorize cephfs client.schedu2 /schedu rw
Error EINVAL: unknown cap type '/schedu'
```
ceph bug, 影响12.2.12，计划在下一版本修复：  
https://tracker.ceph.com/issues/39395


- ceph health 状态 WARN
```bash
ceph health detail
HEALTH_WARN application not enabled on 2 pool(s)
POOL_APP_NOT_ENABLED application not enabled on 2 pool(s)
    application not enabled on pool 'kube'
    application not enabled on pool 'monitor'
    use 'ceph osd pool application enable <pool-name> <app-name>', where <app-name> is 'cephfs', 'rbd', 'rgw', or freeform for custom applications.
```

这一问题是我启用了 dashboard 功能，但是没有指定那些存储池使用哪种应用，按照提示指定一下即可：
```bash
[root@ceph01:/etc/ceph]# ceph osd pool ls
data_data
data_metadata
kube
monitor
```

这里我们有4个存储池，其中 data_X 这两个存储池用作 cephfs 依赖的 pool，data_metadata 存储元数据，data_data 存储数据；  
而 kube 用作 k8s 集群中应用的 rbd 共享存储池，monitor 用作 K8s 集群监控服务 prometheus 的存储池。  
因此按照如下设置，告警就没有了：

```bash
[root@ceph01:~]# ceph osd pool application enable kube rbd
enabled application 'rbd' on pool 'kube'
[root@ceph01:~]# ceph osd pool application enable monitor rbd
enabled application 'rbd' on pool 'monitor'
[root@ceph01:~]# ceph osd pool application enable data_data cephfs
enabled application 'cephfs' on pool 'data_data'
[root@ceph01:~]# ceph osd pool application enable data_metadata cephfs
enabled application 'cephfs' on pool 'data_metadata'
```

## 参考文档

- http://docs.ceph.com/docs/luminous/rados/operations/placement-groups
- https://github.com/kubernetes-incubator/external-storage/tree/master/ceph/rbd  
- https://jimmysong.io/kubernetes-handbook/practice/rbd-provisioner.html  
- https://blog.csdn.net/aixiaoyang168/article/details/79056864  
- https://ceph.com/community/new-luminous-dashboard


