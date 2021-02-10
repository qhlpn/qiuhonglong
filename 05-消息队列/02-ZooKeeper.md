### 注册中心 ZooKeeper【CP】 

> 分布式协调服务
>
> 1. 数据发布 / 订阅【监听器】
> 2. 命名服务
> 3. 分布式锁【临时顺序节点】
> 4. Master选举【eg Kafka，节点竞争，Zab顺序执行】

### 核心概念

+ **数据模型：Tree **

  <img src="E:\projects\grocery\qiuhonglong\05-消息队列\pictures\image-20201209194329666.png" alt="image-20201209194329666" style="zoom:90%;" />

+ **数据节点：ZNode**

  > 节点存放数据 [ max_size = 1M ]，注意：协调服务**不是**用来存放**业务数据**

  + **节点类型**
    + **持久节点**：创建即持久化，直至将其删除
    + **临时节点**：生命周期与客户端会话绑定，只能是叶子节点，不能创建子节点
    + **持久顺序节点**：除了持久，其**子节点**还具有顺序性【/app/00001; /app/0002】
    + **临时顺序节点**：除了临时，**同时可创建子节点**，其**子节点**还具有顺序性**【分布式锁】**
  + **数据结构**
    + **stat：znode 状态信息**
      + 版本版本号
      + 操作事务号
    + **data：znode 存放数据**

+ **ACL 权限控制**

+ **Watcher 客户端监听器**

  <img src="E:\projects\grocery\qiuhonglong\05-消息队列\pictures\image-20201209200251835.png" alt="image-20201209200251835" style="zoom:90%;" />

+ **Session 会话**

  > Session 是 Zookeeper 与 Client 间的 **TCP 长连接**
  >
  > 重要属性：
  >
  > 1. **sessionID**：会话标志，全局唯一
  > 2. **sessionTimeout**：在超时时间内 Client 与 Zookeeper 重连，则之前的会话仍然有效（sessionId）



### Zookeeper 集群 【Zab协议】

<img src="E:\projects\grocery\qiuhonglong\05-消息队列\pictures\image-20201209202004775.png" alt="image-20201209202004775" style="zoom:90%;" />

> 在 ZooKeeper 中除了 Leader / Follower，还引入了 **Observer** 角色
>
> Leader 既可以为客户端提供写服务又能提供读服务
>
> Follower 和 Observer 都只能提供读服务。Follower 和 Observer 唯一的区别在于 Observer 不参与 Leader 选举过程，也不参与写操作的“过半写成功”策略，因此 Observer 机器可以在不影响写性能的情况下提升集群的读性能。

> **顺序执行 + 消息广播 + 奔溃恢复【选举同步】** [见 Zab 协议](../06-分布式/01-数据一致性.md) 