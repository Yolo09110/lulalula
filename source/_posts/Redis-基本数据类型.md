---
title: Redis-基本数据类型
date: 2023-12-09 11:32:23
category: Redis
---

# Redis的几种数据类型

所有数据类型 key-value 键值对的 key 都是字符串类型

## 一、String

### （一）什么是 String

String 是 Redis 中的基本数据类型，value 为字符串值，既可以为字符串，也可以是转换为字符串的整型、浮点型；String 最大为 512M

### （二）应用场景

文本数据、字节数据、序列化后的对象....只要是字符串，都可以往里存。

- String 常见的是用于缓存，value 中存储序列化的对象、json字符串、序列化的图片等等
- 由于 String 对于整型支持 incr decr，也可以用于计数、统计；而且因为是原子性操作，所以不会有安全风险（因为Redis是单线程处理命令）
- 做分布式锁（不要提，防止面试官追问分布式锁相关的东西）

### （三）基础操作

- 设置 key-value 类型的值：set key value
- 批量设置：mset key1 value1 key2 value2 ....
- 不存在才设置 setnx == set key value nx
- 存在才设置 set key value xx   **注意**：没有 setxx
- 设置过期时间：setex key seconds value (单位为s) 等同于 set key value ex seconds

set key value px milliseconds（单位为ms）  **注意**：没有 setpx

- 获取 key 对应的 value：get mget
- 获取所有的 key：keys *
- 获取 key 存储的字符串值的长度：strlen key
- 删除key：del key （会阻塞主线程）   unlink （不会阻塞主线程，适合删除大key）
- 针对 int 编码类型的value，可以使用 incr incrby decr decrby

set 已经存在的 key-value 时，则会覆盖掉旧值以及旧的设定的过期时间

### （四）String底层实现原理

#### 1、编码格式

1. String 存储有三种编码格式，int、embstr、raw
2. Embstr 和 raw 都是由 redisObject 和 SDS（Simple Dynamic String）构成
3. redisObject 中有 type（4位、encoding（4位）、ptr（64位）、lru （24位）、refcount（32位）       总共16个字节； 不管哪种 sdshdr，这个大小都不会变。
4. SDS 中有 len（一个字节，表示已经存储的长度）、alloc（一个字节，表示分配的长度）、flag（一个字节，指示哪种sdshdr）、char[] buf (末尾的 \0 一个字节，buf用于存储数据)，总共四个字节；         针对的是 sdshdr8

##### （1）int

- 针对在 long 表示范围内的整型，使用 INT 编码格式
- 可以使用incr、decr命令；不过只能在 long 范围内，如果超过则报错
- 如果存储的不再是整型或者超出long表示的范围或者使用 append 等修改命令，则会转为 raw 编码
- redisObject 中 encoding 为 redis-object-int，整形数据存储在 ptr 字段内，不使用 SDS                    **注意：**因为 ptr 恰好和 long 型一样都是 64位，所以数据存储在 ptr 中是没有问题的；

##### （2）embstr

- 针对小于等于阈值字节大小的字符串（也叫小字符串）（5.0.5版本的阈值是 44字节），如果超过，则转为 raw（考虑redisObject 和 SDS 头部结构，则整个大小是 44 + 16 + 4 = 64字节）
- redisObject 和 SDS 分配在内存上是连续的，只需要一次分配，并且回收时也是一次回收
- redisObject 的 ptr 变量指向SDS
- embstr 设置为只读，如果对其进行修改（比如 append 命令），则先转变为 raw，再修改
- embstr 只读的原因在于，如果一个字符串值进行了修改，则它就被认定为易变的；因为 redisObject 和 SDS 是一同分配的，如果字符串值修改导致 SDS 需要扩容，则连同 redisObject 一起都要重新分配； 



##### （3）raw

- 针对大于阈值（5.0.5 版本为 44）的字符串（也叫大字符串），以及其他由于修改转为 raw 的字符串值
- redisObject 和 SDS 是分两次分配的，回收也需要两次操作；它们在内存上也是不连续的
- 如果 SDS 需要扩容，即使需要重新分配，复制到其他地方，redisObject 也是不需要重新分配的，只是 ptr 变化



为什么是以44字节为界的呢？（针对redis5.0.5版本）

首先，44字节是 64字节 - 20字节（redisObject 大小）- 4字节（SDS sdshdr8 类型头部结构大小）

