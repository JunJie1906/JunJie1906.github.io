# Redis

### Redis 和 MySQL 缓存如何保证一致性？

一致性分为：

**强一致性**：它要求系统写入什么，读出来的也会是什么，但实现起来对系统的性能影响大

**弱一致性**：系统在写入成功后，不承诺立即可以读到写入的值，也不承诺多久之后数据能够达到一致，但会尽可能地保证到某个时间级别（比如秒级别）后，数据能够达到一致状态

**最终一致性**：最终一致性是弱一致性的一个特例，系统会保证在一定时间内，能够达到一个数据一致的状态。例如我的项目中在秒杀场景下使用 redis 扣减完库存后，发送消息队列扣减库存，这样就能保证最终一致性。

这种最终一致性的实现方式又分为是更新数据库的时候更新缓存还是删除缓存 

**更新缓存和删除缓存的区别**：因为更新缓存时在高并发环境下会导致并发写操作产生的竞态条件。即使使用乐观锁或其他机制来确保一致性，仍然无法完全消除数据不一致的可能性。通过删除缓存，可以确保下一次读取时从数据库中获取最新的数据，并避免潜在的并发写冲突。并且当发生数据库事务回滚时，如果缓存中的数据已经被更新，而事务回滚导致数据库中的数据恢复到之前的状态，那么缓存中的数据将与数据库不一致。通过删除缓存，可以确保在事务回滚后，下一次读取将获取到经过回滚的最新数据。所以选用删除缓存的方式。删除缓存又分为先删除redis在更新数据库还是先更新数据库再删除缓存

**先删除redis，再更新数据库的缺点**：可能导致数据库和redis中的数据都是旧数据（删除redis后，再更新数据时失败了）解决方案 延迟双删 写完数据库后，再删除一次。第二次删除缓存，并非立马就删而是等待时间大于读写缓存的时间即可。

**先更新数据库再删除redis 的缺点**：可能导致数据库和redis的数据不一致（redis删除失败）。出错时使用重试机制异步重新处理或者订阅bilog的方式解决。







### 介绍 redis

Redis 是一种基于内存的数据库，对数据的读写操作都是在内存中完成，因此**读写速度非常快**，常用于**缓存，消息队列、分布式锁等场景**。

Redis 提供了多种数据类型来支持不同的业务场景，比如 String(字符串)、Hash(哈希)、 List (列表)、Set(集合)、Zset(有序集合)、Bitmaps（位图）、HyperLogLog（基数统计）、GEO（地理信息）、Stream（流），并且对数据类型的操作都是**原子性**的，因为执行命令由单线程负责的，不存在并发竞争的问题。

除此之外，Redis 还支持**事务 、持久化、Lua 脚本、多种集群方案（主从复制模式、哨兵模式、切片机群模式）、发布/订阅模式，内存淘汰机制、过期删除机制**等等。



###  为什么用 Redis 作为缓存？

主要是因为 **Redis 具备「高性能」和「高并发」两种特性**。

**1、Redis 具备高性能**

操作 Redis 缓存就是直接操作内存，所以速度相当快。

**2、 Redis 具备高并发**

单台设备的 Redis 的 QPS（Query Per Second，每秒钟处理完请求的次数） 是 MySQL 的 10 倍，Redis 单机的 QPS 能轻松破 10w，而 MySQL 单机的 QPS 很难破 1w。

所以，直接访问 Redis 能够承受的请求是远远大于直接访问 MySQL 的，所以我们可以考虑把数据库中的部分数据转移到缓存中去，这样用户的一部分请求会直接到缓存这里而不用经过数据库。



### Redis 除了做缓存，还能做什么？

分布式锁：通过 Redis 来做分布式锁是一种比较常见的方式。通常情况下，我们都是基于 Redisson 来实现分布式锁。
限流：一般是通过 Redis + Lua 脚本的方式来实现限流。
消息队列：Redis 自带的 list 数据结构可以作为一个简单的队列使用。Redis 5.0 中增加的 stream 类型的数据结构更加适合用来做消息队列
延时队列：Redisson 内置了延时队列（基于 sorted set 实现的）。
分布式 Session ：利用 string 或者 hash 保存 Session 数据，所有的服务器都可以访问。



