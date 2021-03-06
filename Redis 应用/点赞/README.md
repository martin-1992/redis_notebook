### 介绍
　　点赞系统不能在前端限制，会被刷赞，只能在后端来判断一个人是否点赞。

### MySQL 实现
　　最基础的实现，维护两张表。

- 用户 ID + 文章 ID 表，保存每个人的点赞信息。
    1. 用于判断该用户是否已点赞，没有则插入；
    2. 根据文章 ID，获取点赞的用户列表。
- 文章点赞数表，通过文章 ID 获取到文章点赞数，进行加一。

　　上述两个操作使用事务机制，放在一个事务中，保证这两个操作同时成功。考虑并发情况，用户 ID + 文章 ID 为唯一索引，防止重复插入。文章点赞使用乐观锁，锁住某篇文章 ID，进行操作。<br />
　　分库分表时要做两表的冗余操作，单独使用用户 ID 或文章 ID 都没法使用，增加内存空间，还要一致性问题。

#### 数据量过大
　　以 weibo 业务为例，通过分库分表解决。

- 按 ID 取模，把数据分到 N 个表中。缺点是扩展性不好，加表的话需要重新迁移数据；
- 按 ID 时间取模，比如每天建一张表。缺点是会造成冷热不均，最近的是访问频繁的，最久的是没人访问的。可通过冷热库混合部署的方案来缓解，但是部署和维护的成本非常大。

#### 访问量过大
　　先 redis 查询，后 mysql 查询。

- 先从 redis 获取点赞数据，没有再到 mysql。
    1. 空数据得先到数据库获取点赞数据；
    2. 由于点赞更新快，缓存经常失效，要重新获取，还容易导致数据不一致。热点 key 的话，导致缓存击穿。
- 硬件解决，成本大。

### redis 实现
　　获取某篇文章 ID，使用 incr 加一。每个用户维护一张点赞列表，用于判断是否已对某篇文章点过赞。定时将其存到数据库中，大数据量和高并发都可通过堆机器，使用 codis 集群来解决。<br />
　　redis 使用两种结构，string 和 set。

- string，get 用于获取点赞数量，incr 点赞数加一操作；
    1. 还可以使用 hash 来存储，创建多个当天的 hash，每个 hash 存储一定数量的文章 ID 和点赞值；
    2. hash 名包含时间属性，可根据时间远近，区分冷、热数据，将时间久的 hash 删去缓存，有查询时从数据库中获取；
    3. 考虑根据文章 ID 如何路由到对应的 hash，使用对文章 ID 的哈希，路由到对应的 hash。
- set，集合判断该用户是否已对某文章点赞。

```redis
// 设置点赞数量为 898，key 格式为点赞业务:tid:文章 ID，这里文章 ID 为 325
127.0.0.1:6379[2]> set star:tid:325 898
// 使用 incr，将对应 key 加一
127.0.0.1:6379[2]> incr star:tid:325
// get，获取该文章的点赞数量
127.0.0.1:6379[2]> get star:tid:325
```

　　只有点赞数量不行，还需要在后端进行去重，使用 set 来判断该用户是否已点赞过某篇文章。

```redis
// set 的命名格式为点赞业务:list:tid:文章 ID，这里文章 ID 为 325，sadd 为批量添加点赞用户
127.0.0.1:6379[2]> sadd star:list:tid:325 123 456 789
// 判断用户 ID 为 456 是否在集合 star:list:tid:325 中，是则表示已点赞
127.0.0.1:6379[2]> sismember star:list:tid:325 456
```

　　使用 token 机制，防止刷赞。

- 请求开始后，将用户 ID + 资源 ID + 动作标识作为 token，存到 redis 中；
- 该动作没完成，则会存在。完成请求后，会从 redis 中删除；
- 点完赞后，再点赞，redis 检查到没该 token，则点赞不执行。
- 没点赞成功，再点赞。redis 检查到该 token 存在，则点赞执行。

　　缺点是内存利用率低，存在大量重复的 ID。当数据量大时，内存消耗量大。1000 亿 32 位数据为 400GB，但是需要 7.6TB 来存储。

### Counter
　　牺牲部分的通用性，针对微博转发和评论的大数据量和高并发访问的特点来进行定点优化。

- 大量微博没有转发或评论，就不存储，查不到时，就返回 0，这样能节省一半内存；
- 微博的评论和转发关联高，即评论多的，转发的也多。转发多的，评论也多。
    1. 将其放在一起，微博 ID + 评论数和微博 ID + 转发数更改为微博 ID + 评论数 + 转发数的数据结构，省了一个微博 ID 的空间；
    2. 同时，从两次请求（评论和转发）减少为一次请求。

