# redis 笔记

### [Redis 应用](https://github.com/martin-1992/redis_notebook/tree/master/Redis%20%E5%BA%94%E7%94%A8)

- 字符串 string；
    1. 作为计数器，比如点赞数、阅读数、转发数、视频或直播在线观看人数等；
    2. 分布式 ID，每次从 redis 中 incr 获取加一的 ID；
    3. session 管理，每次用户更新或查询登录都直接从 redis 中获取；
    4. 限速、限流，比如 3 分钟只能获取一次验证码、一分钟内只能请求 10 次等。
- 列表 List，作为消息队列或栈；
- 集合 Set；
    1. 求两个集合的交集，可作为共同关注、好友等；
    2. 秒杀集合，结合发布订阅，将请求添加到集合中，进行消费为其创建订单，集合长度超过秒杀数量则返回秒杀失败。
- 哈希表 Hash，缓存对象信息，比如用户信息、购物车信息等；
- 有序集合 Zset，排行榜；
- Geospatial，以给定经纬度为中心，找出某一半径内的元素，可获取身边的人；
- Hyperloglog，基数统计算法，统计 UV；
- 位图 Bitmap。
    1. 作为签到、是否登录等；
    2. 布隆过滤器，redis 4.0 后不用位图，官方有提供。
- 分布式锁。

#### 计数器
　　string 结构，最简单的结构，使用 incr 进行加一。公众号文章、微博，都有阅读数。

#### 消息队列
　　使用列表对象实现，列表是双端队列，可实现队列和栈，分为 Lpush、Lpop、Rpush、Rpop。<br />
　　队列，即模仿 MQ 的入队和出队。

#### 共同关注、好友
　　使用 redis 集合实现，两个集合的交集即可得出共同关注、好友等。

- 差集，SDIFF key1 key2；
- 交集，SINTER key1 key2；
- 并集，SUNION key1 key2。