### 说一下 Redis 和 Memcached 的区别和共同点

共同点：都是基于内存的数据库，一般都用来当做缓存使用。都有过期策略。两者的性能都非常高。
区别：
Redis 支持更丰富的数据类型（支持更复杂的应用场景）。Redis 不仅仅支持简单的 k/v 类型的数据，同时还提供 list，set，zset，hash 等数据结构的存储。
Memcached 只支持最简单的 k/v 数据类型。
Redis 支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用,而 Memcached 把数据全部存在内存之中。
Redis 有灾难恢复机制。 因为可以把缓存中的数据持久化到磁盘上。Redis 在服务器内存使用完之后，可以将不用的数据放到磁盘上。但是，Memcached 在服务器内存使用完之后，就会直接报异常 。
Memcached 是多线程，非阻塞 IO 复用的网络模型；Redis 使用单线程的多路 IO 复用模型。
Redis 支持发布订阅模型、Lua 脚本、事务等功能，而 Memcached 不支持。
emcached 过期数据的删除策略只用了惰性删除，而 Redis 同时使用了惰性删除与定期删除。



### Redis的数据类型以及常见场景 

![img](./MySQL Redis.assets/9fa26a74965efbf0f56b707a03bb9b7f.png)

redis中常用的五种数据结构：string、list、set、zset、hash。

**String**结构底层是 **简单动态字符串（SDS）**，支持扩容，存储字符串。

**list**存储线性有序且可重复的元素，底层数据结构是 **双向链表 和 压缩列表**。目前版本的list由quicklist实现，也就是链表，每个节点由 ziplist 或 listpack 组成

**set**存储无序不可重复的元素，一般用于求交集、差集等，底层数据结构可以是**hash表 或 整数数组**。

**zset**存储的是有序不可重复的元素，zset为每个元素添加了一个score属性作为排序依据，底层数据结构可以是**ziplist 和跳表**。

**hash**类型存储的是键值对，底层数据结构是**ziplist和hash表**。新版本改为listpack和hash



ziplist 组成：

header ： 起始地址、尾元素偏移量（有限）、entry个数

entry：前一个 node 的长度、当前数据类型、数据



与ziplist做对比的话，listpack 牺牲了内存使用率，把删除操作变成设置为空的操作，而不是直接删除，避免了连锁更新的情况。

redis应该是发现极致的内存使用远远不如提高redis的处理性能。由于现在内存比较大，将会是往淡化极致的内存使用率，向更快的方向发力。





后面又支持了四种数据类型：

- BitMap：二值状态统计的场景，比如签到、判断用户登陆状态、连续签到用户总数等；
- HyperLogLog：海量数据基数统计的场景，比如百万级网页 UV 计数等；
- GEO：存储地理位置信息的场景，比如滴滴叫车；
- Stream：消息队列，相比于基于 List 类型实现的消息队列，有这两个特有的特性：自动生成全局唯一消息ID，支持以消费组形式消费数据。但是可能会存在丢失数据的情况。



### Redis的线程模型/单线程架构

Redis 采用的是单线程模型，但是 Redis 的单线程最主要指的是 **执行命令是单线程** 的，而Redis 程序并不是单线程的，Redis 会启动多个后台线程来处理 AOF 刷盘、关闭文件、删除 Key 等耗时的操作，而且因为 Redis 的性能瓶颈有时会出现在网络 I/O 上，所以后来也采用了多个 I/O 线程来处理网络请求。

Redis 之所以快，是因为：