也就是说，问题的本质在于，为什么是以64字节为界。

主要原因在于，CPU cacheLine 的大小为64字节，如果大小控制在64字节及以内，那么CPU只需要在内存中调取一次，就可以获得所有的数据。这样的话，可以更好地利用局部性原理。

#### 2、什么是 SDS

- SDS （Simple Dynamic String），简单动态字符串，是针对普通字符串（也就是结尾为 '\0' 的字符数组）的一些缺点提出来的，是对普通字符串的进一步封装。
- SDS 分为 sdshdr5、sdshdr8、sdshdr16、sdshdr32、sdshdr63 五种
- sdshdr5 中只有一个 flag 字段和 buf 数组，flag 的高三位表示sds的类型，低五位用来存放已存储的数据长度
- sdshdr8、16、32、64 都是由 len、malloc、flag、buf[] 组成，len 表示已经存储的数据大小，malloc 表示分配的 buf[] 大小，flag 表示哪种 sds，buf[] 存储数据
- sdshdr8 中的 len 只有8位，因此能表示的最大长度为 2^8 （128）

#### 3、为什么需要 SDS

这个问题也可以转变为：为什么字符数组不适合？ 字符数组有什么缺点？（Redis 是C语言写的，字符串是用结尾为 '\0' 的  char 型数组表示的）

（1）字符数组的缺点：

- 获取长度需要遍历一遍，时间复杂度为 O(n)
- 扩容比较麻烦，需要重新分配内存然后复制过去
- 非二进制安全（字符数组以 \0 作为结束符，\0没法作为普通字符进行存储）

（2）针对上述缺点，SDS 的解决方案：

- SDS 头部结构中有 len 字段，用于表示已经存储数据的长度，可以直接返回，时间复杂度为 O(1)
- SDS 有预留空间 malloc - len。
- 二进制安全，判断字符数组结束不是靠 \0 ，而是 len 字段

对于预留空间，SDS是这样操作的：当第一次添加字符串值时

如果添加的字符串值长度 len 小于 1m （1024）字节，则 malloc 分配 len * 2 大小的空间，即预留空间大小为 len；

如果 len 大于 1m，则分配 len + 1m 大小的空间，即预留空间大小为 1m  

所以，事实上，预留空间大小为 min（len ， 1m）；

这样的策略节省空间



## 二、Lists

### （一）什么是 Lists

Lists 也是 Redis 的基本数据类型，key 为字符串，value 是字符串列表，可以从两端进行操作（但不是双向链表），不符合先入先出。有元素个数限制，是 2^32 - 1 ,新版本是 2^64 - 1

### （二）应用场景

Lists 可以用来做消息队列，可以用来存储一批任务数据、一批消息等

### （三）基础操作

- 添加：lpush rpush
- 弹出：lpop rpop
- 范围获取：lrange key begin end             eg：获取所有 lrange key 0 -1  （负数代表倒数第几个）
- 删除key：del   unlink
- 删除特定的 value： lrem key count value  (删除从左到右count个value等于value的值，如果count为0，则是全部删除)
- 获取value列表元素个数：llen key

### （四）底层实现

#### 1、编码格式

3.2版本之前，底层是使用 LinkedList 和 ZipList

在列表元素个数小于 512 并且列表所有元素小于 64 字节时，使用 ZipList（注：这是List的规定，而不是ZipList的限制）

其他情况使用 LinkedList

##### （1）ZipList

- ZipList 底层是压缩列表，列表的对象元素紧挨在一起，在元素之前，有 zllen；在之后有zlend；   
- 每一个元素是一个entry，其中有 prevlen 字段，表示前一个对象元素的长度，因此 ZipList 有了从后往前遍历的能力。



##### （2）LinkedList

- LinkedList 底层是双向链表，不是连续分布的
- LinkedList 的表头结构中，有一个 len 字段，表示链表中的节点数量。因此获取节点元素个数的时间复杂度为 O(1)

```C++
typedef struct list {
    listNode*head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;   //长度字段
} list;
```



二者相比：

ZipList 对象元素紧凑存储，内存碎片产生少，并且没有链表节点指针的空间消耗；而 LinkedList 插入的效率比较高

