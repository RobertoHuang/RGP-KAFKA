# 副本机制

> `Kafka`从`0.8`版本开始引入副本`Replica`的机制，其目的是为了增加`Kafka`集群的高可用性
>
> 其中`Leader`副本负责读写，`Follower`副本负责从`Leader`拉取数据做热备，副本分布在不同的`Broker`上
>
> 
>
> `AR(Assigned Replica)`副本集合(`Leader+Follower`的总和)
>
> `ISR(IN-SYNC Replica)`同步副本集表示目前可用`Alive`且消息量与`Leader`相差不多的副本集合
>
> - 副本所在节点必须维持着与`Zookeeper`连接
> - 副本最后一条消息的`Offset`与`Leader`副本的最后一条消息的`Offset`之间差值不能超过指定阈值

## Replica

 `Kafka`使用一个`Replica`对象表示一个分区的副本，主要用于管理`HW`与`LEO`

- `updateLogReadResult`更新副本相关信息(`Leader`收到`Follower`的`Fetch`请求时调用)

  ```reStructuredText
  1.更新_lastCaughtUpTimeMs
      若Follower副本的Fetch请求Offset赶上了Leader的LEO则更新_lastCaughtUpTimeMs当前Fetch请求的时间和_lastCaughtUpTimeMs的最大值，如果Follower副本的Fetch请求Offset赶上了上次Fetch请求时Leader的LEO，则更新_lastCaughtUpTimeMs为_lastCaughtUpTimeMs和上一次Fetch请求时间的最大值
      
  2.更新副本的LSO和LEO信息
  
  3.记录本次Fetch请求Leader的LEO和Fetch时间用于步骤1的逻辑处理
  ```

## Partition

- `getOrCreateReplica`获取/创建副本

  > 如果创建的是本地副本还会创建/恢复对应的`Log`并初始化/恢复`HW`

- `makerLeader/makeFollower`更新副本为`Leader/Follow`副本

  > `Broker`会根据`KafkaController`发出的`LeaderAndISRRequest`请求控制副本的角色切换【按照`PartitionState`指定的信息将`leader/follow`字段指定的副本转换成`Leader/Follow`副本】

- `maybeIncrementLeaderHW`尝试更新`Leader`副本的`HW`

  ```reStructuredText
  1.这个方法将在两种情况下触发:当ISR集合发生增减或是ISR集合中任一副本的LEO发生变化时都可能导致ISR集合中最小的LEO变大，所以这些情况都要调用maybeIncrementLeaderHW方法进行检测
  
  2.在获取HW时是从ISR和认为能追得上的副本中(lastCaughtUpTimeMs<replicaLagTimeMaxMs)选择最小的LEO，之所以也要从能追得上的副本中选择是为了拖住HW的增长，这样稍微落后的Follower副本也能尽快赶上加入ISR副本【在附录中会详细介绍】
  ```

- `maybeShrinkIsr`管理`ISR`集合【收缩】

  ```reStructuredText
  1.ISR集合是由Leader进行管理
  
  2.该定时任务遍历所有本地所有Leader副本，通过检测ISR副本的lastCaughtUpTimeMs字段找出已滞后的ISR集合，并从ISR集合中移除【无论是长时间没有与Leader副本进行同步还是其LEO与HW相差太大都可以从该字段反应出来】。当ISR集合发生变化时需判断是否更新Partition的HW值
  ```

- `maybeExpandIsr`管理`ISR`集合【扩展】

  ```reStructuredText
  随着Follower副本不断与Leader副本进行消息同步，Follower副本的LEO会逐渐后移并最终能赶上Leader副本的HW，此时该Follower副本就有资格进入ISR集合，该方法在Follower副本往Leader抓取数据时调用
  
  1.如果该副本之前不在ISR中，且当前抓取的Offset已经赶上Leader的HW值，则加入到ISR集合中
  ```

- `appendRecordsToLeader`向`Leader`副本对应的`Log`中追加消息

