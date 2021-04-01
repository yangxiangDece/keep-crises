# 一

- RocketMQ如何保证生产者消息不丢失？

  - 使用事务，但是事务会降低吞吐量
  - 使用confirm模式，写成功会返回一个ack，失败会返回nack，具体根据业务来处理
  - 事务机制和confirm机制的区别
    - 事务机制是同步的，你提交一个事务之后会阻塞在那里，但是confirm机制是异步的，你发送消息后就可以继续执行流程，如果消息发送成功后，broker会异步回调接口通知这个消息已经收到了

- RocketMQ如何保证broker消息不丢失？

  - 持久化，发送消息将消息的deliveryMode设置为2，将消息持久化到磁盘
  - 持久化可以和生产者的confirm机制配合起来，只有消息持久化到磁盘之后，才会通知生产者ack；这样即使rabbitmq挂了，没有持久化成功，生产者也收不到ack，生产者可以继续重发

- RocketMQ如何保证消费者消息不丢失？

  - 关闭自动ack，手动ack

- Kafka丢失消息场景

  - Kafka某个broker宕机了，重新选举partition的leader时，此时其他的follower刚好还有些数据没有同步，结果此时leader挂了，然后选举某个follower成leader之后，就丢失数据了

- Kafka如何保证不丢失消息？

  - 给topic设置replication.factor参数，这个值必须大于1，要求每个partition必须有至少2个副本
  - 在Kafka服务端设置min.insync.replicas参数，这个值必须大于1，要求leader至少感知有至少一个follower还跟自己保持联系，没掉队，这样才能确保leader挂了还有一个follower
  - producer端设置acks=all 必须是写入所有replica之后，才能认为写成功了
  - producer端设置retries=max 一旦写入是不，无限重试，卡在这里了

- 如何保证消息的顺序性？

  - rabbitmq
    - 拆分多个queue，每个queue一个consumer，queue里面的消息是按顺序来的；一个queue对应一个consumer，然后这个consumer内部用内存对了排队，然后分发给底层不同的worker来处理
  - Kafka
    - 一个topic，一个partition，一个consumer，内部单线程消费，写N个内存queue，然后N个线程分别消费一个内存queue
    - 一个partition中的数据是有顺序的，生产者在写的时候，可以指定一个key，比如说你指定某个订单id作为key，这个订单的相关数据，一定会被分发到一个partition中去，而且这个partition中的数据一定是有顺序的

-  如何解决消息队列的延时以及过期失效问题？

  - rabbitmq可以设置过期时间，就是TTL，如果消息再queue中积压超过一定时间就会被rabbitmq给清理掉，但是不建议设置过期时间
  - 如果消息丢弃了，只能写程序将丢失的数据一点点查出来，然后重新灌入mq里面，补数据

- 消息队列满了以后该怎么处理？

  - 写临时程序，接入数据来消费，消费一个丢弃一个，快速消费掉所有消息，然后再补已经丢失的数据

- 有几百万消息持续堆积几个小时，如何处理？

  - 先修复consumer的问题，确保其消费速度，然后将现有的consumer都停掉
  - 临时建立好原先10倍或者20倍的queue数量
  - 然后写一个临时的分发数据的consumer程序，这个程序部署上去消费积压的数据，消费之后不做耗时处理，直接均匀轮询写入临时建立好的10倍数量的queue
  - 接着临时征用10倍的机器来部署consumer，每一批consumer消费一个临时queue的数据
  - 这种做法相当于是临时将queue资源和consumer资源扩大10倍，以政策的10倍速度消费数据
  - 等快速消费积压数据之后，得恢复原形部署架构，重新用原来的consumer及其来消费信息

- 如果让你来设计一个消息队列？

  - 首先mq支持可伸缩性，需要的时候可以快速扩容，可以增加吞吐量，分布式系统，参照Kafka的设计理念，broker->topic->partition，每个partition放一个机器，就存一部分数据，扩容时给topic增加partition，然后做数据迁移，增加机器
  - mq的数据落盘问题，落盘才能保证数据完整性，顺序写
  - mq的高可用，多副本->leader & follower -> broker挂了重新选举leader

- redis线程模型

  - 文件事件处理器
    - redis基于reactor模式开发了网络事件处理器，文件事件处理器，file event handler，这个处理器是单线程的，采用IO多路复用机制同时监听多个socket，根据socket上的事件来选中对应的事件处理器来处理事件
    - 如果被监听的socket准备好了执行accept、read、write、close等操作的时候，跟操作对应的文件事件就会产生，这个时候事件处理器就会调用之前关联好的事件处理器来处理这个事件
    - 文件事件处理器是单线程，但是通过IO多路复用机制监听了多个socket,可以实现高性能的网络通信模型
    - 文件事件处理器的结构包含四个部分：多个socket，IO多路复用程序，文件事件分派器，事件处理器(命令请求处理器、命令回复处理器、连接应答处理器)
    - 多个socket可能并发的产生不同的操作，每个操作对应不同的事件，但是IO多路复用会监听多个socket，会将socket放入一个队列中排队，每次从队列中取出一个socket给事件分派器，事件分派器把socket给对应的事件处理器
    - 事件分派器会根据每个socket当前产生的事件(accept、read、write、close)，来选择对应的事件处理器

- 为什么redis单线程还这么快？

  - 纯内存操作
  - 核心是基于非阻塞的IO多路复用机制
  - 单线程反而避免了多线程的频繁切换问题

- redis过期策略

  - 定期删除+惰性删除
  - 定期删除是redis默认每隔100ms随机抽取一些过期时间的key，检查过期并删除
  - 如果定期删除遗漏了很多过期key，并且也没有惰性删除，内存会有大量过期的key，导致redis内存耗尽
    - 内存淘汰机制

- 手写一个LRU算法

  ```java
  public class LRUCache<K,V> extends LinkedHashMap<K,V> {
    	private final int CACHE_SIZE;
    	// cacheSize表示最多可以换成多少条数据
    	public LRUCache(int cacheSize) {
        // 设置一个LinkedHashMap的初始大小，true表示让LinkedHashMap按照访问顺序进行排序，
        // 最近访问的放在前面，最早访问的放在后面
        super((int) Math.ceil(cacheSize / 0.75) + 1, 0.75f, true);
        CACHE_SIZE = cacheSizel;
      }
    
    	@Override
    	protected boolean removeEldestEntry(Map.Entry eldest) {
        	// 当map中的数据量大于指定的缓存个数的时候，就自动删除
        	return size() > CACHE_SIZE;
      }
  ```

- redis replication的核心机制

  - redis采用异步方式复制数据到slave节点，redis2.8以后开始，slave node会周期地确认自己每次复制的数据量
  - 一个master node是可以配置多个slave node的
  - slave node也可以连接其他slave node
  - slave node做复制的时候，是不会block master node的正常操作
  - slave node做复制的时候，也不会block对自己的查询操作，他会用旧的数据集来提供服务，但是复制完成的时候，需要删除旧数据集，加载新数据集，这时候就会暂停对外的服务了
  - slave node 主要用来进行横向扩容，做读写分离，扩容的slave node可以提高吞吐量
  - 采用主从复制，必须开启master node的持久化

- redis完整主从复制流程

- redis主从架构原理

  - 当启动一个slave node的时候，会发送一个PSYNC命令给master node，如果这是slave node重新连接master，那么master node仅仅只会复制slave部分缺少的数据；如果是slave node第一次连接maste node，那么就会触发一次full resynchronization
  - 开始full resynchronization的时候，master会启动一个后台线程，开始生成一份RDB快照文件，同时还会将从客户端收到的所有命令缓存在内存中，RDB文件生成完毕之后，master会将这个RDB发送给slave，slave会先写入本地磁盘，然后再从本地磁盘加载到内存中，然后master会将内存中缓存的写命令发送给slave，slave也会同步这些数据
  - slave node如果跟master node有网络故障，断开了连接，会自动重连，master如果发现有多个slave node都来重新连接，仅仅会启动一个rdb save操作，用一份数据服务所有slave node

- redis主从复制断点续传

  - 从redis2.8 开始，就支持主从复制的断点续传，如果主从复制过程中，网络连接断开了，重新连接后会接着上次复制的地方，继续复制下去，而不是从头开始复制一份
  - master node会在内存中创建一个backlog，master和slave都会保存一个replica offset，还有一个master id，offset 就是保存在backlog中，如果master和slave网络断开重新连接后，slave会让master从上次的replica offset开始继续复制
  - 如果没有找到对应的offset，那么就会进行full resynchronization
  - master会在自身不断累加offset，slave也会不断在自身累加offset，slave每秒会上报自己的offset给master，同时master也会记录每个slave的offset

- redis无磁盘复制

  - master在内存中直接创建rdb，然后发送给slave，不会在自己本地落地磁盘了
  - repl-diskless-sync
  - repl-diskless-sync-delay  等待多久可以开始复制，默认5秒，需要等待slave重新连接过来

- redis过期key

  - slave不会过期key，只会等待master过期key，如果master过期了一个key，或者通过LRU算法淘汰了一个key，那么会模拟一条del命令发送给slave

- redis高可用（故障转移failover，主备切换）

- redis哨兵模式

  - 集群监控，负责监控redis master和slave进程是否正常工作
  - 消息通知，如果某个redis实例有故障，那么哨兵模式负责发送消息作为报警通知管理员
  - 故障转移，如果master node挂掉了，会自动转移到slave node上
  - 配置中心，如果故障转移发送了，通知client客户端新的master地址
  - 哨兵至少需要3个实例，来保证自己的健壮性
  - 哨兵+redis主从的部署架构，是不会保证数据零丢失的，只能保证redis集群的高可用

- 为什么redis哨兵模式2个节点无法正常工作？

  - 哨兵2个节点，如果挂掉了1个，只剩下1个哨兵，是没有办法故障转移的

- redis主从复制数据丢失情况

  - 主备切换过程中，可能造成数据丢失；
    - 因为master - slave 的复制是异步的，所以有可能有部分数据还没复制到slave，master就宕机了，此时这部分数据丢失了
  - 脑裂数据丢失
    - 某个master的网络出现异常突然断开了和其他slave的连接，但是实际上master并没有挂掉，此时哨兵可能就会认为master宕机了，然后开启选举，将其他slave切换成了master，此时集群里面就出现了两个master，这种情况叫做脑裂
    - 此时虽然某个slave被切换成了master，但是可能client端还没来得及切换到新的master，还继续向旧的master写数据，当旧的master再次恢复时，会被作为一个slave挂到新的master，自己的数据被清空，重新从新的master复制数据
  - 解决方案
    - min-slave-to-write 1  减少异步复制的数据丢失
      - 要求至少有1个slave，数据复制和同步的延迟不超过10秒
      - 一旦slave复制数据和ack延时太长，就认为可能master宕机后损失的数据太多了，那么就拒绝写请求，这样可以把master宕机时由于部分数据未同步到slave导致的数据丢失降低在可控范围内
    - min-slave-max-lag 10  减少脑裂的数据丢失
      - 如果一个master出现了脑裂，跟其他的slave丢了连接，上面两个配置可以保证，如果不能继续给指定数量的slave发送数据，而且slave超过10秒没有给自己ack消息，那么就直接拒绝客户端的写请求，这样脑裂后的旧master就不会接收client的新数据，也就避免了数据丢失；
      - 脑裂情况下，最多丢失10秒的数据
    - 当master不接受新的写请求了，client怎么办？
      - client会做降级，写到本地磁盘，client对外接收的请求再做降级，做限流，减慢请求涌入的速度
      - client将请求写入Kafka消息队列，每隔10分钟去队列里取一次，尝试重新发回master

- redis主观宕机和客观宕机

  - 主观宕机：如果一个哨兵觉得一个master挂掉了，这叫主观宕机
    - 如果一个哨兵ping一个master，超过了is-master-down-after-millisecods指定的毫秒数之后
  - 客观宕机：如果quorum数量的哨兵都觉得这个master挂掉了，这叫客观宕机

- redis哨兵和slave集群的自动发现机制

  - 哨兵相互之间的发现，是通过redis的pub/sub系统实现的，通过消费消息相互感知彼此的存在；
  - 每个哨兵都会往_sentinel_:hello 这个channel发送一条消息，内容是自己的host、ip和runid还有对这个master的监控配置，其他哨兵消费这个消息就可以感知到这个哨兵的存在
  - 每个哨兵还会跟其他哨兵交换master的监控配置，互相进行监控配置的同步

- redis slave 的选举

  - 选举slave会考虑slave的信息
    - 跟master断开连接的时长
    - slave优先级
    - 复制offset
    - run id
  - 如果一个slave跟master断开连接已经超过了down-after-millisecods的10倍，外加master宕机的时长，那么slave就被认为不适合选举master  (down-after-milliseconds * 10) + millisecods_since_master_is_in_SDOWN_state
  - 对slave进行排序
    - 按照slave的优先级进行排序，slave priority 越低，优先级越高
    - 如果slave priority相同，那么看replica offset，哪个slave复制了越多的数据，offset越靠后，优先级越高
    - 如果上面两个条件相同，那么选择一个run id比较小的那个slave，id越小表示越早启动，数据越完整

- redis quorum和majority

  - 每次一个哨兵要做主备切换，首先需要quorum数量的哨兵认为主观宕机，然后选举出一个哨兵来做切换，这个哨兵还得得到majority哨兵的授权，才能正式执行切换
  - 如果quorum < majority，比如5个哨兵，majority是3，quorum设置为2，那么就3个哨兵授权可以切换
  - 如果quorum >= majority，那么必须quorum数量的哨兵都授权，比如5个哨兵，quorum是5，那么必须5个哨兵都同意授权，才能正式执行

- redis configuration epoch

  - 执行切换的那个哨兵，会从要切换到的新master那里得到一个configuration epoch，这就是一个version号，每次切换的version号都必须是唯一的，如果第一个选举出的哨兵切换失败了，那么其他哨兵，会等待failover-timeout时间，然后接着继续执行切换，此时会重新获取一个新的configuration epoch，作为新的version号

- redis configuration传播

  - 哨兵完成切换后，会在自己本地更新生成最新的master配置，然后同步给其他的哨兵，就是通过之前说的pub/sub消息机制
  - 一个哨兵完成一次新的切换后，新的master配置是跟着新的version号的，其他的哨兵都是根据版本号的大小更新自己的master配置

- redis持久化

  - AOF和RDB同时使用
  - 用AOF来保证数据不丢失，作为数据恢复的第一选择
  - 用RDB来做不同程度的冷备，在AOF文件都丢失或损坏不可用的时候，还可以使用RDB来进行快速的数据恢复

- redis cluster 和 replication sentinel

  - replication sentinel，一个master，多个slave，要几个slave跟你要求的吞吐量有关系，搭建sentinel集群，保证redis主从架构高可用
  - redis cluster，主要针对的是海量数据+高并发+高可用场景

- redis cluster 基本原理

  - cluster 自动将数据分片，每个master上放一部分数据；提供内置的高可用机制，部分master不可用时，还是可以继续工作
  - 在redis cluster集群架构下，每个redis要开放两个端口，比如一个是6379，另外一个就是加10000的端口号，即16379，这个端口是给集群之间通信使用的，即cluster bus，集群总线，用来故障检测，配置更新，故障转移授权，cluster bus用了另一种二进制协议，主要用于节点间进行高效的数据交换，占用更少的网络带宽和处理时间
  - 一致性hash算法（自动缓存迁移）+ 虚拟节点（自动负载均衡）
  - redis cluster的hash slot算法
    - redis cluster 有固定的16384个hash slot，对每个key计算CRC16值，然后对16384取模，可以获取key对应的hash slot
    - redis cluster 中每个master持有部分slot，比如有3个master，那么可能每个master持有5000多个hash slot
    - hash slot 让node的增加和移除很简单，增加一个master，就将其他的master的hash slot移动部分过去，减少一个master，就将它的hash slot移动到其他的master上去，移动的hash solt 的成本是非常低的
    - 客户端的api， 可以对指定的数据，让他们走同一个hash slot，通过hash tag 来实现

- 为什么删除缓存，而不是更新缓存？

  - 因为缓存有时候并不是只查询数据库就可以实现的，可能需要查询另外两个表的数据，然后进行计算才能计算出缓存的最新值，如果是更新缓存，代价是非常大的，如果频繁地修改一个缓存，那么缓存也会被频繁的更新，但是这个缓存不并一定会访问到，所以还是删除以后，等真正访问的时候再去同步更新缓存数据

- 缓存与数据库双写不一致的情况，如何处理？

  - 更新数据的时候，根据数据的唯一标识，将操作路由到一个jvm内存队列里面，读取数据的时候，如果发现数据不再缓存中，那么将重新读取数据+更新缓存的操作，根据唯一标识，也发送到刚刚的jvm内存队列里面，一个队列对应一个线程
  - 每个线程串行拿到对应的操作，一个一个地执行，这样，一个数据变更操作，先执行，删除缓存，然后再去更新数据库，但是还没有更新完成，此时如果来了一个读请求，读到了空的请求，那么可以先将缓存更新的请求发送到队列中，此时会在队列中积压，然后同步等待缓存更新完成，这里有一个优化点，一个队列中，其实多个更新缓存请求串在一起是没有意义的，因此可以过滤，如果发现队列中已经有一个更新缓存的请求了，那么就不用再放这个更新请求操作了，直接等待前面的更新操作更新完成即可
  - 待那个队列对应的工作线程完成了上一个操作的数据库的修改之后，才会去执行下一个操作，也就是缓存更新的操作，此时会从数据库中读取最新的值，然后写入缓存中，如果请求还在等待时间范围内，不断轮询发现可以取到值了，那么直接返回，如果请求等待的时间超过一定时长，那么这一次直接从数据库中读取当前的旧值
  - 读请求超时阻塞
    - 由于读请求也加入了内存队列，所以每个读请求必须在超时时间范围内返回
    - 最大风险点：可能数据更新很频繁，导致队列中积压了大量更新操作在里面，然后读请求发生大量的超时，最后导致大量的请求直接走数据库，务必通过一些真实模拟测试，看看业务数据频繁程度如何
    - 如果一个内存队列可能积压的更新操作非常特别多，那么就要加机器，让每个机器上部署的服务实例来分摊请求，那么每个内存队列中积压的更新操作就会越少
    - 根据实际项目经验：一般来说数据的写频率是很低的，因此实际上正常来说，在队列中积压的更新操作应该很少的，针对高并发，读缓存架构的项目，一般写请求相对读请求，是非常少的，每秒的qps能到几百就不错了
    - 一秒，500个写操作，5份，没200ms，就100个写操作，单机器，20个内存队列，每个内存队列，可能就积压5个写操作，每个写操作性能测试，一般20ms左右就完成，那么针对每个内存队列中的数据的读请求，也就是最多hang一会儿，200ms以内肯定就能返回了
    - 而且写操作还做了去重操作，所以也就是一个更新缓存的操作在后面，数据更新完成后，读请求触发更新操作也完成，然后临时等待的读请求全部可以读到缓存中的数据
  - 读请求并发了过高
    - 还有一个风险：就是突然间大量读请求会在几十毫秒的延时hang在服务器上，看服务能不能抗住，需要多少机器才能抗住最大的极限情况峰值
    - 但是因为不是所有数据都是在同一时间更新， 缓存也不会在同一时间失效，所以每次可能也就是少量数据的缓存失效了，然后那些数据对应的请求过来，并发量应该也不是很大
    - 按1:99的比列算计算读和写请求，每秒5万的qps，可能只有500次更新操作

- redis并发竞争的问题？如何解决？了解redis事务的CAS方案吗？

  - 就是多客户端同时并发写一个key，可能本来应该先到的数据后到了，导致数据版本错了，或者是多客户端同时获取一个key，修改值之后再写回去，只要顺序错了，数据就错了，redis自己就有天然解决这个问题的CAS类的乐观锁方案
  - 确保同一时间只能有一个系统实例在操作某个key，别人不允许操作
  - 每次要写时间，先判断一下当前这个key的时间戳是否比缓存里的value的时间戳更新，如果更新就可以写，否则不写
  - 即分布式锁+时间戳
  - 在并发量过大的情况下,可以通过消息中间件进行处理,把并行读写进行串行化，把Redis.set操作放在队列中使其串行化,必须的一个一个执行

- 生产环境的redis部署结构？用了哪种集群？有没有做高可用保证？有没有开启持久化机制可以进行数据恢复？线上redis分配几个G的内存？设置了哪些参数？压测后你们redis集群承载多少QPS？

  - redis cluster, 10台机器，5台机器部署了redis主实例，另外5台部署了从实例，每个主实例挂了一个从实例，5个节点对外提供读写服务，每个节点的读写高峰qps可以达到每秒5万，5台机器最多是25万读写请求/s
  - 机器配置，32G+8核CPU+1T硬盘，但是分配给redis的是10g内存，一般生产环境，redis的内存尽量不要超过10G
  - 高可用，任何一台主实例宕机，都会自动故障迁移

- zookeeper分布式锁

- 分库分表

  - sharding-jdbc
  - mycat

- 系统如何不停机迁移到分库分表？

  - 双写，写老库和新库
  - 然后将老库的历史数据通过工具代码导入到新库
  - 直到两边库数据一致

- 分库分表动态扩容方案

- 分库分表后全局id

  - Twitter开源的snowflake 雪花算法，就是把一个64位的long类型的id，1个bit是不用的，用其中的41bit作为毫秒数，用10bit作为工作机器id，12bit作为序列号
  - 1bit：不用，因为二级制里第一个bit如果为1，那么都是负数，但是我们生成的id都是整数，所以第一个bit统一都为0才行
  - 41bit：表示的是时间戳，单位毫秒，41bit可以表示的数字多达2^41-1 ，也就是可以表示2^41-1个毫秒值，换算成年就是表示69年的时间
  - 10bit：记录工作机器id，代表的是这个服务最多可以部署在2^10台机器上，也就是1024台机器，但是10bit里5bit代表机房id，5bit代表机器id，最多代表2^5个机房（32个机房），每个机房可以代表2^5台机器（32台机器）
  - 12bit：这个是用来记录同一个毫秒内产生的不同id，12bit可以代表的最大正数是2^12-1=4096，也就是说用这个12bit代表的数字来区分同一毫秒内的4096个不同的id

- mysql读写分离？

  - 基于主从架构读写分离，只写主库，读取都去从库读取

- mysql主从复制原理？

  - 主库将变更写入binlog日志，然后从库连接到主库之后，从库有一个IO线程，将主库的binlog日志拷贝到自己本地，写入中继日志中，然后从库有一个SQL线程会从中级日志中读取binlog，然后执行binlog日志中的内容，也就是在自己本地在执行一遍SQL，这样就可以保证自己跟主库的数据一致
  - 从库同步主库数据的过程是串行化的，即主库上并行的操作，在从库上会串行执行，所以在高并发场景下，从库的数据一定会比主库慢一些，是有延时的，随意经常出现，刚写入主库的数据可能读取不到，要过几十几百毫秒才能读取到
  - 如果主库突然宕机了，然后恰好数据还没有同步到从库，那么有些数据可能在从库就没有，有些数据可能就丢失了

- 如何解决mysql主从同步的延时问题？

  - mysql提供了半同步机制解决主库数据丢失问题，并行复制解决主从同步延时问题
  - 半同步机制
    - 主库写入binlog日志后，就会强制此时立即将数据同步到从库，从库日志写入自己本地的reply log之后，接着会返回一个ack给主库，主库接收到至少一个从库的ack之后才认为写操作完成了
  - 并行复制机制
    - 从库开启多线程，并行读取replay log中不同库的日志，然后并行执行不同库的日志，这是库级别的并行
  - show status，Seconds_Behind_Master，可以看到从库复制主库落后了几秒
  - 主从复制要根据实际情况场景，建议一般是在读远远多于写，而且读的实时性要求不高；
  - 如果要求实时性比较高，可以直接强制读取主库，可以通过数据库中间件来实现

- 如何设计一个高可用系统？

  - 限流
  - 熔断
    - 系统后端一些依赖，出了一些故障，比如mysql挂了，每次请求都是报错，熔断了，后续的请求就拒绝访问，10分钟后再看mysql恢复情况
  - 降级
  - 运维监控
  - 资源隔离
    - 让系统里面，某一块东西，在故障的情况下，不会耗尽系统所有资源，比如线程资源

- hystrix的设计原则？

  - 对依赖服务调用时出现的延迟和失败进行控制和容错保护
  - 在复杂的分布式系统中，阻止某个依赖服务的故障在整个系统蔓延
  - 提供fail-fast快速失败和快速恢复
  - 提供fallback优雅降级的支持
  - 支持近实时的监控、报警以及运维操作

- hystrix是如何实现这些原则的？

  - 通过HystrixCommand或者HystrixObservableCommand来封装对外部依赖的访问请求，这个访问请求一般会运行在独立的线程中
  - 对于超出我们设定阈值的服务调用，直接进行超时处理，不允许其耗时过长时间阻塞系统
  - 为每一个依赖服务维护一个线程池（船舱隔离技术），或者是semaphore，当线程池已满时，直接拒绝对这个服务的调用
  - 对依赖服务的调用次数、失败次数、拒绝次数，超时次数，进行统计
  - 如果对一个依赖服务的调用次数超过了一定的阈值，自动进行熔断，在一定时间内对服务的调用直接降级，一段时间后再自动尝试恢复
  - 当一个服务调用出现失败，被拒绝，超时，短路等异常情况时，自动调用fallback降级机制
  - 对属性和配置的修改提供近实时的支持

- ZK分布式锁羊群效应

# 真

- CountDownLatch和CyclicBarrier的区别？

  - CountDownLatch：一个线程(或者多个)， 等待另外N个线程完成某个事情之后才能执行

  - CyclicBarrier：N个线程相互等待，任何一个线程完成之前，所有的线程都必须等待

  - 对于CountDownLatch来说，重点是那个“一个线程”, 是它在等待， 而另外那N的线程在把“某个事情”做完之后可以继续等待，可以终止。而对于CyclicBarrier来说，重点是那N个线程，他们之间任何一个没有完成，所有的线程都必须等待

  - 从字面上理解，CountDown表示减法计数，Latch表示门闩的意思，计数为0的时候就可以打开门闩了。Cyclic Barrier表示循环的障碍物。两个类都含有这一个意思：对应的线程都完成工作之后再进行下一步动作，也就是大家都准备好之后再进行下一步。然而两者最大的区别是，进行下一步动作的动作实施者是不一样的。这里的“动作实施者”有两种，一种是主线程（即执行main函数），另一种是执行任务的其他线程，后面叫这种线程为“其他线程”，区分于主线程。对于CountDownLatch，当计数为0的时候，下一步的动作实施者是main函数；对于CyclicBarrier，下一步动作实施者是“其他线程”

  - ```java
    import java.util.Random;
    import java.util.concurrent.CountDownLatch;
    
    public class CountDownLatchTest {
    
        public static void main(String[] args) throws InterruptedException {
            CountDownLatch latch = new CountDownLatch(4);
            for(int i = 0; i < latch.getCount(); i++){
                new Thread(new MyThread(latch), "player"+i).start();
            }
            System.out.println("正在等待所有玩家准备好");
            latch.await();
            System.out.println("开始游戏");
        }
    
        private static class MyThread implements Runnable{
            private CountDownLatch latch ;
    
            public MyThread(CountDownLatch latch){
                this.latch = latch;
            }
    
            @Override
            public void run() {
                try {
                    Random rand = new Random();
                    int randomNum = rand.nextInt((3000 - 1000) + 1) + 1000;//产生1000到3000之间的随机整数
                    Thread.sleep(randomNum);
                    System.out.println(Thread.currentThread().getName()+" 已经准备好了, 所使用的时间为 "+((double)randomNum/1000)+"s");
                    latch.countDown();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
    
            }
        }
    }
    ```

  - 对于CyclicBarrier，假设有一家公司要全体员工进行团建活动，活动内容为翻越三个障碍物，每一个人翻越障碍物所用的时间是不一样的。但是公司要求所有人在翻越当前障碍物之后再开始翻越下一个障碍物，也就是所有人翻越第一个障碍物之后，才开始翻越第二个，以此类推。类比地，每一个员工都是一个“其他线程”。当所有人都翻越的所有的障碍物之后，程序才结束。而主线程可能早就结束了，这里我们不用管主线程

  - ```java
    import java.util.Random;
    import java.util.concurrent.BrokenBarrierException;
    import java.util.concurrent.CyclicBarrier;
    
    public class CyclicBarrierTest {
        public static void main(String[] args) {
            CyclicBarrier barrier = new CyclicBarrier(3);
            for(int i = 0; i < barrier.getParties(); i++){
                new Thread(new MyRunnable(barrier), "队友"+i).start();
            }
            System.out.println("main function is finished.");
        }
    
    
        private static class MyRunnable implements Runnable{
            private CyclicBarrier barrier;
    
            public MyRunnable(CyclicBarrier barrier){
                this.barrier = barrier;
            }
    
            @Override
            public void run() {
                for(int i = 0; i < 3; i++) {
                    try {
                        Random rand = new Random();
                        int randomNum = rand.nextInt((3000 - 1000) + 1) + 1000;//产生1000到3000之间的随机整数
                        Thread.sleep(randomNum);
                        System.out.println(Thread.currentThread().getName() + ", 通过了第"+i+"个障碍物, 使用了 "+((double)randomNum/1000)+"s");
                        this.barrier.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } catch (BrokenBarrierException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
    ```

  - 总结：CountDownLatch和CyclicBarrier都有让多个线程等待同步然后再开始下一步动作的意思，但是CountDownLatch的下一步的动作实施者是主线程，具有不可重复性；而CyclicBarrier的下一步动作实施者还是“其他线程”本身，具有往复多次实施动作的特点

- Semaphore？

  - 在信号量上我们定义两种操作： acquire（获取） 和 release（释放）。当一个线程调用acquire操作时，它要么通过成功获取信号量（信号量减1），要么一直等下去，直到有线程释放信号量，或超时。release（释放）实际上会将信号量的值加1，然后唤醒等待的线程
  - 信号量主要用于两个目的，一个是用于多个共享资源的互斥使用，另一个用于并发线程数的控制

- volatile 总线风暴？

  - MESI（缓存一致性协议）
    - 当CPU写数据时，如果发现操作的变量是共享变量，即在其他CPU中也存在该变量的副本，会发出信号通知其他CPU将该变量的缓存行置为无效状态，因此当其他CPU需要读取这个变量时，发现自己缓存中缓存该变量的缓存行是无效的，那么它就会从内存重新读取
  - 嗅探
    - 每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器对这个数据进行修改操作的时候，会重新从系统内存中把数据读到处理器缓存里
  - 由于Volatile的MESI缓存一致性协议，需要不断的从主内存嗅探和cas不断循环，无效交互会导致总线带宽达到峰值
  - 解决办法：部分volatile和cas使用synchronize

# Java基础

## 集合

### HashMap

- 1.7
  
  - 数组 + 链表、扩容时头插法
  
- 1.8
  
  - 数组 + 链表 + 红黑树、扩容时采用 尾插法
  - 当链表的深度达到8的时候，也就是默认阈值，就会自动扩容把链表转成红黑树的数据结构来把时间复杂度从O（n）变成O（logN）提高了效率
  
- JDK1.7用的是头插法，而JDK1.8及之后使用的都是尾插法，那么他们为什么要这样做呢？
  
  - 因为JDK1.7是用单链表进行的纵向延伸，当采用头插法时会容易出现逆序且环形链表死循环问题。但是在JDK1.8之后是因为加入了红黑树使用尾插法，能够避免出现逆序且链表死循环的问题.
  - 尾插法还会保持元素原本的顺序
  
- 为什么HashMap的容量总是2的n次幂？

  - 关键代码p = tab[i = (n - 1) & hash]，n是一定是2的幂次方，2的幂次方换算成二进制，高位一定是1，然后再减去1，就变成除了低位第一位为0，其他全部为1，所以当hash的二进制和（n-1）的二进制进行位与运算的时候，hash的二进制任何一位变成0或1，那么最终得到的值是不同的，这样扩大了数组散列性
  - 2的幂次方：方便位运算、数据均匀分布
  - 如果设置的不是2的幂次方，HashMap会计算出与该数最接近的数字

- 扩容机制
  - ```java
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    ```
  
  - 扩容大小为原数组的2倍
  
  - 为什么在JDK1.8中进行对HashMap优化的时候，把链表转化为红黑树的阈值是8？
  
    - 数学 —— 泊松分布
    - 桶的长度超过8的概率非常非常小。所以作者应该是根据概率统计而选择了8作为阀值
  
  - 1.7是先扩容再进行插入
  
    - 当你发现你插入的桶是不是为空，如果不为空说明存在值就发生了hash冲突，那么就必须得扩容，但是如果不发生Hash冲突的话，说明当前桶是空的（后面并没有挂有链表），那就等到下一次发生Hash冲突的时候在进行扩容，如果以后都没有发生hash冲突产生，那么就不会进行扩容了，减少了一次无用扩容，也减少了内存的使用
    
  - load_factor 负载因子越大
  
    - 优点：空间利用率高
    - 缺点：Hash冲突概率加大、链表变长、查找效率变低
  
  - load_factor 负载因子越小
  
    - 优点：Hash冲突概率小、链表短、查找效率高
    - 缺点：空间利用率低、频繁扩容耗费性能
  
- 并发情况下的问题
  - 插入的元素可能被覆盖
  - put的时候链表可能形成环形数据结构，会形成死循环 主要原因还是线程不安全
  
- 为什么 HashMap 中 String、Integer 这样的包装类适合作为 key 键？

  - String、Integer 等是 final 类型 ，不可变，保证了 key 的不可更改性，就保证了 Hash值的不可更改性，不会出现放入时和取出时hash不同的情况
  - 内部已经重写了eqauls()、hashcode() ，不容易出现hash值的计算错误

- 为什么不直接采用经过`hashCode（）`处理的哈希码 作为 存储数组`table`的下标位置？

  - 容易出现哈希码与数组大小范围不匹配的情况，即计算出来的哈希码可能不在数组大小范围内，从而导致无法匹配存储位置
  - HashMap的解决方案：哈希码 & (数组长度 - 1)

- 为什么采用 哈希码 **与运算(&)** （数组长度-1） 计算数组下标？

  - 要想让算出来的hash值在数组范围内，就只能取余，即：h % length；但是取余效率低，这个是操作系统决定的，所以采用位运算 &

- hash计算规则

- ```java
  static final int hash(Object key) {
      int h;
      return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
  }
  ```

- put 源码

- ```java
  final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                     boolean evict) {
          Node<K,V>[] tab; Node<K,V> p; int n, i;
          if ((tab = table) == null || (n = tab.length) == 0)
              n = (tab = resize()).length;
          if ((p = tab[i = (n - 1) & hash]) == null)
              tab[i] = newNode(hash, key, value, null);
          else {
              Node<K,V> e; K k;
              if (p.hash == hash &&
                  ((k = p.key) == key || (key != null && key.equals(k))))
                  e = p;
              else if (p instanceof TreeNode)
                  e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
              else {
                  for (int binCount = 0; ; ++binCount) {
                      if ((e = p.next) == null) {
                          p.next = newNode(hash, key, value, null);
                          if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                              treeifyBin(tab, hash);
                          break;
                      }
                      if (e.hash == hash &&
                          ((k = e.key) == key || (key != null && key.equals(k))))
                          break;
                      p = e;
                  }
              }
              if (e != null) { // existing mapping for key
                  V oldValue = e.value;
                  if (!onlyIfAbsent || oldValue == null)
                      e.value = value;
                  afterNodeAccess(e);
                  return oldValue;
              }
          }
          ++modCount;
          if (++size > threshold)
              resize();
          afterNodeInsertion(evict);
          return null;
      }
  ```

- get 源码

- ```java
  final Node<K,V> getNode(int hash, Object key) {
          Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
          if ((tab = table) != null && (n = tab.length) > 0 &&
              (first = tab[(n - 1) & hash]) != null) {
              if (first.hash == hash && // always check first node
                  ((k = first.key) == key || (key != null && key.equals(k))))
                  return first;
              if ((e = first.next) != null) {
                  if (first instanceof TreeNode)
                      return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                  do {
                      if (e.hash == hash &&
                          ((k = e.key) == key || (key != null && key.equals(k))))
                          return e;
                  } while ((e = e.next) != null);
              }
          }
          return null;
      }
  ```

### ConcurrentHashMap 

- 1.7
  
  - 数组 + 链表
  - segment 分段锁
    - Segment数组的意义就是将一个大的table分割成多个小的table来进行加锁，而每一个Segment元素存储的是多个HashEntry数组+链表，同时Segment继承了ReentrantLock
    - 定位一个元素过程需要进行两次Hash操作，第一次Hash定位Segment，第二次Hash定位元素所在数组位置
    - 坏处：定位Hash过程较长
    - 好处：写操作的时候可以只对元素所在的Segment进行加锁即可，不会影响到其他的Segment，这样，在最理想的情况下，ConcurrentHashMap可以最高同时支持Segment数量大小的写操作
  
- 

- 1.8
  - 数组 + 链表 + 红黑树
  
  - CAS + synchronized — cas失败自旋保证成功 — 再失败就用sync保证
  
  - 在ConcurrentHashMap中通过一个Node<K,V>[]数组来保存添加到map中的键值对，而在同一个数组位置是通过链表和红黑树的形式来保存的。但是这个数组只有在第一次添加元素的时候才会初始化，否则只是初始化一个ConcurrentHashMap对象的话，只是设定了一个sizeCtl变量，这个变量用来判断对象的一些状态和是否需要扩容
  
  - 第一次添加元素的时候，默认初期长度为16，当往map中继续添加元素的时候，通过hash值跟数组长度取与来决定放在数组的哪个位置，如果出现放在同一个位置的时候，优先以链表的形式存放，在同一个位置的个数又达到了8个以上，如果数组的长度还小于64的时候，则会扩容数组。如果数组的长度大于等于64了的话，在会将该节点的链表转换成树
  
  - 在扩容完成之后，如果某个节点的是树，同时现在该节点的个数又小于等于6个了，则会将该树转为链表
  
  - put源码
  
    - ```java
      /*
       * 当添加一对键值对的时候，首先会去判断保存这些键值对的数组是不是初始化了，
       * 如果没有的话就初始化数组，然后通过计算hash值来确定放在数组的哪个位置
       * 如果这个位置为空则直接添加，如果不为空的话，则取出这个节点来
       * 如果取出来的节点的hash值是MOVED(-1)的话，则表示当前正在对这个数组进行扩容，复制到新的数组，则当前线程也去帮助复制
       * 最后一种情况就是，如果这个节点，不为空，也不在扩容，则通过synchronized来加锁，进行添加操作
       * 然后判断当前取出的节点位置存放的是链表还是树
       * 如果是链表的话，则遍历整个链表，直到取出来的节点的key来个要放的key进行比较，如果key相等，并且key的hash值也相等的话，则说明是同一个key，则覆盖掉value，否则的话则添加到链表的末尾
       * 如果是树的话，则调用putTreeVal方法把这个元素添加到树中去
       * 最后在添加完成之后，会判断在该节点处共有多少个节点（注意是添加前的个数），如果达到8个以上了的话，
       * 则调用treeifyBin方法来尝试将处的链表转为树，或者扩容数组
       */
      final V putVal(K key, V value, boolean onlyIfAbsent) {
              if (key == null || value == null) throw new NullPointerException();//K,V都不能为空，否则的话跑出异常
              int hash = spread(key.hashCode());//取得key的hash值
              int binCount = 0;//用来计算在这个节点总共有多少个元素，用来控制扩容或者转移为树
              for (Node<K,V>[] tab = table;;) {
                  Node<K,V> f; int n, i, fh;
                  if (tab == null || (n = tab.length) == 0)    
                      tab = initTable();//第一次put的时候table没有初始化，则初始化table
                  else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {//通过哈希计算出一个表中的位置因为n是数组的长度，所以(n-1)&hash肯定不会出现数组越界
                      if (casTabAt(tab, i, null,//如果这个位置没有元素的话，则通过cas的方式尝试添加，注意这个时候是没有加锁的
                                   new Node<K,V>(hash, key, value, null)))//创建一个Node添加到数组中区，null表示的是下一个节点为空
                          break;                   // no lock when adding to empty bin
                  }
                  /*
                   * 如果检测到某个节点的hash值是MOVED，则表示正在进行数组扩张的数据复制阶段，
                   * 则当前线程也会参与去复制，通过允许多线程复制的功能，一次来减少数组的复制所带来的性能损失
                   */
                  else if ((fh = f.hash) == MOVED)    
                      tab = helpTransfer(tab, f);
                  else {
                      /*
                       * 如果在这个位置有元素的话，就采用synchronized的方式加锁，
                       *     如果是链表的话(hash大于0)，就对这个链表的所有元素进行遍历，
                       *         如果找到了key和key的hash值都一样的节点，则把它的值替换到
                       *         如果没找到的话，则添加在链表的最后面
                       *  否则，是树的话，则调用putTreeVal方法添加到树中去
                       *  
                       *  在添加完之后，会对该节点上关联的的数目进行判断，
                       *  如果在8个以上的话，则会调用treeifyBin方法，来尝试转化为树，或者是扩容
                       */
                      V oldVal = null;
                      synchronized (f) {
                          if (tabAt(tab, i) == f) {//再次取出要存储的位置的元素，跟前面取出来的比较
                              if (fh >= 0) {//取出来的元素的hash值大于0，当转换为树之后，hash值为-2
                                  binCount = 1;            
                                  for (Node<K,V> e = f;; ++binCount) {//遍历这个链表
                                      K ek;
                                      if (e.hash == hash &&//要存的元素的hash，key跟要存储的位置的节点的相同的时候，替换掉该节点的value即可
                                          ((ek = e.key) == key ||
                                           (ek != null && key.equals(ek)))) {
                                          oldVal = e.val;
                                          if (!onlyIfAbsent)//当使用putIfAbsent的时候，只有在这个key没有设置值得时候才设置
                                              e.val = value;
                                          break;
                                      }
                                      Node<K,V> pred = e;
                                      if ((e = e.next) == null) {//如果不是同样的hash，同样的key的时候，则判断该节点的下一个节点是否为空，
                                          pred.next = new Node<K,V>(hash, key,//为空的话把这个要加入的节点设置为当前节点的下一个节点
                                                                    value, null);
                                          break;
                                      }
                                  }
                              }
                              else if (f instanceof TreeBin) {//表示已经转化成红黑树类型了
                                  Node<K,V> p;
                                  binCount = 2;
                                  if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,//调用putTreeVal方法，将该元素添加到树中去
                                                                 value)) != null) {
                                      oldVal = p.val;
                                      if (!onlyIfAbsent)
                                          p.val = value;
                                  }
                              }
                          }
                      }
                      if (binCount != 0) {
                          if (binCount >= TREEIFY_THRESHOLD)//当在同一个节点的数目达到8个的时候，则扩张数组或将给节点的数据转为tree
                              treeifyBin(tab, i);    
                          if (oldVal != null)
                              return oldVal;
                          break;
                      }
                  }
              }
              addCount(1L, binCount);//计数
              return null;
          }
      ```
  
  - get源码
  
    - ```java
      /*
       * 相比put方法，get就很单纯了，支持并发操作，
       * 当key为null的时候回抛出NullPointerException的异常
       * get操作通过首先计算key的hash值来确定该元素放在数组的哪个位置
       * 然后遍历该位置的所有节点
       * 如果不存在的话返回null
       */
      public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
          if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
              return e.val;
          }
          else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
          while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
              return e.val;
          }
        }
        return null;
      }
      ```
  
  - 扩容机制
  
    - ```java
      /**
       * 把数组中的节点复制到新的数组的相同位置，或者移动到扩张部分的相同位置
       * 在这里首先会计算一个步长，表示一个线程处理的数组长度，用来控制对CPU的使用，
       * 每个CPU最少处理16个长度的数组元素,也就是说，如果一个数组的长度只有16，那只有一个线程会对其进行扩容的复制移动操作
       * 扩容的时候会一直遍历，直到复制完所有节点，没处理一个节点的时候会在链表的头部设置一个fwd节点，这样其他线程就会跳过他，
       * 复制后在新数组中的链表不是绝对的反序的
       */
      ```
  
    - 所以引起数组扩容的情况如下：
  
      - 只有在往map中添加元素的时候，在某一个节点的数目已经超过了8个，同时数组的长度又小于64的时候，才会触发数组的扩容
      - 当数组中元素达到了sizeCtl的数量的时候，则会调用transfer方法来进行扩容
  
    - 那么在扩容的时候，可以不可以对数组进行读写操作呢？
  
      - 事实上是可以的。当在进行数组扩容的时候，如果当前节点还没有被处理（也就是说还没有设置为fwd节点），那就可以进行设置操作
      - 如果该节点已经被处理了，则当前线程也会加入到扩容的操作中去
  
    - 那么，多个线程又是如何同步处理的呢？
  
      - 在ConcurrentHashMap中，同步处理主要是通过Synchronized和unsafe两种方式来完成的
      - 在取得sizeCtl、某个位置的Node的时候，使用的都是unsafe的CAS方法，来达到并发安全的目的
      - 当需要在某个位置设置节点的时候，则会通过Synchronized的同步机制来锁定该位置的节点
      - 在数组扩容的时候，则通过处理的步长和fwd节点来达到并发安全的目的，通过设置hash值为MOVED
      - 当把某个位置的节点复制到扩张后的table的时候，也通过Synchronized的同步机制来保证线程安全

### Arrays.asList

- asList 得到的知识一个Arrays 的内部类，一个原来数组的视图List，如果对他进行增删操作会报错
- 用 ArrayList 的构造器可以将其转变为真正的 ArrayList

### 集合 fail-fast 策略

- 判断modCount 是否与 exceptedModCount 相等，若不相等，则表示出现并发有其他线程修改了集合；若相等则未出现并发情况
- 抛出 ConcurrentModificationExcetion 异常

### 如何在遍历的同时删除ArrayList中的元素

- 直接使用普通for循环进行删除；

  - 不能再foreach中进行删除，但是可以使用普通的for循环，因为普通的for雄厚没有用到Iterator的遍历，所以就不会进行 fail-fast的检验

  - ```java
    List<String> userNames = new ArrayList<String>() {{
      add("Hollis");
      add("hollis");
      add("HollisChuang");
      add("H");
    }};
    
    for (int i = 0; i < 1; i++) {
      if (userNames.get(i).equals("Hollis")) {
        userNames.remove(i);
      }
    }
    System.out.println(userNames);
    ```

  - 这种方式存在一个问题，那就是remove操作会改变List中元素的下标，可能存在漏删的问题

- 直接使用Iterator进行删除

  - ```java
    List<String> userNames = new ArrayList<String>() {{
      add("Hollis");
      add("hollis");
      add("HollisChuang");
      add("H");
    }};
    
    Iterator iterator = userNames.iterator();
    
    while (iterator.hasNext()) {
      if (iterator.next().equals("Hollis")) {
        iterator.remove();
      }
    }
    System.out.println(userNames);
    
    ```

  - 使用Iterator提供的remove方法，就可以修稿到expectedModCount的值，那么就不会抛出异常了8中

- 使用Java8中提供的filter过滤生成新的集合

### CopyOnWriteArrayList

- CopyOnWriteArrayList相当于线程安全的ArrayList，CopyOnWriteArrayList使用了一种叫写时复制的方法，当有新元素add到CopyOnWriteArrayList时，先从原有的数组中拷贝一份出来，然后在新的数组做写操作，写完之后，再将原来的数组引用指向到新数组
- 这样做的好处是我们可以对CopyOnWrite容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。所以CopyOnWrite容器也是一种读写分离的思想，读和写不同的容器
- 注意：CopyOnWriteArrayList的整个add操作都是在锁的保护下进行的。也就是说add方法是线程安全的。

## 枚举

- 枚举单例模式

  - ```java
    public enum Singleton {
      INSTANCE;
      public void doSomething() {
         System.out.println("doSomething");
      }
      // 调用方式
      public static void main(String[] args) {
          Singleton.INSTANCE.doSomething();
      }
    }
    ```

  - 通过将定义好的枚举[反编译](http://www.hollischuang.com/archives/58)，其实枚举在经过`javac`的编译之后，会被转换成形如`public final class T extends Enum`的定义，枚举中的各个枚举项是通过 static 来定义的

  - 这个类是 final 类型的，不能被继承

    - ```java
      public enum T {
          SPRING,SUMMER,AUTUMN,WINTER;
      }
      ```

    - ```java
      public final class T extends Enum
      {
          //省略部分内容
          public static final T SPRING;
          public static final T SUMMER;
          public static final T AUTUMN;
          public static final T WINTER;
          private static final T ENUM$VALUES[];
          static
          {
              SPRING = new T("SPRING", 0);
              SUMMER = new T("SUMMER", 1);
              AUTUMN = new T("AUTUMN", 2);
              WINTER = new T("WINTER", 3);
              ENUM$VALUES = (new T[] {
                  SPRING, SUMMER, AUTUMN, WINTER
              });
          }
      }
      ```

  - 当一个Java类第一次被真正使用到的时候静态资源被初始化、Java类的加载和初始化过程都是线程安全的（因为虚拟机在加载枚举的类的时候，会使用ClassLoader的loadClass方法，而这个方法使用同步代码块保证了线程安全）所以，创建一个enum类型是线程安全的

## IO

#### BIO、NIO、AIO

- BIO，同步阻塞式IO，简单理解：一个线程处理一个连接，发起和处理IO请求都是同步的
- NIO，同步非阻塞IO，简单理解：一个线程处理多个连接，发起IO请求是非阻塞的但处理IO请求是同步的
- AIO，异步非阻塞IO，简单理解：一个有效请求一个线程，发起和处理IO请求都是异步的
- BIO里用户最关心“我要读”，NIO里用户最关心"我可以读了"，在AIO模型里用户更需要关注的是“读完了”
- NIO一个重要的特点是：socket主要的读、写、注册和接收函数，在等待就绪阶段都是非阻塞的，真正的I/O操作是同步的（消耗CPU但性能非常高）

#### 阻塞IO 和 非阻塞IO

- 这两个概念是程序级别的。主要描述的是程序请求操作系统IO操作后，如果IO资源没有准备好，那么程序该如何处理的问题：前者等待；后者继续执行（并且使用线程一直轮询，直到有IO资源准备好了）

#### 同步IO 和 非同步IO

- 这两个概念是操作系统级别的。主要描述的是操作系统在收到程序请求IO操作后，如果IO资源没有准备好，该如何相应程序的问题：
  - 前者不响应，直到IO资源准备好以后
  - 后者返回一个标记（好让程序和自己知道以后的数据往哪里通知），当IO资源准备好以后，再用事件机制返回给程序

#### Linux种IO模型

- 阻塞式IO模型

  - 当用户线程发出IO请求之后，内核会去查看数据是否就绪，如果没有就绪就会等待数据就绪，而用户线程就会处于阻塞状态，用户线程交出CPU。当数据就绪之后，内核会将数据拷贝到用户线程，并返回结果给用户线程，用户线程才解除block状态

  - ```java
    data = socket.read();
    ```

  - 如果数据没有就绪，就会一直阻塞在read方法

- 非阻塞IO模型

  - 当用户线程发起一个read操作后，并不需要等待，而是马上就得到了一个结果。如果结果是一个error时，它就知道数据还没有准备好，于是它可以再次发送read操作。一旦内核中的数据准备好了，并且又再次收到了用户线程的请求，那么它马上就将数据拷贝到了用户线程，然后返回，所以事实上，在非阻塞IO模型中，用户线程需要不断地询问内核数据是否就绪，也就说非阻塞IO不会交出CPU，而会一直占用CPU

  - ```java
    while(true) {
    	data = socket.read();
    	if (data != error) {
    		// 处理数据
    		break;
    	}
    }
    ```

  - 但是对于非阻塞IO就有一个非常严重的问题，在while循环中需要不断地去询问内核数据是否就绪，这样会导致CPU占用率非常高，因此一般情况下很少使用while循环这种方式来读取数据

- IO 复用模型

  - 在多路复用IO模型中，会有一个线程不断去轮询多个socket的状态，只有当socket真正有读写事件时，才真正调用实际的IO读写操作。因为在多路复用IO模型中，只需要使用一个线程就可以管理多个socket，系统不需要建立新的进程或者线程，也不必维护这些线程和进程，并且只有在真正有socket读写事件进行时，才会使用IO资源，所以它大大减少了资源占用
  - Linux支持IO多路复用的系统调用有select、poll、epoll，这些都是内核级别的，但select、poll、epoll本质上都是同步I/O，先是block住等待就绪的socket，再是block住将数据从内核拷贝到用户内存

- 异步IO模型

## 动态代理

#### 实现方式

- JDK动态代理：java.lang.reflect 包中的Proxy类和InvocationHandler接口提供了生成动态代理类的能力
- Cglib动态代理：Cglib (Code Generation Library )是一个第三方代码生成类库，运行时在内存中动态生成一个子类对象从而实现对目标对象功能的扩展
- JDK动态代理和Cglib动态代理的区别 
  - 使用动态代理的对象必须实现一个或多个接口
  - 使用cglib代理的对象则无需实现接口，达到代理类无侵入

## 语法糖

#### 糖块一、switch 支持 String与枚举

- ```java
  public class switchDemoString {
      public static void main(String[] args) {
          String str = "world";
          switch (str) {
          case "hello":
              System.out.println("hello");
              break;
          case "world":
              System.out.println("world");
              break;
          default:
              break;
          }
      }
  }
  ```

- 反编译后

- ```java
  public class switchDemoString
  {
      public switchDemoString()
      {
      }
      public static void main(String args[])
      {
          String str = "world";
          String s;
          switch((s = str).hashCode())
          {
          default:
              break;
          case 99162322:
              if(s.equals("hello"))
                  System.out.println("hello");
              break;
          case 113318802:
              if(s.equals("world"))
                  System.out.println("world");
              break;
          }
      }
  }
  ```

- 原来字符串的switch是通过equals()和hashCode()方法来实现的

#### 糖块二、泛型

- 泛型擦除，所有类型都是Object类型

- 对于Java虚拟机来说，他根本不认识`Map map`这样的语法。需要在编译阶段通过类型擦除的方式进行解语法糖

- ```java
  Map<String, String> map = new HashMap<String, String>();
  map.put("name", "hollis");
  map.put("wechat", "Hollis");
  map.put("blog", "www.hollischuang.com");
  ```

- 解语法糖后

- ```java
  Map map = new HashMap();
  map.put("name", "hollis");
  map.put("wechat", "Hollis");
  map.put("blog", "www.hollischuang.com");
  ```

- 虚拟机中没有泛型，只有普通类和普通方法，所有泛型类的类型参数在编译时都会被擦除，泛型类并没有自己独有的Class类对象。比如并不存在List<String>.class或是List<Integer>.class，而只有List.class

#### 糖块三、自动装箱与拆箱

- 自动装箱

  - ```java
     public static void main(String[] args) {
        int i = 10;
        Integer n = i;
    }
    ```

  - 反编译后

  - ```java
    public static void main(String args[])
    {
        int i = 10;
        Integer n = Integer.valueOf(i);
    }
    ```

- 自动拆箱

  - ```java
    public static void main(String[] args) {
    
        Integer i = 10;
        int n = i;
    }
    ```

  - 反编译后

  - ```java
    public static void main(String args[])
    {
        Integer i = Integer.valueOf(10);
        int n = i.intValue();
    }
    ```

  - 在装箱的时候自动调用的是`Integer`的`valueOf(int)`方法。而在拆箱的时候自动调用的是`Integer`的`intValue`方法

#### 糖块四、方法变长参数

- ```java
  public static void main(String[] args)
      {
          print("Holis", "公众号:Hollis", "博客：www.hollischuang.com", "QQ：907607222");
      }
  
  public static void print(String... strs)
  {
      for (int i = 0; i < strs.length; i++)
      {
          System.out.println(strs[i]);
      }
  }
  ```

- 反编译后

- ```java
   public static void main(String args[])
  {
      print(new String[] {
          "Holis", "\u516C\u4F17\u53F7:Hollis", "\u535A\u5BA2\uFF1Awww.hollischuang.com", "QQ\uFF1A907607222"
      });
  }
  
  public static transient void print(String strs[])
  {
      for(int i = 0; i < strs.length; i++)
          System.out.println(strs[i]);
  
  }
  ```

- 从反编译后代码可以看出，可变参数在被使用的时候，他首先会创建一个数组，数组的长度就是调用该方法是传递的实参的个数，然后再把参数值全部放到这个数组当中，然后再把这个数组作为参数传递到被调用的方法中

#### 糖块五、枚举

- ```java
  public enum t {
      SPRING,SUMMER;
  }
  ```

- 反编译后

- ```java
  public final class T extends Enum
  {
      private T(String s, int i)
      {
          super(s, i);
      }
      public static T[] values()
      {
          T at[];
          int i;
          T at1[];
          System.arraycopy(at = ENUM$VALUES, 0, at1 = new T[i = at.length], 0, i);
          return at1;
      }
  
      public static T valueOf(String s)
      {
          return (T)Enum.valueOf(demo/T, s);
      }
  
      public static final T SPRING;
      public static final T SUMMER;
      private static final T ENUM$VALUES[];
      static
      {
          SPRING = new T("SPRING", 0);
          SUMMER = new T("SUMMER", 1);
          ENUM$VALUES = (new T[] {
              SPRING, SUMMER
          });
      }
  }
  ```

- 通过反编译后代码我们可以看到，public final class T extends Enum，说明，该类是继承了Enum类的，同时final关键字告诉我们，这个类也是不能被继承的。当我们使用enmu来定义一个枚举类型的时候，编译器会自动帮我们创建一个final类型的类继承Enum类，所以枚举类型不能被继承

#### 糖块六、内部类

- 内部类之所以也是语法糖，是因为它仅仅是一个编译时的概念，outer.java里面定义了一个内部类inner，一旦编译成功，就会生成两个完全不同的.class文件了，分别是outer.class和outer$inner.class。所以内部类的名字完全可以和它的外部类名字相同

#### 糖块七、条件编译

- 所以，Java语法的条件编译，是通过判断条件为常量的if语句实现的。其原理也是Java语言的语法糖。根据if判断条件的真假，编译器直接把分支为false的代码块消除。通过该方式实现的条件编译，必须在方法体内实现，而无法在整个Java类的结构或者类的属性上进行条件编译，这与C/C++的条件编译相比，确实更有局限性。在Java语言设计之初并没有引入条件编译的功能，虽有局限，但是总比没有更强

#### 糖块八、数值字面量

- ```java
  public class Test {
      public static void main(String... args) {
          int i = 10_000;
          System.out.println(i);
      }
  }
  ```

- 反编译后

- ```java
  public class Test
  {
    public static void main(String[] args)
    {
      int i = 10000;
      System.out.println(i);
    }
  }
  ```

- 反编译后就是把_删除了。也就是说 编译器并不认识在数字字面量中的_，需要在编译阶段把他去掉

#### 糖块九、for-each

- for-each的实现原理其实就是使用了普通的for循环和迭代器

#### 糖块十、try-with-resource

- ```java
  public static void main(String... args) {
      try (BufferedReader br = new BufferedReader(new FileReader("d:\\ hollischuang.xml"))) {
          String line;
          while ((line = br.readLine()) != null) {
              System.out.println(line);
          }
      } catch (IOException e) {
          // handle exception
      }
  }
  ```

- 其实背后的原理也很简单，那些我们没有做的关闭资源的操作，编译器都帮我们做了。所以，再次印证了，语法糖的作用就是方便程序员的使用，但最终还是要转成编译器认识的语言

#### 糖块十一、Lambda表达式

#### 泛型擦除的坑

- ```java
      public static void method(List<String> list) {
          System.out.println("invoke method(List<String> list)");
      }
  
      public static void method(List<Integer> list) {
          System.out.println("invoke method(List<Integer> list)");
      }
  }
  ```

- 上面这段代码，有两个重载的函数，因为他们的参数类型不同，一个是List另一个是List ，但是，这段代码是编译通不过的。因为我们前面讲过，参数List和List编译之后都被擦除了，变成了一样的原生类型List，擦除动作导致这两个方法的特征签名变得一模一样

# Java并发编程

## 位运算

- 位异或运算（^）：运算规则是：两个数转为二进制，然后从高位开始比较，如果相同则为0，不相同则为1
- 位与运算符（&）：两个数都转为二进制，然后从高位开始比较，如果两个数都为1则为1，否则为0
- 位或运算符（|）：两个数都转为二进制，然后从高位开始比较，两个数只要有一个为1则为1，否则就为0
- 位非运算符（~）：如果位为0，结果是1，如果位为1，结果是0

## sleep()

- 让一个线程进入阻塞状态，但是不会释放锁，当时时间结束后，就会立即拿到锁，进入就绪状态

## join()

- 让主线程等待子线程执行完毕以后，再继续执行主线程的代码。join内部是调用的wait方法来实现的

## Java对象头

- 第一部分：存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等。官方称为 Mark Word
- 第二部分：类型指针，对象执行它的元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例
- 如果对象是一个Java数组，那在对象头中还必须用一块记录数组长度的数据，因为虚拟机可以通过普通Java对象的元数据信息确定Java对象的大小，但是从数组的元数据中却无法确定数组的大小

## synchronized

- 内部是通过monitorenter、monitorexter指令和monitor对象来实现的；
- ObjectMonitor中有两个队列，_WaitSet 和 _EntryList，用来保存ObjectWaiter对象列表( 每个等待锁的线程都会被封装成ObjectWaiter对象，在Hotspot源码中)_
- owner指向持有ObjectMonitor对象的线程，当多个线程同时访问一段同步代码时，首先会进入 _EntryList 集合，当线程获取到对象的monitor 后进入 _Owner 区域并把monitor中的owner变量设置为当前线程同时monitor中的计数器count加1
- 若线程调用 wait() 方法，将释放当前持有的monitor，owner变量恢复为null，count自减1，同时该线程进入 WaitSet集合中等待被唤醒
- 若当前线程执行完毕也将释放monitor(锁)并复位变量的值，以便其他线程进入获取monitor(锁)；由此看来，monitor对象存在于每个Java对象的对象头中(存储的指针的指向)
- synchronized锁便是通过这种方式获取锁的，也是为什么Java中任意对象可以作为锁的原因，同时也是notify/notifyAll/wait等方法存在于顶级对象Object中的原因
- notify/notifyAll和wait方法，在使用这3个方法时，必须在synchronized代码块或者synchronized方法中，否则就会抛出IllegalMonitorStateException异常，这是因为调用这几个方法前必须拿到当前对象的监视器monitor对象，也就是说notify/notifyAll和wait方法依赖于monitor对象。同时notify/notifyAll方法调用后，并不会马上释放监视器锁，而是在相应的synchronized(){}/synchronized方法执行结束后才自动释放锁，这是交给jvm处理的
- synchronized属于重量级锁，效率低下，因为监视器锁（monitor）是依赖于底层的操作系统的Mutex Lock来实现的，而操作系统实现线程之间的切换时需要从用户态转换到核心态，这个状态之间的转换需要相对比较长的时间
- 在Java 6之后Java官方对从JVM层面对synchronized较大优化
- synchronized是可重入锁，在一个线程调用synchronized方法的同时在其方法体内部调用该对象另一个synchronized方法，也就是说一个线程得到一个对象锁后再次请求该对象锁，是允许的，这就是synchronized的可重入性
- 出现异常时会释放锁

## 锁优化

### 偏向锁

- 如果一个线程获得了锁，那么锁就进入偏向模式，此时Mark Word 的结构也变为偏向锁结构，当这个线程再次请求锁时，无需再做任何同步操作，即获取锁的过程，偏向锁的释放不需要做任何事情，这也就意味着加过偏向锁的MarkValue会一直保留偏向锁的状态，因此即便同一个线程持续不断地加锁解锁，也是没有开销的，偏向锁只有在不同线程请求锁是才会升级为轻量级锁，偏向锁使用了一种等到竞争出现才释放锁的机制，所以当其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁
- 偏向锁默认开启，可以通过JVM参数关闭偏向锁-XX:-UseBiaseLocking=false，那么默认进入轻量级锁状态
- 原理：当一个线程访问同步块并获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程ID，以后该线程在进入和退出同步块时不需要花费CAS操作来加锁和解锁，而只需简单的测试一下对象头的Mark Word里是否存储着指向当前线程的偏向锁，如果测试成功，表示线程已经获得了锁，如果测试失败，则需要再测试下Mark Word中偏向锁的标识是否设置成1（表示当前是偏向锁），如果没有设置，则使用CAS竞争锁，如果设置了，则尝试使用CAS将对象头的偏向锁指向当前线程

### 轻量级锁

- 使用CAS来竞争锁，两条或两条以上的线程竞争同一个锁，则轻量级锁会膨胀成重量级锁
- 线程尝试使用CAS将对象头中的Mark Word替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失败，则自旋获取锁，当自旋获取锁仍然失败时，表示存在其他线程竞争锁(两条或两条以上的线程竞争同一个锁)，则轻量级锁会膨胀成重量级锁

### 自旋锁

- 线程获取锁失败，就过一会再去获取，比如让线程去执行一个无意义的循环，循环结束后再去重新竞争锁，如果竞争不到继续循环，循环过程中线程会一直处于running状态，但是基于JVM的线程调度，会出让时间片，所以其他线程依旧有申请锁和释放锁的机会。自旋需要合理的配置

### 重量级锁

### 锁消除/锁粗化

- 将一些细小的锁，其实不会出现安全问题，但是很多细小的锁，会导致锁竞争，效率低下，可以将多个小锁扩

  展到一个大锁，这样可以减少锁的竞争。这里的大指的的范围

### 锁池

- 假设线程A已经拥有了某个对象(注意:不是类)的锁，而其它的线程想要调用这个对象的某个synchronized方法(或者synchronized块)，由于这些线程在进入对象的synchronized方法之前必须先获得该对象的锁的拥有权，但是该对象的锁目前正被线程A拥有，所以这些线程就进入了该对象的锁池中

### 等待池

- 假设一个线程A调用了某个对象的wait()方法，线程A就会释放该对象的锁后，进入到了该对象的等待池中，等待池中的线程不会去竞争该对象的锁；当有线程调用了对象的 notifyAll()方法（唤醒所有 wait 线程）或 notify()方法（只随机唤醒一个 wait 线程），被唤醒的的线程便会进入该对象的锁池中，锁池中的线程会去竞争该对象锁

## volatile

- 保证变量的可见性、防止指令重排序，内部采用内存屏障来实现的
- 如果对声明了volatile的变量进行写操作，JVM就会向处理器发送一条Lock前缀的指令，将这个变量所在缓存行的数据写回到系统内存。但是，就算写回到内存，如果其他处理器缓存的值还是旧的，再执行计算操作就会有问题。所以，在多处理器下，为了保证各个处理器的缓存是一致的，就会实现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器对这个数据进行修改操作的时候，会重新从系统内存中把数据读到处理器缓存里。volatile并不能保证线程安全问题，只保证可见性，不保证数据一致
- 可见性
  - 嗅探机制 强制失效 处理器嗅探总线
- 有序性
  - 禁止指令重排序 lock前缀指令 内存屏障

## happens-before

- 程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作
- 监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁
- volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读
- 传递性：如果A happens-before B，且B happens-before C，那么A happens-before C
- 注意：两个操作之间具有happens-before关系，并不意味着前一个操作必须要在后一个操作之前执行！happens-before仅仅要求前一个操作（执行的结果）对后一个操作可见，且前一个操作按顺序排在第二个操作之前

## as-if-serial

- 不管怎么重排序（编译器和处理器为了提高效率），单线程 程序的执行结果不能被改变。
- 编译器和处理器不会对存在数据依赖关系的操作做重排序，因为这种重排序会改变程序结果
- 如果操作之间不存在依赖关系，那么编译器和处理器会重排序

## 线程间怎么通信？

- wait/notify机制、共享变量 synchronized 或者 lock 同步机制等
- volatile
- CountDonwLatch
- CyclicBarrier

## ThreadLocal 用来解决什么问题？

- 解决线程数据隔离

## 如何尽可能提高多线程并发性能？

- 使用ThreadLocal
- 减少线程切换
- 使用读写锁 copyonwrite 等机制 这些方面回答

## 读写锁适用于什么场景？

- 读写锁适合并发多、写并发少的场景
- copyonwrite

## 如何实现一个生产者与消费者模型？

- 锁
- 信号量
- 线程通信
- 阻塞队列

## AQS (AbstractQueuedSynchronizer) 队列同步器

- 同步器的设计是基于模板方法模式的，使用者需要继承同步器并重写指定的方法，随后将同步器组合在自定义同步组件的实现中，并调用同步器提供的模板方法，而这些模板方法将会调用使用者重写的方法
- 子类推荐被定义为自定义同步装置的内部类，同步器拥有三个成员变量：sync队列的头结点head、sync队列的尾节点tail和状态state。对于锁的获取，请求形成节点，将其挂载在尾部，而锁资源的转移（释放再获取）是从头部开始向后进行

## BlockingQueue

## CopyOnWrite

## ConcurrentSkipListMap

## ConcurrentLikedQueue

## Semaphore

## CountDownLatch

## CyclicBarrier

## LockSupport

## CompletableFuture

## Fork/Join框架

## Atomic*

## Unsafe

## 线程池

- 工作原理
  - 如果当前运行的线程少于corePoolSize，则创建新线程来执行任务（注意，执行这一步骤需要获取全局锁）
  - 如果运行的线程等于或多于corePoolSize，则将任务加入BlockingQueue
  - 如果无法将任务加入BlockingQueue（队列已满），判断当前运行的线程数是否超过maxmumPoolSize，如果未超出则创建新的线程来处理任务；如果已超出则调用线程池饱和策略
- 重要参数
  - corePoolSize（线程池的基本大小）：当提交一个任务到线程池时，线程池会创建一个线程来执行任务，即使其他空闲的基本线程能够执行新任务也会创建线程，等到需要执行的任务数大于线程池基本大小时就不再创建。如果调用了线程池的prestartAllCoreThreads()方法，线程池会提前创建并启动所有基本线程
  - runnableTaskQueue（任务队列）：用于保存等待执行的任务的阻塞队列。可以选择以下几个阻塞队列
    - ArrayBlockingQueue：是一个基于数组结构的有界阻塞队列，此队列按FIFO（先进先出）原则对元素进行排序
    - LinkedBlockingQueue：一个基于链表结构的阻塞队列，此队列按FIFO排序元素，吞吐量通常要高于ArrayBlockingQueue。静态工厂方法Executors.newFixedThreadPool()使用了这个队列
    - SynchronousQueue：一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于Linked-BlockingQueue，静态工厂方法    ·Executors.newCachedThreadPool使用了这个队列
    - PriorityBlockingQueue：一个具有优先级的无限阻塞队列
  - maximumPoolSize（线程池最大数量）：线程池允许创建的最大线程数。如果队列满了，并且已创建的线程数小于最大线程数，则线程池会再创建新的线程执行任务。值得注意的是，如果使用了无界的任务队列这个参数就没什么效果
  - ThreadFactory：用于设置创建线程的工厂，可以通过线程工厂给每个创建出来的线程设置更有意义的名字。使用开源框架guava提供的ThreadFactoryBuilder可以快速给线程池里的线程设置有意义的名字，代码如下。new ThreadFactoryBuilder().setNameFormat("XX-task-%d").build();
  - RejectedExecutionHandler（饱和策略）：当队列和线程池都满了，说明线程池处于饱和状态，那么必须采取一种策略处理提交的新任务。这个策略默认情况下是AbortPolicy，表示无法处理新任务时抛出异常。在JDK 1.5中Java线程池框架提供了以下4种策略
    - AbortPolicy：直接抛出异常
    - CallerRunsPolicy：只用调用者所在线程来运行任务
    - DiscardOldestPolicy：丢弃队列里最近的一个任务，并执行当前任务
    - DiscardPolicy：不处理，丢弃掉
    - 当然，也可以根据应用场景需要来实现RejectedExecutionHandler接口自定义策略。如记录日志或持久化存储不能处理的任务
  - keepAliveTime（线程活动保持时间）：线程池的工作线程空闲后，保持存活的时间。所以，如果任务很多，并且每个任务执行的时间比较短，可以调大时间，提高线程的利用率
  - TimeUnit
- 线程池关闭
  - 调用shutdown或shutdownNow方法来关闭线程池
  - 它们的原理是遍历线程池中的工作线程，然后逐个调用线程的interrupt方法来中断线程，所以无法响应中断的任务可能永远无法终止
  - shutdownNow首先将线程池的状态设置成STOP，然后尝试停止所有的正在执行或暂停任务的线程，并返回等待执行任务的列表
  - shutdown只是将线程池的状态设置成SHUTDOWN状态，然后中断所有没有正在执行任务的线程。只要调用了这两个关闭方法中的任意一个，isShutdown方法就会返回true
  - 当所有的任务都已关闭后，才表示线程池关闭成功，这时调用isTerminaed方法会返回true。至于应该调用哪一种方法来关闭线程池，应该由提交到线程池的任务特性决定，通常调用shutdown方法来关闭线程池，如果任务不一定要执行完，则可以调用shutdownNow方法
- 线程池监控
  - 扩展ThreadPoolExecutor，重写beforeExecute和afterExecute，在这两个方法里分别做一些任务执行前和任务执行后的相关监控逻辑，还有个terminated方法，是在线程池关闭后回调
- 为什么不允许使用Executors创建线程池
  - 这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险

## CAS

- 一个线程将某一内存地址中的数值A改成了B，接着又改成了A，此时CAS认为是没有变化，其实是已经变化过了，而这个问题的解决方案可以使用版本号标识，每操作一次version加1。在java5中，已经提供了AtomicStampedReference来解决问题
- CAS造成CPU利用率增加。之前说过了CAS里面是一个循环判断的过程，如果线程一直没有获取到状态，cpu资源会一直被占用
- 能保证一个共享变量的原子操作。当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁，或者有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。从Java1.5开始JDK提供了**AtomicReference**类来保证引用对象之间的原子性，可以把多个变量放在一个对象里来进行CAS操作
- ABA问题的解决办法：在变量前面追加版本号或者时间戳，每次变量更新就把版本号加1，则A-B-A就变成1A-2B-3A

## 死锁

- 死锁产生的条件

  - 互斥:资源的锁是排他性的，加锁期间只能有一个线程拥有该资源。其他线程只能等待锁释放才能尝试获取该资源
  - 请求和保持:当前线程已经拥有至少一个资源，但其同时又发出新的资源请求，而被请求的资源被其他线程拥有。此时进入保持当前资源并等待下个资源的状态
  - 不剥夺：线程已拥有的资源，只能由自己释放，不能被其他线程剥夺
  - 循环等待：是指有多个线程互相的请求对方的资源，但同时拥有对方下一步所需的资源。形成一种循环，类似2)请求和保持。但此处指多个线程的关系。并不是指单个线程一直在循环中等待

- 死锁案例

  - ```java
    public class DeadLockDemo implements Runnable{
    
        public static int flag = 1;
    
        //static 变量是 类对象共享的
        static Object o1 = new Object();
        static Object o2 = new Object();
    
        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName() + "：此时 flag = " + flag);
            if(flag == 1){
                synchronized (o1){
                    try {
                        System.out.println("我是" + Thread.currentThread().getName() + "锁住 o1");
                        Thread.sleep(3000);
                        System.out.println(Thread.currentThread().getName() + "醒来->准备获取 o2");
                    }catch (Exception e){
                        e.printStackTrace();
                    }
                    synchronized (o2){
                        System.out.println(Thread.currentThread().getName() + "拿到 o2");//第24行
                    }
                }
            }
            if(flag == 0){
                synchronized (o2){
                    try {
                        System.out.println("我是" + Thread.currentThread().getName() + "锁住 o2");
                        Thread.sleep(3000);
                        System.out.println(Thread.currentThread().getName() + "醒来->准备获取 o2");
                    }catch (Exception e){
                        e.printStackTrace();
                    }
                    synchronized (o1){
                        System.out.println(Thread.currentThread().getName() + "拿到 o1");//第38行
                    }
                }
            }
        }
    
        public static  void main(String args[]){
    
            DeadLockDemo t1 = new DeadLockDemo();
            DeadLockDemo t2 = new DeadLockDemo();
            t1.flag = 1;
            new Thread(t1).start();
    
            //让main线程休眠1秒钟,保证t2开启锁住o2.进入死锁
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
    
            t2.flag = 0;
            new Thread(t2).start();
    
        }
    }
    ```

  - 代码中， t1创建，t1先拿到o1的锁，开始休眠3秒。然后 t2线程创建，t2拿到o2的锁，开始休眠3秒。然后 t1先醒来，准备拿o2的锁，发现o2已经加锁，只能等待o2的锁释放。 t2后醒来，准备拿o1的锁，发现o1已经加锁，只能等待o1的锁释放。 t1,t2形成死锁

- 排查死锁

  - jps显示所有当前Java虚拟机进程名及pid
  - jstack打印进程堆栈信息

- 解决办法

  - 死锁一旦发生，我们就无法解决了。所以我们只能避免死锁的发生。 既然死锁需要满足四种条件，那我们就从条件下手，只要打破任意规则即可
    - （互斥）尽量少用互斥锁，能加读锁，不加写锁。当然这条无法避免
    - （请求和保持）采用资源静态分配策略（进程资源静态分配方式是指一个进程在建立时就分配了它需要的全部资源）.我们尽量不让线程同时去请求多个锁，或者在拥有一个锁又请求不到下个锁时，不保持等待，先释放资源等待一段时间在重新请求
    - （不剥夺）允许进程剥夺使用其他进程占有的资源。优先级
    - （循环等待）尽量调整获得锁的顺序，不发生嵌套资源请求。加入超时

# JVM

## 运行时数据区域

- 程序计数器
- 虚拟机栈
  - 线程私有，生命周期和线程相同
  - 方法执行时会创建一个栈帧用于存储局部变量表、操作数栈、动态链接、方法出口等信息
  - 每个方法从调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中入栈到出栈的过程
  - 局部变量表存放了编译器可知的各种基本数据类型（boolean、byte、char、short、int、float、long、double）、对象引用，其中64位长度的long和double类型的数据会占用2个局部变量空间（slot），其余数据只占用1个
  - 局部变量表所需要的内存空间在编译器间完成分配，当进入一个方法时，这个方法需要在帧中分配多大的局部变量空间是完全确定的，在方法运行期间不会改变局部变量表的大小
  - 如果线程请求的深度大于虚拟机所运行的深度，将会抛出StackOutflowError异常
  - 如果虚拟机可以动态扩展，扩展时无法申请到足够的内存，就会抛出OutOfMemoryError异常
- 本地方法栈
  - 本地方法栈是为Java虚拟机的native方法服务的
  - 本地方法栈也会抛出StackOutflowError异常和OutOfMemoryError异常
- 堆
  - 线程共享，与虚拟机生命周期一致
  - 分代收集算法，Java堆细分为新生代和老年代；新生代包括：Eden空间、From Survivor空间、To Survivor空间；
  - Java虚拟机规范规定，Java堆可以处于物理上不连续的内存空间，只要逻辑上是连续的即可
  - 如果堆中没有内存完成实例分配，并且堆也无法在扩展时，将会抛出OutOfMemoryError异常
- 方法区，JDK1.8已经将方法区移至Metaspace，字符串常量移至Java Heap
  - 线程共享
  - 存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据
  - 虽然Java虚拟机规范把方法区描述为堆的一个逻辑部分，但是它有一个别名叫做Non-Heap（非堆），目的应该是与Java堆区分开来
  - 当方法区无法满足内存分配需求时，将抛出OutOfMemoryError异常
- 运行时常量池
  - 运行时常量区是方法区的一部分。jdk1.8已经移动到Java堆中了
  - 用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放
  - 运行时常量池具备动态性，Java语言并不要求常量一定要编译期才能产生，运行期间也可能将新的常量池放入池中，比如String类中的intern()方法
  - 当常量池无法再申请到内存时会抛出OutOfMemoryError异常
- 直接内存
  - 直接内存不是Java虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域
  - 在jdk1.4新加入的NIO类中，引入了一种基于通道（Channel）与缓冲区（Buffer）的I/O方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作。这样能在一些场景中显著提高性能，因为避免了在Java堆和Native堆中来回复制数据
  - 显然，本机内存不会受到Java堆大小的限制，但是还是会受到本机内存大小的限制
  - 在配置虚拟机参数时，会根据实际内存大小设置-Xmx等参数信息，但忽略了直接内存，使得各个内存区域总和大于物理内存限制，从而导致动态扩展时出现OutOfMemoryError异常

## Java8元空间

- 方法区永久代的弊端
  - PermGen space的全称是Permanent Generation space,是指内存的永久保存区域，说说为什么会内存益出：这一部分用于存放Class和Meta的信息，Class在被 Load的时候被放入PermGen space区域，它和存放Instance的Heap区域不同,所以如果你的APP会LOAD很多CLASS的话,就很可能出现PermGen space错误。这种错误常见在web服务器对JSP进行pre compile的时候
  - 如果JVM发现有的类已经不再需要了，它会去回收（卸载）这些类，将它们的空间释放出来给其它类使用。Full GC会进行持久代的回收
- 方法区永久代存储的信息
  - JVM中类的元数据在Java堆中的存储区域
  - Java类对应的HotSpot虚拟机中的内部表示也存储在这里
  - 类的层级信息，字段，名字
  - 方法的编译信息及字节码
  - 变量
  - 常量池和符号解析
- 永久代的大小
  - 它的上限是MaxPermSize，默认是64M
  - 持久代用完后，会抛出OutOfMemoryError "PermGen space"异常。解决方案：应用程序清理引用来触发类卸载；增加MaxPermSize的大小
  - 需要多大的持久代空间取决于类的数量，方法的大小，以及常量池的大小
- 为什么移除永久带？
  - 它的大小是在启动时固定好的——很难进行调优。-XX:MaxPermSize，设置成多少好呢？
  - HotSpot的内部类型也是Java对象：它可能会在Full GC中被移动，同时它对应用不透明，且是非强类型的，难以跟踪调试，还需要存储元数据的元数据信息（meta-metadata）
  - 简化Full GC：每一个回收器有专门的元数据迭代器
  - 可以在GC不进行暂停的情况下并发地释放类数据
  - 使得原来受限于持久代的一些改进未来有可能实现
  - 根据上面的各种原因，永久代最终被移除，方法区移至Metaspace，字符串常量池移至Java Heap
- 移除持久代后，PermGen空间的状况
  - 这部分内存空间将全部移除
  - JVM的参数：PermSize 和 MaxPermSize 会被忽略并给出警告（如果在启用时设置了这两个参数）
- Metaspace的组成
  - Klass Metaspace
    - Klass Metaspace就是用来存klass的，klass是我们熟知的class文件在jvm里的运行时数据结构，不过有点要提的是我们看到的类似A.class其实是存在heap里的，是java.lang.Class的一个对象实例。这块内存是紧接着Heap的，和我们之前的perm一样，这块内存大小可通过-XX:CompressedClassSpaceSize参数来控制，这个参数前面提到了默认是1G，但是这块内存也可以没有，假如没有开启压缩指针就不会有这块内存，这种情况下klass都会存在NoKlass Metaspace里，另外如果我们把-Xmx设置大于32G的话，其实也是没有这块内存的，因为这么大内存会关闭压缩指针开关。还有就是这块内存最多只会存在一块
  - NoKlass Metaspace
    - NoKlass Metaspace专门来存klass相关的其他的内容，比如method，constantPool等，这块内存是由多块内存组合起来的，所以可以认为是不连续的内存块组成的。这块内存是必须的，虽然叫做NoKlass Metaspace，但是也其实可以存klass的内容，上面已经提到了对应场景
  - Klass Metaspace和NoKlass Mestaspace都是所有classloader共享的，所以类加载器们要分配内存，但是每个类加载器都有一个SpaceManager，来管理属于这个类加载的内存小块。如果Klass Metaspace用完了，那就会OOM了，不过一般情况下不会，NoKlass Mestaspace是由一块块内存慢慢组合起来的，在没有达到限制条件的情况下，会不断加长这条链，让它可以持续工作
  - 元空间与永久代之间最大的区别在于：
    - 元空间并不在虚拟机中，而是使用本地内存。因此，元空间的大小仅受本地内存限制
- 元空间参数
  - -XX:MetaspaceSize：初始空间大小，达到该值就会触发垃圾收集进行类型卸载，同时GC会对该值进行调整：如果释放了大量的空间，就适当降低该值；如果释放了很少的空间，那么在不超过MaxMetaspaceSize时，适当提高该值
  - -XX:MaxMetaspaceSize：最大空间，默认是没有限制的
  - -XX:MinMetaspaceFreeRatio：在GC之后，最小的Metaspace剩余空间容量的百分比，减少为分配空间所导致的垃圾收集 
  - -XX:MaxMetaspaceFreeRatio：在GC之后，最大的Metaspace剩余空间容量的百分比，减少为释放空间所导致的垃圾收集
  - -verbose参数是为了获取类型加载和卸载的信息
- 元空间特点
  - 充分利用了Java语言规范中的好处：类及相关的元数据的生命周期与类加载器的一致
  - 每个加载器有专门的存储空间
  - 只进行线性分配
  - 不会单独回收某个类
  - 省掉了GC扫描及压缩的时间
  - 元空间里的对象的位置是固定的
  - 如果GC发现某个类加载器不再存活了，会把相关的空间整个回收掉
- 元空间内存分配模型
  - 绝大多数的类元数据的空间都从本地内存中分配
  - 用来描述类元数据的类(klasses)也被删除了
  - 分元数据分配了多个虚拟内存空间
  - 给每个类加载器分配一个内存块的列表。块的大小取决于类加载器的类型; sun/反射/代理对应的类加载器的块会小一些
  - 归还内存块，释放内存块列表
  - 一旦元空间的数据被清空了，虚拟内存的空间会被回收掉
  - 减少碎片的策略
- 元空间内存管理
  - 元空间的内存管理由元空间虚拟机来完成。先前，对于类的元数据我们需要不同的垃圾回收器进行处理，现在只需要执行元空间虚拟机的C++代码即可完成。在元空间中，类和其元数据的生命周期和其对应的类加载器是相同的。话句话说，只要类加载器存活，其加载的类的元数据也是存活的，因而不会被回收掉
  - 准确的来说，每一个类加载器的存储区域都称作一个元空间，所有的元空间合在一起就是我们一直说的元空间。当一个类加载器被垃圾回收器标记为不再存活，其对应的元空间会被回收。在元空间的回收过程中没有重定位和压缩等操作。但是元空间内的元数据会进行扫描来确定Java引用
  - 元空间虚拟机负责元空间的分配，其采用的形式为组块分配。组块的大小因类加载器的类型而异。在元空间虚拟机中存在一个全局的空闲组块列表。当一个类加载器需要组块时，它就会从这个全局的组块列表中获取并维持一个自己的组块列表。当一个类加载器不再存活，那么其持有的组块将会被释放，并返回给全局组块列表。类加载器持有的组块又会被分成多个块，每一个块存储一个单元的元信息。组块中的块是线性分配（指针碰撞分配形式）。组块分配自内存映射区域。这些全局的虚拟内存映射区域以链表形式连接，一旦某个虚拟内存映射区域清空，这部分内存就会返回给操作系统
- 元空间Metaspace 调优
  - 使用-XX:MaxMetaspaceSize参数可以设置元空间的最大值，默认是没有上限的，也就是说你的系统内存上限是多少它就是多少
  - -XX:MetaspaceSize选项指定的是元空间的初始大小，如果没有指定的话，元空间会根据应用程序运行时的需要动态地调整大小
    - MaxMetaspaceSize的调优
      - -XX:MaxMetaspaceSize={unlimited}
      - 元空间的大小受限于你机器的内存
      - 元空间的初始大小是21M——这是GC的初始的高水位线，超过这个大小会进行Full GC来进行类的回收
      - 如果启动后GC过于频繁，请将该值设置得大一些
      - 可以设置成和持久代一样的大小，以便推迟GC的执行时间
    - CompressedClassSpaceSize的调优
      - 只有当-XX:+UseCompressedClassPointers开启了才有效
      - -XX:CompressedClassSpaceSize=1G
      - 由于这个大小在启动的时候就固定了的，因此最好设置得大点
      - 没有使用到的话不要进行设置
      - JVM后续可能会让这个区可以动态的增长。不需要是连续的区域，只要从基地址可达就行；可能会将更多的类元信息放回到元空间中；未来会基于PredictedLoadedClassCount的值来自动的设置该空间的大小
  - 正如前面提到了，Metaspace VM管理Metaspace空间的增长。但有时你会想通过在命令行显示的设置参数-XX:MaxMetaspaceSize来限制Metaspace空间的增长。默认情况下，-XX:MaxMetaspaceSize并没有限制，因此，在技术上，Metaspace的尺寸可以增长到无限空间，而你的本地内存分配将会失败
  - 每次垃圾收集之后，Metaspace VM会自动的调整high watermark，推迟下一次对Metaspace的垃圾收集
  - 这两个参数，-XX：MinMetaspaceFreeRatio和-XX：MaxMetaspaceFreeRatio，类似于GC的FreeRatio参数，可以放在命令行
- 提高GC的性能
  - 如果你理解了元空间的概念，很容易发现GC的性能得到了提升
    - Full GC中，元数据指向元数据的那些指针都不用再扫描了。很多复杂的元数据扫描的代码（尤其是CMS里面的那些）都删除了
    - 元空间只有少量的指针指向Java堆。这包括：类的元数据中指向java/lang/Class实例的指针;数组类的元数据中，指向java/lang/Class集合的指针
    - 没有元数据压缩的开销
    - 减少了根对象的扫描（不再扫描虚拟机里面的已加载类的字典以及其它的内部哈希表）
    - 减少了Full GC的时间
    - G1回收器中，并发标记阶段完成后可以进行类的卸载
- 元空间的问题
  - 元空间虚拟机采用了组块分配的形式，同时区块的大小由类加载器类型决定。类信息并不是固定大小，因此有可能分配的空闲区块和类需要的区块大小不同，这种情况下可能导致碎片存在。元空间虚拟机目前并不支持压缩操作，所以碎片化是目前最大的问题

## Java堆内存分配方法

- 指针碰撞
  - 假设Java堆中内存是绝对规整的，所有用过的内存都放到一边，空闲的放到另一边，中间放着一块指针作为分界点的指示器，那分配内存就仅仅是把那个指针向空闲空间那边移动一段和对象大小相当的距离
- 空闲列表
  - 如果Java堆中的内存不是规整的，已使用的内存和空闲的内存相互交错，那就没有办法简单地使用碰撞指针了，虚拟机就必须维护一个列表，记录哪些内存块是可用的，在分配内存的时候从列表中找到一块足够大的内存空间划分给对象实例，并更新列表上的记录
- 选择哪种分配方式是根据Java堆是否规整来决定的，而Java堆是否规整又由所采用的垃圾收集器是否带有压缩整理功能决定
- 在使用Serial、ParNew等带有Compact过程的收集器时，系统采用的分配算法是指针碰撞
- 使用CMS这种基于Mark-Sweep算法的收集器时，采用的分配算法是采用空闲列表

## 分配内存并发安全

- 对象创建在虚拟机是非常频繁的，即使是修改一个指针所指向的位置，在并发情况下也是不安全的，可能出现正在给对象A分配内存，指针还没来得及修改，对象B同时又使用原来的指针来分配内存
- 解决方法
  - 对分配内存空间的动作进行同步处理，即虚拟机采用CAS配上失败重试的方式保证更新操作的原子性
  - 把内存分配的动作按照线程分在不同的空间中进行，即每个线程在Java堆中预先分配一块小内存，成为本地线程分配缓冲（TLAB）。哪个线程要分配内存，就在哪个线程的TLAB上分配，只有TLAB用完并分配新内存时，才需要同步锁定。虚拟机是否使用TLAB，可以通过-XX:+/-UseTLAB 参数来设定

## 对象内存布局

- 对象在内存中存储分为3块区域：对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）
  对象头
  - 第一部分：存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等。官方称为 Mark Word
  - 第二部分：类型指针，对象执行它的元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。查找对象的元数据信息并不一定要经过对象本身？？看后面
  - 如果对象是一个Java数组，那在对象头中还必须用一块记录数组长度的数据，因为虚拟机可以通过普通Java对象的元数据信息确定Java对象的大小，但是从数组的元数据中却无法确定数组的大小
- 实例数据
  - 存储对象真正有效的数据，也是程序代码中所定义的各种类型的字段内容。无论是从父类继承下来的还是在子类定义的，都需要记录下来
- 对齐填充
  - 不是必须存在的，也没有特别的含义。仅仅是起着占位符的作用。因为Hospot VM的自动内存管理系统要求对象起始地址必须是8个字节的整数倍，就是对象大小必须是8的整数倍
  - 而对象头的大小正好是8的整数倍，因此，当对象实例数据部分没有对齐时，就需要通过对齐填充来不全

## 对象回收策略

- 判断对象是否存活
  - 引用计数算法
    - 给对象添加一个引用计数器，每当有一个地方引用它时，计数器就增加1；当引用失效时，计算取就减少1；任何时刻计数器是0的对象就是不可能再被使用的
    - 会出现对象之间的相互循环引用的问题
  - 可达分析算法
    - GC Roots 链
    - 从GC Roots节点开始向下搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots没有任何引用链相连（就是从GC Roots到这个对象不可达）时，则证明这个对象是不可用的
    - 在Java语言中，可作为GC Roots的对象包括
      - 虚拟机栈（栈帧中的本地变量表）中引用的对象
      - 方法区中类静态属性引用的对象
      - 方法区中常量引用的对象
      - 本地方法栈的JNI（Native方法）引用的对象
- 对象引用
  - 强引用（Strong Reference）
    - 类似Object obj = new Object(); 这类引用，只要强引用还存在，垃圾收集器永远不会回收被引用的对象。
  - 软引用（Soft Reference）
    - 当内存不足时，软引用就会被回收掉
    - 可以通过SoftReference类来实现软引用
  - 弱引用（Weak Reference）
    - 无论内存是否充足，当垃圾收集器工作时，弱引用被关联的对象都会回收掉。
      可以通过WeakReference类来实现弱引用
  - 虚引用（Phantom Reference）
    - 是最弱的一种引用关系，一个对象是否有虚引用，完全不会对其生存时间构成影响，也无法通过虚引用取得一个对象实例
    - 为对象设置虚引用关联的唯一目的就是能在这个对象被收集器回收时收到一个系统通知。
      可以通过PhantomReference类来实现虚引用。PhantomReference类的get方法 永远返回null
- 对象死亡前期
  - 如果对象在进行可达分析后发现没有与GC Roots相连接的引用链，那它将会被第一次标记并且进行一次筛选，筛选的条件是对象是否有必要执行finalize()方法
  - 当对象没有覆盖finalize()方法或者finalize()方法已经被虚拟机调用过，虚拟机将这两种情况都视为 没有必要执行。finalize()只会被执行一次
  - 如果对象被判定有必要执行finalize()方法，对象将会被放置到一个叫做F-Queue队列之中，并在稍后有一个虚拟机自动建立的、低优先级的Finalizer线程异步执行，只是调用而已，并不会等待运行结果；如果等待执行结果的话，有可能这个方法是死循环或者非常耗时，会导致整个内存回收系统瘫痪
- 方法区回收
  - 永久代的垃圾收集主要回收两部分内容：废弃常量和无用的类
  - 废弃常量
    - 当一个常量没有任何引用时，就会被系统清理出常量池
  - 无用的类，类需要同时满足下面3个条件才能算是无用的类
    - 该类所有的实例都已经被回收，也就是Java堆中不能做该类的任何实例
    - 加载该类的ClassLoader已经被回收
    - 该类对应的java.lang.Class对象没有在任何地方被引用，无法在任何地方通过发射访问该类的方法
- 对象引用
  - 强引用
    - 这类的引用,只要强引用还存在,垃圾收集器永远不会回收掉被引用的对象
    - 永不回收
  - 软引用
    - 用来描述一些还有用但非必需的对象.对于软引用关联着的对象,在系统将要发生内存溢出异常之前,将会把这些对象列进回收范围之中进行第二次回收.如果这次回收还没有足够的内存,才会抛出内存溢出的异常
    - 内存溢出异常发生之前
  - 弱引用
    - 用来描述非必需对象的,它的强度软引用更弱一点,被弱引用的关联的对象只能生存到下一次垃圾收集发生之前.当垃圾收集器工作时,无论当前内存是否足够,都会回收掉只被弱引用关联的对象
    - 每一次垃圾收集器工作
  - 虚引用
    - 虚引用也称为幽灵引用或者幻影引用.是最弱的一种引用关系。虚引用的存在不会对对象的生存时间构成任何影响，为一个对象设置虚引用的唯一目的就是能在这个对象被收集器回收时收到一个系统通知
    - 无影响，仅在对象被回收时收到一个系统通知

## 垃圾收集算法

- 标记-清除
  - 效率低、内存碎片
- 复制
  - 将内存分为一块较大的Eden空间和两块较小的Survivor空间，每次使用Eden空间和其中一块Survivor。回收时，将Eden和Survivor中还存活的对象一次性第复制到另外一块Survivor空间上，最后清理掉Eden和刚使用过的Survivor空间
  - HotSpot虚拟机默认Eden和Survivor的大小比例是8:1，即每次新生代中可用内存空间为整个新生代容量的90%(80%+10%)，只有10%的内存会被“浪费”
  - 然98%的对象可回收只是一般场景下的数据，没办法保证每次回收都只有不多于10%的对象存活。当Survivor空间不够时，需要依赖其他内存（老年代）进行分配担保
  - 缺点：
    - 在对象存活率较高时就需要进行较多的复制操作，效率将会变低。如果不想浪费50%的空间，就需要有额外的空间进行分配担保，以应对被使用的内存中所有对象100%存活的极端情况，所以在老年代一般不能直接使用复制算法
- 标记-整理
  - 老年代使用、让所有存活的对象都向一端移动，然后直接清理掉端边界意外的内存
- 分代收集
  - 在新生代中，每次垃圾收集时发现有大批对象死去，只有少量存活，那就选用复制算法，只需要付出少量存活对象的复制成功就可以完成收集
  - 在老年代中，因为对象存活率高、没有额外的内存进行分配担保，就必须使用“标记-清除”或者“标记-整理”算法来进行回收

## HotSpot算法的实现

- 枚举根节点
  - GC Roots 链时不可能遍历所有的引用，效率太低，而且GC Root链时必须在一个能确保一致性的快照中进行，即在整个分析期间整个执行系统看起来就像是冻结在某个时间点上，不能出现分析过程中对象引用关系还在不断变化，这点就是导致GC进行时必须停顿所有的Java执行程序（Stop The World）的原因
  - 解决方法：在HotSpot的实现中，使用一组OopMap的数据结构来解决，在类加载完成的时候，HotSpot就把对象内什么偏移量上是什么对象类型的数据计算出来，在JIT编译过程中，也会在特定的位置记录下栈和寄存器中哪些位置是引用的，这样，GC扫描时就可以直接获取数据了
- 安全点（Safepoint）
  - 程序执行过程中并非在所有地方都能停顿下来GC，只有在到达安全点才能暂停，在GC时如何让程序都跑到最近的安全点上停顿下来，解决方案：
    - 抢断式中断
      - 不需要线程的执行代码主动去配合，在GC发生时，首先把所有线程全部中断，如果发现有线程中断的地方不在安全点，就恢复线程，让它跑到安全点，现在几乎没有虚拟机采用抢断式中断
    - 主动式中断
      - 当GC需要中断线程时，不直接对线程操作，仅仅简单设置一个标志，各个线程执行时主动去轮询整个标志，发现中断标志为真时就自己中断挂起
- 安全区域（Safe Region）
  - 如果程序没有分配CPU时间，没有执行的时候，即线程处于sleep或者block状态，这时候线程时无法响应中断请求，就无法到达安全点中断挂起，通过安全区域解决
  - 安全区域是指在一段代码之中，引用关系不会发生变化。在这个区域中的任何地方开始GC都是安全的，安全区域其实是安全点的扩展

## 垃圾收集器

### Serial收集器

- 新生代、复制算法、串行（Stop The World，GC停顿）

### ParNew收集器

- 新生代、复制算法、并行
- Serial的多线程版本，其他和Serial完全一致
- 使用CMS收集老年代时，新生代只能使用ParNew和Serial中的一个
- ParNew收集器是使用-XX:+UseConcMarkSweepGC选项后的默认新生代收集器，也可以使用-XX:+UseParNewGC选项来强制指定它
- 可以使用-XX:ParallelGCThreads参数来限制垃圾收集的线程数

### Parallel Scavenge收集器

- 新生代、复制算法、并行
- 关注吞吐量、吞吐量优先的收集器
- -XX:MaxGCPauseMils 最大垃圾收集停顿时间
  - 单位毫秒
  - GC尽力保证回收时间不超过设定值
  - 吞吐量=运行用户代码的时间/(运行用户代码的时间+垃圾收集的时间)，虚拟机总共运行了100分钟，其中垃圾收集花掉1分钟，吞吐量就是99%
  - 不能简单的以为设置的小一点就能使系统的垃圾收集速度变得更快。GC停顿时间缩短是以牺牲吞吐量和新生代空间来换取的：系统系统把新生代设置的小一些，收集300MB新生代肯定比收集500MB快，这也直接导致了垃圾收集发生的更频繁一些，原来10秒收集一次，每次停顿100毫秒，现在变成了5秒收集一次，每次停顿70毫秒。停顿时间的确在下降，但吞吐量也降下来了
- -XX:GCTimeRatio 吞吐量大小
  - 0-100的取值范围
  - 垃圾收集时间占总时间的比
  - 默认99，即最大允许1%时间做GC
- 注意：上面两个参数是矛盾的，因为停顿时间和吞吐量不可能同时调优，所以只存在一个
- -XX:+UseAdaptiveSizePolicy   GC自适应调节策略
  - 这是一个开关参数，打开后，就不需要手工指定新生的大小（-Xmn）、Eden与Survivor区的比列（-XX:SurvivorRatio）、晋升老年代对象的年龄（-XX:PertenureSizeThreshold）等细节参数，虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或者最大吞吐量，这种调节方式称为GC自适应的调节策略
  - 使用GC自适应的调节策略后，只需要设置基本的内存数据（如-Xmx设置最大堆），然后使用-XX:MaxGCPauseMils或者-XX:GCTimeRatio参数给虚拟机一个优化目标就可以了

### Serial Old收集器

- 老年代、串行、标记-整理算法
- Serial的老年代收集器

### Parallel Old收集器

- 老年代、并行、标记-整理算法
- Parallel Scavenge收集器的老年代版本

### CMS收集器

- 以获取最短GC停顿时间为目标的收集器
- 老年代、并行、低停顿、标记-清除（内存碎片）
- 收集过程
  - 初始标记
    - 初始标记仅仅是标记一下GC Roots能直接关联到的对象，速度很快
  - 并发标记
    - 并发标记阶段就是进行GC Roots Tracing的过程
  - 重新标记
    - 重新标记是为了修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录
  - 并发清除
  - 初始标记和并发标记需要GC停顿
  - 由于整个过程中耗时最长的并发标记和并发清除过程收集器都是和用户线程一起工作，所以总体来说，CMS收集器的内存回收过程是与用户线程一起并发执行的

### G1收集器

- 并行与并发、分代收集、复制算法
  - G1能充分利用多个CPU，多核环境下的硬件优势，来缩短停顿时间，G1收集器可以通过并发的方式让Java程序继续执行
- 空间整合：不会产生内存碎片，收集后能提供规则的可用内存。这种特性有利用程序长期运行，分配大对象时也不会应为找不到连续的内存空间而提前触发下一次GC
- 可预测的停顿
- GC不需要额外的内存空间（CMS需要预留额外的内存空间存储浮动垃圾）
- G1 堆结构
  - heap被划分为一个个相等的不连续的内存区域（regions），每个region都有一个分代的角色：eden、survivor、old
  - 每个角色的数量并没有强制的限定，也就是说对每种分代内存的大小可以动态变化
  - G1最大的特点就是高效的执行回收，优先去执行哪些大量对象可回收的区域（region）
  - G1使用了gc停顿可预测的模型，来满足用户设定的gc停顿时间，根据用户设定的目标时间，G1会自动地选择哪些region要清除，一次清除多少个region
  - G1从多个region中复制存活的对象，然后集中放入一个region中，同时整理，清除内存（复制算法）
  - 对比CMS，G1使用复制算法不会产生内存碎片
  - 对比Parallel Scavenge（复制算法）、Parallel Old（标记整理算法），Parallel会对整个区域做整理导致gc停顿会比较长，而G1只是特定地整理几个region
- G1 分区
  - 每个分区都可能是年轻代也可能是老年代，但是同一个时刻只能属于某个代
  - 年轻代、幸存区、老年代这些概念还存在，成为了逻辑上的概念，这样方便复用之前分代框架的逻辑在物理上不需要连续，则带来了额外的好处：有的分区内垃圾对象特别多，有的特别少，G1会有限回收垃圾对象特别多的分区这样可以花费较少的时间来回收这些分区的垃圾，也就是G1名字的由来，即首先收集垃圾最多的分区
- G1 相对于 CMS的优势
  - G1在压缩空间方面有优势
  - G1通过将内存空间分成区域（region）的方式避免内存碎片
  - Eden、Survivor、Old区不再固定，在内存使用效率上更灵活
  - G1可以通过设置预期停顿时间（Pause Time）来控制垃圾收集时间，避免应用雪崩现象
  - G1在回收内存后会马上同时做合并空闲内存的工作，而CMS默认是在（stop the world）的时候做
  - G1会在Young GC中使用，而CMS只能在old区使用
- G1 应用场景
  - 服务端多核CPU、JVM内存占用较大的应用
  - 应用在运行过程中会产生大量内存碎片、需要经常压缩空间
  - 想要更加可控、可预期的GC停顿周期；防止高并发下的应用雪崩现象
- G1收集器跟踪各个Region里面的垃圾堆积的价值大小（回收所获得的的空间大小及回收所需时间的经验值），在后台维护一个优先列表，每次根据允许的收集时间，优先收回价值最大的Region（也就是Garbage-First名称的由来）。这种使用Region划分内存空间以及优先级的区域回收 方式，保证了G1收集器在有限的时间内可以获取尽可能高的收集效率
- G1收集器为了避免全扫描整个堆，采取的措施
  - Region之间的对象引用以及其他收集器的新生代和老年代之间的对象引用，虚拟机都是使用Remembered Set来避免全堆扫描的
  - G1中每个Region都有一个与之对应的Remembered Set，虚拟机发现程序在对Reference类型的数据进行读写操作时，会产生一个Write Barrier暂时中断写操作，检查Reference引用的对象是否处于同一个Region之中（在分代的例子中就是检查是否老年代中的对象引用新生代中的对象），如果是，便通过CardTable把相关引用信息记录到被引用对象所属的Region的Remembered Set之中
  - 当进行垃圾回收时，在GC根节点的枚举范围中加入Remembered Set即可保证不对全堆扫描也不会有遗漏
- G1收集器运行流程
  - 初始标记
    - 初始标记阶段仅仅只是标记一下GC Roots能直接关联到的对象，并且修改TAMS的值，让下一阶段的用户程序并发运行时，能在正确可用的Region中创建对象
    - 需要停顿线程，但是耗时很短
  - 并发标记
    - 并发标记是从GC Roots开始对堆中的对象进行可达性分析，找出存活的对象
    - 耗时较长，但是可以与用户线程并发执行
  - 最终标记
    - 最终标记阶段是为了修正并发标记期间因用户程序继续运行而导致标记产生变动的那一部分标记记录
    - 虚拟机将这段时间对象变化记录在线程Remembered Set Logs里面，最终标记阶段需要把Remembered Set Logs的数据合并到Remembered Set中，这阶段需要停顿线程，但是可以并行执行
  - 筛选回收
    - 首先对各个Region的回收价值和成本进行排序，根据用户所期望的GC停顿时间来指定回收计划

### 区分与搭配

- 年轻代：Serial、ParNew、Parallel Scavenge
- 老年代：Serial Old、Parallel Old、CMS
- G1收集器年轻代老年代都可用
- 搭配使用：
  - Serial - Serial Old、Serial - CMS
  - ParNew - Serial Old、ParNew - CMS
  - Parallel Scavenge - Parallel Old、Parallel Scavenge - Serial Old

## GC日志

## 垃圾收集器参数总结

- -XX:UseSerialGC
  - 虚拟机运行在Client模式下的默认值，打开次开关后，使用Serial + Serial Old的收集器组合进行内存回收
- -XX:UseParNewGC
  - 打开此开关后，使用ParNew + Serial Old的收集器组合进行内存回收
- -XX:UseConcMarkSweepGC
  - 打开此开关后，使用ParNew + CMS + Serial Old收集器组合进行内存回收。Serial Old收集器将作为CMS收集器出现Concurrent Mode Failure失败后的后备收集器使用
- -XX:UseParallelGC
  - 虚拟机运行在Server模式下的默认值；打开开关后，使用Parallel Scavenge + Serial Old的收集器组合进行内存回收
- -XX:UseParallelOldGC
  - 打开此开关后，使用Parallel Scavenge + Parallel Old的收集器组合进行内存回收
- -XX:SurivivorRatio
  - 新生代中的Eden区域与Survivor区域的容量比值，默认值为8，代表Eden:Survivor=8:1
- -XX:PretenureSizeThreshold
  - 直接晋升到老年代的对象大小，设置这个参数后，大于这个参数的对象将直接在老年代分配
- -XX:MaxTenuringThreshold
  - 晋升到老年代的对象年龄。每个对象在坚持过一次Minor GC之后，年龄就增加1，当超过这个参数值时就进入老年代
- -XX:UseAdaptiveSizePolicy
  - 动态调整Java堆中各个区域的大小以及进入老年代的年龄
- -XX:HandlePromotionFailure
  - 是否允许分配担保失败，即老年代的剩余空间不足以应付新生代的整个Eden和Survivor区的所有对象存活的极端情况
- -XX:ParallelGCThreads
  - 设置并行GC时进行内存回收的线程数
- -XX:GCTimeRatio
  - GC时间占总时间的比率，默认值是99，即允许1%的GC时间，仅在使用Parallel Scavenge收集器时生效
- -XX:MaxGCPauseMillis
  - 设置GC的最大停顿时间，仅在使用Parallel Scavenge收集器时生效
- -XX:CMSInitiatingOccupancyFraction
  - 设置CMS收集器在老年代空间被使用多少后触发垃圾收集。默认值为68%，仅在使用CMS收集器时生效
- -XX:UseCMSCompactAtFullCollection
  - 设置CMS收集器在完成垃圾收集后是否要进行一次内存碎片整理。仅在使用CMS收集器生效
- -XX:CMSFullGCsBeforeCompaction
  设置CMS收集器在进行若干次垃圾收集后再启动一次内存碎片整理。仅在使用CMS收集器生效。

## 内存分配与回收策略

- 对象优先在Eden区中分配。当Eden区没有足够的空间进行分配时，虚拟机将发起一次Minor GC
- 大对象直接进入老年代
- 长期存活的对象将进入老年代，通过对象年龄判定，可以设置阈值
- 动态对象年龄判定
  - 虚拟机并不是永远地要求对象的年龄必须达到了MaxTenuringThreshold才能晋升老年代
  - 如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代，无须等到MaxTenuringThreshold中要求的年龄
- 空间分配担保
  - 在发生Minor GC之前，虚拟机会先检查老年代最大可用的连续空间是否大于新生代所有对象总空间
  - 如果这个条件成立，那么Minor GC可以确保是安全的
  - 如果这个调剂不成立，则虚拟机会查看XX:HandlePromotionFailure设置值是否运行担保失败
  - 如果运行，那么会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小
  - 如果大于，将尝试一次Minor GC，尽管这次Minor GC是有风险的
  - 如果小于，或者XX:HandlePromotionFailure设置不允许冒险，那这时也要改为进行一次Full GC
  - 新生代使用复制收集算法，但为了内存利用率，只是用其中一个Survivor空间来作为轮换备份，因此当出现大量对象在Minor GC后仍然存活的情况（最极端的情况是内存回收后新生代中所有对象都存活），就需要老年代分配担保，把Survivor无法容纳的对象直接进入老年代，前提是老年代还有容纳这些对象的剩余空间，一共有多少对象会活下来在实际完成内存回收之前是无法明确知道的，所以只好取之前每一次回收晋升到老年代容量的平均大小值作为经验值，与老年代的剩余空间进行比较，决定是否进行Full GC来让老年代腾出更多空间
  - 如果某次Minor GC存活后的对象突增，远远高于平均值，依然会导致担保失败，那就只好在失败后重新发起一次Full GC
  - 大部分情况还是会将XX:HandlePromotionFailure打开，避免Full GC过于频繁。

## Jdk监控和故障处理工具

- jps：JVM Process Status Tool，显示指定系统内所有的HotSpot虚拟机进程
- jstat：JVM Statistics Monitoring Tool，用于收集HotSpot虚拟机各方面的运行数据
- jinfo：Configuration Info for Java，显示虚拟机配置信息
- jmap：Memory Map for java，生成虚拟机内存快照
- jhat：JVM Heap Dump Brower，用于分析headdump文件，它会建立一个HTTP/HTML服务器，让用户可以在浏览器上查看分析结果
- jstack：Stack Trace for Java，显示虚拟机的线程快照

## 类加载机制

- 类的生命周期

  - 加载
    - 加载时机
      - 创建类的实例
      - 访问某个类或接口的静态变量，或者对该变量赋值
      - 调用类的静态方法
      - 反射 Class.forName("com.")
      - 初始化一个类的子类
      - Java虚拟机启动被标注为启动类的类
  - 连接
    - 验证：确保被加载的类的正确性
    - 准备：为类的静态变量分配内存，并将其初始化为默认值
    - 解析：把类中的符号引用转换为直接引用
  - 初始化：为类的静态变量赋予正确的初始值
    - 当一个类初始化时，会先初始化父类，依次类推
  - 使用
  - 卸载

- 常量在编译阶段会存入到调用这个常量的方法所在类的常量池中，调用类并没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化。将常量存放到了MyTest的常量池中，之后MyTest和MyParent没有任何关系，即使把MyParent的class文件删除也不影响执行

  - ```java
    public class MyTest {
    			public static void main(String[] args) {
    				System.out.println(MyParent.str);
    			}
    		}
    		class MyParent {
    			public static final String str = "hello world";
    			public static final String str2 = UUID.randomUUID().toString; // 如果引用str2，那么MyParent会初始化，因为编译期无法确定
    			static {
    				System.out.println("MyParent static block");
    			}
    		}
    ```

- 当一个接口初始化时，并不要求其父接口完成初始化

- 类与类加载器

  - 对于任意一个类，都需要由加载它的类加载器和这个类本身一同确立其在Java虚拟机中的唯一性，每一个类加载器，都拥有一个独立的名称空间
  - 比较两个类是否相等，只有在这两个类是有同一个类加载器加载的前提下才有意义，否则，即使这两个类来源于同一个Class文件，被同一个虚拟机加载，只要加载它们的类加载器不同，那这两个类就一定不相等
  - 这里的相等是指Class对象的equals()方法、isAssignableFrom()方法、isInstance()方法的返回结果，也包括使用instanceof关键字做对象所属关系判定等情况

- 双亲委派模式

  - 启动类加载器，使用C++语言实现，是虚拟机自身的一部分

  - 其他类加载器，都是继承自抽象类java.lang.ClassLoader

  - 启动类加载器（Bootstrap ClassLoder）

    - 负责加载<JAVA_HOME>\lib目录中的，或者被-Xbootclasspath参数所指定的路径中的，并且是虚拟机识别的类库。启动类加载器无法被Java程序直接引用，用户在编写自定义类加载器时，如果需要把加载请求委派给引导类加载器，那直接使用null替代即可

  - 扩展类加载器（Extension ClassLoader）

    - 这个类加载器由sun.misc.Launcher$ExtClassLoader实现，负责加载<JAVA_HOME>\lib\ext目录中的，或者被java.ext.dirs系统变量所指定的路径中的所有类库，开发者可以直接使用扩展类加载器

  - 应用类加载器（Application ClassLoader）

    - 这个类加载器由sum.misc.Launcher$Application ClassLoader实现
    - 由于这个类加载器是ClassLoader中的getSystemClassLoader()方法的返回值，所以也称为系统类加载器
    - 它负责加载用户路径（classpath）上所指定的类库，开发者可以直接使用这个类加载器

  - 双亲委派模型的工作过程：

    - 如果一个类加载器收到了类加载器的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一层次的类加载器都是如此，因此所有的请求最终都会传送到顶层的启动类加载器中，只有当父类加载器反馈自己无法完成这个加载请求时，子加载器才会尝试自己去加载

  - 破坏双亲委派模型

    - 举例：JNDI服务，JNDI现在已经是Java的标准服务了，它的代码有启动类加载器加载，它需要由独立厂商实现并部署在应用程序的ClassPath下的JNDI接口提供者（SPI，Service Provider Interface）的代码，但启动类加载器不可能认识这些代码，怎么办？

  - 线程上下文类加载器（Thread Context ClassLoader）

    - 默认线程上下文类加载器为 系统类加载（App ClassLoader）

    - 在双亲委托模型下，类加载器是由上至下的，即下层的类加载器会委托上层进行加载。但是对于SPI来说，有些接口是Java核心库所提供的，而Java核心库是由启动类加载器来加载的，而这些接口的实现却来自于不同的jar包（厂商提供），Java的启动类加载器是不会加载其他来源的jar包，这样传统的双亲委托模型就无法满足SPI的要求，而通过给当前线程设置上下文类加载器，就可以有设置的上下文类加载器来实现对于接口实现类的加载

    - 这个类加载器可以通过java.lang.Thread类的setContextClassLoader()方法进行设置，如果线程创建时还未设置，它将会从父线程继承一个，如果应用程序的全局范围内都没有设置过的话，那这个类加载器就是默认类加载器

    - JNDI服务使用这个线程上下文加载器去加载所需要的SPI代码，也就是父类加载器请求子类加载器去完成类加载的动作

    - Java中所有涉及SPI的加载动作基本都采用这种方式，例如：JNDI、JDBC、JCE、JAXB和JBI等

    - 线程上下文类加载器的一般使用模式（获取 - 使用 - 还原）

      - ```java
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
        		try {
        			// 自定义的ClassLoader，设置以后 下面的方法就会使用自定义类加载器
        			// 记得还原类加载器，这样不影响后面的操作
        			Thread.currentThread().setContextClassLoader(targetClassLoader); 
        			myMethod();
        		} finally {
        			Thread.currentThread().setContextClassLoader(classLoader);
        		}
        ```

- ServiceLoader

  - MATA-INF/services
  - 通过使用线程上下文类加载器来实现

- 类加载器的命名空间

  - 每个类有自己的命名空间
  - 如果自定义类加载器加载classpath下的类，仍然用的是App ClassLoader类加载器，不会使用自定义的

- 类加载路径

  - System.getProperty("sum.boot.class.path");
  - System.getProperty("java.ext.dirs");
  - System.getProperty("java.class.path");
  - 如果将自定义类放入到根类加载器中，那么这个类就会被根类加载器加载

- Class.forName()和ClassLoader都可以对类进行加载

  - ClassLoader就是遵循双亲委派模型最终调用启动类加载器的类加载器，实现的功能是“通过一个类的全限定名来获取描述此类的二进制字节流”，获取到二进制流后放到JVM中。Class.forName()方法实际上也是调用的CLassLoader来实现的。最后调用的方法是forName0这个方法，在这个forName0方法中的第二个参数被默认设置为了true，这个参数代表是否对加载的类进行初始化，设置为true时会类进行初始化，代表会执行类中的静态代码块，以及对静态变量的赋值等操作。也可以调用Class.forName(String name, boolean initialize,ClassLoader loader)方法来手动选择在加载类的时候是否要对类进行初始化
  - JDBC时通常是使用Class.forName()方法来加载数据库连接驱动。这是因为在JDBC规范中明确要求Driver(数据库驱动)类必须向DriverManager注册自己
  - 我们看到Driver注册到DriverManager中的操作写在了静态代码块中，这就是为什么在写JDBC时使用Class.forName()的原因了

## JVM内存调优参数

- MetaspaceSize
  - 初始化的Metaspace大小，控制元空间发生GC的阈值。GC后，动态增加或降低MetaspaceSize。在默认情况下，这个值大小根据不同的平台在12M到20M浮动。使用Java -XX:+PrintFlagsInitial命令查看本机的初始化参数
- MaxMetaspaceSize
  - 限制Metaspace增长的上限，防止因为某些情况导致Metaspace无限的使用本地内存，影响到其他程序。在本机上该参数的默认值为4294967295B（大约4096MB）
- MinMetaspaceFreeRatio
  - 当进行过Metaspace GC之后，会计算当前Metaspace的空闲空间比，如果空闲比小于这个参数（即实际非空闲占比过大，内存不够用），那么虚拟机将增长Metaspace的大小。默认值为40，也就是40%
  - 设置该参数可以控制Metaspace的增长的速度，太小的值会导致Metaspace增长的缓慢，Metaspace的使用逐渐趋于饱和，可能会影响之后类的加载。而太大的值会导致Metaspace增长的过快，浪费内存
- MaxMetasaceFreeRatio
  - 当进行过Metaspace GC之后， 会计算当前Metaspace的空闲空间比，如果空闲比大于这个参数，那么虚拟机会释放Metaspace的部分空间。默认值为70，也就是70%
- MaxMetaspaceExpansion
  - Metaspace增长时的最大幅度。在本机上该参数的默认值为5452592B（大约为5MB）
- MinMetaspaceExpansion
  - Metaspace增长时的最小幅度。在本机上该参数的默认值为340784B（大约330KB为）
- -Xmx：堆的最大值 -Xms：堆的最小值 -Xmn：设置新生代大小
- -XX:NewRatio：
  - 新生代(eden+2*s)和老年代（不包含永久区）的比值表示 新生代:老年代=1:4，即年轻代占堆的1/5
- -XX:SurvivorRatio
  - 设置两个Survivor区和eden的比表示两个Survivor:eden=2:8，即一个Survivor占年轻代的1/10
- -XX:OnOutOfMemoryError：
  - 可以在内存溢出的时候执行一个脚本，比如可以报警发送一个邮件，重启程序等
  - 例如：-XX:OnOutOfMemoryError=D:/tools/jdk1.7/bin/printstack.bat %p
- -XX:+HeapDumpOnOutOfMemoryError：OOM时导出堆到文件
- -XX:+HeapDumpPath：导出OOM的路径
  - 上面两个例子：-Xmx20m -Xms5m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=d:/a.dump
- -XX:PermSize：永久区的初始大小
- -XX:MaxPermSiz：永久区的最大空间
- -XX:+DisableExplicitGC来禁止RMI调用System.gc
- -XX:MaxDirectMemorySize：直接内存大小

## JIT即时编译

- 热点代码

  - 当虚拟机发现某个方法或代码块的运行特别频繁时，就会把这些代码认定为“热点代码”
  - 被多次调用的方法
    - 一个方法被调用得多了，方法体内代码执行的次数自然就多，成为“热点代码”是理所当然的
  - 被多次执行的循环体
    - 一个方法只被调用过一次或少量的几次，但是方法体内部存在循环次数较多的循环体，这样循环体的代码也被重复执行多次，因此这些代码也应该认为是“热点代码”

- 热点探测

  - 基于采样的热点探测
    - 采用这种方法的虚拟机会周期性地检查各个线程的栈顶如果发现某个（或某些）方法经常出现在栈顶，那这个方法就是“热点方法”
    - 优点：实现简单高效，容易获取方法调用关系（将调用堆栈展开即可）
    - 缺点：不精确，容易因为因为受到线程阻塞或别的外界因素的影响而扰乱热点探测
  - 基于计数器的热点探测：
    - 采用这种方法的虚拟机会为每个方法（甚至是代码块）建立计数器，统计方法的执行次数，如果次数超过一定的阈值就认为它是“热点方法”
    - 优点：统计结果精确严谨
    - 缺点：实现麻烦，需要为每个方法建立并维护计数器，不能直接获取到方法的调用关系
  - HotSpot使用第二种 - 基于计数器的热点探测方法

- 确定了检测热点代码的方式，如何计算具体的次数呢？

  - 方法调用计数器：
    - 这个计数器用于统计方法被调用的次数。默认阈值在 Client 模式下是 1500 次，在 Server 模式下是 10000 次
  - 回边计数器：
    - 统计一个方法中循环体代码执行的次数
  - 达到计数器的阈值会触发即时编译
  - 两个计数器的协作（这里讨论的是方法调用计数器的情况）：
    - 当一个方法被调用时，会先检查该方法是否存在被 JIT（后文讲解） 编译过的版本，如果存在，则优先使用编译后的本地代码来执行。如果不存在已被编译过的版本，则将此方法的调用计数器加 1，然后判断方法调用计数器与回边计数器之和是否超过方法调用计数器的阈值。如果已经超过阈值，那么将会向即时编译器提交一个该方法的代码编译请求
  - 当编译工作完成之后，这个方法的调用入口地址就会被系统自动改成新的，下一次调用该方法时就会使用已编译的版本

- 什么是字节码、机器码、本地代码？

  - 字节码是指平常所了解的 .class 文件，Java 代码通过 javac 命令编译成字节码
  - 机器码和本地代码都是指机器可以直接识别运行的代码，也就是机器指令
  - 字节码是不能直接运行的，需要经过 JVM 解释或编译成机器码才能运行
  - 为什么 Java 不直接编译成机器码，这样不是更快吗？
    - 机器码是与平台相关的，也就是操作系统相关，不同操作系统能识别的机器码不同，如果编译成机器码那岂不是和 C、C++差不多了，不能跨平台，Java 就没有那响亮的口号 “一次编译，到处运行”
    - 之所以不一次性全部编译，是因为有一些代码只运行一次，没必要编译，直接解释运行就可以。而那些“热点”代码，反复解释执行肯定很慢，JVM在运行程序的过程中不断优化，用JIT编译器编译那些热点代码，让他们不用每次都逐句解释执行

- 什么是 JIT？

  - 为了提高热点代码的执行效率，在运行时，虚拟机将会把这些代码编译成与本地平台相关的机器码，并进行各种层次的优化，完成这个任务的编译器称为即时编译器（Just In Time Compiler），简称 JIT 编译器

- 什么是编译和解释？

  - 编译器：把源程序的每一条语句都编译成机器语言,并保存成二进制文件,这样运行时计算机可以直接以机器语言来运行此程序,速度很快
  - 解释器：只在执行程序时,才一条一条的解释成机器语言给计算机来执行,所以运行速度是不如编译后的程序运行的快的
  - 通过javac命令将 Java 程序的源代码编译成 Java 字节码，即我们常说的 class 文件。这是我们通常意义上理解的编译
  - 字节码并不是机器语言，要想让机器能够执行，还需要把字节码翻译成机器指令。这个过程是Java 虚拟机做的，这个过程也叫编译。是更深层次的编译。（实际上就是解释，引入 JIT 之后也存在编译）
  - Java 需要将字节码逐条翻译成对应的机器指令并且执行，这就是传统的 JVM 的解释器的功能，正是由于解释器逐条翻译并执行这个过程的效率低，引入了 JIT 即时编译技术
  - 不管是解释执行，还是编译执行，最终执行的代码单元都是可直接在真实机器上运行的机器码，或称为本地代码

- 为何 HotSpot 虚拟机要使用解释器与编译器并存的架构？

  - 解释器与编译器两者各有优势
  - 解释器：
    - 当程序需要迅速启动和执行的时候，解释器可以首先发挥作用，省去编译的时间，立即执行
  - 编译器：
    - 在程序运行后，随着时间的推移，编译器逐渐发挥作用，把越来越多的代码编译成本地代码之后，可以获取更高的执行效率

- 在 HotSpot 中，解释器和 JIT 即时编译器是同时存在的，他们是 JVM 的两个组件。对于不同类型的应用程序，用户可以根据自身的特点和需求，灵活选择是基于解释器运行还是基于 JIT 编译器运行。HotSpot 为用户提供了几种运行模式供选择，可通过参数设定，分别为：解释模式、编译模式、混合模式，HotSpot 默认是混合模式，需要注意的是编译模式并不是完全通过 JIT 进行编译，只是优先采用编译方式执行程序，但是解释器仍然要在编译无法进行的情况下介入执行过程。

- 逃逸分析

  - 当一个对象在方法中被定义后，它可能被外部方法所引用，例如作为调用参数传递到其他方法中，称为方法逃逸。甚至可能被外部线程访问到，譬如赋值给类变量或可以在其他线程中访问的实例变量，称为线程逃逸
  - 如果能证明一个对象不会逃逸到方法或线程之外，也就是别的方法或线程无法通过任何途径访问到这个对象，则可以为这个变量进行一些高效的优化：
    - 栈上分配：
      - 将不会逃逸的局部对象分配到栈上，那对象就会随着方法的结束而自动销毁，减少垃圾收集系统的压力
    - 同步消除：
      - 如果该变量不会发生线程逃逸，也就是无法被其他线程访问，那么对这个变量的读写就不存在竞争，可以将同步措施消除掉（同步是需要付出代价的）
    - 标量替换：
      - 如果一个对象不会被外部访问，并且对象可以被拆散的话，真正执行时可能不创建这个对象，而是直接创建它的若干个被这个方法使用到的成员变量来代替。这种方式不仅可以让对象的成员变量在栈上分配和读写，还可以为后后续进一步的优化手段创建条件

- JIT 运行模式

  - 客户端&服务端模式（-client(c1)、-server(c2)）
    - -client时会采用一个叫做C1的轻量级编译器，-server时会采用一个叫做C2的重量级编译器。-server 相对于 -client来说编译更彻底，性能也更好。java -version查看会有一个-server或者-client的标示
    - 以通过在启动java 的命令中，传入参数(-client 或 -server)来选择编译器（C1或C2）。这两种编译器的最大区别就是，编译代码的时间点不一样。client编译器（C1）会更早地对代码进行编译，因此，在程序刚启动的时候，client编译器比server编译器执行得更快。而server编译器会收集更多的信息，然后才对代码进行编译优化，因此，server编译器最终可以产生比client编译器更优秀的代码
  - JVM为什么要将编译器分为client和server，为什么不在程序启动时，使用client编译器，在程序运行一段时间后，自动切换为server编译器？
    - 这种技术是存在的，一般称之为： tiered compilation。Java7 和Java 8可以使用选项-XX:+TieredCompilation来打开（-server选项也要打开）
    - 在Java 8中，-XX:+TieredCompilation默认是打开的

- JVM JIT参数

  - -Xint

    - 全部使用字节码解释执行，这个是最慢的，慢的惊人，通常要比其他方式慢一个数量级左右

  - -Xcomp

    - 纯编译执行，全部被编译成机器码执行，速度是很快的，但是存在一个缺陷，-Xcomp的策略过于简单

  - Xmixed

    - 是一种自适应的方式，有的地方解释执行，有的地方编译执行，具体的策略要依据profile的统计分析来判断

  - -XX:+TieredCompilation

    - 除了纯编译和默认的mixed之外，jvm 从jdk6u25 之后，引入了分层编译。HotSpot 内置两种编译器，分别是client启动时的c1编译器和server启动时的c2编译器，c2在将代码编译成机器代码的时候需要搜集大量的统计信息以便在编译的时候进行优化，因此编译出来的代码执行效率比较高，代价是程序启动时间比较长，而且需要执行比较长的时间，才能达到最高性能；与之相反， c1的目标是使程序尽快进入编译执行的阶段，所以在编译前需要搜集的信息比c2要少，编译速度因此提高很多，但是付出的代价是编译之后的代码执行效率比较低，但尽管如此，c1编译出来的代码在性能上比解释执行的性能已经有很大的提升，所以所谓的分层编译，就是一种折中方式，在系统执行初期，执行频率比较高的代码先被c1编译器编译，以便尽快进入编译执行，然后随着时间的推移，执行频率较高的代码再被c2编译器编译，以达到最高的性能

      作者：worldcbf
      链接：https://www.jianshu.com/p/318617435789
      来源：简书
      著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

- 为什么不全都去编译执行呢？

  - 因为我们程序中有挺大一部分调用频次比较低，并且一次编译的花销要比解释执行大不少，还会有额外的操作。这时去使用JIT就得不偿失了

# 设计模式

- 设计模式原则
  - 单一职责原则
  - 接口隔离原则
    - Spring的接口拆得很细，类可以实现多个接口，可以按照自己的业务来实现不同的接口
  - 依赖倒转原则
    - 高层模块不应该依赖底层模块，二者都应该依赖其抽象
    - 抽象不应该依赖细节，细节应该依赖抽象
    - 核心思想：面向接口编程
    - 使用接口或者抽象类的目的是指定好规范，而不涉及具体任何的操作，把展现细节的任务交给具体的实现类去完成
    - 底层模块尽量都要有抽象类或者接口，或者两者都有，程序更好扩展
    - 变量的声明类型尽量是抽象类或接口，这样我们的变量引用和实际对象间，就存在一个缓冲层，利于程序扩展和优化
    - 实现方式
      - 通过接口方式传递  void play(IPlay play); 这是接口中的一个方法，方法中再传入接口
      - 通过构造方法传递  private IPlay play;  这是类中的一个属性
  - 里式替换原则
    - 继承实际上让两个类的耦合性增强了，在适当的情况下通通过组合来替换继承
  - 开闭原则
    - 对扩展开放，对修改关闭；用抽象构建框架，用实现扩展细节
  - 迪米特法原则
    - 只与朋友通信；成员变量、方法参数、方法返回值中的类称为直接的朋友，而出现在局部变量的类不是直接的朋友，陌生的类最好不要以局部变量表的形式出现在类的内部
    - 被依赖的类不管有多复杂，都尽量封装在类的内部，对外除了提供的public方法，不对外泄露任何信息
  - 合成复用原则
    -  尽量使用合成/聚合的方式，而不是使用继承

# 网络编程

## OSI七层模型

- 应用层
  - HTTP、FTP、SMTP
- 表示层
- 会话层
  - SMTP、DNS
- 传输层
  - TCP、UDP
- 网络层
  - IP、ICMP、ARP
- 数据链路层
- 物理层

## TCP/IP四层模型

- 应用层
- 传输层
- 网络层
- 数据链路层

## TCP粘包和拆包

- 粘包
  - 服务端一共就读到一个数据包，这个数据包包含客户端发出的两条消息的完整信息，这个时候基于之前逻辑实现的服务端就蒙了，因为服务端不知道第一条消息从哪儿结束和第二条消息从哪儿开始，这种情况其实是发生了TCP粘包
- 拆包
  - 服务端一共收到了两个数据包，第一个数据包只包含了第一条消息的一部分，第一条消息的后半部分和第二条消息都在第二个数据包中，或者是第一个数据包包含了第一条消息的完整信息和第二条消息的一部分信息，第二个数据包包含了第二条消息的剩下部分，这种情况其实是发送了TCP拆，因为发生了一条消息被拆分在两个包里面发送了，同样上面的服务器逻辑对于这种情况是不好处理的
- TCP为提高性能，发送端会将需要发送的数据发送到缓冲区，等待缓冲区满了之后，再将缓冲中的数据发送到接收方。同理，接收方也有缓冲区这样的机制，来接收数据
- 发生TCP粘包、拆包主要是由于下面一些原因：
  - 应用程序写入的数据大于套接字缓冲区大小，这将会发生拆包
  - 应用程序写入数据小于套接字缓冲区大小，网卡将应用多次写入的数据发送到网络上，这将会发生粘包
  - 进行mss（最大报文长度）大小的TCP分段，当TCP报文长度-TCP头部长度>mss的时候将发生拆包
  - 接收方法不及时读取套接字缓冲区数据，这将发生粘包
- 如何解决拆包粘包？
  - 既然知道了tcp是无界的数据流，且协议本身无法避免粘包，拆包的发生，那我们只能在应用层数据协议上，加以控制。通常在制定传输数据时，可以使用如下方法：
    - 使用带消息头的协议、消息头存储消息开始标识及消息长度信息，服务端获取消息头的时候解析出消息长度，然后向后读取该长度的内容
      - 可以制定，首部固定10个字节长度用来保存整个数据包长度，位数不够补0的数据协议
      - 0000000036{"type":"message","content":"hello"}
      - 可以制定，首部4字节网络字节序unsigned int，标记整个包的长度
      - {"type":"message","content":"hello all"}
      - 其中首部四字节*号代表一个网络字节序的unsigned int数据，为不可见字符，紧接着是Json的数据格式的包体数据。
    - 设置定长消息，服务端每次读取既定长度的内容作为一条完整消息
    - 设置消息边界，服务端从网络流中按消息编辑分离出消息内容
      - 假设区分数据边界的标识为换行符"\n"（注意请求数据本身内部不能包含换行符），数据格式为Json，例如下面是一个符合这个规则的请求包
      - {"type":"message","content":"hello"}\n
      - 注意上面的请求数据末尾有一个换行字符(在PHP中用双引号字符串"\n"表示)，代表一个请求的结束。

## TCP

- TCP是全双工的，即客户端在给服务器端发送信息的同时，服务器端也可以给客户端发送信息。而半双工的意思是A可以给B发，B也可以给A发，但是A在给B发的时候，B不能给A发，即不同时，为半双工。 单工为只能A给B发，B不能给A发； 或者是只能B给A发，不能A给B发
- 如果已经建立了连接，但是客户端突然出现故障了怎么办？
  - TCP还设有一个保活计时器，显然，客户端如果出现故障，服务器不能一直等下去，白白浪费资源。服务器每收到一次客户端的请求后都会重新复位这个计时器，时间通常是设置为2小时，若两小时还没有收到客户端的任何数据，服务器就会发送一个探测报文段，以后每隔75秒钟发送一次。若一连发送10个探测报文仍然没反应，服务器就认为客户端出了故障，接着就关闭连接

## TCP三次握手

- 第一次握手
  - SYN=1  seq=J
  - 建立连接时，客户端发送syn包（syn=j）到服务器，并进入SYN_SENT状态，等待服务器确认；SYN：同步序列编号（Synchronize Sequence Numbers）
- 第二次握手
  - SYN=1 ACK=1 ack=J+1 seq=K
  - 服务器收到syn包，必须确认客户的SYN（ack=j+1），同时自己也发送一个SYN包（syn=k），即SYN+ACK包，此时服务器进入SYN_RECV状态
- 第三次握手
  - ACK=1 ack=K+1
  - 客户端收到服务器的SYN+ACK包，向服务器发送确认包ACK(ack=k+1），此包发送完毕，客户端和服务器进入ESTABLISHED（TCP连接成功）状态，完成三次握手

- 为什么要发送特定的数据包，随便发不行吗？
  - 三次握手的另外一个目的就是确认双方都支持TCP，告知对方用TCP传输
  - 第一次握手：Server 猜测Client可能要建立TCP请求，但不确定，因为也可能是Client乱发了一个数据包给自己
  - 第二次握手：通过ack=J+1，Client知道Server是支持TCP的，且理解了自己要建立TCP连接的意图
  - 第三次握手：通过ack=K+1，Server知道Client是支持TCP的，且确实是要建立TCP连接
- 为什么三次握手，两次不行吗？
  - 假如 A 发出的一个连接请求报文段并没有丢失，而是在某个网络节点滞留了，以致延误到连接释放以后的某个时间节点才到达B。本来这是一个早已经失效的报文段，但 B 收到此失效的连接请求报文段后，就误以为时 A 又发出一次新的连接请求。于是就向 A 发出确认报文段，同意建立连接。假定不采用三次握手，那么只要 B 发出确认，新的连接就建立了
  - 由于现在 A 并没有发出建立连接的请求，因此不会理睬 B 的确认，也不会向 B 发送数据，但 B 却以为新的运输连接已经建立，并一直等待 A 发来数据，B 的许多资源就这样浪费了
  - 采用三次握手的办法可以防止上述问题的发生，例如在刚才的情况下，A 不会向 B 的确认发出确认，B 由于收不到确认，就知道 A 并没有要求建立连接

## TCP四次挥手

- 由于TCP连接时全双工的，因此，每个方向都必须要单独进行关闭，这一原则是当一方完成数据发送任务后，发送一个FIN来终止这一方向的连接，收到一个FIN只是意味着这一方向上没有数据流动了，即不会再收到数据了，但是在这个TCP连接上仍然能够发送数据，直到这一方向也发送了FIN。首先进行关闭的一方将执行主动关闭，而另一方则执行被动关闭
- 第一次挥手：Client发送一个FIN，用来关闭Client到Server的数据传送，Client进入FIN_WAIT_1状态
- 第二次挥手：Server收到FIN后，发送一个ACK给Client，确认序号为收到序号+1（与SYN相同，一个FIN占用一个序号），Server进入CLOSE_WAIT状态
- 第三次挥手：Server发送一个FIN，用来关闭Server到Client的数据传送，Server进入LAST_ACK状态
- 第四次挥手：Client收到FIN后，Client进入TIME_WAIT状态，接着发送一个ACK给Server，确认序号为收到序号+1，Server进入CLOSED状态，完成四次挥手
- 为什么建立连接是三次握手，而关闭连接却是四次挥手呢？
  - 因为服务端在LISTEN状态下，收到建立连接请求的SYN报文后，把ACK和SYN放在一个报文里发送给客户端。而关闭连接时，当收到对方的FIN报文时，仅仅表示对方不再发送数据了但是还能接收数据，己方也未必全部数据都发送给对方了，所以己方可以立即close，也可以发送一些数据给对方后，再发送FIN报文给对方来表示同意现在关闭连接，因此，己方ACK和FIN一般都会分开发送
  - 因为当Server端收到Client端的SYN连接请求后，可以直接发送SYN+ACK报文，其中ACK报文是用来应答的，SYN报文是用来同步的，但是关闭连接时，当Server端收到FIN报文时，很可能并不会立即关闭Socket，所以只能先回复一个ACK报文，高数Client端，“你发送的FIN报文我收到了”，只有等我Server端所有的报文都发送完了，我才能发送FIN报文，因此不能一起发送，故需要四次挥手

## HTTPS

- HTTP 是明文传输的，没有加密，不安全，所以就出现了HTTPS
- HTTPS即加密的HTTP，HTTPS并不是一个新协议，而是HTTP+SSL（TLS）。原本HTTP先和TCP（假定传输层是TCP协议）直接通信，而加了SSL后，就变成HTTP先和SSL通信，再由SSL和TCP通信，相当于SSL被嵌在了HTTP和TCP之间
- 协商过程使用非对称加密算法，数据传输使用共享密码加密算法
- 共享密钥加密（对称密钥加密）：
  - 加密和解密同用一个密钥。加密时就必须将密钥传送给对方，那么如何安全的传输呢？
- 公开密钥加密（非对称密钥加密）：
  - 公开密钥加密使用一对非对称的密钥。一把叫做私有密钥，一把叫做公开密钥。私有密钥不能让其他任何人知道，而公开密钥则可以随意发布，任何人都可以获得。使用此加密方式，发送密文的一方使用公开密钥进行加密处理，对方收到被加密的信息后，再使用自己的私有密钥进行解密。利用这种方式，不需要发送用来解密的私有密钥，也不必担心密钥被攻击者窃听盗走
  - 但由于公开密钥比共享密钥要慢，所以我们就需要综合一下他们两者的优缺点，使他们共同使用，而这也是HTTPS采用的加密方式。在交换密钥阶段使用公开密钥加密方式，之后建立通信交换报文阶段则使用共享密钥加密方式
  -  这里就有一个问题，如何证明公开密钥是货真价实的公开密钥。如，正准备和某台服务器建立公开密钥加密方式下的通信时，如何证明收到的公开密钥就是原本预想的那台服务器发行的公开密钥。或许在公开密钥传输过程中，真正的公开密钥已经被攻击者替换掉了。为了解决这个问题，可以使用由数字证书认证机构（CA，Certificate Authority）和其他相关机关颁发的公开密钥证书
  - 接收到证书的客户端可以使用数字证书认证机构的公开密钥，对那张证书上的数字签名进行验证，一旦验证通过，客户端便可以明确两件事：
    - 认证服务器的公开密钥的是真实有效的数字证书认证机构
    - 服务器的公开密钥是值得信赖的
- 工作流程
  - 认证服务器
    - 浏览器内置一个受信任的CA机构列表，并保存了这些CA机构的证书。第一阶段服务器会提供经CA机构认证颁发的服务器证书，如果认证该服务器证书的CA机构，存在于浏览器的受信任CA机构列表中，并且服务器证书中的信息与当前正在访问的网站（域名等）一致，那么浏览器就认为服务端是可信的，并从服务器证书中取得服务器公钥，用于后续流程。否则，浏览器将提示用户，根据用户的选择，决定是否继续。当然，我们可以管理这个受信任CA机构列表，添加我们想要信任的CA机构，或者移除我们不信任的CA机构
  - 认证服务器
    - 浏览器内置一个受信任的CA机构列表，并保存了这些CA机构的证书。第一阶段服务器会提供经CA机构认证颁发的服务器证书，如果认证该服务器证书的CA机构，存在于浏览器的受信任CA机构列表中，并且服务器证书中的信息与当前正在访问的网站（域名等）一致，那么浏览器就认为服务端是可信的，并从服务器证书中取得服务器公钥，用于后续流程。否则，浏览器将提示用户，根据用户的选择，决定是否继续。当然，我们可以管理这个受信任CA机构列表，添加我们想要信任的CA机构，或者移除我们不信任的CA机构
  - 加密通讯
    - 此时客户端服务器双方都有了本次通讯的会话密钥，之后传输的所有Http数据，都通过会话密钥加密。这样网路上的其它用户，将很难窃取和篡改客户端和服务端之间传输的数据，从而保证了数据的私密性和完整性
- 加解密过程 非对称加密+对称加密
  - 用户在浏览器发起HTTPS请求（如 https://www.mogu.com/），默认使用服务端的443端口进行连接
  - HTTPS需要使用一套**CA数字证书**，证书内会附带一个**公钥Pub**，而与之对应的**私钥Private**保留在服务端不公开
  - 服务端收到请求，返回配置好的包含**公钥Pub**的证书给客户端
  - 客户端收到**证书**，校验合法性，主要包括是否在有效期内、证书的域名与请求的域名是否匹配，上一级证书是否有效（递归判断，直到判断到系统内置或浏览器配置好的根证书），如果不通过，则显示HTTPS警告信息，如果通过则继续
  - 客户端生成一个用于对称加密的**随机Key**，并用证书内的**公钥Pub**进行加密，发送给服务端
  - 服务端收到**随机Key**的密文，使用与**公钥Pub**配对的**私钥Private**进行解密，得到客户端真正想发送的**随机Key**
  - 服务端使用客户端发送过来的**随机Key**对要传输的HTTP数据进行对称加密，将密文返回客户端
  - 客户端使用**随机Key**对称解密密文，得到HTTP数据明文
  - 后续HTTPS请求使用之前交换好的**随机Key**进行对称加解
- 加解密过程
  - 用户在浏览器发起HTTPS请求
  - 服务端会返回CA数字证书，证书内包含公钥Pub，私钥存在服务端
  - 客户端严重证书是否是受信CA机构的，同时随机生成客户端RSA公私钥、会话密钥，然后使用服务端的公钥加密发送（客户端私钥、客户端会话密钥）
  - 服务端使用服务器私钥解密数据（得到 客户端公钥、客户端密钥），随机生成会话服务器密钥，然后使用客户端公钥加密发送（服务器会话密钥）
  - 客户端使用私钥解密服务器数据得到 服务器会话密钥
  - 客户端 使用客户端会话秘钥加密后的HTTP数据 发送到服务器
  - 服务器 通过之前的客户端会话密钥解密数据
  - 服务器 使用服务器会话密钥加密后的HTTP数据 发送到客户端
  - 客户端 使用之前的服务器会话密钥解密数据

## 多路复用IO

- select
  - 单个进程可监视的fd数量被限制，即能监听端口的大小有限
  - 对socket进行扫描时时线性扫描，即采用轮询的方法，效率较低
  - 需要维护一个用来存放大量fd数量的数据结构，这样会使得用户空间和内核空间在传递该结构时复制开销大
- poll
  - 大量的fd的数组被整体复制于用户态和内核地址空间之间，而不管这样的复制有没有意义
  - poll还有一个特点时“水平触发”，如果报告了fd后，没有被处理，那么下次poll时会再次报告该fd
- epoll
  - 没有最大并发连接限制，能打开的FD的上限远大于1024（1G的内存上能监听约10万个端口）
  - 效率提升，不是轮询的方式，不会随着FD数目的增加效率下降
  - 内存拷贝，利用mmap() 文件映射内存加速与内核空间的消息传递，即poll使用mmap减少复制开销

# Netty

## Netty 的特点是什么？

- 高并发
  - IO 多路复用、Reator模式、BossGroup、WorkGroup
- 传输快
  - Netty 的传输依赖于零拷贝特性，尽量减少不必要的内存拷贝，实现了更高效率的传输
- 封装好
  - Netty 封装了 NIO 操作的很多细节，提供了易于使用调用接口

## Netty 的零拷贝？

- Netty 的接收和发送 ByteBuffer 采用 DIRECT BUFFERS，使用堆外直接内存进行 Socket 读写，不需要进行字节缓冲区的二次拷贝。如果使用传统的堆内存（HEAP BUFFERS）进行 Socket 读写，JVM 会将堆内存 Buffer 拷贝一份到直接内存中，然后才写入 Socket 中。相比于堆外直接内存，消息在发送过程中多了一次缓冲区的内存拷贝
- Netty提供了组合Buffer对象，可以聚合多个ByteBuffer对象，用户可以像操作一个Buffer那样方便的对组合Buffer进行操作，避免了传统通过内存拷贝的方式将几个小Buffer合并成一个大的Buffer
- Netty的文件传输采用了transferTo方法，它可以直接将文件缓冲区的数据发送到目标Channel，避免了传统通过循环write方式导致的内存拷贝问题

## Netty 的线程模型？

- Netty通过Reactor模型基于多路复用器接收并处理用户请求，内部实现了两个线程池，boss线程池和work线程池，其中boss线程池的线程负责处理请求的accept事件，当接收到accept事件的请求时，把对应的socket封装到一个NioSocketChannel中，并交给work线程池，其中work线程池负责请求的read和write事件，由对应的Handler处理
- 单线程模型：所有I/O操作都由一个线程完成，即多路复用、事件分发和处理都是在一个Reactor线程上完成的。既要接收客户端的连接请求,向服务端发起连接，又要发送/读取请求或应答/响应消息。一个NIO 线程同时处理成百上千的链路，性能上无法支撑，速度慢，若线程进入死循环，整个程序不可用，对于高负载、大并发的应用场景不合适
- 多线程模型：有一个NIO 线程（Acceptor） 只负责监听服务端，接收客户端的TCP 连接请求；NIO 线程池负责网络IO 的操作，即消息的读取、解码、编码和发送；1 个NIO 线程可以同时处理N 条链路，但是1 个链路只对应1 个NIO 线程，这是为了防止发生并发操作问题。但在并发百万客户端连接或需要安全认证时，一个Acceptor 线程可能会存在性能不足问题
- 主从多线程模型：Acceptor 线程用于绑定监听端口，接收客户端连接，将SocketChannel 从主线程池的Reactor 线程的多路复用器上移除，重新注册到Sub 线程池的线程上，用于处理I/O 的读写等操作，从而保证mainReactor只负责接入认证、握手等操作


## Netty 高性能表现在哪些方面？

- 心跳
  - 对服务端：会定时清除闲置会话inactive(netty5)，对客户端:用来检测会话是否断开，是否重来，检测网络延迟，其中idleStateHandler类 用来检测会话状态
- 串行无锁化设计
  - 即消息的处理尽可能在同一个线程内完成，期间不进行线程切换，这样就避免了多线程竞争和同步锁。表面上看，串行化设计似乎CPU利用率不高，并发程度不够。但是，通过调整NIO线程池的线程参数，可以同时启动多个串行化的线程并行运行，这种局部无锁化的串行线程设计相比一个队列-多个工作线程模型性能更优
- 可靠性
  - 链路有效性检测：链路空闲检测机制，读/写空闲超时机制；内存保护机制：通过内存池重用ByteBuf;ByteBuf的解码保护；优雅停机：不再接收新消息、退出前的预处理操作、资源的释放操作
- Netty安全性
  - 支持的安全协议：SSL V2和V3，TLS，SSL单向认证、双向认证和第三方CA认证
- 高效并发编程的体现
  - volatile的大量、正确使用；CAS和原子类的广泛使用；线程安全容器的使用；通过读写锁提升并发性能。IO通信性能三原则：传输（AIO）、协议（Http）、线程（主从多线程）
- 流量整型的作用（变压器）
  - 防止由于上下游网元性能不均衡导致下游网元被压垮，业务流中断；防止由于通信模块接受消息过快，后端业务线程处理不及时导致撑死问题
- TCP参数配置
  - SO_RCVBUF和SO_SNDBUF：通常建议值为128K或者256K；SO_TCPNODELAY：NAGLE算法通过将缓冲区内的小封包自动相连，组成较大的封包，阻止大量小封包的发送阻塞网络，从而提高网络应用效率。但是对于时延敏感的应用场景需要关闭该优化算法

## Netty 高性能体现在哪些方面？

- 传输：IO模型在很大程度上决定了框架的性能，相比于bio，netty建议采用异步通信模式，因为nio一个线程可以并发处理N个客户端连接和读写操作，这从根本上解决了传统同步阻塞IO一连接一线程模型，架构的性能、弹性伸缩能力和可靠性都得到了极大的提升。正如代码中所示，使用的是NioEventLoopGroup和NioSocketChannel来提升传输效率
- 协议：采用什么样的通信协议，对系统的性能极其重要，netty默认提供了对Google Protobuf的支持，也可以通过扩展Netty的编解码接口，用户可以实现其它的高性能序列化框架
- 线程：netty使用了Reactor线程模型，但Reactor模型不同，对性能的影响也非常大，下面介绍常用的Reactor线程模型有三种，分别如下：
  - Reactor单线程模型：
    - 单线程模型的线程即作为NIO服务端接收客户端的TCP连接，又作为NIO客户端向服务端发起TCP连接，即读取通信对端的请求或者应答消息，又向通信对端发送消息请求或者应答消息。理论上一个线程可以独立处理所有IO相关的操作，但一个NIO线程同时处理成百上千的链路，性能上无法支撑，即便NIO线程的CPU负荷达到100%，也无法满足海量消息的编码、解码、读取和发送，又因为当NIO线程负载过重之后，处理速度将变慢，这会导致大量客户端连接超时，超时之后往往会进行重发，这更加重了NIO线程的负载，最终会导致大量消息积压和处理超时，NIO线程会成为系统的性能瓶颈
  - Reactor多线程模型：
    - 有专门一个NIO线程用于监听服务端，接收客户端的TCP连接请求；网络IO操作(读写)由一个NIO线程池负责，线程池可以采用标准的JDK线程池实现。但百万客户端并发连接时，一个nio线程用来监听和接受明显不够，因此有了主从多线程模型
  - 主从Reactor多线程模型：
    - 利用主从NIO线程模型，可以解决1个服务端监听线程无法有效处理所有客户端连接的性能不足问题，即把监听服务端，接收客户端的TCP连接请求分给一个线程池。因此，在代码中可以看到，我们在server端选择的就是这种方式，并且也推荐使用该线程模型。在启动类中创建不同的EventLoopGroup实例并通过适当的参数配置，就可以支持上述三种Reactor线程模型

## TCP 粘包/拆包的原因及解决方法？

- 消息定长：FixedLengthFrameDecoder类
- 包尾增加特殊字符分割：行分隔符类：LineBasedFrameDecoder或自定义分隔符类 ：DelimiterBasedFrameDecoder
- 将消息分为消息头和消息体：LengthFieldBasedFrameDecoder类。分为有头部的拆包与粘包、长度字段在前且有头部的拆包与粘包、多扩展头部的拆包与粘包

# SpringBoot

## 源码

# Spring

## 容器级别扩展接口

- BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry);
  - 动态添加、注册、移除Bean的BeanDefinition
- BeanFactoryPostProcessor#postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory);
  - 允许使用者修改容器中的bean definitions
  - 千万不要进行bean实例化，导致bean被提前实例化
  - 如果在postProcessBeanFactory 方法中执行了bean的初始化，那么会导致@AutoWired和@Resource无法注入bean？
    - 原因：@AutoWired起作用依赖AutowiredAnnotationBeanPostProcessor；@Resource依赖CommonAnnotationBeanPostProcessor；都是BeanPostProcessor，BeanPostProcessors在何处被spring invoke呢？参见registerBeanPostProcessors(beanFactory);在postProcessBeanFactory(beanFactory); 后面被调用，也就是说bean提前初始化的时候，AutowiredAnnotationBeanPostProcessor还没有注册自然就不会被执行到，所以就无法注入bean

## Awre接口

- Awre接口注册时机
  - prepareBeanFactory(beanFactory);  -> beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
  - 在创建容器之后就会注册ApplicationContextAwareProcessor，用来处理Awre接口
- Awre接口调用时机
  - 因为ApplicationContextAwareProcessor是第一个注册，所以会第一个调用
  - EnvironmentAware#setEnvironment(Environment environment);
  - EmbeddedValueResolverAware#setEmbeddedValueResolver(StringValueResolver resolver);
  - ResourceLoaderAware#setResourceLoader(ResourceLoader resourceLoader);
  - ApplicationEventPublisherAware#setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher);
  - MessageSourceAware#setMessageSource(MessageSource messageSource);
  - ApplicationContextAware#setApplicationContext(ApplicationContext applicationContext);

## bean 级别扩展接口

- InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation(Class<?> beanClass, String beanName);
  - 目标对象实例化之前调用，可以用来代替原本该生成的目标对象的实例(比如代理对象)
  - 如果该方法的返回值代替原本该生成的目标对象(即 返回不为空)，后续只有postProcessAfterInitialization方法会调用，其它方法不再调用；否则按照正常的流程走
- InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation(Object bean, String beanName);
  - 目标对象实例化之后调用，这个时候对象已经被实例化，但是该实例的属性还未被设置，都是null
  - 如果该方法返回false，会忽略属性值的设置；如果返回true，会按照正常流程设置属性值
- SmartInstantiationAwareBeanPostProcessor Spring框架内部使用
- SmartInstantiationAwareBeanPostProcessor#predictBeanType(Class<?> beanClass, String beanName);
  - 用于预测Bean的类型，返回第一个预测成功的Class类型，如果不能预测返回null
  - 主要在于BeanDefinition无法确定Bean类型的时候调用该方法来确定类型
- SmartInstantiationAwareBeanPostProcessor#determineCandidateConstructors(Class<?> beanClass, String beanName);
  - 用于选择合适的构造器，比如类有多个构造器，可以实现这个方法选择合适的构造器并用于实例化对象
  - 在postProcessBeforeInstantiation方法和postProcessAfterInstantiation方法之间调用，如果postProcessBeforeInstantiation方法返回了一个新的实例代替了原本该生成的实例，那么该方法会被忽略
- SmartInstantiationAwareBeanPostProcessor#getEarlyBeanReference(Object bean, String beanName);
  - 用于解决循环引用问题，提前暴露
- InstantiationAwareBeanPostProcessor#postProcessProperties(PropertyValues pvs, Object bean, String beanName);
  - 对属性值进行修改(这个时候属性值还未被设置，但是我们可以修改原本该设置进去的属性值)
- BeanPostProcessor#postProcessBeforeInitialization(Object bean, String beanName);
  - 这个时候bean已经被实例化，并且所有该注入的属性都已经被注入，是一个完整的bean
  - 指bean在初始化之前需要调用的方法
- BeanPostProcessor#postProcessAfterInitialization(Object bean, String beanName);
  - 这个时候bean已经被实例化，并且所有该注入的属性都已经被注入，是一个完整的bean
  - 指bean在初始化之后需要调用的方法
- MergedBeanDefinitionPostProcessor#postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName);
  - bean在合并Bean定义之后调用
- DestructionAwareBeanPostProcessor#postProcessBeforeDestruction(Object bean, String beanName);
  - 是bean在Spring在容器中被销毁之前调用
- DestructionAwareBeanPostProcessor#requiresDestruction(Object bean);
  - 默认返回true，为true时才会执行postProcessBeforeDestruction方法

## Spring 事件

- ApplicationContextEvent
  - ContextRefreshedEvent
    - ApplicationContext容器初始化或刷新时触发该事件。此处的初始化是指：所有的Bean被成功装载，后处理Bean被检测并激活，所有Singleton Bean 被预实例化，ApplicationContext容器已就绪可用
    - ContextStartedEvent
    - ContextClosedEvent
    - ContextStoppedEvent
    - RequestHandledEvent
      - Web相关事件，只能应用于使用DispatcherServlet的Web应用。在使用Spring作为前端的MVC控制器时，当Spring处理用户请求结束后，系统会自动触发该事件

## Spring Boot 事件

- SpringApplicationEvent
  - ApplicationStartingEvent
  - ApplicationEnvironmentPreparedEvent
  - ApplicationContextInitializedEvent
  - ApplicationPreparedEvent
  - ApplicationStartedEvent
  - ApplicationReadyEvent
  - ApplicationFailedEvent

## Spring Aop

- JDK 动态代理、CGLIB 代理
- 是编译时期进行织入，还是运行期进行织入？
  - 运行期，生成字节码，再加载到虚拟机中，JDK是利用反射原理，CGLIB使用了ASM原理
- 初始化时期织入还是获取对象时织入？
  - 初始化的时候，已经将目标对象进行代理，放入到spring 容器中，用的使用直接拿出来
- Spring AOP 默认使用jdk动态代理还是cglib？
  - 要看条件，如果实现了接口的类，是使用jdk。如果没实现接口，就使用cglib
  - 也可以强制指定使用cglib，注解参数：targetSource
- AOP 重要点
  - 增强（Advice）
    - AOP（切面编程）是用来给某一类特殊的连接点，添加一些特殊的功能，那么我们添加的功能也就是增强
  - 切点（PointCut）
    - 一个项目中有很多的类，一个类有很多个连接点，当我们需要在某个方法前插入一段增强（advice）代码时，我们就需要使用切点信息来确定，要在哪些连接点上添加增强
    - 如果把连接点当做数据库中的记录，那么切点就是查找该记录的查询条件
    - 所以，一般我们要实现一个切点时，那么我们需要判断哪些连接点是符合我们的条件的，如：方法名是否匹配、类是否是某个类、以及子类等
  - 连接点（JoinPoint）
    - 连接点就是程序执行的某个特定的位置，如：类开始初始化前、类初始化后、类的某个方法调用前、类的某个方法调用后、方法抛出异常后等
    - Spring 只支持类的方法前、后、抛出异常后的连接点
  - 切面（Aspect）
    - 切面由切点和增强(或引介)组成，或者只由增强（或引介）实现
    - 就是我们定义了标识@Aspect注解的类，这个类就是用来切面处理的
- 反射为什么效率低？
  - 很多方法都是通过Unsafe类调用的，是jni的，JVM无法做出优化
- ASM
  - ASM 能够通过改造既有类，直接生成需要的代码。增强的代码是硬编码在新生成的类文件内部的，没有反射带来性能上的付出

## 面试题

- Spring 中使用了哪些设计模式？
  - 工厂模式：spring中的BeanFactory就是简单工厂模式的体现，根据传入唯一的标识来获得bean对象；
  - 单例模式：提供了全局的访问点BeanFactory；
  - 代理模式：AOP功能的原理就使用代理模式（1、JDK动态代理。2、CGLib字节码生成技术代理。）
  - 装饰器模式：依赖注入就需要使用BeanWrapper；
  - 观察者模式：spring中Observer模式常用的地方是listener的实现。如ApplicationListener。
  - 策略模式：Bean的实例化的时候决定采用何种方式初始化bean实例（反射或者CGLIB动态字节码生成）
- AOP源码分析 ？
  - @EnableAspectJAutoProxy给容器（beanFactory）中注册一个AnnotationAwareAspectJAutoProxyCreator对象
  - AnnotationAwareAspectJAutoProxyCreator对目标对象进行代理对象的创建，对象内部，是封装JDK和CGlib两个技术，实现动态代理对象创建的（创建代理对象过程中，会先创建一个代理工厂，获取到所有的增强器（通知方法），将这些增强器和目标类注入代理工厂，再用代理工厂创建对象）
  - 代理对象执行目标方法，得到目标方法的拦截器链，利用拦截器的链式机制，依次进入每一个拦截器进行执行

# Spring MVC

## SpringMVC的流程？

- 用户发送请求至前端控制器DispatcherServlet；
- DispatcherServlet收到请求后，调用HandlerMapping处理器映射器，请求获取Handle；
- 处理器映射器根据请求url找到具体的处理器，生成处理器对象及处理器拦截器(如果有则生成)一并返回给DispatcherServlet；
- DispatcherServlet 调用 HandlerAdapter处理器适配器；
- HandlerAdapter 经过适配调用 具体处理器(Handler，也叫后端控制器)；
- Handler执行完成返回ModelAndView；
- HandlerAdapter将Handler执行结果ModelAndView返回给DispatcherServlet；
- DispatcherServlet将ModelAndView传给ViewResolver视图解析器进行解析；
- ViewResolver解析后返回具体View；
- DispatcherServlet对View进行渲染视图（即将模型数据填充至视图中）
- DispatcherServlet响应用户

# Mybatis

## Mybatis 缓存

- MyBatis 的缓存分为一级缓存和二级缓存,一级缓存放在 session 里面,默认就有,二级缓
  存放在它的命名空间里,默认是不打开的,使用二级缓存属性类需要实现 Serializable 序列化
  接口(可用来保存对象的状态),可在它的映射文件中配置<cache/>
- 默认情况下一级缓存是开启的，而且是不能关闭的
- 一级缓存是指 SqlSession 级别的缓存 原理：使用的数据结构是一个 map，如果两次中间出现 commit 操作 （修改、添加、删除），本 sqlsession 中的一级缓存区域全部清空
- 二级缓存是指可以跨 SqlSession 的缓存。是 mapper 级别的缓存；原理：是通过 CacheExecutor 实现的。CacheExecutor其实是 Executor 的代理对象

## Mybatis 的插件运行原理

- Mybatis 仅可以编写针对 ParameterHandler、ResultSetHandler、StatementHandler、
  Executor 这 4 种接口的插件，Mybatis 通过动态代理，为需要拦截的接口生成代理对象以实
  现接口方法拦截功能，每当执行这 4 种接口对象的方法时，就会进入拦截方法，具体就是
  InvocationHandler 的 invoke()方法，当然，只会拦截那些你指定需要拦截的方法
- 实现 Mybatis 的 Interceptor 接口并复写 intercept()方法，然后在给插件编写注解，指定
  要拦截哪一个接口的哪些方法即可，记住，别忘了在配置文件中配置你编写的插件

## Mybatis 支持延迟加载？

- Mybatis 仅支持 association 关联对象和 collection 关联集合对象的延迟加载，association
  指的就是一对一，collection 指的就是一对多查询。在 Mybatis 配置文件中，可以配置是否
  启用延迟加载 lazyLoadingEnabled=true|false
- 它的原理是，使用 CGLIB 创建目标对象的代理对象，当调用目标方法时，进入拦截器方
  法，比如调用 a.getB().getName()，拦截器 invoke()方法发现 a.getB()是 null 值，那么就会单
  独发送事先保存好的查询关联 B 对象的 sql，把 B 查询上来，然后调用 a.setB(b)，于是 a 的
  对象 b 属性就有值了，接着完成 a.getB().getName()方法的调用。这就是延迟加载的基本原
  理

## Mybatis Mapper 原理

- Dao接口即Mapper接口。接口的全限名，就是映射文件中的namespace的值；接口的方法名，就是映射文件中Mapper的Statement的id值；接口方法内的参数，就是传递给sql的参数
- Mapper接口是没有实现类的，当调用接口方法时，接口全限名+方法名拼接字符串作为key值，可唯一定位一个MapperStatement。在Mybatis中，每一个标签，都会被解析为一个MapperStatement对象
- Mapper接口里的方法，是不能重载的，因为是使用 全限名+方法名 的保存和寻找策略。Mapper 接口的工作原理是JDK动态代理，Mybatis运行时会使用JDK动态代理为Mapper接口生成代理对象proxy，代理对象会拦截接口方法，转而执行MapperStatement所代表的sql，然后将sql执行结果返回

# Mysql

## InnoDB 与 MyISAM 

- InnoDB 支持事务、支持行级锁

## 事务属性(ACID)

- 原子性(Atomicy)
- 一致性(Consisteny)
- 隔离性(Isolation)
- 持久性(Durability)

## 隔离级别

- Read Uncommited 
  - 可以读取未提交记录。此隔离级别，不会使用，忽略
- Read Committed (RC)
  - 快照读
  - 针对当前读，RC隔离级别保证对读取到的记录加锁 (记录锁)，存在幻读现象
  - 普通读：总是读取行的最新版本，如果行被锁定了，则读取该行版本的最小一个快照，从数据库理论的角度看，Read Committed事务隔离级别其实违背了是ACID中的I特性，即隔离性
- Repeatable Read (RR)
  - 针对当前读，RR隔离级别保证对读取到的记录加锁 (记录锁)，同时保证对读取的范围加锁，新的满足查询条件的记录不能够插入 (间隙锁)，不存在幻读现象
  - 普通读：总是读取事务开始时的行数据。不会出现Read Committed事务隔离级别下的问题，即不会出现隔离性的问题
- Serializable
  - Serializable隔离级别下，读写冲突，因此并发度急剧下降，在MySQL/InnoDB下不建议使用

## 事务读现象

- 脏读
  - 事务A读取了事务B更新的数据，然后B回滚操作，那么A读取到的数据是脏数据
- 不可重复读
  - 事务 A 多次读取同一数据，事务 B 在事务A多次读取的过程中，对数据作了更新并提交，导致事务A多次读取同一数据时，结果 不一致
- 幻读
  - 系统管理员A将数据库中所有学生的成绩从具体分数改为ABCDE等级，但是系统管理员B就在这个时候插入了一条具体分数的记录，当系统管理员A改结束后发现还有一条记录没有改过来，就好像发生了幻觉一样，这就叫幻读
- 不可重复读的和幻读很容易混淆，不可重复读侧重于修改，幻读侧重于新增或删除。解决不可重复读的问题只需锁住满足条件的行，解决幻读需要锁表

## MVCC

- MVCC是被Mysql中 事务型存储引擎InnoDB 所支持的
- 应对高并发事务, MVCC比单纯的加锁更高效
- MVCC只在 READ COMMITTED 和 REPEATABLE READ 两个隔离级别下工作
- 其他两个隔离级别够和MVCC不兼容，因为 READ UNCOMMITTED 总是读取最新的数据行, 而不是符合当前事务版本的数据行。而 SERIALIZABLE 则会对所有读取的行都加锁
- MVCC可以使用 乐观(optimistic)锁 和 悲观(pessimistic)锁来实现

## 快照读 和 当前读

- 快照读：读取的是快照版本，也就是历史版本
- 当前读：读取的是最新版本
- 普通的SELECT就是快照读，而UPDATE、DELETE、INSERT、SELECT ...  LOCK IN SHARE MODE、SELECT ... FOR UPDATE是当前读

## 锁定读 和 一致性非锁定读

- 锁定读
  - SELECT ... LOCK IN SHARE MODE
    - 给记录假设共享锁，这样一来的话，其它事务只能读不能修改，直到当前事务提交
  - SELECT ... FOR UPDATE
    - 给索引记录加锁，这种情况下跟UPDATE的加锁情况是一样的
- 一致性非锁定读
  - consistent read （一致性读），InnoDB用多版本来提供查询数据库在某个时间点的快照
  - 如果隔离级别是REPEATABLE READ，那么在同一个事务中的所有一致性读都读的是事务中第一个这样的读读到的快照
  - 如果是READ COMMITTED，那么一个事务中的每一个一致性读都会读到它自己刷新的快照版本
  - Consistent read（一致性读）是READ COMMITTED和REPEATABLE READ隔离级别下普通SELECT语句默认的模式
  - 一致性读不会给它所访问的表加任何形式的锁，因此其它事务可以同时并发的修改它们
- 在默认的隔离级别中，普通的SELECT用的是一致性读不加锁。而对于锁定读、UPDATE和DELETE，则需要加锁，至于加什么锁视情况而定。如果你对一个唯一索引使用了唯一的检索条件，那么只需锁定索引记录即可；如果你没有使用唯一索引作为检索条件，或者用到了索引范围扫描，那么将会使用间隙锁或者next-key锁以此来阻塞其它会话向这个范围内的间隙插入数据
- 在修改的时候一定不是快照读，而是当前读
  - 假设事务A更新表中id=1的记录，而事务B也更新这条记录，并且B先提交，如果按照前面MVVC说的，事务A读取id=1的快照版本，那么它看不到B所提交的修改，此时如果直接更新的话就会覆盖B之前的修改，这就不对了，可能B和A修改的不是一个字段，但是这样一来，B的修改就丢失了，这是不允许的
  - 所以，在修改的时候一定不是快照读，而是当前读
  - 只有普通的SELECT才是快照读，其它诸如UPDATE、删除都是当前读。修改的时候加锁这是必然的，同时为了防止幻读的出现还需要加间隙锁
- 利用MVCC实现一致性非锁定读，这就有保证在同一个事务中多次读取相同的数据返回的结果是一样的，解决了不可重复读的问题
- 利用Gap Locks和Next-Key可以阻止其它事务在锁定区间内插入数据，因此解决了幻读问题

## InnoDB行锁

- 行级锁并不是直接锁记录，而是锁索引。索引分为主键索引和非主键索引两种，如果一条sql 语句操作了主键索引，Mysql 就会锁定这条主键索引
- 如果一条语句操作了非主键索引，MySQL会先锁定该非主键索引，再锁定相关的主键索引
- InnoDB 行锁是通过给索引项加锁实现的，如果没有索引，InnoDB 会通过隐藏的聚簇索引来对记录加锁
- 也就是说：如果不通过索引条件检索数据，那么InnoDB将对表中所有数据加锁，实际效果跟表锁一样。因为没有了索引，找到某一条记录就得扫描全表，要扫描全表，就得锁定表

## 共享锁 排它锁

- 共享锁：对某一资源加共享锁，自身可以读该资源，其他人也可以读该资源（也可以再继续加共享锁，即 共享锁可多个共存），但无法修改。要想修改就必须等所有共享锁都释放完之后。语法为：

  - ```mysql
    select * from table lock in share mode
    ```

- 对某一资源加排他锁，自身可以进行增删改查，其他人无法进行任何操作。语法为：

  - ```mysql
    select * from table for update
    ```

## 间隙锁

- 防止幻读的发生

- 锁定索引记录间隙，确保索引记录的间隙不变。间隙锁是针对事务隔离级别为可重复读或以上级别的

## Next-Key Lock 

- 行锁和间隙锁组合起来就叫Next-Key Lock
- 默认情况下，InnoDB工作在可重复读隔离级别下，并且会以Next-Key Lock的方式对数据行进行加锁，这样可以有效防止**幻读**的发生。Next-Key Lock是行锁和间隙锁的组合，当InnoDB扫描索引记录的时候，会首先对索引记录加上行锁（Record Lock），再对索引记录两边的间隙加上间隙锁（Gap Lock）。加上间隙锁之后，其他事务就不能在这个间隙修改或者插入记录

## MySQL 乐观锁和悲观锁

- 读用乐观锁，写用悲观锁

- 悲观锁：
  - select .... for update 
  - 并发量大
- 乐观锁：
  - 使用 version 或者 timestamp 进行比较
  - 并发量小
  - CAS
  - update items set inventory=inventory-1 where id=100 and inventory-1>0;

## 死锁

- 设置超时时间：innodb_lock_wait_timeout

## 聚集索引与非聚集索引

- 聚集索引一个表只能有一个，而非聚集索引一个表可以存在多个

- 聚集索引存储记录是物理上连续存在，而非聚集索引是逻辑上的连续，物理存储并不连续

- 聚集索引表记录的排列顺序和索引的排列顺序一致，所以查询效率快，只要找到第一个索引值记录，其余就连续性的记录在物理也一样连续存放。聚集索引对应的缺点就是修改慢，因为为了保证表中记录的物理和索引顺序一致，在记录插入的时候，会对数据页重新排序

- 非聚集索引制定了表中记录的逻辑顺序，但是记录的物理和索引不一定一致，两种索引都采用B+树结构，非聚集索引的叶子层并不和实际数据页相重叠，而采用叶子层包含一个指向表中的记录在数据页中的指针方式。非聚集索引层次多，不会造成数据重排。这个指针并不是地址，而是主键，通过主键就可以查到数据

- 创建

  - 如果一个主键被定义了，那么这个主键就是作为聚集索引
  - 如果没有主键被定义，那么该表的第一个唯一非空索引被作为聚集索引
  - 如果没有主键也没有合适的唯一索引，那么innodb内部会生成一个隐藏的主键作为聚集索引，这个隐藏的主键是一个6个字节的列，改列的值会随着数据的插入自增
  - Innodb中的每张表都会有一个聚集索引，而聚集索引又是以物理磁盘顺序来存储的，自增主键会把数据自动向后插入，避免了插入过程中的聚集索引排序问题。聚集索引的排序，必然会带来大范围的数据的物理移动，这里面带来的磁盘IO性能损耗是非常大的。 而如果聚集索引上的值可以改动的话，那么也会触发物理磁盘上的移动，于是就可能出现page分裂，表碎片横生

- 如何解决非聚集索引的二次查询问题？即回表？

  - 复合索引（覆盖索引）

  - 建立两列以上的索引，即可查询复合索引里的列的数据而不需要进行回表二次查询，如index(col1, col2)，执行下面的语句

  - 将单列索引(name)升级为联合索引(name, sex)，也可以避免回表
  
  - ```mysql
    	select col1, col2 from t1 where col1 = '213';
    ```

## 覆盖索引

- 如果一个索引包含(或覆盖)所有需要查询的字段的值，称为‘覆盖索引’。即只需扫描索引而无须回表
- 将单列索引(name)升级为联合索引(name, sex)，也可以避免回表

## 自适应哈希索引

- 哈希索引是一种非常快的等值查找方法（注意：必须是等值，哈希索引对非等值查找方法无能为力），它查找的时间复杂度为常量，InnoDB采用自适用哈希索引技术，它会实时监控表上索引的使用情况，如果认为建立哈希索引可以提高查询效率，则自动在内存中的“自适应哈希索引缓冲区”（详见《MySQL - 浅谈InnoDB体系架构》中内存构造）建立哈希索引
- 之所以该技术称为“自适应”是因为完全由InnoDB自己决定，不需要DBA人为干预。它是通过缓冲池中的B+树构造而来，且不需要对整个表建立哈希索引，因此它的数据非常快
- InnoDB官方文档显示，启用自适应哈希索引后，读和写性能可以提高2倍，对于辅助索引的连接操作，性能可以提高5被，因此默认情况下为开启，我们可以通过参数innodb_adaptive_hash_index来禁用此特性

## 索引下推

- 5.6版本后新增索引下推优化，减少回表次数

- 在使用非主键索引（又叫普通索引或者二级索引）进行查询时，存储引擎通过索引检索到数据，然后返回给MySQL服务器，服务器然后判断数据是否符合条件

- 使用索引下推时，如果存在某些被索引的列的判断条件时，MySQL服务器将这一部分判断条件传递给存储引擎，然后由存储引擎通过判断索引是否符合MySQL服务器传递的条件，只有当索引符合条件时才会将数据检索出来返回给MySQL服务器

- 索引条件下推优化可以减少存储引擎查询基础表的次数，也可以减少MySQL服务器从存储引擎接收数据的次数

- 举例：

  - ```mysql
    　SELECT * from user where  name like '陈%'
      　-- 根据 "最佳左前缀" 的原则，这里使用了联合索引（name，age）进行了查询，性能要比全表扫描肯定要高
      　-- 问题来了，如果有其他的条件呢？假设又有一个需求，要求匹配姓名第一个字为陈，年龄为20岁的用户，此时的sql语句如下：
      　SELECT * from user where  name like '陈%' and age=20
    ```

  - 没有索引下推

    - 会忽略age这个字段，直接通过name进行查询，在(name,age)这课树上查找到了两个结果，id分别为2,1，然后拿着取到的id值一次次的回表查询，因此这个过程需要回表两次

  - 有索引下推

    - InnoDB并没有忽略age这个字段，而是在索引内部就判断了age是否等于20，对于不等于20的记录直接跳过，因此在(name,age)这棵索引树中只匹配到了一个记录，此时拿着这个id去主键索引树中回表查询全部数据，这个过程只需要回表一次

  - 根据explain解析结果可以看出Extra的值为Using index condition，表示已经使用了索引下推

- 索引下推在非主键索引上的优化，可以有效减少回表的次数，大大提升了查询的效率

## 索引失效

### 如果条件中有or

- 如果条件中有or，即使其中有部分条件带索引也不会使用(这也是为什么尽量少用or的原因)，例子中user_id无索引
- 要想使用or，又想让索引生效，只能将or条件中的每个列都加上索引

### 复合索引未用左列字段

- 对于复合索引，如果不使用前列，后续列也将无法使用，类电话簿

### like以%开头

### 需要类型转换

### where中索引列有运算

### where中索引列使用了函数

## count(1)、count(*)、count(字段)的区别

- 实际上count(1)和count(*)并没有区别
- count(*),自动会优化指定到那一个字段。所以没必要去count(1)，用count(*),sql会帮你完成优化的 因此：count(1)和count(*)基本没有差别
- 数据库中count(*)和count(列)根本就是不等价的，count(*)是针对于全表的，而count(列)是针对于某一列的，如果此列值为空的话，count(列)是不会统计这一行的

## 建议使用count(*)符合标准

- 阿里巴巴规范
- 【强制】不要使用 count(列名)或 count(常量)来替代 count()，count()是 SQL92 定义的 标准统计行数的语法，跟数据库无关，跟 NULL 和非 NULL 无关。 说明：count(*)会统计值为 NULL 的行，而 count(列名)不会统计此列为 NULL 值的行

## 为什么MySQL数据库索引选择使用B+树？

- B树
  - 在叶子节点和非叶子节点都存放数据
  - 查询效率不稳定，它的数据分布在整棵树的节点上的任何一个节点
  - 由于非叶子节点也存储了数据，所以查找数据时，会进行多次磁盘IO才会找到想要的数据
- B+树
  - 每次IO的大小是固定的，但是B+树没有非叶子节点没有数据，那么每次IO读取的包含索引的块更多，B+树单次IO的信息量大于B树，所以B+树比B树磁盘IO更少
  - 所有数据都在叶子节点，非叶子节点可以存储更多索引，使得查询的IO次数更少
  - 所有查询都要查找到叶子节点，查询性能稳定
  - 而且叶子节点之间都是链表的结构，所以B+ Tree也是可以支持范围查询的，而B树每个节点 key 和 data 在一起，则无法区间查找
- 简单回答
  - 因为B树不管叶子节点还是非叶子节点，都会保存数据，这样导致在非叶子节点中能保存的指针数量变少
  - 指针少的情况下要保存大量数据，只能增加树的高度，导致IO操作变多，查询性能变低

## 为什么不用红黑树或者二叉排序树？

- 树的查询时间跟树的高度有关，B+树是一棵多路搜索树可以降低树的高度，提高查找效率

## InnoDB一棵B+树可以存放多少行数据？

- 约2千万

## Explain 关键字（MySQL执行计划）

### id

- 表示查询中select操作表的顺序,按顺序从大到依次执行
- id相同：从上往下顺序执行
- id不相同：按顺序从大到小依次执行
- id有相同有不相同的：按顺序从大大小依次执行，遇到相同的再从上往下顺序执行

### select_type

- SIMPLE：简单的select查询，查询中不包含子查询或者UNION
- PRIMARY：查询中若包含任何复杂的子部分，最外层查询则被标记，表示最外层查询，最后执行的那个查询
- SUBQUERY：在SELECT或WHERE列表中包含子查询
- DERIVED：在FROM列表中包含的子查询被标记为DERIVED（衍生），MySQL会递归执行这些子查询，把结果放在临时表里
- UNION：若第二个SELECT出现在UNION之后，则被标记为UNION。若UNION包含在FROM子句的子查询中，最外层SELECT将被标记为DERIVED
- UNION RESULT：从UNION表获取结果的SELECT

### type

- ALL(全表扫描)：
  - Full Table Scan，将遍历全表以找到需要的行
- index(索引扫描)：
  - Full Index Scan，index与ALL的区别为index类型值遍历索引树。通常比ALL快，因为索引文件通常比数据文件小。也就是说虽然all与index都是读全表，但index是从索引中读取的，而all是从硬盘中读
- range(范围扫描)：
  - 只检索给定范围的行，使用一个索引来选择行。key列显示使用了哪个索引一般就是你的where语句中出现了between、<、>、in等的查询这种范围扫描索引扫描比全表扫描要好，因为它只需要开始于索引的某一个点，而结束与另一个点，不用全表扫描
- ref (非唯一索引扫描)：
  - 非唯一索引扫描，返回匹配某个单独值的所有行。它可能会找到多个复合条件的行，所以他应该属于查找和扫描的混合体
- eq_ref(唯一索引扫描)：
  - 唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配。常见主键或唯一索引扫描。
- const(常数引用)：
  - 表示通过索引一次就找到了，const用于比较primary key或者unique索引。因为只匹配一行数据，所以很快如将主键置于where列表中，MySQL就能将该查询转换为为一个常量
- system(系统级别)：
  - 表只有一行记录（等于系统表），这是const类型的特列，平时不会出现，这个也可以忽略不计
- system > const > eq_ef > ref > range > index > ALL

### table

### possible_keys

- 顾名思义,该属性给出了,该查询语句,可能走的索引,(如某些字段上索引的名字)这里提供的只是参考,而不是实际走的索引,也就导致会有possible_Keys不为null,key为空的现象

### key

- 显示MySQL实际使用的索引,其中就包括主键索引(PRIMARY),或者自建索引的名字

### key_len

- 表示索引所使用的字节数，在不损失精度的情况下，长度越短越好

### ref

- 连接匹配条件,如果走主键索引的话,该值为: const, 全表扫描的话,为null值

### rows

- 扫描行数,也就是说，需要扫描多少行,才能获取目标行数,一般情况下会大于返回行数。通常情况下,rows越小,效率越高, 也就有大部分SQL优化，都是在减少这个值的大小
- 注意:  理想情况下扫描的行数与实际返回行数理论上是一致的,但这种情况及其少,如关联查询,扫描的行数就会比返回行数大大增加)

### extra

- 这个属性非常重要,该属性中包括执行SQL时的真实情况信息,如上面所属,使用到的是”using where”，表示使用where筛选得到的值,常用的有：
- “Using temporary”: 使用临时表 “using filesort”: 使用文件排序
- Using filesort：
  - 说明MySQL会对数据使用一个外部的索引排序，而不是按照内部表内的索引顺序进行读取。MySQL无法利用索引完成的排序操作称为“文件排序”
  - 组合索引是：idx_col1_col2_col3
  - explain select col1 from t1 where col1 ='ac' order by col3 会导致Using filesort
  - explain select col1 from t1 where col1 ='ac' order by col2,col3 正解
- Using temporary：
  - 使用了临时表保存中间结果，MySQL在对查询结果排序时使用临时表
  - 组合索引是：idx_col1_col2
  - explain select col1 from t1 where col1 in ('ac','bc','dd') group by col2 会导致Using filesort，Using temporary
  - explain select col1 from t1 where col1 in ('ac','bc','dd') group by col1,col2 正解，group by尤其注意
- Using index：
  - 表示相应的select操作中使用了覆盖索引（Covering index），避免访问了表的数据行，效率不错，如果同时出现using where，表明索引被用来执行索引键值的查找
  - 如果没有同时出现using where，表明索引用来读取数据而非执行查找动作
  - explain select col2 from t1 where col1='aa'	出现了Using where、Using index   表明索引用来执行索引键值的查找
  - explain select col1,col2 from t1;出现了Using index 表明索引用来读取数据而非执行查找操作
- Using where：
  - 表明使用了where 过滤
- Using join buffer：
  - 使用了连接缓存
- Impossible where：
  - where子句的值总是false，不能用来获取任何元祖
- select tables optimized away：
  - 在没有group by 子句的情况下，基于索引优化min/max操作或者对于myisam存储引擎优化count(*)操作，不必等到执行阶段再进行计算。查询执行计划生成的阶段即完成优化
- distinct：
  - 优化distinct操作，在找到第一匹配的元祖后即停止找同样值的动作

## Log

### redo log 重做日志

- 作用
  - 防止在发生故障的时间点，尚有脏页未写入磁盘，在重启mysql服务的时候，根据redo log进行重做，从而达到事务的持久性这一特性
- 内容
  - 物理格式的日志，记录的是物理数据页面的修改的信息，其redo log是顺序写入redo log file的物理文件中去的
- 什么时候产生
  - 事务开始之后就产生redo log，redo log的落盘并不是随着事务的提交才写入的，而是在事务的执行过程中，便开始写入redo log文件中
- 什么时候释放
  - 当对应事务的脏页写入到磁盘之后，redo log的使命也就完成了，重做日志占用的空间就可以重用（被覆盖）
- 重做日志是在事务开始之后逐步写入重做日志文件，而不一定是事务提交才写入重做日志缓存
  - 重做日志有一个缓存区Innodb_log_buffer，Innodb_log_buffer的默认大小为8M(这里设置的16M),Innodb存储引擎先将重做日志写入innodb_log_buffer中
  - 然后会通过以下三种方式将innodb日志缓冲区的日志刷新到磁盘
    - Master Thread 每秒一次执行刷新Innodb_log_buffer到重做日志文件
    - 每个事务提交时会将重做日志刷新到重做日志文件
    - 当重做日志缓存可用空间 少于一半时，重做日志缓存被刷新到重做日志文件
  - 即使某个事务还没有提交，Innodb存储引擎仍然每秒会将重做日志缓存刷新到重做日志文件
  - 这一点是必须要知道的，因为这可以很好地解释再大的事务的提交（commit）的时间也是很短暂的

### undo log 回滚日志

- 作用
  - 保存了事务发生之前的数据的一个版本，可以用于回滚，同时可以提供多版本并发控制下的读（MVCC），也即非锁定读
- 内容
  - 逻辑格式的日志，在执行undo的时候，仅仅是将数据从逻辑上恢复至事务之前的状态，而不是从物理页面上操作实现的，这一点是不同于redo log的
- 什么时候产生
  - 事务开始之前，将当前是的版本生成undo log，undo 也会产生 redo 来保证undo log的可靠性
- 什么时候释放
  - 当事务提交之后，undo log并不能立马被删除，而是放入待清理的链表，由purge线程判断是否由其他事务在使用undo段中表的上一个事务之前的版本信息，决定是否可以清理undo log的日志空间

### binlog 二进制日志

- 作用
  - 用于复制，在主从复制中，从库利用主库上的binlog进行重播，实现主从同步
- 内容
  - 逻辑格式的日志，可以简单认为就是执行过的事务中的sql语句
  - 但又不完全是sql语句这么简单，而是包括了执行的sql语句（增删改）反向的信息，
    - delete对应着delete本身和其反向的insert
    - update对应着update执行前后的版本的信息
    - insert对应着delete和insert本身的信息
  - 在使用mysqlbinlog解析binlog之后一些都会真相大白
  - 因此可以基于binlog做到类似于oracle的闪回功能，其实都是依赖于binlog中的日志记录
- 什么时候产生
  - 事务提交的时候，一次性将事务中的sql语句（一个事务可能对应多个sql语句）按照一定的格式记录到binlog中
  - 这里与redo log很明显的差异就是redo log并不一定是在事务提交的时候刷新到磁盘，redo log是在事务开始之后就开始逐步写入磁盘
  - 因此对于事务的提交，即便是较大的事务，提交（commit）都是很快的，但是在开启了bin_log的情况下，对于较大事务的提交，可能会变得比较慢一些，这是因为binlog是在事务提交的时候一次性写入的造成的
- 什么时候释放
  - binlog的默认是保持时间由参数expire_logs_days配置，也就是说对于非活动的日志文件，在生成时间超过expire_logs_days配置的天数之后，会被自动删除
- 二进制日志的作用之一是还原数据库的，这与redo log很类似，但两者有本质的不同
  - 作用不同：
    - redo log是保证事务的持久性的，是事务层面的，binlog作为还原的功能，是数据库层面的（当然也可以精确到事务层面的），虽然都有还原的意思，但是其保护数据的层次是不一样的
  - 内容不同：
    - redo log是物理日志，是数据页面的修改之后的物理记录，binlog是逻辑日志，可以简单认为记录的就是sql语句
  - 恢复数据时候的效率，基于物理日志的redo log恢复数据的效率要高于语句逻辑日志的binlog
- redo log和binlog的写入顺序
  - 为了保证主从复制时候的主从一致（当然也包括使用binlog进行基于时间点还原的情况），是要严格一致的，MySQL通过两阶段提交过程来完成事务的一致性的，也即redo log和binlog的一致性的，理论上是先写redo log，再写binlog，两个日志都提交成功（刷入磁盘），事务才算真正的完成

### errorlog 错误日志

### slow query log 慢查询日志

### genera log 一般查询日志

### relay log 中继日志

## id 用完

- bigInt类型
- 分库分表
- row_id
  - 在我们使用InnoDB存储引擎来建表时，如果我们自己没有显式地创建主键时，存储引擎会默认找一个具有NOT NULL属性的唯一二级索引列来充当主键，如果我们在建表语句中也没有写具有NOT NULL属性的唯一二级索引列，那存储引擎默认会添加一个名为row_id的主键列
  - 这个row_id列默认是6个字节大小，值得注意的是，设计InnoDB的大叔并不是为每一个用户未显式创建主键的表的row_id列都单独维护一个计数器，而是所有的表都共享一个全局的计数器
  - 写入表的 row_id 是从 0 开始到 2^48-1。达到上限后，下一个值就是 0，然后继续循环
- thread_id

## MySQL 为什么用自增作为主键？

- 如果我们定义了主键(PRIMARY KEY)，那么InnoDB会选择主键作为聚集索引、如果没有显式定义主键，则InnoDB会选择第一个不包含有NULL值的唯一索引作为主键索引、如果也没有这样的唯一索引，则InnoDB会选择内置6字节长的ROWID作为隐含的聚集索引(ROWID随着行记录的写入而主键递增，这个ROWID不像ORACLE的ROWID那样可引用，是隐含的)
- 如果表使用自增主键，那么每次插入新的记录，记录就会顺序添加到当前索引节点的后续位置，当一页写满，就会自动开辟一个新的页
- 如果使用非自增主键（如果身份证号或学号等），由于每次插入主键的值近似于随机，因此每次新纪录都要被插到现有索引页得中间某个位置，此时MySQL不得不为了将新记录插到合适位置而移动数据，甚至目标页面可能已经被回写到磁盘上而从缓存中清掉，此时又要从磁盘上读回来，这增加了很多开销，同时频繁的移动、分页操作造成了大量的碎片，得到了不够紧凑的索引结构，后续不得不通过OPTIMIZE TABLE来重建表并优化填充页面

## varchar(50)代表的含义？

- varchar(50)中50的涵义最多存放50个字符，varchar(50)和(200)存储hello所占空间一样，但后者在排序时会消耗更多内存，因为order by col采用fixed_length计算col长度(memory引擎也一样) 

## int(10)的含义？

- int(10)的意思是假设有一个变量名为id，它的能显示的宽度能显示10位。在使用id时，假如我给id输入10，那么mysql会默认给你存储0000000010。当你输入的数据不足10位时，会自动帮你补全位数。假如我设计的id字段是int(20)，那么我在给id输入10时，mysql会自动补全18个0，补到20位为止
- int(M)的作用于int的范围明显是无关的，int(M)只是用来显示数据的宽度，我们能看到的宽度。当字段被设计为int类型，那么它的范围就已经被写死了，与M无关
- 要查看出不同效果记得在创建类型的时候加 zerofill这个值，表示用 0 填充，否则看不出效果的
- 声明字段是int类型的那一刻起，int就是占四个字节，一个字节 8 位，也就是4*8=32，可以表示的数字个数是 2的 32 次方(2^32 = 4 294 967 296个数字)

## Myisam与Innodb的区别？

- InnoDB支持事物，而MyISAM不支持事物
- InnoDB支持行级锁，而MyISAM支持表级锁
- InnoDB支持MVCC, 而MyISAM不支持
- InnoDB支持外键，而MyISAM不支持
- InnoDB不支持全文索引，而MyISAM支持

## select count(*)哪个更快，为什么？

- myisam更快，因为myisam内部维护了一个计数器，可以直接调取

## 一个6亿的表a，一个3亿的表b，通过外间tid关联，你如何最快的查询出满足条件的第50000到第50200中的这200条数据记录？

- 如果A表TID是自增长,并且是连续的,B表的ID为索引

  - ```mysql
    select * from a,b where a.tid = b.id and a.tid>500000 limit 200;
    ```

- 如果A表的TID不是连续的,那么就需要使用覆盖索引.TID要么是主键,要么是辅助索引,B表ID也需要有索引。

  - ```mysql
    select * from b , (select tid from a limit 50000,200) a where b.id = a .tid;
    ```

## 分库分表

- 垂直拆分
- 水平拆分

## 读写分离

## 主从复制

- 基本原理
  - 主：binlog线程，记录下所有改变了数据库数据的语句，放进master上的binlog中
  - 从：io线程，在使用start slave之后，负责从master上拉去binglog内容，放进自己的relay log中
  - 从：sql执行线程，执行relay log中的语句
- MySQL之间数据复制的基础是二进制日志文件（binary log file）。一台MySQL数据库一旦启用二进制日志后，其作为master，它的数据库中所有操作都会以“事件”的方式记录在二进制日志中，其他数据库作为slave通过一个I/O线程与主服务器保持通信，并监控master的二进制日志文件的变化，如果发现master二进制日志文件发生变化，则会把变化复制到自己的中继日志中，然后slave的一个SQL线程会把相关的“事件”执行到自己的数据库中，以此实现从数据库和主数据库的一致性，也就实现了主从复制
- 对于每一个主从复制的连接，都有三个线程。拥有多个从库的主库为每一个连接到主库的从库创建一个binlog输出线程，每一个从库都有它自己的I/O线程和SQL线程

# Redis

## Redis 为什么这么快？

- 纯内存操作：读取不需要进行磁盘 I/O，所以比传统数据库要快上不少；(但不要有误区说磁盘就一定慢，例如 Kafka 就是使用磁盘顺序读取但仍然较快)
- 单线程，无锁竞争：这保证了没有线程的上下文切换，不会因为多线程的一些操作而降低性能
- 多路 I/O 复用模型，非阻塞 I/O：采用多路 I/O 复用技术可以让单个线程高效的处理多个网络连接请求（尽量减少网络 IO 的时间消耗）
- 高效的数据结构，加上底层做了大量优化：Redis 对于底层的数据结构和内存占用做了大量的优化，例如不同长度的字符串使用不同的结构体表示，HyperLogLog 的密集型存储结构等等

## Redis 常用数据结构及实现？

- 首先在 Redis 内部会使用一个 RedisObject 对象来表示所有的 key 和 value
- 其次 Redis 为了 平衡空间和时间效率，针对 value 的具体类型在底层会采用不同的数据结构来实现

## 集群中数据如何分区？

- 带有虚拟节点的一致性哈希算法分区

## 数据结构

- 容器通用规则

  - 如果容器不存在，则创建一个再进行操作
  - 如果容器里面没有元素了，那么立即删除容器，释放内存
  - 如果一个字符串已经设置了过期时间，然后调用了set方法修改了它，那么它的过期时间会消失

- String

  - Redis 的字符串是动态字符串，是可以修改的字符串，内部类似Java的ArrayList，采用预分配冗余空间的方式来减少内存的频繁分配

  - 当字符串长度小于1M时，扩容2倍；当字符串长度大于1M时，每次扩容1M；字符串最大长度为512M

  - 底层

    - SDS (Simple Dynamic Stirng，简单动态字符串)，它的结构是一个带长度信息的字节数组

      - ```c
        struct SDS<T> {
          T capacity; // 数组容量
          T len;			// 数组长度
          byte flags; // 特殊标识位
          byte[] content; // 数组内容
        }
        ```

- List

  - 相当于LinkedList，有序，插入和删除速度快，时间复杂度为O(1)，但是索引定位慢，时间复杂度为O(1)
  - Redis 的列表结构常用来做异步队列，将需要延后处理的任务结构序列化成字符串放入Redis 的列表，另一个线程从这个列表中轮询数据进行处理
  - lindex 命令
    - lindex 相当于Java集合的get(int index) 方法，它需要对链表进行遍历，性能随着参数index的变大而变差
    - index可以为负数，index=-1表示倒数第一个，-2表示倒数第二个，以此类推
  - 快速列表 quicklist
    - 当数据较少时会使用一块连续的内存存储，这个结构是ziplist，即压缩列表，它将所有的元素紧挨着一起存储，分配的是一块连续的内存
    - 当数据较多时才会变成quicklist，因为普通的链表需要的附加指针空间太大，会比较浪费空间，而且会加重内存的碎片化，比如列表里只是存储了int类型，结构上还需要两个额外的指针 prev和next，所以Redis将链表和 ziplist 组成了quicklist，即将多个ziplist 使用双向指针串联起来使用，这样既满足了快速的插入和删除，又不会出现太大的空间浪费

- Hash 字典

  - 相当于HashMap，无序
  - 不同的是
    - Redis 的字典的值只能是字符串
    - HashMap rehash时需要一次性全部hash，效率低；Redis采用 渐进式rehash
      - 渐进式rehash会在rehash的同时，保留新旧两个hash结构，查询时会同时查询两个hash结构，然后再循循渐进地将就hash的内容一点点迁移到新的hash结构中
  - hash结构也可以用来存储用户信息，不用于字符串一次性需要全部序列化整个对象，hash可以对用户结构中的每个字段单独存储，这样当我们要获取用户信息的时候可以只查询部分信息，这样比一次性全部读取效率高
  - hash缺点：hash结构的存储消耗要高于单个字符串

- Set

  - 相当于HashSet，无序
  - 它的内部实现相当于一个特殊的字典，字典中所有的value都是NULL
  - set结构可以用来存储活动中中奖的用户ID，因为有去重功能
  - 当 set 集合元素都是整数并且元素个数较小时，Redis 会使用 intset 来存储元素。intset 是紧凑的数据结构

- ZSet

  - 类似于SortedSet和HashMap的结合体，有序、唯一
  - 为每一个value赋予了一个score，代表这个value的排序权重
  - 数据结构为 跳跃表
  - zset可以用来保存粉丝列表，value值时粉丝用户的ID，score是关注时间，可以对粉丝列表按关注时间排序
  - zset可以用来保存成绩，value是学生ID，score是学生成绩分数，可以对成绩按分数高低排序
  - 如果 socre 值都一样？
    - zset 的排序元素不只看 score 值，如果 score 值相同还需要再比较 value 值

- 位图 BitMap

  - 就是通过一个bit位来表示某个元素对应的值或者状态,其中的key就是对应元素本身。我们知道8个bit可以组成一个Byte，所以bitmap本身会极大的节省储存空间
  - setbit、getbit、bitcount
  - 使用场景
    - 每日签到、统计活跃用户、登录天数、用户在线状态

- HyperLogLog

  - 统计数据，HyperLogLog提供不精确的去重计数方案，标准误差0.81%
  - HyperLogLog需要占用12K的存储空间
    - 在计数比较小时，它的存储空间采用稀疏矩阵存储，空间占用很小
    - 在计数变大超过了阈值时才会一次性变成稠密矩阵，才会占用12K空间

- 布隆过滤器

  - 如果想知道一个值是不是在一个数据量巨大的数据中时，需要采用布隆过滤器
  - 有一定的误判率
  - 当布隆过滤器说某个值存在时，这个值可能不存在；当它说不存在时，那就肯定不存在
  - 参数调节
    - initial_size 估计的过大，会浪费存储空间，估计的过小，就会影响准确率
    - error_rate 越小，需要的存储空间就越大，对于不需要过去精确的场合，error_rate 设置稍大一些也可以，比如新闻去重时，误判率高一点只会让小部分文章不能被看见，无伤大雅
  - 场景
    - 过滤掉用户看过的新闻
  - 原理
    - 每个布隆过滤器对应到Redis的数据结构里面就是一个大型的位数组和几个不一样的无偏hash函数，所谓无偏就是能够把元素的 hash 值算的比较均匀
    - 添加 key 时，会使用多个 hash 函数对 key 进行 hash 算的一个整数索引值然后对位数组进行取模运算得到一个位置，每个 hash 函数都会算的一个不同的位置。再把位数组的这几个位置都设为 1 就完成了 add 操作
    - 查询 key 时，和 add 一样，也会把 hash 的几个位置都算出来，看看位数组中这几个位置是否都为 1，只要有一个位为 0，那么说明布隆过滤器中这个 key 不存在；如果都是 1，这并不能说明这个 key 就一定存在，只是极有可能存在，因为这些位置为 1 可能时因为其它的 key 存在所致。如果这个数组比较稀疏，这个概率就会很大，如果这个位数组比较拥挤，这个概率就会降低
    - 使用时不要让实际元素远大于初始化大小，当实际元素大小开始超出初始化大小时，应该对布隆过滤器进行重建，重新分配一个 size 更大的过滤器，再将所有的历史元素批量 add 进去

- GeoHah

  - 附近的人

- Scan

- Pub/Sub

## 分布式锁

- 原子操作问题
  - Redis 已经将setnx和expire合并为一个指令，在setnx方法里多加了一个过期时间参数解决了原子操作问题
- 超时问题
  - 如果在加锁和释放锁之间的逻辑执行的太长，以至于超出了锁的超时限制，就会出现问题：
    - 这个时候锁过期了，第二个线程拿到了锁，但是第一个线程还未执行完导致线程不安全
      - 解决：无限续期，第一个线程启动一个定时任务，一直给key续期，这样，当第一个线程未执行完时，锁一直被占有直到被第一个线程释放
    - 第二个线程拿到了锁，第一个线程执行完就是释放掉了锁，第三个线程来又拿到了锁，导致业务错误
      - 解决：set指令的value参数设置一个随机数，释放锁时匹配随机数是否一直，然后再删除key，防止其他线程删除当前线程的key。但是匹配value和删除key不是一个原子操作，需要使用Lua脚本来处理
- 可重入性
  - ThreadLocal 解决
- 集群模式下的分布式锁
  - 在 Sentinel 集群中，主节点挂掉时，从节点会取而代之，客户端上并没有明显感知。原先第一个客户端在主节点中申请成功了一把锁，但是这把锁还没来得及同步到从节点，主节点突然宕机了，然后从节点成为了主节点，这个新的节点中内部没有这个锁，所以当另一个客户端过来请求加锁时，立即就通过了，这样会导致系统中同样一把锁被两个客户端同时持有，不安全性由此产生
  - 解决
    - Redlock算法，大多数机制，加锁时，它会向过半节点发送 set(key, value, nx=true, ex=xxx) 指令，只要过半节点 set 成功，那就认为加锁成功。释放锁时，需要向所有节点发送 del 指令

## 线程模型

- Redis 单线程为什么还这么快？
  - 因为它所有数据都在内存中，单线程避免线程切换带来的开销，Redis底层数据结构优化的非常多
  - 因为Redis时单线程，所以需要小心使用 Redis 指令，防止导致 Redis 卡顿
- Redis 单线程如何处理那么多并发客户端连接？
  - 多路复用IO、Select 事件轮询、非阻塞IO

## 持久化

- AOF
  - 日志追加
  - 内存数据修改的指令记录文本，AOF日志在长期的运行过程中会变大越来越大，数据库重启需要加载 AOF 日志进行指令重放，这个四航局就会无比漫长，所以需要定期进行 AOF 重写，给 AOF 日志进行瘦身
  - Redis 提供了 bgrewriteaof 指令用于对 AOF 日志进行瘦身。其原理就是开辟一个子进程对内存进行遍历转换成一些列 Redis 的操作指令，序列化一个新的 AOF 日志文件中。序列化完毕后再将操作期间发生的增量 AOF 日志追加到这个新的 AOF 日志文件中，追加完毕后就立即替代旧的 AOF 日志文件
  - fsync
    - AOF 日志文件以文件的形式存在，当程序对 AOF 日志文件进行写操作时，实际上是将内容写到了内核为文件描述符分配的一个内存缓存中，然后内核会异步将脏数据刷回到磁盘中
    - 如果机器突然宕机，AOF 日志内容可能还没来得及完全刷到磁盘中，这个时候就会出现日志丢失
    - Linux 提供了 fsync 函数 可将指定文件的内容强制从内核刷到磁盘，但是 fsync 是一个磁盘 IO 操作，很耗时
    - 在生产环境的服务器中，Redis  通常是每隔 1s 左右执行一次 fsync 操作，周期 1s 可以配置的
  - AOF 最多丢失一秒的数据
  - AOF的日志是通过一个叫非常可读的方式记录的，这样的特性就适合做灾难性数据误删除的紧急恢复了，比如公司的实习生通过flushall清空了所有的数据，只要这个时候后台重写还没发生，你马上拷贝一份AOF日志文件，把最后一条flushall命令删了就完事了
  - AOF 文件较大，比 RDB 文件大
- RDB
  - 快照
  - 内存数据的二进制序列化形式
  - 使用多进程机制(COW机制)来实现快照持久化，不会影响 Redis 正常读写功能
    - fork（多进程）
    - Redis 在持久化时会调用 glibc 的函数 fork 产生一个子进程，快照持久化晚期交给子进程来处理，父进程继续处理客户端请求，子进程刚刚产生时，它和父进程共享内存里面的代码段和数据，子进程不会修改现有的内存数据结构，它只是对数据结构进行遍历读取，序列化写到磁盘中
  - 适合做 冷备，运维设置一个定时任务，定时同步到远端的服务器，比如阿里云的服务，这样一旦线上挂了，可以迅速复制一份远端的数据恢复线上数据
- 因为 快照时通过开启子进程的方式进行的，比较消耗资源；AOF 的 fsync 是一个耗时的 IO 操作，会降低Redis 的性能，同时也会增加系统 IO 负担，所以 Redis 主节点不会进行持久化操作，持久化操作都是在从节点进行
- Redis 4.0 混合持久化
  - 将 RDB 文件的内容和增量的 AOF 日志文件存在一起
  - 这里的 AOF 日志不再是全量的日志，而是自持久化开始到持久化结束的这段时间发生的增量 AOF 日志，这部分日志文件很小
  - Redis 重启的时候，可以先加载 RDB 的内容，然后再重放增量 AOF 日志就可以了，效率很高

## 管道

- pipeline

## 高可用

- Redis 不满足「一致性」要求，满足「可用性」，即 满足AP 理论，同时Redis可以保证最终一致性
  - Redis 的主从数据时异步同步的，即不满足一致性要求，当客户端在 Redis 的主节点修改了数据后，立即安徽，即使在主从网络断开的情况下，主节点依旧可正常对外提供修改服务，即满足可用性
  - Redis 保证最终一致性，从节点会努力追赶主节点，最终从节点的状态会和主节点的状态保持一致
- 数据同步机制
  - 主从同步
    - Redis 同步的时指令流，主节点将修改的指令记录在本地的内存 buffer 中，然后异步将 buffer 中的指令同步到从节点，从节点一边执行同步的指令流来达到和主节点一样的状态，一边向主节点反馈自己同步到哪里了（offset 偏移量）
    - 内存 buffer 是有限的，所以 Redis 主库不能讲所有的指令都记录在内存的 buffer 中，Redis 的复制内存 buffer 时一个定长的环形数组，如果数组内容满了，就从头开始覆盖前面的内容
    - 如果网络不好，从节点短时间内无法和主节点同步，当网络恢复时，Redis 的主节点中那些没有同步的指令在 buffer 中可能已经被后续的指令覆盖掉了，这时候就需要快照同步
  - 快照同步
    - 首先在主库上进行一次 bgsave 将当前内存的数据全部快照到磁盘文件中，然后将快照文件的内容全部传送到从节点，从节点收到快照文件，立即执行一次全量加载，加载之前先要把当前内存的数据情况，加载完毕后通知主节点继续进行增量同步
    - 在整个快照同步进行的过程中，主节点的复制 buffer 还在不停的往前移动，如果快照同步的时间过程或者复制 buffer 调小，都会导致同步期间的增量指令在复制 buffer 中被覆盖，会导致快照同步完成后无法进行增量复制，然后会再次发起快照同步，可能会陷入快照同步的死循环
  - 当启动一台slave 的时候，他会发送一个psync命令给master ，如果是这个slave第一次连接到master，他会触发一个全量复制。master就会启动一个线程，生成RDB快照，还会把新的写请求都缓存在内存中，RDB文件生成后，master会将这个RDB发送给slave的，slave拿到之后做的第一件事情就是写进本地的磁盘，然后加载进内存，然后master会把内存里面缓存的那些新增的数据都发给slave
  - 数据传输的时候断网了或者服务器挂了怎么办啊？
    - 传输过程中有什么网络问题啥的，会自动重连的，并且连接之后会把缺少的数据补上的，偏移量
- 哨兵模式 Sentinel
  - Sentinel 负责持续监控主从节点的健康，当主节点挂掉时，自动选择一个最优的朱从节点切换为主节点，客户端来连接集群时，会首先连接 sentinel，通过 sentinel 来查询主节点的地址，然后再去连接主节点进行数据交互
  - 当主节点发生故障时，客户端会重新向 sentinel 要地址，sentinel 会将最新的主节点地址告诉客户端
  - 主要功能
    - 集群监控：负责监控 Redis master 和 slave 进程是否正常工作
    - 消息通知：如果某个 Redis 实例有故障，那么哨兵负责发送消息作为报警通知给管理员
    - 故障转移：如果 master node 挂掉了，会自动转移到 slave node 上
    - 配置中心：如果故障转移发生了，通知 client 客户端新的 master 地址
- 集群模式 Cluster
  - Redis Cluster 将所有数据划分为 16384 的 slots，每个节点复制其中一部分槽位，槽位的信息存储在每个节点中
  - 当客户端来连接集群时，它会得到一份集群的槽位配置信息，这样当客户的要查找某个 key 时，可以直接定位到目标节点
  - Redis Cluster 的每个节点会将集群的配置信息持久化到配置文件中，所以必须确保配置文件是可写的，而且精力不要依靠人工修改配置文件

## 哨兵模式

### Sentinel 做到故障转移时基于三个定时任务

- 每隔 10s 每个sentinel会对master节点和slave节点执行info命令
  - 作用就是发现slave节点，并且确认主从关系，因为redis-Sentinel节点启动的时候是知道master节点的，但是不知道相应的slave节点的信息
- 每隔 2s，sentinel 都会通过 master 节点内部的 channel 来交换信息（基于发布订阅）
  - 作用是通过 master 节点的频道来交互每个 sentinel 对 master 节点的判断信息
-  每隔 1s 每个 sentinel 对其他的redis节点（master，slave，sentinel）执行ping操作，对于master来说
  - 若超过30s内没有回复，就对该master进行主观下线并询问其他的Sentinel节点是否可以客观下线

### sentinel没有配置从节点信息如何知道从节点信息的？

- 每隔10秒，sentinel进行向主节点发送info命令，用于发现新的slave节点

### 如何判断一个节点的需要主观下线的？

- 每隔1秒每个sentinel对其他的redis节点（master，slave，sentinel）执行ping操作，对于master来说 若超过down-after-milliseconds内没有回复，就对该节点进行主观下线并询问其他的Sentinel节点是否可以客观下线

### 主观下线

- 每隔1秒每个sentinel对其他的redis节点（master，slave，sentinel）执行ping操作，若超过down-after-milliseconds内没有回复，就对该节点进行主观下线

### 客观下线

- 当sentinel主观下线的节点是主节点时，sentinel会通过命令sentinel is-master-down-by-addr来询问其sentinel对主节点的判断，如果超过quorum个数就认为主节点需要客观下线, 所有sentinel节点对redis节点失败达成共识

### 选举领头 Sentinel

- raft 算法
- 每个做主观下线的 sentinel 节点向其他 sentinel 节点发送命令，要求将自己设置为领导者
- 选举时是先到先得
- 收到命令的 sentinel 节点如果没有同意通过其他 sentinel 节点发送的命令，那么将同意该请求，否则拒绝
- 如果某个sentinel被半数以上的sentinel设置成领头，那么该sentinel既为领头
- 如果在限定时间内，没有选举出领头sentinel 或者有多个 sentinel 节点成为了领导者，那么暂定一段时间，再选举

### 选举从服务为 Master

- 选择健康状态从节点（排除主观下线、断线），排除5秒钟没有心跳的、排除主节点失联超过10*down-after-millisecends
- 选择优先级高的从节点优先级列表
- 选择偏移量大的
  - 偏移量大，数据最新
- 选择runid小的
  - runid 小表示启动的时间越早，数据越完整

### 故障转移

- 对选举出来的从 slave 执行 slave of no one 命令
- 更新应用程序端的链接到新的主节点
- 对其他从节点变更 master 为新的节点
- 修复原来的 master 并将其设置为新 master 的 slave
- 当已下线的服务重新上线时，sentinel 会向其发送 slaveof 命令，让其成为新主的从

## 过期策略

- 过期的 key 集合
  - Redis 会将每个设置了过期时间的 key 放入到这个独立的字典中，以后会定时遍历这个字典来删除到期的 key
- 定时扫描策略
  - Redis 默认每秒进行十次过期扫描，它不会遍历过期字典中所有的 key，而是采用了一种简单的贪心策略
    - 从过期字典中 随机 20 个 key
    - 删除这 20 个 key 中已经过期的 key
    - 如果过期的 key 比率超过 1/4，那就重复步骤 1
    - 为了防止过期扫描不会出现循环过度，导致线程卡死，算法还增加了扫描超时时间不超过 25 ms
  - 尽量将 key 的过期时间 随机化，防止同时有大批量的 key 过期，导致 Redis 卡顿
- 惰性删除
- 从库过期策略
  - 从库不会进行过期扫描，主库在 key 到期时，会在 AOF 文件里增加一条 del 指令，同步到所有的从库
  - 因为指令是异步的，所以可能会导致主库过期的 key 在从库中并没有过期

## 淘汰机制 LRU

- 当 Redis 内存超出物理内存限制时，内存的数据会开始和磁盘产生频繁的交换(swap)，交换会让 Redis 的性能急剧下降，为了限制最大内存，可以通过 maxmemory 来限制内存内存超出期望大小
- 当实际内存超出 maxmemory 时，Redis 提供了几种可选策略
  - noevication
    - 不会继续服务写请求（del 请求可以继续服务）,读请求可以继续进行,这样可以保证不会丢失数据，但是会让现实的业务不能持续进行
    - 默认的淘汰策略
  - volatile-lru
    - 淘汰设置了过期时间的 key，最少使用的 key 优先被淘汰
    - 没有设置过期时间的 key 不会被淘汰
  - volatile-ttl
    - key 的剩余过期时间 ttl 越小月有限被淘汰
  - volatile-random
    - 随机淘汰过期 key 集合中随机的 key
  - allkeys-lru
    - 淘汰的 key 对象是全体的 key 集合，最少使用的 key 有限被淘汰
    - 没有设置过期时间的 key 也会被淘汰
  - allkeys-random
    - 随机淘汰所有 key 集合中随机的 key
  - volatile-xxx 和 allkeys-xxx 区别
    - volatile-xxx 策略只会正对带过期时间的 key 进行淘汰
    - allkeys-xxx 策略会对所有的 key 进行淘汰
    - 如果你只是拿 Redis 做缓存，那应该使用 allkeys-xxx，客户端写缓存时不必携带过期时间
    - 如果你想同时使用 Redis 的持久化功能，那就使用 volatile-xxx 策略，这样可以保留没有设置过期时间的 key，它们是永久的 key 不会被 LRU 算法淘汰
- LRU 算法
  - 实现 LRU 算法需要 key/value 字典，还需要附加一个链表，链表的元素排列顺序就是元素最近被访问的时间顺序，当字典的某个元素被访问时，它在链表的位置会被移动到表头，当空间满时，移除尾部元素即可

## 常见问题

### 缓存雪崩

- 同一时间大面积的 key 过期，导致所有的请求都打到 数据库上
- 解决
  - 缓存过期时间 添加随机值
  - 集群部署
  - 热点数据 永不过期

### 缓存穿透

- 缓存和数据库都没有的数据，而用户不断发起请求，全部打到数据库，最终会击垮数据库
- 解决方案
  - 对参数做校验，不合法的参数直接return，比如小于0的id或者id特别大的数据直接过滤掉
    - 这里我想提的一点就是，我们在开发程序的时候都要有一颗“不信任”的心，就是不要相信任何调用方，比如你提供了API接口出去，你有这几个参数，那我觉得作为被调用方，任何可能的参数情况都应该被考虑到，做校验，因为你不相信调用你的人，你不知道他会传什么参数给你
    - 举个简单的例子，你这个接口是分页查询的，但是你没对分页参数的大小做限制，调用的人万一一口气查 Integer.MAX_VALUE 一次请求就要你几秒，多几个并发你不就挂了么？是公司同事调用还好大不了发现了改掉，但是如果是黑客或者竞争对手呢？在你双十一当天就调你这个接口会发生什么，就不用我说了吧。这是之前的Leader跟我说的，我觉得大家也都应该了解下
  - 缓存中不存在，数据库也没有，可以将对应的 key 的 value 设置为null
  - 在运维层面上 对单个 IP 每秒访问次数做限制，超过限制直接过滤掉
  - 布隆过滤器
    - 利用高效的数据结构和算法快速判断出你这个Key是否在数据库中存在，不存在你return就好了，存在你就去查了数据库刷新到缓存再return
  - 被拦截的请求，做服务降级，给用户一个友好的提示页面

### 缓存击穿

- 缓存雪崩时大面积的缓存失效，导致数据库挂掉；而缓存击穿是有一个 key 非常热点，有大量的请求在访问这个key，这时候这个 key 过期时间到了，持续的大量请求就会达到数据库，导致数据库挂掉
- 解决
  - key失效时，采用互斥锁来同步数据库数据
  - 热点数据不失效

### 双写一致性

- 场景

  - 请求A进行写操作，删除缓存
  - 请求B查询发现缓存不存在
  - 请求B去数据库查询得到旧值
  - 请求B将旧值写入缓存
  - 请求A将新值写入数据库
  - 这是便出现了数据不一致问题。采用延时双删策略得以解决

- 延时双删

  - ```java
    public void write(String key,Object data){
        redisUtils.del(key);
        db.update(data);
        Thread.Sleep(100);
        redisUtils.del(key);
    }
    ```

  - 这么做，可以将1秒内所造成的缓存脏数据，再次删除。这个时间设定可根据俄业务场景进行一个调节

- 为什么是删除缓存，而不是更新缓存？

  - 如果你频繁修改一个缓存涉及的多个表，缓存也频繁更新。但是问题在于，这个缓存到底会不会被频繁访问到？
  - 有些数据时冷数据，所以没必要更新缓存，直接删除缓存即可

### 并发竞争

- 分布式锁

### 热点 key

- 缓存时间不失效
- 多级缓存
- 布隆过滤器
- 读写分离

## 限流

- setnx ex

  - 我们在使用Redis的分布式锁的时候，大家都知道是依靠了setnx的指令，在CAS（Compare and swap）的操作的时候，同时给指定的key设置了过期实践（expire），我们在限流的主要目的就是为了在单位时间内，有且仅有N数量的请求能够访问我的代码程序。所以依靠setnx可以很轻松的做到这方面的功能。比如我们需要在10秒内限定20个请求，那么我们在setnx的时候可以设置过期时间10，当请求的setnx数量达到20时候即达到了限流效果，当然这种做法的弊端是很多的，比如当统计1-10秒的时候，无法统计2-11秒之内，如果需要统计N秒内的M个请求，那么我们的Redis中需要保持N个key等等问题

- zset 窗口滑动、zset 会越来越大

  - 我们可以将请求打造成一个zset数组，当每一次请求进来的时候，value保持唯一，可以用UUID生成，而score可以用当前时间戳表示，因为score我们可以用来计算当前时间戳之内有多少的请求数量。而zset数据结构也提供了range方法让我们可以很轻易的获取到2个时间戳内有多少请求
  - 通过上述代码可以做到滑动窗口的效果，并且能保证每N秒内至多M个请求，缺点就是zset的数据结构会越来越大。实现方式相对也是比较简单的

- 令牌桶算法

  - 定时push、然后 leftpop

  - 令牌桶算法提及到输入速率和输出速率，当输出速率大于输入速率，那么就是超出流量限制了

  - 也就是说我们每访问一次请求的时候，可以从Redis中获取一个令牌，如果拿到令牌了，那就说明没超出限制，而如果拿不到，则结果相反

  - 依靠List的leftPop来获取令牌

    - ```java
      // 输出令牌
      public Response limitFlow2(Long id){
          Object result = redisTemplate.opsForList().leftPop("limit_list");
          if(result == null){
               return Response.ok("当前令牌桶中无令牌");
          }
          return Response.ok(articleDescription2);
      }
      ```

  - 再依靠Java的定时任务，定时往List中rightPush令牌，当然令牌也需要唯一性，所以我这里还是用UUID进行了生成

    - ```java
      // 10S的速率往令牌桶中添加UUID，只为保证唯一性
      @Scheduled(fixedDelay = 10_000,initialDelay = 0)
      public void setIntervalTimeTask(){
          redisTemplate.opsForList().rightPush("limit_list",UUID.randomUUID().toString());
      }
      ```

    - 

- 漏斗限流 funnel

- redis cell

## 多路复用IO

- read 会阻塞直到有数据读取或者资源关闭
- 写不会阻塞除非缓冲区满了
- 非阻塞IO 提供了一个选项 no_blocking 读写都不会阻塞，读多少写多少，取决于内核套接字字节分配
- 非阻塞IO 的问题，线程要读数据了，读了一点就返回，那线程什么时候又可以继续读？
- 通过 select 事件循环解决，但是效率低，现在都是 epoll

## 秒杀系统

### 高并发

- 单一职责，单独部署服务
- 缓存雪崩、穿透、击穿问题

### 超卖

- 加锁
- redis 判断库存和减库存两个动作，使用 Lua 脚本来解决原子性问题

### 恶意请求

- 布隆过滤器

### 链接暴露

- 链接加密，URL动态化，前端代码获取url后台校验才能通过

### Nginx

### 资源静态化

### 按钮控制

### 限流

- 前端限流：一般秒杀不会让你一直点的，一般都是点击一下或者两下然后几秒之后才可以继续点击，这也是保护服务器的一种手段
- 后端限流：秒杀的时候肯定是涉及到后续的订单生成和支付等操作，但是都只是成功的幸运儿才会走到那一步，那一旦100个产品卖光了，return了一个false，前端直接秒杀结束，然后你后端也关闭后续无效请求的介入了

### 库存预热

- 提前将商品的库存加载到redis内存中

### 消息队列

# Zookeeper

## 应用场景

- 统一命名、统一配置、统一集群、服务器节点动态上下线、分布式锁

## 特点

- zookeeper：一个领导者（Leader），多个跟随者（Follower）组成的集群
- 集群中只要有半数以上节点存活，zookeeper集群就能正常服务
- 全局数据一致，每个server保存一份相同的数据副本，client无论连接到哪个server，数据都是一致的
- 更新请求一致，来自同一个client的更新请求按其发生顺序依次执行
- 数据更新原子性，一次数据要么成功，要么失败
- 实时性，在一定时间范围内，client能读到最新的数据

## 数据结构

- zookeeper的数据结构整体上可以看作是一课树，每个节点称作一个ZNode，每一个ZNode默认能够存储1MB的数据，每个ZNode都可以通过其路径唯一标识

## 节点类型

- 持久（Persistent）：客户端和服务器端断开连接后，创建的节点不删除
- 短暂（Ephemeral）：客户端和服务器断开连接后，创建的节点自己删除
- 持久化目录节点：客户端与zookeeper断开连接后，该节点依旧存在
- 持久化顺序编号目录节点：客户端与zookeeper断开连接后，该节点依旧存在，只是zookeeper给该节点名称进行顺序编号（在名称的后面增加数字）
  - 创建znode时设置顺序标识，znode名称后附加一个值，顺序号是一个单调递增的计数器，有父节点维护
  - 在分布式系统中，顺序号可以被用于为所有的事件进行全局排序，这样客户端可以通过顺序号推断事件的顺序
- 临时目录节点：客户端与zookeeper断开连接后，该节点被删除
- 临时顺序编号目录节点：客户端与zookeeper断开连接后，该节点被删除，只是zookeeper给该节点名称进行顺序编号

## 节点信息

- cZxid：创建znode的更改的事务ID
- mZxid:：znode最后一次修改的更改的事务ID
- pZxid：znode关于添加和移除子结点更改的事务ID
- ctime：表示znode的创建时间 (以毫秒为单位)
- mtime：表示znode的最后一次修改时间 (以毫秒为单位)
- dataVersion：znode上数据变化的次数
- cversion：znode子结点变化的次数
- aclVersion znode结点上ACL变化的次数
- ephemeralOwner：如果znode是短暂类型的结点，这代表了znode拥有者的session ID，如果znode不是短暂类型的结点，这个值为0
- dataLength：znode数据域的长度
- numChildren：znode中子结点的个数
- 实现中Zxid是一个64为的数字，它高32位是epoch用来标识leader关系是否改变，每次一个leader被选出来，它都会有一个 新的epoch。低32位是个递增计数
- 对节点的每一个操作都将致使这个节点的版本号增加。每个节点维护着三个版本号，他们分别为：
  - version：节点数据版本号
  - cversion：子节点版本号
  - aversion：节点所拥有的ACL版本号

## 节点操作

- create  创建znode（父znode必须存在）
- delete  删除znode（znode没有子节点）
- exists  测试znode是否存在，并获取他的元数据
- getAcl/setAcl  为znode获取/设置Acl
- getChildren  获取znode所有子节点列表
- getData/setData  获取/设置znode的相关数据
- sync  使客户端的znode视图与zookeeper同步
  - create -e /test content	创建短暂节点
  - create -s /testa content   创建带序号的节点
  - set /testa content  修改节点
  - get /testb watch  对testb增加监听，当节点数据发生变化时，会收到消息，仅仅监听一次，下次更改就不再生效
  - delete /testb  删除节点
  - rmr /test  递归删除包括子节点
  - stat /test  查看节点状态
- 更新ZooKeeper操作是有限制的。delete或setData必须明确要更新的Znode的版本号，我们可以调用exists找到。如果版本号不匹配，更新将会失败
- 更新ZooKeeper操作是非阻塞式的。因此客户端如果失去了一个更新(由于另一个进程在同时更新这个Znode)，他可以在不阻塞其他进程执行的情况下，选择重新尝试或进行其他操作

## watch 机制

- ZooKeeper可以为所有的读操作设置watch，这些读操作包括：exists()、getChildren()及getData()。watch事件是一次性的触发器，当watch的对象状态发生改变时，将会触发此对象上watch所对应的事件。watch事件将被异步地发送给客户端，并且ZooKeeper为watch机制提供了有序的一致性保证。理论上，客户端接收watch事件的时间要快于其看到watch对象状态变化的时间
- 首先创建一个main线程，在main线程中创建zookeeper客户端，这时就会创建两个线程，一个负责网络连接通信（connet），一个负责监听（listener）。通过connect线程将注册的监听事件发送到zookeeper。在zookeeper的注册监听器列表中将注册的监听事件添加到列表中。zookeeper监听到有数据或路径变化，就会将这个消息发送给listener线程。listener线程内部就调用我们自己写的业务逻辑方法process()方法。
- 常用监听
  - 监听节点的数据变化  get path watch
  - 监听子节点增减的变化  ls path watch
- ZooKeeper所管理的watch可以分为两类：
  - 数据watch(data  watches)：getData和exists负责设置数据watch
  - 孩子watch(child watches)：getChildren负责设置孩子watch
- 可以通过操作返回的数据来设置不同的watch：
  - getData和exists：返回关于节点的数据信息
  - getChildren：返回孩子列表
- 因此
  - 一个成功的setData操作将触发Znode的数据watch
  - 一个成功的create操作将触发Znode的数据watch以及孩子watch
  - 一个成功的delete操作将触发Znode的数据watch以及孩子watch
- watch注册与触发
  - exists操作上的watch，在被监视的Znode创建、删除或数据更新时被触发
  - getData操作上的watch，在被监视的Znode删除或数据更新时被触发。在被创建时不能被触发，因为只有Znode一定存在，getData操作才会成功
  - getChildren操作上的watch，在被监视的Znode的子节点创建或删除，或是这个Znode自身被删除时被触发。可以通过查看watch事件类型来区分是Znode，还是他的子节点被删除：NodeDelete表示Znode被删除，NodeDeletedChanged表示子节点被删除
  - Watch由客户端所连接的ZooKeeper服务器在本地维护，因此watch可以非常容易地设置、管理和分派。当客户端连接到一个新的服务器 时，任何的会话事件都将可能触发watch。另外，当从服务器断开连接的时候，watch将不会被接收。但是，当一个客户端重新建立连接的时候，任何先前 注册过的watch都会被重新注册

## 写流程

- client向zookeeper的server1上写数据，发送一个写请求
- 如果server1不是leader，那么server1会把接受到的请求进一步转发给leader，因为每个zookeeper的server里面有一个是leader。这个leader会将写请求广播给各个server，比如server1和server2，各个server写成功后就会通知leader
- 当leader收到大多数的server数据写成功了，那么就说明数据写成功了，如果这里三个节点的话，只要两个节点数据写成功了，那么就认为数据写成功了，写成功之后，leader会告诉server1数据写成功了，只要大部分server写成功了即可，它们之间再进行同步

## 分布式锁

## 选举机制

- 半数机制：集群中半数以上机器存活，集群可用，所以zookeeper适合安装奇数台服务器
- 工作原理：
  - Zookeeper的核心是原子广播，这个机制保证了各个server之间的同步。实现这个机制的协议叫做Zab协议。Zab协议有两种模式，它们分别是恢复模式和广播模式。当服务启动或者在领导者崩溃后，Zab就进入了恢复模式，当领导者被选举出来，且大多数server的完成了和leader的状态同步以后，恢复模式就结束了。状态同步保证了leader和server具有相同的系统状态
  - 一旦leader已经和多数的follower进行了状态同步后，他就可以开始广播消息了，即进入广播状态。这时候当一个server加入zookeeper服务中，它会在恢复模式下启动，发现leader，并和leader进行状态同步。待到同步结束，它也参与消息广播。Zookeeper服务一直维持在Broadcast状态，直到leader崩溃了或者leader失去了大部分的followers支持
  - 为了保证事务的顺序一致性，zookeeper采用了递增的事务id号（zxid）来标识事务。所有的提议（proposal）都在被提出的时候加上了zxid。实现中zxid是一个64位的数字，它高32位是epoch用来标识 leader关系是否改变，每次一个leader被选出来，它都会有一个新的epoch，标识当前属于那个leader的统治时期。低32位用于递增计数
  - 当leader崩溃或者leader失去大多数的follower，这时候zk进入恢复模式，恢复模式需要重新选举出一个新的leader，让所有的server都恢复到一个正确的状态，每个Server启动以后都询问其它的Server它要投票给谁
  - 对于其他server的询问，server每次根据自己的状态都回复自己推荐的leader的id和上一次处理事务的zxid（系统启动时每个server都会推荐自己）收到所有Server回复以后，就计算出zxid最大的哪个Server，并将这个Server相关信息设置成下一次要投票的Server
  - 计算这过程中获得票数最多的的sever为获胜者，如果获胜者的票数超过半数，则改server被选为leader。否则，继续这个过程，直到leader被选举出来，leader就会开始等待server连接，Follower连接leader，将最大的zxid发送给leader，Leader根据follower的zxid确定同步点。完成同步后通知follower 已经成为uptodate状态。Follower收到uptodate消息后，又可以重新接受client的请求进行服务了
  - 每个Server在工作过程中有三种状态：
    - LOOKING：寻找Leader状态。当服务器处于该状态时，它会认为当前集群中没有Leader，因此需要进入Leader选举状态
    - FOLLOWING：跟随者状态。表明当前服务器角色是Follower
    - LEADING：领导者状态。表明当前服务器角色是Leader
    - OBSERVING：观察者状态。表明当前服务器角色是Observer

## 选举流程

### 初始化时期Leader选举

- 若进行Leader选举，则至少需要两台机器，这里选取3台机器组成的服务器集群为例。在集群初始化阶段，当有一台服务器Server1启动时，其单独无法进行和完成Leader选举，当第二台服务器Server2启动时，此时两台机器可以相互通信，每台机器都试图找到Leader，于是进入Leader选举过程。选举过程如下
  - 每个Server发出一个投票。由于是初始情况，Server1和Server2都会将自己作为Leader服务器来进行投票，每次投票会包含所推举的服务器的myid和ZXID，使用(myid, ZXID)来表示，此时Server1的投票为(1, 0)，Server2的投票为(2, 0)，然后各自将这个投票发给集群中其他机器
  - 接受来自各个服务器的投票。集群的每个服务器收到投票后，首先判断该投票的有效性，如检查是否是本轮投票、是否来自LOOKING状态的服务器
  - 处理投票。针对每一个投票，服务器都需要将别人的投票和自己的投票进行PK，PK规则如下
    - 优先检查ZXID。ZXID比较大的服务器优先作为Leader
    - 如果ZXID相同，那么就比较myid。myid较大的服务器作为Leader服务器
  - 对于Server1而言，它的投票是(1, 0)，接收Server2的投票为(2, 0)，首先会比较两者的ZXID，均为0，再比较myid，此时Server2的myid最大，于是更新自己的投票为(2, 0)，然后重新投票，对于Server2而言，其无须更新自己的投票，只是再次向集群中所有机器发出上一次投票信息即可
  - 统计投票。每次投票后，服务器都会统计投票信息，判断是否已经有过半机器接受到相同的投票信息，对于Server1、Server2而言，都统计出集群中已经有两台机器接受了(2, 0)的投票信息，此时便认为已经选出了Leader
  - 改变服务器状态。一旦确定了Leader，每个服务器就会更新自己的状态，如果是Follower，那么就变更为FOLLOWING，如果是Leader，就变更为LEADING

### 运行时期Leader选举

- 在Zookeeper运行期间，Leader与非Leader服务器各司其职，即便当有非Leader服务器宕机或新加入，此时也不会影响Leader，但是一旦Leader服务器挂了，那么整个集群将暂停对外服务，进入新一轮Leader选举，其过程和启动时期的Leader选举过程基本一致。假设正在运行的有Server1、Server2、Server3三台服务器，当前Leader是Server2，若某一时刻Leader挂了，此时便开始Leader选举。选举过程如下
  - 变更状态。Leader挂后，余下的非Observer服务器都会讲自己的服务器状态变更为LOOKING，然后开始进入Leader选举过程
  - 每个Server会发出一个投票。在运行期间，每个服务器上的ZXID可能不同，此时假定Server1的ZXID为123，Server3的ZXID为122；在第一轮投票中，Server1和Server3都会投自己，产生投票(1, 123)，(3, 122)，然后各自将投票发送给集群中所有机器
  - 接收来自各个服务器的投票。与启动时过程相同
  - 处理投票。与启动时过程相同，此时，Server1将会成为Leader
  - 统计投票。与启动时过程相同
  - 改变服务器的状态。与启动时过程相同

### Leader选举算法分析

- 在3.4.0后的Zookeeper的版本只保留了TCP版本的FastLeaderElection选举算法。当一台机器进入Leader选举时，当前集群可能会处于以下两种状态
  - 集群中已经存在 Leader
  - 集群中不存在 Leader
- 对于集群中已经存在Leader而言，此种情况一般都是某台机器启动得较晚，在其启动之前，集群已经在正常工作，对这种情况，该机器试图去选举Leader时，会被告知当前服务器的Leader信息，对于该机器而言，仅仅需要和Leader机器建立起连接，并进行状态同步即可。而在集群中不存在Leader情况下则会相对复杂，其步骤如下
  - 第一次投票。无论哪种导致进行Leader选举，集群的所有机器都处于试图选举出一个Leader的状态，即LOOKING状态，LOOKING机器会向所有其他机器发送消息，该消息称为投票。投票中包含了SID（服务器的唯一标识）和ZXID（事务ID），(SID, ZXID)形式来标识一次投票信息。假定Zookeeper由5台机器组成，SID分别为1、2、3、4、5，ZXID分别为9、9、9、8、8，并且此时SID为2的机器是Leader机器，某一时刻，1、2所在机器出现故障，因此集群开始进行Leader选举。在第一次投票时，每台机器都会将自己作为投票对象，于是SID为3、4、5的机器投票情况分别为(3, 9)，(4, 8)， (5, 8)
  - 变更投票。每台机器发出投票后，也会收到其他机器的投票，每台机器会根据一定规则来处理收到的其他机器的投票，并以此来决定是否需要变更自己的投票，这个规则也是整个Leader选举算法的核心所在，其中术语描述如下
    - vote_sid：接收到的投票中所推举Leader服务器的SID
    - vote_zxid：接收到的投票中所推举Leader服务器的ZXID
    - self_sid：当前服务器自己的SID
    - self_zxid：当前服务器自己的ZXID
    - 每次对收到的投票的处理，都是对(vote_sid, vote_zxid)和(self_sid, self_zxid)对比的过程
      - 规则一：如果vote_zxid大于self_zxid，就认可当前收到的投票，并再次将该投票发送出去
      - 规则二：如果vote_zxid小于self_zxid，那么坚持自己的投票，不做任何变更
      - 规则三：如果vote_zxid等于self_zxid，那么就对比两者的SID，如果vote_sid大于self_sid，那么就认可当前收到的投票，并再次将该投票发送出去
      - 规则四：如果vote_zxid等于self_zxid，并且vote_sid小于self_sid，那么坚持自己的投票，不做任何变更
  - 确定Leader。经过第二轮投票后，集群中的每台机器都会再次接收到其他机器的投票，然后开始统计投票，如果一台机器收到了超过半数的相同投票，那么这个投票对应的SID机器即为Leader。此时Server3将成为Leader
    - 由上面规则可知，通常那台服务器上的数据越新（ZXID会越大），其成为Leader的可能性越大，也就越能够保证数据的恢复。如果ZXID相同，则SID越大机会越大

## Observer

- Zookeeper需保证高可用和强一致性
- 为了支持更多的客户端，需要增加更多Server
- Server增多，投票阶段延迟增大，影响性能
- 权衡伸缩性和高吞吐率，引入Observer
- Observer不参与投票
- Observers接受客户端的连接，并将写请求转发给leader节点
- 加入更多Observer节点，提高伸缩性，同时不影响吞吐率

## 脑裂

- 什么是脑裂？
  - 由于心跳超时（网络原因导致的）认为leader死了，但其实leader还存活着
  - 由于假死会发起新的leader选举，选举出一个新的leader，但旧的leader网络又通了，导致出现了两个leader ，有的客户端连接到老的leader，而有的客户端则连接到新的leader
  - 出现脑裂就会导致数据不一致
  - 实际上Zookeeper集群中是不会轻易出现脑裂问题的，原因在于过半机制
- zookeeper 的脑裂时什么原因导致的？
  - 主要原因是Zookeeper集群和Zookeeper client判断超时并不能做到完全同步，也就是说可能一前一后，如果是集群先于client发现，那就会出现上面的情况。同时，在发现并切换后通知各个客户端也有先后快慢。一般出现这种情况的几率很小，需要leader节点与Zookeeper集群网络断开，但是与其他集群角色之间的网络没有问题，还要满足上面那些情况，但是一旦出现就会引起很严重的后果，数据不一致
- zookeeper 是如何解决脑裂的？
  - zooKeeper默认采用了Quorums这种方式来防止"脑裂"现象。即只有集群中超过半数节点投票才能选举出Leader。这样的方式可以确保leader的唯一性,要么选出唯一的一个leader,要么选举失败。在zookeeper中Quorums作用如下：
    - 集群中最少的节点数用来选举leader保证集群可用
    - 通知客户端数据已经安全保存前集群中最少数量的节点数已经保存了该数据。一旦这些节点保存了该数据，客户端将被通知已经安全保存了，可以继续其他任务。而集群中剩余的节点将会最终也保存了该数据
  - zookeeper除了可以采用上面默认的Quorums方式来避免出现"脑裂"，还可以可采用下面的预防措施：
    - 添加冗余的心跳线，例如双线条线，尽量减少“裂脑”发生机会
    - 启用磁盘锁。正在服务一方锁住共享磁盘，"裂脑"发生时，让对方完全"抢不走"共享磁盘资源。但使用锁磁盘也会有一个不小的问题，如果占用共享盘的一方不主动"解锁"，另一方就永远得不到共享磁盘。现实中假如服务节点突然死机或崩溃，就不可能执行解锁命令。后备节点也就接管不了共享资源和应用服务。于是有人在HA中设计了"智能"锁。即正在服务的一方只在发现心跳线全部断开（察觉不到对端）时才启用磁盘锁。平时就不上锁了
    - 设置仲裁机制。例如设置参考IP（如网关IP），当心跳线完全断开时，2个节点都各自ping一下 参考IP，不通则表明断点就出在本端，不仅"心跳"、还兼对外"服务"的本端网络链路断了，即使启动（或继续）应用服务也没有用了，那就主动放弃竞争，让能够ping通参考IP的一端去起服务。更保险一些，ping不通参考IP的一方干脆就自我重启，以彻底释放有可能还占用着的那些共享资源

## zookeeper是如何保证事务的顺序一致性的？

- zookeeper采用了全局递增的事务Id来标识，所有的proposal（提议）都在被提出的时候加上了zxid，zxid实际上是一个64位的数字，高32位是epoch（时期; 纪元; 世; 新时代）用来标识leader周期，如果有新的leader产生出来，epoch会自增，低32位用来递增计数。当新产生proposal的时候，会依据数据库的两阶段过程，首先会向其他的server发出事务执行请求，如果超过半数的机器都能执行并且能够成功，那么就会开始执行

## zookeeper节点宕机如何处理？

- Zookeeper本身也是集群，推荐配置不少于3个服务器。Zookeeper自身也要保证当一个节点宕机时，其他节点会继续提供服务
- 如果是一个Follower宕机，还有2台服务器提供访问，因为Zookeeper上的数据是有多个副本的，数据并不会丢失
- 如果是一个Leader宕机，Zookeeper会选举出新的Leader
- ZK集群的机制是只要超过半数的节点正常，集群就能正常提供服务。只有在ZK节点挂得太多，只剩一半或不到一半节点能工作，集群才失效
- 所以
  - 3个节点的cluster可以挂掉1个节点(leader可以得到2票>1.5)
  - 2个节点的cluster就不能挂掉任何1个节点了(leader可以得到1票<=1)

## Zookeeper对节点的watch监听通知是永久的吗？为什么不是永久的？

- 一个Watch事件是一个一次性的触发器，当被设置了Watch的数据发生了改变的时候，则服务器将这个改变发送给设置了Watch的客户端，以便通知它们
- 为什么不是永久的，举个例子，如果服务端变动频繁，而监听的客户端很多情况下，每次变动都要通知到所有的客户端，给网络和服务器造成很大压力
- 一般是客户端执行getData(“/节点A”,true)，如果节点A发生了变更或删除，客户端会得到它的watch事件，但是在之后节点A又发生了变更，而客户端又没有设置watch事件，就不再给客户端发送。在实际应用中，很多情况下，我们的客户端不需要知道服务端的每一次变动，我只要最新的数据即可

# Dubbo

## Netty

## SPI

- ExtensionLoader

## 容错机制

- Failover Cluster
  - 默认容错机制
  - 失败自动切换，当出现失败，重试其它服务器 [1]。通常用于读操作，但重试会带来更长延迟。可通过 retries="2" 来设置重试次数(不含第一次)
- Failfast Cluster
  - 快速失败，只发起一次调用，失败立即报错。通常用于非幂等性的写操作，比如新增记录
- Failsafe Cluster
  - 失败安全，出现异常时，直接忽略。通常用于写入审计日志等操作
- Failback Cluster
  - 失败自动恢复，后台记录失败请求，定时重发。通常用于消息通知操作。
- Forking Cluster
  - 并行调用多个服务器，只要一个成功即返回。通常用于实时性要求较高的读操作，但需要浪费更多服务资源。可通过 forks="2" 来设置最大并行数

## 负载均衡

- Random LoadBalance
  - 随机，按权重设置随机概率
  - 在一个截面上碰撞的概率高，但调用量越大分布越均匀，而且按概率使用权重后也比较均匀，有利于动态调整提供者权重
- RoundRobin LoadBalance
  - 轮询，按公约后的权重设置轮询比率
  - 存在慢的提供者累积请求的问题，比如：第二台机器很慢，但没挂，当请求调到第二台时就卡在那，久而久之，所有请求都卡在调到第二台上
- LeastActive LoadBalance
  - 最少活跃调用数，相同活跃数的随机，活跃数指调用前后计数差
  - 使慢的提供者收到更少请求，因为越慢的提供者的调用前后计数差会越大
- ConsistentHash LoadBalance
  - 一致性 Hash，相同参数的请求总是发到同一提供者
  - 当某一台提供者挂时，原本发往该提供者的请求，基于虚拟节点，平摊到其它提供者，不会引起剧烈变动

## 协议

### dubbo://

- 缺省协议
- 特性
  - 连接个数：单连接
  - 连接方式：长连接
  - 传输协议：TCP
  - 传输方式：NIO 异步传输
  - 序列化：Hessian 二进制序列化
  - 适用范围：传入传出参数数据包较小（建议小于100K），消费者比提供者个数多，单一消费者无法压满提供者，尽量不要用 dubbo
  - 协议传输大文件或超大字符串
  - 适用场景：常规远程服务方法调用
- 约束：
  - 参数及返回值需实现 Serializable 接口
  - 参数及返回值不能自定义实现 List, Map, Number, Date, Calendar 等接口，只能用 JDK 自带的实现，因为 hessian 会做特殊处理，自定义实现类中的属性值都会丢失
  - 接口增加方法，对客户端无影响，如果该方法不是客户端需要的，客户端不需要重新部署。输入参数和结果集中增加属性，对客户端无影响，如果客户端并不需要新属性，不用重新部署
  - 输入参数和结果集属性名变化，对客户端序列化无影响，但是如果客户端不重新部署，不管输入还是输出，属性名变化的属性值是获取不到的
  - 总结：服务器端和客户端对领域对象并不需要完全一致，而是按照最大匹配原则
- 为什么要消费者比提供者个数多?
  - 因 dubbo 协议采用单一长连接，假设网络为千兆网卡，根据测试经验数据每条连接最多只能压满 7MByte(不同的环境可能不一样，供参考)，理论上 1 个服务提供者需要 20 个服务消费者才能压满网卡
- 为什么不能传大包?
  - 因 dubbo 协议采用单一长连接，如果每次请求的数据包大小为 500KByte，假设网络为千兆网卡，每条连接最大 7MByte(不同的环境可能不一样，供参考)，单个服务提供者的 TPS(每秒处理事务数)最大为：128MByte / 500KByte = 262。单个消费者调用单个服务提供者的 TPS(每秒处理事务数)最大为：7MByte / 500KByte = 14。如果能接受，可以考虑使用，否则网络将成为瓶颈
- 为什么采用异步单一长连接?
  - 因为服务的现状大都是服务提供者少，通常只有几台机器，而服务的消费者多，可能整个网站都在访问该服务，比如 Morgan 的提供者只有 6 台提供者，却有上百台消费者，每天有 1.5 亿次调用，如果采用常规的 hessian 服务，服务提供者很容易就被压跨，通过单一连接，保证单一消费者不会压死提供者，长连接，减少连接握手验证等，并使用异步 IO，复用线程池，防止 C10K 问题

### rmi://

- RMI 协议采用 JDK 标准的 java.rmi.* 实现，采用阻塞式短连接和 JDK 标准序列化方式
- 特性
  - 连接个数：多连接
  - 连接方式：短连接
  - 传输协议：TCP
  - 传输方式：同步传输
  - 序列化：Java 标准二进制序列化
  - 适用范围：传入传出参数数据包大小混合，消费者与提供者个数差不多，可传文件。
  - 适用场景：常规远程服务方法调用，与原生RMI服务互操作
- 约束
  - 参数及返回值需实现 Serializable 接口
  - dubbo 配置中的超时时间对 RMI 无效，需使用 java 启动参数设置：-Dsun.rmi.transport.tcp.responseTimeout=3000，参见下面的 RMI 配置

### hessian://

- Hessian协议用于集成 Hessian 的服务，Hessian 底层采用 Http 通讯，采用 Servlet 暴露服务，Dubbo 缺省内嵌 Jetty 作为服务器实现
- Dubbo 的 Hessian 协议可以和原生 Hessian 服务互操作，即：
  - 提供者用 Dubbo 的 Hessian 协议暴露服务，消费者直接用标准 Hessian 接口调用
  - 或者提供方用标准 Hessian 暴露服务，消费方用 Dubbo 的 Hessian 协议调用
- 特性
  - 连接个数：多连接
  - 连接方式：短连接
  - 传输协议：HTTP
  - 传输方式：同步传输
  - 序列化：Hessian二进制序列化
  - 适用范围：传入传出参数数据包较大，提供者比消费者个数多，提供者压力较大，可传文件。
  - 适用场景：页面传输，文件传输，或与原生hessian服务互操作
- 约束
  - 参数及返回值需实现 Serializable 接口
  - 参数及返回值不能自定义实现 List, Map, Number, Date, Calendar 等接口，只能用 JDK 自带的实现，因为 hessian 会做特殊处理，自定义实现类中的属性值都会丢失
- Hessian 是 Caucho 开源的一个 RPC 框架，其通讯效率高于 WebService 和 Java 自带的序列化

### http://

- 基于 HTTP 表单的远程调用协议，采用 Spring 的 HttpInvoker 实现
- 特性
  - 连接个数：多连接
  - 连接方式：短连接
  - 传输协议：HTTP
  - 传输方式：同步传输
  - 序列化：表单序列化
  - 适用范围：传入传出参数数据包大小混合，提供者比消费者个数多，可用浏览器查看，可用表单或URL传入参数，暂不支持传文件
  - 适用场景：需同时给应用程序和浏览器 JS 使用的服务
- 约束
  - 参数及返回值需符合 Bean 规范

### webservice://

### thrift://

- 当前 dubbo 支持的 thrift 协议是对 thrift 原生协议的扩展，在原生协议的基础上添加了一些额外的头信息，比如 service name，magic number 等

### memcached://

### redis://

### rest://

### grpc://

## 服务暴露过程

- ServiceBean
  - Dubbo 服务导出过程始于 Spring 容器发布刷新事件，Dubbo 在接收到事件后，会立即执行服务导出逻辑
  - 整个逻辑大致可分为三个部分
    - 第一部分是前置工作，主要用于检查参数，组装 URL
    - 第二部分是导出服务，包含导出服务到本地 (JVM)，和导出服务到远程两个过程
    - 第三部分是向注册中心注册服务，用于服务发现
  - 服务导出的入口方法是 ServiceBean 的 onApplicationEvent。onApplicationEvent 是一个事件响应方法，该方法会在收到 Spring 上下文刷新事件后执行服务导出操作
  - 实现了initializingBean，通过 afterPropertiesSet 方法
    - get provider
    - set provider
    - 各种信息，保存在 ServiceBean
- doExport
  - 检查参数
  - doExporturl 暴露URL
    - 加载注册中心
    - 循环协议，代理工厂获取invoke 封装为 invoker
      - 通过 SPI 获取协议实例
      - 本地暴露
      - 远程暴露 注册 url 到 注册中心
      - 启动服务器 Netty 监听端口

## 服务引用

- ReferenceBean
  - 当我们的服务被注入到其他类中时，Spring 会第一时间调用 getObject 方法，并由该方法执行服务引用逻辑
  - 按照惯例，在进行具体工作之前，需先进行配置检查与收集工作。接着根据收集到的信息决定服务用的方式，有三种
    - 第一种是引用本地 (JVM) 服务
    - 第二是通过直连方式引用远程服务
    - 第三是通过注册中心引用远程服务
  - 不管是哪种引用方式，最后都会得到一个 Invoker 实例。如果有多个注册中心，多个服务提供者，这个时候会得到一组 Invoker 实例，此时需要通过集群管理类 Cluster 将多个 Invoker 合并成一个实例。合并后的 Invoker 实例已经具备调用本地或远程服务的能力了，但并不能将此实例暴露给用户使用，这会对用户业务代码造成侵入。此时框架还需要通过代理工厂类 (ProxyFactory) 为服务接口生成代理类，并让代理类去调用 Invoker 逻辑。避免了 Dubbo 框架代码对业务代码的侵入，同时也让框架更容易使用
  - ReferenceBean 通过 FactoryBean 类的 getObject 方法来实现 初始化

## 服务字典 Directory

- 服务目录中存储了一些和服务提供者有关的信息，通过服务目录，服务消费者可获取到服务提供者的信息，比如 ip、端口、服务协议等。通过这些信息，服务消费者就可通过 Netty 等客户端进行远程调用。在一个服务集群中，服务提供者数量并不是一成不变的，如果集群中新增了一台机器，相应地在服务目录中就要新增一条服务提供者记录。或者，如果服务提供者的配置修改了，服务目录中的记录也要做相应的更新。如果这样说，服务目录和注册中心的功能不就雷同了吗？确实如此，这里这么说是为了方便大家理解。实际上服务目录在获取注册中心的服务配置信息后，会为每条配置信息生成一个 Invoker 对象，并把这个 Invoker 对象存储起来，这个 Invoker 才是服务目录最终持有的对象。Invoker 有什么用呢？看名字就知道了，这是一个具有远程调用功能的对象。讲到这大家应该知道了什么是服务目录了，它可以看做是 Invoker 集合，且这个集合中的元素会随注册中心的变化而进行动态调整
- StaticDirectory
  - StaticDirectory 即静态服务目录，顾名思义，它内部存放的 Invoker 是不会变动的。所以，理论上它和不可变 List 的功能很相似
- RegistryDirectory
  - RegistryDirectory 是一种动态服务目录，实现了 NotifyListener 接口。当注册中心服务配置发生变化后，RegistryDirectory 可收到与当前服务相关的变化。收到变更通知后，RegistryDirectory 可根据配置变更信息刷新 Invoker 列表。RegistryDirectory 中有几个比较重要的逻辑，第一是 Invoker 的列举逻辑，第二是接收服务配置变更的逻辑，第三是 Invoker 列表的刷新逻辑

## 服务路由 Router

- 服务目录在刷新 Invoker 列表的过程中，会通过 Router 进行服务路由，筛选出符合路由规则的服务提供者。服务路由包含一条路由规则，路由规则决定了服务消费者的调用目标，即规定了服务消费者可调用哪些服务提供者。Dubbo 目前提供了三种服务路由实现，分别为条件路由 ConditionRouter、脚本路由 ScriptRouter 和标签路由 TagRouter

## 集群 Cluster

- 为了避免单点故障，现在的应用通常至少会部署在两台服务器上。对于一些负载比较高的服务，会部署更多的服务器。这样，在同一环境下的服务提供者数量会大于1。对于服务消费者来说，同一环境下出现了多个服务提供者。这时会出现一个问题，服务消费者需要决定选择哪个服务提供者进行调用。另外服务调用失败时的处理措施也是需要考虑的，是重试呢，还是抛出异常，亦或是只打印异常等。为了处理这些问题，Dubbo 定义了集群接口 Cluster 以及 Cluster Invoker。集群 Cluster 用途是将多个服务提供者合并为一个 Cluster Invoker，并将这个 Invoker 暴露给服务消费者。这样一来，服务消费者只需通过这个 Invoker 进行远程调用即可，至于具体调用哪个服务提供者，以及调用失败后如何处理等问题，现在都交给集群模块去处理。集群模块是服务提供者和服务消费者的中间层，为服务消费者屏蔽了服务提供者的情况，这样服务消费者就可以专心处理远程调用相关事宜。比如发请求，接受服务提供者返回的数据等。这就是集群的作用
- Dubbo 提供了多种集群实现，包含但不限于 Failover Cluster、Failfast Cluster 和 Failsafe Cluster 等
- 工作原理
  - 集群工作过程可分为两个阶段，第一个阶段是在服务消费者初始化期间，集群 Cluster 实现类为服务消费者创建 Cluster Invoker 实例，即上图中的 merge 操作。第二个阶段是在服务消费者进行远程调用时。以 FailoverClusterInvoker 为例，该类型 Cluster Invoker 首先会调用 Directory 的 list 方法列举 Invoker 列表（可将 Invoker 简单理解为服务提供者）。Directory 的用途是保存 Invoker，可简单类比为 List<Invoker>，其实现类 RegistryDirectory 是一个动态服务目录，可感知注册中心配置的变化，它所持有的 Invoker 列表会随着注册中心内容的变化而变化。每次变化后，RegistryDirectory 会动态增删 Invoker，并调用 Router 的 route 方法进行路由，过滤掉不符合路由规则的 Invoker。当 FailoverClusterInvoker 拿到 Directory 返回的 Invoker 列表后，它会通过 LoadBalance 从 Invoker 列表中选择一个 Invoker。最后 FailoverClusterInvoker 会将参数传给 LoadBalance 选择出的 Invoker 实例的 invoke 方法，进行真正的远程调用

## 负载均衡 LoadBalance

- LoadBalance 中文意思为负载均衡，它的职责是将网络请求，或者其他形式的负载“均摊”到不同的机器上。避免集群中部分服务器压力过大，而另一些服务器比较空闲的情况。通过负载均衡，可以让每台服务器获取到适合自己处理能力的负载。在为高负载服务器分流的同时，还可以避免资源浪费，一举两得。负载均衡可分为软件负载均衡和硬件负载均衡。在我们日常开发中，一般很难接触到硬件负载均衡。但软件负载均衡还是可以接触到的，比如 Nginx。在 Dubbo 中，也有负载均衡的概念和相应的实现。Dubbo 需要对服务消费者的调用请求进行分配，避免少数服务提供者负载过大。服务提供者负载过大，会导致部分请求超时。因此将负载均衡到每个服务提供者上，是非常必要的。Dubbo 提供了4种负载均衡实现，分别是基于权重随机算法的 RandomLoadBalance、基于最少活跃调用数算法的 LeastActiveLoadBalance、基于 hash 一致性的 ConsistentHashLoadBalance，以及基于加权轮询算法的 RoundRobinLoadBalance

# SpringCloud

## Alibaba SpringCloud

## 微服务的优点缺点?说下开发项目中遇到的坑?

- 优点
  - 每个服务直接足够内聚，代码容易理解
  - 开发效率高，一个服务只做一件事，适合小团队开发
  - 松耦合，有功能意义的服务
  - 可以用不同语言开发，面向接口编程。
  - 易于第三方集成
  - 微服务只是业务逻辑的代码，不会和HTML,CSS或其他界面结合.
  - 可以灵活搭配，连接公共库/连接独立库
- 缺点
  - 分布式系统的责任性
  - 多服务运维难度加大
  - 系统部署依赖，服务间通信成本，数据一致性，系统集成测试，性能监控

## SpringCloud 和Dubbo区别？

- 服务调用方式 dubbo是RPC springcloud Rest Api
- 注册中心,dubbo 是zookeeper springcloud是eureka，也可以是zookeeper
- 服务网关,dubbo本身没有实现，只能通过其他第三方技术整合，springcloud有Zuul路由网关，作为路由服务器，进行消费者的请求分发,springcloud支持断路器，与git完美集成配置文件支持版本控制，事物总线实现配置文件的更新与服务自动装配等等一系列的微服务架构要素
- dubbo 基于TCP，SpringCloud 基于Http

## Eureka和Zookeeper区别？

- Eureka取CAP的CP，注重可用性，Zookeeper取CAP的AP注重一致性
- Zookeeper在选举期间注册服务瘫痪，虽然服务最终会恢复，但选举期间不可用
- eureka的自我保护机制，会导致一个结果就是不会再从注册列表移除因长时间没收到心跳而过期的服务。依然能接受新服务的注册和查询请求，但不会被同步到其他节点。不会服务瘫痪
- Zookeeper有Leader和Follower角色，Eureka各个节点平等
- Zookeeper采用过半数存活原则，Eureka采用自我保护机制解决分区问题
- Eureka 已经停止维护了

## 阿里巴巴注册中心Nacos

## 什么是服务熔断？什么是服务降级？

- 服务直接的调用，比如在高并发情况下出现进程阻塞，导致当前线程不可用，慢慢的全部线程阻塞，导致服务器雪崩
- 服务熔断：相当于保险丝，出现某个异常，直接熔断整个服务，而不是一直等到服务超时。通过维护一个自己的线程池，当线程到达阈值的时候就启动服务降级，如果其他请求继续访问就直接返回fallback的默认值

## 什么是Ribbon？

- Ribbon客户端组件提供一系列完善的配置项如连接超时，重试等。简单的说，就是在配置文件中列出Load Balancer（简称LB）后面所有的机器，Ribbon会自动的帮助你基于某种规则（如简单轮询，随机连接等）去连接这些机器。我们也很容易使用Ribbon实现自定义的负载均衡算法

## 什么是Feigin？它的优点是什么？

- feign采用的是基于接口的注解
- feign整合了ribbon，具有负载均衡的能力
- 整合了Hystrix，具有熔断的能力
- 启动类添加@EnableFeignClients
- 定义一个接口@FeignClient(name=“xxx”)指定调用哪个服务

## Ribbon和Feign的区别？

- Ribbon都是调用其他服务的，但方式不同
- 启动类注解不同，Ribbon是@RibbonClient feign的是@EnableFeignClients
- 服务指定的位置不同，Ribbon是在@RibbonClient注解上声明，Feign则是在定义抽象方法的接口中使用@FeignClient声明
- 调用方式不同，Ribbon需要自己构建http请求，模拟http请求然后使用RestTemplate发送给其他服务，步骤相当繁琐。Feign需要将调用的方法定义成抽象方法即可

# 消息队列

## RocketMQ

- 优点
  - 单机支持 1 万以上持久化队列
  - RocketMQ 的所有消息都是持久化的，先写入系统 PageCache，然后刷盘，可以保证内存与磁盘都有一份数据，访问时，直接从内存读取
  - 模型简单，接口易用（JMS 的接口很多场合并不太实用）
  - 性能非常好，可以大量堆积消息在broker中
  - 支持多种消费，包括集群消费、广播消费等
  - 各个环节分布式扩展设计，主从HA
  - 开发度较活跃，版本更新很快
- 缺点
  - 支持的客户端语言不多，目前是java及c++，其中c++不成熟
  - RocketMQ社区关注度及成熟度也不及前两者
  - 没有web管理界面，提供了一个CLI(命令行界面)管理工具带来查询、管理和诊断各种问
  - 没有在 mq 核心中去实现JMS等接口
- 高可用
  - Producer：消息发送者
  - Consumer：消息消费者
  - Broker：暂存消息和传输消息
  - NameServer：管理broker
  - Topic：区分消息的种类；一个发送者可以发送消息给一个或者多个Topic；一个消息的接收者可以订阅一个或者多个Topic消息
  - Message Queue：相当于是Topic的分区；用于并行发送消息和接收消息
- 集群特点
  - NameServer是一个几乎无状态节点，可以集群直接部署，节点直接不需要任何信息同步
  - Broker分为Master和Slave，一个Master可以对应多个Slave，但是一个Slave只能对应一个Master，Master和Slave的对应关系通过指定相同的BrokerName，不同的BrokerId来定义，BrokerId为0表示Master，1表示Slave。Master也可以部署多个，每个Broker与NameServer集群中的节点建立长连接，定时注册Topic信息到所有的NameServer
  - Producer与NameServer集群中的一个节点（随机选择）建立长连接，定期从NameServer取Topic路由信息，并向提供Topic服务和Master建立长连接，且定时向Master发送心跳。Producer完全无状态，可集群部署
  - Consumer与NameServer集群中的一个节点（随机选择）建立长连接，定期从NameServer取Topic路由信息，并向提供Topic服务和Master、Slave建立长连接，且定时向Master、Slave发送心跳。Consumer既可以从Master订阅消息，也可以从Slave订阅消息，订阅规则有Broker配置决定
- 集群模式
  - 单Master模式
    - 这种方式风险较大，一旦Broker重启或者岩机时，会导致整个服务不可用。不建议线上使用，可用于本地测试
  - 多Master模式 
    - 一个集群无Slave，全是Master，如2个Master或者3个Master
      - 优点：配置简单，单个Master岩机或者重启维护对应用无影响，在磁盘配置为RAID10时，即使机器岩机不可恢复情况下，由于RAID10磁盘非常可靠，消息也不会丢失，性能最高
      - 缺点：单台机器岩机期间，这台机器上未被消息的消息在机器恢复之前不可订阅，消息实时性会受到影响。
  - 多Master多Slave模式（异步） 
    - 每个Master配置一个Slave，多对Master-Slave，HA采用异步复制方式，主备有短暂消息延迟（毫秒级）
      - 优点：即使磁盘损坏，消息丢失的非常少，且消息实时性不会受影响，同时Master岩机后，消费者仍然能从Slave消费，而且此过程对应用透明，不需要人工干预，性能通多Master模式几乎一样
      - 缺点：Master岩机，磁盘损坏情况下丢失少量消息
  - 多Master多Slave模式（同步）
    - 每个Master配置一个Slave，多对Master-Slave，HA采用同步复制方式，即只有主备都写成功了，才向应用返回成功
      - 优点：数据与服务都无单点故障，Master岩机情况下，消息无延迟，服务可用性与数据可用性都非常高
      - 缺点：性能比异步复制模式略低（大约10%），发送单个消息的延迟略高，且目前版本在主节点岩机后，备机不能自动切换为主机
- 集群工作流程
  - 启动NameServer，NameServer启动后监听端口，等待Broker、Producer、Consumer连接上来，相当于一个路由控制中心
  - Broker启动，跟所有的NameServer保持长连接，定时发送心跳包。心跳包包含当前Broker信息（IP+端口）以及存储所有Topic信息。注册成功后，NameServer集群中就有Topic跟Broker的映射关系
  - 收发消息前，先创建Topic，创建Topic时需要指定该Topic要存储在哪些Broker上，也可以在发送消息时自动创建Topic。
    Producer发送消息，启动时先跟NameServer集群中的其中一台建立长连接，并从NameServer中获取当前发送的Topic存在哪些Broker上，轮询从队列列表中选择一个队列，然后与队列所在的Broker建立长连接从而向Broker发送消息
  - Consumer跟Producer类似，跟其中一台NameServer建立长连接，获取当前订阅Topic存在哪些Broker上，然后直接跟Broker建立通信通道，开始消费消息
- 心跳机制
  - 单个Broker跟所有Namesrv保持心跳请求，心跳间隔为30秒，心跳请求中包括当前Broker所有的Topic信息。Namesrv会反查Broer的心跳信息， 如果某个Broker在2分钟之内都没有心跳，则认为该Broker下线，调整Topic跟Broker的对应关系。但此时Namesrv不会主动通知Producer、Consumer有Broker宕机
  - Consumer跟Broker是长连接，会每隔30秒发心跳信息到Broker。Broker端每10秒检查一次当前存活的Consumer，若发现某个Consumer 2分钟内没有心跳， 就断开与该Consumer的连接，并且向该消费组的其他实例发送通知，触发该消费者集群的负载均衡(rebalance)
  - 生产者每30秒从Namesrv获取Topic跟Broker的映射关系，更新到本地内存中。再跟Topic涉及的所有Broker建立长连接，每隔30秒发一次心跳。 在Broker端也会每10秒扫描一次当前注册的Producer，如果发现某个Producer超过2分钟都没有发心跳，则断开连接
- 队列数量
  - 每个主题可设置队列个数，自动创建主题时默认4个，需要顺序消费的消息发往同一队列，比如同一订单号相关的几条需要顺序消费的消息发往同一队列， 顺序消费的特点的是，不会有两个消费者共同消费任一队列，且当消费者数量小于队列数时，消费者会消费多个队列。至于消息重复，在消 费端处理。RocketMQ 4.3+支持事务消息，可用于分布式事务场景(最终一致性)
- 管理界面：rocketmq-externals  可以在GitHub上直接搜索
- 消息类型
  - 同步消息
  - 异步消息
  - 单向消息：发送完后就不关心了 不管是否发送成功 用于一些不重要的消息通知
  - 顺序消息
    - 消息有序指的是可以按照消息的发送顺序来消费（FIFO）。RocketMQ可以严格保证消息有序，可以分为分区有序或者全局有序
    - 顺序消费的原理：在默认的情况下消息发送会采取Round Robin轮询方式把消息发送到不同的queue（分区队列）；而消费消息的时候从多个queue上拉取消息，这种情况发送和消费是不能保证顺序。但是如果控制发送的顺序消息只依次发送到同一个queue中，消费的时候只从这个queue上依次拉取，则就保证了顺序。当发送和消费者参与的queue只有一个，则是全局有序。如果是多个queue参与，则分为分区有序，即相对每个queue，消息都是有序的
  - 延时消息
  - 批量消息
  - 过滤消息
    - tag过滤
    - sql过滤
  - 事务消息
    - 事务消息不支持延时消息和批量消息
    - 为了避免单个消息被检查太多次而导致半队列消息累积，RocketMQ默认将单个消息的检查次数限制为15次，但是用户可以通过Broker配置文件的transactionCheckMax参数修改次限制。如果已经检查某条消息超过N次（N=transactionCheckMax）则Broker将丢弃此消息，并在默认情况下同时打印错误日志。用户可以通过重写AbstarctTransactionCheckListener类修改次行为
    - 事务消息将在Broker配置文件中的参数transactionMsgTimeout这样的特定时间长度之后被检查。当发送事务消息时，用户还可以通过设置用户属性CHECK_IMMUNINY_TIME_IN_SECONDS来改变限制，该参数优先于transactionMsgTimeout参数
    - 提交给用户的目标主题消息可能会失败，目前这依日志的记录而定。它的高可用性通过RocketMQ本身的高可用性机制来保证，如果希望确保事务消息不丢失，并且事务完整性得到保证，建议使用同步的双写机制
    - 事务消息的生产者ID不能与其他类型消息的生产者ID共享。与其他类型消息不同，事务消息允许反向查询，MQ服务器能通过它们的生产者ID查询消费者
- 消费消息
  - DefaultMQProducer consumer = new DefaultMQProducer("group1");
  - consumer.setMessageModel(MessageModel.CLUSTERING); 设置消息消费模式模式
  - 默认是MessageModel.CLUSTERING 负载均衡模式
  - 负载均衡模式
  - 广播模式：每个消费者都会消费相同的消息
- 组件
  - 消费者组
    - 消费者组（Consumer Group）一类 Consumer 的集合名称，这类 Consumer 通常消费同一类消息并且消费逻辑一致，所以将这些 Consumer 分组在一起。消费者组与生产者组类似，都是将相同角色的分组在一起并命名，分组是个很精妙的概念设计，RocketMQ 正是通过这种分组机制，实现了天然的消息负载均衡。消费消息时通过 Consumer Group 实现了将消息分发到多个消费者服务器实例，比如某个 Topic 有9条消息，其中一个 Consumer Group 有3个实例（3个进程或3台机器），那么每个实例将均摊3条消息，这也意味着我们可以很方便的通过加机器来实现水平扩展
  - 拉取型消费者
    - 拉取型消费者（Pull Consumer）主动从消息服务器拉取信息，只要批量拉取到消息，用户应用就会启动消费过程，所以 Pull 称为主动消费型
  - 推送型消费者
    - 推送型消费者（Push Consumer）封装了消息的拉取、消费进度和其他的内部维护工作，将消息到达时执行的回调接口留给用户应用程序来实现。所以 Push 称为被动消费类型，但从实现上看还是从消息服务器中拉取消息，不同于 Pull 的是 Push 首先要注册消费监听器，当监听器处触发后才开始消费消息
  - 名称服务器 NameServer
    - 名称服务器（NameServer）用来保存 Broker 相关元信息并给 Producer 和 Consumer 查找 Broker 信息。NameServer 被设计成几乎无状态的，可以横向扩展，节点之间相互之间无通信，通过部署多台机器来标记自己是一个伪集群。每个 Broker 在启动的时候会到 NameServer 注册，Producer 在发送消息前会根据 Topic 到 NameServer 获取到 Broker 的路由信息，Consumer 也会定时获取 Topic 的路由信息。所以从功能上看应该是和 ZooKeeper 差不多，据说 RocketMQ 的早期版本确实是使用的 ZooKeeper ，后来改为了自己实现的 NameServer 
  - 消息
    - 消息（Message）就是要传输的信息。一条消息必须有一个主题（Topic），主题可以看做是你的信件要邮寄的地址。一条消息也可以拥有一个可选的标签（Tag）和额处的键值对，它们可以用于设置一个业务 key 并在 Broker 上查找此消息以便在开发期间查找问题
  - 主题
    - 主题（Topic）可以看做消息的规类，它是消息的第一级类型。比如一个电商系统可以分为：交易消息、物流消息等，一条消息必须有一个 Topic 。Topic 与生产者和消费者的关系非常松散，一个 Topic 可以有0个、1个、多个生产者向其发送消息，一个生产者也可以同时向不同的 Topic 发送消息。一个 Topic 也可以被 0个、1个、多个消费者订阅
  - 标签
    - 标签（Tag）可以看作子主题，它是消息的第二级类型，用于为用户提供额外的灵活性。使用标签，同一业务模块不同目的的消息就可以用相同 Topic 而不同的 Tag 来标识。比如交易消息又可以分为：交易创建消息、交易完成消息等，一条消息可以没有 Tag 。标签有助于保持您的代码干净和连贯，并且还可以为 RocketMQ 提供的查询系统提供帮助
  - 消息队列
    - 消息队列（Message Queue），主题被划分为一个或多个子主题，即消息队列。一个 Topic 下可以设置多个消息队列，发送消息时执行该消息的 Topic ，RocketMQ 会轮询该 Topic 下的所有队列将消息发出去
  - 消息消费模式
    - 消息消费模式有两种：集群消费（Clustering）和广播消费（Broadcasting）。默认情况下就是集群消费，该模式下一个消费者集群共同消费一个主题的多个队列，一个队列只会被一个消费者消费，如果某个消费者挂掉，分组内其它消费者会接替挂掉的消费者继续消费。而广播消费消息会发给消费者组中的每一个消费者进行消费
- RocketMQ具有很好动态伸缩能力(非顺序消息)，伸缩性体现在Topic和Broker两个维度
  - Topic维度：假如一个Topic的消息量特别大，但集群水位压力还是很低，就可以扩大该Topic的队列数，Topic的队列数跟发送、消费速度成正比
  - Broker维度：如果集群水位很高了，需要扩容，直接加机器部署Broker就可以。Broker起来后向Namesrv注册，Producer、Consumer通过Namesrv 发现新Broker，立即跟该Broker直连，收发消息
- 消息存储
  - 磁盘如果使用得当，磁盘的速度完全可以匹配上网络的数据传输速度。目前高性能磁盘，顺序写速度为600MB/s，超过了一般网卡的传输速度。但是磁盘随机写的速度只有大概100kb/s，和顺序写的性能差了6000倍！因为有如此巨大的速度差别，好的消息队列系统会比普通的消息队列系统速度快多个数量级。RocketMQ的消息采用顺序写，保证了消息存储的速度
  - RocketMQ采用零拷贝技术，即Java NIO中的MappedByteBuffer，提供消息存盘和网络发送的速度。采用MappedByteBuffer这种内存映射的方式有几个限制，其中之一是一次只能映射1.5~2.0G的文件至用户态的虚拟内存，这也是为何RocketMQ默认设置单个CommitLog日志提供数据文件为1G的原因
  - 消息存储结构：
    - RocketMQ消息的存储是由ConsumeQueue和CommitLog配合完成的，消息真正的物理存储文件是CommitLog，ConsumeQueue是消息的逻辑队列，类似数据库的索引文件，存储的是指向物理存储的地址，每个Topic下的每个MessageQueue都有一个对应的ConsumeQueue文件
    - IndexFile：为了消息查询提供了一种通过key和时间区间来查询消息的方法，这种通过IndexFile来查找消息的方法不影响发送与消费消息的主流程
    - RocketMQ的消息的存储是由ConsumeQueue和CommitLog配合来完成的，ConsumeQueue中只存储很少的数据，消息主体都是通过CommitLog来进行读写。 如果某个消息只在CommitLog中有数据，而ConsumeQueue中没有，则消费者无法消费，RocketMQ的事务消息实现就利用了这一点
      - CommitLog：是消息主体以及元数据的存储主体，对CommitLog建立一个ConsumeQueue，每个ConsumeQueue对应一个（概念模型中的）MessageQueue，所以只要有 CommitLog在，ConsumeQueue即使数据丢失，仍然可以恢复出来
      - ConsumeQueue：是一个消息的逻辑队列，存储了这个Queue在CommitLog中的起始offset，log大小和MessageTag的hashCode。每个Topic下的每个Queue都有一个对应的 ConsumeQueue文件，例如Topic中有三个队列，每个队列中的消息索引都会有一个编号，编号从0开始，往上递增。并由此一个位点offset的概念，有了这个概念，就可以对 Consumer端的消费情况进行队列定义
  - 刷盘机制
    - 同步
      - 在返回成功状态时，消息已经被写入到磁盘了。具体流程是，消息写入内存的PageCache后，立即通知刷盘线程刷盘，然后等待刷盘完成，刷盘线程执行完成后唤醒等待线程，返回消息成功的状态
    - 异步
      - 在返回写成功状态时，消息可能只是被写入了内存的PageCache，写操作的返回快，吞吐量达。当内存中的消息量积累到一定程度时，统一触发写磁盘动作，快速写入
    - 配置
      - 通过Broker配置文件里的flushDiskType配置，SYNC_FLUSH、ASYNC_FLUSH
- 高可用性机制
  - RocketMQ的高性能在于顺序写盘(CommitLog)、零拷贝和跳跃读(尽量命中PageCache)，高可靠性在于刷盘和Master/Slave，另外NameServer 全部挂掉不影响已经运行的Broker,Producer,Consumer
  - 刷盘和主从同步均为异步(默认)时，broker进程挂掉(例如重启)，消息依然不会丢失，因为broker shutdown时会执行持久化。 当物理机器宕机时，才有消息丢失的风险。另外，master挂掉后，消费者从slave消费消息，但slave不能写消息
  - Master角色支持读写操作，Slave角色只支持读操作
  - 如何保证消费高可用？
    - 在Consumer的配置文件中，并不需要设置从Master读还是从Slave读，当Master不可用或者繁忙的时候，Consumer会自动切换到Slave读。有了这个自动切换机制，即使当一个Master角色的机器出现故障后，Consumer仍然可以从Slave读取消息，不影响Consumer程序，这样就达到了消费端的高可用性
  - 如何保证发送高可用？
    - 在创建Topic时，把Topic的多个Message Queue创建在多个Broker组上（相同的Broker名称，不同的brokerId的机器组成一个Broker组），这样当一个Broker组的Master不可用后，其他组的Master仍然可用，Producer仍然可以发送消息，这样就达到了发送端的高可用性。RocketMQ目前还不支持把Slave自动转换为Master，如果机器资源不足，需要把Slave转换为Master，则需要手动停止Slave角色的Broker，更改配置文件，用新的配置启动Broker
  - 消息主从复制
    - 同步复制
      - 同步复制是等Master和Slave都写成功后，才反馈给客户端写成功状态；在同步复制方式下，如果Master出故障，Slave上有全部的备份数据，容易恢复，但是同步复制会增大数据写入延迟，降低系统吞吐量
    - 异步复制
      - 异步复制是只要Master成功后就反馈给客户端写成功状态；在异步复制下，系统拥有较低的延迟和较高的吞吐量，但是如果Master出故障，有些数据没有写入Slave，就会丢失数据
    - 配置
      - 通过Broker配置文件中的brokerRole参数设置：ASYNC_MASTER、SYNC_MASTER、SLAVE三个值中的一个
    - 实际场景使用
      - 刷盘方式采用异步，主从复制采用同步。这样即使有一台机器出故障，也能保证消息不丢失
- 负载均衡
  - 发送端负载均衡
    - Producer端，每个实例发送消息时，默认会轮询所有的message queue发送，以达到让消息平均落到不同的queue上，而由于queue可以散落到不同的broker，所以消息就发送到不同的broker下。默认采用Roundbin方式轮询每个queue消息，如：第一条消息发送到queue0，第二条消息发送到queue1，以此类推
  - 消费端负载均衡
    - 集群模式
      - Consumer端，每条消息只需要投递到订阅这个Topic的Consumer Group下的一个实例即可。RocketMQ采用主动拉取的方式并消费消息，在拉取的时候需要明确指定拉取哪一条message queue。而每当实例的数量有变更，都会触发一次所有实例的负载均衡，这时候会按照queue的数量和实例的数量平均分配queue给每个实例
        - 默认的分配算法是AllocateMessageQueueAveragely
        - AllocateMessageQueueAveragelyByCircle算法，也是平均分摊每一条queue，以环状轮流分配queue的形式
      - 需要注意的是，集群模式下，queue都是只允许分配只一个实例，这是由于如果多个实例同时消费一条queue的消息，由于拉取这些消息的是consumer主动控制的，那样会导致同一个消息在不同的实例下消费多次，所以算法上都是一个queue只分配一个实例，一个consumer实例可以允许同时分到不同的queue
      - 通过增加consumer的实例去分摊queue的消费，可以起到水平扩展的消费能力。而有实例下线的时候，会重写触发负载均衡，这时候原来分配的queue将分配到其他实例继续消费
      - 但是如果consumer实例是数量比queue的数量还多，多出来的consumer实例将无法分到queue，也就无法消费到消息，也就无法起到分摊负载的作用了。所以需要控制让consumer的总数量小于等于queue的数量
    - 广播模式
      - 由于广播模式是需要将每一条消息投递到每一个消费者实例上，所以就不存在消息分摊的说法。每一个consumer分配到queue的时候，所有消费者都分到所有的queue
- 消息重试
  - 顺序消息的重试
    - 当消费者消费消息失败后，RocketMQ会自动不断进行消息重试（每隔1秒），这时应用会出现消息消费被阻塞的情况。因此使用顺序消息时，务必保证应用能够及时监控消费失败的情况，避免阻塞现象的发生
  - 无序消息的重试
    - 无序消息（普通、定时、延时、事务），当消费者消费失败时，可以通过设置返回状态达到消息重试的结果。无序消息只针对集群方式有效，广播消息不支持消息重试，失败了就失败了
  - 重试次数
    - RocketMQ默认运行每条消息最多16次，每次间隔：10s 30s 1m 2m 3m 4m 5m 6m  ..... 1h 2h
    - 如果16次仍然失败了，则会将消息放入到死信队列中
  - 消费失败后，重试配置方式
    - 集群消费方式下，消息消费失败后期望消息重试，需要在消息监听器接口的实现明确进行配置（三种方式任选一种）：
      - 返回Action.ReconsumeLater（推荐）
      - 返回null
      - 抛出异常
  - 消息失败后，不重试配置方式
    - 集群消费方式下，消息消费失败后不希望消息在重试，需要捕获逻辑中可能抛出的异常，最终返回Action.CommitMessage
  - 自定义重试次数
    - RocketMQ运行Consumer启动的时候设置最大重试次数，重试时间间隔将按照如下策略
      - 最大重试次数小于16次，则按照RocketMQ默认的重试方式
      - 最大重试次数大于16次，超过16次的重试时间间隔均为每次2小时
- 死信队列
  - 加入死信队列的消息，最多保存3天，3天后会被删除，因此，请在死信消息产生后3天内处理
  - 一个死信队列对应一个Group ID，而不是对应单个消费者实例
  - 如果Group ID未产生死信消息，消息队列RocketMQ不会为其创建死信队列
  - 一个死信队列包含了Group ID产生的所有死信消息，不论该消息属于哪个Topic
- 消息幂等性

## Kafka

- Kafka是最初由Linkedin公司开发，是一个分布式、支持分区的（partition）、多副本的（replica），基于zookeeper协调的分布式消息系统，它的最大的特性就是可以实时的处理大量数据以满足各种需求场景：比如基于hadoop的批处理系统、低延迟的实时系统、storm/Spark流式处理引擎，web/nginx日志、访问日志，消息服务等等，用scala语言编写

### 特性

- 高吞吐量、低延迟：Kafka每秒可以处理几十万条消息，它的延迟最低只有几毫秒，每个topic可以分多个partition, consumer group 对partition进行consume操作
- 可扩展性：Kafka集群支持热扩展
- 持久性、可靠性：消息被持久化到本地磁盘，并且支持数据备份防止数据丢失
- 容错性：允许集群中节点失败（若副本数量为n,则允许n-1个节点失败）
- 高并发：支持数千个客户端同时读写

### 缺点

- Kafka单机超过64个队列/分区，Load会发生明显的飙高现象，队列越多，load越高，发送消息响应时间变长
- 使用短轮询方式，实时性取决于轮询间隔时间
- 消费失败不支持重试
- 支持消息顺序，但是一台代理宕机后，就会产生消息乱序
- 社区更新较慢
- Apache Kafka设计的首要目标是解决LinkedIn网站中海量的用户操作行为记录、页面浏览记录，后继的Apache Kafka版本也都是将“满足高数据吞吐量”作为版本优化的首要目标。为了达到这个目标，Apache Kafka甚至在其他功能方面上做了一定的牺牲，例如：消息的事务性

### Kafka的使用场景

- 日志收集：一个公司可以用Kafka可以收集各种服务的log，通过Kafka以统一接口服务的方式开放给各种consumer，例如hadoop、Hbase、Solr等
- 消息系统：解耦和生产者和消费者、缓存消息等
- 用户活动跟踪：Kafka经常被用来记录web用户或者app用户的各种活动，如浏览网页、搜索、点击等活动，这些活动信息被各个服务器发布到Kafka的topic中，然后订阅者通过订阅这些topic来做实时的监控分析，或者装载到hadoop、数据仓库中做离线分析和挖掘
- 运营指标：Kafka也经常用来记录运营监控数据。包括收集各种分布式应用的数据，生产各种操作的集中反馈，比如报警和报告
- 流式处理：比如spark streaming和storm
- 事件源

### Kafka Cluster结构

- Kafka集群中broker之间的关系不是主从关系，各个broker在集群中地位一样，我们可以随意的增加或删除任何一个broker节点
- Kafka将分区的多个副本分为两种角色：Leader和Follower
  - Leader Broker是主要服务节点，消息只会从消息生产者发送给Leader Broker，消息消费者也只会从Leader Broker中Pull消息
  - Follower Broker为副本服务节点，正常情况下不会公布给生产者或者消费者直接进行操作。Follower Broker服务节点将会主动从Leader Broker上Pull消息
- 整个Kafka集群中，可以有多个消息生产者。这些消息生产者可能在同一个物理节点上，也可能在不同的物理节点。它们都必须知道哪些Kafka Broker List是将要发送的目标：消息生产者会决定发送的消息将会送入Topic的哪一个分区（Partition）
- 消费者都是按照“组”的单位进行消息隔离：在同一个Topic下，Kafka会为不同的消费者组创建独立的index索引定位。也就是说当消息生产者发送一条消息后，同一个Topic下不同组的消费者都会收到这条信息
- 同一组下的消息消费者可以消费Topic下一个分区或者多个分区中的消息，但是一个分区中的消息只能被同一组下的某一个消息消费者所处理。也就是说，如果某个Topic下只有一个分区，就不能实现消息的负载均衡。另外Topic下的分区数量也只能是固定的，不可以在使用Topic时动态改变，这些分区在Topic被创建时使用命令行指定或者参考Broker Server中配置的默认值
- 当一个消费者上线，并且在消费消息之前。首先会通过zookeeper协调集群获取当前消费组中其他消费者的连接状态，并得到当前Topic下可用于消费的分区和该消费者组中其他消费者的对应关系。如果当前消费者发现Topic下所有的分区都已经有一一对应的消费者了，就将自己置于挂起状态（和broker、zookeeper的连接还是会建立，但是不会到分区Pull消息），以便在其他消费者失效后进行接替
- 如果当前消费者连接时，发现整个Kafka集群中存在一个消费者（记为消费者A）关联Topic下多个分区的情况，且消费者A处于繁忙无法处理这些分区下新的消息（即消费者A的上一批Pull的消息还没有处理完成）。这时新的消费者将接替原消费者A所关联的一个（或者多个）分区，并且一直保持和这个分区的关联
- 由于Kafka集群中只保证同一个分区（Partition）下消息队列中消息的顺序。所以当一个或者多个消费者分别Pull一个Topic下的多个消息分区时，您在消费者端观察的现象可能就是消息顺序是混乱的。这里我们一直在说消费者端的Pull行为，是指的Topic下分区中的消息并不是由Broker主动推送到（Push）到消费者端，而是由消费者端主动拉取（Pull）

### Kafka Broker Leader的选举

- Kakfa Broker集群受Zookeeper管理。所有的Kafka Broker节点一起去Zookeeper上注册一个临时节点，因为只有一个Kafka Broker会注册成功，其他的都会失败，所以这个成功在Zookeeper上注册临时节点的这个Kafka Broker会成为Kafka Broker Controller，其他的Kafka broker叫Kafka Broker follower。（这个过程叫Controller在ZooKeeper注册Watch）。这个Controller会监听其他的Kafka Broker的所有信息，如果这个Kafka broker controller宕机了，在zookeeper上面的那个临时节点就会消失，此时所有的Kafka broker又会一起去Zookeeper上注册一个临时节点，因为只有一个Kafka Broker会注册成功，其他的都会失败，所以这个成功在Zookeeper上注册临时节点的这个Kafka Broker会成为Kafka Broker Controller，其他的Kafka broker叫Kafka Broker follower
- 例如：一旦有一个broker宕机了，这个Kafka broker controller会读取该宕机broker上所有的partition在zookeeper上的状态，并选取ISR列表中的一个replica作为partition leader（如果ISR列表中的replica全挂，选一个幸存的replica作为leader; 如果该partition的所有的replica都宕机了，则将新的leader设置为-1，等待恢复，等待ISR中的任一个Replica“活”过来，并且选它作为Leader；或选择第一个“活”过来的Replica（不一定是ISR中的）作为Leader），这个broker宕机的事情，Kafka controller也会通知zookeeper，zookeeper就会通知其他的Kafka broker

### 消费者组 （Consumer Group）

- 各个consumer（consumer 线程）可以组成一个组（Consumer group ），partition中的每个message只能被组（Consumer group ）中的一个consumer（consumer 线程）消费，如果一个message可以被多个consumer（consumer 线程）消费的话，那么这些consumer必须在不同的组。Kafka不支持一个partition中的message由两个或两个以上的同一个consumer group下的consumer thread来处理，除非再启动一个新的consumer group。所以如果想同时对一个topic做消费的话，启动多个consumer group就可以了，但是要注意的是，这里的多个consumer的消费都必须是顺序读取partition里面的message，新启动的consumer默认从partition队列最头端最新的地方开始阻塞的读message。Kafka为了保证吞吐量，只允许同一个consumer group下的一个consumer线程去访问一个partition。如果觉得效率不高的时候，可以加partition的数量来横向扩展，那么再加新的consumer thread去消费。如果想多个不同的业务都需要这个topic的数据，起多个consumer group就好了，大家都是顺序的读取message，offset的值互不影响。这样没有锁竞争，充分发挥了横向的扩展性，吞吐量极高。这也就形成了分布式消费的概念
- 当启动一个consumer group去消费一个topic的时候，无论topic里面有多个少个partition，无论我们consumer group里面配置了多少个consumer thread，这个consumer group下面的所有consumer thread一定会消费全部的partition；即便这个consumer group下只有一个consumer thread，那么这个consumer thread也会去消费所有的partition。因此，最优的设计就是，consumer group下的consumer thread的数量等于partition数量，这样效率是最高的
- 一个consumer group下，无论有多少个consumer，这个consumer group一定会去把这个topic下所有的partition都消费了。当consumer group里面的consumer数量小于这个topic下的partition数量的时候，就会出现一个conusmer thread消费多个partition的情况，总之是这个topic下的partition都会被消费。如果consumer group里面的consumer数量等于这个topic下的partition数量的时候，此时效率是最高的，每个partition都有一个consumer thread去消费。当consumer group里面的consumer数量大于这个topic下的partition数量的时候，就会有一个consumer thread空闲。因此，我们在设定consumer group的时候，只需要指明里面有几个consumer数量即可，无需指定对应的消费partition序号，consumer会自动进行rebalance
- 如果producer的流量增大，当前的topic的parition数量=consumer数量，这时候的应对方式就是横向扩展：增加topic下的partition，同时增加这个consumer group下的consumer

### Consumer

- Consumer处理partition里面的message的时候是o（1）顺序读取的。所以必须维护着上一次读到哪里的offsite信息。high level API,offset存于Zookeeper中，low level API的offset由自己维护。一般来说都是使用high level api的。Consumer的delivery gurarantee，默认是读完message先commmit再处理message，autocommit默认是true，这时候先commit就会更新offsite+1，一旦处理失败，offsite已经+1，这个时候就会丢message；也可以配置成读完消息处理再commit，这种情况下consumer端的响应就会比较慢的，需要等处理完才行
- 一般情况下，一定是一个consumer group处理一个topic的message。Best Practice是这个consumer group里面consumer的数量等于topic里面partition的数量，这样效率是最高的，一个consumer thread处理一个partition。如果这个consumer group里面consumer的数量小于topic里面partition的数量，就会有consumer thread同时处理多个partition（这个是Kafka自动的机制，我们不用指定），但是总之这个topic里面的所有partition都会被处理到的。。如果这个consumer group里面consumer的数量大于topic里面partition的数量，多出的consumer thread就会闲着啥也不干，剩下的是一个consumer thread处理一个partition，这就造成了资源的浪费，因为一个partition不可能被两个consumer thread去处理。所以我们线上的分布式多个service服务，每个service里面的Kafka consumer数量都小于对应的topic的partition数量，但是所有服务的consumer数量只和等于partition的数量，这是因为分布式service服务的所有consumer都来自一个consumer group，如果来自不同的consumer group就会处理重复的message了（同一个consumer group下的consumer不能处理同一个partition，不同的consumer group可以处理同一个topic，那么都是顺序处理message，一定会处理重复的。一般这种情况都是两个不同的业务逻辑，才会启动两个consumer group来处理一个topic）
- 如果producer的流量增大，当前的topic的parition数量=consumer数量，这时候的应对方式就是很想扩展：增加topic下的partition，同时增加这个consumer group下的consumer

### Topic & Partition

- Topic是逻辑上的概念，而partition是物理上的概念，每个partition对应于一个log文件，该log文件存储的就是producer生成的数据。producer生产的数据会不断追加到该log文件的末端，而且每条数据都有自己的offset。消费者组中的每个消费者，都会实时记录自己消费到了哪个offset，以便出错恢复时，从上次的位置继续消费
- 由于生产者会不断地追加log文件到末尾，为防止log文件过大导致数据定位效率低下，Kafka采取了分片和索引的机制，每个partition分为多个segment，每个segment对应两个文件：.index文件和.log文件，这些文件都是在一个文件夹下，该文件夹的命名规则为：topic名称+分区序号。例如：first这个topic有三个分区，则对应的文件夹名称：first-0,first-1,first
  - index和log文件，以当前segment的第一条消息的offset命名。默认log文件的大小为1G，当超过这个数字 则会新建另一个log文件
- Topic相当于传统消息系统MQ中的一个队列queue，producer端发送的message必须指定是发送到哪个topic，但是不需要指定topic下的哪个partition，因为Kafka会把收到的message进行load balance，均匀的分布在这个topic下的不同的partition上（ hash(message) % [broker数量]  ）。物理上存储上，这个topic会分成一个或多个partition，每个partiton相当于是一个子queue。在物理结构上，每个partition对应一个物理的目录（文件夹），文件夹命名是[topicname]_[partition]_[序号]，一个topic可以有无数多的partition，根据业务需求和数据量来设置。在Kafka配置文件中可随时更高num.partitions参数来配置更改topic的partition数量，在创建Topic时通过参数指定parittion数量。Topic创建之后通过Kafka提供的工具也可以修改partiton数量
- 配置策略
  - 一个Topic的Partition数量大于等于Broker的数量，可以提高吞吐率
  - 同一个Partition的Replica尽量分散到不同的机器，高可用
- 当add a new partition的时候
  - partition里面的message不会重新进行分配，原来的partition里面的message数据不会变，新加的这个partition刚开始是空的，随后进入这个topic的message就会重新参与所有partition的load balance

### Partition Replica

- 每个partition可以在其他的Kafka broker节点上存副本，以便某个Kafka broker节点宕机不会影响这个Kafka集群。存replica副本的方式是按照Kafka broker的顺序存。例如有5个Kafka broker节点，某个topic有3个partition，每个partition存2个副本，那么partition1存broker1,broker2，partition2存broker2,broker3。。。以此类推（replica副本数目不能大于Kafka broker节点的数目，否则报错。这里的replica数其实就是partition的副本总数，其中包括一个leader，其他的就是copy副本）。这样如果某个broker宕机，其实整个Kafka内数据依然是完整的。但是，replica副本数越高，系统虽然越稳定，但是回来带资源和性能上的下降；replica副本少的话，也会造成系统丢数据的风险
- 怎样传送消息：producer先把message发送到partition leader，再由leader发送给其他partition follower。（如果让producer发送给每个replica那就太慢了）
- 在向Producer发送ACK前需要保证有多少个Replica已经收到该消息：根据ack配的个数而定
- 怎样处理某个Replica不工作的情况：如果这个部工作的partition replica不在ack列表中，就是producer在发送消息到partition leader上，partition leader向partition follower发送message没有响应而已，这个不会影响整个系统，也不会有什么问题。如果这个不工作的partition replica在ack列表中的话，producer发送的message的时候会等待这个不工作的partition replca写message成功，但是会等到time out，然后返回失败因为某个ack列表中的partition replica没有响应，此时Kafka会自动的把这个部工作的partition replica从ack列表中移除，以后的producer发送message的时候就不会有这个ack列表下的这个部工作的partition replica了
- 怎样处理Failed Replica恢复回来的情况：如果这个partition replica之前不在ack列表中，那么启动后重新受Zookeeper管理即可，之后producer发送message的时候，partition leader会继续发送message到这个partition follower上。如果这个partition replica之前在ack列表中，此时重启后，需要把这个partition replica再手动加到ack列表中。（ack列表是手动添加的，出现某个部工作的partition replica的时候自动从ack列表中移除的）

### Delivery Mode

- Kafka producer 发送message不用维护message的offsite信息，因为这个时候，offsite就相当于一个自增id，producer就尽管发送message就好了。而且Kafka与AMQ不同，AMQ大都用在处理业务逻辑上，而Kafka大都是日志，所以Kafka的producer一般都是大批量的batch发送message，向这个topic一次性发送一大批message，load balance到一个partition上，一起插进去，offsite作为自增id自己增加就好。但是Consumer端是需要维护这个partition当前消费到哪个message的offsite信息的，这个offsite信息，high level api是维护在Zookeeper上，low level api是自己的程序维护。（Kafka管理界面上只能显示high level api的consumer部分，因为low level api的partition offsite信息是程序自己维护，Kafka是不知道的，无法在管理界面上展示 ）当使用high level api的时候，先拿message处理，再定时自动commit offsite+1（也可以改成手动）, 并且kakfa处理message是没有锁操作的。因此如果处理message失败，此时还没有commit offsite+1，当consumer thread重启后会重复消费这个message。但是作为高吞吐量高并发的实时处理系统，at least once的情况下，至少一次会被处理到，是可以容忍的。如果无法容忍，就得使用low level api来自己程序维护这个offsite信息，那么想什么时候commit offsite+1就自己搞定了

### Partition leader与follower

- partition也有leader和follower之分。leader是主partition，producer写Kafka的时候先写partition leader，再由partition leader push给其他的partition follower。partition leader与follower的信息受Zookeeper控制，一旦partition leader所在的broker节点宕机，zookeeper会冲其他的broker的partition follower上选择follower变为parition leader

### Topic分配partition和partition replica的算法

- 将Broker（size=n）和待分配的Partition排序
- 将第i个Partition分配到第（i%n）个Broker上
- 将第i个Partition的第j个Replica分配到第（(i + j) % n）个Broker上

- 生产者指定分区
  - 开发人员可以在消息生产者端指定发送的消息将要传送到Topic下的哪一个分区（partition），但前提条件是开发人员清楚这个Topic有多少个分区，否则开发人员就不知道怎么编写代码了
  - 开发人员可以有两种方式进行分区指定：
    - 创建消息对象KeyedMessage时，指定方法中partKey/key的值
    - 重新实现Kafka.producer.Partitioner接口，以便覆盖掉默认实现
  - 使用KeyedMessage类构造消息对象时，可以指定4个参数，他们分别是：topic名称、消息Key、分区Key和message消息内容
    - 发送消息时需要将数据封装为一个ProducerRecord对象，其中的参数可以直接指定partition或消息key，分区规则
      - 指明partition的情况下，直接将指明的值作为partition值
      - 没有指明partition值但是有key值，将key的hash值与topic的partition数进行取余得到partition值
      - 既没有partition值也没有key值，第一次调用时随机生成一个整数（后面每次调用在这个整数上自增），将这个值与topic可用的partition总数取余得到partition值，也就是round-robin算法

### 消息投递可靠性

- 一个消息如何算投递成功，Kafka提供了三种模式
  - 第一种是啥都不管，发送出去就当作成功，这种情况当然不能保证消息成功投递到broker；
  - 第二种是Master-Slave模型，只有当Master和所有Slave都接收到消息时，才算投递成功，这种模型提供了最高的投递可靠性，但是损伤了性能
  - 第三种模型，即只要Master确认收到消息就算投递成功；实际使用时，根据应用特性选择，绝大多数情况下都会中和可靠性和性能选择第三种模型
  - 消息在broker上的可靠性，因为消息会持久化到磁盘上，所以如果正常stop一个broker，其上的数据不会丢失；但是如果不正常stop，可能会使存在页面缓存来不及写入磁盘的消息丢失，这可以通过配置flush页面缓存的周期、阈值缓解，但是同样会频繁的写磁盘会影响性能，又是一个选择题，根据实际情况配置
  -   消息消费的可靠性，Kafka提供的是“At least once”模型，因为消息的读取进度由offset提供，offset可以由消费者自己维护也可以维护在zookeeper里，但是当消息消费后consumer挂掉，offset没有即时写回，就有可能发生重复读的情况，这种情况同样可以通过调整commit offset周期、阈值缓解，甚至消费者自己把消费和commit offset做成一个事务解决，但是如果你的应用不在乎重复消费，那就干脆不要解决，以换取最大的性能
- 强一致性和弱一致性
  - 在Kafka的实现中，强一致性复制是指当Leader Partition收到消息后，将在所有Follower partition完成这条消息的复制后才认为消息处理成功，并向消息生产者返回ack信息
  - 弱一致性复制是指当Leader partition收到消息后，只要Leader Broker自己完成了消息的存储就认为消息处理成立，并向消息生产者返回ack信息（复制过程随后由Broker节点自行完成）
- 当acks设置为0时，生产者端不会等待Server Broker回执任何的ACK确认信息
- 当acks设置为1时，生产者发送消息将等待这个分区的Leader Server Broker 完成它本地的消息记录操作，但不会等待这个分区下其它Follower Server Brokers的操作
- 当acks设置为“all”时，消息生产者发送消息时将会等待目标分区的Leader Server Broker以及所有的Follower Server Brokers全部处理完，才会得到ACK确认信息。这样的处理逻辑下牺牲了一部分性能，但是消息存储可靠性是最高的
- 消息生产者配置中的“request.required.acks”属性来设置消息的复制性要求
- 故障处理
  - LEO：每个副本中的最后一个offset，即最大的offset，用来表示当前最新的数据
  - HW：所有副本中最小的LEO，用来表示当前follower同步到哪里了
  - follower故障
    - follower发生故障后会被踢出ISR，待该follower恢复后，follower会读取本地磁盘记录的上次的HW，并将log文件高于HW的部分截取掉，从HW开始向leader进行同步。等该follower的LEO大于等于该partition的HW，即follower追上leader之后，就可以重新加入ISR
  - leader故障
    - leader发生故障后，会从ISR中选取一个新的leader，之后，为保证多个副本之间的数据一致性，其余的follower会将各自的log文件高于HW的部分截掉，然后从新的leader上同步数据
    - 注意：这只能保证副本之间的数据同步，并不能保证数据之间不丢失数据或者不重复

- 消费者 
  -  Kafka中的Producer和consumer采用的是push-and-pull模式，即Producer只管向broker push消息，consumer只管从broker pull消息，两者对消息的生产和消费是异步的
  - pull模式不足之处是，如果Kafka没有数据，消费者能会陷入空循环中，一直返回空数据。所以Kafka的消费者在消费数据时会传入一个时间参数timeout，如果当前没有数据可以提供，consumer会等待一端时间再返回，防止空循环
  - 分区分配策略
    - RoundRobin 轮询
    - Rang（默认）
  - offset的维护是通过消费者组来维护的，当其中一个消费者挂掉以后，可以通过其他消费者继续消费，其他消费者通过消费者组维护的offset继续消费
  - 自动提交offset
  - 手动提交offset
    - 虽然自动提交offset十分便利，但是由于是基于时间提交的，所以很难把控offset的提交时机，因此Kafka提供了手动提交offset
    - 手动提交分为commitSync（同步提交）和commitAsync（异步提交）；两者相同点是将本次poll的一批数据最高的偏移量提交；不同点是同步提交阻塞当前线程，一直到提交成功为止，并且会自动失败重试；而异步提交没有失败重试机制，故有可能提交失败
    - 无论是同步提交还是异步提交，都有可能导致数据的漏消费或者重复消费。先提交offset后消费，有可能造成数据的漏消费；而先消费后提交offset，有可能导致数据的重复消费
  - 自定义offset

### Kakfa设计思想

- Partition ack
  - 当ack=1，表示producer写partition leader成功后，broker就返回成功，无论其他的partition follower是否写成功。当ack=2，表示producer写partition leader和其他一个follower成功的时候，broker就返回成功，无论其他的partition follower是否写成功。当ack=-1[parition的数量]的时候，表示只有producer全部写成功的时候，才算成功，Kafka broker才返回成功信息。这里需要注意的是，如果ack=1的时候，一旦有个broker宕机导致partition的follower和leader切换，会导致丢数据

- message状态
  - 在Kafka中，消息的状态被保存在consumer中，broker不会关心哪个消息被消费了被谁消费了，只记录一个offset值（指向partition中下一个要被消费的消息位置），这就意味着如果consumer处理不好的话，broker上的一个消息可能会被消费多次

- message持久化
  - Kafka中会把消息持久化到本地文件系统中，并且保持o(1)极高的效率。我们众所周知IO读取是非常耗资源的性能也是最慢的，这就是为了数据库的瓶颈经常在IO上，需要换SSD硬盘的原因。但是Kafka作为吞吐量极高的MQ，却可以非常高效的message持久化到文件。这是因为Kafka是顺序写入o（1）的时间复杂度，速度非常快。也是高吞吐量的原因。由于message的写入持久化是顺序写入的，因此message在被消费的时候也是按顺序被消费的，保证partition的message是顺序消费的。一般的机器,单机每秒100k条数据

- message有效期
  - Kafka会长久保留其中的消息，以便consumer可以多次消费，当然其中很多细节是可配置的

- Produer
  - Producer向Topic发送message，不需要指定partition，直接发送就好了。Kafka通过partition ack来控制是否发送成功并把信息返回给producer，producer可以有任意多的thread，这些Kafka服务器端是不care的。Producer端的delivery guarantee默认是At least once的。也可以设置Producer异步发送实现At most once。Producer可以用主键幂等性实现Exactly once

- Kafka高吞吐量
  - Kafka的高吞吐量体现在读写上，分布式并发的读和写都非常快，写的性能体现在以o(1)的时间复杂度进行顺序写入。读的性能体现在以o(1)的时间复杂度进行顺序读取， 对topic进行partition分区，consume group中的consume线程可以以很高能性能进行顺序读
  - Kafka delivery guarantee(message传送保证)：（1）At most once消息可能会丢，绝对不会重复传输；（2）At least once 消息绝对不会丢，但是可能会重复传输；（3）Exactly once每条信息肯定会被传输一次且仅传输一次，这是用户想要的 

- 批量发送
  - Kafka支持以消息集合为单位进行批量发送，以提高push效率
- push-and-pull
  - Kafka中的Producer和consumer采用的是push-and-pull模式，即Producer只管向broker push消息，consumer只管从broker pull消息，两者对消息的生产和消费是异步的
- push-and-pull
  - Kafka中的Producer和consumer采用的是push-and-pull模式，即Producer只管向broker push消息，consumer只管从broker pull消息，两者对消息的生产和消费是异步的。
- Kafka集群中broker之间的关系
  - 不是主从关系，各个broker在集群中地位一样，我们可以随意的增加或删除任何一个broker节点。
- 负载均衡方面
  -  Kafka提供了一个 metadata API来管理broker之间的负载（对Kafka0.8.x而言，对于0.7.x主要靠zookeeper来实现负载均衡）
- 同步异步
  - Producer采用异步push方式，极大提高Kafka系统的吞吐率（可以通过参数控制是采用同步还是异步方式）
- 分区机制partition
  - Kafka的broker端支持消息分区partition，Producer可以决定把消息发到哪个partition，在一个partition 中message的顺序就是Producer发送消息的顺序，一个topic中可以有多个partition，具体partition的数量是可配置的。partition的概念使得Kafka作为MQ可以横向扩展，吞吐量巨大。partition可以设置replica副本，replica副本存在不同的Kafka broker节点上，第一个partition是leader,其他的是follower，message先写到partition leader上，再由partition leader push到parition follower上。所以说Kafka可以水平扩展，也就是扩展partition
- 离线数据装载
  - Kafka由于对可拓展的数据持久化的支持，它也非常适合向Hadoop或者数据仓库中进行数据装载
- 实时数据与离线数据
  - Kafka既支持离线数据也支持实时数据，因为Kafka的message持久化到文件，并可以设置有效期，因此可以把Kafka作为一个高效的存储来使用，可以作为离线数据供后面的分析。当然作为分布式实时消息系统，大多数情况下还是用于实时的数据处理的，但是当cosumer消费能力下降的时候可以通过message的持久化在淤积数据在Kafka
- 插件支持：
  - 现在不少活跃的社区已经开发出不少插件来拓展Kafka的功能，如用来配合Storm、Hadoop、flume相关的插件
- 解耦:  
  - 相当于一个MQ，使得Producer和Consumer之间异步的操作，系统之间解耦
- 冗余:  
  - replica有多个副本，保证一个broker node宕机后不会影响整个服务
- 扩展性 
  - broker节点可以水平扩展，partition也可以水平增加，partition replica也可以水平增加
- 峰值
  - 在访问量剧增的情况下，Kafka水平扩展, 应用仍然需要继续发挥作用
- 可恢复性
  - 系统的一部分组件失效时，由于有partition的replica副本，不会影响到整个系统
- 顺序保证性
  - 由于Kafka的producer的写message与consumer去读message都是顺序的读写，保证了高效的性能
- 缓冲
  - 由于producer那面可能业务很简单，而后端consumer业务会很复杂并有数据库的操作，因此肯定是producer会比consumer处理速度快，如果没有Kafka，producer直接调用consumer，那么就会造成整个系统的处理速度慢，加一层Kafka作为MQ，可以起到缓冲的作用
- 异步通信：
  - 作为MQ，Producer与Consumer异步通信

### 发送消息的过程

- Kafka的Producer发送消息采用的是异步发送的方式。在消息发送的过程中，涉及到了两个线程-main线程和sender线程，以及一个线程共享变量-RecordAccumulator。main线程将消息发送至RecordAccumulator，Sender线程不断从RecordAccumulator中拉取消息发送到Kafka broker
- 同步发送和异步发送
  - 生产者配置中的“producer.type”属性进行指定。当该属性值为“sync”时，表示使用同步发送的方式；当该属性值为“async”时，表示使用异步发送方式
  - 在异步发送方式下，开发人员调用send方法发送消息时，这个消息并不会立即被发送到topic指定的Leader partition所在的Broker，而是会存储在本地的一个缓冲区域（一定注意是客户端本地）。当缓冲区的状态满足最长等待时间或者最大数据量条数时，消息会以一个设置值批量发送给Broker。缓存区的数据按照batch.num.messages设置的数值被一批一批的发送给目标Broker（默认为200条），如果消息的滞留时间超过了queue.buffering.max.ms设置的值（单位毫秒，默认值为5000）就算没有达到batch.num.messages的数值，消息也会被发送
- 自定义拦截器

### Kafka事务

- Producer事务
  - 为了实现分区跨会话的事务，Kafka引入一个全局唯一的TransactionId，并将Producer获得的PID和TransactionId绑定。这样当Producer重启后就可以通过正在进行的TransactionId获得原来的id
  - 为了管理Transaction，Kafka引入了一个全新的组件Transaction Coordinator。Producer就是通过和Transaction Coordinator交互获得Transaction Id对应的任务状态。Transaction Coordinator还负责将事务所有写入Kafka的一个内部topic，这样即使整个服务重启，由于事务状态获得保存，进行中的事务状态可以得到恢复，从而继续进行
- Consumer事务
  - 消费者的事务保证很弱，无法保证Commit的信息被精确消费，这是因为consumer可以通过offset任意消费，而且不同的segment生命周期不同，用一事务的消息可能会出现重启后被删除的情况

### 面试题

- Kafka的分区器、序列化器、拦截器执行顺序：拦截器 -> 序列化器 -> 分区器
- 消费者提交消费位移时提交的是当前的最新消息的offset还是offset+1？
  - offset+1
- 有哪些情况会造成重复消费？
  - 先处理数据，再提交offset
- 有哪些情况会造成漏消费？
  - 先提交offset，再处理数据
-  topic的分区可以不可以增加或减少？
  - 可增不可减
- Kafka有没有内部的topic？
  - 有，它的作用是：给普通的消费者存offset

- Kafka监控平台：Kafka Eagle
- 实时数据与离线数据：Kafka既支持离线数据也支持实时数据，因为Kafka的message持久化到文件，并可以设置有效期，因此可以把Kafka作为一个高效的存储来使用，可以作为离线数据供后面的分析。当然作为分布式实时消息系统，大多数情况下还是用于实时的数据处理的，但是当cosumer消费能力下降的时候可以通过message的持久化在淤积数据在Kafka
- 插件支持：现在不少活跃的社区已经开发出不少插件来拓展Kafka的功能，如用来配合Storm、Hadoop、flume相关的插件

### Kafka存储原理

- Kafka中发布订阅的对象是topic。我们可以为每类数据创建一个topic，把向topic发布消息的客户端称作producer，从topic订阅消息的客户端称作consumer。Producers和consumers可以同时从多个topic读写数据。一个Kafka集群由一个或多个broker服务器组成，它负责持久化和备份具体的Kafka消息
  - Broker：Kafka节点，一个Kafka节点就是一个broker，多个broker可以组成一个Kafka集群
  - Topic：一类消息，消息存放的目录即主题，例如page view日志、click日志等都可以以topic的形式存在，Kafka集群能够同时负责多个topic的分发
  - Partition：topic物理上的分组，一个topic可以分为多个partition，每个partition是一个有序的队列
  - Segment：partition物理上由多个segment组成，每个Segment存着message信息
  - Producer : 生产message发送到topic
  - Consumer : 订阅topic消费message, consumer作为一个线程来消费
  - Consumer Group：一个Consumer Group包含多个consumer, 这个是预先在配置文件中配置好的。各个consumer（consumer 线程）可以组成一个组（Consumer group ），partition中的每个message只能被组（Consumer group ） 中的一个consumer（consumer 线程 ）消费，如果一个message可以被多个consumer（consumer 线程 ） 消费的话，那么这些consumer必须在不同的组。Kafka不支持一个partition中的message由两个或两个以上的consumer thread来处理，即便是来自不同的consumer group的也不行。它不能像AMQ那样可以多个BET作为consumer去处理message，这是因为多个BET去消费一个Queue中的数据的时候，由于要保证不能多个线程拿同一条message，所以就需要行级别悲观所（for update）,这就导致了consume的性能下降，吞吐量不够。而Kafka为了保证吞吐量，只允许一个consumer线程去访问一个partition。如果觉得效率不高的时候，可以加partition的数量来横向扩展，那么再加新的consumer thread去消费。这样没有锁竞争，充分发挥了横向的扩展性，吞吐量极高。这也就形成了分布式消费的概念

- - -
### Kafka原理概念

- 持久化
  - Kafka使用文件存储消息(append only log),这就直接决定Kafka在性能上严重依赖文件系统的本身特性.且无论任何OS下,对文件系统本身的优化是非常艰难的.文件缓存/直接内存映射等是常用的手段.因为Kafka是对日志文件进行append操作,因此磁盘检索的开支是较小的;同时为了减少磁盘写入的次数,broker会将消息暂时buffer起来,当消息的个数(或尺寸)达到一定阀值时,再flush到磁盘,这样减少了磁盘IO调用的次数.对于Kafka而言,较高性能的磁盘,将会带来更加直接的性能提升
- 性能
  - 除磁盘IO之外,我们还需要考虑网络IO,这直接关系到Kafka的吞吐量问题.Kafka并没有提供太多高超的技巧;对于producer端,可以将消息buffer起来,当消息的条数达到一定阀值时,批量发送给broker;对于consumer端也是一样,批量fetch多条消息.不过消息量的大小可以通过配置文件来指定.对于Kafka broker端,似乎有个sendfile系统调用可以潜在的提升网络IO的性能:将文件的数据映射到系统内存中,socket直接读取相应的内存区域即可,而无需进程再次copy和交换(这里涉及到"磁盘IO数据"/"内核内存"/"进程内存"/"网络缓冲区",多者之间的数据copy)
  - 其实对于producer/consumer/broker三者而言,CPU的开支应该都不大,因此启用消息压缩机制是一个良好的策略;压缩需要消耗少量的CPU资源,不过对于Kafka而言,网络IO更应该需要考虑.可以将任何在网络上传输的消息都经过压缩.Kafka支持gzip/snappy等多种压缩方式
- 负载均衡
  - Kafka集群中的任何一个broker,都可以向producer提供metadata信息,这些metadata中包含"集群中存活的servers列表"/"partitions leader列表"等信息(请参看zookeeper中的节点信息). 当producer获取到metadata信息之后, producer将会和Topic下所有partition leader保持socket连接;消息由producer直接通过socket发送到broker,中间不会经过任何"路由层"
  - 异步发送，将多条消息暂且在客户端buffer起来,并将他们批量发送到broker;小数据IO太多,会拖慢整体的网络延迟,批量延迟发送事实上提升了网络效率;不过这也有一定的隐患,比如当producer失效时,那些尚未发送的消息将会丢失
- Topic模型
  - 其他JMS实现,消息消费的位置是有prodiver保留,以便避免重复发送消息或者将没有消费成功的消息重发等,同时还要控制消息的状态.这就要求JMS broker需要太多额外的工作.在Kafka中,partition中的消息只有一个consumer在消费,且不存在消息状态的控制,也没有复杂的消息确认机制,可见Kafka broker端是相当轻量级的.当消息被consumer接收之后,consumer可以在本地保存最后消息的offset,并间歇性的向zookeeper注册offset.由此可见,consumer客户端也很轻量级
  - Kafka中consumer负责维护消息的消费记录,而broker则不关心这些,这种设计不仅提高了consumer端的灵活性,也适度的减轻了broker端设计的复杂度;这是和众多JMS prodiver的区别.此外,Kafka中消息ACK的设计也和JMS有很大不同,Kafka中的消息是批量(通常以消息的条数或者chunk的尺寸为单位)发送给consumer,当消息消费成功后,向zookeeper提交消息的offset,而不会向broker交付ACK.或许你已经意识到,这种"宽松"的设计,将会有"丢失"消息/"消息重发"的危险
- 消息传输一致
  - Kafka提供3种消息传输一致性语义：最多1次，最少1次，恰好1次
    - 最少1次：可能会重传数据，有可能出现数据被重复处理的情况
    - 最多1次：可能会出现数据丢失情况
    - 恰好1次：并不是指真正只传输1次，只不过有一个机制。确保不会出现“数据被重复处理”和“数据丢失”的情况
  - at most once: 消费者fetch消息,然后保存offset,然后处理消息;当client保存offset之后,但是在消息处理过程中consumer进程失效(crash),导致部分消息未能继续处理.那么此后可能其他consumer会接管,但是因为offset已经提前保存,那么新的consumer将不能fetch到offset之前的消息(尽管它们尚没有被处理),这就是"at most once"
  - at least once: 消费者fetch消息,然后处理消息,然后保存offset.如果消息处理成功之后,但是在保存offset阶段zookeeper异常或者consumer失效,导致保存offset操作未能执行成功,这就导致接下来再次fetch时可能获得上次已经处理过的消息,这就是"at least once"。"Kafka Cluster"到消费者的场景中可以采取以下方案来得到“恰好1次”的一致性语义：最少1次＋消费者的输出中额外增加已处理消息最大编号：由于已处理消息最大编号的存在，不会出现重复处理消息的情况
- 副本
  - Kafka中,replication策略是基于partition,而不是topic;Kafka将每个partition数据复制到多个server上,任何一个partition有一个leader和多个follower(可以没有);备份的个数可以通过broker配置文件来设定。leader处理所有的read-write请求,follower需要和leader保持同步.Follower就像一个"consumer",消费消息并保存在本地日志中;leader负责跟踪所有的follower状态,如果follower"落后"太多或者失效,leader将会把它从replicas同步列表中删除.当所有的follower都将一条消息保存成功,此消息才被认为是"committed",那么此时consumer才能消费它,这种同步策略,就要求follower和leader之间必须具有良好的网络环境.即使只有一个replicas实例存活,仍然可以保证消息的正常发送和接收,只要zookeeper集群存活即可
  - 选择follower时需要兼顾一个问题,就是新leader server上所已经承载的partition leader的个数,如果一个server上有过多的partition leader,意味着此server将承受着更多的IO压力.在选举新leader,需要考虑到"负载均衡",partition leader较少的broker将会更有可能成为新的leader
- log
  - 每个log entry格式为"4个字节的数字N表示消息的长度" + "N个字节的消息内容";每个日志都有一个offset来唯一的标记一条消息,offset的值为8个字节的数字,表示此消息在此partition中所处的起始位置..每个partition在物理存储层面,有多个log file组成(称为segment).segment file的命名为"最小offset".Kafka.例如"00000000000.Kafka";其中"最小offset"表示此segment中起始消息的offset
  - 获取消息时,需要指定offset和最大chunk尺寸,offset用来表示消息的起始位置,chunk size用来表示最大获取消息的总长度(间接的表示消息的条数).根据offset,可以找到此消息所在segment文件,然后根据segment的最小offset取差值,得到它在file中的相对位置,直接读取输出即可
- 分布式
  - Kafka使用zookeeper来存储一些meta信息,并使用了zookeeper watch机制来发现meta信息的变更并作出相应的动作(比如consumer失效,触发负载均衡等)
  - Broker node registry: 当一个Kafka broker启动后,首先会向zookeeper注册自己的节点信息(临时znode),同时当broker和zookeeper断开连接时,此znode也会被删除
  - Broker Topic Registry: 当一个broker启动时,会向zookeeper注册自己持有的topic和partitions信息,仍然是一个临时znode
  - Consumer and Consumer group: 每个consumer客户端被创建时,会向zookeeper注册自己的信息;此作用主要是为了"负载均衡".一个group中的多个consumer可以交错的消费一个topic的所有partitions;简而言之,保证此topic的所有partitions都能被此group所消费,且消费时为了性能考虑,让partition相对均衡的分散到每个consumer上
  - Consumer id Registry: 每个consumer都有一个唯一的ID(host:uuid,可以通过配置文件指定,也可以由系统生成),此id用来标记消费者信息
  - Consumer offset Tracking: 用来跟踪每个consumer目前所消费的partition中最大的offset.此znode为持久节点,可以看出offset跟group_id有关,以表明当group中一个消费者失效,其他consumer可以继续消费
  - Partition Owner registry: 用来标记partition正在被哪个consumer消费.临时znode。此节点表达了"一个partition"只能被group下一个consumer消费,同时当group下某个consumer失效,那么将会触发负载均衡(即:让partitions在多个consumer间均衡消费,接管那些"游离"的partitions)
  - 当consumer启动时,所触发的操作
    - A) 首先进行"Consumer id Registry"
    - B) 然后在"Consumer id Registry"节点下注册一个watch用来监听当前group中其他consumer的"leave"和"join";只要此znode path下节点列表变更,都会触发此group下consumer的负载均衡.(比如一个consumer失效,那么其他consumer接管partitions)
    - C) 在"Broker id registry"节点下,注册一个watch用来监听broker的存活情况;如果broker列表变更,将会触发所有的groups下的consumer重新balance
  - 1) Producer端使用zookeeper用来"发现"broker列表,以及和Topic下每个partition leader建立socket连接并发送消息
  - 2) Broker端使用zookeeper用来注册broker信息,已经监测partition leader存活性
  - 3) Consumer端使用zookeeper用来注册consumer信息,其中包括consumer消费的partition列表等,同时也用来发现broker列表,并和partition leader建立socket连接,并获取消息
- Leader的选择
  - Kafka的核心是日志文件，日志文件在集群中的同步是分布式数据系统最基础的要素
  - 如果leaders永远不会down的话我们就不需要followers了！一旦leader down掉了，需要在followers中选择一个新的leader.但是followers本身有可能延时太久或者crash，所以必须选择高质量的follower作为leader.必须保证，一旦一个消息被提交了，但是leader down掉了，新选出的leader必须可以提供这条消息。大部分的分布式系统采用了多数投票法则选择新的leader,对于多数投票法则，就是根据所有副本节点的状况动态的选择最适合的作为leader.Kafka并不是使用这种方法
  - Kafka动态维护了一个同步状态的副本的集合（a set of in-sync replicas），简称ISR，在这个集合中的节点都是和leader保持高度一致的，任何一条消息必须被这个集合中的每个节点读取并追加到日志中了，才回通知外部这个消息已经被提交了。因此这个集合中的任何一个节点随时都可以被选为leader.ISR在ZooKeeper中维护。ISR中有f+1个节点，就可以允许在f个节点down掉的情况下不会丢失消息并正常提供服。ISR的成员是动态的，如果一个节点被淘汰了，当它重新达到“同步中”的状态时，他可以重新加入ISR.这种leader的选择方式是非常快速的，适合Kafka的应用场景
  - 一个邪恶的想法：如果所有节点都down掉了怎么办？Kafka对于数据不会丢失的保证，是基于至少一个节点是存活的，一旦所有节点都down了，这个就不能保证了
  - 实际应用中，当所有的副本都down掉时，必须及时作出反应。可以有以下两种选择
    - 等待ISR中的任何一个节点恢复并担任leader
    - 选择所有节点中（不只是ISR）第一个恢复的节点作为leader
  - 这是一个在可用性和连续性之间的权衡。如果等待ISR中的节点恢复，一旦ISR中的节点起不起来或者数据都是了，那集群就永远恢复不了了。如果等待ISR意外的节点恢复，这个节点的数据就会被作为线上数据，有可能和真实的数据有所出入，因为有些数据它可能还没同步到。Kafka目前选择了第二种策略，在未来的版本中将使这个策略的选择可配置，可以根据场景灵活的选择
  - 这种窘境不只Kafka会遇到，几乎所有的分布式数据系统都会遇到
- 副本管理
  - 以上仅仅以一个topic一个分区为例子进行了讨论，但实际上一个Kafka将会管理成千上万的topic分区.Kafka尽量的使所有分区均匀的分布到集群所有的节点上而不是集中在某些节点上，另外主从关系也尽量均衡这样每个几点都会担任一定比例的分区的leader
  - 优化leader的选择过程也是很重要的，它决定了系统发生故障时的空窗期有多久。Kafka选择一个节点作为“controller”,当发现有节点down掉的时候它负责在游泳分区的所有节点中选择新的leader,这使得Kafka可以批量的高效的管理所有分区节点的主从关系。如果controller down掉了，活着的节点中的一个会备切换为新的controller
- Leader与副本同步
  - 对于某个分区来说，保存正分区的"broker"为该分区的"leader"，保存备份分区的"broker"为该分区的"follower"。备份分区会完全复制正分区的消息，包括消息的编号等附加属性值。为了保持正分区和备份分区的内容一致，Kafka采取的方案是在保存备份分区的"broker"上开启一个消费者进程进行消费，从而使得正分区的内容与备份分区的内容保持一致。一般情况下，一个分区有一个“正分区”和零到多个“备份分区”。可以配置“正分区+备份分区”的总数量，关于这个配置，不同主题可以有不同的配置值。注意，生产者，消费者只与保存正分区的"leader"进行通信
  - Kafka允许topic的分区拥有若干副本，这个数量是可以配置的，你可以为每个topic配置副本的数量。Kafka会自动在每个副本上备份数据，所以当一个节点down掉时数据依然是可用的
  - Kafka的副本功能不是必须的，你可以配置只有一个副本，这样其实就相当于只有一份数据
  - 创建副本的单位是topic的分区，每个分区都有一个leader和零或多个followers.所有的读写操作都由leader处理，一般分区的数量都比broker的数量多的多，各分区的leader均匀的分布在brokers中。所有的followers都复制leader的日志，日志中的消息和顺序都和leader中的一致。followers向普通的consumer那样从leader那里拉取消息并保存在自己的日志文件中
  - 许多分布式的消息系统自动的处理失败的请求，它们对一个节点是否着（alive）”有着清晰的定义。Kafka判断一个节点是否活着有两个条件
    - 节点必须可以维护和ZooKeeper的连接，Zookeeper通过心跳机制检查每个节点的连接
    - 如果节点是个follower,他必须能及时的同步leader的写操作，延时不能太久
  - 符合以上条件的节点准确的说应该是“同步中的（in sync）”，而不是模糊的说是“活着的”或是“失败的”。Leader会追踪所有“同步中”的节点，一旦一个down掉了，或是卡住了，或是延时太久，leader就会把它移除。至于延时多久算是“太久”，是由参数replica.lag.max.messages决定的，怎样算是卡住了，怎是由参数replica.lag.time.max.ms决定的
  - 只有当消息被所有的副本加入到日志中时，才算是“committed”，只有committed的消息才会发送给consumer，这样就不用担心一旦leader down掉了消息会丢失。Producer也可以选择是否等待消息被提交的通知，这个是由参数acks决定的
  - Kafka保证只要有一个“同步中”的节点，“committed”的消息就不会丢失

### Kafka拓扑结构

-  一个典型的Kafka集群中包含若干Producer（可以是web前端FET，或者是服务器日志等），若干broker（Kafka支持水平扩展，一般broker数量越多，集群吞吐率越高），若干ConsumerGroup，以及一个Zookeeper集群。Kafka通过Zookeeper管理Kafka集群配置：选举Kafka broker的leader，以及在Consumer Group发生变化时进行rebalance，因为consumer消费Kafka topic的partition的offsite信息是存在Zookeeper的。Producer使用push模式将消息发布到broker，Consumer使用pull模式从broker订阅并消费消息
- 分析过程分为以下4个步骤
  - topic中partition存储分布
  - partiton中文件存储方式 (partition在linux服务器上就是一个目录（文件夹）)
  - partiton中segment文件存储结构
  - 在partition中如何通过offset查找message

- topic中partition储存分布
  - Kafka集群会保存所有的消息，不管消息有没有被消费；我们可以设定消息的过期时间，只有过期的数据才会被自动清除以释放磁盘空间。比如我们设置消息过期时间为2天，那么这2天内的所有消息都会被保存到集群中，数据只有超过了两天才会被清除
  - Kafka只维护在Partition中的offset值，因为这个offsite标识着这个partition的message消费到哪条了。Consumer每消费一个消息，offset就会加1。其实消息的状态完全是由Consumer控制的，Consumer可以跟踪和重设这个offset值，这样的话Consumer就可以读取任意位置的消息
  - 把消息日志以Partition的形式存放有多重考虑，第一，方便在集群中扩展，每个Partition可以通过调整以适应它所在的机器，而一个topic又可以有多个Partition组成，因此整个集群就可以适应任意大小的数据了；第二就是可以提高并发，因为可以以Partition为单位读写了
  - 通过上面介绍的我们可以知道，Kafka中的数据是持久化的并且能够容错的。Kafka允许用户为每个topic设置副本数量，副本数量决定了有几个broker来存放写入的数据。如果你的副本数量设置为3，那么一份数据就会被存放在3台不同的机器上，那么就允许有2个机器失败。一般推荐副本数量至少为2，这样就可以保证增减、重启机器时不会影响到数据消费。如果对数据持久化有更高的要求，可以把副本数量设置为3或者更多
  - Kafka中的topic是以partition的形式存放的，每一个topic都可以设置它的partition数量，Partition的数量决定了组成topic的message的数量。Producer在生产数据时，会按照一定规则（这个规则是可以自定义的）把消息发布到topic的各个partition中。上面将的副本都是以partition为单位的，不过只有一个partition的副本会被选举成leader作为读写用
  - 关于如何设置partition值需要考虑的因素。一个partition只能被一个消费者消费（一个消费者可以同时消费多个partition），因此，如果设置的partition的数量小于consumer的数量，就会有消费者消费不到数据。所以，推荐partition的数量一定要大于同时运行的consumer的数量。另外一方面，建议partition的数量大于集群broker的数量，这样leader partition就可以均匀的分布在各个broker中，最终使得集群负载均衡。在Cloudera,每个topic都有上百个partition。需要注意的是，Kafka需要为每个partition分配一些内存来缓存消息数据，如果partition数量越大，就要为Kafka分配更大的heap space
  
- partition中文件存储方式
  - 每个partiton(目录)相当于一个巨型文件被平均分配到多个大小相等segment(段)数据文件中。但每个段segment file消息数量不一定相等，这种特性方便old segment file快速被删除
  - 每个partiton只需要支持顺序读写就行了，segment文件生命周期由服务端配置参数决定
  - 这样做的好处就是能快速删除无用文件，有效提高磁盘利用率
- partiton中segment文件存储结构
  - producer发message到某个topic，message会被均匀的分布到多个partition上（随机或根据用户指定的回调函数进行分布），Kafka broker收到message往对应partition的最后一个segment上添加该消息，当某个segment上的消息条数达到配置值或消息发布时间超过阈值时，segment上的消息会被flush到磁盘，只有flush到磁盘上的消息consumer才能消费，segment达到一定的大小后将不会再往该segment写数据，broker会创建新的segment
  - segment file组成：由2大部分组成，分别为index file和data file，此2个文件一一对应，成对出现，后缀".index"和“.log”分别表示为segment索引文件、数据文件
  - segment文件命名规则：partion全局的第一个segment从0开始，后续每个segment文件名为上一个全局partion的最大offset(偏移message数)。数值最大为64位long大小，19位数字字符长度，没有数字用0填充
  - 每个segment中存储很多条消息，消息id由其逻辑位置决定，即从消息id可直接定位到消息的存储位置，避免id到位置的额外映射
- 在partition中如何通过offset查找message
  - segment index file采取稀疏索引存储方式，它减少索引文件大小，通过mmap可以直接内存操作，稀疏索引为数据文件的每个对应message设置一个元数据指针,它 比稠密索引节省了更多的存储空间，但查找起来需要消耗更多的时间
  - Kafka会记录offset到zk中。但是，zk client api对zk的频繁写入是一个低效的操作。0.8.2 Kafka引入了native offset storage，将offset管理从zk移出，并且可以做到水平扩展。其原理就是利用了Kafka的compacted topic，offset以consumer group,topic与partion的组合作为key直接提交到compacted topic中。同时Kafka又在内存中维护了的三元组来维护最新的offset信息，consumer来取最新offset信息的时候直接内存里拿即可。当然，Kafka允许你快速的checkpoint最新的offset信息到磁盘上

### Partition Replication原则

- Kafka高效文件存储设计特点
  - Kafka把topic中一个parition大文件分成多个小文件段，通过多个小文件段，就容易定期清除或删除已经消费完文件，减少磁盘占用
  - 通过索引信息可以快速定位message和确定response的最大大小
  - 通过index元数据全部映射到memory，可以避免segment file的IO磁盘操作
  - 通过索引文件稀疏存储，可以大幅降低index文件元数据占用空间大小

### Kafka broker特性

- 无状态的Kafka Broker
  - Broker没有副本机制，一旦broker宕机，该broker的消息将都不可用
  - Broker不保存订阅者的状态，由订阅者自己保存
  - 无状态导致消息的删除成为难题（可能删除的消息正在被订阅），Kafka采用基于时间的SLA(服务水平保证)，消息保存一定时间（通常为7天）后会被删除
  - 消息订阅者可以rewind back到任意位置重新进行消费，当订阅者故障时，可以选择最小的offset进行重新读取消费消息

- Message的交付与生命周期
  - 不是严格的JMS， 因此Kafka对消息的重复、丢失、错误以及顺序型没有严格的要求。（这是与AMQ最大的区别）
  -  Kafka提供at-least-once delivery,即当consumer宕机后，有些消息可能会被重复delivery
  - 因每个partition只会被consumer group内的一个consumer消费，故Kafka保证每个partition内的消息会被顺序的订阅
  -  Kafka为每条消息为每条消息计算CRC校验，用于错误检测，crc校验不通过的消息会直接被丢弃掉
- 压缩
  - Kafka支持以集合（batch）为单位发送消息，在此基础上，Kafka还支持对消息集合进行压缩，Producer端可以通过GZIP或Snappy格式对消息集合进行压缩。Producer端进行压缩之后，在Consumer端需进行解压。压缩的好处就是减少传输的数据量，减轻对网络传输的压力，在对大数据处理上，瓶颈往往体现在网络上而不是CPU
  - 那么如何区分消息是压缩的还是未压缩的呢，Kafka在消息头部添加了一个描述压缩属性字节，这个字节的后两位表示消息的压缩采用的编码，如果后两位为0，则表示消息未被压缩
- 消息可靠性
  - 在消息系统中，保证消息在生产和消费过程中的可靠性是十分重要的，在实际消息传递过程中，可能会出现如下三中情况
    - 一个消息发送失败
    - 一个消息被发送多次
    - 最理想的情况：exactly-once ,一个消息发送成功且仅发送了一次
  - 有许多系统声称它们实现了exactly-once，但是它们其实忽略了生产者或消费者在生产和消费过程中有可能失败的情况。比如虽然一个Producer成功发送一个消息，但是消息在发送途中丢失，或者成功发送到broker，也被consumer成功取走，但是这个consumer在处理取过来的消息时失败了
  - 从Producer端看：Kafka是这么处理的，当一个消息被发送后，Producer会等待broker成功接收到消息的反馈（可通过参数控制等待时间），如果消息在途中丢失或是其中一个broker挂掉，Producer会重新发送（我们知道Kafka有备份机制，可以通过参数控制是否等待所有备份节点都收到消息）
  - 从Consumer端看：前面讲到过partition，broker端记录了partition中的一个offset值，这个值指向Consumer下一个即将消费message。当Consumer收到了消息，但却在处理过程中挂掉，此时Consumer可以通过这个offset值重新找到上一个消息再进行处理。Consumer还有权限控制这个offset值，对持久化到broker端的消息做任意处理
- 备份机制
  - 备份机制是Kafka0.8版本的新特性，备份机制的出现大大提高了Kafka集群的可靠性、稳定性。有了备份机制后，Kafka允许集群中的节点挂掉后而不影响整个集群工作。一个备份数量为n的集群允许n-1个节点失败。在所有备份节点中，有一个节点作为lead节点，这个节点保存了其它备份节点列表，并维持各个备份间的状体同步

### Kafka高效性相关设计

- 消息的持久化
  - Kafka高度依赖文件系统来存储和缓存消息(AMQ的nessage是持久化到mysql数据库中的)，因为一般的人认为磁盘是缓慢的，这导致人们对持久化结构具有竞争性持怀疑态度。其实，磁盘的快或者慢，这决定于我们如何使用磁盘。因为磁盘线性写的速度远远大于随机写。线性读写在大多数应用场景下是可以预测的
- 常数时间性能保证
  - 每个Topic的Partition的是一个大文件夹，里面有无数个小文件夹segment，但partition是一个队列，队列中的元素是segment,消费的时候先从第0个segment开始消费，新来message存在最后一个消息队列中。对于segment也是对队列，队列元素是message,有对应的offsite标识是哪个message。消费的时候先从这个segment的第一个message开始消费，新来的message存在segment的最后
  - 消息系统的持久化队列可以构建在对一个文件的读和追加上，就像一般情况下的日志解决方案。它有一个优点，所有的操作都是常数时间，并且读写之间不会相互阻塞。这种设计具有极大的性能优势：最终系统性能和数据大小完全无关，服务器可以充分利用廉价的硬盘来提供高效的消息服务
  - 事实上还有一点，磁盘空间的无限增大而不影响性能这点，意味着我们可以提供一般消息系统无法提供的特性。比如说，消息被消费后不是立马被删除，我们可以将这些消息保留一段相对比较长的时间（比如一个星期）

### Kafka 生产者-消费者

- 消息系统通常都会由生产者，消费者，Broker三大部分组成，生产者会将消息写入到Broker，消费者会从Broker中读取出消息，不同的MQ实现的Broker实现会有所不同，不过Broker的本质都是要负责将消息落地到服务端的存储系统中。具体步骤如下：
  - 生产者客户端应用程序产生消息：
    - 客户端连接对象将消息包装到请求中发送到服务端
    - 服务端的入口也有一个连接对象负责接收请求，并将消息以文件的形式存储起来
    - 服务端返回响应结果给生产者客户端
  - 消费者客户端应用程序消费消息：
    - 客户端连接对象将消费信息也包装到请求中发送给服务端
    - 服务端从文件存储系统中取出消息
    - 服务端返回响应结果给消费者客户端
    - 客户端将响应结果还原成消息并开始处理消息
- Producers
  - Producers直接发送消息到broker上的leader partition，不需要经过任何中介或其他路由转发。为了实现这个特性，Kafka集群中的每个broker都可以响应producer的请求，并返回topic的一些元信息，这些元信息包括哪些机器是存活的，topic的leader partition都在哪，现阶段哪些leader partition是可以直接被访问的
  - Producer客户端自己控制着消息被推送到哪些partition。实现的方式可以是随机分配、实现一类随机负载均衡算法，或者指定一些分区算法。Kafka提供了接口供用户实现自定义的partition，用户可以为每个消息指定一个partitionKey，通过这个key来实现一些hash分区算法。比如，把userid作为partitionkey的话，相同userid的消息将会被推送到同一个partition
  - 以Batch的方式推送数据可以极大的提高处理效率，Kafka Producer 可以将消息在内存中累计到一定数量后作为一个batch发送请求。Batch的数量大小可以通过Producer的参数控制，参数值可以设置为累计的消息的数量（如500条）、累计的时间间隔（如100ms）或者累计的数据大小(64KB)。通过增加batch的大小，可以减少网络请求和磁盘IO的次数，当然具体参数设置需要在效率和时效性方面做一个权衡
  - Producers可以异步的并行的向Kafka发送消息，但是通常producer在发送完消息之后会得到一个future响应，返回的是offset值或者发送过程中遇到的错误。这其中有个非常重要的参数“acks”,这个参数决定了producer要求leader partition 收到确认的副本个数，如果acks设置数量为0，表示producer不会等待broker的响应，所以，producer无法知道消息是否发送成功，这样有可能会导致数据丢失，但同时，acks值为0会得到最大的系统吞吐量
  - 若acks设置为1，表示producer会在leader partition收到消息时得到broker的一个确认，这样会有更好的可靠性，因为客户端会等待直到broker确认收到消息。若设置为-1，producer会在所有备份的partition收到消息时得到broker的确认，这个设置可以得到最高的可靠性保证
  - Kafka 消息有一个定长的header和变长的字节数组组成。因为Kafka消息支持字节数组，也就使得Kafka可以支持任何用户自定义的序列号格式或者其它已有的格式如Apache Avro、protobuf等。Kafka没有限定单个消息的大小，但我们推荐消息大小不要超过1MB,通常一般消息大小都在1~10kB之前
  - 发布消息时，Kafka client先构造一条消息，将消息加入到消息集set中（Kafka支持批量发布，可以往消息集合中添加多条消息，一次行发布），send消息时，producer client需指定消息所属的topic
- Consumers
  - Kafka提供了两套consumer api，分为high-level api和sample-api。Sample-api 是一个底层的API，它维持了一个和单一broker的连接，并且这个API是完全无状态的，每次请求都需要指定offset值，因此，这套API也是最灵活的
  - 在Kafka中，当前读到哪条消息的offset值是由consumer来维护的，因此，consumer可以自己决定如何读取Kafka中的数据。比如，consumer可以通过重设offset值来重新消费已消费过的数据。不管有没有被消费，Kafka会保存数据一段时间，这个时间周期是可配置的，只有到了过期时间，Kafka才会删除这些数据。（这一点与AMQ不一样，AMQ的message一般来说都是持久化到mysql中的，消费完的message会被delete掉）
  - High-level API封装了对集群中一系列broker的访问，可以透明的消费一个topic。它自己维持了已消费消息的状态，即每次消费的都是下一个消息
  - High-level API还支持以组的形式消费topic，如果consumers有同一个组名，那么Kafka就相当于一个队列消息服务，而各个consumer均衡的消费相应partition中的数据。若consumers有不同的组名，那么此时Kafka就相当与一个广播服务，会把topic中的所有消息广播到每个consumer
  - High level api和Low level api是针对consumer而言的，和producer无关
  - High level api是consumer读的partition的offsite是存在zookeeper上。High level api 会启动另外一个线程去每隔一段时间，offsite自动同步到zookeeper上。换句话说，如果使用了High level api， 每个message只能被读一次，一旦读了这条message之后，无论我consumer的处理是否ok。High level api的另外一个线程会自动的把offiste+1同步到zookeeper上。如果consumer读取数据出了问题，offsite也会在zookeeper上同步。因此，如果consumer处理失败了，会继续执行下一条。这往往是不对的行为。因此，Best Practice是一旦consumer处理失败，直接让整个conusmer group抛Exception终止，但是最后读的这一条数据是丢失了，因为在zookeeper里面的offsite已经+1了。等再次启动conusmer group的时候，已经从下一条开始读取处理了
  - Low level api是consumer读的partition的offsite在consumer自己的程序中维护。不会同步到zookeeper上。但是为了Kafka manager能够方便的监控，一般也会手动的同步到zookeeper上。这样的好处是一旦读取某个message的consumer失败了，这条message的offsite我们自己维护，我们不会+1。下次再启动的时候，还会从这个offsite开始读。这样可以做到exactly once对于数据的准确性有保证
- Consumer group
  - 允许consumer group（包含多个consumer，如一个集群同时消费）对一个topic进行消费，不同的consumer group之间独立消费
  - 为了对减小一个consumer group中不同consumer之间的分布式协调开销，指定partition为最小的并行消费单位，即一个group内的consumer只能消费不同的partition
- Consumer与Partition的关系
  - 如果consumer比partition多，是浪费，因为Kafka的设计是在一个partition上是不允许并发的，所以consumer数不要大于partition数
    - 如果consumer比partition少，一个consumer会对应于多个partitions，这里主要合理分配consumer数和partition数，否则会导致partition里面的数据被取的不均匀
    - 如果consumer从多个partition读到数据，不保证数据间的顺序性，Kafka只保证在一个partition上数据是有序的，但多个partition，根据你读的顺序会有不同
    - 增减consumer，broker，partition会导致rebalance，所以rebalance后consumer对应的partition会发生变化
    - High-level接口中获取不到数据的时候是会block的
  - 负载低的情况下可以每个线程消费多个partition。但负载高的情况下，Consumer 线程数最好和Partition数量保持一致。如果还是消费不过来，应该再开 Consumer 进程，进程内线程数同样和分区数一致
  - 消费消息时，Kafka client需指定topic以及partition number（每个partition对应一个逻辑日志流，如topic代表某个产品线，partition代表产品线的日志按天切分的结果），consumer client订阅后，就可迭代读取消息，如果没有消息，consumer client会阻塞直到有新的消息发布。consumer可以累积确认接收到的消息，当其确认了某个offset的消息，意味着之前的消息也都已成功接收到，此时broker会更新zookeeper上地offset registry

### 高效的数据传输

- 发布者每次可发布多条消息（将消息加到一个消息集合中发布）， consumer每次迭代消费一条消息
- 不创建单独的cache，使用系统的page cache。发布者顺序发布，订阅者通常比发布者滞后一点点，直接使用Linux的page cache效果也比较后，同时减少了cache管理及垃圾收集的开销
3.  使用sendfile优化网络传输，减少一次内存拷贝

### Kafka 与 Zookeeper

- Zookeeper 协调控制
  - 管理broker与consumer的动态加入与离开。(Producer不需要管理，随便一台计算机都可以作为Producer向Kakfa Broker发消息)
  - 触发负载均衡，当broker或consumer加入或离开时会触发负载均衡算法，使得一个consumer group内的多个consumer的消费负载平衡。（因为一个comsumer消费一个或多个partition，一个partition只能被一个consumer消费）
  - 维护消费关系及每个partition的消费信息
- Zookeeper上的细节
  - 每个broker启动后会在zookeeper上注册一个临时的broker registry，包含broker的ip地址和端口号，所存储的topics和partitions信息
  - 每个consumer启动后会在zookeeper上注册一个临时的consumer registry：包含consumer所属的consumer group以及订阅的topics
  - 每个consumer group关联一个临时的owner registry和一个持久的offset registry。对于被订阅的每个partition包含一个owner registry，内容为订阅这个partition的consumer id；同时包含一个offset registry，内容为上一次订阅的offset

### Kafka快的原因

- Kafka操作的是序列文件I / O（序列文件的特征是按顺序写，按顺序读），为保证顺序，Kafka强制点对点的按顺序传递消息，这意味着，一个consumer在消息流（或分区）中只有一个位置
- Kafka不保存消息的状态，即消息是否被“消费”。一般的消息系统需要保存消息的状态，并且还需要以随机访问的形式更新消息的状态
- 而Kafka 的做法是保存Consumer在Topic分区中的位置offset，在offset之前的消息是已被“消费”的，在offset之后则为未“消费”的，并且offset是可以任意移动的，这样就消除了大部分的随机IO
- Kafka支持点对点的批量消息传递
- Kafka的消息存储在OS pagecache（页缓存，page cache的大小为一页，通常为4K，在linux读写文件时，它用于缓存文件的逻辑内容，从而加快对磁盘上映像和数据的访问）

### Kafka为什么分区

- Kafka可以将主题划分为多个分区（Partition），会根据分区规则选择把消息存储到哪个分区中，只要如果分区规则设置的合理，那么所有的消息将会被均匀的分布到不同的分区中，这样就实现了负载均衡和水平扩展。另外，多个订阅者可以从一个或者多个分区中同时消费数据，以支撑海量数据处理能力
- 若没有分区，一个topic对应的消息集在分布式集群服务组中，就会分布不均匀，即可能导致某台服务器A记录当前topic的消息集很多，若此topic的消息压力很大的情况下，服务器A就可能导致压力很大，吞吐也容易导致瓶颈。有了分区后，假设一个topic可能分为10个分区，Kafka内部会根据一定的算法把10分区尽可能均匀分布到不同的服务器上，比如：A服务器负责topic的分区1，B服务器负责topic的分区2，在此情况下，Producer发消息时若没指定发送到哪个分区的时候，Kafka就会根据一定算法上个消息可能分区1，下个消息可能在分区2。当然高级API也能自己实现其分发算法

- Kafka在于分布式架构，RabbitMQ基于AMQP协议来实现，RocketMQ的思路来源于Kafka，改成了主从结构，在事务性可靠性方面做了优化。广泛来说，电商、金融等对事务性要求很高的，可以考虑RabbitMQ和RocketMQ，对性能要求高的可考虑Kafka

# 线上问题分析

- dump
  - 线程Dump
  - 内存Dump
  - gc情况
- dump获取及分析工具
  - jstack
  - jstat
  - jmap
  - jhat
  - Arthas
- dump分析死锁
- dump分析内存泄露
- 自己编写各种outofmemory，stackoverflow程序
  - HeapOutOfMemory
  - Young OutOfMemory
  - MethodArea OutOfMemory
  - ConstantPool OutOfMemory
  - DirectMemory OutOfMemory
  - Stack OutOfMemory Stack OverFlow
- Arthas
  - jvm相关
  - class/classloader相关
  - monitor/watch/trace相关
  - options
  - 管道
  - 后台异步任务
- 常见问题解决思路
  - 内存溢出
  - 线程死锁
  - 类加载冲突
  - load飙高
  - CPU利用率飙高
  - 慢SQL
- 使用工具尝试解决以下问题，并写下总结
  - 当一个Java程序响应很慢时如何查找问题
  - 当一个Java程序频繁FullGC时如何解决问题
  - 如何查看垃圾回收日志
  - 当一个Java应用发生OutOfMemory时该如何解决
  - 如何判断是否出现死锁
  - 如何判断是否存在内存泄露
  - 使用Arthas快速排查Spring Boot应用404/401问题
  - 使用Arthas排查线上应用日志打满问题利用Arthas排查Spring Boot应用NoSuchMethodErro

# 操作系统知识

- Linux的常用命令
  - find、grep、ps、cp、move、tar、head、tail、netstat、lsof、tree、wget、curl、ping、ssh、echo、free、top
    进程间通信
- 服务器性能指标
  - load
  - CPU利用率
  - 内存使用情况
  - qps
  - rt
- 进程同步
  - 生产者消费者问题
  - 哲学家就餐问题
  - 读者写者问题
- 缓冲区溢出
- 分段和分页
- 虚拟内存与主存
- 虚拟内存管理
- 换页算法
- 二进制中的原码、反码、补码
  - 二进制的最高位是符号位：0表示正数，1表示负数
  - 正数的原码、反码、补码都一样
  - 负数的反码 =  它的原码符号位不变，其他位取反（0 ->1 ; 1->0 ）
  - 负数的补码 = 它的反码 +1
  - 0的反码、补码都是0
  - 在计算机运算的时候，都是以补码的方式来运算的
  - 负数的反码=绝对值位-128

# 架构篇

## CAP

- C - Consistent，一致性
- A - Availability，可用性
- P - Partition tolerance，分区容忍性
- CAP原理概括：网络分区发生时，一致性和可用性两难全
  - 在网络分区发生时，两个分布式节点之间无法进行通信，我们对一个节点进行的修改操作无法同步到另外一个节点，所以数据的 「一致性」将无法满足，因为两个分布式节点的数据不再保持一致。除非牺牲「可用性」，也就是暂停分布式节点服务，在网络分区发生时，不再提供修改数据的功能，直到网络状况完全恢复正常在继续对外提供服务

## BASE理论

- BASE：全称：Basically Available(基本可用)，Soft state（软状态）,和 Eventually consistent（最终一致性）三个短语的缩写，来自 ebay 的架构师提出
- Base 理论是对 CAP 中一致性和可用性权衡的结果，其来源于对大型互联网分布式实践的总结，是基于 CAP 定理逐步演化而来的。其核心思想是：
  - 既是无法做到强一致性（Strong consistency），但每个应用都可以根据自身的业务特点，采用适当的方式来使系统达到最终一致性（Eventual consistency）
- Basically Available(基本可用)
  - 响应时间上的损失：正常情况下的搜索引擎0.5秒即返回给用户结果，而基本可用看的搜索结果可能要1秒，2秒甚至3秒（超过3秒用户就接受不了了）
  - 功能上的损失：在一个电商网站上，正常情况下，用户可以顺利完成每一笔订单，但是到了促销时间，可能为了应对并发，保护购物系统的稳定性，部分用户会被引导到一个降级页面
- Soft state（软状态）
  - 什么是软状态呢？相对于原子性而言，要求多个节点的数据副本都是一致的，这是一种 “硬状态”
  - 软状态指的是：允许系统中的数据存在中间状态，并认为该状态不影响系统的整体可用性，即允许系统在多个不同节点的数据副本存在数据延时
  - 原子性（硬状态） -> 要求多个节点的数据副本都是一致的,这是一种"硬状态"
  - 软状态（弱状态） -> 允许系统中的数据存在中间状态,并认为该状态不影响系统的整体可用性,即允许系统在多个不同节点的数据副本存在数据延迟
- Eventually consistent（最终一致性）
  - 弱一致性
  - 和强一致性相对
  - 系统并不保证连续进程或者线程的访问都会返回最新的更新过的值。系统在数据写入成功之后，不承诺立即可以读到最新写入的值，也不会具体的承诺多久之后可以读到。但会尽可能保证在某个时间级别（比如秒级别）之后，可以让数据达到一致性状态
  - 系统能够保证在没有其他新的更新操作的情况下，数据最终一定能够达到一致的状态，因此所有客户端对系统的数据访问最终都能够获取到最新的值
  - 对于软状态,我们允许中间状态存在，但不可能一直是中间状态，必须要有个期限，系统保证在没有后续更新的前提下,在这个期限后,系统最终返回上一次更新操作的值,从而达到数据的最终一致性,这个容忍期限（不一致窗口的时间）取决于通信延迟，系统负载，数据复制方案设计，复制副本个数等，DNS是一个典型的最终一致性系统
- 最终一致性模型变种
  - 因果一致性（Causal consistency）
    - 如果节点A在更新完某个数据后通知了节点B,那么节点B的访问修改操作都是基于A更新后的值,同时,和节点A没有因果关系的C的数据访问则没有这样的限制
  - 读己之所写（Read your writes）
    - 因果一致性的特定形式，一个节点A总可以读到自己更新的数据
  - 会话一致性（Session consistency）
    - 访问存储系统同一个有效的会话，系统应保证该进程读己之所写
  - 单调读一致性（Monotonic read consistency）
    - 一个节点从系统中读取一个特定值之后，那么该节点从系统中不会读取到该值以前的任何值
  - 单调写一致性（Monotonic write consistency）
    - 一个系统要能够保证来自同一个节点的写操作被顺序执行（保证写操作串行化）
  - 实践中，往往5个系统进行排列组合，当然，不只是分布式系统使用最终一致性，关系型数据库在某个功能上，也是使用最终一致性的，比如备份，数据库的复制过程是需要时间的，这个复制过程中，业务读取到的值就是旧的。当然，最终还是达成了数据一致性。这也算是一个最终一致性的经典案例
- BASE和ACID的区别与联系
  - ACID是传统数据库常用的设计理念, 追求强一致性模型
  - BASE支持的是大型分布式系统，提出通过牺牲强一致性获得高可用性
  - 总的来说，BASE 理论面向大型高可用可扩展的分布式系统，与ACID这种强一致性模型不同，常常是牺牲强一致性来获得可用性，并允许数据在一段时间是不一致的。虽然两者处于【一致性-可用性】分布图的两级，但两者并不是孤立的，对于分布式系统来说，往往依据业务的不同和使用的系统组件不同，而需要灵活的调整一致性要求，也因此，常常会组合使用ACID和BASE

## 分布式

- 分布式理论
  - 2PC、3PC、CAP、BASE

- 分布式事务
  - 本地事务&分布式事务
  - 可靠消息最终一致性
  - 最大努力通知
  - TCC、SAGA
  - Seta
- 分布式数据库
  -  Mycat、Otter、Hbase

- 分布式文件系统
  - Fastdfs、Hdfs

- 分布式缓存
  - 缓存一致性、缓存命中率、缓存冗余
  - 缓存雪崩、缓存穿透、缓存击穿

- 限流降级
  - 熔断器模式
  - Hystrix、Sentinal

- 分布式算法
  - 拜占庭问题与算法
  - 2PC、3PC
  - 共识算法
  - Paxos 算法与 Raft 算法
  - ZAB算法

## 领域驱动设计

- 实体、值对象
- 聚合、聚合根
- 限界上下文
- DDD如何分层
- 充血模型和贫血模型
- DDD和微服务有什么关系

## 微服务

- SOA
- 康威定律
- ServiceMesh
- sidecar
- Docker & Kubernets
- Spring Boot
- Spring Cloud

## 高并发

- 分库分表
- 横向拆分与水平拆分
- 分库分表后的分布式事务问题
- CDN技术
- 消息队列
- RabbitMQ、RocketMQ、ActiveMQ、Kafka 各个消息队列的对比

## 高可用

- 双机架构
  - 主备复制
  - 主从复制
  - 主主复制
- 异地多活

## 高性能

- 高性能数据库
  - 读写分离
  - 分库分表
- 高性能缓存
  - 缓存穿透
  - 缓存雪崩
  - 缓存热点
- 负载均衡
- PPC、TPC

## 负载均衡

- 负载均衡分类
  - 二层负载均衡
  - 三层负载均衡
  - 四层负载均衡
  - 七层负载均衡
- 负载均衡工具
  - LVS
  - KeepAlived
  - Nginx
  - HAProxy
- 负载均衡算法
  - 静态负载均衡算法：轮询，比率，优先权
  - 动态负载均衡算法: 最少连接数,最快响应速度，观察方法，预测法，动态性能分配，动态服务器补充，服务质量，服务类型，规则模式

## DNS

- DNS原理
- DNS设计

## CDN

- 数据一致性

## 云计算

- IaaS
- SaaS
- PaaS
- 虚拟化技术
- openstack
- Serverlsess



## 搜索引擎

## Solr

## Lucene

- Lucene 是apache软件基金会一个开放源代码的全文检索引擎工具包，是一个全文检索引擎的架构，提供了完整的查询引擎和索引引擎，部分文本分析引擎。它不是一个完整的搜索应用程序，而是为你的应用程序提供索引和搜索功能。lucene 能够为文本类型的数据建立索引，所以你只要能把你要索引的数据格式转化的文本的，Lucene 就能对你的文档进行索引和搜索。比如你要对一些 HTML 文档，PDF 文档进行索引的话你就首先需要把 HTML 文档和 PDF 文档转化成文本格式的，然后将转化后的内容交给 Lucene 进行索引，然后把创建好的索引文件保存到磁盘或者内存中，最后根据用户输入的查询条件在索引文件上进行查询
- Lucene基于倒排索引
- 全文检索大体分两个过程，索引创建 (Indexing) 和搜索索引 (Search) 
- 全文检索的确加快了搜索的速度，但是多了索引的过程，两者加起来不一定比顺序扫描快多少
  - 加上索引的过程，全文检索不一定比顺序扫描快，尤其是在数据量小的时候更是如此。而对一个很大量的数据创建索引也是一个很慢的过程
  - 然而两者还是有区别的，顺序扫描是每次都要扫描，而创建索引的过程仅仅需要一次，以后便是一劳永逸的了，每次搜索，创建索引的过程不必经过，仅仅搜索创建好的索引就可以了。这也是全文搜索相对于顺序扫描的优势之一：一次索引，多次使用
- 索引步骤
  - 获取内容： Lucene本身没有提供获取内容的工具或者组件，内容是要开发者自己提供相应的程序。这一步包括使用网络爬虫或蜘蛛程序来搜索和界定需要索引的内容。当然，数据来源可能包括数据库、分布式文件系统、本地xml等等。lucene作为一款核心搜索库，不提供任何功能来实现内容获取
  - 建立文档：获取原始内容后，需要对这些内容进行索引，必须将这些内容转换成部件（文档）。文档主要包括几个带值的域，比如标题，正文，摘要，作者和链接。如果文档和域比较重要的话，还可以添加权值。设计完方案后，需要将原始内容中的文本提取出来写入各个文档，这一步可以使用文档过滤器，开源项目如Tika，实现很好的文档过滤。如果要获取的原始内容存储于数据库中，有一些项目通过无缝链接内容获取步骤和文档建立步骤就能轻易地对数据库表进行航所以操作和搜索操作，例如DBSight，Hibernate Search，LuSQL，Compass和Oracle/Lucene集成项目
  - 文档分析： 搜索引擎不能直接对文本进行索引：必须将文本分割成一系列被称为语汇单元的独立的原子元素。每一个语汇单元能大致与语言中的“单词”对应起来，这个步骤决定文档中的文本域如何分割成语汇单元系列。lucene提供了大量内嵌的分析器可以轻松控制这步操作
  - 文档索引： 将文档加入到索引列表中。Lucene在这一步骤中提供了强档的API，只需简单调用提供的几个方法就可以实现出文档索引的建立。为了提供好的用户体验，索引是必须要处理好的一环：在设计和定制索引程序时必须围绕如何提高用户的搜索体验来进行
- 构建文档对象
  - 获取原始内容的目的是为了索引，在索引前需要将原始内容创建成文档（Document），文档中包括一个一个的域（Field），域中存储内容
  - 我们可以将磁盘上的一个文件当成一个document，Document中包括一些Field（file_name文件名称、file_path文件路径、file_size文件大小、file_content文件内容）
    - 每个Document可以有多个Field
    - 不同的Document可以有不同的Field
    - 同一个Document可以有相同的Field（域名和域值都相同）
    - 每个文档都有一个唯一的编号，就是文档id
- 分析文档
  - 将原始内容创建为包含域（Field）的文档（document），需要再对域中的内容进行分析，分析的过程是经过对原始文档提取单词、将字母转为小写、去除标点符号、去除停用词等过程生成最终的语汇单元，可以将语汇单元理解为一个一个的单词
  - 比如下边的文档经过分析如下：
    - 原文档内容：
      - Lucene is a Java full-text search engine.  
    - 分析后得到的语汇单元：
      - lucene、java、full、search、engine
  - 每个单词叫做一个Term，不同的域中拆分出来的相同的单词是不同的term。term中包含两部分一部分是文档的域名，另一部分是单词的内容。K V形式
  - 例如：文件名中包含apache和文件内容中包含的apache是不同的term
- 创建索引 
  - 对所有文档分析得出的语汇单元进行索引，索引的目的是为了搜索，最终要实现只搜索被索引的语汇单元从而找到Document（文档）
  - 创建索引是对语汇单元索引，通过词语找文档，这种索引的结构叫倒排索引结构
  - 传统方法是根据文件找到该文件的内容，在文件内容中匹配搜索关键字，这种方法是顺序扫描方法，数据量大、搜索慢
  - 倒排索引结构是根据内容（词语）找文档
  - 倒排索引结构也叫反向索引结构，包括索引和文档两部分，索引即词汇表，它的规模较小，而文档集合较大
  - Field域的属性 
  - 是否分析
    - 是否对域的内容进行分词处理。前提是我们要对域的内容进行查询
  - 会否索引
    - 将Field分析后的词或整个Field值进行索引，只有索引才能搜索到。比如：商品名称、商品简介分析后进行索引，订单号、身份证号不用分析但也要索引，这些将来都要作为查询条件
  - 会否存储
    - 将Field值存储到文档中，存储在文档中的Field才可以从Document中获取，凡是要从Document中获取的Field都要存储。
  - Field的子类
    - StringField(FieldName,FieldValue,Field.Store.YES)
      - 不分析、进行索引，可以通过 Field.Store.YES或Field.Store.NO来决定是否保存到Document中。
    - LongPint(FieldName,FieldValue)
      - 分析、进行索引，索引不存储
    - StoreField(FieldName,FieldValue) 支持多种类型
      - 不分析、不进行索引，会存储到Document。
    - TextField(FieldName,FieldValue,Field.Store.NO)  TextField(FieldName,Reader)  文本或者流
    - 分析、进行索引
- 查询索引
  - 返回的顺序是通过关键字在文档中出现次数倒序排列，即出现次数最多的文档排在前面。
  - IndexSearcher搜索API
  - search(query,n)
    - 根据query搜索，返回评分最高的n条记录
  - search(query,filter,n)
    - 根据query搜索，添加过滤策略，返回评分最高的n条记录
  - search(query,n,sort)
    - 根据query搜索，添加排序策略，返回评分最高的n条记录
  - search(booleanQuery,filter,n,sort)
    - 根据query搜索，添加过滤策略，添加排序策略，返回评分最高的n条记录
  - 搜索使用的分析器要和索引使用的分析器一致

## Elasticsearch

- Elasticsearch是一个基于Lucene构建的开源、分布式、RESTful接口全文检索引擎。Elasticsearch还是一个分布式文档数据库，其中每个字段均是被索引的数据且可被搜索，它能够扩展至数以百计的服务器存储以及处理PB级的数据。它可以在很短的时间内存储、搜索和分析大量的数据
- Elasticsearch就是为高可用和扩展性而生的。可以通过性能更强的服务器来完成，称为垂直扩展或者向上扩展，或增加更多的服务器来完成，称为水平扩展或者向外扩展
- Elasticsearch自身就是分布式的，它直到如何管理多个节点来完成扩展和实现高可用性，即添加了机器以后应用不需要做任何改动
- Elasticsearch术语
  - 索引词（term）
    - 索引词是一个能够被索引的精确值。foo、Foo、FOO几个单词是不同的索引词。索引词（term）是可以通过term查询进行精确的搜索
  - 文本（text）
    - 文本是一段普通的非结构化文字。通常，文本会被分析成一个个的索引词，存储在Elasticsearch的索引库。为了让文本能够进行搜索，文本字段需要事先进行分析；当对文本中的关键词进行查询的时候，搜索引擎应该根据搜索条件搜索出原文本
  - 分析（analysis）
    - 分析是将文本转换为索引词的过程，分析的过程依赖于分词器
  - 集群（cluster）
    - 集群由一个或多个节点组成，堆外提供索引和搜索功能。在所有节点中，一个集群有一个唯一的名称默认为Elasticsearch。此名称很重要，因为每个节点只能是集群的一部分，当该节点被设置为相同的集群名称时，就会自动加入集群
      当需要很多个集群时，要确保每个集群名称不能重复，否则，节点可能会加入错误的集群
      一个节点只能只能加入一个集群。此外还可以拥有多个独立的集群，每个集群都有其不同的集群名称。例如，在开发过程中，你可以建立开发集群库和集群库，分别为开发、测试服务
  - 节点（node）
    - 一个节点是一个逻辑上独立的服务，它是集群的一部分，可以存储数据，并参与集群的索引和搜索功能。就像集群一样，节点也有唯一的名称，在启动的时候分配
    - 在网络中Elasticsearch集群通过节点名称进行管理和通信。一个节点可以被配置加入一个特定的集群。默认情况下，每个节点会加入名为Elasticsearch的集群中，这意味着如果你在网络中启动多个节点，它们能彼此发现并自动加入一个名为Elasticsearch的集群中。在一个集群中，你可以拥有多个你想要的节点。当网络中没有集群运行的时候，只要启动任何一个节点，这个节点会默认生成一个新的集群，这个集群会有一个节点
  - 路由（routing）
    - 当存储一个文档的时候，它会存储在唯一的主分片中，具体哪个分片是通过散列值进行选择。默认情况下，这个值是由文档的ID生成。如果文档有一个指定的父文档，则从父文档ID中生成，该值可以在存储文档的时候进行修改
  - 分片（shard）
    - 分片是单个Lucene实例，这是ElasticSearch管理的比较底层的功能。索引是指向主分片和副本分片的逻辑空间。对于使用，只需要指定分片的数量，其他不需要做过多的事情
    - 在开发过程中，我们对应的对象都是索引，ElasticSearch会自动管理集群中的所有分片，当发生故障时，ElasticSearch会把分片移动到不同的节点或者添加新的节点
    - 一个索引可以存储很大的数据，这些空间可以超过一个节点的物理存储限制。ElasticSearch将索引分解成多个分片。当你创建一个索引，你可以简单定义你想要的分配数量。每个分片本身是一个全功能的、独立的单元，可以托管在集群中的任何节点
  - 主分片（primary shard）
    - 每个文档都存储在一个分片中，当你存储一个文档的时候，系统会首先存储在主分片中，然后会复制到不同的副本中。默认情况下，一个索引有5个分片。可以事先指定分片的数量，当主分片一旦建立，则分片数量不能修改
  - 副本分片（replica shard）
    - 每个分片有零个或多个副本。副本主要是主分片的复制，主要作用：
      - 增加高可用性：当分片失败的时候，可以从副本分片中选择一个作为主分片
      - 提高性能：当查询的时候可以到主分片或者副本分片中进行查询。默认情况下，一个主分片配有一个副本，但副本的数量可以在后面动态的配置增加。副本分片必须部署在不同的节点上，不能部署在和主分片相同的节点上
    - 分片的原因：
      - 允许水平分割扩展数据
      - 允许分配和并行操作（可能在多个节点上）从而提高性能和吞吐量
      - 这部分功能对用户是透明的，用户不需要做任何操作，ElasticSearch会自动处理
  - 复制（replica）
    - 当网络中某个节点出现问题的时候，复制可以对故障进行转移，保证系统的高可用
    - 复制的原因：
      - 它提供了高可用，当节点出故障的时候不受影响。一个复制的分片不会存储在同一个节点中
      - 它允许你扩展搜索量，提高并发量，因为搜索可以在所有副本上并行执行
      - 默认情况下，每个索引分配5个主分片和一个副本分片，这意味着你的集群节点至少要有两个节点，你将拥有5个主分片和5个副分片共计10个分片
      - 每个ElasticSearch分片是一个Lucene的索引。有文档存储限制，你可以在一个单一的Lucene索引中存储的最大值为lucene-5843，integer.max_value -128个文档，可以使用_cat/shards_AP监控分片大小
  - 索引（index）
  - 类型（type）
    - 在索引中，可以定义一个或多个类型，类型是索引的逻辑分区。在一般情况下，一种类型被定义为具有一组公共字段的文档。比如：用户数据类型、博客数据类型、评论数据类型
  - 文档（document）
    - 文档是存储在ElasticSearch中的一个JSON格式的字符串。它就像在关系数据库表的一行
      每个存储在索引中的一个文档都有一个类型和一个ID，每个文档都是一个JSON对象，存储了零个或者多个字段，或者键值对。原始的JSON文档被存储在一个叫作_source的字段中。当搜索文档时默认返回的就是这个字段
  - 映射（mapping）
    - 映射像关系型数据库中的表结构，每一个索引都有一个映射，它定义了索引中的每一个字段类型，以及一个索引范围内的设置。一个映射可以事先被定义，或者在第一次存储文档的时候自动识别
  - 字段（field）
    - 文档中包含零个或多个字段，字段可以是一个简单的值（如字符串、整数、日期），也可以是一个数组或者对象的嵌套结构。字段类似于关系数据库中的列。每一个字段都对应一个字段类型，例如整数、字符串、对象等。字段还可以指定如何分析该字段的值
  - 来源自动（source field）
    - 默认情况下，源文档存储被存储在_source这个字段中，当你查询的时候也是返回这个字段。这允许你可以从搜索结果中访问原始的对象。这个对象返回一个精确的JSON字符串，这个对象不显示索引分析后的其他任何数据
  - 主键（ID）
    - ID是一个文件的唯一标识，如果在存库的时候没有提供ID，系统会自动生成一个ID，文档的index/type/id必须是唯一的
- es 写数据底层原理
  - 先写入内存 buffer，在 buffer 里的时候数据是搜索不到的；同时将数据写入 translog 日志文件。如果 buffer 快满了，或者到一定时间，就会将内存 buffer 数据 refresh 到一个新的 segment file 中，但是此时数据不是直接进入 segment file 磁盘文件，而是先进入 os cache 。这个过程就是 refresh
  - 每隔 1 秒钟，es 将 buffer 中的数据写入一个新的 segment file，每秒钟会产生一个新的磁盘文件 segment file，这个 segment file 中就存储最近 1 秒内 buffer 中写入的数据。但是如果 buffer 里面此时没有数据，那当然不会执行 refresh 操作，如果 buffer 里面有数据，默认 1 秒钟执行一次 refresh 操作，刷入一个新的 segment file 中
  - 操作系统里面，磁盘文件其实都有一个东西，叫做 os cache，即操作系统缓存，就是说数据写入磁盘文件之前，会先进入 os cache，先进入操作系统级别的一个内存缓存中去。只要 buffer中的数据被 refresh 操作刷入 os cache中，这个数据就可以被搜索到了
  - 为什么叫 es 是准实时的？ NRT，全称 near real-time。默认是每隔 1 秒 refresh 一次的，所以 es 是准实时的，因为写入的数据 1 秒之后才能被看到。可以通过 es 的 restful api 或者 java api，手动执行一次 refresh 操作，就是手动将 buffer 中的数据刷入 os cache中，让数据立马就可以被搜索到。只要数据被输入 os cache 中，buffer 就会被清空了，因为不需要保留 buffer 了，数据在 translog 里面已经持久化到磁盘去一份了
  - 重复上面的步骤，新的数据不断进入 buffer 和 translog，不断将 buffer 数据写入一个又一个新的 segment file 中去，每次 refresh 完 buffer 清空，translog 保留。随着这个过程推进，translog 会变得越来越大。当 translog 达到一定长度的时候，就会触发 commit 操作
  - commit 操作发生第一步，就是将 buffer 中现有数据 refresh 到 os cache 中去，清空 buffer。然后，将一个 commit point写入磁盘文件，里面标识着这个 commit point 对应的所有 segment file，同时强行将 os cache 中目前所有的数据都 fsync 到磁盘文件中去。最后清空 现有 translog 日志文件，重启一个 translog，此时 commit 操作完成
  - 这个 commit 操作叫做 flush。默认 30 分钟自动执行一次 flush，但如果 translog 过大，也会触发 flush。flush 操作就对应着 commit 的全过程，我们可以通过 es api，手动执行 flush 操作，手动将 os cache 中的数据 fsync 强刷到磁盘上去
  - translog 日志文件的作用是什么？你执行 commit 操作之前，数据要么是停留在 buffer 中，要么是停留在 os cache 中，无论是 buffer 还是 os cache 都是内存，一旦这台机器死了，内存中的数据就全丢了。所以需要将数据对应的操作写入一个专门的日志文件 translog 中，一旦此时机器宕机，再次重启的时候，es 会自动读取 translog 日志文件中的数据，恢复到内存 buffer 和 os cache 中去
  - translog 其实也是先写入 os cache 的，默认每隔 5 秒刷一次到磁盘中去，所以默认情况下，可能有 5 秒的数据会仅仅停留在 buffer 或者 translog 文件的 os cache 中，如果此时机器挂了，会丢失 5 秒钟的数据。但是这样性能比较好，最多丢 5 秒的数据。也可以将 translog 设置成每次写操作必须是直接 fsync 到磁盘，但是性能会差很多
  - 其实 es 第一是准实时的，数据写入 1 秒后可以搜索到；可能会丢失数据的。有 5 秒的数据，停留在 buffer、translog os cache、segment file os cache 中，而不在磁盘上，此时如果宕机，会导致 5 秒的数据丢失
  - 总结一下，数据先写入内存 buffer，然后每隔 1s，将数据 refresh 到 os cache，到了 os cache 数据就能被搜索到（所以我们才说 es 从写入到能被搜索到，中间有 1s 的延迟）。每隔 5s，将数据写入 translog 文件（这样如果机器宕机，内存数据全没，最多会有 5s 的数据丢失），translog 大到一定程度，或者默认每隔 30mins，会触发 commit 操作，将缓冲区的数据都 flush 到 segment file 磁盘文件中
  - 数据写入 segment file 之后，同时就建立好了倒排索引
- es 删除/更新数据底层原理
  - 如果是删除操作，commit 的时候会生成一个 .del 文件，里面将某个 doc 标识为 deleted 状态，那么搜索的时候根据 .del 文件就知道这个 doc 是否被删除了
  - 如果是更新操作，就是将原来的 doc 标识为 deleted 状态，然后新写入一条数据
  - buffer 每 refresh 一次，就会产生一个 segment file，所以默认情况下是 1 秒钟一个 segment file，这样下来 segment file 会越来越多，此时会定期执行 merge。每次 merge 的时候，会将多个 segment file 合并成一个，同时这里会将标识为 deleted 的 doc 给物理删除掉，然后将新的 segment file 写入磁盘，这里会写一个 commit point，标识所有新的 segment file，然后打开 segment file 供搜索使用，同时删除旧的 segment file

# 运维

## 网络

- VMware 网络
  - 在VMware中，虚拟机的网络连接主要是由VMware创建的虚拟交换机(也叫做虚拟网络)负责实现的，VMware可以根据需要创建多个虚拟网络。在Windows系统的主机上，VMware最多可以创建20个虚拟网络，每个虚拟网络可以连接任意数量的虚拟机网络设备在Linux系统的主机上，VMware最多可以创建255个虚拟网络，但每个虚拟网络仅能连接32个虚拟机网络设备
  - VMware的虚拟网络都是以"VMnet+数字"的形式来命名的，例如 VMnet0、VMnet1、VMnet2……以此类推(在Linux系统的主机上，虚拟网络的名称均采用小写形式，例如 vmnet0 )
  - 当我们安装VMware时，VMware会自动为3种网络连接模式各自创建1个虚拟机网络：VMnet0(桥接模式)、VMnet8(NAT模式)、VMnet1(仅主机模式)。此外，我们也可以根据需要自行创建更多的虚拟网络
- VMware 桥接模式
  - VMware桥接模式，也就是将虚拟机的虚拟网络适配器与主机的物理网络适配器进行交接，虚拟机中的虚拟网络适配器可通过主机中的物理网络适配器直接访问到外部网络(例如图中所示的局域网和Internet，下同)。简而言之，这就好像在上图所示的局域网中添加了一台新的、独立的计算机一样。因此，虚拟机也会占用局域网中的一个IP地址，并且可以和其他终端进行相互访问。桥接模式网络连接支持有线和无线主机网络适配器。如果你想把虚拟机当做一台完全独立的计算机看待，并且允许它和其他终端一样的进行网络通信，那么桥接模式通常是虚拟机访问网络的最简单途径
- VMware NAT模式
  - NAT，是Network Address Translation的缩写，意即网络地址转换。NAT模式也是VMware创建虚拟机的默认网络连接模式。使用NAT模式网络连接时，VMware会在主机上建立单独的专用网络，用以在主机和虚拟机之间相互通信。虚拟机向外部网络发送的请求数据"包裹"，都会交由NAT网络适配器加上"特殊标记"并以主机的名义转发出去，外部网络返回的响应数据"包裹"，也是先由主机接收，然后交由NAT网络适配器根据"特殊标记"进行识别并转发给对应的虚拟机，因此，虚拟机在外部网络中不必具有自己的IP地址。从外部网络来看，虚拟机和主机在共享一个IP地址，默认情况下，外部网络终端也无法访问到虚拟机
  - 此外，在一台主机上只允许有一个NAT模式的虚拟网络。因此，同一台主机上的多个采用NAT模式网络连接的虚拟机也是可以相互访问的
  - 前面我们已经提到，默认情况下，外部网络无法访问到虚拟机，不过我们也可以通过手动修改NAT设置实现端口转发功能，将外部网络发送到主机指定端口的数据转发到指定的虚拟机上。比如，我们在虚拟机的80端口上"建立"了一个站点，只要我们设置端口转发，将主机88端口上的数据转发给虚拟机的80端口，就可以让外部网络通过主机的88端口访问到虚拟机80端口上的站点
- VMware 仅主机模式
  - 仅主机模式，是一种比NAT模式更加封闭的的网络连接模式，它将创建完全包含在主机中的专用网络。仅主机模式的虚拟网络适配器仅对主机可见，并在虚拟机和主机系统之间提供网络连接。相对于NAT模式而言，仅主机模式不具备NAT功能，因此在默认情况下，使用仅主机模式网络连接的虚拟机无法连接到Internet(在主机上安装合适的路由或代理软件，或者在Windows系统的主机上使用Internet连接共享功能，仍然可以让虚拟机连接到Internet或其他网络)
  - 在同一台主机上可以创建多个仅主机模式的虚拟网络，如果多个虚拟机处于同一个仅主机模式网络中，那么它们之间是可以相互通信的；如果它们处于不同的仅主机模式网络，则默认情况下无法进行相互通信(可通过在它们之间设置路由器来实现相互通信)

## Linux

## Docker

## Kubernetes

# 数据结构与算法

- 简单数据结构
  - 栈
  - 队列
  - 链表
  - 数组
  - 哈希表
  - 栈和队列的相同和不同之处
  - 栈通常采用的两种存储结构
  - 两个栈实现队列，和两个队列实现栈
- 树
  - 二叉树
  - 字典树
  - 平衡树
  - 排序树
  - B树
  - B+树
  - R树
  - 多路树
  - 红黑树
- 堆
  - 大根堆
  - 小根堆
- 图
  - 有向图
  - 无向图
  - 拓扑
- 稳定的排序算法
  - 冒泡排序
  - 插入排序
  - 鸡尾酒排序
  - 桶排序
  - 计数排序
  - 归并排序
  - 原地归并排序
  - 二叉排序树排序
  - 鸽巢排序
  - 基数排序
  - 侏儒排序
  - 图书馆排序
  - 块排序
- 不稳定的排序算法
  - 选择排序
  - 希尔排序
  - Clover排序算法
  - 梳排序
  - 堆排序
  - 平滑排序
  - 快速排序
  - 内省排序
  - 耐心排序
- 各种排序算法和时间复杂度
  - 常用的时间复杂度所耗费的时间从小到大依次是：O(1) < O(logn) < (n) < O(nlogn) < O(n^2) < O(n^3) < O(2^n) < O(n!) < O(n^n)
  - O(logn) 
    - int i=1,n=100;while(i<n) {i=i*2 }*
    - 由于每次i*2之后，就距离n更近一步，假设有x个2相乘后大于或等于n，则会退出循环
    - 于是由2^x=n得到x=log(2)n，x等于log以2为底n的对数，所以这个循环的时间复杂度为O(logn)
  - 算法的最坏情况与平均情况
    - 假如我们查找一个有n个随机数字数组中的某个数字，最好的情况是第一个数字就是它，那么算法的时间复杂度为O(1)，但是也有可能这个数字在最后一个数字上，那么时间复杂度为O(n)
    - 平均运行时间是期望的运行时间
    - 最坏运行时间是一种保证。在应用中，这是一种最重要的需求，通常除非特别指定，我们提到的运行时间都是最坏情况的运行时间
  - 当让我们求“复杂度”时，通常指的是时间复杂度
  - 如何快速查找到未知长度的单链表中间的元素呢？
    - 利用快慢指针原理：设置两个指针search，mid都指向单链表的头结点，其中search的移动速度是mid的2倍，当search指向尾节点的时候，mid正好就在中间了。这也是标尺的思想
  - 约瑟夫问题：每报数到第三个就自杀，然后继续，直到最后，只会剩下两人
  - 循环链表：尾节点指向头结点
  - 判断单链表中是否有环：链表的尾节点指向了链表中的某一个节点
    - 使用p、q两个指针，p总是向前走，但q每次都从头走起，对于每个节点，看p走的步数是否和q一样。如图，当p从6走到3时，用了6步，此时若q从head出发，则只需两步就到3了，因而步数不等，出现矛盾，存在唤
    - 使用快慢指针，使用p、q两个指针，p每次向前一步，q每次向前走两步，若在某个时候p==q，则存在环
- 深度优先和广度优先搜索
- 全排列
- 贪心算法
- KMP算法
- hash算法
- 海量数据处理
  - 分治
  - hash映射
  - 堆排序
  - 双层桶划分
  - Bloom Filter
  - bitmap
  - 数据库索引
  - mapreduce

# 大数据

## Hadoop

## Hive

## Hbase

## 流式计算

- Storm
- Spark
- Flink

## 分布式日志收集

- flume
- Kafka
- logstash

## 数据挖掘

- mahout

# 网络安全知识

- XSS
  - XSS的防御
- CSRF
- 注入攻击
  - SQL注入
  - XML注入
  - CRLF注入
- 文件上传漏洞
- 加密与解密
  - 对称加密
  - 非对称加密
  - 哈希算法
  - 加盐哈希算法
- 加密算法
  - MD5，SHA1、DES、AES、RSA、DSA
- 彩虹表
- DDOS攻击
  - DOS攻击
  - DDOS攻击
  - memcached为什么可以导致DDos攻击
  - 什么是反射型DDoS
  - 如何通过Hash碰撞进行DOS攻击
- SSL、TLS，HTTPS
- 脱库、洗库、撞库

# 区块链

- 哈希算法
- Merkle树
- 公钥密码算法
- 共识算法
- Raft协议
- Paxos 算法与 Raft 算法
- 拜占庭问题与算法
- 消息认证码与数字签名
- 比特币
  - 挖矿
  - 共识机制
  - 闪电网络
  - 侧链
  - 热点问题
  - 分叉
- 以太坊
- 超级账本

# 人工智能

- 数学基础
- 机器学习
- 人工神经网络
- 深度学习
- 应用场景
- 常用框架
  - TensorFlow
  - DeepLearning4J

# LOT

# 其他语言

- Groovy
- Kotlin
- Python
- Go
- NodeJs
- Swift
- Rust

  