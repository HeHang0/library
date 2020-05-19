## 常用命令

### setnx
并发锁，设置setnx k v，值为当前时间加上超时时间和一个随机数的时间戳，设置成功，则抢到锁；失败，则获取k的v，即拿到超时时间，判断是否过期，如果过期，则加锁成功，使用getset k v重新设置v，他返回旧的v，如果旧的v不同，则加锁失败，被其他人抢去；否则失败，等待重试；解锁就只需获取v，判断是否过期，过期直接删了k

## 底层数据结构

所有的redis对象都是由redisObject来表示
```c++
typedef struct redisObject{
    //类型
    unsigned type:4;
    //编码
    unsigned encoding:4;
    //指向底层数据结构的指针
    void *ptr;
    //引用计数
    int refcount;
    //记录最后一次被程序访问的时间
    unsigned lru:22;
}
```
type：表示redis的5种基本类型，string、list、hash、set、zset；
encoding与*ptr：对象的 *prt 指针指向对象底层的数据结构，而数据结构由 encoding 属性来决定；
  
### sds（simple dynamic string，简单动态字符串）

数据结构：len，free int，data byte[]；len保存长度，free记录未使用的字节量，data保存字符串；
len可以减少获取长度时的遍历，可以减少字符串拼接时空间不足而产生的缓冲区溢出，减少修改时内存重新分配次数；可以预分配，分配更多的空间防止后续增加，懒释放，缩短时不直接释放，而计算free来预备后续的增加；
        
### list（链表）

c无链表，redis自己实现了；
listNode：*prev，*next listNode，*value；list双向链表，有head和tail标识头尾，len表长度

### hash（哈希表）

hash的结构体，table为具体的k、v值；hash会采用渐进式策略来rehash（当需要缩扩容时，重新hash），大量的k、v势必堵塞当前进行，导致redis不可用，所以一般是新生成一个hash，分步操作的，新的增一定在新的hash上，查的时候两个挨着查；
```c++
typedef struct dictht{
    //哈希表数组，它为指针，指向一个指针数组，指针数组中的指针为具体kv结构体的地址，这个地址为根据k算出来的hash值地址在使用sizemask属性算出具体的index值
    dictEntry **table;
    //哈希表大小
    unsigned long size;
    //哈希表大小掩码，用于计算索引值，总是等于 size-1
    unsigned long sizemask;
    //该哈希表已有节点的数量
    unsigned long used;
}
```
每个k、v存在在下面这个结构体中，*next为指向下一个值的指针，这就是链地址法，为了防止hash冲突而使用的
```c++
typedef struct dictEntry{
    //键
    void *key;
    //值
    union{
        void *val;
        uint64_tu64;
        int64_ts64;
    }v;
    //指向下一个哈希表节点，形成链表
    struct dictEntry *next;
}
```

### skiplist（跳跃表）

数据结构
```c++
// 节点
typedef struct zskiplistNode {
    //层
    struct zskiplistLevel{
        //前进指针，指向这一层的下一个节点
        struct zskiplistNode *forward;
        //跨度
        unsigned int span;
    }level[];
    //后退指针
    struct zskiplistNode *backward;
    //分值
    double score;
    //成员对象
    robj *obj;
}

// 跳表
typedef struct zskiplist{
    //表头节点和表尾节点
    structz skiplistNode *header, *tail;
    //表中节点的数量
    unsigned long length;
    //表中层数最大的节点的层数
    int level;
}
```

搜索：从最高层的链表节点开始，如果比当前节点要大和比当前层的下一个节点要小，那么则往下找，也就是和当前层的下一层的节点的下一个节点进行比较，以此类推，一直找到最底层的最后一个节点，如果找到则返回，反之则返回空;

插入：首先确定插入的层数，有一种方法是假设抛一枚硬币，如果是正面就累加，直到遇见反面为止，最后记录正面的次数作为插入的层数。当确定插入的层数k后，则需要将新元素插入到从底层到k层;

删除：在各个层中找到包含指定值的节点，然后将节点从链表中删除即可，如果删除以后只剩下头尾两个节点，则删除这一层;

### intset（整数集合）

保存int16_t、int32_t 或者int64_t 的整数值；同一集合内所有元素类型统一；

```c++
typedef struct intset{
    //编码方式
    uint32_t encoding;
    //集合包含的元素数量
    uint32_t length;
    //保存元素的数组
    int8_t contents[];
}
```