- `checkEnoughReplicasReachOffset`检查是否足够多的`Follower`副本已同步消息

## ReplicaManager副本管理器

- `appendToLocalLog`追加消息
  - 检测目标`Topic`是否是`Kafka`的内部`Topic`及是否允许向内部`Topic`追加数据
  - 根据`topicPartition`获取对应的`Partition`【`Partition`是否存在及是否是`OfflinePartition`】
  - 调用`Partition`的`appendRecordsToLeader`方法往`Log`中追加消息
- `readFromLocalLog`读取消息
  - `minOneMessage`保证即使在有单次读取大小限制的情况下，至少读取到一条数据
  - 根据`TopicPartition`获取`Replica`信息【`fetchOnlyFromLeader`来判断是否必须为`Leader`副本】
  - 根据`readOnlyCommitted`决定`maxOffsetOpt`【`lastStableOffset`是与事务有关的参数】
  - 从`Log`中读取数据并返回【限速处理`shouldLeaderThrottle`】

##  数据同步

### 数据同步线程

- `AbstractFetcherThread`数据同步线程抽象类

  - `addPartitions(Map[TopicAndPartition, Long])`添加需要抓取的`TopicPartition`信息

  - `removePartitions(Set[TopicPartition])`删除指定`TopicPartition`的抓取任务

  - `delayPartitions(Iterable[TopicPartition], Long)`滞后`TopicPartition`抓取任务

  - `doWork()`:线程执行体，构造并同步发送`FetchRequest`，接收并处理`FetchResponse`

    ```reStructuredText
    1.buildFetchRequest创建数据抓取请求
    
    2.processFetchRequest发送并处理请求响应，最终写入副本的Log实例中
        2.1.同步发送请求获取响应【发送fetch请求失败则会退避replica.fetch.backoff.ms时间】
        2.2.结果返回期间可能发生日志截断或分区被删除重加等操作，因此这里只对offset与请求offset一致并且PartitionFetchState的状态为isReadyForFetch进行处理。调用processPartitionData()方法将拉取到的消息追加到本地副本的日志文件中，如果返回结果有错误消息就对相应错误进行相应的处理
        
    3.更新PartitionStates【Follower会根据自身拥有多少个需要同步的TopicPartition来创建对应的PartitionFetchState，这个东西记录了从Leader的哪个Offset开始获取数据】
    ```

- `ReplicaFetcherThread`副本数据同步线程

  - `processPartitionData`处理抓取线程返回的数据

    ```reStructuredText
    1.将获取到的消息集合追加到Log中
    
    2.更新本地副本的HW【本地副本的LEO和返回结果中Leader的HW值取较小值作为本地副本HW】
    
    3.更新本地副本的LSO【若返回结果中Leader的LSO值大于本地副本的LSO则更新本地副本的LSO值】
    ```

  - `handleOffsetOutOfRange`处理`Follower`副本请求的`offset`超过`leader`副本的`offset`范围

    ```reStructuredText
    handleOffsetOutOfRange方法主要处理如下两种情况:
        1.假如当前本地id=1的副本现在是Leader其LEO假设为1000，而另一个在ISR的副本id=2其LEO为800，此时出现网络抖动id=1的机器掉线后又上线，但此时副本的Leader实际上已经变成了id=2的机器，而id=2的机器LEO为800，这时候id=1的机器启动副本同步线程去id=2上机器拉取数据，希望从offset=1000的地方开始拉取，但是id=2的机器最大的offset才是800
    
        2.假设一个Replica(id=1)其LEO是10，它已经掉线好几天了，这个Partition的Leader的Offset范围是[100~800]，那么当id=1的机器重新启动时，它希望从offset=10的地方开始拉取数据，这时候就发生了OutOfRange，不过跟上面不同的是这里是小于Leader的Offset范围
    
    handleOffsetOutOfRange方法针对两种情况提供的解决方案:
        1.如果Leader的LEO小于当前的LEO，则对本地log进行截断(截断到Leader的LEO)
    
        2.如果Leader的LEO大于当前的LEO则说明有有效的数据可以同步，接下来要判断从哪里开始同步。如果Follow宕机比较久后再启动，可能Leader已经对部分日志进行清理，当前副本的LEO小于Leader副本的LSO的情况，当前副本需要截断所有日志并滚动新日志与Leader进行同步
    ```

  - `handlePartitionsWithErrors`将对应分区的同步操作滞后【`PartitionFetchState`置为`delay`】

