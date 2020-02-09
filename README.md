# redis 设计与实现笔记：

### [Redis 持久化机制](https://github.com/martin-1992/redis_notebook/tree/master/Redis%20%E6%8C%81%E4%B9%85%E5%8C%96%E6%9C%BA%E5%88%B6)
　　将内存的数据保存到磁盘中，防止机器意外断电、重启导致数据丢失。Redis 有两种持久化方法：

- RDB 持久化，快照形式，全量备份。生成一个压缩的二进制文件 RDB 文件，保存的是数据库的键值对数据；
- AOF 持久化，增量备份。每隔一段时间（比如每隔一秒）保存 Redis 服务器所执行的写命令。

### [Redis 过期机制](https://github.com/martin-1992/redis_notebook/tree/master/Redis%20%E8%BF%87%E6%9C%9F%E6%9C%BA%E5%88%B6)
　　Redis 的过期策略为定期删除 + 惰性删除。

- 定时删除（主动删除），对内存友好，但对 CPU 不友好。 在 CPU 使用高峰时，执行定时删除，会影响其性能。同时，创建定时器用到 Redis 服务器的时间事件，它是使用无序链表，导致查找一个事件的时间复杂度为 O(N)；
- 惰性删除（被动删除），对 CPU 时间最友好，但对内存最不友好。 获取键时，检查是否过期，过期则删除。如果有很多键没有访问获取（数据太旧），则没法删除，相当于内存泄漏；
- 定期删除（主动删除），前两种的折中。 设定删除操作执行的时长，每隔一段时间，检查一次并删除。

### [Redis 内存淘汰机制](https://github.com/martin-1992/redis_notebook/tree/master/Redis%20%E5%86%85%E5%AD%98%E6%B7%98%E6%B1%B0%E6%9C%BA%E5%88%B6)
　　当 Redis 的内存超过期望内存 maxmemory 时，会执行内存淘汰机制。Redis 的过期机制有可能会导致内存泄漏，需要对 key 执行淘汰策略。

### [布隆过滤器](https://github.com/martin-1992/redis_notebook/tree/master/%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8)
　　内存占用小，解决缓存穿透的方法。
