### 1 Ceph集群搭建

#### 1.1 ceph-ansible 部署

+ **配置主机环境**

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
      
      # firewalld 结构和使用
      # http://www.excelib.com/article/287/show/#I8ClPj
      # https://blog.csdn.net/yl3395017/article/details/105215027
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
      
      # 同步外部
      # ntpdate cn.pool.ntp.org
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
      
      pvcreate pvremove
      vgcreate vgremove vgextend vgreduce
      lvcreate lvremove
      ```

+ 配置ceph-ansible

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
      ```
      
    + ansible ssh 免密登录

      ```
      http://www.linuxboy.net/ansiblejc/144350.html
      ```

+ **运行playbook**

```shell
ansible-playbook site.yml
```



#### 1.2 手动部署

+ **安装**

  ``` shell
  yum install ceph-mon ceph-mgr ceph-mgr-dashboard ceph-osd -y
  ```

+ **mon** at3

  1. 创建配置文件 /etc/ceph/ceph.conf

     ```yaml
     cat >> /etc/ceph/ceph.conf << EOF
     [global]
     fsid = 4c596c2c-69e3-47c0-a8b0-82ba499ea9ca
     mon_initial_members = node-1, node-2, node-3
     mon_host = 192.168.211.57, 192.168.211.58, 192.168.211.59
     public_network = 192.168.211.0/24
     cluster_network = 192.168.211.0/24
     auth_cluster_required = cephx
     auth_service_required = cephx
     auth_client_required = cephx
     osd_pool_default_size = 3
     osd_pool_default_min_size = 1
     osd_pool_default_pg_num = 64
     osd_pool_default_pgp_num = 64
     osd_crush_chooseleaf_type = 0
     EOF
     ```

  2. 配置集群 monmap 信息

     ```shell
     monmaptool --create --add node-1 192.168.211.57  --add node-2 192.168.211.58 --add node-3 192.168.211.59 --fsid 4c596c2c-69e3-47c0-a8b0-82ba499ea9ca /var/lib/ceph/tmp/monmap
     ```

  3. 配置集群的 client.admin 角色密钥环

     ``` shell
     sudo ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'
     ```

  4. 配置主机的 monitor 角色密钥环

     ```shell
     sudo ceph-authtool --create-keyring /var/lib/ceph/tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *' 
     sudo ceph-authtool /var/lib/ceph/tmp/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
     ```

  5. 配置文件夹权限

     ``` shell
     chown ceph:ceph -R /var/lib/ceph/tmp/
     ```

  6. 初始化主机的monitor数据（生成 /var/lib/ceph/mon/ceph-node-1 目录  {cluster}-{hostname}）

     ```
     ceph-mon --cluster ceph --mkfs -i node-1 --monmap /var/lib/ceph/tmp/monmap --keyring /var/lib/ceph/tmp/ceph.mon.keyring
     ceph-mon --cluster ceph --mkfs -i node-2 --monmap /var/lib/ceph/tmp/monmap --keyring /var/lib/ceph/tmp/ceph.mon.keyring
     ceph-mon --cluster ceph --mkfs -i node-3 --monmap /var/lib/ceph/tmp/monmap --keyring /var/lib/ceph/tmp/ceph.mon.keyring
     ```

  7. 配置文件夹权限

     ```shell
     chown ceph:ceph -R /var/lib/ceph/mon
     ```

  8. 启动 monitor 服务

     ```shell
     systemctl start ceph-mon@node-1
     systemctl enable ceph-mon@node-1
     ```
     
  9. 停止 monitor 服务并踢出集群

     ``` shell
     ceph mon remove node-1
     systemctl stop ceph-mon@node-1
     systemctl disable ceph-mon@node-1
     ```

  10. 去除警告

      ```
      @ 启动v2版本
      ceph mon enable-msgr2
      @ 模式
      ceph config set mon auth_allow_insecure_global_id_reclaim false
      ```

  11. （第二种方式）添加节点

      ``` 
      # 拷贝集群的 
      /etc/ceph/ceph.conf  
      /etc/ceph/ceph.client.admin.keyring 
      /var/lib/ceph/tmp/monmap
      /var/lib/ceph/tmp/ceph.mon.keyring
      到部署的主机上
      
      # 配置文件权限
      chown ceph:ceph -R /var/lib/ceph/tmp
      
      # 初始化monitor目录
      ceph-mon --cluster ceph --mkfs -i node-2 --monmap /var/lib/ceph/tmp/monmap --keyring /var/lib/ceph/tmp/ceph.mon.keyring
      
      # 配置文件权限
      chown ceph:ceph -R /var/lib/ceph/mon
      
      # 启动monitor服务
      systemctl start ceph-mon@node-2
      systemctl enable ceph-mon@node-2
      ```

