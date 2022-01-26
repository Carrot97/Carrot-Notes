# Redis详解

## 一、数据结构

### 1. 简单动态字符串SDS

Redis使用SDS代替C的字符串

#### 定义

```c
struct sdshdr {
    // 记录buf数组已使用的字节数量
	int len;
    // 记录buf数组未使用的字节数量
    int free;
    // 字节数组，用于保存字符串
    char buf[];
}
```

需要注意，字节数组buf在字符穿的多末尾加了'/0'来兼容C语言字符串的方法，因此buf的长度=len+free+1。

#### 特点

1. **常数复杂度获取字符串长度**

2. **杜绝缓冲区溢出**

   > SDS在修改字符串时会首先判断现有空间是否满足修改所需

3. **减少修改字符串时带来的内存重新分配次数**

   > C语言中每次对字符串的需改都将引入以此内存重新分配操作，如果忘记将会出现两种问题。
   >
   > 1. 空间不足时append——内存溢出
   > 2. trim后未释放空间——内存泄露
   >
   > 因此，redis采用空间预分配和惰性空间释放解决该问题。

   - 空间预分配

     > 修改后的长度小于1MB：将free和len设置为一样
     >
     > 修改后的长度大于等于1MB：直接分配1MB的free空间

   - 惰性空间释放

     > 字符串trim后不直接进行空间释放，而是将多余空间记录在free字段中。

4. **二进制安全**

   > 通过len来读取字符串，而不是通过结尾的空字符'/0'，因此内容可以保存空字符。

5. **兼容C语言字符串方法**

### 2. 链表

客户端状态、不定长缓冲区

#### 定义

listNode

```C
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
}
```

list

```C
typedef struct list {
    listNode *head;
    listNode *tail;
    unsigned long len;
    // 节点值复制函数
    void *(*dup) (void *ptr);
    // 节点值释放函数
    void *(*free) (void *ptr);
    // 节点值对比函数
    int (*match) (void *ptr, void *key);
}
```

### 3. 字典

#### 定义

字典

```C
typedef struct dict {
	// 类型特点函数
    dictType *type;
    // 私有数据
    void *privdata;
    // 哈希表
    dictht ht[2];
    // rehash索引，不进行rehash时为-1
    int rehashidx;
}
```

哈希表

```C
typedef struct dictht {
    dictEntry **table;
    // 哈希表的大小，底层数组大小
    unsigned long size;
    // 计算哈希表大小的掩码，用于计算索引值，等于size-1
    unsigned long sizemask;
    // 哈希表已有节点的数量，kv数量
    unsigned long used;
}
```

节点

```C
typedef struct dictEntry {
    // 键
    void *key;
    // 值
    union {
        void *val;
        unint64_t u64;
        int64_t s64;
    } V;
    // 指向下一个节点的指针
    struct dictEntry *next;
}
```

#### 特点

1. **哈希算法**

   > 先计算哈希值在计算索引值，哈希算法为murmurHash2。
   >
   > ```C
   > hash = dict->type->hashFunction(key);
   > index = hash & dict->[x].sizemask;
   > ```

2. **使用拉链法解决键冲突**

3. **rehash**

   >流程：
   >
   >1. 为ht[1]分配空间，分配规则如下：
   >   - 扩张操作：第一个大于等于ht[0].used*2的2的n次幂；
   >   - 收缩操作：第一个大于等于ht[0].used的2的n次幂。
   >
   >2. 所有ht[0]中的键值对移动至ht[1];
   >3. ht[1]赋值给ht[0]，ht[1]指向一个新的空白哈希表对象。
   >
   >时机：
   >
   >- 扩张：未执行持久化操作（BGSAVE和BGREWRITEAOF）时大于等于1，执行持久化操作时大于等于5；
   >- 收缩：负载因子小于0.1。
   >
   >渐进式：
   >
   >- 每个增删改查操作会将ht[0].table[rehashidx]上的键值对rehash至ht[1]；
   >- rehash时，所有的读操作将在ht[0]和ht[1]上先后进行，所有的写操作只在ht[1]上进行。

### 4. 跳跃表

#### 定义

跳跃表

```C
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    // 表中层数量最大的节点层数
    int level;
}
```

跳跃表节点

```C
typedef struct zskiplistNode {
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度
        unsigned int span;
    } level[];
    // 后退指针
    struct zskiplistNode *backward;
    // 分值
    double score;
    // 成员对象
    robj *obj;
}
```

