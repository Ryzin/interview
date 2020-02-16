---
title: Redis
date: 2019-08-21T11:00:41+08:00
draft: false
categories: database
---

# Redis

## 线程模型

Redis 在处理网络请求是使用单线程模型，并通过 IO 多路复用来提高并发。但是在其他模块，比如：持久化，会使用多个线程。

Redis 内部使用文件事件处理器 `file event handler`，**这个文件事件处理器是单线程的，所以 `redis` 才叫做单线程的模型**。它采用 IO 多路复用机制同时监听多个 `socket` ，将产生事件的 `socket` 压入内存队列中，事件分派器根据 `socket` 上的事件类型来选择对应的事件处理器进行处理。

文件事件处理器的结构包含 4 个部分：
  - 多个 socket
  - IO 多路复用程序
  - 文件事件分派器
  - 事件处理器（连接应答处理器、命令请求处理器、命令回复处理器）

多个 `socket` 可能会并发产生不同的操作，每个操作对应不同的文件事件，但是 `IO` 多路复用程序会监听多个 `socket` ，会将产生事件的 `socket` 放入队列中排队，事件分派器每次从队列中取出一个 `socket` ，根据 `socket` 的事件类型交给对应的事件处理器进行处理。

客户端与 redis 的一次通信过程：

![image](images/f0dacdd3779b836ad75fe6b886af1fff.png)

### 为啥 redis 单线程模型也能效率这么高？
  - 纯内存操作
  - 核心是基于非阻塞的 IO 多路复用机制
  - 单线程反而避免了多线程的频繁上下文切换问题

## 数据结构

Redis的外围由一个键、值映射的字典构成。与其他非关系型数据库主要不同在于：Redis中值的类型不仅限于 **字符串**，还支持如下抽象数据类型：
  - **List**：字符串列表
  - **Set**：无序不重复的字符串集合
  - **Soret Set**：有序不重复的字符串集合
  - **HashTable**：键、值都为字符串的哈希表

值的类型决定了值本身支持的操作。Redis支持不同无序、有序的列表，无序、有序的集合间的交集、并集等高级服务器端原子操作。

## 持久化：

  - 使用快照，一种半持久耐用模式。不时的将数据集以异步方式从内存以RDB格式写入硬盘。
  - 1.1版本开始使用更安全的 `AOF` 格式替代，一种只能追加的日志类型。将数据集修改操作记录起来。Redis 能够在后台对只可追加的记录作修改来避免无限增长的日志。

```
  1. aof文件比rdb更新频率高，优先使用aof还原数据。
  2. aof比rdb更安全也更大
  3. rdb性能比aof好
  4. 如果两个都配了优先加载AOF
```

## 一致性哈希算法

一致哈希 是一种特殊的哈希算法。在使用一致哈希算法后，哈希表槽位数（大小）的改变平均只需要对 `K/n` 个关键字重新映射，其中 `K` 是关键字的数量，`n` 是槽位数量。然而在传统的哈希表中，添加或删除一个槽位的几乎需要对所有关键字进行重新映射。

> 一致哈希也可用于实现健壮缓存来减少大型 Web 应用中系统部分失效带来的负面影响

### 需求

在使用 `n` 台缓存服务器时，一种常用的负载均衡方式是，对资源 `o` 的请求使用 `hash(o)= o mod n` 来映射到某一台缓存服务器。当增加或减少一台缓存服务器时这种方式可能会改变所有资源对应的 `hash` 值，也就是所有的缓存都失效了，这会使得缓存服务器大量集中地向原始内容服务器更新缓存。

因此需要一致哈希算法来避免这样的问题。 一致哈希尽可能使同一个资源映射到同一台缓存服务器。这种方式要求增加一台缓存服务器时，新的服务器尽量分担存储其他所有服务器的缓存资源。减少一台缓存服务器时，其他所有服务器也可以尽量分担存储它的缓存资源。

一致哈希算法的主要思想是将每个缓存服务器与一个或多个哈希值域区间关联起来，其中区间边界通过计算缓存服务器对应的哈希值来决定。如果一个缓存服务器被移除，则它所对应的区间会被并入到邻近的区间，其他的缓存服务器不需要任何改变。

### 实现

一致哈希将每个对象映射到圆环边上的一个点，系统再将可用的节点机器映射到圆环的不同位置。查找某个对象对应的机器时，需要用一致哈希算法计算得到对象对应圆环边上位置，沿着圆环边上查找直到遇到某个节点机器，这台机器即为对象应该保存的位置。