具体的元素类型由encoding决定，当新的元素类型的长度比当前要大时，例如int16的集合新增了int32元素，则进行升级；升级时根据新元素扩展新的contens大小，并分配空间，然后将已存在元素进行类型转换后，按顺序放入数组中，保证整个集合有序；然后将新的元素插入到合适的位置；升级后不支持降级；

### ziplist（压缩链表）

redis为了节省内存而开发的结构，是由一系列特殊编码的连续内存块组成的顺序型数据结构，一个ziplist可包含多个数据节点，每个节点可保存一个字节数组或一个整数值；<br>
压缩列表并不是对数据利用某种算法进行压缩，而是将数据按照一定规则编码在一块连续的内存区域，目的是节省内存；

每个节点的结构：<br>
|previous_entry_ength|encoding|content|previous_entry_ength|

记录压缩列表前一个字节的长度；<br>
encoding：保存的是节点的content的内容类型以及长度；<br>
content：用于保存节点的内容，节点内容类型和长度由encoding决定；

### quicklist快表

redis3.2版本新增的数据结构；考虑到链表的附加空间相对太高，prev 和 next 指针就要占去 16 个字节 (64bit 系统的指针是 8 个字节)，另外每个节点的内存都是单独分配，会加剧内存的碎片化，影响内存管理效率；后续redis使用ql代替linkedlist；他是ziplist与linkedlist的组合：

```c++
struct quicklistNode {
    quicklistNode *prev;
    quicklistNode *next;
    ziplist *zl;         // 指向压缩列表
    int32 size;          // ziplist字节总数
    int16 count;         // ziplist中元素数量
    int2 encoding;       // 存储形式，表示原生字节数组还是LZF压缩存储
    ...
}
struct quicklist {
    quicklistNode *head;
    quicklistNode *next;
    long count;           // 元素总数
    int nodes;            // ziplist节点个数
    int compressDepth;    // LZF算法压缩深度
}
```

每个节点使用linkedlist结构，但存储的数据为指向ziplist的指针

## 五大基本数据结构

### string
支持批量，MGET、MSET，但注意不要太大了，会影响整体相应时间，因为是单进程单线程的；setnx可以用于实现分布式锁；incr（increment）自增，decr自减，incrby增加某一个值；redis的float是用string存的，用的时候转换下；
  
string可以有int、raw、embstr(只读的)编码；string使用reidsobject和sds保存；raw需要分配的时候需要分配两次，ro一次，sds一次，而embstr只需一次，ro和sds同时分配，挨在一起；当int类型不是int或保存的长度超过long时转化为raw；

### list
链表，一般用作队列；老版本中结构为ziplist或者双向链表；小数据量时ziplist，大了转为list；3.2后使用quicklist结构

### set
无序不可重复集合，可以求交集并集差集；sdiff返回一个集合与另一个的差集，sdiffstore与前面一样，但是返回的的数据保存在目标集合中，如果它已存在则覆盖；sinter返回多个集合的交集；sinterstore与sdiffstore一样，只不过返回的是交集；sismember，key是不是他的成员；smembers返回所有成员；sunion返回多个集合的并集，同样存在sunionstore；

set可使用intset或者hash实现，intset所有的元素都在intset中；hash则使用k存储所有的元素，v设为null；两种格式转换，当所有元素为int同时元素个数小于512时，使用intest，否则hash；

### zset
有序集合，每个string有自己的score值；zcard返回元素个数；zcount zset min max返回zset中范围[min,max]之间的元素个数；zrange zset start stop WITHSCORE返回zset中指定区间大小的成员按score从小到大（desc则使用zrevrange），start、stop为下标；zrank返回成员的排名默认从小到大（desc用zrevrank）；

zset可以使用ziplist或者skiplist；使用ziplist时，name紧挨着score，然后按照从小到大的顺序进行排列；<br>
并不是以单纯的skiplist作为另一种存储，而是使用zset(hash的dictht与skiplist组合)作为存储结构，ht的k为元素名、v为score，skiplist的obj为名，score为值；他们两个均使用指针来共享数据，不会造成内存浪费；使用两者的原因为ht的O(1)查找，sl的范围查找；还是小数据用ziplist超过后使用zset；

### hash
hkeys获得所有key，hlen获得总数，hmget、hmset批量增取，hvals获取所有value；

hash使用ziplist或者hashtable来存储，还是少量短数据时使用ziplist，否则hashtable；