#### 特点

1. 通过计算跨度和即可得到元素的排名；
2. 每个节点的层高都是1至32之间的随机数；

### 5. 整数集合

#### 定义

```C
typedef struct intset {
    // 编码方式
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
}
```

#### 特点

1. 升级

   > 提高灵活性：一种数据结构可以表达三种数据格式；
   >
   > 节约空间：只有在需要的时候才提升表达需要的空间。

2. 不支持降级操作

### 6. 压缩列表

#### 定义

压缩列表

```C
typedef struct ziplist {
	// 记录整个ziplist占用的字节数
    uint32_t zlbytes;
    // 记录ziplist为节点头的偏移量
    uint32_t zltail;
    uint16_t zllen;
    // ziplist节点
    entryX;
    // 尾部标记值
    uint8_t zlend;
}
```

节点

1. previous_entry_length

​	前一个节点的长度，长度可以为1字节或5字节：

- 1字节：前一个节点长度小于254字节；
- 5字节：前一个节点长度大于等于254字节，第一个字节存储254，后四个字节存储真正的长度。

2. encoding

- 字节数组：1、2和5字节表示，最高位分别为00、01、10；
- 整数：1字节表示，11开头。

3. content

#### 特点

连锁更新：

添加新节点导致后面节点的previous_entry_length存不下其长度。

### 7. 对象

#### 基本定义

```C
typedef struct redisObject {
    // 类型
	unsigned type;
    // 编码
	unsigned encoding;
	void *ptr;
    // 通过引用计数算法来实现GC
    int refcount;
    // 上一次被访问时间
    unsigned lru;
}
```

1. 类型

   > REDIS_STRING：字符串对象
   >
   > REDIS_LIST：列表对象
   >
   > REDIS_HASH：哈希对象
   >
   > REDIS_SET：集合对象
   >
   > REDIS_ZSET：有序集合对象

2. 编码

   > REDIS_ENCODING_INT：long类型的整数
   >
   > REDIS_ENCODING_EMBSTR：embstr编码的简单动态字符串
   >
   > REDIS_ENCODING_RAW：简单动态字符串
   >
   > REDIS_ENCODING_HT：字典
   >
   > REDIS_ENCODING_LINKEDLIST：双端链表
   >
   > REDIS_ENCODING_ZIPLIST：压缩链表
   >
   > REDIS_ENCODING_INTSET：整数集合
   >
   > REDIS_ENCODING_SKIPLIST：跳表或字典

3. 类型和编码的对应关系

   > REDIS_STRING：
   >
   > 1. REDIS_ENCODING_INT
   > 2. REDIS_ENCODING_EMBSTR
   > 3. REDIS_ENCODING_RAW
   >
   > REDIS_LIST：
   >
   > 1. REDIS_ENCODING_ZIPLIST
   > 2. REDIS_ENCODING_LINKEDLIST
   >
   > REDIS_HASH：
   >
   > 1. REDIS_ENCODING_ZIPLIST
   > 2. REDIS_ENCODING_HT
   >
   > REDIS_SET：
   >
   > 1. REDIS_ENCODING_INTSET
   > 2. REDIS_ENCODING_HT
   >
   > REDIS_ZSET：
   >
   > 1. REDIS_ENCODING_ZIPLIST
   > 2. REDIS_ENCODING_SKIPLIST和REDIS_ENCODING_HT

4. refcount 

- 内存回收：redis采用引用计数算法实现内存回收；
- 对象共享：字符串实现引用共享节约内存空间，另外在redis启动时会直接创建0-9999整数的sds。

4. lru

#### 字符串对象

> REDIS_ENCODING_INT：存储可以用long类型表示的整数值（过长的整型数或浮点数转为字符串）；
>
> REDIS_ENCODING_EMBSTR：存储长度小于等于32字节的字符串，obj和sds一起分配一段连续内存，节省分配和回收内存的时间；
>
> REDIS_ENCODING_RAW：存储长度大于32字节的字符串。

#### 列表对象

> 满足以下连个条件时使用ziplist编码：
>
> - 元素长度均小于64字节；
> - 元素数量小于512个。

#### 哈希对象

> 满足以下连个条件时使用ziplist编码：
>
> - kv中的键和值长度均小于64字节；
> - kv数量小于512个。

#### 集合对象

> 满足以下连个条件时使用intset编码：
>
> - 元素均为整数值；
> - 元素数量小于512个。