当删除一台节点机器时，这台机器上保存的所有对象都要移动到下一台机器。添加一台机器到圆环边上某个点时，这个点的下一台机器需要将这个节点前对应的对象移动到新机器上。更改对象在节点机器上的分布可以通过调整节点机器的位置来实现。

## [实践](https://yikun.github.io/2016/06/09/%E4%B8%80%E8%87%B4%E6%80%A7%E5%93%88%E5%B8%8C%E7%AE%97%E6%B3%95%E7%9A%84%E7%90%86%E8%A7%A3%E4%B8%8E%E5%AE%9E%E8%B7%B5/)

> 假设有1000w个数据项，100个存储节点，请设计一种算法合理地将他们存储在这些节点上。

看一看普通Hash算法的原理：

![](images/1c5e07626a9cadd5f1ea8acd85838067.png)

```
for item in range(ITEMS):
    k = md5(str(item)).digest()
    h = unpack_from(">I", k)[0]
    # 通过取余的方式进行映射
    n = h % NODES
    node_stat[n] += 1
```

普通的Hash算法均匀地将这些数据项打散到了这些节点上，并且分布最少和最多的存储节点数据项数目小于 `1%`。之所以分布均匀，主要是依赖 Hash 算法（实现使用的MD5算法）能够比较随机的分布。

然而，我们看看存在一个问题，由于 **该算法使用节点数取余的方法，强依赖 `node` 的数目**，因此，当是 `node` 数发生变化的时候，`item` 所对应的 `node` 发生剧烈变化，而发生变化的成本就是我们需要在 `node` 数发生变化的时候，数据需要迁移，这对存储产品来说显然是不能忍的。

#### 一致性哈希

普通 `Hash` 算法的劣势，即当 `node` 数发生变化（增加、移除）后，数据项会被重新“打散”，导致大部分数据项不能落到原来的节点上，从而导致大量数据需要迁移。

那么，一个亟待解决的问题就变成了：当 `node` 数发生变化时，如何保证尽量少引起迁移呢？即当增加或者删除节点时，对于大多数 item ，保证原来分配到的某个 node ，现在仍然应该分配到那个 node ，将数据迁移量的降到最低。

![](images/3de376ea57386b890483b27cf131f24d.png)

```
for n in range(NODES):
    h = _hash(n)
    ring.append(h)
    ring.sort()
    hash2node[h] = n
for item in range(ITEMS):
    h = _hash(item)
    n = bisect_left(ring, h) % NODES
    node_stat[hash2node[ring[n]]] += 1
```

**虽然一致性Hash算法解决了节点变化导致的数据迁移问题，但是，数据项分布的均匀性很差**。

![](images/5e6b9afd23ff44415b434d05ed0449ce.png)

主要是因为这 100 个节点 Hash 后，在环上分布不均匀，导致了每个节点实际占据环上的区间大小不一造成的。

#### 改进 -- 虚节点

当我们将 node 进行哈希后，这些值并没有均匀地落在环上，因此，最终会导致，这些节点所管辖的范围并不均匀，最终导致了数据分布的不均匀。

![](images/c807b7a0af060a874fdb27abf5caf289.png)

```
for n in range(NODES):
    for v in range(VNODES):
        h = _hash(str(n) + str(v))
        # 构造ring
        ring.append(h)
        # 记录hash所对应节点
        hash2node[h] = n
ring.sort()
for item in range(ITEMS):
    h = _hash(str(item))
    # 搜索ring上最近的hash
    n = bisect_left(ring, h) % (NODES*VNODES)
    node_stat[hash2node[ring[n]]] += 1
```

通过增加虚节点的方法，使得每个节点在环上所“管辖”更加均匀。这样就既保证了在节点变化时，尽可能小的影响数据分布的变化，而同时又保证了数据分布的均匀。也就是靠增加“节点数量”加强管辖区间的均匀。

## 集群

### 哨兵 -- Sentinel

`Redis-Sentinel` 是 Redis 官方推荐的 **高可用性( `HA` )解决方案**，当用 Redis 做 `Master-slave` 的高可用方案时，假如 `master` 宕机了， Redis 本身(包括它的很多客户端)都没有实现自动进行主备切换，而 `Redis-sentinel` 本身也是一个独立运行的进程，它能监控多个 `master-slave` 集群，发现 `master` 宕机后能进行自懂切换。

