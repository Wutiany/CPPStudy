### 一、数据结构

#### 1、string

set、get、mset、mget、incr key +1、decr key -1

**String** 类型是 **Redis** 中最常使用的类型，内部的实现是通过 **SDS**（Simple Dynamic String ）来存储的。SDS 类似于 **Java** 中的 **ArrayList**，可以通过预分配冗余空间的方式来减少内存的频繁分配。

- **缓存功能：String**字符串是最常用的数据类型，不仅仅是**Redis**，各个语言都是最基本类型，因此，利用**Redis**作为缓存，配合其它数据库作为存储层，利用**Redis**支持高并发的特点，可以大大加快系统的读写速度、以及降低后端数据库的压力。
- **计数器：**许多系统都会使用**Redis**作为系统的实时计数器，可以快速实现计数和查询的功能。而且最终的数据结果可以按照特定的时间落地到数据库或者其它存储介质当中进行永久保存。
- **共享用户Session：**用户重新刷新一次界面，可能需要访问一下数据进行重新登录，或者访问页面缓存**Cookie**，但是可以利用**Redis**将用户的**Session**集中管理，在这种模式只需要保证**Redis**的高可用，每次用户**Session**的更新和获取都可以快速完成。大大提高效率

#### 2、**Hash：**

hget、hset、hmget、hmset、

这个是类似 **Map** 的一种结构，这个一般就是可以将结构化的数据，比如一个对象（前提是**这个对象没嵌套其他的对象**）给缓存在 **Redis** 里，然后每次读写缓存的时候，可以就操作 **Hash** 里的**某个字段**。

但是这个的场景其实还是多少单一了一些，因为现在很多对象都是比较复杂的，比如你的商品对象可能里面就包含了很多属性，其中也有对象。我自己使用的场景用得不是那么多。

#### 3、List

lpop、lpush、rpop、rpush、blpop、brpop（阻塞等待）

**List** 是有序列表

比如可以通过 **List** 存储一些列表型的数据结构，类似粉丝列表、文章的评论列表之类的东西。

比如可以通过 **lrange** 命令，读取某个闭区间内的元素，可以基于 **List** 实现分页查询，这个是很棒的一个功能，基于 **Redis** 实现简单的高性能分页，可以做类似微博那种下拉不断分页的东西，性能高，就一页一页走。

比如可以搞个简单的消息队列，从 **List** 头怼进去，从 **List** 屁股那里弄出来。

**List**本身就是我们在开发过程中比较常用的数据结构了，热点数据更不用说了。

- **消息队列：Redis**的链表结构，可以轻松实现阻塞队列，可以使用左进右出的命令组成来完成队列的设计。比如：数据的生产者可以通过**Lpush**命令从左边插入数据，多个数据消费者，可以使用**BRpop**命令阻塞的“抢”列表尾部的数据。
- 文章列表或者数据分页展示的应用。

比如，我们常用的博客网站的文章列表，当用户量越来越多时，而且每一个用户都有自己的文章列表，而且当文章多时，都需要分页展示，这时可以考虑使用**Redis**的列表，列表不但有序同时还支持按照范围内获取元素，可以完美解决分页查询功能。大大提高查询效率



#### 4、**Set**

sadd、srem、sinter（交集）sunion（并集	）

**Set** 是无序集合，会自动去重的那种。

直接基于 **Set** 将系统里需要去重的数据扔进去，自动就给去重了，如果你需要对一些数据进行快速的全局去重，你当然也可以基于 **JVM** 内存里的 **HashSet** 进行去重，但是如果你的某个系统部署在多台机器上呢？得基于**Redis**进行全局的 **Set** 去重。

可以基于 **Set** 玩儿交集、并集、差集的操作，比如交集吧，我们可以把两个人的好友列表整一个交集，看看俩人的共同好友是谁？对吧。

反正这些场景比较多，因为对比很快，操作也简单，两个查询一个**Set**搞定。



#### 5、**Sorted Set**

zadd、zrem、zincrby、zrevrangebyscore key max min 返回有序集中指定分数区间内的成员，分数从高到低排序

zrevrank key member 返回有序集合中指定成员的排名，有序集成员按分数值递减(从大到小)排序