- Redis 的大部分操作**都在内存中完成**，并且采用了高效的数据结构。（*因此 Redis 瓶颈可能是机器的内存或者网络带宽，而并非 CPU，既然 CPU 不是瓶颈，那么自然就采用单线程的解决方案了*）；
- Redis 采用单线程模型可以**避免了多线程之间的竞争**，省去了多线程切换带来的时间和性能上的开销，而且也不会导致死锁问题。
- Redis 采用了 **I/O 多路复用机制**处理大量的客户端 Socket 请求，使其能够处理并发请求。*（IO 多路复用机制是指一个线程处理多个 IO 流，就是我们经常听到的 select/epoll 机制。简单来说，在 Redis 只运行单线程的情况下，该机制允许内核中，同时存在多个监听 Socket 和已连接 Socket。内核会一直监听这些 Socket 上的连接请求或数据请求。一旦有请求到达，就会交给 Redis 线程处理，这就实现了一个 Redis 线程处理多个 IO 流的效果）*



### AOF 

AOF ：Redis 在执行完一条 写操作 命令后，就会把该命令以追加的方式写入到AOF文件里；

Redis 重启时，会读取该文件中的命令，然后逐一执行命令来进行数据恢复。

#### AOF 刷盘策略

Redis 执行完写操作命令后，会将命令追加到缓冲区，然后调用 write() 把缓冲区数据写入 AOF 文件，此时数据在 内核缓冲区 ，等待内核写入硬盘，何时写入硬盘由配置决定：

- **Always**，它的意思是每次写操作命令执行完后，同步将 AOF 日志数据写回硬盘；
- **Everysec**，它的意思是每次写操作命令执行完后，先将命令写入到 AOF 文件的内核缓冲区，然后每隔一秒将缓冲区里的内容写回到硬盘；
- **No**，意味着不由 Redis 控制写回硬盘的时机，转交给操作系统控制写回的时机，也就是每次写操作命令执行完后，先将命令写入到 AOF 文件的内核缓冲区，再由操作系统决定何时将缓冲区内容写回硬盘。

#### AOF 重写

因为随着执行的写操作命令越来越多，AOF 文件的大小会越来越大。 AOF 文件过大就会带来性能问题，比如重启 Redis 后，需要读 AOF 文件的内容以恢复数据，如果文件过大，整个恢复的过程就会很慢。

aof文件：

set (a,1)

set (b, 2)

set (a,2)

```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        int n =nums.length;
        for (int i = 0; i < n; ++i) {
            for (int j = i + 1; j < n; ++j) {
                if (nums[i] + nums[j] == target) {
                    return new int[]{i, j};
                }
            }
        }
        return new int[0];
    }
}
```





所以，Redis 为了避免 AOF 文件越写越大，提供了 **AOF 重写机制**，当 AOF 文件的大小超过所设定的阈值后，Redis 就会启用 AOF 重写机制，来压缩 AOF 文件。

AOF 重写机制是在重写时，读取当前数据库中的所有键值对，然后将每一个键值对用一条命令记录到「新的 AOF 文件」，等到全部记录完后，就将新的 AOF 文件替换掉现有的 AOF 文件。

Redis 的 AOF 重写过程是由后台子进程 bgrewriteaof 来完成的，触发重写机制后，主进程就会创建重写 AOF 的子进程，此时父子进程共享物理内存，重写子进程只会对这个内存进行只读，重写 AOF 子进程会读取数据库里的所有数据，并逐一把内存数据的键值对转换成一条命令，再将命令记录到重写日志（新的 AOF 文件）。

重写过程中，主进程依然可以正常处理命令，为了解决这种数据不一致问题，Redis 设置了一个 **AOF 重写缓冲区**，当 Redis 执行完一个写命令之后，它会**同时将这个写命令写入到 「AOF 缓冲区」和 「AOF 重写缓冲区」**。子进程完成 AOF 重写工作后，主进程将 AOF 重写缓冲区的数据追加到新的 AOF 文件中，然后改名覆盖现有的 AOF 文件。



### RDB

RDB 是将某一时刻的内存数据，以二进制的方式写入磁盘。

因为 AOF 日志记录的是操作命令，不是实际的数据，所以用 AOF 方法做故障恢复时，需要全量把日志都执行一遍，一旦 AOF 日志非常多，会造成 Redis 的恢复操作缓慢。因此在 Redis 恢复数据时， RDB 恢复数据的效率会比 AOF 高些。

Redis 提供了 save 和 bgsave，他们的区别就在于是否阻塞主线程。