#### 有序集合对象

> 1. 满足以下连个条件时使用ziplist编码：
>    - 元素数量小于128个；
>    - 所有元素长度小于64字节。
> 2. 同时使用字典和跳表实现zset，通过字典实现O(1)的查找，通过跳表实现O(N)的range和rank。



## 二、单机数据库

### 1. 数据库

#### 定义

db结构

```C
typedef struct redisDb {
    // 采用字典构建的键空间
    dict *dict;
    // 过期字典
    dict *expires;
}
```

#### 重要操作

1. 读写时键空间的维护操作

   > - 读取键后（写操作也要先读）服务器对是否命中进行统计；
   > - 读取键后修改键的lru字段；
   > - 读取键前先判断是否过期，若过期则先删除；
   > - 若键被watch则对其修改后标记为dirty；
   > - 每次修改都会对dirty键计数器+1（触发rdb）；
   > - 若开启数据库通知功能，则在修改后进行相应的通知。

2. 过期时间

   > 设置命令的转换关系：EXPIRE -> PEXPIRE -> **PEXPIREAT** <- EXPIREAT；
   >
   > 保存方式：采用单独的过期字典来保存，过期字典的键与键空间的键共享。

3. 过期删除策略

   > 惰性删除：
   >
   > - 特点：对CPU友好，但容易造成内存泄露；
   > - 实施：在取出键时进行过期检查。
   >
   > 定期删除：
   >
   > - 特点：对CPU和内存都相对友好，不易确定定期时常；
   >
   > - 实施：定期对键进行随机检查。
   >
   >   ```python
   >   DEFAULT_DB_NUMBERS = 16
   >   DEFAULT_KEY_NUMBERS = 20
   >   current_db = 0
   >             
   >   def activeExpireCycle():
   >   	db_numbers = min(server.dbnum, DEFAULT_DB_NUMBERS)
   >   	for i in range(db_numbers):
   >           # 对db进行循环操作
   >           if current_db == db_nums: current_db = 0
   >           redisDb = server.db[current_db]
   >           current_db += 1
   >           for j in range(DEFAULT_KEY_NUMBERS):
   >               if redisDb.expires.size() == 0: break
   >               key_with_ttl = redisDb.expires.get_random_key()
   >               if is_expired(key_with_ttl):
   >                   delete_key(key_with_ttl)
   >               if reach_time_limit(): return
   >   ```

4. RDB、AOF和复制功能对过期键的处理

   > RDB：save时不保存过期键，读取时若为主机则不读取过期键，若为从机则读取过期键；
   >
   > AOF：save时写入同时删除时写入，读取时不读；
   >
   > 复制：主机过期删除，从机过期不删除而是等待主机删除命令。**此处存在一致性问题**

### 2. RDB持久化

#### 写入

- SAVE：服务器进程执行
- BGSAVE：派生子进程执行

> 执行BGSAVE时：
>
> 1. 拒绝SAVE和BGSAVE命令；
> 2. 推迟BGREWRITEAOF命令，如果BGREWRITEAOF命令正在执行则拒绝BGSAVE命令。

#### 加载

> Redis会首先采用AOF文件还原数据库，若未开启AOF功能则采用RDB；
>
> 加载RDB文件时服务器进程一直处于阻塞状态。

#### 自动间隔性保存

> 默认：900秒1次；300秒10次；60秒10000次。

#### RDB文件结构

- RDB整体

  > |REDIS|db_version|databases|EOF|check_sum|
  >
  > EOF：377

- databases部分

  > |SELECTDB|db_number|EXPIRETIME_MS|time|REDIS_RDB_TYPE_SET|key|value|
  >
  > SELECTDB：376
  >
  > EXPIRETIME_MS：374

- 字符串

  > |ENCODING|integer|
  >
  > 无压缩：|len|string| 压缩：|REDIS_RDB_ENC_LZF|compressed_len|origin_len|compressed_string|

- 链表、集合和哈希表按顺序排
- intset、ziplist压缩乘一个字符串

### 3. AOF持久化

#### 文件类型：纯文本

#### 写入

> 服务器在执行完每条写命令后将命令追加至AOF缓冲区

#### 同步

> AOF通过时间事件来进行文件同步
>
> 三种模式
>
> - always：每个命令都同步；
>
> - everysec：每秒同步一次；
>
> - no：操作系统决定何时同步。
>
> **always性能最差，但最安全，no和everysec性能相近，但everysec更安全**