它的主要功能有以下几点

  - 不时地监控 redis 是否按照预期良好地运行;
  - 如果发现某个 redis 节点运行出现状况，能够通知另外一个进程(例如它的客户端);
  - 能够进行自动切换。当一个 master 节点不可用时，能够选举出 master 的多个slave (如果有超过一个 slave 的话)中的一个来作为新的 master ,其它的 slave 节点会将它所追随的 master 的地址改为被提升为 master 的 slave 的新地址。

很显然，**只使用单个 sentinel 进程来监控 redis 集群是不可靠的**，当 sentinel 进程宕掉后( sentinel 本身也有单点问题，single-point-of-failure)整个集群系统将无法按照预期的方式运行。所以有必要将sentinel集群，这样有几个好处：

  - 即使有一些sentinel进程宕掉了，依然可以进行redis集群的主备切换；
  - 如果只有一个sentinel进程，如果这个进程运行出错，或者是网络堵塞，那么将无法实现redis集群的主备切换;
  - 如果有多个sentinel，redis的客户端可以随意地连接任意一个sentinel来获得关于redis集群中的信息。

### [Redis Cluster](https://redis.io/topics/cluster-tutorial)

Redis Cluster 是一种服务器 `Sharding` 技术，3.0版本开始正式提供。

Redis Cluster中，Sharding 采用 *slot(槽)* 的概念，一共分成 `16384` 个槽，这有点儿类 `pre sharding` 思路。对于每个进入 Redis 的键值对，根据 key 进行散列，分配到这 `16384` 个 slot 中的某一个中。使用的hash算法也比较简单，就是 `CRC16` 后 `16384` 取模。要保证 `16384` 个槽对应的 node 都正常工作，**如果某个 node 发生故障，那它负责的 slots 也就失效，整个集群将不能工作**。

> 16384 = 2048 * 8 bit，2k 大小的 bit 数

为了增加集群的可访问性，官方推荐的方案是将 node 配置成 **主从结构**，即一个 master 主节点，挂 `n` 个 slave 从节点。这时，如果主节点失效，Redis Cluster 会根据选举算法从 slave 节点中选择一个上升为主节点，整个集群继续对外提供服务。

对客户端来说，**整个 cluster 被看做是一个整体**，客户端可以连接任意一个 node 进行操作，就像操作单一 Redis 实例一样，当客户端操作的 key 没有分配到该 node 上时，Redis 会返回**转向指令**，指向正确的 node ，这有点儿像浏览器页面的 `302  redirect` 跳转。

### Redis Sharding 集群

Redis Sharding 是客户端 Sharding 的方案，其主要思想是采用哈希算法 **将 Redis 数据的 key 进行散列**，通过 hash 函数，特定的 key 会映射到特定的 Redis 节点上。这样，客户端就知道该向哪个 Redis 节点操作数据。

Jedis 的 Redis Sharding 实现具有如下特点：
  - **采用一致性哈希算法(consistent hashing)**，将key和节点name同时hashing，然后进行映射匹配，采用的算法是MURMUR_HASH。采用一致性哈希而不是采用简单类似哈希求模映射的主要原因是当增加或减少节点时，不会产生由于重新匹配造成的rehashing。一致性哈希只影响相邻节点key分配，影响量小。
  - 为了避免一致性哈希只影响相邻节点造成节点分配压力， `ShardedJedis` 会对每个Redis 节点根据名字(没有，Jedis会赋予缺省名字)会 **虚拟化出160个虚拟节点** 进行散列。根据权重 `weight` ，也可虚拟化出160倍数的虚拟节点。用虚拟节点做映射匹配，可以在增加或减少 `Redis` 节点时，key 在各 `Redis` 节点移动再分配更均匀，而不是只有相邻节点受影响。
  - **ShardedJedis 支持 keyTagPattern 模式**，即抽取 key 的一部分 `keyTag` 做 `sharding` ，这样通过合理命名 key ，可以将一组相关联的key放入同一个 Redis 节点，这在避免跨节点访问相关数据时很重要。

## string

Redis 没有直接使用 C 语言传统的字符串表示（以空字符结尾的字符数组，以下简称 C 字符串）， 而是自己构建了一种名为简单动态字符串（simple dynamic string，SDS）的抽象类型， 并将 SDS 用作 Redis 的默认字符串表示。

在 Redis 里面， C 字符串只会作为字符串字面量（string literal）， 用在一些无须对字符串值进行修改的地方， 比如打印日志。