RDB 在生成快照时候，主线程依然是可以接收请求的，使用的是 **写时复制技术（Copy-On-Write, COW）**技术，就是主线程执行写操作时，被修改的数据会复制一份副本，然后 bgsave 子进程会把该副本数据写入 RDB 文件，主线程仍然可以直接修改原来的数据。



### 混合持久化

RDB 优点是数据恢复速度快，但是快照的频率不好把握。频率太低，丢失的数据就会比较多，频率太高，就会影响性能。

AOF 优点是丢失数据少，但是数据恢复不快。

为了集成了两者的优点，提出了混合持久化，既保证了 Redis 重启速度，又降低数据丢失风险。

混合持久化是在 AOF 重写过程生效的，先将内存数据以 RDB 方式写入到 AOF 文件，然后主线程处理的操作命令会被记录在重写缓冲区里，然后缓冲区命令以 AOF 方式写入到 AOF 文件，写入完成后用新的 AOF 文件替换旧的的 AOF 文件。也就是说，AOF 文件的**前半部分是 RDB 格式的全量数据，后半部分是 AOF 格式的增量数据**。



### 过期删除策略

Redis 会把该 key 带上过期时间存储到一个**过期字典**中，查询一个 key 时，Redis 会检查该 key 是否存在于过期字典中。

#### 惰性删除策略

不主动删除过期键，每次从数据库访问 key 时，都检测 key 是否过期，如果过期则删除该 key。

惰性删除策略的**优点**：

- 因为每次访问时，才会检查 key 是否过期，所以此策略只会使用很少的系统资源，因此，惰性删除策略对 **CPU 时间最友好**。

惰性删除策略的**缺点**：

- 如果一个 key 已经过期，而这个 key 又仍然保留在数据库中，那么只要这个过期 key 一直没有被访问，它所占用的内存就不会释放，造成了一定的**内存空间浪费**。所以，惰性删除策略对内存不友好。

#### 定期删除策略

每隔一段时间「随机」从数据库中取出一定数量的 key 进行检查，并删除其中的过期key。

1. 从过期字典中随机抽取 20 个 key；
2. 检查这 20 个 key 是否过期，并删除已过期的 key；
3. 如果本轮检查的已过期 key 的数量，超过 5 个（20/4），也就是「已过期 key 的数量」占比大于 25%，则继续重复步骤 1；如果已过期的 key 比例小于 25%，则停止继续删除过期 key，然后等待下一轮再检查。

定期删除策略的**优点**：

- 通过限制删除操作执行的时长和频率，来减少删除操作对 CPU 的影响，同时也能删除一部分过期的数据减少了过期键对空间的无效占用。

定期删除策略的**缺点**：

- 难以确定删除操作执行的时长和频率。如果执行的太频繁，就会对 CPU 不友好；如果执行的太少，那又和惰性删除一样了，过期 key 占用的内存不会及时得到释放。

#### Redis 主从模式中，对过期键会如何处理？

当 Redis 运行在主从模式下时，**从库不会进行过期扫描，从库对过期的处理是被动的**。也就是即使从库中的 key 过期了，如果有客户端访问从库时，依然可以得到 key 对应的值，像未过期的键值对一样返回。

从库的过期键处理依靠主服务器控制，**主库在 key 到期时，会在 AOF 文件里增加一条 del 指令，同步到所有的从库**，从库通过执行这条 del 指令来删除过期的 key。



### 内存淘汰策略

分为「在设置了过期时间的数据中进行淘汰」和「在所有数据范围内进行淘汰」

在设置了过期时间的数据中进行淘汰：

- **volatile-random**：随机淘汰设置了过期时间的任意键值；
- **volatile-ttl**：优先淘汰更早过期的键值。
- **volatile-lru**：淘汰所有设置了过期时间的键值中，最久未使用的键值；
- **volatile-lfu**：淘汰所有设置了过期时间的键值中，最少使用的键值；

在所有数据范围内进行淘汰：