对于对象元素数量多的，应该采用 LinkedList，因为ZipList 的对象元素是紧挨着的，插入一个对象元素后续的对象元素都要移动，时间复杂度为 O(n)，而 LinkedList 插入的时间复杂度为 O(1)

对于对象元素大的，也应该采用 LinkedList，因为ZipList 的元素对象是紧挨着的，如果插入新的对象元素，由于后续的对象元素都很大，移动的效率会很低

LinkedList 和 ZipList 查询一个对象元素的时间复杂度都是 O(n)

在3.2版本以后，使用了 QuickList 

##### （3）QuickList

QuickList 是将 ZipList 和 LinkedList 的优点结合起来的新的编码格式，QuickList 有两个指针，指向多个 QuickListNode 连接成的链表的头和尾，而每个 QuickListNode 存储一个 ZipList

QuickList 是压缩列表组成的双向链表



### （五）底层数据结构压缩列表

#### 1、什么是压缩列表

从名称就可以看出，（就是排列紧凑的列表）是一种内存紧凑存储元素的数据结构，内存上连续分布，节省内存空间，不易产生空间碎片，也没有链表指针的空间开销；并且，由于内存上连续，CPU使用缓存命中率高（空间局部性会好一些）

压缩列表在 Redis 中有两种编码格式，ZIPLIST 和 LISTPACK，一般说的压缩列表就是指 ZIPLIST，LISTPACK 是 5.0 引入，直到7.0才完全替代 ZIPLIST

#### 2、压缩列表解决什么问题

节省内存空间（节省链表指针的开销）、小数据量时，遍历访问性能更好（顺序连续存储、缓存命中率好）

缓存命中率体现在是顺序存储，数据可以一次性从内存调入CPU中（空间局部性）

#### 3、ZIPLIST 的整体结构

ZIPLIST 的整体结构从前往后是 zlbytes、zltail、zllen、entry1、entry2.... zlend

- zlbytes：表示整个 ZIPLIST 的字节大小，包含 zlbytes 本身
- zltail：表示从 ZIPLIST 开始到最后一个 entry 的距离，zlbytes = zltail + 最后一个 entry 的长度 + zlend 长度（一个字节）； 

可以通过 zltail 和 ZIPLIST 的起始地址定位到最后一个 entry ； 如果是一个空的 ZIPLIST，则 zltail 定位到的是 zlend

- zllen：表示 entry 的个数
- zlend：一个特殊的 entry ；表示 ZIPLIST 的结尾，是用一个全一字节 11111111 表示
- entry：存储的对象元素

#### 4、ZIPLIST 的节点结构（entry）

entry 由三部分构成，从前往后是 prevlen、encoding、entry-data

- prevlen：表示前一个 entry 的长度，通过这个字段，可以定位到上一个 entry 的开始，ZIPLIST 才有了从后往前遍历的能力；prevlen 默认是一个字节，但是当前一个元素的大小超过 254 （2^8 - 1）时（不能是 255，因为 11111111 是结尾标志，不能使用），prevlen 就会变为5个字节，最高字节是 11111110（254），用于标识该prevlen为5字节的，剩下的字节表示上一个 entry 的长度。 **因为这个**，导致了ZIPLIST 最主要的问题——连锁更新
- encoding：encoding 是一个整型字段，encoding 用来表示 entry-data 的数据类型以及长度，一般是高几位代表类型，低位代表长度；但是对于 int 类型，每一种encoding代表一种特定长度的int； 特别的，对于 1111**** 这个编码，除了 11111111（代表结束）和 11111110（代编8个字节的INT64），其他的就表示特定的 INT 类型的数，比如 11110001 就代表 1，此时 entry-data 就没有了；                 **因为有了 encoding 表示 entry-data 的长度，才使得 ZIPLIST 有了从前往后遍历的能力**



- entry-data：存放的实际数据

根据 encoding 表示 entry-data 的类型和长度，使得有了从前往后遍历的能力，这是因为，entry 的三个部分中，prevlen 可以通过最高字节是否是 11111110 判断出是一个字节还是五个字节长；encoding 可以根据最高几位得出 encoding 的长度，比如最高位为 00 ，则代表 encoding 是一个字节长；entry-data 的长度可以由 encoding 获得；因此，虽然只有 entry-data 的长度是可以直接获得的，但是整个 entry 的长度也是可以确定的。

#### 5、ZIPLIST 查询数据