**Sorted set** 是排序的 **Set**，去重但可以排序，写进去的时候给一个分数，自动根据分数排序。

有序集合的使用场景与集合类似，但是set集合不是自动有序的，而**Sorted set**可以利用分数进行成员间的排序，而且是插入时就排序好。所以当你需要一个有序且不重复的集合列表时，就可以选择**Sorted set**数据结构作为选择方案。

- 排行榜：有序集合经典使用场景。例如视频网站需要对用户上传的视频做排行榜，榜单维护可能是多方面：按照时间、按照播放量、按照获得的赞数等。
- 用**Sorted Sets**来做带权重的队列，比如普通消息的score为1，重要消息的score为2，然后工作线程可以选择按score的倒序来获取工作任务。让重要的任务优先执行。

微博热搜榜，就是有个后面的热度值，前面就是名称



#### 6、Hyperloglog

pfadd key element 、 pfcount key 、 pfmerge destkey sourcekey[sourcekey...]

用来做基数统计的算法，HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定 的、并且是很小的。

在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64 个不同元素的基 数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。

HyperLogLog 主要的**应用场景**就是进行基数统计。这个问题的应用场景其实是十分广泛的。例如：对于 Google 主页面而言，同一个账户可能会访问 Google 主页面多次。于是，在诸多的访问流水中，如何计算出 Google 主页面每天被多少个不同的账户访问过就是一个重要的问题。那么对于 Google 这种访问量巨大的网页而言，其实统计出有十亿 的访问量或者十亿零十万的访问量其实是没有太多的区别的，因此，在这种业务场景下，为了节省成本，其实可以只计算出一个大概的值，而没有必要计算出精准的值。

存在一定误差，占用内存少，稳定占用 12k 左右内存，可以统计 2^64 个元素，对于上面举例的应用场景，建议使用。



#### 7、Redis GEO

Redis GEO 主要用于存储地理位置信息，并对存储的信息进行操作

- geoadd：添加地理位置的坐标。
- geopos：获取地理位置的坐标。
- geodist：计算两个位置之间的距离。
- georadius：根据用户给定的经纬度坐标来获取指定范围内的地理位置集合。
- georadiusbymember：根据储存在位置集合里面的某个地点获取指定范围内的地理位置集合。
- geohash：返回一个或多个位置对象的 geohash 值。



#### 8、Redis Stream

Redis Stream 主要用于消息队列（MQ，Message Queue），Redis 本身是有一个 Redis 发布订阅 (pub/sub) 来实现消息队列的功能，但它有个缺点就是消息无法持久化，如果出现网络断开、Redis 宕机等，消息就会被丢弃。

而 Redis Stream 提供了消息的持久化和主备复制功能，可以让任何客户端访问任何时刻的数据，并且能记住每一个客户端的访问位置，还能保证消息不丢失。

Redis Stream 的结构如下所示，它有一个消息链表，将所有加入的消息都串起来，每个消息都有一个唯一的 ID 和对应的内容：



### 二、redis事务

Redis 事务可以一次执行多个命令， 并且带有以下三个重要的保证：

- 批量操作在发送 EXEC 命令前被放入队列缓存。
- 收到 EXEC 命令后进入事务执行，事务中任意命令执行失败，其余的命令依然被执行。
- 在事务执行过程，其他客户端提交的命令请求不会插入到事务执行命令序列中

单个 Redis 命令的执行是原子性的，但 Redis 没有在事务上增加任何维持原子性的机制，所以 Redis 事务的执行并不是原子性的。

事务可以理解为一个打包的批量执行脚本，但批量指令并非原子化的操作，中间某条指令的失败不会导致前面已做指令的回滚，也不会造成后续的指令不做。



### 三、持久化

**RDB** 把整个 Redis 的数据保存在单一文件中，比较适合用来做灾备，但缺点是快照保存完成之前如果宕机，这段时间的数据将会丢失，另外保存快照时可能导致服务短时间不可用。

save（同步）
客户端向redis发布save命令，创建一个快照文件。

bgsave（异步）
客户端向redis发布bgsave命令，redis会fork一个子进程负责把快照文件写入到磁盘，父进程继续处理命令请求。

