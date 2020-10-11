### 计数器
　　string 结构，最简单的结构，使用 incr 进行加一。公众号文章、微博，都有阅读数。

```redis
127.0.0.1:6379[2]> get example_key
36
127.0.0.1:6379[2]> incr example_key
37
```

### session 管理
　　每次用户更新或查询登录都直接从 redis 中获取。

### 消息队列
　　使用列表对象实现，列表是双端队列，可实现队列和栈，分为 Lpush、Lpop、Rpush、Rpop。<br />
　　队列，即模仿 MQ 的入队和出队。

### 延时队列
　　value 为序列化的任务消息，score 为下一次任务消息运行的时间。轮询 zset 中 score 值大于 now 的任务消息进行处理。<br />
　　在多线程情况下，为防止多个线程竞争轮询，写成 Lua 脚本保证原子性。

### 服务发现
　　zset 保存单个服务列表，value 为服务地址，score 为心跳时间。

- 服务提供者每隔几秒调用 zadd，类似使用心跳来汇报存活；
- 停止服务时，服务提供者调用 zrem 来移除自己；
- 还有一个线程，用于轮询，处理没使用 zadd 发送心跳的服务提供者，将其移除。

### 共同关注、好友
　　使用 redis 集合实现，两个集合的交集即可得出共同关注、好友等。

- 差集，SDIFF key1 key2；
- 交集，SINTER key1 key2；
- 并集，SUNION key1 key2。

### 防止并发，限流
　　string 结构，使用 incr 加一。当 incr 加的值超过限制值，则返回请求频繁的提示。<br />
　　比如，现在有个业务需求，需要限制一分钟只能请求 100 次。

- 给 key 设置一个过期时间，为 1 分钟；
- 每次进行 incr 成功后，本地机器获取该 key 的值。如果超过 100，则返回异常或提示。

#### 短信限制
　　另一种场景，以发短信为例，限制每 3 分钟只能发送一条。注意，**需考虑设置过期时间失败的问题，如果异常重启，导致过期时间没有设置成功，则会导致 key 永久存在，** 用户无法获取到第二条信息。

- 将业务标识 + 手机号，作为 key；
- 发送短信后，对该 key 进行 incr，值为 1 存储到 redis 中，设置过期时间为 3 分钟；
- 如果有新的短信请求过来，进行 incr 后判断返回的值，大于 1，表示有多次请求，则发出异常信息。

```java
// 手机号作为 key
String key = "SMS_LIMIT_" + phone;
int exp = 180;
try (Jedis redis = getRedis()) {
    // 加一后，获取值
    Long count = redis.incrBy(key.getBytes(), val);
    // 设置过期时间，高并发下即使执行到 increment，再切换另个线程，执行 increment，也不会影响 count 的值
    if (count == 1) {
        try {
            redis.expire(key, exp);
        } catch (Exception e) {
            redis.expire(key, exp);
        } finally {
            if (redis.ttl(key) == -1) {
                try {   
                    redis.expire(key, exp);
                } finally {
                    // 设置过期时间失败，发送 MQ 处理
                    sendMQ(key);
                }
            }
        }
    }

    if (count > 1) {
        System.out.print("每三分钟只能发送一次短信");
    }
} finally {
    redis.close();
}
```

### [点赞](https://github.com/martin-1992/redis_notebook/tree/master/Redis%20%E5%BA%94%E7%94%A8/%E7%82%B9%E8%B5%9E)
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

### 分布式 ID
　　最简单的，使用 string 的 incr。每次获取全局业务 ID，到 redis 中进行 incr，操作成功，则使用该 ID。缺点是业务量大，频繁请求，网络开销大。<br />
　　第二种是从 redis 中获取一批 key 进行操作，使用 string 的 incrby，一次性增加 2000 等。

```redis
127.0.0.1:6379[2]> get example_key
1000
127.0.0.1:6379[2]> incrby example_key 2000
3000
```

　　incrby 成功后，在本地机器 A 发放 1000 ~ 3000 的用户 ID。即使本地缓存丢失，就浪费一批 ID，可重新获取新的一批。

### 缓存对象信息
　　以购物车为例，使用哈希表 hash 来实现，key 格式为 cart:购物车 ID，比如 cart:3234，购物车 ID 为 3234 的哈希表。

```redis
# 在购物车 ID 为 3234 中，添加商品 ID 为 20030，数量为 5
hset cart:3234 20030 5
# 为购物车 ID 为 3234 的，增加 20030 数量 3
hincrby cart:3234 20030 3
# 购物车 ID 为 3234 的商品总数
hlen cart:3234
# 删除商品
hdel cart:3234 20030
# 获取购物车所有商品
hgetall cart:3234
```

### 布隆过滤器

### 分布式锁

```java
public booelan getLock(String lockKey, String lockValue){
    if (shardedXCommands.set(key, lockValue, 10, TimeUnit.SECONDS, false)) {
        return true;
    }
    return false;
}
```

### 排行榜
　　有序集合 Zset 实现排行榜，消息排序（根据消息权重区分重要消息、普通消息来排序）等。<br />
　　比如统计主播的收益排行榜，主播 id 作为 member，当天打赏的活动礼物对应的热度值作为 score, 通过 zrangebyscore 获取主播活动日榜。

### 发布和订阅，秒杀
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

### 身边的人
　　Geospatial 属于 redis 的特殊数据类型，**以给定经纬度为中心，找出某一半径内的元素。** <br />
　　GEO 底层实现原理是 Zset，使用 Zset 命令来操作。

- geoadd china:city 116.40 39.90 beijing 114.05 22.52 shengzhen，添加，需要经纬度；
- geopos china:city beijing，获取指定 key 的经纬度；
- geodist china:city beijing shengzhen km，获取两地的距离；
- georadius weixin:person 115 20 10km count 100，查询经纬度 115、20 为中心，寻找 10km 以内的人，筛选获取 100 个。

### 统计 UV
　　Hyperloglog 属于 redis 的特殊数据类型，作为基数统计算法，**可用于估计网页的 UV。** <br />
　　对比使用 set 来保存用户 ID，Hyperloglog 优点是占用内存固定，只要 12 kb，缺点是为估计，只能计数，且有 0.81% 错误率。<br />
　　如果需要精确统计，并获得每个人的 ID，使用 set。

### Bitmap
　　只有两个状态 0 和 1，非常省内存。可用为统计用户信息，比如签到、是否登录等。

- setbit 标识:日期 uid 1，比如 uid 为 546 的用户在 2020.03.15 签到，则为 setbit SignIn:20200315 546 1；
- getbit 标识:日期 uid，获取某位用户是否签到，比如 setbit SignIn:20200315 546；

　　还可以用于统计阅读量，知道哪个用户阅读该篇文章。将用户 ID（可能为字符串、不连续整数） 映射到一张连续的整数表中。<br />
　　为一篇文章创建一个 bitmap，当用户阅读了该篇文章，则 bitmap 对应位置赋值为 1。缺点是如果一篇文章只有几个人阅读，则该 bitmap 会有大量 0，很浪费空间，使用咆哮位图来解决。

### reference

- [Redis 缓存性能实践及总结](https://mp.weixin.qq.com/s/bsAw0VKhP_SYngvKMoByAQ)
- [Redis 精确去重计数 —— 咆哮位图](https://zhuanlan.zhihu.com/p/68019716)