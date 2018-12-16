
## 客户端
　　Redis 服务器是典型的一对多服务器程序，一个服务器可以与多个客户端建立网络连接，每个客户端可以向服务器发送命令请求，而服务器则接收并处理客户端发送的命令请求，并向客户端返回命令回复。<br />
　　通过使用由 I / O 多路复用技术实现的文件事件处理器，Redis 服务器使用单线程单进程来处理命令请求，并与多个客户端进行网络通信。对于每个与服务器进行连接的客户端，服务器都为这些客户端建立相应的 redis.h / redisClient 结构（客户端状态），这个结构保存了客户端当前的状态信息，以及执行相关功能时用的的数据结构。<br />
　　Redis 服务器状态结构的 clients 属性是一个链表，保存了所有与服务器连接的客户端的状态结构，对客户端执行批量操作，或查找某个指定的客户端，都通过遍历该链表来完成。
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/redis_notebook/master/chapter_13/chapter_13_p1.png)

### 客户端属性
　　

#### 套接字描述符
　　客户端状态的 fd 属性记录了客户端正在使用的套接字描述符，该值可以是 -1 或大于 -1 的整数：
- 伪客户端，套接字描述符为 -1。因为伪客户端处理的命令请求来源与 AOF 文件（还原数据库状态）或 Lua 脚本（执行该脚本包含的 Redis 命令），而不是网络，所以不需要套接字连接；
- 普通客户端的 fd 属性的值为大于 -1 的整数，因要使用套接字与服务器进行通信。

#### 名字
　　默认情况下，一个连接到服务器的客户端是没有名字的，使用 CLIENT name 命令可以为客户端设置一个名字：
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/redis_notebook/master/chapter_13/chapter_13_p2.png)

#### 标志
　　客户端的标志属性 flags 记录了客户端的角色，以及客户端目前所处的状态，该值可以是单个标志或多个标志的二进制。每个标志使用一个常量表示，一部分标志记录了客户端的角色：
  
- 在主从服务器进行复制操作时，主服务器会成为从服务器的客户端，而从服务器也会成为主服务器的客户端。REDIS_MASTER 标志表示主服务器，REDIS_SLAVE 标志表示从服务器；
- REDIS_PRE_PSYNC 标志，表示客户端代表的是一个版本低于 Redis 2.8 的从服务器，主服务器不能使用 PSYNC 命令与这个从服务器进行同步；
- REDIS_LUA_CLIENT 标志，表示客户端是专门处理 Lua 脚本里包含的 Redis 命令的伪客户端。

　　还有一部分标志记录客户端所处的状态，比如 REDIS_MONITOR，表示客户端在执行 MONITOR 命令，这里就不一一列出，都定义在 redis.h 文件里。

#### 输入缓冲区
　　客户端状态的输入缓冲区用于保存客户端发送的命令请求，使用 querybuf 属性保存命令请求的 SDS 值。输入缓冲区的大小会根据输入内容动态地缩小或扩大，但最大大小不超过 1GB，否则服务器将关闭这个客户端。
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/redis_notebook/master/chapter_13/chapter_13_p3.png)

#### 命令与命令参数
　　在服务器将客户端发送的命令请求保存到客户端状态的 querybuf 属性后，服务器将对命令请求的内容进行分析，并将得出的命令参数以及命令参数的个数分别保存到客户端状态的 argv 属性和 argc 属性。<br  />
　　argv 属性是一个数组，数组中的每个项都是一个字符串对象，其中 argv[0] 是要执行的命令，而之后的其它项则是传给命令的参数，argc 属性则记录 argv 数组的长度。
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/redis_notebook/master/chapter_13/chapter_13_p4.png)

#### 命令的实现参数
　　服务器从协议内容分析得出 argv 属性和 argc 属性后，根据项 argv[0] 的值，在命令列表中查找命令所对应的命令实现函数，比如 argv[0] 为 "SET"，则查找 "SET" 对应的 RedisCommand 结构，并将客户端状态 redisClinet 的 cmd 指针指向目标 redisCommand 结构，如下图：

![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/redis_notebook/master/chapter_13/chapter_13_p5.png)

#### 输出缓冲区
　　执行命令得到的命令回复先被保存在客户端状态的输出缓冲区里，有两个输出缓冲区：
  
- 固定大小的缓冲区用于保存长度比较小的回复，默认为 16KB；

![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/redis_notebook/master/chapter_13/chapter_13_p6.png)

- 可变大小的缓冲区使用链表来连接多个字符串对象，所以用于保存长度较大的回复。理论上来说，能保存任意长的命令回复，但为避免客户端的回复过大，占用过多的服务器资源，服务器会在缓冲区的大小超出限制范围时，执行限制，一种为硬性限制，即超出缓冲区大小，则关闭客户端。另一种为软性限制，输出缓冲区的大小超过软性限制的大小，但没超过硬性限制，则 obuf_soft_limit_reached_time 属性记录超过软性限制的时长，当超过一定时长，则关闭客户端。
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/redis_notebook/master/chapter_13/chapter_13_p7.png)

#### 身份验证
　　客户端状态 redisClient 的 authenticated 属性用于记录客户端是否通过身份验证，0 为未通过，1 为通过。为 0 时，只能使用 auth 命令，其余会被拒绝。

#### 时间
　　
- ctime 属性记录了创建客户端的时间，用于计算客户端与服务器的连接时长；
- lastinteraction 属性记录了最近一次客户端与服务器的互动时间，即发送命令或接收命令，用于计算客户端的空转时间；
- obuf_soft_limit_reached_time 属性记录输出缓冲区第一次到达软性限制（soft limit）的时间；

### 客户端的创建与关闭
　　如果客户端使用 connect 函数通过网络连接与服务器进行连接的普通客户端，则服务器调用连接事件处理器，为客户端创建相应的客户端状态，并将这个新的客户端状态添加到服务器状态结构 clients 链表的末尾。如下，c3 为新创建的客户端状态，添加到 clients 链表的末尾：
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/redis_notebook/master/chapter_13/chapter_13_p8.png)

#### 关闭普通客户端
　　普通客户端被关闭原因：
- 客户端进程退出或被杀死；
- 客户端向服务器发送带有不符合协议格式的命令请求；
- 客户端成为了 CLIENT KILL 命令的目标；
- 客户端的空转时间超过了 timeout 选项设置的值时；
- 客户端发送的命令请求超过了输入缓冲区的限制大小（默认为 1GB）；
- 要发送给客户端的命令回复的大小超过了输出缓冲区的限制大小。

#### 伪客户端

- 处理 Lua 脚本的伪客户端在服务器初始化时创建，这个客户端会一直存在，直到服务器关闭；
- 载入 AOF 文件时使用的伪客户端在载入工作时开始时动态创建，载入工作完毕后关闭。
