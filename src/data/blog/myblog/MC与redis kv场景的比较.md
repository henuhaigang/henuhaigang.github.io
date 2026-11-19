
---
layout:     post
title:      mc redis  对比及相关知识记录
pubDatetime: 2026-10-29T00:59:04.866Z
modDatetime: 2026-10-29T00:59:04.866Z
author:     henuhaigang
header-img: img/post-bg-algorithm.jpg
catalog: true
tags:
    - 分布式缓存
    - mc
    - redis
    
description: 分布式缓存
---
---
你这段描述实际上涉及一个非常经典的后端架构问题：

**为什么 Memcached（MC）在简单 Key-Value 缓存场景下，可能比 Redis 有更高的吞吐量？为什么 Redis 功能更丰富，但在某些纯缓存读取场景下反而没有 Memcached 快？**

这里面最核心的区别并不是简单的「Redis 单线程，Memcached 多线程」，而是两者在以下几个方面的设计理念不同：

1. 内存管理：Memcached 的 Slab Allocator 与 Redis 的动态内存分配。
2. 数据组织：Memcached 的 Item/Chunk 组织方式与 Redis 的对象、字典组织方式。
3. 内存访问：内存定位、指针访问、CPU Cache 命中率。
4. 并发模型：Memcached 多线程与 Redis 命令执行模型。
5. 内存碎片：为什么 Memcached 会出现有空闲内存却无法存储数据。
6. 适用场景：什么业务适合 Memcached，什么业务适合 Redis。

我从底层内存布局开始讲，逐步深入到 CPU、线程模型和实际业务架构。

# **一、先理解两者最根本的设计差异**

先看一个简单场景。

假设我们有一个缓存集群，需要存储 1000 万条数据：

```
key1 -> value1
key2 -> value2
key3 -> value3
...
key10000000 -> value10000000
```

每条数据都是简单的字符串 Key-Value，没有 List、Set、Hash、ZSet 等复杂数据结构。

两者的设计思路：

**Memcached：以固定规格的内存块为核心**

**内存池 / Slab Allocator**

**Class 1**

固定大小 Chunk

Chunk 1

Chunk 2

Chunk 3

**Class 2**

固定大小 Chunk

Chunk 1

Chunk 2

Chunk 3

**Class 3**

固定大小 Chunk

Chunk 1

Chunk 2

Chunk 3

不同规格的内存块分别管理，Item 放入适合自己的 Chunk。

核心思想：空间规格化，降低动态分配的复杂度。

**Redis：以动态对象和内存分配为核心**

**Redis 数据字典**

Hash Table

Bucket 1

指向对象

Bucket 2

指向对象

Bucket 3

指向对象

Redis Object A

SDS + 元数据

Redis Object B

SDS + 元数据

根据对象大小和类型进行内存分配，底层通常使用 jemalloc。

核心思想：提供丰富的数据结构和灵活的内存管理能力。

注意：这只是两种内存管理思路的简化示意图，Memcached 并不是完全没有 Hash Table，Redis 也不是每次读写都要重新动态分配内存。

# **二、Memcached 的核心：Slab Allocator**

这是理解 Memcached 性能的关键。

## **2.1 为什么要设计 Slab？**

先考虑一个问题：

假设我们有 1GB 内存，需要存储大量 Key-Value。

如果采用传统的动态内存分配：

```
malloc(100);
malloc(200);
malloc(500);
malloc(80);
free(...);
malloc(300);
```

不断进行内存分配、释放，会产生什么问题？

例如：