- **allkeys-random**：随机淘汰任意键值;
- **allkeys-lru**：淘汰整个键值中最久未使用的键值；
- **allkeys-lfu**：淘汰整个键值中最少使用的键值。



#### Redis 如何实现LRU

Redis 实现的是一种**近似 LRU 算法**，目的是为了更好的节约内存，它的实现是方式随机采样的方式来淘汰数据，它是随机取 5 个值（此值可配置），然后淘汰最久没有使用的那个。

Redis 对象头中的 lru 字段存储时间戳：

**在 LRU 算法中**，Redis 对象头的 24 bits 的 lru 字段是用来记录 key 的访问时间戳

**在 LFU 算法中**，Redis对象头的 24 bits 的 lru 字段被分成两段来存储，高 16bit 存储 访问时间戳；低 8bit 记录 key 的访问频次。



### 什么是 Redis 事务？

redis 事务提供了一种将多个命令请求打包的功能。然后，再按顺序执行打包的所有命令，并且不会被中途打断。
Redis 可以通过 MULTI，EXEC，DISCARD 和 WATCH 等命令来实现事务功能。
使用MULTI命令来开始一个事务 使用EXEC命令来提交事务
使用WATCH命令监视键，在其他客户端对它们进行修改时取消事务。
使用DISCARD命令来取消事务并清除所有已添加到事务中的命令

## 集群

主从复制：写一定是在主服务器上，然后主服务器同步给从服务器。缺点：当主服务器挂掉的时候，不能自动切换到从服务器上。主从服务器存储数据一样，内存可用性差。
优点：在一定程度上分担主服务器读的压力。
哨兵模式：构建多个哨兵节点监视主从服务器，当主服务器挂掉的时候，自动将对应的从服务器切换成主服务器。优点：实现自动切换，可用性高 。
缺点：主从服务器存储数据一致，内存可用性差。还要额外维护一套哨兵系统，较为麻烦。
集群模式：采用无中心节点的方式实现。多个主服务器相连，一个主服务器可 以有多个从服务器，不同的主服务器存储不同的数据。优点：可用性更高，内存可用性高。

### 主从复制

主服务器可以进行读写操作，当发生写操作时自动将写操作同步给从服务器，而从服务器一般是只读，并接受主服务器同步过来写操作命令，然后执行这条命令。也就是说，所有的数据修改只在主服务器上进行，然后将最新的数据同步给从服务器，这样就使得主从服务器的数据是一致的。

连接过程：

1，启动一个从节点，从节点发送一个PSYNC命令，主节点收到后开起一个后台线程生成一份**RDB**文件发送给从节点，从节点收到后先存入磁盘，再加载进内存（全量同步）。 

2，RDB发送完成后，主机点还会发送在生成RDB期间，缓存中新出现的写命令发送给从节点 。

3，当从节点断开并重新连接时主节点会复制期间丢失的数据给从节点（增量同步）。

1 2 3 

### 哨兵

使用 Redis 主从服务的时候，会有一个问题，就是当 Redis 的主从服务器出现故障宕机时，需要手动进行恢复。为了解决这个问题，Redis 增加了哨兵模式（**Redis Sentinel**），实现了**主从节点故障转移**。它会监测主节点是否存活，如果发现主节点挂了，它就会选举一个从节点切换为主节点，并且把新主节点的相关信息通知给从节点和客户端。

#### Redis 哨兵集群是通过什么方式组建的？

**哨兵节点之间是通过 Redis 的发布者/订阅者机制来相互发现的**。

在主从集群中，主节点上有一个名为 sentinel 的频道，不同哨兵就是通过它来相互发现，实现互相通信的。

哨兵把自己的 IP 地址和端口的信息发布到该频道上，如果其他哨兵订阅了该频道。那么此时，其他哨兵就可以从这个频道直接获取哨兵 A 的 IP 地址和端口号。然后，哨兵 B、C 可以和哨兵 A 建立网络连接。

并且哨兵还可以通过 INFO 命令来获取从节点的信息，进而和从节点建立连接。



#### 如何检测节点是否下线？主观下线与客观下线的区别?