+ **mgr**

  + 拷贝集群的 /etc/ceph/ceph.conf  /etc/ceph/ceph.client.admin.keyring 到部署的主机上

  + 创建主机的 mgr 目录

     ``` shell
     mkdir /var/lib/ceph/mgr/ceph-node-1    # {cluster}-{hostname}
     ```

  + 创建集群的主机 mgr 用户，并生成密钥环放在指定目录

     ``` shell
     ceph auth get-or-create mgr.node-1 mon 'allow profile mgr' osd 'allow *' mds 'allow *' -o /var/lib/ceph/mgr/ceph-node-1/keyring
     ```

  + 配置文件夹权限

     ``` shell
     chown ceph:ceph -R /var/lib/ceph/mgr
     ```

  + 启动 mgr 服务

     ```shell
     systemctl start ceph-mgr@node-1
     systemctl enable ceph-mgr@node-1
     ```

  + 启动 mgr.dashboard 模块

     ``` shell
     ceph mgr module enable dashboard
     ceph config set mgr mgr/dashboard/ssl false
     ceph dashboard create-self-signed-cert
     ceph config set mgr mgr/dashboard/server_addr 0.0.0.0  
     ceph config set mgr mgr/dashboard/server_port 8080 
     ceph config set mgr mgr/dashboard/ssl_server_port 8443 
     ceph dashboard ac-user-create <username> -i <file-containing-password> administrator
     ceph dashboard ac-user-create <username> <password> administrator
     ```

  + 启动 mgr.prometheus 模块

     ``` shell
     ceph mgr module enable prometheus
     ceph config set mgr mgr/prometheus/server_addr 0.0.0.0
     ceph config set mgr mgr/prometheus/server_port 9283
     ```

  + 查看模块状态

     ``` shell
     [root@node-1 ~]# ceph mgr services
     {
         "dashboard": "https://node-1:8443/",
         "prometheus": "http://node-1:9283/"
     }
     ```

+ **osd**

    1. 拷贝集群的 /etc/ceph/ceph.conf  /etc/ceph/ceph.client.admin.keyring 到部署的主机上

    2. 获取集群的 client.bootstrap-osd 密钥环，指定名称放到指定目录

       ```shell
       ceph auth get client.bootstrap-osd -o /var/lib/ceph/bootstrap-osd/ceph.keyring
       ```

    3. 挂载磁盘并启动 osd 服务

       ```
       ceph-volume lvm create --bluestore --data centos/data --block.wal centos/wal --block.db centos/meta
       ```
       
    4. 删除 osd 服务

       ``` shell
       systemctl stop ceph-osd@0
       systemctl disable ceph-osd@0
       ceph-volume lvm zap --destroy --osd-id 0 # ceph-volume lvm zap {vg/lv}
       ceph osd out osd.0
       ceph osd down osd.0
       ceph osd purge osd.0 --yes-i-really-mean-it
       ```



### 2 Ceph命令操作

> architecture：https://docs.ceph.com/en/latest/architecture/
> 						  https://is-cloud.blog.csdn.net/article/details/89419873
> cluster.operations：https://docs.ceph.com/en/latest/rados/operations/
>                                      https://docs.ceph.com/en/latest/rados/man/
>                                      https://blog.51cto.com/hongchen99/2507959

```shell
ceph [-s|-w] # 静态|动态
ceph health detail # 详细信息，解决方法
ceph df # 存储使用情况
ceph {mon/osd/pg/...} dump # 详情

ceph --show-config
# 修改 ceph  配置 （1.2 临时生效，3 永久生效）
# 1. ceph config
ceph config set mon auth_allow_insecure_global_id_reclaim false
ceph config get mon.node-1 auth_allow_insecure_global_id_reclaim # get 需要指明具体的节点，set 则不用
# 2. 套接字 / 进程
ceph --admin-daemon /var/run/ceph/ceph-mon.mode-1.asok config [show|set...]
ceph daemon osd.0 config [get|set...]
# 3. ceph.conf 
vi /etc/ceph/ceph.conf
```

