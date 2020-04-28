1. Redis线上两种部署方式：
 - Redis的哨兵模式
 - Redis Cluster集群模式

2. Redis Sentinel是Reids的一种高可用方案
 - 哨兵模式下Redis，通过哨兵选举主服务器，做到自动故障转移
 - 哨兵模式还能实现监控、通知及服务发现等功能
 - Jedis中，通过JedisSentinelPool来处理哨兵模式Redis
3. JedisSentinelPool源码解读
 - 进入JedisSentinelPool类，这个类和JedisPool类类似。JedisSentinelPool继承自JedisPoolAbstract，JedisPool也继承自JedisPoolAbstract；
 - JedisSentinelPool有很多构造方法来构建。先看下其中一个构造方法对initSentinels的调用，通过对哨兵初始化，得到master，然后再初始化Master的Pool；
 - 其实可以看到所有的构造方法都要传入masterName和sentinels集合
 - initSentinels初始化哨兵的方法得到Master，遍历哨兵，通过Jedis获取master的地址和端口，得到的是List，这里的List就两个值，一个host，一个端口，如此得到Master；下面还有个遍历哨兵的循环，就是建立每个每个哨兵对应的MasterListener，便于后面的切换操作；

4. Redis集群模式，数据自动分片(分成16384个Hash Slot，每个Key对应一个Slot)，集群模式下，有一个节点挂掉后，其他节点的Slot还可以继续提供服务。
5. 单节点下可以对多个Key做批量操作，集群模式下就不能这样做批量操作了，因为Key分布在多个节点下，，无法同时对多个节点的Key做批量操作。
6. Redis Cluster中是通过JedisCluster来支持的。目前Jedis只从Master读数据，无法做到读写分离，而Lettuce是可以的。如果一定要Jedis做读写分离，那只能定制了。
7. JedisCluster源码解读

 - 进入JedisCluster类，继承自BinaryJedisCluster类。观察JedisCluster构造方法，可以传入一个节点或者节点集合，只要连接上一个节点，就可以得到集群的信息了；
 - JedisCluster构造函数中，传入一个节点，也会是转换成集合。跟踪构造函数内方法，最终到父类BinaryJedisCluster内的构造函数
 - 进入BinaryJedisCluster构造函数，内部创建了JedisSlotBasedConnectionHandler；
 - 进入JedisSlotBasedConnectionHandler类，看到该类的创建是基于父类的构造方法；
 - 进入父类JedisSlotBasedConnectionHandler，继续跟踪构造方法，就到了该类的父类JedisClusterConnectionHandler中
 - JedisClusterConnectionHandler构造方法中，创建了JedisClusterInfoCache这个缓存类，就是缓存一些配置信息和池；下面就是初始化SlotCache方法initializeSlotsCache，
 - 进入方法initializeSlotsCache，循环节点，通过方法cache.discoverClusterNodesAndSlots(jedis)，来处理集群节点和Slot之间关系；
 - 进入discoverClusterNodesAndSlots方法，这个方法就是做节点和Slot之间的对应关系，方法assignSlotsToNode中就是做对应关系分配；
 - 节点和Slot关系处理完后，回到JedisSlotBasedConnectionHandler类中的getConnection方法，循环集群中JedisPool，如果得到一个可以用的就返回，否则就关闭。
 - JedisSlotBasedConnectionHandler类中还有一个根据当前Key对应的Slot获取连接的方法；