- 查询 ZIPLIST 的数据总量：通常情况下，直接通过 zllen 就可以获得 entry 的个数，时间复杂度为 O(1)； 为什么是通常情况呢，因为 zllen 只有两个字节，最大可以表示 11111110 （11111111 表示结尾），即 65534 个entry，如果 entry 个数超过 65534，则 zllen 会置为0，提示需要遍历才能得出长度，此时时间复杂度为 O(n)
- 在 ZIPLIST 中查询指定数据的节点：需要遍历，时间复杂度为 O(n)

#### 6、ZIPLIST 更新数据

ZIPLIST 更新数据是比较消耗资源的，因为底层是压缩列表，插入一个对象元素时，后边的都需要移动，尤其是对象元素数量多 或者 对象元素普遍比较大的情况。尾部插入就不需要移动；综合起来，时间复杂度还是为 O(n)

并且，ZIPLIST 存在一个最主要的问题——连锁更新。

连锁更新是指，当某个 entryA 前插入一个大于 254 字节的 newentry 时，entryA 的 prevlen 就要从一个字节变为五个字节， 而如果 entryA 的总长度因为 prevlen 的变化超过了 254 字节时，后边的 entryC 的 prevlen 又会从一个字节变为五个字节..... 如此下去，则后续很多 entry 都要发生变化，而每一次 entry 的变化，都会使后续的都发生移动

虽然连锁更新的概率很小，但是还是 ZIPLIST 的最主要的问题，主要是因为会导致性能不稳定；连锁更新的时间复杂度为 O(n^2)

#### 7、LISTPACK 优化

连锁更新的原因就在于：由于需要从后往前遍历的能力，所以有了 prevlen 字段，因为这个字段长度的变化，才导致了连锁更新问题的发生。要解决连锁更新问题，就要对 entry 的结构进行改变。

LISTPACK 采取的优化办法是，改变 entry 的结构。分为三部分：encoding-type、element-data、element-tot-len。重点就在于 element-tot-len

##### element-tot-len

这个字段表示当前 encoding-type + element-data 的长度（取消了 ZIPLIST 中表示上一个 entry 长度的 prevlen），由于该字段只是表示当前 entry 一部分的长度，所以前面插入大的 entry 也是没有影响的。

element-tot-len 采用了一种比较巧妙地方式，它是从后往前扫描，每个字节的高位是标识 element-tot-len 是否结束的标志。为 1 代表整个字节不是结束，需要继续往前扫描，如果是 0 则代表这是最后一个字节，停止扫描。最终所有的字节去掉最高位合并起来就是正确的长度

利用这样的方式，encoding-type + element-data 的长度无论是多大，都可以用 element-tot-len 表示。

比如说 element-tot-len 为 00001000 10001100 ，从后往前第一个字节为 10001100，最高位为1，代表继续扫描。第二个字节为 00001000，最高位为0，停止扫描；两个字节都去掉最高位，则最终的结果是 0001000 0001100 ，代表 1024 + 4 + 8 = 1036

### （六）面试题

1、ZIPLIST 的优点（可以与 LinkedList 比较着说）

节省内存（避免链表节点指针的开销），产生空间碎片概率小、一次性分配内存（因为是连续的，因此一次性分配就可以，而 LinkedList 需要多个节点多次分配）、空间局部型好（小数据量遍历时性能更好，缓存命中率高）

2、QUICKLIST的设计是怎样的

为了应对ziplist的问题，在3.2版本实现的quicklist在ziplist的基础上，通过链表将ziplist串联起来，每一个链表元素都是一个ziplist,这样就可以减少新增ziplist时的内存分配，而且quicklist限制了ziplist的大小，当一个ziplist大小过大，就会新增一个quicklist节点

3、为什么又出现了LISTPACK的设计

虽然quicklist的调和了ziplist和linklist的一些优缺点，但是由于quicklist使用quicklistNode指针ziplist,也会增加较大的内存开销，所以，又实现出了listpack结构，listpack延续ziplist的紧凑型结构，但是区别的是listpack中的每一个节点不会记录前一个节点的长度，可以避免连锁更新问题



## 三、Set

### （一）什么是 set

set 就是一系列对象元素的无序、不重复集合；

### （二）set 的应用场景

适用于无序、不重复的场景，比如某个用户关注的博主、公众号之类的，这些信息可以放进一个集合；并且set 因为有并交叉运算，可以轻松获得用户之间的共同关注之类的信息