#### [点赞](https://github.com/martin-1992/redis_notebook/tree/master/Redis%20%E5%BA%94%E7%94%A8/%E7%82%B9%E8%B5%9E)
　　部分内容来自 [[WeiDesign]微博计数器的设计(下)](https://blog.cydu.net/weidesign/2012/09/09/weibo-counter-service-design-2/#)，介绍了在大数据量下，如何对 redis 存储进行优化，从 key 和 value 两方面入手。<br />
　　用到的方法有，**一是为 0 的都不存储，查询不到即返回 0。二是 key 和 value 使用数组，在进行压缩。三是根据 ID 的时间属性区分冷、热数据，冷数据存到磁盘。**

- MySQL 实现，维护三张表；
    1. 表为用户 ID + 文章 ID，保存每个人的点赞信息，判断该用户是否点赞过某文章，获取某文章的点赞用户列表；
    2. 文章点赞数表，key 为文章 ID，获取文章点赞数；
    3. 用户点赞数表，key 为用户 ID，获取用户点赞数。
- redis 实现，使用 string + set；
    1. string，get 用于获取点赞数量，incr 点赞数加一操作；
    2. 还可以使用 hash 来存储，创建多个当天的 hash，每个 hash 存储一定数量的文章 ID 和点赞值。hash 名包含时间属性，可根据时间远近，区分冷、热数据，将时间久的 hash 删去缓存，有查询时从数据库中获取；
    4. set，集合判断该用户是否已对某文章点赞。
- 微博的计数器基于 redis 二次开发，主要在存储方面进行优化；
    1. 为 0 的数据不存储，不存储是种默认值，表示返回 0，可节省一半内存；
    2. 两个数组，key 为微博 ID 数组，value 为对应微博 ID 数组的评论数和转发数的数组，比如微博 ID 数组为 [35, 36]，value 为 [[464, 239], [10, 15]]，微博 ID 为 35，其转发数和评论数分别为 464、239 等。 
    3. 将两个数组进行 LZF 压缩，再存储，节省空间；
- 微博 ID，应包含时间属性，可根据时间来区分冷、热数据，冷数据存在磁盘，热数据存在内存，使用定时器来清除 90 天以前的数据。

#### 分布式 ID
　　最简单的，使用 string 的 incr。每次获取全局业务 ID，到 redis 中进行 incr，操作成功，则使用该 ID。缺点是业务量大，频繁请求，网络开销大。<br />
　　第二种是从 redis 中获取一批 key 进行操作，使用 string 的 incrby，一次性增加 2000 等。incrby 成功后，在本地机器 A 发放 1000 ~ 3000 的用户 ID。即使本地缓存丢失，就浪费一批 ID，可重新获取新的一批。<br />
　　如果有多台机器来作为分布式 ID 集群，比如 3 台机器，每台机器的初始 ID 为 1，2，3，每次 incrby 3 即可，即每次增加步长为 3。

#### 缓存对象信息
　　以购物车为例，使用哈希表 hash 来实现，key 格式为 cart:购物车 ID，比如 cart:3234，购物车 ID 为 3234 的哈希表。<br />
　　哈希表适合对象存储，string 适合字符串存储。

#### 发布和订阅，秒杀
　　两个 redis 集合 set，一个 setA 查询该用户是否已秒杀过，另一个 setB 为已被秒杀商品的集合，如果该 setB 长度超过指定的秒杀数，则表示抢完。<br />
　　用户抢购秒杀商品时，是将消息推送到另一个频道，由该频道将用户 ID 添加到已被秒杀商品的集合 setB。另外一个线程，处理 setB 中的元素，为其创建订单记录，创建成功，再将消息推送到另一个频道，由前端解析消息，告知用户秒杀成功。

- 先到 redis 的用户集合 map 中，查询该用户是否已经抢过，抢过则返回；
- 通过秒杀活动 ID，查询活动商品表是否存在秒杀商品。不存在，则返回该秒杀活动不存在；
- 存在，根据用户 ID，查询秒杀活动订单表。
    1. 存在表示该用户已经抢到商品；
    2. 不存在，获取则获取活动商品表的数量，如小于 0，表示商品已抢完。
- 拼接 key + 活动 ID，获取秒杀商品集合 map。判断该 Map 的长度，如果长度超过秒杀数量，表示抢完，则直接返回；
- Map 长度没超过秒杀数量，则包装成消息，发送到 redis 的订单频道 orderChannel；
- redis 的订单频道 orderChannel 接到消息后，解析消息，将用户 ID 和活动 ID 添加到秒杀集合中；
- 定时任务，定时遍历秒杀集合，为集合中的用户创建秒杀订单记录，同时会将该用户添加到用户集合，防止同一个用户多次抢购；
- 创建完订单记录，商品数量减一后，会发送消息到 imChannel。由前端解析该消息，推给用户表示已经抢购成功。会有个订单编号，用户点击跳转到订单详情页面完成后面的支付逻辑。

#### 排行榜
　　有序集合 Zset 实现排行榜，消息排序（根据消息权重区分重要消息、普通消息来排序）等。<br />
　　比如统计主播的收益排行榜，主播 id 作为 member，当天打赏的活动礼物对应的热度值作为 score, 通过 zrangebyscore 获取主播活动日榜。

#### [布隆过滤器](https://github.com/martin-1992/redis_notebook/tree/master/%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8)
　　**内存占用小，解决缓存穿透的方法。** 缓存穿透指查询不存在的值，会去存储层查询。大流量情况下，会对存储层造成大的负担。<br />
　　**它由一个 bit 数组和一组 Hash 算法构成，可用于判断一个元素是否在一个集合中。** 它是将所有地址经过多个 Hash 算法，映射到一个 bit 数组，其数学原理在于两个完全随机的数字相冲突的概率很小。

#### [Redis 分布式锁](https://github.com/martin-1992/redis_notebook/tree/master/Redis%20%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81)

- 使用 jedis 客户端，自己实现分布式锁；
- Redission 封装了分布式锁的实现，可直接调用；
- RedLock，解决主从的锁同步问题。
    1. 比如机器 A 向主机器申请锁，这锁没有同步到 B 机器，然后主机器宕机了。这时 B 机器去从机器能申请到锁；
    2. RedLock 解决方法是使用少数服从多数原则，机器 A 申请锁时，是向多个 redis 机器申请锁，当获得多数锁时，才成功获取锁，适用于更严格的场景。

#### 身边的人
　　Geospatial 属于 redis 的特殊数据类型，**以给定经纬度为中心，找出某一半径内的元素。** <br />
　　GEO 底层实现原理是 Zset，使用 Zset 命令来操作。

- geoadd china:city 116.40 39.90 beijing 114.05 22.52 shengzhen，添加，需要经纬度；
- geopos china:city beijing，获取指定 key 的经纬度；
- geodist china:city beijing shengzhen km，获取两地的距离；
- georadius weixin:person 115 20 10km count 100，查询经纬度 115、20 为中心，寻找 10km 以内的人，筛选获取 100 个。

#### 统计 UV
　　Hyperloglog 属于 redis 的特殊数据类型，作为基数统计算法，**可用于估计网页的 UV。** <br />
　　对比使用 set 来保存用户 ID，Hyperloglog 优点是占用内存固定，只要 12 kb，缺点是为估计，只能计数，且有 0.81% 错误率。<br />
　　如果需要精确统计，并获得每个人的 ID，使用 set。

#### Bitmap
　　只有两个状态 0 和 1，非常省内存。可用为统计用户信息，比如签到、是否登录等。

- setbit 标识:日期 uid 1，比如 uid 为 546 的用户在 2020.03.15 签到，则为 setbit SignIn:20200315 546 1；
- getbit 标识:日期 uid，获取某位用户是否签到，比如 setbit SignIn:20200315 546；

### [Redis 事务](https://github.com/martin-1992/redis_notebook/tree/master/Redis%20%E4%BA%8B%E5%8A%A1)
　　非原子性的，有一条命令失败，也会执行下去，**本质是将多条命令放到一起执行。** <br />

- MULTI 命令，表示事务开始。这时开始输入其他命令，会进入 redis 的命令事务队列；
- EXEC 表示事务执行，redis 会执行事务队列中的命令；
- DISCARD 表示丢弃事务队列中的所有命令，Redis 不支持回滚操作。

### Redis 乐观锁
　　WATCH 命令是一个乐观锁，原理为 CAS，检查被监视的键是否有修改过，是则拒绝执行事务。

### [Redis 持久化机制](https://github.com/martin-1992/redis_notebook/tree/master/Redis%20%E6%8C%81%E4%B9%85%E5%8C%96%E6%9C%BA%E5%88%B6)
　　将内存的数据保存到磁盘中，防止机器意外断电、重启导致数据丢失。Redis 有两种持久化方法：

- RDB 持久化，快照形式，全量备份。生成一个压缩的二进制文件 RDB 文件，保存的是数据库的键值对数据；
- AOF 持久化，增量备份。每隔一段时间（比如每隔一秒）保存 Redis 服务器所执行的写命令。

### [Redis 过期机制](https://github.com/martin-1992/redis_notebook/tree/master/Redis%20%E8%BF%87%E6%9C%9F%E6%9C%BA%E5%88%B6)
　　Redis 的过期策略为定期删除 + 惰性删除。注意，为防止大批 key 同时过期，导致 redis 频繁扫描影响性能。**建议将过期时间随机化，防止同时过期。** <br />
　　从库不会进行过期扫描，而是进行主从同步，存在主从延迟情况。

- 定时删除（主动删除），对内存友好，但对 CPU 不友好。在 CPU 使用高峰时，执行定时删除，会影响其性能。同时，创建定时器用到 Redis 服务器的时间事件，它是使用无序链表，导致查找一个事件的时间复杂度为 O(N)；
- 惰性删除（被动删除），对 CPU 时间最友好，但对内存最不友好。 获取键时，检查是否过期，过期则删除。如果有很多键没有访问获取（数据太旧），则没法删除，相当于内存泄漏；
- 定期删除（主动删除），前两种的折中。 设定删除操作执行的时长，每隔一段时间，检查一次并删除。

### [Redis 内存淘汰机制](https://github.com/martin-1992/redis_notebook/tree/master/Redis%20%E5%86%85%E5%AD%98%E6%B7%98%E6%B1%B0%E6%9C%BA%E5%88%B6)
　　当 Redis 的内存超过期望内存 maxmemory 时，会执行内存淘汰机制。Redis 的过期机制有可能会导致内存泄漏，需要对 key 执行淘汰策略。<br />
　　常用的有近似 LRU 算法，Redis 随机抽取 5 个 key，检查这 5 个 key 最后一次的访问时间戳，丢弃最旧的那个，一直重复该流程。

### redis 热点 key

- 用户消费的数据远大于生产的数据，比如热卖商品或秒杀商品、热点新闻、热点评论等，这些典型的读多写少的场景会产生热点问题；
- 某个 key 的请求超过一台单机的性能极限，假设一台单机能承受 3000 请求，这时有 5000 请求查询，也为热点 key。

#### 解决方案

- redis 集群扩容，增加分片副本，均衡读流量；
- 对于超过 redis 单机的性能极限的，将热 key 进行备份，分成 key1、key2 ... 等，分到不同分片，访问时随机选一个，分散流量；
- 同时将热 key 缓存到本机，减少 redis 请求。

### redis 性能测试
　　使用 redis-benchmark，比如 redis-benchmark -h localhost -p 6379 -c 100 -n 100000，表示测试 redis 地址为 localhost:6369，100 个并发连接，100000 个请求。

### redis 应对缓存方案

- 缓存击穿，热点 key 过期，导致大量请求打到数据库。
    1. 使用分布式锁，保证只有一个请求到数据库获取，接着存到 redis 中；
    2. 不让热点 key 失效，先识别出，并主动将其存到 redis 中。
- 缓存穿透，查询不存在的数据，直接穿透到数据库查询；
    1. 布隆过滤器；
    2. 回种短期空值。
- 缓存雪崩，大量 key 同时过期。
    1. 设置 key 的过期时间时在加上个随机值；

### redis 踩坑

#### 重设 set 命令去掉过期时间
　　当一个 key 用 set 设置过期时间后，在重新设置不带过期时间，会导致过期时间消失。<br />
　　在项目中需要合理评估 redis 容量，避免频繁 set 导致没有过期策略，间接导致内存溢出。

#### Jedis 2.9.0 及以下版本过期设置 BUG
　　Jedis 调用 expireAt 命令时有 bug，最终调用的是 pexpire 命令，这个 bug 会导致 key 过期时间很长，出现内存溢出。解决方法，是升级。

#### Redis-standalone 架构禁止使用非 0 库
　　RedisTemplate 在执行 execute 方法，会获取连接。当选择库大于 1 时，会调用 select db 进行切换。<br />
　　注意，执行完 execute 方法后，会释放连接时，redis 会将库重新切换为 0。**即每次连接会从 0 切换到另一个库，执行完后又切换回 0 库，这样切换损耗性能。** <br />
　　Rediscluster 集群数据库，默认 0 库，无法选择其他数据库。

#### 时间复杂度 O(N) 命令
　　redis 是单线程的，而这种命令，其执行时间与集合的长度有关，如果集合长度非常长，会占用 redis 唯一个线程来进行处理。<br />
　　这时在出现高并发查询该集合，导致每个请求变长，CPU 涨到 100%，阻塞了其他命令执行。建议 set、hash、list 等集合中的元素个数不超过 1000。

#### 使用 pipeline 一次返回多个
　　比如查询榜单，一次查询日榜、周榜、月榜等，同时返回三个榜单数据。对比三次查询，更优。

#### 禁止使用 Monitor、Keys、Save、BGREWRITEAOF 命令

- Monitor 命令，在高并发下，会存在内存暴增和影响 Redis 性能的隐患；
- keys 命令，会遍历所有 key，前面提到 O(N) 下，如果 key 非常多，会阻塞其他命令；
- Save 命令，会阻塞当前 redis 服务器，先进行持久化，直到持久化完成；
- BGREWRITEAOF 命令，手动 AOF，同样对内存较大的实例会造成长时间的阻塞。

### redis 运维

#### 慢查询

- slowlog get 5，获取慢查询日志；
- CacheCloud 工具监控

#### redis 实例的 CPU 使用率
　　redis 是单线程，一般使用率为 10%，如高于 20%，需考虑进行 RDB 持久化。

#### redis 分片负载均衡
　　使用命令 redis-cli -p{port} -h{host} --stat 获取 Redis-cluster 每个分片 requests 流量均衡情况，超过 12W 需告警。

#### BigKey
　　定时扫描大 key，进行优化。命令为 redis-cli -h 127.0.0.1 -p {port} --bigkeys 或 redis-memory-for-key -s {IP} -p {port} XXX_KEY。

#### Redis 占用内存大小
　　Info memory 命令查看，避免在高并发场景下，由于分配的 MaxMemory 被耗尽，带来的性能问题。<br />
　　重点关注 used_memory_human 配置项对应的 value 值，增量过高时，需要重点评估。