**AOF** 对日志文件的写入操作使用的追加模式，将redis每次执行的命令存到单独的日志文件中，当重启redis时会从持久化的日志中恢复数据。，支持每秒同步、每次修改同步和不同步，缺点就是相同规模的数据集，AOF 要大于 RDB，AOF 在运行效率上往往会慢于 RDB。

always
每条redis命令都写入磁盘命令，不丢数据，但io消耗大

everysec
每秒执行一次同步，将多个命令写入磁盘中，每秒一次，但丢一秒数据

no
由操作系统决定什么时候同步到磁盘中，方便，但不可控



可以一起使用！

redis 支持同时开启开启两种持久化方式，我们可以综合使用 AOF 和 RDB 两种持久化机制，用 AOF 来保证数据不丢失，作为数据恢复的第一选择; 用 RDB 来做不同程度的冷备，在 AOF 文件都丢失或损坏不可用的时候，还可以使用 RDB 来进行快速的数据恢复



### 四、高可用

#### 1、主从同步

（1）RDB全量同步

（2）AOF增量同步（同步建立连接过后新产生的这一批数据）

 （3）连接建立之后的命令传播阶段，由master采用策略往各个从机传送AOF，丛机拿到之后一般还要重写AOF之后再执行。这样一个完整的流程才可以保证数据的一致性。

优点：

（1）主从模式可以提高 Redis 的整体运行速度，因为使用主从模式就可以实现数据的读写分离，把写操作的请求分发到主节点上，把其他的读操作请求分发到从节点上，这样就减轻了 Redis 主节点的运行压力，并且提高了 Redis 的整体运行速度。

（2）不但如此使用主从模式还实现了 Redis 的高可用，当主服务器宕机之后，可以很迅速的把从节点提升为主节点，为 Redis 服务器的宕机恢复节省了宝贵的时间。

（3）并且主从复制还降低了数据丢失的风险，因为数据是完整拷贝在多台服务器上的，当一个服务器磁盘坏掉之后，可以从其他服务器拿到完整的备份数据

缺点：

就是当 Redis 的主节点宕机之后，必须人工介入手动恢复。

#### 2、哨兵模式

Redis 哨兵提供了自动容灾修复的功能.Redis 哨兵模块存储在 Redis 的 src/redis-sentinel 目录. 我们可以使用命令`./src/redis-sentinel sentinel.conf`来启动哨兵功能。

哨兵的工作原理是每个哨兵会以每秒钟 1 次的频率，向已知的主服务器和从服务器，发送一个 PING 命令。如果最后一次有效回复 PING 命令的时间，超过了配置的最大下线时间（Down-After-Milliseconds）时，默认是 30s，那么这个实例会被哨兵标记为主观下线。

如果一个主服务器被标记为主观下线，那么正在监视这个主服务器的所有哨兵节点，要以每秒 1 次的频率确认主服务器是否进入了主观下线的状态。如果有足够数量（quorum 配置值）的哨兵证实该主服务器为主观下线，那么这个主服务器被标记为客观下线。此时所有的哨兵会按照规则（协商）自动选出新的主节点服务器，并自动完成主服务器的自动切换功能，而整个过程都是无须人工干预的

功能：

集群监控：负责监控 Redis master 和 slave 进程是否正常工作。

消息通知：如果某个 **Redis** 实例有故障，那么哨兵负责发送消息作为报警通知给管理员。

故障转移：如果 master node 挂掉了，会自动转移到 slave node 上。

配置中心：如果故障转移发生了，通知 client 客户端新的 master 地址。



#### 3、集群模式

将数据分布在不同的主服务器上，以此来降低系统对单主节点的依赖，并且可以大大提高 Redis 服务的读写性能。Redis 集群除了拥有主从模式 + 哨兵模式的所有功能之外，还提供了多个主从节点的集群功能. Redis 集群可以实现数据分片服务，也就是说在 Redis 集群中有 16384 个槽位用来存储所有的数据，当我们有 N 个主节点时，可以把 16384 个槽位平均分配到 N 台主服务器上。当有键值存储时，Redis 会使用 crc16 算法进行 hash 得到一个整数值，然后用这个整数值对 16384 进行取模来得到具体槽位，再把此键值存储在对应的服务器上，读取操作也是同样的道理，这样我们就实现了数据分片的功能。



