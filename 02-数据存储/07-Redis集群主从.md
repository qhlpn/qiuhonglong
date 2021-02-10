### Redis 高性能、高可用

+ **主从架构（ 读写分离 ）**

  > master - slave：全量复制 + 增量复制 + 异步复制 + 断点续传

  ![](.\pictures\071.png)

  + **redis.conf**

    ```shell
    # 开启 无磁盘化 全量复制，直接在内存中生成 RDB
    repl-diskless-sync yes
    
    # 等待 5s 后再开始复制，用于多 slave 连接
    repl-diskless-sync-delay 5
    
    # 停止复制情况：rdb文件传输时间超过 60s
    #            异步复制所需内存持续超过 64M、一次性超过 256M
    client-output-buffer-limit slave 256MB 64MB 60
    ```

  + **主从搭建**

    ```shell
    # master redis.conf
    bind 127.0.0.1  # 注释掉，即 bind 0.0.0.0
    daemonize no    # yes 时 docker -d 启动失败
    requirepass 123456  # 访问密码
    maxclients <>   # 最大连接数
    maxmemory <>	# 最大内存
    maxmemory-policy volatile-lru # 内存淘汰策略
    appendonly yes  # 开启 AOF
    
    # slave redis.conf 除了上面，还有
    replicaof 172.17.0.6 6379 # 作为哪个主节点的从节点
    masterauth 123456
    
    # redis 6.09
    $ docker run -d -p 6379:6379 -v /opt/redis/redis01/conf/redis.conf:/etc/redis/redis.conf -v /opt/redis/redis01/data/:/data redis redis-server /etc/redis/redis.conf
    # docker redis 默认无配置文件，需手动 redis-server 指定开启
    
    $ docker run -d -p 6378:6379 -v /opt/redis/redis02/conf/redis.conf:/etc/redis/redis.conf -v /opt/redis/redis02/data/:/data redis redis-server /etc/redis/redis.conf
    
    $ redis-cli
    $ auth 123456
    
    $ set username nick   # master set
    $ get username        # slave get
    ```

+ **Sentinel 哨兵**

  > Redis 主备模式下，当 master 宕机时，写服务无法使用，需要手动切换。Sentienl 则可以实现自动主备切换。

  <img src=".\pictures\072.png" style="zoom:75%;" />

  + **Sentinels 集群**

    + **Raft 分布式一致性**

      > 解决问题：对集群进行操作时如何保证各节点数据的一致性
      >
      > 核心概念：选举任期、超过半数、节点状态【跟随者、候选者、领导者】
      >

      + **Leader选举**：每个跟随着节点 **随机的选举超时** 时间，直到仅有一个节点 **最先** 发起候选投票，**超过半数** 节点同意则晋升为领导者。领导者是分布式系统的 **操作入口** 。
      + **日志复制流程**：客户端想更新数据，Leader **先记录**日志，此时状态为 **uncommitted**；并将日志同步给 Fllower节点，当超过半数节点同步日志时，Leader 状态变为 **commited** **再更新**数据；最后通知其它所有节点更新数据至集群状态一致。    
      + **网络分区容错**：不同分区下的节点会 **各自选举**，产生多个 Leader；当网络恢复正常时，选举任期高的 Leader 留下，其它 Leader 转化为 Follower，并根据 uncommitted 日志进行操作回滚；最后同步所有节点。

      + **故障转移执行者**

        > 当哨兵 ping redis-master 超过 ```is-master-down-after-milliseconds``` 时则主观认为 master 宕机，并通知其它哨兵。当某哨兵接收 **超过半数** 的宕机通知，则开启 **执行故障转移的哨兵选举** 流程。

    + **哨兵注册发现**
      
      > 通过 redis 的**发布/订阅系统**实现， 即监听 ```__sentinel__:hello``` 管道。

  + **Redis 主备切换**
    
    + **Master 选举算法**
    
      > 对 slave 的相关信息进行排序：
        >
        > 1. 与 master 断开连接的时长，越短优先级越高
        > 2. slave priority 值越低，优先级越高
        > 3. replica offset 同步数据越多，优先级越高
    
  + **数据丢失问题**
    
    > Sentinel 只能保证 Redis 主从可用，不保证数据零丢失，是 AP 模型
    
    + **异步数据复制**：master 在部分数据还未同步到 slave 前就宕机
    + **多个master（脑裂）**：master 与集群网络分区时，哨兵会再选举出新的 master；当分区结束时，旧的 master 会作为 slave，清空自身数据，挂到新的 master 上，从而丢失 **分区期间** 接收到的客户端数据。
    
    > 解决方法：`min-slaves-to-write 1`        `min-slaves-max-lag 10` 
      >
      > 要求主机至少有1个从机，同时数据同步时的延迟不能超过10s，否则主机拒绝写请求。
    
    

### Redis Cluster

