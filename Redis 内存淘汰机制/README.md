### 内存淘汰机制
　　当 Redis 的内存超过期望内存 maxmemory 时，会执行内存淘汰机制，有几种策略。

- **noeviction。** 不执行写请求，但读请求可以，默认策略；
- **allkeys-lru。** 从所有 key 的集合中，最近最少使用的 key 被移除；
- **volatile-lru。** 从设置过期时间的 key 集合中，移除最近最少使用的 key。没有设置过期时间的 key，则不会被移除；
- **allkeys-random。** 从所有 key 的集合中，随机移除某个 key；
- **volatile-random。** 从过期 key 集合中，随机移除某个 key；
- **volatile-ttl。** 从过期 key 集合中，移除最早时间的 key。


### 近似 LRU 算法
　　LRU 算法由哈希表和双向链表组成，当有 key 被使用时，则将 key 移动到表头，没使用的则会逐渐移动到末尾。这种需要消耗大量内存，所以 Redis 使用近似 LRU 算法。

- 每个 key 有一个额外的属性，为最后一次被访问的时间戳；
- 当内存超过 maxmemory，近似 LRU 算法会随机抽取 5（默认） 个 key，检查比较这 5 个 key 的最后一次被访问的时间戳，淘汰掉最旧的那个；
- 如果内存还超过 maxmemory，则重复前面流程；
- allkeys-lru，从所有 key 中选取。volatile-lru，从过期的 key 中选取。