[File:External Fragmentation.svg - Wikimedia Commons](https://images.openai.com/static-rsc-4/jG64ueS-mVlXYvWfXNXENzQK1f_Zr9mgaWY_L9rIhIMSSpcjYwjUyrE832TXX95H47Xfqcy_3jQ4HgPclhdVmtjaA0h-d9QXlOWYFDwrnOunFmANPhf9y6N2lYh0RXZ_9g1kqUUyQz8NCRDy6l0EY1FM9sqtOV7IQVVyjc-4BEI?purpose=inline)

[commons.wikimedia.org](https://commons.wikimedia.org/wiki/File%3AExternal_Fragmentation.svg?utm_source=chatgpt.com)

假设内存布局如下：

```
| 100KB 已使用 | 50KB 空闲 | 200KB 已使用 |
| 80KB 空闲   | 300KB 已使用 | 100KB 空闲 |
```

现在需要申请 150KB 内存。

虽然总空闲内存有：

50 + 80 + 100 = 230KB

但是没有任何一块连续的 150KB 空间。

于是分配失败。

这就是**外部内存碎片（External Fragmentation）**。

传统 malloc/free 的内存管理在大量小对象、高频分配释放场景中，可能产生比较严重的内存碎片和管理开销。

Memcached 的设计思路是：

既然缓存对象大小变化很多，那么我干脆提前把内存组织成不同规格的内存块。

这就是 Slab Allocator。

## **2.2 Slab 到底是什么？**

Memcached 的内存管理主要分为三个层次：

```
Memcached Memory
      |
      v
  Slab Class
      |
      v
    Slab
      |
      v
    Chunk
      |
      v
    Item
```

分别解释。

**第一层：Slab Class**

Slab Class 是一个内存规格类别。

例如：

| Class   | Chunk 大小 |
| ------- | -------- |
| Class 1 | 96 B     |
| Class 2 | 120 B    |
| Class 3 | 150 B    |
| Class 4 | 187 B    |
| Class 5 | 234 B    |
| Class 6 | 292 B    |
| Class 7 | 365 B    |

上面的数字只是为了方便理解的示例。

Memcached 默认采用一定的增长因子（通常为 1.25）来生成不同规格的 Chunk。

也就是说：

```
96
96 × 1.25 = 120
120 × 1.25 = 150
150 × 1.25 = 187.5
...
```

不同版本、配置和对齐规则会影响实际规格。

**第二层：Slab**

一个 Slab 是一块实际分配的内存区域。

Memcached 默认的 Slab Page 大小通常为 1MB。

例如：

```
Class 1
    |
    +---- Slab 1 (1MB)
    +---- Slab 2 (1MB)
    +---- Slab 3 (1MB)

Class 2
    |
    +---- Slab 1 (1MB)
    +---- Slab 2 (1MB)
```

每个 Slab Page 被划分成若干个相同大小的 Chunk。

**第三层：Chunk**

Chunk 是真正分配给缓存 Item 使用的内存单元。

例如：

假设一个 Slab Page 大小为 1MB。

Chunk 大小为 100B。

那么：

\\[ \frac{1024 \times 1024}{100} \approx 10485 \\]

一个 Slab Page 可以放置约 10485 个 Chunk（这里暂时忽略页面管理开销和对齐因素）。

结构如下：

**一个 Slab Page（1MB）**

Chunk

Chunk

Chunk

Chunk

Chunk

Chunk

Chunk

Chunk

Chunk

Chunk

Chunk

Chunk

空闲

空闲

空闲

空闲

已分配

空闲

示意图：每个 Chunk 的大小完全相同，实际数量取决于 Chunk 大小。

这就解决了传统动态内存分配中大量不同尺寸对象造成的外部碎片问题。

## **2.3 Memcached 如何根据 Key-Value 大小选择 Chunk？**

假设现在有三个缓存对象：

```
Key A + Value A = 90B
Key B + Value B = 130B
Key C + Value C = 220B
```

Memcached 不会给每个对象单独申请刚好大小的内存，而是寻找一个能够容纳它的最小 Chunk。

假设规格如下：

```
Chunk 1：100B
Chunk 2：125B
Chunk 3：156B
Chunk 4：195B
Chunk 5：244B
```

分配过程：

| 对象实际大小 | 分配 Chunk | 浪费空间 |
| ------ | -------- | ---- |
| 90B    | 100B     | 10B  |
| 130B   | 156B     | 26B  |
| 220B   | 244B     | 24B  |

这就是 Slab 的内存分配策略：

**一个对象只能放入一个能够容纳它的 Chunk，不能把多个 Chunk 拼起来存放一个 Item。**

这里就出现了一个非常重要的问题：

假设一个对象需要 126B，而当前最大可用 Chunk 是 125B。

即使整个 Memcached 还有大量空闲内存，也不能将这个对象放入 125B 的 Chunk。

它必须寻找更大的 Chunk Class。

这就是你提到的「明明还有空闲内存，却无法存储」的根本原因之一。

不过这里需要区分两种碎片。

- 内部碎片：Chunk 分配给对象后，剩余的空间不能被其他对象使用。
- Class 间碎片：某个 Class 有大量空闲 Chunk，但另一个 Class 的 Chunk 已经用完。

后者正是 Memcached 典型的内存利用率问题。

## **2.4 为什么会出现内存明明空闲，却无法存储？**

这是一个非常值得深入理解的场景。

假设 Memcached 总共配置了 1GB 内存。

经过一段时间运行，形成如下状态：

**Memcached 内存分布（假设数据）**

**Class A：100B Chunk**

300MB / 300MB

已全部使用，没有空闲 Chunk

**Class B：200B Chunk**

100MB / 400MB

剩余 300MB 空闲

**Class C：400B Chunk**

200MB / 300MB

剩余 100MB 空闲

**总计**

# **400MB**

仍有约 400MB 已分配页面空间未被 Chunk 使用，但 Class A 已经没有可用 Chunk。

现在有大量 100B 左右的缓存对象需要写入。

发生什么？

```
新对象需要 Class A
       |
       v
Class A 没有空闲 Chunk
       |
       v
尝试从 Class A 的 Slab 中回收 Item
       |
       +---- 可以回收 ----> 淘汰旧数据，写入新数据
       |
       +---- 无法满足 ----> 需要获取新的 Slab Page
                                  |
                                  v
                          检查是否有可用内存
                                  |
                      +-----------+-----------+
                      |                       |
                  有可用内存                没有可用内存
                      |                       |
                分配新的 Page          需要淘汰或回收
```

关键就在于：

Class B 和 Class C 的空闲 Chunk，不能直接拿给 Class A 使用。

因为它们的尺寸规格不同。

这就是所谓的 Slab Class 之间的内存不均衡。

### **2.5 Memcached 有没有办法解决这个问题？**

有。

这里不能把 Memcached 理解成完全固定、永远不能调整的内存分区。

Memcached 支持 Slab Reassignment（Slab 页面重新分配）。

例如：

```
Class A：需要更多内存
Class B：大量空闲内存
```

可以把 Class B 的某些 Slab Page 回收，然后重新分配给 Class A。

大致过程：

```
Class B
   |
   v
选择一个 Slab Page
   |
   v
迁移或淘汰其中的 Item
   |
   v
释放 Slab Page
   |
   v
重新分配给 Class A
```

这里有一个细节：

一个 Slab Page 里面可能存在大量有效缓存数据，因此重新分配不是简单地修改一个指针就完成了。需要处理页面中的 Item，包括迁移、回收等工作。

Memcached 还有 Slab Automove 机制，可以根据不同 Class 的内存压力动态调整 Slab Page 的分配。

因此，准确的结论应该是：

**Memcached 存在 Class 间内存利用率不均衡问题，但可以通过 Slab Reassignment 和 Automove 缓解，而不是完全无法解决。**

# **三、Memcached 为什么读取速度快？**

现在进入真正有意思的地方。

你提到：

> 这种特殊结构使其在数据定位数组寻址与读写上的速度非常快。

这个说法有一定道理，但需要纠正一个细节：

**Memcached 并不是通过 Key 直接计算数组下标，然后 O(1) 读取 Value。**

它仍然需要 Hash Table。

例如：

```
Key = user:10001
```

首先计算 Hash：

```
hash = hash_func("user:10001");
```

然后通过 Hash 定位 Bucket：

```
bucket = hash & hash_table_mask;
```

找到 Bucket 后，再比较链表或者其他冲突处理结构中的 Key。

所以 Memcached 和 Redis 一样，都需要经过 Hash 查找。

真正的区别主要在于：

- Memcached 的 Item 采用紧凑的内存组织方式。
- Item 的内存来自对应的 Slab Chunk。
- Chunk 的规格固定，避免了每个对象都进行独立的通用内存分配。
- Memcached 的线程模型可以充分利用多核 CPU。

我们把整个读取流程拆开。

## **3.1 Memcached 的读取过程**

假设：

```
get user:10001
```

内部可以简化为：

**客户端发送 GET 请求**

**网络线程接收请求**

**计算 Key 的 Hash 值**

**定位 Hash Bucket**

**查找 Item，比较 Key**

**读取 Value 并返回客户端**

注意：

Memcached 并不是完全没有指针访问，也不是整个读取过程都是数组下标访问。

Hash Bucket 到 Item 仍然可能涉及指针跳转。

但是，Slab 带来的好处是：内存分配和对象管理相对简单，内存布局也比较规整。

## **3.2 Chunk 为什么能提高性能？**

这里涉及一个非常重要的计算机原理：

**CPU Cache（CPU 缓存）和内存局部性。**

CPU 访问数据时，并不是每次都直接访问物理内存。

它有多级缓存：

```
CPU Core
   |
   +---- L1 Cache
   |
   +---- L2 Cache
   |
   +---- L3 Cache
   |
   +---- Main Memory
```

CPU Cache 的访问速度远高于内存。

通常 CPU Cache Line 为 64 Bytes（常见 x86 平台）。

假设一个 Item 的元数据和部分 Key 位于相邻内存位置。

CPU 读取其中一部分时，可能顺便把周围的数据加载到 Cache Line。

后续访问这些相邻数据，就有机会直接命中 CPU Cache。

Slab 的固定规格分配和较为紧凑的 Item 布局，有利于改善内存局部性。

但这里要注意：

**固定 Chunk 不等于 CPU Cache 一定命中。**

Hash Table 本身可能发生随机访问，Item 也可能分布在不同 Slab Page 中。

因此，不能简单认为 Memcached 只因为 Slab 就一定比 Redis 有更好的 CPU Cache 命中率。

# **四、Redis 为什么在简单 Key-Value 读取上可能更慢？**

我们先看 Redis 的内部数据结构。

Redis 并不是简单地把 Key 和 Value 放到一个数组里面。

它需要维护一套对象体系。

以：

```
SET user:10001 Tom
```

为例。

Redis 内部大致涉及以下结构：

```
Redis DB
   |
   v
dict (Hash Table)
   |
   v
dictEntry
   |
   +---- key
   |      |
   |      v
   |    RedisObject
   |      |
   |      v
   |    SDS
   |
   +---- value
          |
          v
       RedisObject
          |
          v
         SDS
```

这里需要解释几个核心结构。

## **4.1 RedisObject**

Redis 中有一个非常重要的结构：

```
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS;
    int refcount;
    void *ptr;
} robj;
```

这是 Redis 对象体系中的典型结构示意，具体字段会随 Redis 版本变化。

它包含：

| 字段       | 作用              |
| -------- | --------------- |
| type     | 数据类型            |
| encoding | 底层编码            |
| lru      | LRU/LFU 等淘汰相关信息 |
| refcount | 引用计数            |
| ptr      | 指向底层数据          |

为什么 Redis 需要 RedisObject？

因为 Redis 不只是一个简单的 Key-Value 缓存。

它支持：

```
String
List
Hash
Set
Sorted Set
Stream
```

而且每种数据结构还可能存在不同的底层编码。

例如：

- List 可以使用 quicklist。
- Hash 可以使用 listpack 或 hashtable。
- Set 可以使用 intset 或 hashtable。
- Sorted Set 可以使用 listpack 或 skiplist + dict。

Redis 通过统一的对象抽象管理这些数据类型。

这就带来了额外的元数据和管理成本。

当然，不是所有 Redis 数据都需要相同数量的内存分配，整数编码、共享对象、紧凑编码等机制都能减少开销。

## **4.2 SDS：Redis 的字符串结构**

Redis 的字符串并不是简单的 C 字符串。

它采用 SDS（Simple Dynamic String）。

例如：

```
struct sdshdr {
    size_t len;
    size_t alloc;
    unsigned char flags;
    char buf[];
};
```

这里是概念化的结构示意，实际 Redis 使用多种不同宽度的 SDS Header。

为什么不直接使用 C 字符串？

C 字符串有几个问题：

1. 需要通过 `\0` 判断字符串结束。
2. 获取字符串长度需要遍历，时间复杂度 O(N)。
3. 字符串修改可能涉及内存重新分配。
4. 无法直接安全地存储包含 `\0` 的二进制数据。

SDS 通过维护长度和分配空间等信息解决这些问题。

但是也意味着 Redis 字符串存在一定的元数据开销。

## **4.3 Redis 的 Hash Table**

Redis 的数据库核心结构之一是 Dict。

可以简化理解为：

```
typedef struct dictEntry {
    void *key;
    void *v;
    struct dictEntry *next;
} dictEntry;
```

每个 DictEntry 里面：

- key 指向 Key 对象。
- v 指向 Value 对象。
- next 用于解决 Hash 冲突。

因此一次读取可能涉及：

```
Hash Table Bucket
      |
      v
   dictEntry
      |
      +---- key 指针
      |
      +---- value 指针
                 |
                 v
             RedisObject
                 |
                 v
                SDS
```

这意味着可能存在多次指针跳转。

如果数据没有位于 CPU Cache 中，就可能产生更多的内存访问延迟。

另外，Redis 还需要考虑：

- 字典扩容。
- 渐进式 Rehash。
- 对象引用计数。
- 过期时间管理。
- 淘汰策略。
- RDB/AOF 持久化。
- 复制与集群相关功能。

这些都是额外的系统管理成本。

但是需要强调：这些功能并不意味着每次 GET 都会完整执行一遍。很多功能只在特定条件下发生。

**所以 Redis 在简单字符串读取时，可能比 Memcached 有更多的数据结构和内存管理开销，但不能把 Redis 的性能差距简单归因于 RedisObject 和 SDS。**

# **五、一个更重要的区别：Memcached 多线程，Redis 命令执行通常是单线程**

这部分对理解高并发性能非常重要。

很多人认为：

> Memcached 比 Redis 快，是因为 Memcached 使用多线程，而 Redis 使用单线程。

这个结论有一定道理，但不够完整。

我们需要理解两者的并发模型。

## **5.1 Memcached 的多线程模型**

Memcached 采用多线程架构。

其典型架构包括：

- Main Thread：负责监听连接、接收新连接。
- Worker Threads：负责处理客户端请求。
- Libevent：用于事件驱动网络处理。

可以简化为：

**Main Thread**

监听端口、Accept 新连接

**Worker Threads**

**Worker 1**

独立事件循环

处理请求

**Worker 2**

独立事件循环

处理请求

**Worker 3**

独立事件循环

处理请求

**共享缓存数据和内存管理结构**

例如，服务器有 16 个 CPU 核心。

可以配置 Memcached 使用多个 Worker Thread。

当有大量客户端请求时：

```
请求 A ---> Worker 1
请求 B ---> Worker 2
请求 C ---> Worker 3
请求 D ---> Worker 4
...
请求 N ---> Worker 16
```

多个线程可以同时执行缓存操作。

这意味着 Memcached 能够充分利用多核 CPU。

当然，Memcached 内部并不是所有操作都没有锁。共享 Hash Table、Item、Slab 等结构需要相应的并发保护，锁竞争也可能成为瓶颈。

## **5.2 Redis 的单线程模型**

Redis 的经典模型是：

```
多个客户端
     |
     v
网络事件循环
     |
     v
接收请求
     |
     v
解析命令
     |
     v
执行命令
     |
     v
返回结果
```

Redis 主要通过 I/O 多路复用实现网络连接管理。

例如 Linux 下的 epoll：

```
Client 1 ----\
Client 2 -----\
Client 3 ------> epoll
Client 4 -----/     |
Client 5 ----/      v
                 Redis Event Loop
                       |
                       v
                   执行命令
```

假设同时有 10 万个客户端连接。

Redis 不需要为每个连接创建一个线程。

而是通过 epoll 监听大量连接的读写事件。

当某个连接有数据到达时，Redis 才处理对应的请求。

这里有一个非常重要的区别：

**I/O 多路复用解决的是大量网络连接的管理问题，不等于命令执行可以并行。**

经典 Redis 命令执行模型中，同一时刻通常只有一个主执行线程在执行命令。

例如：

```
GET user:10001
GET user:10002
GET user:10003
```

这些命令通常由主线程依次执行。

因此：

- 不需要为普通命令执行设计复杂的多线程同步机制。
- 命令执行天然具有串行化特征。
- 单条命令执行期间不会被另一条普通命令插入执行。
- 但是 CPU 密集型命令可能阻塞其他客户端请求。

### **5.3 Redis 6 以后是不是也支持多线程？**

支持。

Redis 6 引入了 I/O Threads。

但这里要区分：

| 工作内容      | 经典 Redis | Redis 6+     |
| --------- | -------- | ------------ |
| 网络连接管理    | 主线程事件循环  | 主线程事件循环      |
| Socket 读取 | 主线程      | 可由 I/O 线程分担  |
| 协议解析      | 主要由主线程处理 | 部分 I/O 工作可并行 |
| 命令执行      | 主线程      | 仍然主要由主线程     |
| 命令结果写回    | 主线程      | 可由 I/O 线程分担  |

Redis 6 的多线程 I/O 并没有把普通命令执行改造成多线程并行执行。

所以：

```
Memcached：
多个 Worker 可以并行处理缓存操作

Redis：
多个 I/O 线程可以帮助处理网络 I/O
核心命令执行仍然主要由主线程串行完成
```

这也是为什么在纯粹的简单 Key-Value 场景中，Memcached 有机会利用多核 CPU 获得更高的总体吞吐量。

不过，Redis 也有多种性能优化，例如 Pipeline、I/O Threads、批量操作，以及不同版本中的网络栈优化。

不能只通过线程数量判断实际性能。

# **六、为什么 Redis 单线程反而有优势？**

这里涉及一个经常在架构师面试中出现的问题：

**既然多线程能提高吞吐量，为什么 Redis 不直接把命令执行也改成多线程？**

因为多线程不是没有成本的。

假设有一个共享 Hash Table：

```
线程 1：GET user:10001
线程 2：SET user:10001 Tom
线程 3：DEL user:10001
```

如果多个线程同时访问、修改共享数据结构，就需要考虑：

1. 锁竞争。
2. 数据结构并发安全。
3. 内存释放时机。
4. 引用计数并发修改。
5. 事务执行顺序。
6. 多命令原子性。
7. 死锁与锁粒度。

例如：

```
synchronized (cache) {
    // 查询缓存
}
```

如果所有缓存操作都使用同一把锁，那么多线程虽然存在，但实际还是串行执行。

如果使用细粒度锁，又会增加实现复杂度。

Redis 的思路是：

**把核心命令执行串行化，减少共享数据结构的并发控制成本。**

例如：

```
MULTI
SET account:1 100
SET account:2 200
EXEC
```

Redis 能够保证事务中的命令按照顺序执行，不会被其他客户端的普通命令插入。

这里需要强调：

Redis 的事务不等于关系数据库事务，不提供传统数据库那样的回滚机制。

但其命令串行执行模型确实使很多原子操作更容易实现。

## **6.1 单线程为什么能够跑得很快？**

因为 Redis 的主要工作很多时候不是 CPU 密集型计算，而是：

- Hash 查找。
- 内存读取。
- 简单数据结构操作。
- 网络数据读写。

这些操作本身比较轻量。

另外，Redis 避免了普通命令执行过程中大量的线程切换和锁同步。

但是，如果命令需要处理很大的数据量，例如：

```
KEYS *
SMEMBERS huge_set
SORT huge_list
```

就可能消耗大量 CPU 时间，阻塞其他命令。

所以 Redis 的单线程既是优势，也是限制。

# **七、你提到的「Redis 一致性更好」，这个说法需要重新理解**

这句话很容易产生误解。

Memcached 和 Redis 都是缓存系统。

Redis 的单线程命令执行确实具有很好的原子性特征，但不能直接说 Redis 的数据一致性就一定比 Memcached 好。

我们区分三个概念。

### **7.1 命令原子性**

Redis：

```
INCR counter
```

这个操作由主线程执行，不会被其他普通命令插入。

Memcached：

```
incr counter 1
```

同样支持原子的计数操作。

因此，不能说 Memcached 的单条命令不具备原子性。

### **7.2 并发修改一致性**

例如：

```
线程 A：读取 value = 100
线程 B：读取 value = 100
线程 A：写入 value = 110
线程 B：写入 value = 120
```

这可能产生更新覆盖问题。

无论使用 Redis 还是 Memcached，如果业务使用 GET + 修改 + SET 这种多个独立命令实现业务操作，都需要考虑并发竞争。

Redis 可以使用 Lua、事务或其他原子操作来解决。

Memcached 可以使用 CAS（Check And Set）机制解决乐观并发更新问题。

### **7.3 分布式数据一致性**

再考虑：

```
Redis Master
     |
     v
Redis Replica
```

主从复制是异步的常见模式。

主节点写入成功，不代表所有从节点已经同步。

Memcached 本身也没有 Redis 那样内置的主从复制、持久化和集群数据复制体系。

所以如果讨论的是分布式一致性，Redis 也不能简单地被认为是强一致系统。

我的建议是：面试时不要说「Redis 单线程，所以一致性更好」。

更准确的说法是：

> Redis 通过单线程执行模型，天然简化了单实例内部命令的串行化和原子性实现；Memcached 通过多线程提高并发处理能力，同时使用锁、CAS 等机制处理并发访问。但两者的分布式一致性需要另外分析，不能由单线程或多线程直接推导。

# **八、两者的内存管理机制对比**

现在把前面的内容放在一起。

| 对比维度   | Memcached            | Redis                              |
| ------ | -------------------- | ---------------------------------- |
| 核心内存管理 | Slab Allocator       | jemalloc 等分配器                      |
| 内存组织   | Slab Page、Chunk、Item | RedisObject、SDS、Dict 等             |
| 内存分配   | 按 Chunk Class 分配     | 根据对象和分配器规格动态分配                     |
| 外部碎片   | Slab 机制减少传统外部碎片      | 可能出现内存碎片                           |
| 内部碎片   | Chunk 尺寸不匹配产生        | 分配器尺寸级别及对象结构产生                     |
| 数据结构   | 主要是简单 Key-Value      | String、Hash、List、Set、ZSet、Stream 等 |
| 网络处理   | 多线程事件驱动              | I/O 多路复用，可配置 I/O 多线程               |
| 命令执行   | 多 Worker 并行          | 主要由主线程串行执行                         |
| 并发控制   | 需要处理共享数据结构的并发访问      | 单实例核心命令执行简化了并发控制                   |
| CAS    | 支持                   | 有 WATCH 等机制，语义不同                   |
| 持久化    | 不提供 Redis 那样的内置持久化体系 | RDB、AOF                            |
| 复制     | 无内置 Redis 式复制体系      | 主从复制                               |
| 复杂数据结构 | 不支持 Redis 式复杂结构      | 丰富                                 |

## **8.1 Redis 为什么也会出现内存碎片？**

Redis 使用 jemalloc 等内存分配器。

jemalloc 也不是简单地每次申请多少就分配多少。

它同样会按照不同的 Size Class 管理内存。

例如：

```
申请 100B
实际分配 112B

申请 200B
实际分配 224B

申请 300B
实际分配 320B
```

具体规格取决于分配器版本和配置，这里只是示意。

Redis 还需要管理：

- RedisObject。
- SDS。
- DictEntry。
- 数据结构节点。
- 过期字典。
- 其他内部元数据。

因此，一个业务 Value 的实际内存占用，往往远远大于 Value 本身的字节数。

例如：

```
业务数据：
Key = user:10001
Value = Tom
```

实际内存可能涉及：

```
DictEntry
RedisObject(Key)
SDS(Key)

RedisObject(Value)
SDS(Value)
```

此外还有 Hash Table 的 Bucket、分配器元数据、对齐空间等。

所以，如果你存储的是几千万条只有几十字节的简单 Key-Value，Redis 的对象管理开销可能相当可观。

不过需要注意，Redis 的紧凑编码、整数编码以及 jemalloc 的优化也能有效降低这些开销，不能简单地认为每个 Redis Key-Value 都必然产生固定数量的独立 malloc。

# **九、一个实际业务场景：1000 万用户信息缓存**

假设我们有一个用户系统：

```
Key:
user:10001

Value:
{"name":"Tom","age":20}
```

现在有 1000 万用户。

每个 Value 大约 100B。

我们暂时忽略 Key、对象元数据、分配器开销，只计算业务数据：

\\[ 10000000 \times 100B \approx 1GB \\]

实际上，整个缓存系统的内存需要远大于 1GB。

现在考虑两种方案。

## **方案 A：Memcached**

```
                    Application
                         |
                         v
                  Memcached Client
                         |
              +----------+----------+
              |          |          |
              v          v          v
            MC-1       MC-2       MC-3
              |          |          |
           Slab       Slab       Slab
           Class      Class      Class
```

优点：

1. 简单 Key-Value 读写。
2. 多线程利用多核 CPU。
3. Slab 减少通用动态内存分配开销。
4. 没有 Redis 那样复杂的数据结构体系。
5. 适合大量简单缓存对象。

缺点：

1. 不同 Slab Class 之间可能出现内存利用不均衡。
2. Value 大小跨度大时，内部碎片可能增加。
3. 不具备 Redis 那样丰富的数据操作能力。
4. 没有内置的数据持久化和复制体系。
5. 缓存节点故障后的数据恢复通常需要依赖上游数据库重新加载。

## **方案 B：Redis**

```
                    Application
                         |
                         v
                     Redis Client
                         |
              +----------+----------+
              |          |          |
              v          v          v
           Redis-1   Redis-2    Redis-3
              |          |          |
           Dict       Dict       Dict
           Object     Object     Object
           SDS        SDS        SDS
```

优点：

1. 丰富的数据结构。
2. 支持原子操作、Lua 等。
3. 支持 RDB、AOF。
4. 支持主从复制。
5. 支持过期策略和多种淘汰策略。
6. 适合复杂业务缓存。

缺点：

1. 对象和数据结构管理存在额外开销。
2. 主线程执行命令可能成为 CPU 瓶颈。
3. 复杂数据结构可能产生额外的内存和操作成本。
4. 大量小对象也可能造成较高的内存管理开销。

# **十、到底什么情况下应该选择 Memcached？**

我给你一个实际的架构选型思路。

更适合 Memcached

**场景一：海量简单 KV 读取**

- 用户基本信息缓存
- 商品简单属性缓存
- 页面片段缓存
- 大量无复杂操作的临时数据

更适合 Memcached

**场景二：Value 大小比较稳定**

- 对象结构相对固定
- Key-Value 大小分布集中
- 读取远多于复杂更新
- 不依赖复杂数据结构

更适合 Redis

**场景三：需要原子操作**

- 秒杀库存
- 计数器
- 限流器
- 分布式锁
- 排行榜

更适合 Redis

**场景四：需要复杂数据结构**

- 用户关注关系
- 粉丝集合
- 消息队列
- 延迟任务
- 实时排行榜

还有一点值得注意：如果业务要求缓存节点故障后能够通过持久化恢复数据，Redis 的 RDB/AOF 和复制能力也会是重要的选型因素。

# **十一、再深入一层：如果让我从架构师角度做性能选型**

假设现在公司有一个业务：

- 5000 万用户。
- 每个用户有一个简单的用户信息缓存。
- Value 大小基本在 100B～200B。
- 读 QPS：100 万。
- 写 QPS：5 万。
- 读写都是简单 GET、SET。
- 不需要 List、Set、Hash 等复杂数据结构。

我会考虑 Memcached。

但是不会仅凭理论就直接决定。

我会做以下几个测试：

| 测试维度        | 测试目的             |
| ----------- | ---------------- |
| GET 吞吐量     | 比较纯读取能力          |
| SET 吞吐量     | 比较写入能力           |
| P99 延迟      | 比较尾延迟            |
| CPU 利用率     | 比较多核利用能力         |
| RSS 内存      | 比较真实内存占用         |
| 内存碎片        | 比较内存利用率          |
| 不同 Value 大小 | 比较 Slab Class 分布 |
| 高并发连接       | 比较网络线程模型         |
| 节点故障        | 比较数据恢复能力         |

特别需要测试不同 Value 大小的分布：

```
测试 A：
90% 的 Value = 100B
10% 的 Value = 120B

测试 B：
Value 从 100B 到 10KB 随机分布

测试 C：
Value 从 100B 到 1MB 随机分布
```

这三种场景下，Memcached 的 Slab 内存利用情况可能差异很大。

另外还要测试 Memcached 的 Slab Automove、Chunk 增长因子、最大 Item 大小，以及 Redis 的 jemalloc 碎片率。

最终看的是：

\\[ \text{实际性能} = f(\text{数据分布},\text{并发模型},\text{内存布局},\text{网络},\text{CPU}) \\]

而不是简单比较两个产品的理论性能。

# **十二、最后总结：真正应该记住的几个原理**

我把整个问题浓缩成五个知识点，这几个知识点也是面试时最值得展开的地方。

**1. Memcached 为什么适合简单 KV？**

Slab Allocator 将内存划分为不同规格的 Slab Class，每个 Slab Page 再划分为固定大小的 Chunk。降低了通用动态内存分配的复杂度，同时采用多线程处理请求，能够充分利用多核 CPU。

**2. 为什么 Memcached 会出现有空闲内存却无法存储？**

因为不同 Slab Class 的 Chunk 不能直接混用。某个 Class 没有可用 Chunk 时，其他 Class 的空闲 Chunk 不一定能满足它的分配需求。这是 Class 间内存利用不均衡问题，可以通过 Slab Reassignment 缓解。

**3. Redis 为什么在简单 KV 上可能有额外开销？**

Redis 使用统一的对象抽象和丰富的数据结构，涉及 Dict、RedisObject、SDS 等结构，同时需要进行对象管理、引用计数和内存分配。对于纯简单 KV，这些能力可能带来不必要的额外开销。

**4. Memcached 为什么能够获得更高吞吐量？**

其多 Worker 并发模型可以利用多核 CPU，Slab 机制也能降低一部分内存分配开销。但最终性能取决于具体数据分布、并发程度、网络和硬件，不能认为 Memcached 在所有 KV 场景都比 Redis 快。

**5. Redis 单线程为什么仍然有价值？**

它简化了单实例核心数据结构的并发控制，命令执行天然串行化，容易实现原子操作。但是单线程也可能在 CPU 密集型命令下成为性能瓶颈。

## **十三、我建议你继续研究的三个底层问题**

结合 Java 后端和架构师的知识体系，我认为接下来有三个问题特别值得深入，它们之间还有很强的关联：

1. **Slab Allocator 与 jemalloc 到底有什么区别？** 为什么 Redis 也使用内存分配器的 Size Class，却没有完全采用 Memcached 的 Slab 管理模式？
2. **CPU Cache Line、内存局部性、伪共享（False Sharing）与锁竞争**：为什么内存布局会直接影响高并发缓存系统的 QPS？这部分可以深入到 CPU 缓存一致性协议和 CAS。
3. **Redis 与 Memcached 的多线程性能对比**：如果 Redis 也采用多线程执行命令，理论上是否就能超过 Memcached？为什么 Redis 选择了现在的架构？

我尤其建议研究第 2 个问题。它能把你之前学习的 Java 并发、CPU 缓存、内存屏障、锁竞争和高并发架构真正串联起来。

这三个问题其实可以串成一条完整的技术知识链：

**内存分配器 → 内存布局 → CPU Cache → 多线程并发 → 锁竞争 → 缓存系统吞吐量。**

如果把这条链路真正理解了，你就不仅能解释 Memcached 为什么快，还能理解 Redis 为什么采用单线程命令执行、为什么 Netty 要使用 EventLoop、为什么 Disruptor 要设计 RingBuffer，以及为什么高并发系统经常需要减少锁和对象分配。

我按照三个专题讲解，并尽量从底层原理过渡到 Java 实际开发。

# **专题一：Slab Allocator 与 jemalloc 到底有什么区别？**

这是理解 Memcached 和 Redis 内存管理的基础。

## **1.1 首先理解：为什么需要内存分配器？**

假设一个缓存系统需要保存 1000 万条数据。

```
user:10001 -> Tom
user:10002 -> Jack
user:10003 -> Alice
...
```

每条数据的大小可能不同。

例如：

```
对象 A：100B
对象 B：120B
对象 C：150B
对象 D：300B
对象 E：1024B
```

如果每次申请内存都直接向操作系统申请，会产生非常大的开销。

因为操作系统的内存管理涉及虚拟内存、页表、物理页等机制。

所以通常不会每次 `malloc()` 都直接向操作系统申请内存。

而是：

```
应用程序
    |
    v
malloc / free
    |
    v
用户态内存分配器
    |
    v
预先管理的内存区域
    |
    v
必要时向操作系统申请更多内存
```

这里的内存分配器，就是我们需要理解的东西。

它负责：

- 内存申请。
- 内存释放。
- 内存复用。
- 内存碎片管理。
- 不同规格对象的分配。
- 多线程分配时的并发管理。

常见的内存分配器包括：

| 分配器            | 典型使用场景               |
| -------------- | -------------------- |
| glibc malloc   | Linux C/C++ 程序       |
| jemalloc       | Redis 常见构建配置、其他高性能服务 |
| tcmalloc       | Google 生态及其他高性能系统    |
| mimalloc       | Microsoft 开源内存分配器    |
| Slab Allocator | Memcached 的对象内存管理机制  |

注意：Slab Allocator 与 jemalloc 并不是完全同一层面的概念。Slab 是一种内存管理思想和机制，jemalloc 是一个完整的通用内存分配器，内部也采用 Size Class、Slab 等类似技术。

## **1.2 Slab Allocator：提前把内存分成不同规格**

假设我们给 Memcached 分配 1GB 内存。

它不会将 1GB 切成 1000 万个完全不同大小的区域。

而是按照对象大小划分不同的 Class。

例如：

```
Class 1：96B
Class 2：120B
Class 3：150B
Class 4：187B
Class 5：234B
Class 6：292B
Class 7：365B
```

每个 Class 有自己的 Slab Page。

一个 Page 又被分成多个相同大小的 Chunk。

[Journey to the centre of memcached | by Skyscanner Engineering | Medium](https://images.openai.com/static-rsc-4/zfQ5cX3AqQQGu2UG9j65q64BRUXFcrmunZowto7AL-9Zy2Ix25CggbD7ChJCXnPtQAQ-qIRdty_jISQsXrme3HiUS6aRxuWxeK309xbvKU4sAyACjreWMeQ-PhGcpHHwiWWh7RvEgcQXGwv28XboejKWuPyiVIy5eR6v2whxQWs?purpose=inline)

[medium.com](https://medium.com/%40SkyscannerEng/journey-to-the-centre-of-memcached-b239076e678a?utm_source=chatgpt.com)

例如：

```
Class 1
  |
  +--- Slab Page 1
  |       |
  |       +--- Chunk 1
  |       +--- Chunk 2
  |       +--- Chunk 3
  |
  +--- Slab Page 2
          |
          +--- Chunk 1
          +--- Chunk 2
          +--- Chunk 3
```

当来了一个 110B 的对象：

```
110B
  |
  v
寻找最小可用 Chunk
  |
  v
120B Chunk
  |
  v
分配
```

它不会申请 110B，而是直接使用 120B 的 Chunk。

优点：

1. 分配速度快。
2. 不需要每次寻找任意大小的连续内存。
3. 减少通用 malloc/free 的管理开销。
4. 同一个 Class 的 Chunk 大小一致，管理简单。

缺点：

1. 对象大小不匹配会浪费空间。
2. 不同 Class 之间可能出现内存分布不均衡。
3. 对象大小跨度特别大时，内存利用率可能降低。

这里你需要记住一个概念：

**Slab 通过牺牲一部分空间利用率，换取更简单、更快速、更可预测的对象内存分配。**

## **1.3 jemalloc：它为什么不只是简单的 Slab？**

Redis 常见的内存分配器是 jemalloc。

jemalloc 也是按照不同大小管理内存，但它的管理体系更加完整。

可以简化为：

```
jemalloc
   |
   +--- Small Allocation
   |       |
   |       +--- Size Class
   |       +--- Slab
   |
   +--- Large Allocation
   |       |
   |       +--- Extent
   |
   +--- Huge Allocation
           |
           +--- 独立的大块内存管理
```

这里涉及几个重要概念。

### **A. Size Class**

jemalloc 会把不同大小的内存申请映射到不同规格。

例如：

```
申请 100B
  |
  v
分配 112B

申请 200B
  |
  v
分配 224B
```

具体尺寸依版本、架构和配置而定。

这和 Memcached 的 Chunk Class 思想类似。

### **B. Arena**

这是 jemalloc 一个非常重要的概念。

Arena 可以理解为一套相对独立的内存管理区域。

为什么需要 Arena？

假设 32 个线程同时申请内存：

```
Thread 1 ----\
Thread 2 -----\
Thread 3 ------> 同一个全局内存分配锁
Thread 4 -----/
...
Thread 32 ---/
```

如果所有线程都访问同一个全局内存管理结构，就容易出现锁竞争。

jemalloc 使用多个 Arena 来降低这种竞争。

例如：

```
                  jemalloc
                     |
       +-------------+-------------+
       |             |             |
       v             v             v
    Arena 1       Arena 2       Arena 3
       |             |             |
    Thread 1      Thread 2      Thread 3
```

实际线程和 Arena 的关联不是简单的一对一，还可能受线程缓存、配置和绑定策略影响。

核心思想：

**把一个巨大的共享内存管理结构拆成多个相对独立的管理区域，减少并发分配时的锁竞争。**

### **C. Thread Cache / TCache**

这是 jemalloc 提升多线程分配性能的重要机制。

假设线程 A 需要申请 64B 内存。

如果每次都去 Arena 中申请：

```
Thread A
   |
   v
Arena
   |
   v
查找可用内存
   |
   v
分配
```

就可能涉及锁和共享数据结构访问。

而使用线程本地缓存后：

```
Thread A
   |
   v
TCache
   |
   +--- 有空闲 64B 对象？
             |
         +--- 有：直接分配
         |
         +--- 无：向 Arena 获取一批
```

这就是批量获取、局部复用的思想。

线程可以优先使用自己缓存的空闲对象，降低访问共享分配器的频率。

注意，TCache 不是无限增长的，它有容量限制，也需要处理缓存回收和线程退出等问题。

## **1.4 两者的本质区别**

| 对比    | Memcached Slab             | jemalloc                |
| ----- | -------------------------- | ----------------------- |
| 定位    | 缓存 Item 内存管理               | 通用动态内存分配器               |
| 内存规格  | Slab Class                 | Size Class              |
| 内存组织  | Page / Chunk               | Arena / Slab / Extent 等 |
| 多线程优化 | 结合 Memcached 的 Worker 和锁机制 | Arena、TCache 等          |
| 小对象分配 | 按固定 Chunk                  | 按 Size Class            |
| 大对象   | 受 Item 和 Class 管理限制        | 有独立的大块分配路径              |
| 内存复用  | Slab Page、Chunk 复用         | 多层缓存和 extent 管理         |
| 适用范围  | Memcached 缓存对象             | 通用 C/C++ 动态内存           |

### **这里有一个容易忽略的问题**

既然 jemalloc 也使用 Size Class 和 Slab，为什么 Redis 不直接采用 Memcached 的 Slab 方案？

实际上，Redis 使用 jemalloc 时，已经在底层使用了类似的分级分配思想。

区别在于：

- Memcached 的 Slab 直接服务于缓存 Item。
- jemalloc 是通用分配器，需要服务 Redis 各种大小、各种生命周期的对象。
- Redis 的数据结构和对象可能不断变化，需要更加灵活的内存分配。
- Redis 需要处理对象释放、字典扩容、编码转换等多种场景。

因此不是「Redis 没有 Slab」，而是两者在不同层次上管理内存。

# **专题二：CPU Cache Line、内存局部性、伪共享和锁竞争**

这是我认为最值得深入的部分。

它可以解释为什么两个算法时间复杂度都是 O(1)，实际性能却可能相差很大。

## **2.1 CPU 为什么需要 Cache？**

我们先从 CPU 访问内存讲起。

CPU 的执行速度非常快，但是主内存的访问延迟相对较高。

如果 CPU 每次读取数据都必须等待主内存，那么大量时间都会浪费在等待上。

于是 CPU 引入了多级缓存：

[Medium](https://images.openai.com/static-rsc-4/AIxXqKrLT4M6ZOO3UzxNWqQluMLPT6TLnouV7DwvLBOt0Gzg8URKwVN7WhL8kNoIRXirGBGMM9J94toU92N_2XyQ94pl-AGyiAuxnb8VNaf2R_72-AGibVSh3TOFUmJZdoCGRnViKvRs5M-areohRn70Re6x99LhKxGEpzotP_w?purpose=inline)

[tonylixu.medium.com](https://tonylixu.medium.com/operating-system-hierarchy-of-memory-81dfae60f145?utm_source=chatgpt.com)

典型结构：

```
CPU Core 0
  |
  +--- L1 Cache
  |
  +--- L2 Cache

CPU Core 1
  |
  +--- L1 Cache
  |
  +--- L2 Cache

       |
       v

    L3 Cache
       |
       v
   Main Memory
```

一般来说：

| 存储层次     | 特点               |
| -------- | ---------------- |
| L1 Cache | 最快，容量最小，通常每个核心独立 |
| L2 Cache | 容量更大，通常每个核心独立    |
| L3 Cache | 通常多个核心共享         |
| 主内存      | 容量大，访问延迟高        |

具体结构依 CPU 型号而不同。

这里要记住：

**CPU 访问一个内存地址时，通常不是只读取这个地址，而是以 Cache Line 为单位加载数据。**

在常见 x86 服务器上，Cache Line 通常为 64B。

## **2.2 什么是 Cache Line？**

假设内存中有这样一段数据：

```
地址 0x1000

+------+------+------+------+------+
|  A   |  B   |  C   |  D   |  E   |
+------+------+------+------+------+
       一个 Cache Line（示意）
```

假设 CPU 读取变量 A。

即使只需要 A，也可能把 A 所在的整个 Cache Line 加载进 CPU Cache。

如果之后访问 B、C、D，这些数据已经在 Cache Line 中，就可能直接命中缓存。

这就是空间局部性（Spatial Locality）。

还有时间局部性（Temporal Locality）：

如果刚刚访问过某个变量，短时间内再次访问它，那么它很可能仍然在 Cache 中。

例如：

```
for (int i = 0; i < 1000000; i++) {
    sum += array[i];
}
```

数组连续存储。

CPU 访问：

```
array[0]
array[1]
array[2]
array[3]
...
```

这种访问模式非常友好。

但是：

```
for (int i = 0; i < 1000000; i++) {
    sum += linkedList.get(i);
}
```

如果底层是链表，访问每个元素都需要沿着指针跳转。

内存地址可能分布在不同区域。

CPU 预取器很难像处理连续数组一样有效预取数据。

这就是为什么：

**相同的 O(N) 算法，数组往往比链表快很多。**

这也是为什么 Redis、Memcached、Netty、Disruptor 等高性能系统都非常重视内存布局。

## **2.3 为什么指针跳转会影响性能？**

来看两个数据结构。

数组：

```
Array

+------+------+------+------+
|  A   |  B   |  C   |  D   |
+------+------+------+------+
  0      1      2      3
```

链表：

```
Node A ----> Node B ----> Node C ----> Node D
```

假设每个 Node 分布在不同的内存区域：

```
Node A：0x1000
Node B：0x8000
Node C：0x3000
Node D：0x9000
```

读取 Node A 后，CPU 不一定能够提前知道下一个 Node 在哪里。

所以：

```
数组：
连续内存访问
Cache Line 利用率高
CPU 预取效果好

链表：
指针跳转
可能发生多次 Cache Miss
内存访问延迟更高
```

这也是为什么一些高性能系统宁愿使用稍微复杂的数组结构，也不愿意使用大量零散的小对象。

当然，Hash Table 也有随机访问的特点，因此 Redis 和 Memcached 都不可能完全消除 Cache Miss。

## **2.4 什么是伪共享（False Sharing）？**

这是多线程性能优化中非常重要的一个知识点。

先看一段 Java 代码：

```
public class Counter {
    volatile long count1 = 0;
    volatile long count2 = 0;
}
```

假设：

```
Thread 1：不断修改 count1

Thread 2：不断修改 count2
```

你可能认为：

两个线程操作的是不同变量，应该不会互相影响。

但实际上可能存在问题。

假设：

```
Cache Line（64B）

+------------------+------------------+
| count1           | count2           |
+------------------+------------------+
```

两个变量恰好位于同一个 Cache Line。

现在：

```
CPU Core 1 修改 count1
CPU Core 2 修改 count2
```

虽然两个线程修改的是不同变量，但是它们修改的是同一条 Cache Line。

CPU 为了保证多核缓存一致性，需要协调 Cache Line 的所有权。

于是发生：

```
Core 1 修改 count1
       |
       v
Cache Line 状态发生变化
       |
       v
Core 2 的对应 Cache Line 失效
       |
       v
Core 2 修改 count2
       |
       v
Core 1 的 Cache Line 又失效
```

两个核心不断争夺 Cache Line 的所有权。

这就是伪共享。

[2.5 CPU 是如何执行任务的？ | 小林coding | Java面试学习](https://images.openai.com/static-rsc-4/bweMr48SUCkcfoDZ2iembv4nU652h0cXKYS13UIG6nDyju3Mbaez9WZR5QZImbjAd17-3dcPGztOvtDWY3mEGg5UOEgrmUT4W6qxeS5r2031jgMvXLF_1sQ-HnY1U0lIGssHi8QceLeV_xuT46ec3ciHuZqoub7unpkcjXNVFL4?purpose=inline)

[xiaolincoding.com](https://xiaolincoding.com/os/1_hardware/how_cpu_deal_task.html?utm_source=chatgpt.com)

为什么叫伪共享？

因为从程序逻辑上看：

```
count1 和 count2 完全独立
```

但从 CPU 缓存一致性机制看：

```
它们共享同一个 Cache Line
```

于是产生了不必要的缓存一致性流量。

### **2.5 Java 如何解决伪共享？**

一种方式是填充字段，让不同线程使用的变量尽量位于不同 Cache Line。

例如：

```
public class PaddedCounter {

    volatile long count1;

    // 填充字段，避免 count1 和 count2 过于靠近
    long p1, p2, p3, p4, p5, p6, p7;

    volatile long count2;

    // 其他填充字段
    long p8, p9, p10, p11, p12, p13, p14;
}
```

这是一种概念性示例，不能保证 Java 对象字段的最终物理布局一定符合预期。

Java 中更专业的方式包括：

- `@Contended`：JVM 支持的字段填充机制。
- Disruptor 的 Padding 思想。
- JMH 基准测试验证。

例如：

```
import jdk.internal.vm.annotation.Contended;

public class Counter {

    @Contended
    volatile long count1;

    @Contended
    volatile long count2;
}
```

注意：

`@Contended` 属于内部 API，在部分 JDK 环境下需要额外的编译和 JVM 参数，例如 `-XX:-RestrictContended`，具体还取决于 JDK 版本和模块访问限制。

### **2.6 伪共享和 volatile 有什么关系？**

这里联系一下你之前研究的 Java 内存模型。

`volatile` 保证可见性和一定的有序性语义。

但是它并不意味着每次读写都直接访问主内存。

CPU 仍然可以使用 Cache。

为了保证多个核心观察到一致的结果，CPU 需要通过缓存一致性协议协调缓存行状态。

常见 x86 处理器使用 MESI 或其变体。

MESI 包括：

| 状态 | 含义           |
| -- | ------------ |
| M  | Modified，已修改 |
| E  | Exclusive，独占 |
| S  | Shared，共享    |
| I  | Invalid，无效   |

当一个核心需要修改某个 Cache Line 时，通常需要获得该缓存行的独占或修改权限，并使其他核心中的副本失效。

这就是伪共享背后的硬件原因。

注意：MESI 是理解原理的模型，现代 CPU 的具体协议可能更复杂。

## **2.7 什么是锁竞争？**

再看一个经典问题。

```
public class Counter {

    private long count = 0;

    public synchronized void increment() {
        count++;
    }
}
```

假设有 100 个线程同时执行：

```
counter.increment();
```

会发生什么？

因为所有线程都要获取同一把锁：

```
Thread 1 ----\
Thread 2 -----\
Thread 3 ------> synchronized lock
Thread 4 -----/
...
Thread 100 ---/
```

同一时刻只有一个线程能够进入临界区。

其他线程需要等待。

这就是锁竞争。

锁竞争会带来：

1. 线程阻塞和唤醒。
2. 上下文切换。
3. CPU 缓存行在核心间迁移。
4. 锁状态维护成本。
5. 临界区串行化。

其中有一个容易忽略的因素：

即使没有发生线程阻塞，多个 CPU 核心对同一锁变量执行 CAS，也可能造成严重的 Cache Line 竞争。

因此，高并发场景下，锁的成本不只是上下文切换。

### **2.8 如何解决锁竞争？**

常见有四种思路。

**方案一：锁分段**

不要所有线程共用一把锁。

```
原来：

所有请求 ---> Lock 1

优化后：

请求 ---> Lock 1
请求 ---> Lock 2
请求 ---> Lock 3
请求 ---> Lock 4
```

例如 ConcurrentHashMap 的分段思想，以及基于 Bucket 的锁设计。

注意，JDK 8 ConcurrentHashMap 已经不再采用传统 Segment 数组作为核心结构，而是通过 CAS、synchronized 等机制实现并发控制。

**方案二：线程本地化**

每个线程维护自己的计数器。

```
ThreadLocal<Long> counter;
```

这样减少了共享变量的竞争。

缺点是需要合并各线程的数据。

**方案三：批量处理**

不要每次请求都操作共享结构。

例如：

```
原来：

请求 1 -> 更新共享计数器
请求 2 -> 更新共享计数器
请求 3 -> 更新共享计数器

优化：

线程本地累计 1000 次
       |
       v
一次性更新共享计数器
```

**方案四：无锁算法**

使用 CAS、原子变量、环形队列等减少显式锁竞争。

但注意：

无锁不代表没有竞争。CAS 本身也可能产生缓存行争夺。

# **专题三：如果 Redis 也采用多线程执行命令，能不能超过 Memcached？**

这是三个专题中最有架构价值的问题。

我的答案是：

**理论上可以，但不是简单地把 Redis 的命令执行改成多线程，就一定能获得更高性能。**

我们从 Redis 的架构设计开始分析。

## **3.1 Redis 为什么最初选择单线程？**

Redis 最初的设计目标之一，就是构建一个高性能的内存数据服务。

它的核心操作通常很简单：

```
GET user:10001
SET user:10001 Tom
INCR counter
HGET user:10001 name
```

这些操作大多数只涉及内存访问和简单计算。

如果使用多线程执行：

```
Thread 1：GET user:10001
Thread 2：SET user:10002 Tom
Thread 3：INCR counter
Thread 4：HGET user:10003 name
```

就必须解决多个线程同时访问 Redis 内部数据结构的问题。

例如 Redis 的 Dict：

```
Redis DB
   |
   v
Hash Table
   |
   +--- Bucket 1
   +--- Bucket 2
   +--- Bucket 3
```

假设两个线程同时操作同一个 Bucket：

```
Thread 1：SET user:10001 Tom
Thread 2：DEL user:10001
```

可能发生：

- 一个线程正在插入 DictEntry。
- 另一个线程正在删除 DictEntry。
- 一个线程正在扩容 Hash Table。
- 另一个线程正在遍历 Hash Table。

这就需要复杂的并发控制。

## **3.2 假设 Redis 改成多线程，会遇到什么问题？**

我们假设有 8 个 CPU 核心。

采用多线程执行命令：

**Redis Network Layer**

**Worker 1**

执行命令

**Worker 2**

执行命令

**Worker 3**

执行命令

**Worker 4**

执行命令

**共享 Redis 数据结构**

Dict / SDS / List / Set / ZSet

假设性的多线程 Redis 架构，并非 Redis 当前默认架构。

我们逐个分析。

### **问题一：Hash Table 并发安全**

假设两个线程分别操作不同 Key：

```
Thread 1：SET user:10001 Tom
Thread 2：SET user:10002 Jack
```

如果两个 Key 落在不同 Bucket，理论上可以并行。

但是如果它们落在同一个 Bucket，就可能需要同步。

另外，Hash Table 扩容和 Rehash 是更复杂的问题。

Redis Dict 支持渐进式 Rehash。

例如：

```
原始 Hash Table：

Table 0
  |
  +--- Bucket 0
  +--- Bucket 1
  +--- Bucket 2

开始扩容：

Table 0
Table 1

逐步迁移 Bucket
```

如果多个线程同时执行命令，那么：

- 谁负责迁移 Bucket？
- 其他线程如何访问正在迁移的数据？
- 如何保证读写正确？
- 如何避免重复迁移？
- 如何保证删除操作安全？

这些问题都需要额外的并发设计。

### **问题二：内存释放安全**

假设：

```
Thread 1：读取一个 Value
Thread 2：删除这个 Value
```

Thread 2 如果直接释放内存：

```
free(value);
```

而 Thread 1 还在使用这个 Value，就可能产生 Use-After-Free。

单线程命令执行模型中，这类命令之间的竞争容易避免。

多线程执行后，就需要考虑：

- 引用计数。
- 内存回收时机。
- 延迟释放。
- RCU 等技术。
- 对象生命周期管理。

这会增加实现复杂度。

### **问题三：原子性与事务**

例如：

```
MULTI
INCR account:1
INCR account:2
EXEC
```

Redis 的事务命令按照队列顺序执行，执行期间不会被其他客户端的普通命令插入。

如果采用多线程执行：

```
Thread 1：执行事务中的 INCR account:1
Thread 2：执行 SET account:1 100
Thread 1：执行事务中的 INCR account:2
```

就需要确保事务语义不会被破坏。

可以通过锁、事务调度器、分片等方式解决，但实现成本会提高。

### **问题四：锁竞争可能抵消并行收益**

假设：

```
8 个线程
100 万 QPS
```

如果所有线程都需要争夺一把全局锁：

```
Thread 1 ----\
Thread 2 -----\
Thread 3 ------> Global Lock
...
Thread 8 -----/
```

那么实际还是串行执行。

如果采用细粒度锁：

```
Bucket 1 -> Lock 1
Bucket 2 -> Lock 2
Bucket 3 -> Lock 3
```

又会出现：

- 锁管理开销。
- 锁竞争。
- 死锁风险。
- Cache Line 争夺。
- 数据结构扩容时的全局协调。

所以，多线程架构并不意味着性能必然线性提升。

## **3.3 那为什么 Memcached 多线程能够做得比较好？**

这里需要回到两者的业务模型。

Memcached 的核心功能比较简单：

```
GET
SET
DELETE
INCR
DECR
```

它没有 Redis 那样丰富的复杂数据结构。

对于大量独立的 Key-Value：

```
user:10001 -> Tom
user:10002 -> Jack
user:10003 -> Alice
```

不同请求之间有比较大的并行空间。

可以把缓存分布在不同 Bucket 或 Item 上，由多个 Worker 处理。

当然，Memcached 的 Hash Table、Item、Slab 等共享结构依然需要同步，并不是完全无锁。

它的优势是：功能边界较简单，能够围绕简单 KV 操作设计多线程并发模型。

## **3.4 Redis 有没有尝试过其他并发方案？**

Redis 实际上并不是完全排斥多线程。

需要区分几个层次：

| 机制                      | 作用               |
| ----------------------- | ---------------- |
| I/O Threads             | 并行处理部分网络读写       |
| 主线程执行命令                 | 维持核心命令执行模型       |
| Redis Cluster           | 将数据分布到多个节点       |
| 多实例部署                   | 利用多个 CPU 核心      |
| Redis 7 Functions / Lua | 在单个执行上下文中执行服务端脚本 |

Redis 6 引入 I/O Threads 后，可以让网络 I/O 工作利用更多 CPU 核心。

例如：

```
              多个客户端
                   |
                   v
             Redis 网络层
                   |
          +--------+--------+
          |        |        |
          v        v        v
        I/O 1    I/O 2    I/O 3
          |        |        |
          +--------+--------+
                   |
                   v
             主线程执行命令
                   |
                   v
              返回结果
```

这里 I/O Threads 的主要价值，是把网络数据读取、协议处理和写回等工作中的一部分分摊给多个线程。

不同版本和配置下，具体哪些网络处理步骤由 I/O 线程执行会有所差异。

但命令执行仍然主要由主线程负责。

## **3.5 Redis 如何真正利用多个 CPU 核心？**

最常见的方式其实不是改造一个 Redis 实例，而是部署多个 Redis 实例。

例如：

```
16 Core Server
      |
      +--- Redis Instance 1 -> CPU Core 1
      |
      +--- Redis Instance 2 -> CPU Core 2
      |
      +--- Redis Instance 3 -> CPU Core 3
      |
      +--- Redis Instance 4 -> CPU Core 4
      |
      +--- ...
```

每个实例有自己的：

- 事件循环。
- 数据字典。
- 内存管理。
- 命令执行线程。

多个实例之间没有共享核心数据结构，因此可以并行执行命令。

Redis Cluster 也是类似思想的扩展。

它将 Key 分配到不同 Slot：

\\[ 16384 \text{ Hash Slots} \\]

不同节点负责不同的 Slot。

例如：

```
Redis Cluster

Node 1：0 ~ 5000
Node 2：5001 ~ 10000
Node 3：10001 ~ 16383
```

这样不同节点可以并行处理不同请求。

当然，实际 Slot 分布不一定像上面一样连续，也可以重新分配。

这就是通过水平扩展来利用多核和多机资源的思想。

# **专题四：把这些原理联系到 Java 高并发编程**

我再带你把前面的知识迁移到 Java。

这是非常重要的一步，因为理解了底层原理，才能在实际项目中判断如何优化。

## **4.1 为什么 Java 的 ArrayList 通常比 LinkedList 更适合遍历？**

看代码：

```
List<Integer> arrayList = new ArrayList<>();

List<Integer> linkedList = new LinkedList<>();
```

两者遍历：

```
for (Integer value : arrayList) {
    sum += value;
}
```

```
for (Integer value : linkedList) {
    sum += value;
}
```

两者时间复杂度都是 O(N)。

但是实际性能通常有明显差距。

原因：

ArrayList：

```
连续的引用数组
    |
    v
Cache Line 预取
    |
    v
连续访问
```

LinkedList：

```
Node 1
  |
  v
Node 2
  |
  v
Node 3
  |
  v
Node 4
```

需要不断进行指针跳转。

而且 LinkedList 每个节点还需要额外的对象头、前后指针等内存。

所以在很多遍历场景下，ArrayList 的性能更好。

这就是内存局部性的重要性。

## **4.2 为什么 ConcurrentHashMap 比 Hashtable 更适合高并发？**

Hashtable 的很多方法使用 synchronized：

```
public synchronized V get(Object key) {
    // ...
}
```

这意味着多个线程可能竞争同一把对象锁。

ConcurrentHashMap 则采用更细粒度的并发控制。

JDK 8 中主要使用：

- CAS。
- synchronized（局部 Bucket 级别的同步）。
- volatile。
- 节点链表和红黑树。

简化理解：

```
Hashtable：

所有操作
    |
    v
全局锁
    |
    v
执行
```

```
ConcurrentHashMap：

Bucket 1 -> 可能独立操作
Bucket 2 -> 可能独立操作
Bucket 3 -> 可能独立操作
Bucket 4 -> 可能独立操作
```

不同 Bucket 上的部分操作可以并发进行。

但也要注意：

如果多个线程竞争同一个 Key，或者发生扩容、树化等操作，依然需要同步。

## **4.3 为什么 Disruptor 的性能很高？**

[Disruptor框架深度解析与实战指南 - 染沁 - 博客园](https://images.openai.com/static-rsc-4/__uDFjeursvORtNWEp4AYr77P6rpitNWoqCL9hLs9X1NXZvvRPaQ84jS38S9dw3idIUKJlVa9CSsDDGbPevuHe1YVsm2kAw9AjjPWbWl1VdvcQRNIh6-OrjDkMJP_YxN2Hl8Hyo4BMqGN4MgzIERwLXTJYGQu7WHvEujnLOIWMc?purpose=inline)

[cnblogs.com](https://www.cnblogs.com/lcha-coding/p/19129068?utm_source=chatgpt.com)

Disruptor 是一个非常值得研究的高性能并发框架。

它的核心思想包括：

1. RingBuffer：使用预分配的环形数组。
2. Sequence：使用序列号管理生产者和消费者进度。
3. CAS：协调并发访问。
4. Padding：降低伪共享。
5. 减少锁竞争和对象分配。

普通队列可能需要频繁进行：

```
创建 Node
分配内存
更新指针
释放 Node
```

Disruptor：

```
提前创建 RingBuffer
       |
       v
生产者申请 Sequence
       |
       v
写入预分配槽位
       |
       v
消费者读取
       |
       v
重复使用槽位
```

这就和 Memcached 的 Slab 有一个共同思想：

**尽量预分配内存，减少高频运行过程中的动态内存分配。**

但两者并不完全相同：

- Memcached 的 Slab 主要解决缓存对象内存管理问题。
- Disruptor 的 RingBuffer 主要解决高并发生产消费和数据传递问题。

它们只是都利用了预分配和内存复用的思想。

## **4.4 为什么 Netty 使用池化内存？**

Netty 也是一个非常典型的例子。

如果每次收到网络数据都创建新的 ByteBuf：

```
ByteBuf buffer = Unpooled.buffer(1024);
```

在大量网络请求下，可能出现：

- 频繁内存分配。
- 频繁释放。
- GC 压力。
- 内存碎片。
- 分配器竞争。

Netty 提供了 PooledByteBufAllocator。

核心思想是：

```
预先分配内存区域
       |
       v
按照规格管理内存
       |
       v
需要时分配 ByteBuf
       |
       v
使用完毕归还内存池
       |
       v
后续请求复用
```

这与 Slab、jemalloc 的分级内存管理有相通之处。

Netty 的池化分配器还涉及 Arena、Chunk、Page、Subpage 等概念。

对于 Java 后端工程师，我非常建议你把这套内存管理机制研究透彻。

# **专题五：真正做高并发性能优化时，应该怎么思考？**

前面讲了很多底层原理，现在我们把它变成一套可以用于线上系统的分析方法。

假设有一个 Java 服务：

- 16 核 CPU。
- 32GB 内存。
- 每秒 50 万次缓存读取。
- 主要操作 Redis GET。
- CPU 使用率达到 95%。
- Redis 网络延迟正常，但整体请求延迟越来越高。

你应该怎么排查？

我建议按照下面的顺序。

## **5.1 第一步：判断 CPU 到底消耗在哪里**

不能看到 CPU 95% 就直接认为 Redis 单线程瓶颈。

需要分别检查：

```
Java 应用 CPU
Redis CPU
网络中断 CPU
系统态 CPU
用户态 CPU
```

Linux 常用命令：

```
top -H -p <pid>
```

查看进程各线程 CPU 使用情况。

```
pidstat -t -p <pid> 1
```

查看线程级 CPU 消耗。

```
perf top -p <pid>
```

观察热点函数。

Redis 可以使用：

```
redis-cli INFO CPU
redis-cli INFO STATS
redis-cli INFO MEMORY
redis-cli INFO COMMANDSTATS
```

如果发现 Redis 主线程 CPU 很高，而 I/O 线程比较空闲，可能是命令执行成为瓶颈。

## **5.2 第二步：判断是不是内存访问瓶颈**

如果 CPU 并没有完全打满，但请求延迟很高，需要进一步分析。

例如：

```
perf stat -p <pid> \
  -e cycles,instructions,cache-references,cache-misses
```

重点观察：

- CPU cycles。
- Instructions。
- Cache References。
- Cache Misses。

如果 Cache Miss 很高，可能存在内存访问局部性较差的问题。

注意，Cache Miss 不能单独作为性能瓶颈结论，还要结合 IPC、内存带宽、CPU 型号和采样分析。

## **5.3 第三步：判断是不是锁竞争**

Java 可以使用：

```
jcmd <pid> Thread.print
```

查看线程状态和堆栈。

还可以使用：

```
async-profiler
```

观察 CPU、锁和分配热点。

如果发现大量线程阻塞在同一个锁上，就需要考虑：

- 是否存在全局锁。
- 是否可以分段锁。
- 是否可以线程本地化。
- 是否可以批量操作。
- 是否可以减少共享变量。

## **5.4 第四步：判断是不是内存分配问题**

Java 可以观察：

```
jstat -gc <pid> 1000
```

查看 GC 变化。

如果短生命周期对象大量创建，就可能产生较大的 GC 压力。

优化方向：

- 对象复用，但避免不必要的复杂对象池。
- 减少临时对象。
- 使用批量处理。
- 使用 ByteBuffer 池化。
- 使用 Netty PooledByteBufAllocator。
- 对合适的数据结构采用预分配。

这里要注意：Java 对象池并不一定比直接创建对象快。现代 JVM 的 TLAB、逃逸分析和 GC 优化已经非常成熟，应当先测量再决定是否池化。

# **六、最后做一个知识体系总结**

这张图可以作为你以后分析高性能系统的知识地图。

**高性能缓存系统**

**内存管理**

Slab / jemalloc

Size Class

Arena / TCache

内存碎片

对象复用

**CPU 架构**

CPU Cache

Cache Line

Cache Miss

内存局部性

伪共享

**并发模型**

多线程 / 单线程

CAS

锁竞争

线程本地化

无锁算法

**系统架构**

水平扩展

Redis Cluster

数据分片

批量处理

连接管理

**最终目标：提高吞吐量、降低 P99 延迟、提高资源利用率**

# **七、给你留三个架构师级思考题**

这三个问题不是简单背答案，而是检验是否真正理解了上面的知识。

**知识点自测**

3/3 已作答

**1. 假设 Redis 主线程 CPU 已经达到 100%，I/O Threads 几乎没有负载。你会优先考虑什么方案？**

继续增加 I/O Threads

使用 Pipeline、优化命令，必要时分片到多个实例

直接增加 Redis 连接数

增加客户端线程数

回答正确

瓶颈在命令执行而不是网络 I/O。应先定位热点命令、降低每条命令成本，再考虑多实例或 Cluster 分片。

**2. 假设 Memcached 总内存还有 40%，但某个 Slab Class 已经无法分配 Chunk。最合理的解释是什么？**

Linux 物理内存不足

CPU Cache Miss 太高

Class 间内存分配不均衡

Hash Table 一定发生了死锁

回答正确

不同 Slab Class 的 Chunk 不能直接混用，可以检查 Slab 分布并评估 Slab Reassignment / Automove。

**3. 两个 Java 线程分别更新不同的 volatile long，为什么仍然可能发生性能下降？**

volatile 禁止 CPU 使用缓存

可能发生 False Sharing

Java 所有 volatile 变量共用一把锁

long 一定不是原子操作

回答正确

两个变量可能处于同一个 Cache Line，导致缓存行在 CPU 核心间反复失效和转移。

# **3 / 3**

重新答题

**我的建议：** 接下来可以实际做一个 Java 微型实验项目：分别实现普通 HashMap、分段锁 HashMap、ThreadLocal 计数器、伪共享 Counter 和 Padding Counter，再用 JMH 测试吞吐量、延迟和 CPU 表现。这样你就能把今天讲的内存布局、Cache Line、CAS、锁竞争真正转化为自己的技术能力，而不是停留在概念层面。