#### 4、哈希槽又是如何映射到 Redis 实例上呢?

1、通过对key使用 CRC16 算法，计算出一个 16 bit 的值  ；

2、将 16 bit 的值对 16384 执行取模，得到 0 ～ 16383 的数表示 key 对应的哈希槽。

3、根据该槽信息定位到对应的实例。



#### 5、客户端又怎么确定访问的数据分布在哪个实例上呢？

在 Redis 集群中，节点负责存储数据、记录集群的状态（包括键值到正确节点的映射）。集群节点通过Gossip 协议来传播集群的信息，这样可以：发现新的节点、 发送ping包（用来确保所有节点都在正常工作中）、在特定情况发生时发送集群消息， 并且在需要的时候在从节点中推选出主节点。

由于每个节点都记录了slot的映射关系，这样当客户端连接任何一个节点实例，实例就将哈希槽与实例的映射关系响应下发给客户端，客户端就会将哈希槽与实例映射信息缓存在本地。

当客户端请求时，会计算出键所对应的哈希槽，再通过本地缓存的哈希槽实例映射信息定位到数据所在实例上，再将请求发送给对应的实例。

#### 6、什么是 Redis 重定向机制？

如果集群通过扩容节点或者缩减节点，会重新分配slot对应的关系，当客户端将请求发送到实例上，这个实例没有相应的数据，该 Redis 实例会告诉客户端将请求发送到其他的实例上。由于集群节点不能代理（proxy）请求，所以客户端会接收到重定向错误（redirections errors） -MOVED 和 -ASK， 然后将命令重定向到其他节点。

（1）MOVED 重定向  

（2）ASK 重定向  



#### 7、当某个节点发生故障，对集群有什么影响？

1. 当某个Slave节点故障的时候，集群可用性不受影响；
2. 当Master节点故障的时候，大概有cluster-node-timeout配置的时间内不可用，默认值15s；当某个Master节点下有多个Slave节点的时候会进行投票选举新的Master，使用的是Raft协议；
3. 当Master及其Slave节点都不可用的时候，整个集群都不可用；如果只有Slave节点恢复，集群依然不可用；只有故障的Master节点恢复，集群才恢复可用。



#### 8、发生了故障，Cluster 如何实现故障转移？

1. 从下线的 Master 及节点的 Slave 节点列表选择一个节点成为新主节点。
2. 新主节点会撤销所有对已下线主节点的 slot 指派，并将这些 slots 指派给自己。
3. 新的主节点向集群广播一条 PONG 消息，这条 PONG 消息可以让集群中的其他节点立即知道这个节点已经由从节点变成了主节点，并且这个主节点已经接管了原本由已下线节点负责处理的槽。
4. 新的主节点开始接收处理槽有关的命令请求，故障转移完成。



### 五、缓存穿透

**缓存穿透**。产生这个问题的原因可能是外部的恶意攻击，绕过缓存频繁请求DB。

例如，对用户信息进行了缓存，但恶意攻击者使用不存在的用户id频繁请求接口，导致查询缓存不命中，然后穿透 DB 查询依然不命中。这时会有大量请求穿透缓存访问到 DB。

解决：

（1）对不存在的用户，在缓存中保存一个空对象进行标记，防止相同 ID 再次访问 DB。不过有时这个方法并不能很好解决问题，可能导致缓存中存储大量无用数据

（2）布隆过滤器

#### 1、布隆过滤器

这个就是Redis实现的BloomFilter，BloomFilter非常简单，如下图所示，假设已经有3个元素a、b和c，分别通过3个hash算法h1()、h2()和h2()计算然后对一个bit进行赋值，接下来假设需要判断d是否已经存在，那么也需要使用3个hash算法h1()、h2()和h2()对d进行计算，然后得到3个bit的值，恰好这3个bit的值为1，这就能够说明：d可能存在集合中。再判断e，由于h1(e)算出来的bit之前的值是0，那么说明：e一定不存在集合中