``` yaml
[global]
fsid = 020a529f-3efc-4b04-9edb-ecb4858ddcea
mon_host = [v2:192.168.89.54:3300,v1:192.168.89.54:6789],[v2:192.168.89.55:3300,v1:192.168.89.55:6789],[v2:192.168.89.56:3300,v1:192.168.89.56:6789]
mon_initial_members = E01-P01-DN-001.gd.cn,E01-P01-DN-002.gd.cn,E01-P01-DN-003.gd.cn
public_network = 192.168.88.0/23

max_open_files = 1310720
mon_max_pg_per_osd = 512
osd_crush_update_on_start = false
ms_bind_port_min = 10000
ms_bind_port_max = 15000
mon_osd_down_out_subtree_limit = host

#----------------------- RGW -----------------------
rgw_frontends = "beast port=7480 num_threads=1024 enable_keep_alive=yes keep_alive_timeout_ms=120000"
rgw_bucket_index_max_aio = 64
rgw_dynamic_resharding = false
rgw_num_rados_handles = 16
rgw_thread_pool_size = 1024
rgw_cache_enabled = true
rgw_max_chunk_size = 4194304
rgw_cache_lru_size = 100000
rgw_enable_gc_threads = true
rgw_gc_max_objs = 1024
rgw_gc_obj_min_wait = 0
rgw_gc_processor_max_time = 3600
rgw_gc_processor_period = 30
rgw_gc_max_concurrent_io = 300
rgw_gc_max_trim_chunk = 256
rgw_enable_ops_log = false
rgw_enable_usage_log = true
rgw_usage_log_tick_interval = 1
rgw_usage_log_flush_threshold = 1
rgw_usage_max_shards = 32
rgw_usage_max_user_shards = 1
rgw_zone =
#---------------------------------------------------
mon_allow_pool_delete = false
[osd]
osd_memory_target = 1073741824
bluestore_min_alloc_size_hdd = 16384
bluestore_max_blob_size_hdd = 524288
bluefs_allocator = bitmap
bluestore_allocator = bitmap
bluestore_prefer_deferred_size_hdd = 2048
bluestore_cache_size_hdd = 1073741824
bluestore_deferred_batch_ops_hdd = 0
bluestore_cache_meta_ratio = 0.8
bluestore_cache_kv_ratio = 0.2
bluestore_rocksdb_options = compression=kNoCompression,max_write_buffer_number=6,min_write_buffer_number_to_merge=1,write_buffer_size=67108864,target_file_size_base=134217728,max_bytes_for_level_base=73400320,max_background_flushes=32,max_background_compactions=32,use_direct_reads=true,use_direct_io_for_flush_and_compaction=true

osd_max_write_size = 512
osd_op_num_shards = 8
osd_op_num_threads_per_shard = 8
osd_recovery_max_active = 1
osd_recovery_sleep = 0.0
osd_recovery_sleep_hdd = 0.1
osd_recovery_sleep_ssd = 0.0
osd_recovery_max_chunk = 33554432
osd_recovery_op_priority = 32
osd_recovery_max_single_start = 1
osd_recovery_thread_timeout = 120
osd_recovery_max_omap_entries_per_chunk = 131072
osd_max_backfills = 1
osd_backfill_scan_min = 256
osd_backfill_scan_max = 1024
osd_backfill_retry_interval = 10.0

osd_max_scrubs = 1
osd_scrub_auto_repair = true
osd_scrub_chunk_max = 1
osd_scrub_chunk_min = 1
osd_scrub_max_interval = 2592000
osd_scrub_min_interval = 0
osd_scrub_sleep = 1
osd_scrub_begin_hour = 0
osd_scrub_end_hour = 24
osd_scrub_interval_randomize_ratio = 1.0
osd_deep_scrub_interval = 604800
osd_deep_scrub_randomize_ratio = 1.0
osd_scrub_load_threshold = 40.0

osd_client_watch_timeout = 15
osd_heartbeat_grace = 20
osd_heartbeat_interval = 5
```





#### 2.1 用户权限

> https://docs.ceph.com/en/latest/rados/operations/user-management/#
> https://is-cloud.blog.csdn.net/article/details/89679912 
> ceph client 包括 rbd / radosgw / cephfs / cli-rest-api 等

+ 身份认证原理：**共享密钥**

  <img src="pictures/image-20210304195854979.png" alt="image-20210304195854979" style="zoom:90%;" />

  > 1. 用户通过客户端向 MON 发起请求。
  > 2. 客户端将**用户名**传递到 MON。
  > 3. MON 对用户名进行检查，若用户存在，则通过加密用户密钥生成一个 session key 并返回客户端。
  > 4. 客户端通过共享密钥解密 session key，只有**拥有相同用户密钥环文件的客户端**可以完成解密。
  > 5. 客户端得到 session key 后，客户端持有 session key 再次向 MON 发起请求
  > 6. MON 生成一个 ticket，同样使用用户密钥进行加密，然后发送给客户端。
  > 7. 客户端同样通过共享密钥解密得到 ticket。
  > 8. 往后，客户端持有 ticket 向 MON、OSD 发起请求。

  **故：**只要客户端拥有任意用户的密钥环文件，客户端就可以执行特定用户所具有权限的所有操作

+ ceph -s

  ```shell
  # 实际上执行的是
  ceph -s --conf /etc/ceph/ceph.conf --name client.admin --keyring /etc/ceph/ceph.client.admin.keyring
  # 默认使用client.admin用户，并使用默认路径下密钥环
  ```
  
+ ceph auth  用户管理

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

+ ceph-authtool 密钥环文件管理

  > https://docs.ceph.com/en/latest/man/8/ceph-authtool/

  ```shell
  ceph-authtool -n client.l --gen-key --cap osd 'allow rwx' --cap mon 'allow rwx' --create-keyring /etc/ceph/client.l.keyring  # 生成用户密钥环文件
  ceph auth import -i /etc/ceph/client.l.keyring # 从密钥环文件导入用户到集群
  ceph auth get client.admin -o /etc/ceph/ceph.client.admin.keyring # 将用户导出到密钥环
  ceph -n client.admin --keyring=/etc/ceph/ceph.client.admin.keyring health # 指定密钥环用户来操作
  ```



#### 2.2 mon

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



#### 2.3 osd

+ ceph-volume

  > https://docs.ceph.com/en/nautilus/man/8/ceph-volume/

  ```shell
  ceph-volume lvm [ trigger | create | activate | prepare | zap | list | batch ]
  ceph-volume simple [ trigger | scan | activate ]
  ceph-volume [-h] [–cluster CLUSTER] [–log-level LOG_LEVEL] [–log-path LOG_PATH]
  ceph-volume inventory
  ```
  
+ ceph osd

  > https://blog.csdn.net/mpu_nice/article/details/106354561

  ``` shell
  ceph osd [ blocklist | blocked-by | create | new | deep-scrub | df | down | dump | erasure-code-profile | find | getcrushmap | getmap | getmaxosd | in | ls | lspools | map | metadata | ok-to-stop | out | pause | perf | pg-temp | force-create-pg | primary-affinity | primary-temp | repair | reweight | reweight-by-pg | rm | destroy | purge | safe-to-destroy | scrub | set | setcrushmap | setmaxosd | stat | tree | unpause | unset ]
  ```