> **集群 + 主从**：Redis cluster 支撑 N 个 Redis master node，每个 master node 可以挂载多个 slave node，实现了高可用和高性能 

+ **数据一致性**

  + **gossip 协议**

    > 集中式：将元数据（节点信息、故障信息）统一存储在组件上进行维护，比如 zookeeper。优点：时效性好。缺点：组件接收更新元数据请求的压力大。

    <img src=".\pictures\073.png" style="zoom:70%;" />

    > gossip：所有节点持有一份元数据，当节点更新了数据，就通知其它节点进行 **数据同步** 。优点：分散节点更新i元数据的请求。缺点：更新有时延滞后。

    > gossip 消息命令有： meet、ping、pong、fail

    > 传播机制：**周期** 发送 + **固定** 个数 + **随机** 路线。log(20)(base 4) = 2.16，表示集群中有20个节点，受感染节点每周期会随机向另外4个节点（可以重复感染）成功同步数据。全部感染最少需要2.16个周期。

    <img src=".\pictures\074.png" style="zoom: 30%;" />

+ **分布式寻址算法**

  + **一致性哈希 = 哈希环 + 虚拟节点 **

    + **哈希环**：解决 **节点动态扩收容时全部节点失效需要重哈希问题** 。将节点进行哈希，定位到哈希环上，比如通过IP来哈希。缓存数据先哈希定位环上，归入顺时针方向的第一个节点中。这样收扩容时只需要对受影响数据进行转移，而不影响其它数据。

    + **虚拟节点**：解决 **节点过少哈希不均匀造成的缓存倾斜存储问题** 。对每个节点（比如将信息进行虚拟映射扩充）计算多个哈希值，虚拟节点均匀定位到哈希环上，实现负载均衡。

    <img src=".\pictures\075.png" style="zoom:45%;" />

  + **Hash Slot【Redis-Cluster 使用】**

    > Redis-Cluster 将 16384 个 虚拟槽 分配给集群中的 Masters。Key 哈希取模定位到 Slot 上，机器宕机则将其 Slot 转移到其它机器上，数据不会全部失效。Key 定位的是 Slot 而不是 机器。 

    

+ **Redis-Cluster 搭建**

  + **原生搭建**

    ```shell
    # common 配置
    bind 0.0.0.0
    daemonize no
    requirepass 123456
    masterauth 123456
    maxmemory-policy volatile-lru
    appendonly yes
    # cluster 配置
    cluster-enabled yes
    cluster-config-file cluster.conf    # 保存集群配置
    cluster-node-timeout 15000          # 节点不可访问的最大时间
    cluster-replica-validity-factor 10  # 集群副本有效因子
    cluster-require-full-coverage  no   # 节点全活时集群功能才有效 no
    
    $ docker run -d -p 637x:6379 -v /opt/redis/redis.conf:/etc/redis/redis.conf redis redis-server /etc/redis/redis.conf
    
    $ reids-cli
    $ auth 123456
    
    # 节点加入集群
    $ cluster nodes  # 查看集群下的节点
    $ cluster meet redis-ip:port # 邀请其它节点接入集群
    
    # 指派槽位
    $ cluster addslots slot-number
    # assign-slot.sh 
    start=$1
    end=$2
    for slot in `seq ${start} ${end}`
    do
        echo "slot:${slot}"
        /usr/local/bin/redis-cli -h 127.0.0.1 -p 6379 -a 123456 cluster addslots ${slot}
    done
    $ sh assign-slot.sh 0 5461
    $ sh assign-slot.sh 5462 10922
    $ sh assign-slot.sh 10923 16383
    $ cluster nodes
    
    # 指定从机
    $ cluster replicate master-id # 指定当前节点作为主节点的从机
    ```

  + **快速搭建**

    ```shell
    # 启动集群 cluster create 指令
    $ redis-cli --cluster create 172.17.0.6:6379 172.17.0.7:6379 172.17.0.8:6379 172.17.0.9:6379 172.17.0.10:6379 172.17.0.11:6379 --cluster-replicas 1 -a 123456 # --cluster-replicas 2 一主二从
    
    # 新节点加入集群 cluster add-node 指令
    $ redis-cli --cluster add-node 172.17.0.12:6379 172.17.0.6:6379 --cluster-master-id masterId -a 123456 # [newNode oldNode] --cluster-master-id 指定主节点ID，若不指定则加入集群后称为主节点
    
    # 迁移槽位和数据 cluster reshard 指令
    $ redis-cli --cluster reshard 172.17.0.12:6379 -a 123456  # 扩容时
    $ redis-cli --cluster reshard --cluster-from sendId --cluster-to recieveId --cluster-slots slotNum recieveIp:Port -a 123456
    
    # 集群停机节点 cluster delete 指令
    redis-cli --cluster del-node 172.17.0.12:6379 edc8ff41aef320beb5081c5b50bf32485a7ffb9e -a 123456
    ```