当 Redis 需要的不仅仅是一个字符串字面量， 而是一个可以被修改的字符串值时， Redis 就会使用 SDS 来表示字符串值： 比如在 Redis 的数据库里面， 包含字符串值的键值对在底层都是由 SDS 实现的。

| C字符串     | SDS     |
| :------------- | :------------- |
|获取字符串长度的复杂度为 O(N) 。	|获取字符串长度的复杂度为 O(1) 。|
|API 是不安全的，可能会造成缓冲区溢出。	|API 是安全的，不会造成缓冲区溢出。|
|修改字符串长度 N 次必然需要执行 N 次内存重分配。	|修改字符串长度 N 次最多需要执行 N 次内存重分配。|
|只能保存文本数据。	|可以保存文本或者二进制数据。|
|可以使用所有 <string.h> 库中的函数。	|可以使用一部分 <string.h> 库中的函数。|

### 缓冲区溢出

因为 C 字符串不记录自身的长度， 所以 `strcat` 假定用户在执行这个函数时， 已经为 `dest` 分配了足够多的内存， 可以容纳 `src` 字符串中的所有内容， 而一旦这个假定不成立时， 就会产生缓冲区溢出。

举个例子， 假设程序里有两个在内存中紧邻着的 C 字符串 `s1` 和 `s2` ， 其中 s1 保存了字符串 `"Redis"` ， 而 s2 则保存了字符串 `"MongoDB"` ， 如图所示。

![](images/a9832e14ba184a4049f979e521ef050b.png)

如果一个程序员决定通过执行：

```
strcat(s1, " Cluster");
```

将 `s1` 的内容修改为 `"Redis Cluster"` ， 但粗心的他却忘了在执行 `strcat` 之前为 `s1` 分配足够的空间， 那么在 `strcat` 函数执行之后， `s1` 的数据将溢出到 `s2` 所在的空间中， 导致 `s2` 保存的内容被意外地修改， 如图所示。

![](images/cc9ac0419ae3f5059076c2d66f867931.png)

与 `C` 字符串不同， `SDS` 的空间分配策略完全杜绝了发生缓冲区溢出的可能性： **当 SDS API 需要对 SDS 进行修改时， API 会先检查 SDS 的空间是否满足修改所需的要求**， 如果不满足的话， API 会自动将 `SDS` 的空间扩展至执行修改所需的大小， 然后才执行实际的修改操作， 所以使用 `SDS` 既不需要手动修改 `SDS` 的空间大小， 也不会出现前面所说的缓冲区溢出问题。

### 减少修改字符串时带来的内存重分配次数

  - 空间预分配：解决 append 问题
  - 惰性空间释放：解决 strim 问题

### 二进制安全

C 字符串中的字符必须符合某种编码（比如 `ASCII`）， 并且 **除了字符串的末尾之外， 字符串里面不能包含空字符**， 否则最先被程序读入的空字符将被误认为是字符串结尾 —— 这些限制使得 C 字符串只能保存文本数据， 而不能保存像图片、音频、视频、压缩文件这样的二进制数据。

## zset底层实现

跳跃表（skiplist）是一种有序数据结构， 它通过在每个节点中维持多个指向其他节点的指针， 从而达到快速访问节点的目的。

`Redis` 使用跳跃表作为有序集合键的底层实现之一：
  - 如果一个有序集合包含的元素数量比较多，
  - 有序集合中元素的成员（`member`）是比较长的字符串时

Redis 就会使用跳跃表来作为有序集合键的底层实现。

和链表、字典等数据结构被广泛地应用在 Redis 内部不同， **Redis 只在两个地方用到了跳跃表， 一个是实现有序集合键， 另一个是在集群节点中用作内部数据结构**， 除此之外， 跳跃表在 Redis 里面没有其他用途。

## BitMap 实现

Bitmaps 并不是实际的数据类型，而是定义在 String 类型上的一个面向字节操作的集合。因为字符串是二进制安全的，最大长度是 512M ，所以最长拥有 {{<katex>}}2^32{{</katex>}} 个不同字节。

Redis 提供以下 BitMap 操作接口：`setBit`、`getBit`、`bitCount`、`bitop`、`bitpos`。其中 `setBit`、`getBit`都是 {{<katex>}}O(1){{</katex>}} 复杂度的操作。