+ 操作示例

  ```shell
  # 新增 osd 
  # bluestore: db + meta + wal
  # create 版
  ceph-volume lvm create --bluestore --data {dev or vg/lv} --block.wal {} --block.db {}
  # if block device, a logical volume will be created
  
  # 等同于 prepare + activate
  ceph-volume lvm prepare --bluestore --data {dev or vg/lv} --block.wal {} --block.db {} 
  ceph-volume lvm list
  ceph-volume lvm activate {ID} {FSID}
  
  # 扩容（新增）osd 后会触发 rebalance 重分布（部分PGs重新定位到新的osd）
  # 设置标志位临时关闭，适用于当前业务流量较大的时候
  ceph osd set norebalance 
  ceph osd set nobackfill
  ceph osd unset norebalance
  ceph osd unset nobackfill
  
  # 清空磁盘数据  dd if=/dev/zero of=/dev/sdc bs=1M count=10 conv=fsync 
  ceph-volume lvm zap /dev/sdc
  
  # osd 副本间数据一致性检查
  ceph osd scrub {who}
  ceph pg scrub {pg.id}  # ceph pg dump -> pg.id
  ```

  

#### 2.4 mgr




#### 2.5 rbd 块存储

> 命令文档：https://docs.ceph.com/en/latest/man/8/rbd/

+ 创建Pool资源池

  > https://docs.ceph.com/en/latest/rados/operations/pools/#

  ```shell
  ceph osd pool create poolName 64 64   -- replicated[default] 副本 | erasure 纠删码
  
  PGs = ((total_number_of_OSD * 100) / max_replication_count) / pool_count
  结算的结果往上取靠近2的N次方的值。比如总共OSD数量是160，复制份数3，pool数量也是3，那么每个pool分配的PG数量就是2048
  
  ceph.conf: osd_pool_default_size = 3
  ceph osd pool set poolName size 1   
  
  ceph osd pool application enable poolName [rbd | rgw | cephfs] -- 启用应用类型
  ceph osd pool application get poolName
  ceph osd lspools
  ceph osd pool get poolName pg_num    
  ceph osd pool set poolName pg_num 128
  ceph osd pool delete {pool-name} [{pool-name} --yes-i-really-really-mean-it]
  ```

+ 创建RBD块

  > https://docs.ceph.com/en/latest/rbd/rados-rbd-cmds/#basic-block-device-commands

  ```shell
  rbd create --size {megabytes} [--image-feature feature-name (layering)] {pool-name}/{image-name}
  rbd ls {poolname}
  rbd info {pool-name}/{image-name}
  rbd rm {pool-name}/{image-name}
  rbd status {pool-name}/{image-name}  -> watcher
  ceph osd blacklist add watcher
  ceph osd blacklist rm watcher
  
  # 扩容有三个层面：磁盘扩容、分区扩容、文件系统扩容
  rbd resize {pool-name}/{image-name} --size 20G  # 磁盘扩容 fdisk -l
  fdisk /dev/rbd0  # 若直接挂裸盘，则可跳过
  resize2fs /dev/rbd0 # 文件系统扩容 df -h
  ```