Redis Sentinel它通过监控 Redis实例的状态来实现自动故障转移。Redis Sentinel有以下两种方式：
1.主观下线
如果一个 Sentinel 进程在指定时间内没有收到 Redis 实例的 ping 响应，那么它就会将该 Redis 实例标记为主观下线。这个判断是基于 Sentinel 进程本身的主观认定。
2.客观下线
除了主观下线外，Redis Sentinel 还支持客观下线的检测。当多个 Sentinel 进程都认为某个 Redis 实例已经下线时，该 Redis 实例就会被标记为客观下线。一般来说，当超过一半的 Sentinel 进程在指定时间内都认为某个 Redis 实例已经下线时，该 Redis 实例就会被标记为客观下线。

#### 哨兵选举机制

需要选择一个哨兵节点做为 leader 进行故障转移，所以需要选主。

当一个哨兵节点判断主节点为「客观下线」，这个哨兵节点就是候选者，

候选者会向其他哨兵发送命令，表明希望成为 Leader 来执行主从切换，并让所有其他哨兵对它进行投票。

每个哨兵只有一次投票机会，如果用完后就不能参与投票了，可以投给自己或投给别人，但是只有候选者才能把票投给自己。

每位候选者都会先给自己投一票，然后向其他哨兵发起投票请求；当一个哨兵节点收到投票请求时，如果判断主节点主观下线，就把票投给此节点。

如果一个「候选者」满足：

- 第一，拿到半数以上的赞成票；
- 第二，拿到的票数同时还需要大于等于哨兵配置文件中的 quorum 值。

就能成为 Leader，并向其他节点发送心跳消息，让它们知道新的leader已经产生。



#### 新的主库选择出来后，如何进行故障的转移？

1.主节点失效检测
2.选择新的主节点
3.执行故障转移操作
4.恢复原来的主节点 一旦 Sentinel 检测到原来的主节点已经恢复，它会将原来的主节点重新加入 Redis 集群中，并将其作为新的从节点进行复制。



### 切片集群

当 Redis 缓存数据量大到一台服务器无法缓存时，就需要使用 **Redis 切片集群**（Redis Cluster ）方案。

Redis Cluster 主要是为了解决以下问题：
容量限制：单机 Redis 的容量受限于内存大小，如果需要存储更多的数据，就需要使用更多的 Redis 实例，而这样会导致数据分散在多个实例中，管理和维护难度增加。
性能瓶颈：单机 Redis 的性能在处理大规模数据时会受到限制，例如大量的读写请求会导致单机 Redis 的响应速度变慢。
可用性：单机 Redis 面临单点故障问题，如果 Redis 实例出现故障，就会导致整个 Redis 服务不可用。

Redis Cluster 方案采用哈希槽（Hash Slot），来处理数据和节点之间的映射关系，**一个切片集群共有 16384（2^14） 个哈希槽**，这些哈希槽类似于数据分区，每个键值对都会根据它的 key，被映射到一个哈希槽中，具体执行过程是这样的：

- 根据键值对的 key，按照 CRC16 算法计算一个 16 bit 的值。
- 再用 16bit 值对 16384 取模，得到 0~16383 范围内的模数，每个模数代表一个相应编号的哈希槽。

接下来这些哈希槽会被映射到具体的 Redis 节点上，有两种方案：

- **平均分配：** 在使用 cluster create 命令创建 Redis 集群时，Redis 会自动把所有哈希槽平均分布到集群节点上。比如集群中有 9 个节点，则每个节点上槽的个数为 16384/9 个。
- **手动分配：** 可以使用 cluster meet 命令手动建立节点间的连接，组成集群，再使用 cluster addslots 命令，指定每个节点上的哈希槽个数。



### 脑裂

由于网络问题，集群节点之间失去联系，此时主从数据是不同步的；

这时，哨兵也发现主节点失联了，它就认为主节点挂了，重新选举了个主节点，产生两个主服务。

等网络恢复，旧主节点会降级为从节点，再与新主节点进行同步复制的时候，由于会从节点会清空自己的缓冲区，所以导致之前客户端写入的数据丢失了。



### 如何解决脑裂