### 数据同步线程管理器

- `AbstractFetcherManager`数据同步线程管理器抽象类

  `fetcherThreadMap KEY:BrokerIdAndFetcherId VALUE:AbstractFetcherThread`:实现消息的拉取是由`AbstractFetcherThread`负责，`fetcherThreadMap `保存了`BrokerIdAndFetcherId`与同步线程关系

  - `addFetcherForPartitions(Map[TopicPartition, BrokerAndInitialOffset])`添加副本同步任务

    ```reStructuredText
    1.计算出Topic-Partition对应的fetcher id
    
    2.根据BrokerIdAndFetcherId获取对应的replica fetcher线程(若无调用createFetcherThread创建新的fetcher线程)，将TopicPartition记录到FetcherThreadMap中
    ```

  - `removeFetcherForPartitions(Set[TopicPartition])`停止指定副本同步任务

    ```reStructuredText
    1.调用FetcherThread.removePartitions删除指定的Fetch任务
    
    2.若Fetcher线程不再为任何Follower副本进行同步将在shutdownIdleFetcherThreads中被停止
    ```

  - `shutdownIdleFetcherThreads()`某些同步线程负责同步的`TopicPartition`数量为0则停止该线程

  - `closeAllFetchers()`停止所有抓取线程同时清空`fetcherThreadMap `，清空统计数据

  - `createFetcherThread(fetcherId: Int, sourceBroker: BrokerEndPoint)`创建同步线程

- `ReplicaFetcherManager`副本数据同步线程管理器实现类 - > `ReplicaFetcherThread`

  `ReplicaFetcherManager`继承`AbstractFetcherManager`实现了抽象方法`createFetcherThread`

##  ReplicaManager副本管理器

- 关于`ReplicaManager`

  ```reStructuredText
  1.LogManager(负责管理本节点上所有的日志(Log)实例)是ReplicaManager的成员变量，ReplicaManager通过LogManager可以对相应的日志实例进行操作
  
  2.在ReplicaManager中有一个变量allPartitions负责管理本节点所有的Partition实例
  
  3.在创建Partition实例时会将ReplicaManager作为成员变量传入Partition实例中，因此Partition可以通过ReplicaManager获得LogManager实例、BrokerId等
  
  4.Partition会为它的每一个副本创建一个Replica对象实例，但只会为那个在本地副本创建Log对象实例
  
  ReplicaManager通过对Partition对象的管理【ReplicaManager.allPartitions】控制Partition对应的Replica实例【Partition.allReplicasMap】，Replica实例通过Log对象实例管理着其底层的存储内容
  ```

- `startup()` `ReplicaManager`的启动

  在`KafkaServer`启动时会初始化`ReplicaManager`并启动，`ReplicaManager`启动主要初始化相应定时任务

  - `maybeShrinkIsr`周期性检查`ISR`中是否有`Replica`过期需要从`ISR`中移除

  - `maybePropagateIsrChanges`周期性检查`TopicPartition`的`ISR`是否需要改动，如果需要更新到`ZK`上

    ```reStructuredText
    1.isrChangeSet集合不为空
    
    2.最后一次ISR集合发生变化的时间距今已超过5秒
    
    3.上次写入Zookeeper时间距今已超过60秒
    ```

  - `shutdownIdleReplicaAlterLogDirsThread`停止空闲的副本日志目录转移抓取线程

- `stopReplicas`关闭副本

  ```reStructuredText
  1.停止对指定分区的Fetch操作
  
  2.调用stopReplica()关闭指定的分区副本【判断是否需要删除Log】
  ```