#### 加载

> 创建一个不带网络连接功能的为客户端对AOF文件中命令进行重新执行

#### AOF重写

> 服务器在AOF重写期间执行完每条写命令后将命令同时追加至AOF缓冲区和AOF重写缓冲区

### 4. 事件

Redis是一个事件驱动的应用程序

主程序

```python
def main():
	init_server()
	while server_is_not_shutdown():
		aeProcessEvents()
	clean_server()
```

主循环

```python
def aeProcessEvents():
	// 到达时间最近的时间事件
	time_event = aeSearchNearestTimer()
	remaind_ms = time_event.when - unix_ts_now()
	remaind_ms = max(0, remaind_ms)
	timeval = create_timeval_with_ms(remaind_ms)
	// 根据remaind_ms决定阻塞的时间
	aeApiPoll(timeval)
	processFileEvents()
	processTimeEvents()
```

#### 文件事件

Redis基于Reactor模式开发了网络事件处理器，采用多路复用I/O模型

> 套接字1~N -> I/O多路复用程序 -> 文件事件分派器 -> 事件处理器1~N
>
> - I/O多路复用程序通过队列向文件时间分派器传送套接字；
> - I/O多路复用程序有四种实现，按优先级排序为：evport > epoll > kqueue > select；
> - 客户端的写操作会使得套接字可读，触发服务器READABLE事件；客户端的读操作会使得套接字可写，触发服务器WRITABLE操作。
>
> 事件处理器分类：
>
> - 监听套接字关联连接应答处理器；
> - 为连接后的客户端关联命令请求处理器；
> - 执行完命令后，为客户端关联命令回复处理器；
> - 主从复制时，主从服务器关联复制处理器。

#### 时间事件

分类：定时事件、周期性事件；

定义:

1. id
2. when：毫秒级时间戳，记录时间事件的到达时间；
3. timeProc：时间事件的处理器。

​	**所有的时间事件都被存放在一个无需链表中**

流程：

```python
def processTimeEvents():
    for time_event in all_time_event():
        if time_event.when <= unix_ts_now():
            retval = time_event.timeProc()
            // 如果是定时事件
            if retval == AE_NOMORE：
            	delete_time_event_from_server(time_event)
            else:
                update_when(time_event, retval)
```

实现：serverCron函数（默认周期100毫秒）**第6节中有详细描述**

### 5. 客户端

服务器通过链表维护客户端状态

#### 定义

```C
typedef struct redisClient {
    // 指向select的db
    redisDb *db
    // 套接字描述符
    int fd;
    // 名字
    robj *name;
    // 二进制标识
    int flags;
    // 输入缓冲区(不能超过1GB)
    sds querybuf;
    // 命令参数
    robj **argv;
   	// 参数长度(注意命令本身也是参数)
    int argc;
    // 命令实现函数
    struct redisCommand *cmd;
    // 定长输出缓冲区（默认为16KB）
    char buf[REDIS_REPLY_CHUNK_BYTES];
    int bufpos;
    // 不定长缓冲区
    list *reply;
    // 身份验证标识位（只有开启密码才会使用）
    int authenticated;
    // 创建时间
    time_t ctime;
    // 最后一次互动时间
    time_t lastinteraction;
    // 缓冲区超出软性限制的时间
    time_t obuf_soft_limit_reached_time;
    // 从机的监听端口
    int slave_listening_port;
}
```

#### 何时关闭客户端

1. 进程退出或被杀死；
2. 发送了非法格式的命令；
3. 执行client kill命令；
4. 空转时间过长；
5. 输入输出缓冲区超过限制；

### 6. 服务器

#### 定义

```C
struct redisServer {
    // 大小通过dbnum来指定
    redisDb *db;
    // 默认大小为16
    int dbnum;
    // 记录RDB保存条件的数组
    struct saveparam {
        time_t seconds;
        int changes;
    } *saveparams;
    // 修改计数器
    long dirty;
    // 上一次执行RDB持久化的时间
    time_t lastsave;
    // AOF缓冲区
    sds aof_buf;
    // AOF重写缓冲区(只有在AOF重写时使用)
    sds aof_rewrite_buf;
    // 客户端状态
    list *clients;
    // 时间缓存（秒级）
    time_t unixtime;
    // 时间缓存（毫秒级）
    long mstime;
    // lru时钟
    unsigned lrulock;
    // 上次进行抽样的时间
    long ops_sec_last_sample_time;
    // 上次抽样命令执行数量
    long ops_sec_last_sample_ops;
    // 环形数组，记录抽样结果（默认大小16）
    long ops_sec_samples[REDIS_OPS_SEC_SAMPLES];
    // 环形数组的索引
    int ops_sec_idx;
    // BGREWRITEAOF推迟标志位
    int aof_rewrite_scheduled;
    // 持久化的pid
    pid_t rdb_child_pid;
    pid_t aof_child_pid;
    // 主服务器的信息
    char *masterhost;
    int masterport;
}
```

