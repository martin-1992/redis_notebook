### 事务
　　提供一种将多个命令请求打包，然后一次性、按顺序地执行多个命令的机制。在事务执行期间，服务器会事务中的所有命令都执行完。**即使一条命令失败了，也会继续执行下去，直到所有命令都执行完，所以 Redis 事务是非原子性的。** <br />
　　有三个命令，MULTI 表示事务开始、EXEC 表示事务执行、DISCARD 表示丢弃事务队列中的所有命令，Redis 不支持回滚操作。

```redis
redis> MULTI
OK

redis> SET book-name "Mastering C++ in 21 days"
QUEUED

redis> GET book-name
QUEUED

redis> SADD tag "C++" "Programming" "Mastering Series"
QUEUED

redis> SMEMBERS tag
QUEUED

redis> EXEC
1) OK
2) "Mastering C++ in 21 days"
3) (integer) 3
4) 1) "Mastering Series"
   2) "C++"
   3) "Programming"
```

　　以上面为例，分成三个阶段。

- 输入 MULTI，客户端从非事务状态，切换为事务状态；
- 输入多个命令，这些命令会缓存到服务器的一个事务队列中；
- 输入 EXEC，执行整个事务队列。完成后，一次性返回所有指令的执行结果。

### Redis 乐观锁
　　WATCH 命令是一个乐观锁，原理为 CAS，检查被监视的键是否有修改过，是则拒绝执行事务。

### pipline
　　使用 EXEC 执行多条命令时，客户端是每条指令发送一次，浪费网络资源。通过 pipline，可将多条指令一次发送，多次 IO 减少为单次 IO 操作。