- `makeFollower`将指定分区的`Local Replica`切换为`Follower`

  ```reStructuredText
  1.调用partition的makeFollower将Local Replica切换为Follower，后面就不会接受这个partition的Produce请求了，如果有Client再向这台Broker发送数据会返回相应的错误
  
  2.停止对这些partition的副本同步(如果本地副本之前是Follower现在还是Follower，先关闭的原因是:这些Partition的Leader发生了变化)，这样可以保证本地副本将不会有新的数据追加
  
  3.清空时间轮中的produce和fetch请求【tryCompleteDelayedProduce、tryCompleteDelayedFetch】
  
  4.若Borker没有掉线则向这些Partition的新Leader启动副本同步线程【不一定每个Partition数据同步都会启动一个Fetch线程，对于一个Broker只会启动num.replica.fetchers个线程。这个Topic-Partition会分配到哪个fetcher线程上是根据topic名和partition id进行计算得到的】
  ```

- `makeLeaders`将指定分区的`Local Replica`切换为`Leader`

  ```reStructuredText
  1.先停止对这些Partition的副本同步流程，因为这些Partition的本地副本以及被选举为Leader
  
  2.将这些Partition本地副本设置为Leader，并且开始更新对应的Meta信息(记录与其他Follower相关信息)
  ```

- `becomeLeaderOrFollower`更新副本为`Leader/Follow`副本

  ```reStructuredText
  1.添加定时任务checkpointHighWatermarks将HW的值刷新到replication-offset-checkpoint中
  ```

- `maybeUpdateMetadataCache`更新集群缓存，并返回需要删除的`Partition`信息

  ```reStructuredText
  1.清空本节点的AliveNodes和AliveBrokers记录，并更新为最新的记录
  
  2.对于要删除的TopicPartition从缓存中删除，并记录下来作为这个方法的返回
  
  3.对于其他的TopicPartition执行UpdateOrCreate操作
  ```

## 附录 - 关于副本机制的小知识

- `Follower`为什么要维护`HW`

  【保护机制】假设只有`Leader`维护了`HW`信息，一旦`Leader`宕机就没有其它`Broker`知道现在`HW`是多少

- `Leader`维护了`ISR`(同步副本集合)，每个`Partition`当前的`Leader`和`ISR`信息会记录在`Zookeeper`中

  - 所有`Follower`都会和`Leader`通信获取最新消息，所以由`Leader`来管理`ISR`最合适
  - `Leader`副本总是包含在`ISR`中，正常情况下只有`ISR`中的副本才有资格被选举为`Leader`

- `OffsetCheckPoint`:用于管理`Offset`相关文件

  - `replication-offset-checkpoint`:以`TopicPartion`为`Key`记录`HW`的值，其文件结构如下

    >第一行: 版本号
    >
    >第二行:当前写入`TopicPartition`的记录个数
    >
    >其他每行格式:`Topic Partition Offset`，如:`topic-test 0 0`

  - `recovery-point-offset-checkpoint`:

- 在`maybeIncrementLeaderHW`为什么需要`lastCaughtUpTimeMs<replicaLagTimeMaxMs`的副本的`LEO`

  ```reStructuredText
  主要是抑制HW增长，使得Follower副本能有机会进入到ISR集合中，举个例子
  
  1.如果开始只有Leader副本在ISR集合中，此时不断有生产者往Leader副本写入数据则HW值会一直增长
  
  2.这时候Follower副本启动了往Leader发送Fetch请求，如果没有lastCaughtUpTimeMs字段，由于Leader副本不断有数据写入且此时ISR集合只有Leader副本，所以Leader的HW会一直增长，这会导致即使Follower拼命Fetch，但是Fetch请求的Offset始终赶不上Leader的HW值，即Follower一直无法加入ISR集合
  
  3.倘若在增长HW时使用lastCaughtUpTimeMs即可抑制HW抑制增长，这样Follower的Fetch请求的Offset就有可能赶上Leader的HW值，即可加入ISR集合中
  ```

  