> `string，list(列表)，set(集合)，zset(有序集合)，hash`

## 1. 字符串

我们知道Redis是用C语言开发的，但是底层存储不是使用C语言的字符串类型，而是自己开发了一种数据类型SDS进行存储，SDS即*Simple Dynamic String* ，是一种动态字符串。

```c
struct sdshdr{
 int len;/*字符串长度*/
 int free;/*未使用的字节长度*/
 char buf[];/*保存字符串的字节数组*/
}
```



![](https://gitee.com/QingHui/picGo-img-bed/raw/master/img/image-20210910114106386.png)



SDS与C语言的字符串有什么区别呢？

- C语言获取字符串长度是从头到尾遍历，时间复杂度是O(n)，而SDS有len属性记录字符串长度，时间复杂度为O(1)。

- 避免缓冲区溢出。SDS在需要修改时，会先检查空间是否满足大小，如果不满足，则先扩展至所需大小再进行修改操作。

- 空间预分配。当SDS需要进行扩展时，Redis会为SDS分配好内存，并且根据特定的算法分配多余的free空间，避免了连续执行字符串添加带来的内存分配的消耗。

- 惰性释放。如果需要缩短字符串，不会立即回收多余的内存空间，而是用free记录剩余的空间，以备下次扩展时使用，避免了再次分配内存的消耗。

- 二进制安全。c语言在存储字符串时采用N+1的字符串数组，末尾使用'\0'标识字符串的结束，如果我们存储的字符串中间出现'\0'，那就会导致识别出错。而SDS因为记录了字符串的长度len，则没有这个问题。

> Redis 规定了字符串的长度不得超过 512 MB 

  ### 1.1 编码

　　字符串对象的编码可以是int，raw或者embstr。

　　1、int 编码：保存的是可以用 long 类型表示的整数值。

　　2、raw 编码：保存长度大于32字节的字符串（redis3.2版本之前是39字节，之后是44字节）。

　　3、embstr 编码：保存长度小于32字节的字符串（redis3.2版本之前是39字节，之后是44字节）。

## 2. hash

### 2.1 编码

哈希对象的编码有两种，分别是：**ziplist、hashtable**。

当哈希对象保存的键值对数量小于 512，并且所有键值对的长度都小于 64 字节时，使用ziplist(压缩列表)存储；否则使用 hashtable 存储。

Redis中的hashtable跟Java中的HashMap类似，都是通过"数组+链表"的实现方式解决部分的哈希冲突。直接看源码定义。

```c
typedf struct dict{
    dictType *type;//类型特定函数，包括一些自定义函数，这些函数使得key和value能够存储
    void *private;//私有数据
    dictht ht[2];//两张hash表 
    int rehashidx;//rehash索引，字典没有进行rehash时，此值为-1
    unsigned long iterators; //正在迭代的迭代器数量
}dict;

typedef struct dictht{
     //哈希表数组
     dictEntry **table;
     //哈希表大小
     unsigned long size;
     //哈希表大小掩码，用于计算索引值
     //总是等于 size-1
     unsigned long sizemask;
     //该哈希表已有节点的数量
     unsigned long used; 
}dictht;

typedf struct dictEntry{
    void *key;//键
    union{
        void val;
        unit64_t u64;
        int64_t s64;
        double d;
    }v;//值
    struct dictEntry *next；//指向下一个节点的指针
}dictEntry;
```

![结构图](https://gitee.com/QingHui/picGo-img-bed/raw/master/img/v2-402a9fd1ec990958a04243435f71a778_1440w.jpg)

下面讲一下扩容和收缩。当哈希表保存的键值太多或者太少时，就会通过rehash来进行相应的扩容和收缩。

**扩容和收缩的过程**：

1、如果执行扩展操作，会基于原哈希表创建一个大小等于 ht[0].used*2n 的哈希表（也就是每次扩展都是根据原哈希表已使用的空间扩大一倍创建另一个哈希表）。相反如果执行的是收缩操作，每次收缩是根据已使用空间缩小一倍创建一个新的哈希表。

2、重新利用哈希算法，计算索引值，然后将键值对放到新的哈希表位置上。

3、所有键值对都迁徙完毕后，释放原哈希表的内存空间。

在redis中执行扩容和收缩的规则是：

- 服务器目前没有执行 BGSAVE 命令或者 BGREWRITEAOF (持久化)命令，并且负载因子大于等于1。
- 服务器目前正在执行 BGSAVE 命令或者 BGREWRITEAOF (持久化)命令，并且负载因子大于等于5。

负载因子 = 哈希表已保存节点数量 / 哈希表大小。

**渐进式rehash**

什么是渐进式，也就是说扩容和收缩不是一次性，集中式地完成，而是通过多次逐渐地完成的。为什么要采用这种方式呢？如果是几十个键值，那么rehash当然可以瞬间完成，如果是几十万，几百万的键值要一次性进行rehash，势必会导致redis性能严重下降，自然而然地redis开发者就想到采用渐进式rehash。过程如下：

在rehash时，会使用rehashidx字段保存迁移的进度，rehashidx为0表示迁移开始。

在迁移过程中ht[0]和ht[1]会同时保存数据，ht[0]指向旧哈希表，ht[1]指向新哈希表，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作以外，还会顺带将ht[0]的元素迁移到ht[1]中。

随着字典操作的不断执行，最终会在某个时间节点，ht[0]的所有键值都会被迁移到ht[1]中，rehashidx设置为-1，代表迁移完成。如果没有执行字典操作，redis也会通过定时任务去判断rehash是否完成，没有完成则继续rehash。

rehash完成后，ht[0]指向的旧表会被释放, 之后会将新表的持有权转交给ht[0], 再重置ht[1]指向NULL。

**渐进式rehash的优缺点**：

优点是把rehash操作分散到每一个字典操作和定时函数上，避免了一次性集中式rehash带来的服务器压力。

缺点是在rehash期间需要使用两个hash表，占用内存稍大。

hash类型的常用命令有：hget、hset、hgetall 等。

## 3. list(链表)

列表对象的编码有两种，分别是：ziplist、linkedlist。当列表的长度小于 512，并且所有元素的长度都小于 64 字节时，使用ziplist存储；否则使用 linkedlist 存储

```c
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

typedef struct listIter {
    listNode *next;
    int direction;
} listIter;

typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```

我们可以看到，listNode就是链表中的节点元素，通过prev和next组成双向链表。

![](https://gitee.com/QingHui/picGo-img-bed/raw/master/img/image-20210910114220850.png)

list则记录了头结点head，尾结点tail，还有链表长度len，match函数用于比较两个节点的值是否相等，操作起来更加方便。

![](https://gitee.com/QingHui/picGo-img-bed/raw/master/img/image-20210910114021899.png)

list类型常用的命令有：lpush、rpush、lpop、rpop、lrange等。

## 4. set(集合)



set类型的特点很简单，无序，不重复，跟Java的HashSet类似。它的编码有两种，分别是intset和hashtable。如果value可以转成整数值，并且长度不超过512的话就使用intset存储，否则采用hashtable。

hashtable在前面讲hash类型时已经讲过，这里的set集合采用的hashtable几乎一样，只是哈希表的value都是NULL。这个不难理解，比如用Java中的HashMap实现一个HashSet，我们只用HashMap的key就是了。

我们讲一讲intset，先看源码。

```c
typedef struct intset{
    uint32_t encoding;//编码方式

    uint32_t length;//集合包含的元素数量

    int8_t contents[];//保存元素的数组
}intset;
```

encoding有三种，分别是INTSET_ENC_INT16、INSET_ENC_INT32、INSET_ENC_INT64，代表着整数值的取值范围。Redis会根据添加进来的元素的大小，选择不同的类型进行存储，可以尽可能地节省内存空间。

length记录集合有多少个元素，这样获取元素个数的时间复杂度就是O(1)。

contents，存储数据的数组，数组按照从小到大有序排列，不包含任何重复项。

![](https://gitee.com/QingHui/picGo-img-bed/raw/master/img/image-20210910114248870.png)

这里我们可能会提出疑问，如果一开始存的是INTSET_ENC_INT16(范围在-32,768~32,767)，如果这时添加了一个40000的数，怎么升级为INSET_ENC_INT32呢？

升级过程是这样的：

1、根据新元素的类型扩展数组contents的空间。

2、从尾部将数据插入。

3、根据新的编码格式重置之前的值，因为这时的contents存在着两种编码的值。从插入的数据的位置，也就是尾部，从后到前将之前的数据按照新的编码格式进行移动和设置。从后到前调整是为了防止数据被覆盖。

升级的优点在于，根据存储的数据大小选择合适的编码方式，节省了内存。

缺点在于，升级会消耗系统资源。而且升级是不可逆的，也就是一旦对数组进行升级，编码就会一直保持升级后的状态。

set数据类型常用的命令有：sadd、spop、smembers、sunion等等。

Redis为set类型提供了求交集，并集，差集的操作，可以非常方便地实现譬如共同关注、共同爱好、共同好友等功能。

## 5.zset(有序集合)

zset是Redis中比较有特色的数据类型，它和set一样是不可重复的，区别在于多了score值，用来代表排序的权重。也就是当你需要一个有序的，不可重复的集合列表时，就可以考虑使用这种数据类型。

zset的编码有两种，分别是：ziplist、skiplist。当zset的长度小于 128，并且所有元素的长度都小于 64 字节时，使用ziplist存储；否则使用 skiplist 存储。

这里要讲一下skiplist，也就是跳跃表。它的底层实现比较复杂，这里简单地提一下。

![](https://gitee.com/QingHui/picGo-img-bed/raw/master/img/image-20210910114314351.png)

跳跃表的数据结构如上图所示，为什么要设计成这样呢？好处在于查询的时候，可以减少时间复杂度，如果是一个链表，我们要插入并且保持有序的话，那就要从头结点开始遍历，遍历到合适的位置然后插入，如果这样性能肯定是不理想的。

所以问题的关键在于**能不能像使用二分查找一样定位到插入的点**，答案就是使用跳跃表。比如我们要插入38，那么查找的过程就是这样。

首先从L4层，查询87，需要查询1次。

然后到L3层，查询到在->24->87之间，需要查询2次。

然后到L2层，查询->48，需要查询1次。

然后到L1层，查询->37->48，查询2次。确定在37->48之间是插入点。

有没有发现经过L4，L3，L2层的查询后已经跳过了很多节点，当到了L1层遍历时已经把范围缩小了很多。这就是跳跃表的优势。这种方式有点类似于二分查找，所以他的时间复杂度为**O(logN)**。

其实生活中也有这种例子，类似于快递填写的地址是省->市->区->镇->街，当快递公司在送快递时就根据地址层层缩小范围，最终锁定在一个很小的区域去搜索，提高了效率。

zet常用的命令有：zadd、zrange、zrem、zcard等。

zset的特点非常适合应用于开发排行榜的功能，比如三天阅读排行榜，游戏排行榜等等。