### 1 Ceph集群搭建

#### 1.1 配置环境

+ 配置IP和网络，比如

  ```shell
  # 管理机：   10.190.180.240
  # Ceph集群： 10.190.180.241 ~ 10.190.180.243
  vi /etc/sysconfig/network-scripts/ifcfg-eth0
  # 配置以下属性
      BOOTPROTO=static
      ONBOOT=yes
      IPADDR=10.190.180.240
      GATEWAY=10.190.181.254
      NETMASK=255.255.254.0
      DNS1=202.96.134.133
      DNS2=202.96.128.86
  systemctl restart network.service
  ```

+ 关闭防火墙和SELinux

  ```shell
  # 所有机子
  systemctl stop firewalld
  systemctl disable firewalld
  systemctl status firewalld
  setenforce 0
  vi /etc/selinux/config  # 修改 selinux=disabled
  ```

+ 设置时钟源

  ```shell
  # 所有机子
  yum install ntp -y
  # 集群机子
  vi /etc/ntp.conf  # 修改 server 10.190.180.240 iburst
  # 所有机子
  systemctl restart ntpd
  systemctl enable ntpd
  ntpq -pn
  ```

+ 配置YUM源

  ```shell
  # 所有机子
  cd /etc/yum.repos.d/
  vi /etc/yum.repos.d/internal.repo
  # 配置以下属性
  	[internal]
      name=internal
      baseurl=内网仓库地址
      enabled=1
      gpgcheck=0
      priority=1
  yum repolist
  ```

+ 磁盘分区

  ```shell
  # 集群机子
  # 逻辑卷管理：PV VG LV 增删扩缩
  lsblk
  pvcreate /dev/sdb # 将物理磁盘初始化为物理卷 PV
  vgcreate ceph /dev/sdb # 创建卷组 VG，并将 PV 加入 VG
  lvcreate -n osd -L 10G ceph # 基于卷组 VG 创建逻辑卷 LV
  mkfs.xfs /dev/ceph/osd # 格式化
  mount /dev/ceph/osd /mnt # 挂载
  df -h
  ```



#### 1.2 管理机安装ceph-ansible

+ 安装 Ansible

  ``` shell
  yum -y install ansible
  ```

+ 安装 Ceph-Ansible

  ```shel
  git clone -b stable-4.0 https://github.com/ceph/ceph-ansible.git --recursive
  ```

+ 解决环境依赖

  ```shell
  # 管理机
  yum install -y python-pip
  pip install pip==19.3.1
  pip install -r ceph-ansible/requirements.txt  # ansible netaddr
  ```

+ 修改ansible配置文件

  ```shell
  vi /etc/ansible/hosts
  # 配置Ceph集群节点地址以及机子角色
  [all:vars]
  ansible_ssh_port=22
  ansible_ssh_user="root"
  ansible_ssh_pass=""
  [mons]
  192.168.88.179
  192.168.88.180
  192.168.88.181
  [mgrs]
  192.168.88.179
  192.168.88.180
  192.168.88.181
  [osds]
  192.168.88.179
  192.168.88.180
  192.168.88.181
  
  vi /etc/ansible/ansible.cfg
  host_key_checking = False  # 跳过ssh第一次连接时输入yes
  ```