当主节点发现从节点下线的总数量超过阈值时，那么禁止主节点进行写数据，直接把错误返回给客户端。

在 Redis 的配置文件中有两个参数我们可以设置：

- min-slaves-to-write ，主节点必须要有至少 n 个从节点连接，如果小于这个数，主节点会禁止写数据。
- min-slaves-max-lag ，主从数据复制和同步的延迟不能超过 m 秒，如果超过，主节点会禁止写数据。

举个例子：

假设我们将 min-slaves-to-write 设置为 1，把 min-slaves-max-lag 设置为 10s，主库因为某些原因卡住了 15s，导致哨兵判断主库客观下线，开始进行主从切换。

同时，因为原主库卡住了 15s，没有一个从库能和原主库在 10s 内进行数据复制，原主库也无法接收客户端请求了。

这样一来，主从切换完成后，也只有新主库能接收请求，不会发生脑裂，也就不会发生数据丢失的问题了。



## 场景

### 缓存穿透、击穿、雪崩的区别

缓存穿透：客户端访问不存在的数据，使得请求直达存储层，导致负载过大，直至宕机。原因可能是业务层误删了缓存和库中的数据，或是有人恶意访问不存在的数据。
解决方式：

1.存储层未命中后，返回空值存入缓存层，客户端再次访问时，缓存层直接返回空值。

2.将数据存入布隆过滤器，访问缓存之前经过滤器拦截，若请求的数据不存在则直接返回空值。



缓存击穿：一份热点数据，它的访问量非常大，在它缓存失效的瞬间，大量请求直达存储层，导致服务崩溃。

解决方案：

1.永不过期：对热点数据不设置过期时间。
2.加互斥锁，当一个线程访问该数据时，另一个线程只能等待，这个线程访问之后，缓存中的数据将被重建，届时其他线程就可以从缓存中取值。



缓存雪崩：大量数据同时过期、或是redis节点故障导致服务不可用，缓存层无法提供服务，所有的请求直达存储层，造成数据库宕机。
解决方案：

1.避免数据同时过期，设置随机过期时间。

2.启用降级和熔断措施。

3.设置热点数据永不过期。

4.采用redis集群，一个宕机，另外的还能用

### Redis如何与数据库保持双写一致性

一致性分为：

**强一致性**：它要求系统写入什么，读出来的也会是什么，但实现起来对系统的性能影响大

**弱一致性**：系统在写入成功后，不承诺立即可以读到写入的值，也不承诺多久之后数据能够达到一致，但会尽可能地保证到某个时间级别（比如秒级别）后，数据能够达到一致状态

**最终一致性**：最终一致性是弱一致性的一个特例，系统会保证在一定时间内，能够达到一个数据一致的状态。例如我的项目中在秒杀场景下使用 redis 扣减完库存后，发送消息队列扣减库存，这样就能保证最终一致性。

这种最终一致性的实现方式又分为是更新数据库的时候更新缓存还是删除缓存 因为更新缓存时在高并发环境下会导致并发写操作产生的竞态条件。即使使用乐观锁或其他机制来确保一致性，仍然无法完全消除数据不一致的可能性。通过删除缓存，可以确保下一次读取时从数据库中获取最新的数据，并避免潜在的并发写冲突。并且当发生数据库事务回滚时，如果缓存中的数据已经被更新，而事务回滚导致数据库中的数据恢复到之前的状态，那么缓存中的数据将与数据库不一致。通过删除缓存，可以确保在事务回滚后，下一次读取将获取到经过回滚的最新数据。所以选用删除缓存的方式。删除缓存又分为先删除redis在更新数据库还是先更新数据库再删除缓存

先删除redis，再更新数据库。缺点：可能导致数据库和redis中的数据都是旧数据（删除redis后，再更新数据时失败了）解决方案 延迟双删 写完数据库后，再删除一次。第二次删除缓存，并非立马就删而是等待时间大于读写缓存的时间即可。然后是先更新数据库再删除redis 。缺点：可能导致数据库和redis的数据不一致（redis删除失败）。出错时使用重试机制异步重新处理或者订阅binlog的方式解决。



