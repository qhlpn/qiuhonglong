### 消息队列

+ **提高响应性能（异步）**
+ **提高系统可用性（削峰）**
+ **降低系统耦合性（解耦）**



### Kafka 架构

<img src="pictures\image-20201127160901921.png" alt="image-20201127160901921" style="zoom:50%;" />

+ **Producer **：生产者

+ **Consumer**：消费者

+ **Consumer Group**：消费者组，组内消费者不会消费同个消息，组可实现消息 **广播和单播**

+ **Broker**：Kafka 服务器

+ **Topic**：主题（逻辑概念），用来区分不同类型的消息，类似于数据库的表

+ **Partition**：分区（物理存储），每个 Partition 对应一个目录，里面存储消息文件和索引文件，**分区可并发读写提速**

+ **Offset**：偏移量，记录 Group 消费 Partition 所处位置。**可实现Partition 在组内的消费有序**，**不保证主题级别有序性**

+ **Replication**：副本，每个 Partition 数据可在**其它** Broker 存有副本，**实现高可用，不支持读写分离（高版本支持）**

+ **Record**：消息记录，包括 key value 和 timestamp 字段

  

### Producer

> 生产者发送消息（**推模式**），先后经过  `拦截器`  `序列化器`  `分区器`，最终由  `累加器`  批量发送到服务器。

+ **客户端组件**

  + **KafkaProducer**：生产者客户端，启动主进程
  + **RecordAccumulator**：消息收集器，收集消息并**分批缓存**起来
  + **Sender**：发送器，负责读取收集器缓存的**批量信息**，交由 Selector 进行网络传输
  + **Selector**：选择器，处理**网络连接**和**数据读写**处理

  <img src="pictures\image-20201127163859254.png" alt="image-20201127163859254" style="zoom:70%;" />

  <img src="pictures\image-20201201200920315.png" alt="image-20201201200920315" style="zoom:50%;" />

  > 1. 创建消息，Topic Value 必填，Key Partition 选填
  > 2. Seriallizer：对key value **序列化** 为二进制（网络传输格式要求）
  > 3. Partitioner：消息 **计算分区** partition （人工指定 / key 哈希求余 / 轮询）
  > 4. RecordAccumulator：进行**分批缓存**（使用双端队列，同批消息的 topic 和 partition 相同）
  > 5. Sender：进一步将属于同个Broker的多个批次消息进行**打包**，再发送

+ **常见参数**

  ```properties
  bootstrap.server: 	broker ip + port
  key.serializer: 	key序列化器
  value.serializer: 	value序列化器
  batch.num.messages:	批量值（仅对 async 模式）
  request.required.acks: dft 0 无需等待leader确认 / 1 需leader确认写入本地log / -1 需所有备份都完成（仅对 async 模式）
  request.timeout.ms:	确认超时时间
  message.send.max.retries:	最大重试次数 dft 3
  retry.backoff.ms:	每次尝试增加的额外间隔时间 dft 300
  partitioner.class:	消息分区策略，dft kafka.producer.DefaultPartitioner，可自定义实现 kafka.producer.Partitioner
  producer.type:		消息发送模式（dft sync 同步 / async 异步）
  compression.topic:	压缩方式（dft none / gzip / snappy / lz4）
  compressed.topics:	要压缩的topic，dft null
  topic.metadata.refresh.interval.ms:	定期获取元数据时间 dft 600000（0则每次发送后获取，负数则只在发送失败获取），leader不可以时也会主动获取
  queue.buffering.max.ms:	缓存消息最大持续时间 dft 5000 （仅对 async 模式）
  queue.buffering.max.message:	缓存消息最大数量 dft 10000 （仅对 async 模式）
  queue.enqueue.timeout.ms:	dft -1 （0 缓存满则丢掉 / -1 满则阻塞 / 正值代表满则阻塞的时间 仅对 async 模式）
  ```



### Consumer / Group

> 消费者接收消息（**拉模式**），**将 offset 放在客户端管理** 

> Consumer / Group 结合可实现 **单播 和 广播**