#### 命令执行过程

1. 将协议格式的命令存入输入缓冲区；

2. 解析协议得到分离的命令参数，存入argv和argc；

3. 匹配执行该命令的函数，存入cmd；

4. 执行预备操作：

   > 检查cmd指针是否执行null；
   >
   > 检查命令参数个数是否合法；
   >
   > 检查是否通过身份验证；
   >
   > 检查是否需要先GC；
   >
   > 若上一次BGSAVE出错，则拒绝命令；
   >
   > 若客户端订阅了频道，则至能执行订阅的命令；
   >
   > 若数据库正在载入，则只能执行flag=1的命令；
   >
   > 若服务器正在执行lua脚本超时阻塞，则只能执行关闭命令；
   >
   > 若服务器正在执行事务，则只能执行exec、discard、multi和watch，其他的进入事务队列；
   >
   > 若打开了监视器功能，则先将参数发给监视器，之后执行。

5. 调用实现函数；

6. 执行后续操作：

   > 慢日志；
   >
   > 命令计数和记录耗时；
   >
   > AOF缓冲写入；
   >
   > 主从复制。

7. 将结果反馈给客户端；

8. 客户端打印结果。

#### serverCorn函数

1. 更新服务器时间缓存（每100毫秒）；
2. 更新LRU时钟（每10秒）；
3. 更新服务器每秒执行命令次数：先计算平均毫秒命令，在乘1000；
4. 更新服务器峰值内存记录；
5. 处理SIGTERM信号：关闭时仅修改标志位，通过时间事件进行关闭，关闭时先RDB；
6. 管理客户端资源：超时的客户端，修正输入缓冲区；
7. GC；
8. 检查持久化：有BGREWRITEAOF被推迟 -> RDB条件满足 -> AOF条件满足；
9. AOF文件写入；
10. 关闭输出缓冲区超出限制的客户端；
11. 自身执行计数。

#### 初始化服务器

1. 初始化服务器状态结构（默认属性）；
2. 载入配置项（覆盖默认属性）；
3. 初始化服务器数据结构；
4. 还原数据库状态（先AOF再RDB）；
5. 执行事件循环。



## 多机数据库的实现

### 1. 复制

#### 旧版SYNC（没有增量同步功能，短线重连开销大）

- 同步

  > 1. 从机发送SYNC命令；
  > 2. 主机进行BGSAVE，并发送给从机；
  > 3. 主机发送缓冲区保存的所有写命令。

- 命令传播

#### 新版PSYNC

> 第一次复制：PSYNC ? -1 -> +FULLRESYNC <runid> <offset>
>
> 部分同步：PSYNC <runid> <offset> -> +CONTINUE

#### 具体实现

1. 设置主服务器的地址和端口；
2. 连理套接字连接；
3. 发送ping命令，确认连接有效且主服务器能正常工作（失败后重复）；

4. 身份验证（只有主从同时关闭，或同时开且且通过后才能继续同步）；
5. 发送端口信息，告知主服务自己的监听端口；
6. 同步；
7. 命令传播。

#### 心跳检测

从服务器每秒向主服务器发送检测命令

- 检测主从服务器的网络连接状态；
- 辅助实现min-slaves选项（如果主机小于预先配置的**稳定**从机**数量**则拒绝写命令）；
- 检测命令丢失（offset比对）。

### 2. Sentinel

#### 定义

sentinel

```c
struct sentinelState {
	// 当前纪元，用于选举时计数
	uint64_t current_epoch;
    // 主服务器字典
    dict *master;
    // 是否进入TILT模式
    int tilt;
    // 正在执行的脚本数
    int running_sripts;
    // 进入TILT模式的时间
    mstime_t tilt_start_time;
    // 最后一次执行时间处理器的时间
    mstime_t prevoius_time;
    // 用户脚本队列
    list *scripts_queue;
}
```