+ 映射RBD块到磁盘

  ```shell
  rbd device map {pool-name}/{image-name}
  # 取消映射
  rbd device unmap /dev/rbd0
  rbd device ls
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

+ RBD快照克隆（COW）

  > https://docs.ceph.com/en/latest/rbd/rbd-snapshot/
  >
  > https://zhangchenchen.github.io/2017/06/05/ceph-rbd-snapshot/

  + 增量快照
    + COW(Copy-On-Write)：写时复制，**COW 在创建快照时，并不会发生物理的数据拷贝动作**，仅是拷贝了原始数据所在的源数据块的物理位置元数据。**在创建了快照之后，**一旦源数据块中的原始数据被改写，则会将源数据块上的原始数据拷贝到新数据块中，然后将新数据写入到源数据块中覆盖原始数据。其中所有的源数据块就组成了所谓的源数据卷，而新数据块组成了快照卷。 COW 有一个很明显的缺点，就是会降低源数据卷的写性能，因为每次改写新数据，**实际上都进行了两次写操作。**
    
    + ROW(Redirect-On-Write)：写时重定向，创建快照时，ROW 也会 Copy 一份源数据指针表作为快照数据指针表，此时两张表的指针记录都相同的。在创建快照之后，也就是在快照时间点之后，发生了写操作，那么**新数据会直接被写入到快照卷中**，然后再更新源数据指针表的记录，使其指向新数据所在的快照卷地址。为了保证快照数据的完整性，在创建快照时，源数据卷状态会由读写**变成只读的**。如果做了多次快照，**就产生了一个快照链**，磁盘卷始终挂载在快照链的最末端。
  
    + **区别：**COW 的快照卷存放的是原始数据，而 ROW 的快照卷存放的是新数据。
    
    + **COW的优缺点**
    
      优势：COW 在进行快照操作之前，不会占用任何的存储资源，也不会影响系统性能。
    
      劣势：降低源数据卷的写性能。当修改源数据时，会发生两次写操作：将源数据写入快照卷中，将新数据写入源数据卷中。
    
      如果主机写入数据频繁，那么这种方式将非常消耗I/O。
    
    + 优势：不会降低源数据卷的写性能。源数据卷创建快照后的写操作会被重定向，所有的写 I/O 都被重定向到新卷中，而所有快照卷数据(旧数据)均保留在只读的源数据卷中。因此更新源数据只需要一个写操作，解决了 COW 写两次的性能问题。
    
      劣势：
    
      1. 没有一个完整的快照卷。ROW 的快照卷数据映射表保存的是源数据卷的原始副本，而源数据卷数据指针表保存的则是更新后的副本。因此，当创建了多个快照时，会产生一个快照链，使原始数据的访问快照卷和源数据卷数据的追踪以及快照的删除将变得异常复杂。在恢复快照时会不断地合并快照文件，造成较大的系统开销。
      2. 单机读性能下降。由于采用了重定向写，**使得原本连续的源数据分散到了快照数据卷中**，连续写变成了随机写，**造成读性能下降。**
    
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
  
    https://zhoubofsy.github.io/2019/04/28/storage/ceph/rbd-export-import/
  
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
  
+ 任务队列

  ``` shell
  ceph rbd task add remove <image_spec>
  ceph rbd task add flatten <image_spec>
  ceph rbd task add migration commit <image_spec>
  ceph rbd task add migration abort <image_spec>
  ceph rbd task cancel <task_id>
  ceph rbd task list {<task_id>}
  ceph rbd task add trash remove <image_id_spec>
  ceph rbd task add migration execute <image_spec>
  ```




#### 2.6 rgw 对象存储

> radosgw 对象存储网关：https://docs.ceph.com/en/latest/man/8/radosgw/#   
> 对象存储无状态（无磁盘、目录等），只有对外的上传和下载接口（无更新）

> 接口高可用：keepalived（VIP：虚拟IP）+ haproxy（负载均衡）



#### 2.7 cephfs 文件存储



#### 2.8 rados object

> object 对象操作：https://docs.ceph.com/en/latest/man/8/rados/

``` shell
rbd info {pool-name}/{image-name}  --> block_name_prefix
rados -p {pool-name} ls | grep {block_name_prefix}   # object list
rados -p {pool-name} stat {block_name_prefix}.000000000000423  # object stat 大小
ceph osd map {pool-name} {block_name_prefix}.000000000000423 # object 映射到 pg osd 
```



#### 2.8 crush map

> https://blog.csdn.net/weillee9000/article/details/102842642
>
> https://www.dovefi.com/post/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3crush1%E7%90%86%E8%A7%A3crush_map%E6%96%87%E4%BB%B6/

```
ceph osd crush add {id-or-name} {weight} [{bucket-type}={bucket-name} ...]
ceph osd crush add osd.0 1.000 host=node-1
ceph osd crush remove {id-or-name} [{bucket-type}={bucket-name} ...]
ceph osd crush set {id-or-name} {weight} [{bucket-type}={bucket-name} ...]
ceph osd crush reweight {id-or-name} {weight}

ceph df detail
ceph osd tree
ceph osd dump