<img src="pictures\image-20201127170453911.png" alt="image-20201127170453911" style="zoom:70%;" />

> 1. 消费者客户端根据配置创建连接器
>
> 2. **拉取线程**从服务端拉取消息写入缓存队列
>
> 3. **消费进程**轮询队列信息，进行业务逻辑消费

<img src="pictures\image-20201201204341802.png" alt="image-20201201204341802" style="zoom: 67%;" />



+ **多线程消费 方案**

  > **KafkaConsumer** 是 **线程不安全** 的（多个线程共享同个consumer，同时进行consumer.poll，会有数据问题）

  + **同个Group中建立多个Consumer，不同线程对应不同的Consumer，进行消息获取和消息处理**

    <img src="pictures\image-20201130102545260.png" alt="image-20201130102545260" style="zoom:80%;" />
  
  ```java
  public class Consumer1 implements Runnable {
      private final KafkaConsumer consumer = ;
      public void run() {
          consumer.subscribe(Arrays.asList("topic"));
      }
  }
  executors.submit(consumer1);
  ...
  ```

  + **消息接收由单个线程跑单个Consumer；消息处理由不同的线程分别对应不同的Worker。**
**消息接收和消息处理解耦，即 Reactor模型 = IO多路复用 + 事件驱动**
    
  
  <img src="pictures\image-20201130174408275.png" alt="image-20201130174408275" style="zoom:80%;" />
  
    ```java
    private final KafkaConsumer<String, String> consumer;
    private ExecutorService executors;
    executors = new ThreadPoolExecutor(
                workerNum, workerNum, 0L, TimeUnit.MILLISECONDS,
                new ArrayBlockingQueue<>(1000),
                new ThreadPoolExecutor.CallerRunsPolicy());
    while (true) {
    	ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
      	for (final ConsumerRecord record : records) {
        	executors.submit(new Worker(record));
      	}
    }
    ```
  
+ **常见参数**

  ```properties
  bootstrap.servers:	broker ip + port
  group.id:			组id
  key.deserializer:	key反序列化
  value.deserializer:	value反序列化
  request.timeout.ms:	请求等待时间，超过则可重发可失败
  session.timeout.ms:	心跳间隔时间 dft 10s
  max.poll.records:	每次拉取消息数 dft 500（须在session.timeout.ms内处理完）
  fetch.max.bytes:	每次拉去最大字节数
  enable.auto.commit:	是否自动提交位移（最好手动提交）
  auto.offset.reset:	消息没携带偏移量时（无效）时应作何处理。latest dft 最近的记录 / earliest 最初始的记录
  ```

+ **Rebalance 重平衡**

  > 同组Group 的 消费者Consumer 如何分配 某主题Topic 的所有 消息Record

  + **Rebalance 触发条件**
    
    + 组Group成员 **Consumer数量** 发生变更
    + 主题的 **Partition 分区数** 发生变更
    
    > 重平衡期间，所有消费者都不能消费消息，造成消费组短暂不可用。
    
  + **组内分配策略**
    
    + **RangePartitionAssignor 默认 等分连续区间**
    + **RoundRobinAssignor 轮询分配**
    + **AbstractPartitionAssignor 自定义 ** （手动控制主题级别的消息有序）
  
+ **Zookeeper**

  + zookeeper 保存元信息，消费消息前后，需要 **拉取 和 更新** 信息

    > 保存 topic.partition.**offset** for consumer_group 消费偏移量
    >
    > 保存 consumer_group 消费组的成员列表

  + 新版本 Kafka 将 offset 存储 **从 Zookeeper 转移到 Brokers**

  + **三种消费情况**

    > 至少一次：拉取  处理【若宕机】  更新
    >
    > 至多一次：拉取  更新【若宕机】  处理
    >
    > 正好一次：拉取 【更新  处理】 (原子，prepare commit 事务)

  <img src="pictures\image-20201201204135402.png" alt="image-20201201204135402" style="zoom:100%;" />

### 消息物理存储形式

