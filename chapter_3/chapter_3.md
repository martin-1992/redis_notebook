## 链表
　　Redis 的链表为双向链表，被广泛用于实现 Redis 的各种功能，比如列表键、发布与订阅、慢查询、监视器等。

```redis
reids> LLEN integers
(integer) 1024

redis> LRANGE integers 0 3
1)"1"
2)"2"
3)"3"
```
  
### 链表节点
　　每个链表节点由一个 listNode 结构表示，每个节点都有一个指向前置节点和后置节点的指针。

![avatar](chapter_3_p1.png)

```c
typedef struct listNode {
    // 前置节点
    struct listNode * prev;
    // 后置节点
    struct listNode * next;
    // 节点的值
    void * value;
} listNode;
```

### 链表结构
　　每个链表使用一个 list 结构来表示，这个结构带有表头节点指针 head、表尾节点指针 tail，链表长度 len，复制链表节点所保存的值函数 drup，释放链表节点所保存的值 free 函数，对比链表节点所保存的值和另一个输入值是否相等的 match 函数。

```c
typedef struct list {
    // 表头节点
    listNode * head;
    // 表尾节点
    listNode * tail;
    // 链表所包含的节点数量
    unsigned long len;
    // 节点值复制函数
    void *(*dup) (void *ptr);
    // 节点值释放函数
    void (*free) (void *ptr);
    // 节点值对比函数
    int (*match) (void *ptr, void *key);
} list;
```

![avatar](chapter_3_p2.png)

### Redis 链表特性

- 双向，链表节点带有 prev 和 next 指针，获取某个节点的前置节点和后置节点的复杂度为 O(1)；
- 无环，表头节点的 prev 指针和表尾节点的 next 指针都指向 NULL，对链表的访问以 NULL 为终点；
- 带表头指针和表尾的指针，通过 list 结构的 head 指针和 tail 指针，可获取表头节点和表尾节点，其复杂度为 O(1)；
- 带链表长度计数器，通过 list 结构的 len 属性获取链表节点数，复杂度为 O(1)；
- 多态，链表节点使用 void* 指针来保存节点值，可通过 dup、free、match 三个属性为节点值设置类型特定函数，使得 Redis 的链表可以保存各种不同类型的值；