ceph osd crush rule dump
ceph osd getcrushmap -o ./crushmap
crushtool -d crushmap -o decrushmap
crushtool -c decrushmap -o newcrushmap
ceph osd setcrushmap -i newcrushmap
```





#### 2.9  log

> cd /var/log/ceph/



#### 2.10 fio 压测

> https://www.jianshu.com/p/ee6ee9ca37e5
>
> https://blog.csdn.net/qq_42776455/article/details/108793391
>
> https://www.cnblogs.com/raykuan/p/6914748.html

+ 顺序写（测吞吐量）

  ``` shell
  fio -filename= -direct=1 -iodepth 1 -thread -rw=write -ioengine=psync -bs=1M -size=200m -numjobs=30 -runtime=60 -group_reporting -name=mytest
  ```

+ 顺序读（测吞吐量）

  ``` shell
  fio -filename= -direct=1 -iodepth 1 -thread -rw=read -ioengine=psync -bs=1M -size=1G -numjobs=30 -runtime=60 -group_reporting -name=mytest
  ```

+ 混合顺序读写（测吞吐量）

  ``` shell
  fio -filename= -direct=1 -iodepth 1 -thread -rw=rw -rwmixread=50 -ioengine=psync -bs=1M -size=1G -numjobs=30 -runtime=60 -group_reporting -name=mytest
  ```

+ 随机写（测IOPS）

  ``` shell
  fio -filename= -direct=1 -iodepth 1 -thread -rw=randwrite -ioengine=psync -bs=4k -size=200m -numjobs=30 -runtime=60 -group_reporting -name=mytest
  ```

+ 随机读（测IOPS）

  ``` shell
  fio -filename= -direct=1 -iodepth 1 -thread -rw=randread -ioengine=psync -bs=4k -size=1G -numjobs=30 -runtime=60 -group_reporting -name=mytest
  ```

+ 混合随机读写（测IOPS）

  ``` shell
  fio -filename= -direct=1 -iodepth 1 -thread -rw=randrw -rwmixread=50 -ioengine=psync -bs=1M -size=1G -numjobs=30 -runtime=60 -group_reporting -name=mytest
  ```

> 1. 测试随机读写时，numjobs从8开始，12..16..20..逐渐往上加，直到IOPS不再上升
> 2. 测试顺序读写时，numjobs从1开始，2..3..4..往上加，基本思路与以上描述一致
> 3. 测试随机读写时要关注IOPS，不要关注IO吞吐；测试顺序读写时要关注IO吞吐，不要关注IOPS

#### 2.11 qos

QoS 限制、存储池扩容、存储规格及存储策略配置

ceph

​	At the image level: `rbd config image set $pool/$image rbd_qos_iops_limit 10`

​	At the pool level: `rbd config pool set $pool rbd_qos_iops_limit 10`

cgroup

http://www.manongjc.com/article/90348.html







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



#### 3.4 rados存储

>  一个文件首先按照配置的大小切分成多个对象，对象经过哈希算法映射到不同的归置组PG中，整个PG再通过Crush算法映射到多个OSD中（第一个OSD是主节点，其它的是副本从节点）。

<img src=".\pictures\image-20210108092906061.png" alt="image-20210108092906061" style="zoom: 80%;" />



+ **File：**用户需要存储或访问的文件
+ **Objects：**RADOS基本存储单元，即对象，由File进行切分（ID / Binary Data / Metadata）
+ **PG(Placement Group)：**一个PG包含Objects，整个PG映射到多个OSD上（一主多从）。是实现上的逻辑概念，物理上不存在。PG数量固定，不会随着OSD动态伸缩。
+ **PGP：**PG分裂（手动增加数量）后，为避免新增的PG在OSD之间迁移，故引入PGP记录之前PG个数，让原先的PG保持之前的分布
+ **Pool**：对多个PG设置相同的Namespace，定义成逻辑概念上的Pool，可以对不同的业务场景做隔离（pool size 即 PG 映射到 size 个 osd）
+ **OSDs：**每一块物理硬盘对应一个osd进程。每个OSD上分布多个PG，每个PG会自动散落在不同OSD上。如果扩容OSD，那么相应的PG会进行迁移到新的OSD上，保证每个OSD上的PG数量均衡
+ **Bucket：**C-RUSH算法树的中间节点（区别于rgw对象存储中的bucket）

> 上面的映射可归纳为：file → (Pool, Object) → (Pool, PG) → OSD set → OSD/Disk



#### 3.5 crush算法

> Object在通过PG存储到实际的OSD设备上时，会通过C-RUSH算法，按照预定好的规则选择N个OSD进行存储，即CRUSH(pgid) --> (osd1,osd2 ...)。不同对象存储到OSD设备位置无必然联系，对相同对象进行重复计算，其存储位置必然相同。	

<img src=".\pictures\image-20210204143506575.png" alt="image-20210204143506575" style="zoom: 50%;" />

> ceph通过cluster_map存储分布式集群的结构拓扑，集群物理结构定义为树形结构，默认10种层级。
>
> 每一个最末端的的物理设备，也叫叶子节点就叫device，比如osd。
>
> 所有的中间节点就叫做bucket，bucket可以是一些devices的集合也可以是低一级的buckets的集合。

> 若以host为单位，则不同副本将落到不同host，如host0，再落到dev0 dev1

> 若以rack为单位，则不同副本将落到不同rack，如rack0，再落到dev0 ... 8



#### 3.6 place group

> https://blog.csdn.net/weixin_44389885/article/details/86621686
>
> https://blog.csdn.net/u014706515/article/details/100586053
>
> 指的是 pg 的状态（如osd挂了，则其上面的pg改变状态） pg:osd = 1:n
>
> 正常状态：100% active + clean

| 状态       | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| Active     | 活跃态。PG可以正常处理来自客户端的读写请求                   |
| Clean      | 干净态。PG当前不存在待修复的对象， Acting Set（OSD进程）和Up Set（crush osd）内容一致，并且大小等于存储池的副本数 <br />pg 1.a4edac00 (1.0) -> up ([1,0,2], p1) acting ([1,0,2], p1) |
| Activating | Peering已经完成，PG正在等待所有PG实例同步并固化Peering的结果（Info、Log等） |
|            |                                                              |
| Degraded   | 降级态。Peering完成后，PG检测到存在不一致（需要被同步/修复）的OSD对象，或者当前 ActingSet（OSD进程） 小于存储池副本数 |
| Undersized | 当前 ActingSet（OSD进程） 小于存储池副本数 // Degraded + Undersized 仍可以正常读写 |
|            |                                                              |
| Peering    | 正在同步态。PG正在执行同步处理                               |
| Peered     | 等待态。Peering已经完成。但是 PG当前 Acting Set 规模小于存储池最小副本数（min_size）// 此时 IO 将阻塞挂起，无法读写 |
| Remapped   | 重映射态。Peering已经完成，但 ActingSet 与 UpSet 不一致。// 可正常读写（在OSD挂掉或扩容时，PG会按照Crush算法重新分配PG所属OSD编号，并把PGRemap到新OSD） |
|            |                                                              |
| Recovery   | PG 通过 PGLog 针对数据不一致的OSD对象进行同步修复。 // PGLog 在osd_max_pg_log_entries=10000条以内时，可通过PGLog增量恢复数据 |
| Backfill   | PG 无法通过 PGLog 恢复所需数据，则需要通过完全拷贝当前 OSD Primary 进行全量同步 // 如果 PGLog 超过osd_max_pg_log_entries=10000条， 这个时候需要全量恢复数据 |
|            |                                                              |
| Stale      | 未刷新态。 PG 存储的所有 OSD 都挂掉（单个挂掉可转移）；或者 Mon 没有检测到 OSD Primary 统计信息（网络抖动） |
| Down       | 宕机态。当前剩余在线的 OSD 不足以完成数据修复                |

#### 3.7 ceph csi

https://github.com/ceph/ceph-csi/blob/devel/docs/design/proposals/rbd-snap-clone.md

+ rbd create source

``` 
[utils.go:176] ID: 674 Req-ID: pvc-4e1ba0f1-cf74-44c2-ab8e-ae14cfffe585 GRPC call: /csi.v1.Controller/CreateVolume
[rbd_util.go:242] ID: 674 Req-ID: pvc-4e1ba0f1-cf74-44c2-ab8e-ae14cfffe585 rbd: create rbd/csi-vol-387c2163-6bfd-11ed-90a1-0000006cf3fe size 10240M (features: [layering]) using mon 192.168.211.31:6789
```

+ rbd snap create

```
[utils.go:176] ID: 676 Req-ID: snapshot-d47037e9-68a9-48ef-bb66-bd3981997506 GRPC call: /csi.v1.Controller/CreateSnapshot
[rbd_util.go:1266] ID: 676 Req-ID: snapshot-d47037e9-68a9-48ef-bb66-bd3981997506 rbd: snap create rbd/csi-vol-387c2163-6bfd-11ed-90a1-0000006cf3fe@csi-snap-ed39d536-6bfd-11ed-90a1-0000006cf3fe using mon 192.168.211.31:6789
[rbd_util.go:1325] ID: 676 Req-ID: snapshot-d47037e9-68a9-48ef-bb66-bd3981997506 rbd: clone rbd/csi-vol-387c2163-6bfd-11ed-90a1-0000006cf3fe@csi-snap-ed39d536-6bfd-11ed-90a1-0000006cf3fe rbd/csi-snap-ed39d536-6bfd-11ed-90a1-0000006cf3fe (features: [deep-flatten layering]) using mon 192.168.211.31:6789
[rbd_util.go:1279] ID: 676 Req-ID: snapshot-d47037e9-68a9-48ef-bb66-bd3981997506 rbd: snap rm rbd/csi-vol-387c2163-6bfd-11ed-90a1-0000006cf3fe@csi-snap-ed39d536-6bfd-11ed-90a1-0000006cf3fe using mon 192.168.211.31:6789
[rbd_util.go:1266] ID: 676 Req-ID: snapshot-d47037e9-68a9-48ef-bb66-bd3981997506 rbd: snap create rbd/csi-vol-387c2163-6bfd-11ed-90a1-0000006cf3fe@csi-snap-ed39d536-6bfd-11ed-90a1-0000006cf3fe using mon 192.168.211.31:6789
[rbd_util.go:704] ID: 676 Req-ID: snapshot-d47037e9-68a9-48ef-bb66-bd3981997506 clone depth is (1), configured softlimit (4) and hardlimit (8) for rbd/csi-snap-ed39d536-6bfd-11ed-90a1-0000006cf3fe
```

**Deep-flatten** makes `rbd flatten` work on all the snapshots of an image, in addition to the image itself. Without it, snapshots of an image will still rely on the parent, so the parent will not be delete-able until the snapshots are deleted. Deep-flatten makes a parent independent of its clones, even if they have snapshots.

+ rbd snap clone

```
[utils.go:176] ID: 685 Req-ID: pvc-ab9bb529-616e-4878-bb6f-ecc2bd3b09ac GRPC call: /csi.v1.Controller/CreateVolume
[rbd_util.go:1187] ID: 685 Req-ID: pvc-ab9bb529-616e-4878-bb6f-ecc2bd3b09ac setting disableInUseChecks: true image features: [layering] mounter: rbd
[rbd_util.go:1325] ID: 685 Req-ID: pvc-ab9bb529-616e-4878-bb6f-ecc2bd3b09ac rbd: clone rbd/csi-snap-ed39d536-6bfd-11ed-90a1-0000006cf3fe@csi-snap-ed39d536-6bfd-11ed-90a1-0000006cf3fe rbd/csi-vol-c834d502-6bff-11ed-90a1-0000006cf3fe (features: [layering]) using mon 192.168.211.31:6789
[rbd_util.go:704] ID: 685 Req-ID: pvc-ab9bb529-616e-4878-bb6f-ecc2bd3b09ac clone depth is (2), configured softlimit (4) and hardlimit (8) for rbd/csi-vol-c834d502-6bff-11ed-90a1-0000006cf3fe
```

+ rbd delete source

```
[utils.go:176] ID: 686 Req-ID: 0001-0024-edeca71c-f155-44da-af0d-9a5d29203f93-0000000000000009-387c2163-6bfd-11ed-90a1-0000006cf3fe GRPC call: /csi.v1.Controller/DeleteVolume
[rbd_util.go:540] ID: 686 Req-ID: 0001-0024-edeca71c-f155-44da-af0d-9a5d29203f93-0000000000000009-387c2163-6bfd-11ed-90a1-0000006cf3fe rbd: delete csi-vol-387c2163-6bfd-11ed-90a1-0000006cf3fe-temp using mon 192.168.211.31:6789, pool rbd
[rbd_util.go:540] ID: 686 Req-ID: 0001-0024-edeca71c-f155-44da-af0d-9a5d29203f93-0000000000000009-387c2163-6bfd-11ed-90a1-0000006cf3fe rbd: delete csi-vol-387c2163-6bfd-11ed-90a1-0000006cf3fe using mon 192.168.211.31:6789, pool rbd
[rbd_util.go:493] ID: 686 Req-ID: 0001-0024-edeca71c-f155-44da-af0d-9a5d29203f93-0000000000000009-387c2163-6bfd-11ed-90a1-0000006cf3fe executing [rbd task add trash remove rbd/ffebe2a5b4cce --id admin --keyfile=/tmp/csi/keys/keyfile-279865204 -m 192.168.211.31:6789] for image (csi-vol-387c2163-6bfd-11ed-90a1-0000006cf3fe) using mon 192.168.211.31:6789, pool rbd
[cephcmds.go:60] ID: 686 Req-ID: 0001-0024-edeca71c-f155-44da-af0d-9a5d29203f93-0000000000000009-387c2163-6bfd-11ed-90a1-0000006cf3fe command succeeded: ceph [rbd task add trash remove rbd/ffebe2a5b4cce --id admin --keyfile=***stripped*** -m 192.168.211.31:6789]
```

+ rbd delete snap

```
[utils.go:176] ID: 687 Req-ID: 0001-0024-edeca71c-f155-44da-af0d-9a5d29203f93-0000000000000009-ed39d536-6bfd-11ed-90a1-0000006cf3fe GRPC call: /csi.v1.Controller/DeleteSnapshot
[rbd_util.go:1279] ID: 687 Req-ID: 0001-0024-edeca71c-f155-44da-af0d-9a5d29203f93-0000000000000009-ed39d536-6bfd-11ed-90a1-0000006cf3fe rbd: snap rm rbd/csi-snap-ed39d536-6bfd-11ed-90a1-0000006cf3fe@csi-snap-ed39d536-6bfd-11ed-90a1-0000006cf3fe using mon 192.168.211.31:6789
[rbd_util.go:540] ID: 687 Req-ID: 0001-0024-edeca71c-f155-44da-af0d-9a5d29203f93-0000000000000009-ed39d536-6bfd-11ed-90a1-0000006cf3fe rbd: delete csi-snap-ed39d536-6bfd-11ed-90a1-0000006cf3fe using mon 192.168.211.31:6789, pool rbd
[rbd_util.go:493] ID: 687 Req-ID: 0001-0024-edeca71c-f155-44da-af0d-9a5d29203f93-0000000000000009-ed39d536-6bfd-11ed-90a1-0000006cf3fe executing [rbd task add trash remove rbd/ffebe9f8d96ba --id admin --keyfile=/tmp/csi/keys/keyfile-647801411 -m 192.168.211.31:6789] for image (csi-snap-ed39d536-6bfd-11ed-90a1-0000006cf3fe) using mon 192.168.211.31:6789, pool rbd
[cephcmds.go:60] ID: 687 Req-ID: 0001-0024-edeca71c-f155-44da-af0d-9a5d29203f93-0000000000000009-ed39d536-6bfd-11ed-90a1-0000006cf3fe command succeeded: ceph [rbd task add trash remove rbd/ffebe9f8d96ba --id admin --keyfile=***stripped*** -m 192.168.211.31:6789]
```

+ rbd delete clone

```
[utils.go:176] ID: 688 Req-ID: 0001-0024-edeca71c-f155-44da-af0d-9a5d29203f93-0000000000000009-c834d502-6bff-11ed-90a1-0000006cf3fe GRPC call: /csi.v1.Controller/DeleteVolume
[rbd_util.go:540] ID: 688 Req-ID: 0001-0024-edeca71c-f155-44da-af0d-9a5d29203f93-0000000000000009-c834d502-6bff-11ed-90a1-0000006cf3fe rbd: delete csi-vol-c834d502-6bff-11ed-90a1-0000006cf3fe-temp using mon 192.168.211.31:6789, pool rbd
[rbd_util.go:540] ID: 688 Req-ID: 0001-0024-edeca71c-f155-44da-af0d-9a5d29203f93-0000000000000009-c834d502-6bff-11ed-90a1-0000006cf3fe rbd: delete csi-vol-c834d502-6bff-11ed-90a1-0000006cf3fe using mon 192.168.211.31:6789, pool rbd
[rbd_util.go:493] ID: 688 Req-ID: 0001-0024-edeca71c-f155-44da-af0d-9a5d29203f93-0000000000000009-c834d502-6bff-11ed-90a1-0000006cf3fe executing [rbd task add trash remove rbd/ffebe97205856 --id admin --keyfile=/tmp/csi/keys/keyfile-887617734 -m 192.168.211.31:6789] for image (csi-vol-c834d502-6bff-11ed-90a1-0000006cf3fe) using mon 192.168.211.31:6789, pool rbd
[cephcmds.go:60] ID: 688 Req-ID: 0001-0024-edeca71c-f155-44da-af0d-9a5d29203f93-0000000000000009-c834d502-6bff-11ed-90a1-0000006cf3fe command succeeded: ceph [rbd task add trash remove rbd/ffebe97205856 --id admin --keyfile=***stripped*** -m 192.168.211.31:6789]
```



```shell
# rbd clone
validate_parent: parent snapshot must be protected

# rbd rm
rbd: image has snapshots - these must be deleted with 'rbd snap purge' before the image can be removed.

# rbd snap rm
librbd::Operations: snapshot is protected

# rbd snap unprotect
cannot unprotect: at least 2 children
```