优势：
1. 基于最小的单位bit进行存储，所以非常省空间。
2. 设置时候时间复杂度O(1)、读取时候时间复杂度O(n)，操作是非常快的。
3. 二进制数据的存储，进行相关计算的时候非常快。
4. 方便扩容

限制：redis中bit映射被限制在512MB之内，所以最大是 {{<katex>}}2^32{{</katex>}} 位。

## 缓存穿透、缓存击穿、缓存雪崩

### 缓存穿透

访问一个不存在的key，缓存不起作用，请求会穿透到 DB，流量大时 DB 会挂掉。

#### 解决方案

  - 采用布隆过滤器，使用一个足够大的`bitmap`，用于存储可能访问的 `key`，不存在的key直接被过滤；

  - 访问key未在DB查询到值，也将空值写进缓存，但可以设置较短过期时间。

### 缓存雪崩

大量的 key 设置了相同的过期时间，导致在缓存在同一时刻全部失效，造成瞬时DB请求量大、压力骤增，引起雪崩。

#### 解决方案

可以给缓存设置过期时间时加上一个随机值时间，使得每个 key 的过期时间分布开来，不会集中在同一时刻失效。

### 缓存击穿

对于一些设置了过期时间的key，如果这些key可能会在某些时间点被超高并发地访问，是一种非常“热点”的数据。这个时候，需要考虑一个问题：缓存被“击穿”的问题，这个和缓存雪崩的区别在于这里针对某一 key 缓存，前者则是很多key。

缓存在某个时间点过期的时候，恰好在这个时间点对这个Key有大量的并发请求过来，这些请求发现缓存过期一般都会从后端DB加载数据并回设到缓存，这个时候大并发的请求可能会瞬间把后端 DB 压垮。

#### 解决方案

在缓存失效的时候（判断拿出来的值为空），不是立即去 `load db` ，而是先使用缓存工具的某些带成功操作返回值的操作（比如Redis的 `SETNX`）去 `set` 一个 `mutex key` ，当操作返回成功时，再进行 `load db` 的操作并回设缓存；否则，就重试整个 `get` 缓存的方法。


## Redis分布式锁

  - 加锁：`redis.set(String key, String value, String nxxx, String expx, int time)`
  - 解锁：通过 Lua 脚本执行 `if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end`

## 数据淘汰机制

### 对象过期

Redis回收过期对象的策略：定期删除+惰性删除

  - **惰性删除**：当读/写一个已经过期的key时，会触发惰性删除策略，直接删除掉这个过期key
  - **定期删除**：由于惰性删除策略无法保证冷数据被及时删掉，所以Redis会定期主动淘汰一批已过期的key

### 内存淘汰

Redis提供了下面几种淘汰策略供用户选择，其中默认的策略为noeviction策略：

  - `noeviction`：当内存使用达到阈值的时候，所有引起申请内存的命令会报错。
  - `allkeys-lru`：在主键空间中，优先移除最近未使用的key。
  - `volatile-lru`：在设置了过期时间的键空间中，优先移除最近未使用的key。
  - `allkeys-random`：在主键空间中，随机移除某个key。
  - `volatile-random`：在设置了过期时间的键空间中，随机移除某个key。
  - `volatile-ttl`：在设置了过期时间的键空间中，具有更早过期时间的key优先移除。

> 这里补充一下主键空间和设置了过期时间的键空间，举个例子，假设我们有一批键存储在Redis中，则有那么一个哈希表用于存储这批键及其值，如果这批键中有一部分设置了过期时间，那么这批键还会被存储到另外一个哈希表中，这个哈希表中的值对应的是键被设置的过期时间。设置了过期时间的键空间为主键空间的子集。

#### 非精准的LRU

上面提到的LRU（Least Recently Used）策略，实际上 **Redis 实现的 LRU 并不是可靠的 LRU**，也就是名义上我们使用LRU算法淘汰键，但是实际上被淘汰的键并不一定是真正的最久没用的，这里涉及到一个权衡的问题，如果需要在全部键空间内搜索最优解，则必然会增加系统的开销，Redis是单线程的，也就是同一个实例在每一个时刻只能服务于一个客户端，所以耗时的操作一定要谨慎。

为了在一定成本内实现相对的LRU，早期的 Redis 版本是 **基于采样的 LRU** ，也就是放弃全部键空间内搜索解改为采样空间搜索最优解。自从 Redis3.0 版本之后，Redis 作者对于基于采样的 LRU 进行了一些优化，目的是在一定的成本内让结果更靠近真实的 LRU。