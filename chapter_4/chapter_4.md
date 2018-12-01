
## 字典
　　字典被广泛用于实现 Redis 的各种功能，Redis 的数据库就是使用字典来作为底层实现的，对数据库的增、删、查、改操作也是构建在对字典的操作之上的。字典也是哈希键的底层实现之一，当一个哈希键包含的键值对比较多，又或者键值对中的元素都是比较长的字符串时，Redis 就会使用字典作为哈希键的底层实现。如下，在数据库中创建一个键为 "msg"，值为 "hello world" 的键值对，保存在数据库的字典里；

```redis
redis> SET msg "hello world"
OK
```

### 字典的实现
　　Redis 的字典使用哈希表作为底层实现，一个哈希表里面可以有多个哈希表节点，而每个哈希表节点就保存了字典中的一个键值对。
  
```c
typedef struct dictht {
    // 哈希表数组
    dictEntry **table;
    // 哈希表大小
    unsigned long size;
    // 哈希表大小掩码，用于计算索引值，总是等于 size-1
    unsigned long sizemask;
    // 该哈希表已有节点的数量
    unsigned long used;
} dictht;
```

　　table 属性是一个数组，数组中的每个元素都是一个指向 dict.h / dictEntry 结构的指针，每个 dictEntry 结构保存着一个键值对。

- size 属性，记录哈希表的大小，即 table 数组的大小；
- used 属性，记录哈希表目前已有节点（键值对）的数量；
- sizemask 属性，总是等于 size-1，该属性和哈希值决定一个键应该被放到 table 数组的哪个索引上面；
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/redis_notebook/master/chapter_4/chapter_4_p1.png)

#### 哈希表节点
　　哈希表节点使用 dictEntry 结构表示，每个 dictEntry 结构都保存着一个键值对：

```c
typedef struct dictEntry {
    // 键
    void *key;
    // 值，可为一个指针，或一个uint64_t 整数，或一个 int64_t 整数
    union {
        void *val;
        uint64_tu64;
        int64_ts64;
    } v;
    
    // 指向下个哈希表节点，形成链表，解决哈希冲突问题
    struct dictEntry *next;
} dictEntry;
```

　　如下，当有两个哈希键相同时，则使用 next 指针，将两个索引值相同 的键 k1 和 k0 连接在一起。
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/redis_notebook/master/chapter_4/chapter_4_p2.png)

#### 字典
　　Redis 中的字典由 dict.h / dict 结构表示：

```c
typedef struct dict {
    // 类型特定函数
    dictType *type;
    // 私有数据
    void *privdata;
    // 哈希表
    dictht ht[1];
    // rehash 索引，当 rehash 不在进行时，值为 -1
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */
} dict;
```

- type 属性和 privdata 属性时针对不同类型的键值对，为创建多态字典而设置的。type 属性是一个指向 dictType 结构的指针，每个 dictType 结构保存了一簇用于操作特定类型键值对的函数，Redis 会为用途不同的字典设置不同的类型特定函数。而 privdata 属性则保存了需要传给那些类型特定函数的可选参数；
- ht 属性是一个包含两个项的数组，数组中的每个项都是一个 dictht 哈希表，一般情况下，字典只使用 ht[0] 哈希表，ht[1] 哈希表只会在对 ht[0] 哈希表进行 rehash 时使用；
- rehashidx，记录了 rehash 目前的进度，如没有进行 rehash，则值为 -1。

![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/redis_notebook/master/chapter_4/chapter_4_p3.png)

#### 哈希算法
　　Redis 使用 MurmurHash2 算法来计算键的哈希值。将一个新的键值对添加到字典里时，需要先根据键值对的键计算出哈希值和索引值，然后在根据索引值，将包含新建值对的哈希表节点放到哈希表数组的指定索引上面。如下，Redis 计算哈希值和索引值：

```c
// 使用字典设置的哈希函数，计算键 key 的哈希值
hash = dict -> type -> hashFunction(key);

//  使用哈希表的 sizemask 属性和哈希值，计算出索引值
// 根据情况不同，ht[x] 可以是 ht[0] 或 ht[1]
index = hash & dict -> ht[x].sizemask;

// 计算键 k0 的哈希值
hash = dict -> type -> hashFunction(k0);
// 假设计算出的哈希值为8，sizemask=3，所以键 k0 的索引值为0，即
// 包含键值对的 k0 和 v0 的节点应放到哈希表数组的索引 0 位置上
index = hash & dict -> ht[0].sizemask = 8 & 3 = 0;
```

![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/redis_notebook/master/chapter_4/chapter_4_p4.png)

#### 解决键冲突
　　当有两个或以上数量的键被分配到了哈希表数组的同一个索引上面时，称为键冲突（哈希冲突）。<br />
　　Redis 的哈希表使用链地址法来解决键冲突，即每个哈希表节点都有一个指针，被分配到同一个索引上的多个节点可以用 next 指针来构成一个单向链表，解决键冲突问题。最坏情况下，是所有键都在同一个链表上，这样复杂度为 O(N)。<br />
　　因为 dictEntry 节点组成的链表没有指向链表表尾的指针，考虑到速度，所以将新节点添加到链表的表头位置（复杂度为O(1)）。如下，新节点  k2 通过哈希算法被分配到索引 2 上，与 k1 冲突，通过 next 指针构成一个链表，k2 节点被放到节点头。
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/redis_notebook/master/chapter_4/chapter_4_p5.png)

