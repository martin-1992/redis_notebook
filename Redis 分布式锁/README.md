### 分布式锁条件

- 互斥性。任意时刻，只有一个客户端持有锁；
- 不会出现死锁。客户端崩溃，导致一直持有锁无法释放，其他客户端无法获得锁。通过超时解决死锁问题；
- 高可用。部分 Redis 节点奔溃，仍可正常运行；
- 加锁和解锁需为同一个客户端。

### 单机的加锁代码
　　最简单的通过 set 命令可实现加锁和超时设置，但这存在两个个问题，一是超时导致线程没执行完任务接解锁，建议不要执行太长时间的任务。二是没法校验加速和解锁是否为同一个客户端。

- 假设第一个线程的执行时间超过超时时间，第一个线程没执行完，但锁已经释放；
- 第二个线程获取锁，执行任务；
- 然后第一个线程执行完，会释放锁，这时释放的是第二个线程锁，因为没法验加速和解锁是否为同一个客户端；
- 于是，第三个线程在第二个线程还没执行完任务时，就获得了锁。

　　为解决这个问题，需要在 set 同时传进一个随机数，解锁时先比较随机数是否一样。比如第一线程加锁生成随机数 53，超时仍在执行。第二线程获得锁，生成随机数 78，这时第一线程执行完解锁，比较随机数不一致，于是第一线程就没法解除第二线程的锁。

```java
public class RedisLock {

    private static final String LOCK_SUCCESS = "OK";
    private static final String SET_IF_NOT_EXIST = "NX";
    private static final String SET_WITH_EXPIRE_TIME = "PX";

    /**
     * 尝试获取分布式锁
     * @param jedis Redis 客户端
     * @param lockKey 锁
     * @param requestId 请求标识
     * @param expireTime 过期时间
     * @return 是否获取成功
     */
    public static boolean tryGetDistributedLock(Jedis jedis, String lockKey, String requestId, int expireTime) {
        // 加锁，原子性
        String result = jedis.set(lockKey, requestId, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, expireTime);
		// 不可重入。如果加锁执行成功，但返回失败，这次在执行加锁会失败，只能等超时解锁
        if (LOCK_SUCCESS.equals(result)) {
            return true;
        }
        return false;
    }
}
```

- key 为锁，requestId 为随机数，用于解锁时判断是否同一个客户端；
- SET_IF_NOT_EXIST 表示 key 不存在时，进行 set 操作，存在则不进行任何操作，只有一个客户端能持有锁，满足互斥性；
- SET_WITH_EXPIRE_TIME，expireTime。加上过期时间，时间为 expireTime，解决死锁问题；

#### 解锁代码
　　通过 Lua 脚本来获取锁对应的 requestId 值，然后判断是否与当前客户端的 requestId 相等，是则解锁。因为获取锁和判断 requestId 相等，是两条命令，非原子性。所以使用 Lua 脚本确保原子性。

```java
public class RedisTool {

    private static final Long RELEASE_SUCCESS = 1L;

    /**
     * 释放分布式锁
     * @param jedis Redis客户端
     * @param lockKey 锁
     * @param requestId 请求标识
     * @return 是否释放成功
     */
    public static boolean releaseDistributedLock(Jedis jedis, String lockKey, String requestId) {
        
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
		// eval 执行 lua 代码
        Object result = jedis.eval(script, Collections.singletonList(lockKey), Collections.singletonList(requestId));

        if (RELEASE_SUCCESS.equals(result)) {
            return true;
        }
        return false;

    }
}
```

### 集群的分布式锁
　　集群中，Redis 的主节点挂了，会由从节点替代，继续运行。如果使用单机的分布式锁，则主节点挂了后，锁还在。从节点由于没有锁，所以从节点能加锁。这时会出现两把锁，即两个客户端拥有两把锁。<br />
　　集群的分布式锁有以下几种：

- MySQL，适用数据库的唯一索引，分为悲观锁和乐观锁，不适用高分布式环境；
    1. 悲观锁，比如创建一张表，要获取锁时，先去这张表获取，类似行锁；
    2. 乐观锁，在表中加一个版本号字段，使用 CAS 方式更新。
- ZK 分布式锁，根据临时顺序节点的特性来创建分布式锁，性能较差，不适用高分布式的环境；
- Redis 分布式锁；
    1. 使用 jedis 客户端，自己实现分布式锁；
    2. Redission 封装了分布式锁的实现，可直接调用；
    3. RedLock，解决主从的锁同步问题，比如机器 A 向主机器申请锁，这锁没有同步到 B 机器，然后主机器宕机了。这时 B 机器去从机器能申请到锁。RedLock 解决方法是使用少数服从多数原则，机器 A 申请锁时，是向多个 redis 机器申请锁，当获得多数锁时，才成功获取锁，适用于更严格的场景。
- 自研分布式锁，如谷歌的 Chubby。

```java
public class RedisTool {

    private static final String LOCK_SUCCESS = "OK";

    private static final String SET_IF_NOT_EXIST = "NX";

    private static final String SET_WITH_EXPIRE_TIME = "PX";

    /**
     * 尝试获取分布式锁
     * @param jedis Redis 客户端
     * @param lockKey 锁
     * @param requestId 请求标识
     * @param expireTime 过期时间
     * @return 是否获取成功
     */
    public static boolean tryGetDistributedLock(Jedis jedis, String lockKey, String requestId, int expireTime) {
        String result = jedis.set(lockKey, requestId, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, expireTime);
        if (LOCK_SUCCESS.equals(result)) {
            return true;
        }
        return false;
    }
}
```

### reference

- [Redis 分布式锁的正确实现方式（Java 版）](https://mp.weixin.qq.com/s/qJK61ew0kCExvXrqb7-RSg)
- [漫画：什么是分布式锁](https://mp.weixin.qq.com/s/8fdBKAyHZrfHmSajXT_dnA)
- [搞懂“分布式锁”，看这篇文章就对了](https://mp.weixin.qq.com/s/hoZB0wdwXfG3ECKlzjtPdw)
- [基于 Redis 的分布式锁](https://crossoverjie.top/2018/03/29/distributed-lock/distributed-lock-redis/)