+ 修改ceph-ansible配置文件

  ```shell
  cd ceph-ansible/group_vars/
  
  # 根据模板创建文件
  cp mons.yml.sample mons.yml
  cp mgrs.yml.sample mgrs.yml
  cp osds.yml.sample osds.yml
  cp all.yml.sample all.yml
  
  # 配置 all.yml
      ---  
      # 配置 Ceph 源
      ceph_origin: repository
      ceph_repository: community
      # ceph_mirror: http://mirrors.aliyun.com/ceph
      # ceph_stable_key: http://mirrors.aliyun.com/ceph/keys/release.asc
      ceph_mirror: # 配置内部yum源
      ceph_stable_release: nautilus
      ceph_stable_repo: "{{ ceph_mirror }}/rpm-{{ ceph_stable_release }}"
      # 集群网络配置
      public_network: 10.190.180.0/22
      cluster_network: 10.190.180.0/23    
      # mon
      monitor_interface: eth0
      # osd
      osd_objectstore: bluestore
      # mgr
      dashboard_enabled: false
      # overrides
      ceph_conf_overrides:
        global:
          osd_pool_default_pg_num: 64
          osd_pool_default_pgp_num: 64
          osd_pool_default_size: 1
          #common_single_host_mode: true
        mon:
          mon_allow_pool_create: true
  	ntp_daemon_type: ntpd
  # 配置 osds.yml
  # bluestore：data + metadata + wal
      ---
      lvm_volumes:
        - data: osd-data
          data_vg: ceph
          wal: osd-wal
          wal_vg: ceph
          db: osd-meta
          db_vg: ceph
  
  # 配置 site.yml
  cd ceph-ansible
cp site.yml.sample site.yml
  
  # 注释掉不用的 hosts
  # 注释掉 - import_playbook: dashboard.yml
  ```
  



#### 1.3 创建集群

```shell
ansible-playbook site.yml
```



### 2 Ceph命令操作

> architecture：https://docs.ceph.com/en/latest/architecture/
> cluster.operations：https://docs.ceph.com/en/latest/rados/operations/

```shell
ceph -s 
ceph health detail
```



#### 2.1 用户权限

> https://docs.ceph.com/en/latest/rados/operations/user-management/#
> https://is-cloud.blog.csdn.net/article/details/89679912 
> ceph client 包括 rbd / radosgw / cephfs / cli-rest-api 等

+ ceph auth

  ```shell
  ceph auth ls  # 所有用户信息
  ceph auth add {USERTYPE.USERID} [{daemon} 'allow [r|w|x|*|...] [pool={pool-name}]']  # 新增用户
  ceph auth caps {USERTYPE.USERID} [{daemon} 'allow [r|w|x|*|...] [pool={pool-name}]']  # 用户授权
  ceph auth del {USERTYPE.USERID} # 删除用户
  
  # 用户名
  osd.1
      	# 用户密码
          key: AQCpmBpgshmRHRAAFln+JqLcJy+IXY6tInrVlw==
          # 允许用户在 mon 进行 osd 访问
          caps: [mon] allow profile osd
          # 允许用户在 osd 进行 rwx
          caps: [osd] allow *
          
  # profile osd (Monitor only)
  # Gives a user permissions to connect as an OSD to other OSDs or monitors. Conferred on OSDs to enable OSDs to handle replication heartbeat traffic and status reporting.
  
  # profile bootstrap-osd (Monitor only)
  # Gives a user permissions to bootstrap an OSD. Conferred on deployment tools such as ceph-volume, cephadm, etc. so that they have permissions to add keys, etc. when bootstrapping an OSD.
  ```

+ ceph-authtool

  > https://docs.ceph.com/en/latest/man/8/ceph-authtool/

  ```shell
  ceph-authtool -n client.l --gen-key --cap osd 'allow rwx' --cap mon 'allow rwx' --create-keyring /etc/ceph/client.l.keyring  # 创建用户并授权并生成密钥环
  ceph auth import -i /etc/ceph/client.l.keyring # 从密钥环文件导入用户到集群
  ceph auth get client.admin -o /etc/ceph/ceph.client.admin.keyring # 将用户导出到密钥环
  ceph -n client.admin --keyring=/etc/ceph/ceph.client.admin.keyring health # 指定密钥环用户来操作
  ```



#### 2.2 MON

