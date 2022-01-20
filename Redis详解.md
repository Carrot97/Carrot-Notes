# Redis详解

## 一、数据结构

### 1. 简单动态字符串SDS

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

服务器

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
}
```

客户端

```C
typedef struct redisClient {
    // 指向select的db
    redisDb *db
}
```

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

