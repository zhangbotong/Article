* RDB + AOF 仍有 1s 的数据可能丢失，AOF 未落盘
* 主从数据延迟，不一致情况。的确不是强一致的。
* IO 多路复用
* 切片集群

TODO：

* 秒杀

[TOC]

# 基础

## 数据结构

### v 数据类型

* String
* Hash
* List
* Set
* Sorted Set
* bitmap
* hyperloglog
* Streams -- 消息队列

**哈希单点快，sorted set 可根据 score 范围查询。**

### 底层实现

* String
* 数组 -- 压缩列表
* 链表（双向） -- 跳表
* 哈希表

![数据类型与底层数据结构](/Users/kyrie/Documents/Note/Redis/数据类型与底层数据结构.webp)

![数据结构时间复杂度](/Users/kyrie/Documents/Note/Redis/数据结构时间复杂度.webp)

数组 & 压缩列表，优势在于占用空间连续且少，可以利用局部性原理效率很高。

### 存储结构

数组（hash 桶） + k 指针 + v 指针

![全局哈希表](/Users/kyrie/Documents/Note/Redis/全局哈希表.webp)

存在问题：哈希冲突

解决：2 个全局哈希表，**渐进式 rehash**，由于单线程，如果全部用来 rehash 会阻塞正常使用，所谓渐进式hash就是在每次请求时顺带进行一次 rehash 拷贝，将一次大量拷贝的开销，分摊到了多次处理请求中。若没有请求 Redis 会直接定期执行。

慢速：由于是全局 hash，单元素操作很快，但范围操作就很慢，对比 mysql 的 B+ 树索引解决了范围查询问题，但随之而来的就是降低单元素操作效率，trade off。

## IO 模型

单线程：网络 IO + 数据读写 

多线程：持久化、主从复制、异步删除

why 单线程：Redis 性能瓶颈不在于数据读写，而在于网络 IO。若用多线程还需控制共享变量加锁 ，如：length 维护，得不偿失。

单线程快机制：多路复用的高性能网络 IO，非阻塞。同时监听多个 socket，有些只连接未使用，只处理那些真正使用的 socket 请求，类似于 医生 + 分诊台，

![多路复用网络 IO](/Users/kyrie/Documents/Note/Redis/多路复用网络 IO.webp)

## 持久化

* AOF -- statement
* RDB -- row

fork 子进程可能阻塞主进程：

虽然应用 copy on write 机制，可以避免一次性大量拷贝内存数据（与渐进式 hash 异曲同工），但仍需拷贝内存页表，使主子进程指向同一片内存，可能阻塞。子进程写 AOF，父进程正常读，当写时，父进程要申请新内存拷贝该数据（Cow），内存分配以页为单位 4k，若 bigkey 申请内存会阻塞，子进程写 rdb 同理。

### AOF（Append only file）

* 写后日志 -- 避免了语法检查写入错误命令/不阻塞当前操作
* 主线程写
* AOF 重写新log，不污染原 log，解决日志文件过大。 -- 后台线程（Not important because of rdb）

先写内存文件，再落盘

问题：

* 同步落盘，阻塞主线程，也会有非常短暂的数据丢失；
* 非同步落盘，down 机有数据丢失。1s

### RDB(Redis database)

* 子进程
* 数据紧凑，恢复数据快

Copy on write：主进程 fork 子进程，并复制内存页表给子进程。子进程写 rdb，父进程读正常，当需要写，父进程 copy ，申请新内存进行写，父子分离，子进程还指向原来内存。

### Hybrid

两次 RDB 之间用 AOF

# 高可用 -- 主从

主从同步：RDB 

# 实践

* String 存储数字，占空间大。

  <img src="/Users/kyrie/Documents/Note/Redis/String存数字.webp" alt="String存数字" style="zoom:40%;" />

  哈希存储（压缩列表）：127.0.0.1:6379> hset 1101000 060 3302000080

* 聚合统计

  * 交、差、并：set，速度慢，可以在一个从库进行
  * bitmap：二值状态如签到：
  * hyperloglog：高性能，允许部分误差，如直播间人数统计
  * GEO：地理位置。二分法表示经纬度

# Feature

* Redis 变慢原因
  * 慢查询
  * 大量过期
  * redis变慢问题优化自我总结：
    1.cpu：redis实例绑定同一个cpu物理核；网络中断和redis实例绑定同一个cpu socket
    2.内存：关闭大内存页，观察swap，增加物理内存
    3.磁盘：数据丢失容忍度aof策略选择；AOF使用SDD；Huge page
    4.慢操作：集合操作，bigkey删除，返回集合改为SCAN多次操作
    5.大量过期：过期时间加随机数