+ 新增 mon

  ```shell
  # 1. 拷贝原集群 ceph.conf，在其基础上添加 mon 节点 node-2 信息
  [global]
  fsid = ac0231f0-2a1f-4b83-9aee-081fe1068851
  mon initial members = node-1,node-2
  mon host = 10.190.180.241,10.190.180.242
  # 2. 根据配置好的 ceph.conf，手动创建 monmap
  monmaptool --create --clobber --fsid ac0231f0-2a1f-4b83-9aee-081fe1068851 --add node-1 10.190.180.241 --add node-2 10.190.180.242 /tmp/monmap
  # 如果是修复 mon，可直接在原可用集群 ceph mon getmap -o /tmp/monmap
  # 3. 从原可用集群上获取 client.admin 角色密钥环
  ceph auth get client.admin -o /etc/ceph/ceph.client.admin.keyring
  # 4. 初始化数据文件
  ceph-mon -i node-1 --mkfs --monmap /tmp/monmap --keyring /etc/ceph/ceph.mon.keyring -c /etc/ceph/ceph.conf
  # 5. 配置标志位文件
  touch /var/lib/ceph/mon/ceph-node-2/done
  # 6. 启动 mon 服务
  systemctl start ceph-mon@node-2
  systemctl enable ceph-mon.target
  ```
  
+ monmaptool 工具 

  ```shell
  # 生成 monmap
  monmaptool --create --fsid 6da67f55-6178-486b-98d1-0286def1914f --add node-1 10.193.52.132 /tmp/monmap
  # 获取 monmap
  monmaptool --create --generate -c /etc/ceph/ceph.conf /tmp/monmap
  # 查看 monmap
  monmaptool --print monmap
  # 删除指定字段
  monmaptool --rm node-1 /tmp/monmap
  # 新增字段
  monmaptool --add node-2 10.112.101.143 /tmp/monmap
  ```



#### 2.3 OSD

+ 新增 osd

  ```shell
  # bluestore: db + meta + wal
  # create 版
  ceph-volume lvm create --bluestore --data {dev or vg/lv} --block.wal {} --block.db {} 
  # if block device, a logical volume will be created
  
  # 等同于 prepare + activate
  ceph-volume lvm prepare --bluestore --data {dev or vg/lv} --block.wal {} --block.db {} 
  ceph-volume lvm list
  ceph-volume lvm activate {ID} {FSID}
  ```

+ ceph-volume

  > https://docs.ceph.com/en/nautilus/man/8/ceph-volume/

  ```
  ceph-volume lvm [ trigger | create | activate | prepare | zap | list | batch ]
  ceph-volume simple [ trigger | scan | activate ]
  ceph-volume [-h] [–cluster CLUSTER] [–log-level LOG_LEVEL] [–log-path LOG_PATH]
  ceph-volume inventory
  ```



#### 2.4 MGR

+ 新增 mgr

  ```shell
  # 1. 创建集群用户，并生成密钥环放在指定目录
  ceph auth get-or-create mgr.node-1 mon 'allow profile mgr' osd 'allow *' mds 'allow *' -o /var/lib/ceph/mgr/ceph-node-1/keyring
  # 2. 启动 mgr 服务
  systemctl start ceph-mgr@node-1
  ```

+ 开启 dashboard 模块

  ``` shell
  # 1. 安装包
  yum install ceph-mgr-dashboard
  # 2. 启动模块
  ceph mgr module enable dashboard
  # 3. 配置
  ceph config set mgr mgr/dashboard/ssl true  # 默认 true
  ceph dashboard create-self-signed-cert
  ceph config set mgr mgr/dashboard/server_addr 0.0.0.0
  ceph config set mgr mgr/dashboard/server_port 8080  # 默认 
  ceph config set mgr mgr/dashboard/ssl_server_port 8843 # 默认
  ceph dashboard ac-user-create root 123456 administrator
  
  ceph mgr module ls # "dashboard": "https://node-1:8443/"
  ```

+ 开启 prometheus 模块

  ``` shell
  ceph mgr module enable prometheus
  ceph config set mgr mgr/prometheus/server_addr 0.0.0.0  # 默认
  ceph config set mgr mgr/prometheus/server_port 9283  # 默认
  
  ceph mgr module ls # "prometheus": "http://node-1:9283/"
  ```



#### 2.5 RBD块存储

> 命令文档：https://docs.ceph.com/en/latest/man/8/rbd/

+ 创建Pool资源池

  > https://docs.ceph.com/en/latest/rados/operations/pools/#

  ```shell
  ceph osd pool create poolName 64 64
  ceph osd lspools
  ceph osd pool get poolName pg_num    
  ceph osd pool set poolName pg_num 128
  ```