#### 优化数据结构
　　微博 ID + 评论数 + 转发数的数据结构如下，用数字来存储，而不是字符串，两个数字只占 16 个字节。

```redis
struct item {
    int64_t weibo_id;
    int repost_num;
    int comment_num; 
};
```

- 程序启动时，开启内存 table_size * sizeof(item)；
- 插入时，使用 h1 = hash1(weibo_id) 和 h2 = hash2(weibo_id)；
    1. (h1 + h2*s) % table_size 为空，则将 item 存到该位置，s=0；
    2. 已有值，即哈希冲突，则 s + 1，继续使用 (h1 + h2*s) % table_size，并判断是否为空。为空则插入，不为空，则 s++，继续寻找可插入位置。
- 查询时，比较 weibo_id 和 item.weibo_id 是否一致，是则返回数据，如果没查到为空，则返回 0；
- 删除时，不直接删除，而是设置标志位。下次插入时，可以复用该内存。

#### 转发和评论数 Value 的优化
　　大部分转发和评论在几百、几千这种，优化 item 结构，使用更小内存的字节值。

```redis
struct item {
    int64_t weibo_id;
    unsigned short repost_num; 
    unsigned short comment_num;
};
```

　　当转发数和评论数大于 65535，则记录一个特殊的标志位 FFFF，去另外的字典中查询，使用正常 int。

#### value 优化
　　将 item 拆分成两个数组，key 数组和 value 数组，索引一致，key 可找到对应的 value。<br />
　　value 数组为二维的，包含转发和评论，**由于 value 数组中的值包含很多 0，可做压缩，存到一个 mini block。** 进行 LZF 压缩，存储压缩后的数据。获取时，先进行解压缩。

- 评论数和转发数直接使用 int，更简单，压缩后省内存；
- 还可以加列，变为多维数组。

#### key 优化
　　weibo_id 为 8byte，前面提到使用数组形式，进行压缩后能到 4byte。<br />

- 根据 weibo_id 的时间属性，将大的 table 划分成多个小的 table，用于将冷数据就存到磁盘中；
- 每个小 table 的数据都是 weibo_id 相近的，这些相近的 weibo_id，其前缀是相同的，将前缀作为小 table 的属性，只存储 weibo_id 后半部分，能省空间。

```redis
struct item{ 
    int weibo_id_low;
    unsigned short repost_num;
    unsigned short comment_num;
};
```

### 批量查询
　　Feed 页批量查询，一次获取 N 条微博，批量访问获取其计数。

#### 冷热数据
　　weibo 的数据存在以前的没人访问，不用存到内存中，只要放到磁盘可以。即热数据在内存，冷数据在磁盘。使用 LRU，会使得 struct item 膨胀，占用更多内存，且为 0 的数据也要存储。<br />

- 解决方法，weibo_id 是包含时间的，根据时间来划分区间；
    1. 超过半年的保存到磁盘上；
    2. 半年内的留在内存中；
    3. 当有用户查询半年前的，则去磁盘中获取，保存到 Cold Cache。
- 为了方便把旧的数据 dump 到磁盘，把那个大的 table_size 拆成多个小的 table，每个 table 都是不同的；
- 时间区间内的 weibo 计数，dump 的时候，以小的 table 为单位；
- 为了提高磁盘的查询效率，dump 之前先排序，并在内存中建好索引，索引建在 Block 上，而非 key 上；
- 一个 Block 可以是 4KB 甚至更长，根据索引最多一次随机 IO 把这个 Block 取出来，内存中再查询完成。

#### 数据的持久化
　　内存中的数据，分为快照和日志增量。

- 定期把 Block 完整的 dump 到磁盘中，形成 unsorted block；
- 每一次内存操作都会有相应的 Append log。

　　机器故障时，从磁盘上的 Block 上加载，再追加 Append log 中的操作日志来恢复数据。

#### 一致性保证
　　计数为非幂等性，使用消息队列，类似于 transid 的方案来做除重。另外，定期以实际的存储数据为准，来做校验。
　　单点问题使用主备结构解决，存在主节点宕机，没将最新点赞数更新到旧节点，导致旧节点升级为主节点后，点赞数不对。

#### 分布式化
　　按照 weiboid 取模，划分到 4 套上。每套 Master 存储后面又挂 2 个 Slave, 一方面是均摊读的压力，另一方面主要是容灾。当主挂掉的时候，还有副在，不影响读，也能够切换。

### 总结

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
- 微博 ID，应包含时间属性，可根据时间来区分冷、热数据，冷数据存在磁盘，热数据存在内存。

### reference

- [[WeiDesign]微博计数器的设计(下)](https://blog.cydu.net/weidesign/2012/09/09/weibo-counter-service-design-2/#)

