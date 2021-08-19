一、概念

主要模块

* 索引
  * 最左前缀
  * varchar部分索引
* 锁
* 事务
* log：binlog, redolog, undolog



* 团队协作正如分布式、分库分表，管理好了能提升性能，但是有管理成本

# 二、Content

1. order by 原理

   内存中排序--快排：

   	* 全字段排序（name， age， gender）
   	* rowid 排序（name，id），再回表

   外部排序--归并

2. 索引失效

   * 索引**字段**用了函数操作；eg. Month(creation_time)
   * 隐式类型转换 -- 函数操作；eg. name=123
   * 字符编码转换 -- 函数操作；eg. Utf8 --> utf8mb4

3. 幻读

   幻读：当前读 + 新插入行

   不可重复读：当前读 + 修改行

4. 间隙锁加锁规则

   1. RR
      * 访问到的索引才会加锁，加锁单位是 next-key-lock
      * 等值查询：
        * 唯一索引：next-key lock 退化为 行锁
        * 非唯一索引：需要访问到第一个不满足条件的值，且最后一个值的加锁退化间隙锁（最后一个不加行锁）
      * 范围查询bug：查询到第一个不满足条件的值
   2. RC
      - 加锁规则同上，next-key-lock 换成行锁，且优化：释放不满足条件的行锁

   https://mp.weixin.qq.com/s/jLTgF8U8DiVZLfEIn6vUAw

   https://helloworlde.github.io/blog/blog/MySQL/MySQL-%E4%B8%AD%E5%85%B3%E4%BA%8Egap-lock-next-key-lock-%E7%9A%84%E4%B8%80%E4%B8%AA%E9%97%AE%E9%A2%98.html
   
5. 写log

   1. 写 binlog

      Binlog cache(每个线程一个) --> page cache --〉 disk

      ![写binlog](/Users/kyrie/Documents/Article/assets/mysql_写binlog.webp)

      Mix：因为有些 statement 格式的 binlog 可能会导致主备不一致(eg. Limit 主备选错索引)，所以要使用 row 格式。但 row 模式缺点是很占空间，所以 mysql 折中，判断 sql 是否可能引起主备不一致，若可能则用 row，不可能则用 statement

   2. 写redolog

      redo buffer(线程共用一个) --> page cache --> disk

      ![写redolog](/Users/kyrie/Documents/Article/assets/mysql_写redolog.webp)

   3. group commit

6. 主从

   ![主从同步](/Users/kyrie/Documents/Article/assets/mysql_主从同步.png)

   1. 主从延迟解决方案
      - 要求准确的去读主库
   
7. Buffer pool LRU

   ![Buffer pool LRU](/Users/kyrie/Documents/Article/assets/mysql_Buffer pool LRU.webp)

   * Young 区：普通 LRU
   * old 区：主要作用是扫描大表时，不会影响 yong 区；
     * 换入 page 查到 old 区头部；
     * 每次访问 old 区，判断该 page 存活时间 > 1s ？是--放入 yong 头；否--不动。因为针对大表扫描，一个 page 中有多条记录会被多次访问，但是顺序扫描所以时间不会超过1s，因此留在了 old 区。

8. Join

   1. 被驱动表能用索引时，用小表做驱动表。扫描行数：N + N*log2(M);
   2. 不能用索引时，也是小表做驱动表。buffer，小表清空buffer次数少，另一个表遍历的少。