![img](https://pic2.zhimg.com/80/v2-ebfcf147b5003d7b3129a370fc9ba3a1_720w.jpg)

bitmap并不是一种真实的数据结构，它本质上是String数据结构，只不过操作的粒度变成了位，即bit。因为String类型最大长度为512MB，所以bitmap最多可以存储2^32个bit。



### 六、缓存击穿

**缓存击穿**，就是某个热点数据失效时，大量针对这个数据的请求会穿透到数据源。

解决这个问题有如下办法。

1. 可以使用互斥锁更新，保证同一个进程中针对同一个数据不会并发请求到 DB，减小 DB 压力。
2. 使用随机退避方式，失效时随机 sleep 一个很短的时间，再次查询，如果失败再执行更新。
3. 针对多个热点 key 同时失效的问题，可以在缓存时使用固定时间加上一个小的**随机数**，避免大量热点 key 同一时刻失效



### 七、缓存雪崩

**缓存雪崩**，产生的原因是缓存挂掉，这时所有的请求都会穿透到 DB。

解决方法：

1. 使用快速失败的熔断策略，减少 DB 瞬间压力；
2. 使用主从模式和集群模式来尽量保证缓存服务的高可用。

* 设置**过期时间**，引入随机值，避免大量缓存同时过期
* 系统启动时的**数据预热**，提前加载缓存数据
* **分布式锁**保证同一时间只有一个线程加载数据，并更新缓存
* **高可用**和**容灾设计**，使用主从复制或者 Redis Cluster 来实现高可用性
* **限流**和**熔断**，控制请求的并发量，避免请求过载（可以使用工具如 Redis 漏桶算法、令牌桶算法等来实现限流）
* 监控和警报，建立完善的监控系统，实时监控 Redis 的性能指标、缓存命中率、缓存失效率等，及时发现异常情况并触发报警，以便及时处理



### 八、IO多路复用

IO多路复用是一种同步的IO模型。利用IO多路复用模型，可以实现一个线程监视多个文件句柄；一旦某个文件句柄就绪，就能够通知到对应应用程序进行相应的读写操作；没有文件句柄就绪时就会阻塞应用程序，从而释放出CPU资源。

IO可以理解为，在操作系统中，数据在内核态和用户态之间的读、写操作，大部分情况下是指网络IO；

多路大部分情况下是指多个TCP连接，也就是多个Socket 或者多个Channel；

复用是指复用一个或多个线程资源。IO多路复用意思就是说，一个或多个线程处理多个 TCP 连接。尽可能地减少系统开销，无需创建和维护过多的进程/线程。

（1）epoll

在客户端操作服务器时，会创建三种文件描述符，简称FD：分别是writefds（写描述符）、readfds（读描述符）和 exceptfds（异常描述符）。

epoll模型是采用时间通知机制来触发相关的IO操作。它没有FD个数限制，而且从用户态拷贝到内核态只需要一次。它主要通过系统底层的函数来注册、激活FD，从而触发相关的 IO 操作，这样大大提高了性能。主要是通过调用以下三个系统函数：

（1）epoll_create()函数，在系统启动时，会在Linux内核里面申请一个B+树结构的文件系统，然后，返回epoll对象，也是一个FD。

（2）epoll_ctl()函数，每新建一个连接的时候，会同步更新epoll对象中的FD，并且绑定一个 callback回调函数。

（3）epoll_wait()函数，遍历所有的callback集合，并触发对应的 IO 操作

epoll模型最大的优点是将轮询改成了回调，大大提高了CPU执行效率，也不会随FD数量的增加而导致效率下降

### 九、内存淘汰策略

**（1）立即删除**

**redis一直遍历着所有被设置了过期时间的key，来检测是否已经到了过期时间，然后对它进行删除。**

**立即删除能保证内存中数据的最大新鲜度**，因为它能保证键值对在过期后马上被删除，所占用的内存也会被释放，但这样对cpu是不友好的，**因为遍历/删除都会占用cpu的时间，如果刚好碰倒cpu很忙的时候，就会给cpu造成额外的压力，产生极大的性能消耗，同时也会影响读取操作。**

**（2）惰性删除**

数据到达过期时间，不做处理，等下次访问该数据的时候
如果未过期，返回该数据
**如果过期了，就删除这个key，返回不存在**

惰性删除的缺点是，**它对内存极其不友好。**
如果这个key已经过期，而且是一个大key，**如果这个key不被访问，就会一直存在于内存中**，那么它永远也不会被删除。我们甚至可以将这种情况视为内存泄漏-无用的垃圾数据占用了大量的内存，而服务器却不去释放他们，对于非常依赖内存的redis而言，肯定不是一个好消息。

**3）定期删除策略**
定期删除策略是上面两种方式的**折中处理**：
**每隔一段时间执行一次删除过期key操作**，并通过限制删除操作执行时长和频率来减少删除操作对cpu的影响。**定期轮询redis库中的时效性数据，采用随机抽取的策略，抽到过期的key就进行删除。**

特点1：**cpu性能占用设置有峰值，检测频率和自行控制**
特点2：**内存压力不是很大，长期占用内存的冷数据会被持续清理**

缺点：
1）**定期删除执行的时长和频率很难界定**，如果执行地太频繁，或者执行时间太长，这种策略就会变为立即删除策略，消耗cpu，如果执行地太不频繁，又会像惰性删除策略一样，出现浪费内存的情况。
2）**会有一直都没被抽查到的key（漏网之鱼）的存在**。

这时，为了解决这些问题，我们的redis**缓存淘汰策略**就登场了

LRU：最近最少使用

1）**noeviction: 不会驱逐任何key**
2）**allkeys-lru: 对所有最近最少使用的key使用算法进行删除**
3）**volatile-lru: 对所有设置了过期时间最近最少使用的key使用LRU算法进行删除**
4）**allkeys-random: 对所有key随机删除**
5）**volatile-random: 对所有设置了过期时间的key随机删除**
6）**volatile-ttl: 删除马上要过期的key**
7）**allkeys-lfu: 对所有key使用LFU算法进行删除**
8）**volatile-lfu: 对所有设置了过期时间的key使用LFU算法进行删除**

### 十、LRU算法

1. 最开始时，内存空间是空的，因此依次进入A、B、C是没有问题的
2. 当加入D时，就出现了问题，内存空间不够了，因此根据LRU算法，内存空间中A待的时间最为久远，选择A,将其淘汰
3. 当再次引用B时，内存空间中的B又处于活跃状态，而C则变成了内存空间中，近段时间最久未使用的
4. 当再次向内存空间加入E时，这时内存空间又不足了，选择在内存空间中待的最久的C将其淘汰出内存，这时的内存空间存放的对象就是E->B->D

~~~go
package lru

import "container/list"

type LRUCache struct {
	capacity int
	cache    map[int]*list.Element
	list     *list.List
}
type Pair struct {
	key   int
	value int
}

func Constructor(capacity int) LRUCache {
	return LRUCache{
		capacity: capacity,
		list:     list.New(),
		cache:    make(map[int]*list.Element),
	}
}

func (this *LRUCache) Get(key int) int {
	if elem, ok := this.cache[key]; ok {
		this.list.MoveToFront(elem)
		return elem.Value.(Pair).value
	}
	return -1
}

func (this *LRUCache) Put(key int, value int) {
	if elem, ok := this.cache[key]; ok {
		this.list.MoveToFront(elem)
		elem.Value = Pair{key, value}
	} else {
		if this.list.Len() >= this.capacity {
			delete(this.cache,this.list.Back().Value.(Pair).key)
			this.list.Remove(this.list.Back())
		}
		this.list.PushFront(Pair{key, value})
		this.cache[key] = this.list.Front()
	}
}
~~~



### 十一、分布式锁



### 十二、数据一致性

mysync、canel

### 十三、大key问题

1、该key每次都需要整存整取

尝试将对象分拆成几个K.V， 使用multiGet获取值。 拆分旨在降低单次操作的压力，将操作压力平摊到多个Redis实例，降低对单个redis的I/O影响。

2、该对象每次只需存取部分数据

- 类似上一种方案，拆分成几个K.V

- 也可将这个大

  对象存储

  在一个hash，每个field代表一个具体属性    

  - hget、hmget获取部分value
  - hset，hmset来更新部分属性

3、key本身具备强相关性

比如多个K代表一个对象，每个K是对象的一个属性，这种可直接按照特定对象的特征来设置一个新K——Hash结构， 原先的K则作为这个新Hash 的field。