### rehash
　　为了让哈希表的负载因子保持在一个合理范围，当哈希表保存的键值对数量太多或太少时，需要对其进行扩容或收缩。扩展和收缩哈希表的工作通过 rehash（重新散列）来完成，步骤如下：
- 为字典的 ht[1] 哈希表分配空间，哈希表的空间大小取决于是要扩展还是收缩，以及当前哈希表 ht[0] 当前包含的键值对数量（即 ht[0].used 属性的值）。如果执行的是扩展操作，那么 ht[1] 的大小为第一个大于等于 ht[0].used * 2 的 2^n（2 的 n 次方幂）。如果是收缩操作，ht[1] 的大小为第一个大于等于 ht[0].used 的 2^n；
- 将保存在 ht[0] 中的所有键值对 rehash 到 ht[1] 上面，rehash 指的是重新计算键的哈希值和索引值，然后将键值对放到 ht[1] 哈希表的指定位置上；
- 当 ht[0] 包含的所有键值对都迁移到了 ht[1] 之后，ht[0] 变为空表，释放 ht[0]，将 ht[1] 设置为 ht[0]，并在 ht[1] 新创建一个空白哈希表，为下一次 rehash 做准备。

　　举例，对 ht[0] 进行扩展操作：
- ht[0].used 当前的值为 4，4 * 2 = 8，而8（2^3）恰好是第一个大于等于 4 的 2 的 n 次方，所以将 ht[1] 哈希表的大小设置为 8，如 ht[0].used = 5，则 5 * 2 =10，第一个大于等于 10 的数为 2^4=16，如下 ht[1].size = 8，sizemask 的值从 0 到 7；

![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/redis_notebook/master/chapter_4/chapter_4_p6.png)

- 将 ht[0] 包含的四个键值对都 rehash 到 ht[1]；

![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/redis_notebook/master/chapter_4/chapter_4_p7.png)

- 释放 ht[0]，并将 ht[1] 设置为 ht[0]，然后为 ht[1] 分配一个空白哈希表；

![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/redis_notebook/master/chapter_4/chapter_4_p8.png)

#### 哈希表的扩展与收缩
　　满足下面任何一个条件，即自动对哈希表进行扩展，服务器执行扩展操作所需的负载因子根据  BGSAVE 命令或 BGREWRITEAOF 命令是否正在执行而不同。<br />
　　因为在执行 BGSAVE 命令或 BGREWRITEAOF 命令的过程中，Redis 需要创建当前服务器进程的子进程，而大多数操作系统都采用写时复制技术来优化子进程的使用效率。所以在子进程存在期间，服务器会提高执行扩展操作所需的负载因子，尽量避免在子进程存在期间进行哈希表扩展操作，避免不必要的内存写入操作，最大限度节约内存：
- 服务器目前没执行 BGSAVE 命令或 BGREWRITEAOF 命令，并且哈希表的负载因子大于等于 1；
- 服务器正在执行 BGSAVE 命令或 BGREWRITEAOF 命令，并且哈希表的负载因子大于等于 5；

　　哈希表的负载因子计算公式，比如哈希表的大小为 4，包含四个键值对，则其负载因子为 4 / 4 = 1。而当哈希表的负载因子小于 0.1 时，自动对哈希表执行收缩操作。

```c
// 负载因子 = 哈希表已保存节点数量 / 哈希表大小
load_factor = ht[0].used / ht[0].size
```

### 渐进式 rehash
　　扩展或收缩哈希表需要将 ht[0] 里面的所有键值对 rehash 到 ht[1] 里面，但 rehash 是分多次、渐进式完成的，**因为一次性完成的话，如哈希表的数量非常大，则将全部进行 rehash 的计算量可能会导致服务器在一段时间内停止服务，**所以分成多次将 ht[0] 里面的键值对慢慢 rehash 到 ht[1]：
- 为 ht[1] 分配空间，让字典同时拥有 ht[0] 和 ht[1] 两个哈希表；
- 在字典中维持一个索引计数器变量 rehashidx，并将它的值设置为 0，表示开始 rehash；
- 在 rehash 进行期间，每次对字典执行添加、删除、查找或更新操作时，程序除了执行指定的操作外，还会将 ht[0] 哈希表在 rehashidx 索引上的所有键值对 rehash 到 ht[1]，当 rehash 完成后，rehashidx 属性值加一。比如查找 ht[0] 的索引 1 上的键值对时，将 ht[0] 的索引 1 上的所有键值对 rehash 到 ht[1] 上；
- 随着字典操作的不断执行，最终 ht[0] 的所有键值对都被 rehash 到 ht[1] 上，这时将 rehashidx 设为 -1，表示完成 rehash；
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/redis_notebook/master/chapter_4/chapter_4_p9.png)

　　**渐进式 rehash 在于采取分而治之方法，将 rehash 键值对所需的计算工作均摊到对字典的每个添加、删除、查找和更新操作上，避免一次 rehash 带来的巨大计算量。**<br />
　　在进行渐进式 rehash 的过程中，字典会同时使用 ht[0] 和 ht[1] 两个哈希表。比如，要在字典里查找一个键的话，会现在 ht[0] 进行查找，没找到则到 ht[1] 继续查找，其它操作也类似。另外，在渐进式 rehash 执行期间，新添加到字典的键值对都保存到 ht[1]，保证 ht[0] 包含的键值对只减不增。