### （三）常用操作

创建新的set、向set中添加对象元素： sadd

获取所有对象元素：smenbers

获取set 对象元素个数：scard

删除对象元素：srem key member1， member2 .....

判断是否是 set 中的：sismember

获取范围内：sscan key cursor [match patten] [count n]     **注意：**  对intset编码的没用

交并差：sinter key1 key2 key3 ...    sunion key1 key2 key3...     sdiff key1 key2 key3...

### （四）底层实现

set 出于性能和内存的双重考虑，底层有两种编码格式，intset 和 hashtable

对于元素都是整型，并且个数少于 512 的，采用 intset，其他采用 hashtable

#### 1、intset

intset 是内存连续的，查找 member 时，使用二分查找，也就是说，它其实是有序的



#### 2、hashtable

hashtable 的 key 用来存储元素，value 都为 null，因此，hashtable 查找元素的时间复杂度为 O(1)





## 四、Hash

### （一）什么是 Hash

Hash 是 Redis 中的基本数据类型，它是 field 和 value 都是字符串的 hash 表；Redis 中每个 hash 最多可以存储 2^32 - 1 对键值对

### （二）Hash 的应用场景

适合 O(1) 查找某个 filed 对应 value 的场景，适合 key/value 形式的数据存储

Hash可以用来存储一些类似一对一对配置信息的数据（类似 json）

### （三）Hash 常用操作 

新建hash、添加：hset key field1 value1 filed2 value2 .....     、 hsetnx

获取 hash 中 filed 对应的 value ： hget key filed

获取所有的 filed 和 value：hgetall

获取 hash 对的个数：hlen key

获取 hash 对：hscan       **注意** *scan 类型的命令对于 ZIPLIST、INTSET 等内存连续型是无效的

删除 hash 对：hdel key filed1 filed2 filed3......

删除hash ：del 、unlink

### （四）底层实现

Hash 底层有两种实现编码 ZipList（在7.0之后被 ListPack 替代）和  HashTable

- 当 hash 对的 filed 和 value 都小于 64 字节并且 filed 数量少于 512 时，使用 ZipList（跟 List 类似）
- 其他情况使用 HashTable

#### 1、ZipList

hash 对的 filed 和 value 单独作为 entry 存储，整个 ZipList 在内存上连续。



#### 2、HashTable

跟 set 不同的是，hash 的底层 hashtable 的 value 不为 null，而是存的 hash 对的 value。





## 五、HashTable

HashTable 扮演着快速索引的角色，可以使用 O(1) 的时间复杂度查询到 key 对应的 value

### （一）HashTable 的结构

HashTable 实现是 dictht 对象，内部有 size、sizemask、used、table 数组

- size：HashTable 的存储空间
- sizemask：哈希掩码，用于与哈希值按位相与获得存储的位置；等于 size - 1；
- used：实际存储的键值对的数量
- Table 数组：存储实际数据；数组的每一个元素是一个 bucket，用于存储一串哈希值相同的键值对；（也就是解决哈希冲突采用拉链法）



###  （二）哈希表渐进式扩容/缩容

为什么要扩容缩容呢？

当数据存储的越来越多，哈希冲突概率越来越大，链表长度变长，此时的查询效率就不再是 O(1),而是O(n)，n 为链表的长度；此时就需要扩容

当数据存储的很少时，为节省空间，就进行缩容

一般扩容/缩容的操作是，新分配一块内存，然后将数据一次性迁移过去；这样就造成了一个问题：数据量很大的时候，迁移是非常耗时的，会导致服务器在一段时间内停止服务；为解决这个问题，就提出了渐进式扩容；

首先，在 dictht 上又封装了一层 dict，也就是常说的字典（Redis 存储数据就是使用的字典）；dict 的构造：

```Java
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; 
    unsigned long iterators；
} dict;
```

- type：指向 dictType 的指针，保存了一系列特定函数，比如计算 hash 值的 hashFunction() 
- ht[2]：ht[0] 是当前存储数据的 dictht； ht[1] 是扩容或缩容时新分配内存的 dictht
- rehashidx：是迁移时的索引，指向当前要迁移的数据；为 0 代表扩容开始；为 -1 代表完成

当要扩容/缩容时，首先给 ht[1] 分配内存：