+ **Partition  Replica  Log  LogSegment**

  <img src="pictures\image-20201202193647382.png" alt="image-20201202193647382" style="zoom:90%;" />

  > 如图 topic 划分成3个 partition，数据分布在不同的主机上（分片提高性能）；同时，每个 partition 有主从副本，两者**需分布在不同**的主机上，不能同时宕机（副本提高可用性）。 
  >
  > 每个 partition 物理存储上都为1个目录 log，目录中 **partition 被平均切割的大小相等 segment file**，其中 .index 和 .log 成对存在，表示 segment 索引文件和数据文件。
  >
  > **消息通过三个属性确定**：**offset**（在 partition 中的偏移量（第几条）） **size** （消息大小） **data**（消息内容）

  ```
  | --topic1-p0
      | --00000000000000000000.index
      | --00000000000000000000.log
      | --00000000000000368769.index
      | --00000000000000368769.log
      | --00000000000000737337.index
      | --00000000000000737337.log
      | --00000000000001105814.index
      | --00000000000001105814.log
  | --topic2-p0
  | --topic2-p1
  ```

  <img src="pictures\image-20201202195520785.png" alt="image-20201202195520785" style="zoom:100%;" />

  > 以 <3, 497> 为例，3 代表 数据文件中 第3个 消息（**个数偏移**），全局上第 368769 + 3 = 368772 个消息（**offset**）；497 代表 消息在文件中的**物理地址偏移**

### Kafka 集群（高可用 高性能）

+ **Partition 负载均衡 + 副本冗余 ** 

  > Partition 分为主从，其中 Leader  负责**读写**，Preferred **只**负责同步冗余  **（高可用，前期不支持读写分离，高版本支持）** 
  >
  > 集群中主节点的写入请求，**request.required.acks** 可控制数据可靠性等级（如是否等同步到从节点再返回响应）
  >
  > 主节点应负载均衡散落在各个 Broker 上 **（高性能）** 

  <img src="pictures\image-20201203194520982.png" alt="image-20201203194520982" style="zoom:67%;" />

  

+ **Kafka 选举基于 Zookeeper 实现** 

  > Kafka **集群** 需要选出 **Leader Broker** 对内管理（故障转移）
  >
  > Zookeeper 四种目录：永久、永久顺序、临时、临时顺序
  >
  > Zookeeper 利用 **临时** 特性进行选主。节点谁先写入谁是 Leader；Leader 宕机则删除目录，节点重新竞争。

  <img src="pictures\image-20201203192226471.png" alt="image-20201203192226471" style="zoom:80%;" />

  > Zookeeper 是 **CP**，用在对 **数据一致性要求高** 的场景，不能保证每次服务请求的可用性。
  >
  > Zookeeper **自己的**集群 基于 **Zab** 协议。 



### 消息幂等性

+ 前面可知，consumer 有 三种消费情况：至多一次、至少一次、**正好一次（幂等，不重复消费）**

  + **原子操作** 可以直接实现正好一次

    > prepare commit 二阶段提交

  + **业务操作** 可以在重复消费情况下实现正好一次

    > 比如 MySQL唯一键插入报错、Redis集合Set天然幂等、生成uuid做记录，消费时比对



### 消息消费有序

> 首先，消息应该打进同一个 Partition（可人工指定 key，也可指定 pid），从而进入同一个消费者。
>
> 其次，由于消费者拉取消息后会多线程处理，故消息应再次打入同一个 Queue（可根据 key）,每个线程消费一个Queue。

<img src="pictures\image-20201203203137547.png" alt="image-20201203203137547" style="zoom:90%;" />



### 消息可靠传输

> 即：如何防止消息丢失问题

+ 消费端：关闭 **拉取消息后 offset 自动提交**，改成 **处理完消息后** 手动提交offset，再自行解决幂等

+ 生产端：设置 **request.required.acks = -1**, 保证数据在集群中强一致，如果失败则不断重试 **message.send.max.retries = bigint** 
+ Broker端：开启 **Partiton 副本**冗余，即 **min.insync.replicas > 1**，防止宕机数据丢失