+ 创建RBD块

  > https://docs.ceph.com/en/latest/rbd/rados-rbd-cmds/#basic-block-device-commands

  ```shell
  rbd create --size {megabytes} [–image-feature feature-name] {pool-name}/{image-name}
  rbd ls {poolname}
  rbd info {pool-name}/{image-name}
  ```

+ 映射RBD块到磁盘

  ```shell
  rbd device map {pool-name}/{image-name}
  ```

+ 磁盘逻辑卷操作

  > https://blog.csdn.net/u011350541/article/details/88722263

  ```shell
  pvcreate /dev/sdb # 将物理磁盘初始化为物理卷 PV
  vgcreate ceph /dev/sdb # 创建卷组 VG，并将 PV 加入 VG
  lvcreate -n osd -L 10G ceph # 基于卷组 VG 创建逻辑卷 LV
  mkfs.xfs /dev/ceph/osd # 格式化
  mount /dev/ceph/osd /mnt # 挂载
  ```

+ RBD回收站机制

  ```shell
  rbd trash mv {pool-name}/{image-name}      # 移至回收站 
  rbd trash restore {pool-name}/{image-id}   # 从回收站恢复
  rbd trash rm {pool-name}/{image-id}        # 删除回收站
  ```

+ RBD快照机制

  > https://docs.ceph.com/en/latest/rbd/rbd-snapshot/

  + 基础操作

    ``` shell 
    rbd snap create {pool-name}/{image-name}@{snap-name}   # 创建
    rbd snap ls {pool-name}/{image-name}
    rbd snap rollback {pool-name}/{image-name}@{snap-name}   # 回滚恢复
    rbd snap rm {pool-name}/{image-name}@{snap-name}  
    ```

  + 克隆 Layering / Copy-On-Write

    ```shell
    rbd snap create {pool-name}/{image-name}@{snap-name}      # 创建快照
    rbd snap protect {pool-name}/{image-name}@{snapshot-name}  # 设置保护位
    rbd clone {pool-name}/{parent-image}@{snap-name} {pool-name}/{child-image-name}  # 快速克隆
    rbd flatten {pool-name}/{image-name}  # 扁平化，解除父子关系
    ```

  + 持久化备份

    + 全量导出

      ```shell
      rbd snap export {pool-name}/{image-name}@{snap-name} {path}
      rbd snap import {snap-file} {pool-name}/{image-name}
      ```

    + 增量导出

      ```shell
      rbd snap export-diff {pool-name}/{image-name}@{snap-name} {path}
      rbd snap import-diff {snap-file} {pool-name}/{image-name}
      ```



### 3 Ceph基础知识



#### 3.1 存储方案

+ **DAS(Direct-attached Storage)：**直连存储裸设备，物理接口插拔，本地操作
+ **NAS(Network Attached Storage)：** **网络**附加存储，**共享**文件系统，如NFS服务器
+ **SAN(Storage Area Network)：**存储区域**网络**，**共享**裸设备。
+ **OS(Object Storage)：** 整合任意可达磁盘，存储**静态不可编辑文件**

| ======== | DAS                                  | NAS                                     | SAN                                            | OS                                          |
| -------- | ------------------------------------ | --------------------------------------- | ---------------------------------------------- | ------------------------------------------- |
| 方式     | 服务器是有SCSI或FC协议连接到存储阵列 | 服务器使用TCP网络协议连接至文件共享存储 | 服务器使用一个存储区域网络IP或FC连接到存储阵列 | 通过网络API访问一个无限扩展的分布式存储系统 |
| 协议类型 | SCSI总线/FC光纤                      | NFS/CIFS                                | IP-SAN/FC-SAN                                  | S3/REST                                     |
| 表现形式 | 一块有空间大小的裸磁盘/dev/sdb       | 映射到存储中的一个目录                  | 一块有空间大小的裸磁盘/dev/sdb                 | 无限使用的存储空间，通过PUT/GET上传和下载   |



#### 3.2 Ceph架构

> Ceph可以将多台服务器组成集群，把这些机器中的磁盘资源整合到一块，形成资源池（支持PB级别 K M G T P），按需分配给客户端使用，即 软件定义存储