扩容时，分配的内存大小为大于等于 ht[0] 的 used * 2 的第一个二次方幂；

缩容时，为大于等于 used 的第一个二次方幂；

分配完内存，将 rehashidx 置为0，代表扩容开始；每发生一次增删改查，就将 rehashidx 指向的数据迁移到 ht[1] 中；直到所有的数据迁移完成，释放 ht[0] 的空间，ht[1] 和 ht[0] 的指针交换，将 ht[1] 又指向空的哈希表；最后将 rehashidx 置为 -1，代表完成

 注意，扩容或者缩容开始后，ht[0] 这个表数据只会少，不会多；

新增数据时，只会新增到 ht[1] ；删除、修改、查找数据时，在 ht[0]、ht[1] 都要查找，找到进行操作；

> 在迁移过程中，rehashidx 指向的可能为空（此处数据之前可能被删了），此时并不是直接返回，而是继续往下搜索，如果连续10个都为空，则返回；

### （二）哈希表扩容/缩容时机

扩容缩容跟负载因子有关；负载因子 = used / size

- 当负载因子大于 1 小于 5时，如果此时服务器有 BGSAVE 、BGREWRITEAOF 命令执行，则不扩容；如果没有这些命令执行，则扩容
- 当负载因子大于 5 时，不管有没有命令执行，都扩容
- 当负载因子小于 0.1 并且没有上述命令执行时，缩容



## 六、跳表

### （一）什么是跳表

跳表也是链表，只不过是多级索引的链表，通过多级索引，就可以实现一次性跨越多个节点；可以将普通链表 O(n) 的操作时间复杂度降为 O(logn)

### （二）跳表的结构

跳表的结构：

```Java
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```

- header、tail 为头尾节点
- length 代表节点个数
- level 代表最大有几层索引

跳表节点的结构：

```Java
typedef struct zskiplistNode {
    //Zset对象的元素值
    sds ele ;
    //元素权重值
    double score;
    //后向指针
    struct zskiplistNode *backward;
    //节点的level数组，保存每层上的前向指针和跨度
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
} zskiplistNode; 
```

sds：Simple Dynamic String，存放数据的，之前学过

score：分值，就是按照这个字段对数据进行排序

backward:  从后向前的指针，一个节点只有一个，这个指针是针对第一层的，其他层没有后向指针；

level 数组：数组有几个元素代表有几层索引，每一个元素的 forward 存放的是指向该层下一个节点的指针，span 是到该层下一个节点实际跨越的距离; 在zset 使用跳表作为编码时，zrank 命令就使用了 span



当 zset 使用跳表作为编码方式的时候，zrevrange 命令就是依靠的 backward 字段

### （三）节点的索引层数

节点的索引层数是随机的，每一个节点在新建的时候就随机确定了层数；初始都为一层，每加一层的概率是 0.25，也就是说第一层每四个有一个进入第二层；第二层每四个进入第三层......

层数具有最大限制，目前使用的版本 5.0 限制是 64



## 七、ZSet

### （一）什么是 ZSet

Zset 是一种有序、不重复集合，是一组按照关联分数（score）排序的字符串集合；这里的分数可以由任何指标计算得来

### （二）ZSet 使用场景

有序的场景，比如点赞榜、积分榜、优先度排行等

### （三）常用操作

添加修改：zadd key score member .... 、zrem key member ...  (zadd 可以搭配 nx、xx、lt、bt 等参数)

zcard key：查询 member 个数

zcount key min max： 查询分数在 min max 之间的元素个数

zrange key begin end （withscores）：查询位置范围内的 member ，如果由 withscore 选项，则连同分数一块查出

zrevrange key begin end（withscores）：反向查找

zrank key member：查询 member 的排名，从 0 开始

zscore key member ：查询member 的分数 

### （四）底层实现

ZSet 的底层实现有两种，压缩列表和跳表

- 当member 数量少于 128，并且每个 member 小于 64 字节时，使用压缩列表
- 其他情况使用跳表 skiplist + hashtable

#### 1、压缩列表

5.0版本使用的还是 ziplist



#### 2、skiplist + hashtable

使用 skiplist 存储 member（sds） 和 score（score 字段）

hashtable 存储 key 为 member，value 为 score；（hashtable 的唯一作用就是 zscore 命令，复杂度为 O(1) ）  
