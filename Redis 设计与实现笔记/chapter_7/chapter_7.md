
## 压缩列表
　　如果一个列表只包含少量列表项，并且每个列表项要么是小整数值，要么是长度比较短的字符串，那么 Redis 会使用压缩列表来做列表键的底层实现。同理哈希键也是，只包含少量键值对，并且键和值是小整数值或长度比较短的字符串。压缩列表是列表键和哈希键的底层实现之一。<br />

### 压缩列表的构成
　　压缩列表是 Redis 为了节约内存开发的，由一系列特殊编码的连续内存块组成的顺序型数据结构。一个压缩列表包含任意多个节点，每个节点保存一个字节数组或一个整数值。<br />
　　如下为包含三个节点的压缩列表：
- zlbytes 属性值为 0x50（十进制为 80），表示压缩列表的总长为 80 字节；
- zltail 属性的值为 0x3c（十进制为 60），表示如果有一个指向压缩列表起始地址的指针 p，用该指针 p 加上偏移量 60，就能计算出表尾节点 entry3 的地址；
- zllen 属性的值为 0x3（十进制为 3），表示压缩列表包含三个节点；
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/redis_notebook/master/chapter_7/chapter_7_p1.png)

### 压缩列表节点的构成
　　由 previous_entry_length、encoding、content 三个部分组成。

#### previous_entry_length

　　节点的 previous_entry_length 属性以字节为单位，记录压缩列表中前一个节点的长度，previous_entry_length 可以是 1 字节或 5 字节。如果前一节点的长度小于 254 字节，则 previous_entry_length 属性的长度为 1 字节，前一节点的长度保存在该字节里。如果大于 254 字节，则 previous_entry_length 为 5 字节，该属性第一字节设置为 0xFE（十进制 254），后面四个字节保存前一节点的长度。<br />
　　如下，0xFE 表示是一个五字节长的 previous_entry_length 属性，而后面四个字节 0x00002766（十进制为 10086）是前一节点的实际长度。

![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/redis_notebook/master/chapter_7/chapter_7_p2.png)

　　因为节点的 previous_entry_length 属性记录了前一个节点的长度，所以根据当前节点的起始地址可计算出前一个节点的起始地址。比如，有一个指向当前节点起始地址的指针 c，用当前指针 c 减去当前节点 previous_entry_length 的值，就能得出指向前一个节点起始地址的指针 p，压缩列表的从表尾向表头遍历操作就是使用这一原理实现的。通过某个节点起始地址的指针，根据这个指针以及这个节点的 previous_entry_length 值，就能找到前一个指针，以此类推。
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/redis_notebook/master/chapter_7/chapter_7_p3.png)

#### content 和 encoding
　　节点的 content 保存节点的值，可以是一个字节数组或整数，值的的类型和长度由节点的 encoding 属性决定。如下，encoding 前两位 00 表示节点保存的是一个字节数组，后六位 001011 表示字节数组长度为 11。
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/redis_notebook/master/chapter_7/chapter_7_p4.png)

### 连锁更新
　　前面提到，每个节点的 previous_entry_length 属性记录前一个节点的长度，小于 254 字节，则使用一字节长的空间来保存长度值，否则，使用 5 字节长来保存长度值。<br />
　　有这样一种情况，当一个压缩列表保存的节点都是在 250 字节到 253 字节之间，每个节点只需使用一个字节的 previous_entry_length 来记录。这时，如果在表头添加新节点，且该节点长度大于 254 字节，则 e1 的 previous_entry_length 为一字节没法保存 大于 254 字节的，所以需要对压缩列表进行重新分配空间，将 e1 节点的 previous_entry_length 扩展到 5 字节。这样 e1 本身的长度在 250 字节到 253 字节之间，扩展就变成 254 到 257 字节之间，于是 e2 的 previous_entry_length 也扩展到 5 字节，以此类推。<br />
　　这种连续多次空间扩展操作即为 "连锁更新"，删除节点也有可能导致。因为连锁更新在最坏情况下需要对压缩列表执行 N 词空间重新分配操作，而每次空间分配的最坏复杂度为 O(N)，所以连锁更新的最坏复杂度为 O(N^2)。
