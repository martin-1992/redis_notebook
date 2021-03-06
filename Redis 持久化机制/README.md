### 持久化
　　将内存的数据保存到磁盘中，防止机器意外断电、重启导致数据丢失。Redis 有两种持久化方法：

- [RDB 持久化](https://github.com/martin-1992/redis_notebook/blob/master/Redis%20%E6%8C%81%E4%B9%85%E5%8C%96%E6%9C%BA%E5%88%B6/RDB%20%E6%8C%81%E4%B9%85%E5%8C%96.md)，快照形式，全量备份。生成一个压缩的二进制文件 RDB 文件，保存的是数据库的键值对数据；
- [AOF 持久化](https://github.com/martin-1992/redis_notebook/blob/master/Redis%20%E6%8C%81%E4%B9%85%E5%8C%96%E6%9C%BA%E5%88%B6/AOF%20%E6%8C%81%E4%B9%85%E5%8C%96.md)，增量备份。每隔一段时间（比如每隔一秒）保存 Redis 服务器所执行的写命令。

### 混合持久化

- 使用 RDB 做冷备，会丢失大量数据；
- 使用 AOF，则启动要花很长时间。

　　混合持久化，同时使用 RDB 和 AOF，先加载 RDB 的内容，剩余丢失部分在用 AOF 文件加载。AOF 一般每隔一秒同步磁盘一次，最多丢失一秒的数据。想要不丢失，则使用消息队列。
