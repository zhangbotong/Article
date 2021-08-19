# MySQL 中的优化思路

* 随机写 --> 顺序写。redo log
* 读写锁（读并发） --> MVCC（读写并发）
* 索引 -- 跳表

# Redis 思维

COW（copy on write），指针指向存储区域，而非直接复制。

git 分支 -- 指针；svn 分支 -- copy content