+ **应用层**支持三种存储：块存储、文件存储、对象存储
+ 提供**CRUSH分布式**存储，去中心，可扩展，无需单点维护元数据
+ 数据**强一致性**，写入OSD时先同步完副本再返回响应

<img src=".\pictures\image-20210204142047315.png" alt="image-20210204142047315" style="zoom:80%;" />

+ **RADOS(Reliable Autonomic Object Store)：**底层完整的对象存储系统（Ceph中所有数据都以对象形式存储），RADOS主要包括OSD和Monitor
+ **LIBRADOS：**客户端直接访问RADOS所需的基础库
+ **上层应用接口：**提供抽象层次更高的应用接口
  + **RGW：**提供对象存储接口（区分RADOS对象）
  + **RBD：**提供块存储设备接口
  + **CephFS：**提供文件系统接口



#### 3.3 Ceph组件

+ **osd(object storage daemon)：**ceph通过osd管理物理硬盘，每一块盘对应一个osd进程，**负责处理集群数据存储、复制、恢复、均衡等**
+ **mon(monitor)：**维护cluster map（五张表）信息，包括其它组件元信息、crush算法信息，**保证集群数据一致性**  **Paxos 算法**
+ **mgr(manager)：**监控ceph集群，**采集存储利用率、系统负载等运行指标**，并对外提供ui dashboard界面
+ **mds(metadata server)：**专为**cephfs文件存储**提供的**元数据缓存服务器**，提高查询性能



#### 3.4 RADOS存储

>  一个文件首先按照配置的大小切分成多个对象，对象经过哈希算法映射到不同的归置组PG中，每个PG再通过C-RUSH算法将对象存储到经状态过滤的多个OSD中（第一个OSD是主节点，其它的是副本从节点）。

<img src=".\pictures\image-20210108092906061.png" alt="image-20210108092906061" style="zoom: 80%;" />



+ **File：**用户需要存储或访问的文件
+ **Objects：**RADOS基本存储单元，即对象，由File进行切分（ID / Binary Data / Metadata）
+ **PG(Placement Group)：**一个PG包含Objects，一个PG映射到多个OSD上（一主多从）。是实现上的逻辑概念，物理上不存在。PG数量固定，不会随着OSD动态伸缩。
+ **PGP：**相当于PG存放的OSD副本排列组合。假设PG映射到3个osd，即osd1，osd2，osd3，副本数为2，如果pgp=1，那么pg存放的副本osd的组合就有一种，比如（osd1, osd2）
+ **Pool**：对多个PG设置相同的Namespace，定义成逻辑概念上的Pool，可以对不同的业务场景做隔离（pool size 即 PG 映射到 size 个 osd）
+ **OSDs：**每一块物理硬盘对应一个osd进程。每个OSD上分布多个PG，每个PG会自动散落在不同OSD上。如果扩容OSD，那么相应的PG会进行迁移到新的OSD上，保证每个OSD上的PG数量均衡
+ **Bucket：**C-RUSH算法树的中间节点（区别于rgw对象存储中的bucket）

> 上面的映射可归纳为：file → (Pool, Object) → (Pool, PG) → OSD set → OSD/Disk



#### 3.5 CRUSH算法

> Object在通过PG存储到实际的OSD设备上时，会通过C-RUSH算法，按照预定好的规则选择N个OSD进行存储，即CRUSH(pgid) --> (osd1,osd2 ...)。不同对象存储到OSD设备位置无必然联系，对相同对象进行重复计算，其存储位置必然相同。	

<img src=".\pictures\image-20210204143506575.png" alt="image-20210204143506575" style="zoom: 50%;" />

> ceph通过cluster_map存储分布式集群的结构拓扑，集群物理结构定义为树形结构，默认10种层级，每个**中间节点**称为**bucket**，**叶子节点**一定是**osd**

> 若以host为单位，如host0，则落到dev0 dev1
>
> 若以rack为单位，如rack0，则落到dev0 ... 8个



#### 3.6 PG 状态

+ 