监听的服务器实例

```C
typedef struct sentinelRedisInstance {
	// 服务器状态
    int flags;
    // 实例名字（主服务器是配置文件中起的名字，从服务器为ip:port）
    char *name;
    // 运行id
    char *runid;
    // 配置纪元
    uint64_t config_epoch;
    // 实例的地址
    sentinelAddr *addr;
    // 主观下线周期
    mstime_t down_after_period;
    // 客观下线票数
    int querum;
    // 故障转移时，同时同步的从机数量
    int parallel_syncs;
    // 刷新故障迁移状态的最大时限
    mstime_t failover_timeout;
    // 从机状态
    dict slaves;
    // 哨兵状态
    dict sentinels;
}
```

#### 初始化

1. 初始化服务器；

2. 使用Sentinel专用代码；

3. 初始化Sentinel状态；

4. 初始化masters属性；

5. 创建与主服务器的链接（命令连接、订阅连接）:

   > 订阅连接无法缓存信息，不能在接收的同时发送命令
   >
   > 订阅链接用于哨兵的互相发现

#### 获取信息

1. 获取主服务器状态（每10秒发送一次INFO命令，自动发现从服务器）；

2. 获取从服务器状态（每10秒发送一次INFO命令）；

3. 向主从服务器发送消息（每2秒向订阅频道发送一次监控命令）；
4. 接收主从服务器的订阅消息（更新服务器状态和哨兵状态，自动发现其他哨兵）；
5. 创建与其他哨兵的连接（仅需要命令连接）；

#### 状态监控（针对主从服务器和其他哨兵）

1. 检测主观下线（每秒一次发送PING命令）；
2. 检测客观下线（SENTINEL is-master-down-by-addr <ip> <port> <current_epoch> <runid>），同时进行选举；

#### 故障转移

1. 选出新的主服务器（在线、近5秒成功通信、偏移量最大且id最小的从机）；
2. 修改从服务器复制目标；
3. 将旧的主服务器改为从服务器。

### 3. 集群

#### 定义（CRC16计算槽位）

节点

```C
struct clusterNode {
	// 节点创建时间
	mstime_t ctime;
	// 名字，40个16进制字符组成
	char name[REDIS_CLUSTER_NAMELEN];
	// 状态位
	int flags;
	// 当前纪元
	uint64_t configEpoch;
	// 地址
	char ip[REDIS_IP_STR_LEN];
	int port;
	// 连接节点的信息
	clusterLink *link;
    // 处理的槽位
    unsigned char slots[16384/8];
    int numslots;
}
```

连接

```C
typedef struct clusterLink {
	mstime_t ctime;
	// 套接字描述符
	int fd;
	// 输出缓冲区
	sds sndbuf;
	// 输入缓冲区
	sds rcvbuf;
	// 连接相关节点
	struct clusterNode *node;
}
```

集群状态

```C
typedef struct clusterState {
	// 指向当前节点的指针
	clusterNode *myself;
	// 当前纪元
	uint64_t currentEpoch;
	// 当前状态
	int state;
	// 至少处理一个槽的节点数量
	int size;
	// 集群节点名单
	dict *nodes;
    // 所有槽位的指派信息
    clusterNode *slot[16384];
    // 键与槽之间的关系（用于批量操作）
    zskiplist *slots_to_keys;
    // 正在迁入的槽位
    clusterNode *importing_slots_from[16384];
    // 正在迁出的槽位
    clusterNode *migrating_slots_to[16384];
}
```

#### 集群连接

1. A节点在自己的clusterState.nodes中为B节点创建clusterNode结构；
2. A向B发送meet消息；
3. B节点在自己的clusterState.nodes中为A节点创建clusterNode结构，并回复pong；
4. A收到后回复ping。

#### 查询数据库

1. 计算键所在的槽位；
2. 若指派给了当前节点则直接执行命令；
3. 若指派给了其他节点则重定向到其他节点。

#### 重新分片

1. redis-trib通知目标节点和源节点，准备好对slot的迁移；
2. 源节点向redis-trib发送最多count个slot内的**键名**；
3. redis-trib命令源节点开始迁移；
4. 重复2-3直至迁移完毕；
5. redis-tri通知集群内的其他节点slot已成功迁移。

**迁移过程中查询命令会先查源节点，源节点没有再查目的节点**