### redis常用的缓存读写策略

旁路缓存模式
写:先更新 db 然后直接删除 cache 。
读:从 cache 中读取数据，读取到就直接返回 cache 中读取不到的话，就从 db 中读取数据返回 再把数据放到 cache 中。
读写穿透
服务端把 cache 视为主要数据存储，从中读取数据并将数据写入其中。cache 服务负责将此数据读取和写入 db，从而减轻了应用程序的职责。
异步缓存写入
异步缓存写入只更新缓存，不直接更新 db，改为异步批量的方式来更新 db。

### redis如何实现一个分布式锁

最简单的一种 加锁：setnx，解锁：del,问题：如果忘记解锁，将会出现死锁。
第二种分布式锁的实现方式：setnx+expire,解锁：del(key).问题：由于setnx和expire的非原子性，当第二步挂掉，仍然会出现死锁。
第三种方式：加锁：将setnx和expire变成原子性操作，set(key,1,30,NX),解锁：del（key）。但是有一种情况比如此时有进程A,如果进程A在任务没有执行完毕时,锁被到期释放了。这种情况下进程A在任务完成后依然会尝试释放锁,因为它的代码逻辑规定它在任务结束后释放锁,但是它的锁早已经被释放过了,那这种情况它释放的就可能是其他线程的锁。为解决这种情况,我们可以在加锁时为key赋一个随机值,来充当进程的标识,进程要记住这个标识。当进程解锁的时候进行判断,是自己持有的锁才能释放,否则不能释放。另外判断,释放这两步需要保持原子性,否则如果第二步失败,就会造成死锁。而获取和删除命令不是原子的,这就需要采用Lua脚本,通过Lua脚本将两个命令编排在一起,而整个Lua脚本的执行是原子的。



### 使用 Redis 实现一个排行榜怎么做？

Redis 中有一个叫做 sorted set 的数据结构经常被用在各种排行榜的场景
相关的一些 Redis 命令: ZRANGE (从小到大排序)、 ZREVRANGE （从大到小排序）、ZREVRANK (指定元素排名)。



### Redis 为什么还有无磁盘复制模式？

Redis 无磁盘复制模式是一种新的主从复制方式，其主要优点是可以减少网络带宽的占用，提高主从同步的速度，同时也减轻了主节点的磁盘负载。
他是将主节点中的数据直接传输给从节点，而需要将数据先写入到磁盘中。主节点不需要生成 RDB 文件或 AOF 文件，因此也不需要进行磁盘的读写操作。



### Redis Cluster 的优势主要包括：

横向扩展：Redis Cluster 可以方便地实现数据的横向扩展，通过增加 Redis 节点的数量，可以增加集群的容量和性能，而无需更改应用程序。
高可用性：Redis Cluster 采用主从复制机制来保证数据的高可用性。当主节点失效时，可以自动切换到从节点，从而避免单点故障问题。
分布式管理：Redis Cluster 可以方便地进行节点的添加、删除和重分片等操作，而无需停止服务。
自动负载均衡：Redis Cluster 可以根据每个节点的负载情况，自动将请求分配到最适合的节点上，从而实现负载均衡。

### Redis Cluster 是如何分片的？

Redis Cluster 使用哈希槽分片策略，将数据按照键名的哈希值映射到一个 0 到 16383 的哈希槽上，然后将每个哈希槽分配到不同的节点上进行存储。

### Redis Cluster 支持重新分配哈希槽吗？

是的，如果遇到新增节点、节点故障、节点扩容等情况，这些变化可能会导致哈希槽的分布不再均匀，进而影响集群的性能和可用性。Redis Cluster 提供了一种叫做“resharding”的机制，可以重新分配哈希槽，使其均匀地分布在新的节点上。

### Redis Cluster 扩容缩容期间可以提供服务吗？

在 Redis Cluster 扩容或缩容期间，集群仍然可以提供服务，但可能会对集群的性能和可用性产生一定的影响。

### Redis Cluster 中的节点是怎么进行通信的？

通过 Gossip 协议进行通信；