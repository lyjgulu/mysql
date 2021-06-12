## mysql的 Buffer 详解

### Buffer Pool

- Buffer Pool淘汰策略(LRU，与redis的LRU实现思想一致，方式大不相同)


1. change buffer
- todo（09讲）

  log buffer区包含(redo log buffer和undo log buffer)
  binlog buffer

changebuffer merge
从磁盘读入数据页到内存（老版本的数据页）；
从 change buffer 里找出这个数据页的 change buffer 记录 (可能有多个），依次应用，得到新版数据页；
写 redo log。这个 redo log 包含了数据的变更和 change buffer 的变更。

#### buffer pool flush(内存脏页刷盘)(redo log部分看mysql的各种log)
1. InnoDB的redo log写满了。这时候系统会停止所有更新操作，把checkpoint往前推进，redo log留出空间可以继续写。
![flushByRedoLog](https://raw.githubusercontent.com/lyjgulu/mysql/main/image/flushByRedoLog.png)
2. 系统内存不足。当需要新的内存页，而内存不够用的时候，就要淘汰一些数据页，空出内存给别的数据页使用。如果淘汰的是“脏页”，就要先将脏页写到磁盘。刷脏页一定会写盘，就保证了每个数据页有两种状态：
    - 一种是内存里存在，内存里就肯定是正确的结果，直接返回；
    - 另一种是内存里没有数据，就可以肯定数据文件上是正确的结果，读入内存后返回。这样的效率最高。
3. 系统空闲时，后台将内存脏页刷盘
4. mysql正常关闭，会讲内存脏页刷盘

- InnoDB用缓冲池（buffer pool）管理内存，缓冲池中的内存页有三种状态：
    - 还没有使用的。
    - 使用了并且是干净页。
    - 使用了并且是脏页。
  
- flush性能影响
    - 一个查询要淘汰的脏页个数太多，会导致查询的响应时间明显变长；(对应为一个长事务，且事务中存在较多竞争少的查询sql)
        - 解决办法：尽量将无关的查询sql放在事务外部
    - 日志写满，更新全部堵住，写性能跌为0，这种情况对敏感业务来说，是不能接受的。
  
- flush时机
    - 计算脏页比例和redo log写入比例取较大值，与设置的innodb_io_capacity比较，进行刷盘。
- flush对磁盘的影响
    - 对于 IOPS 较低的机械硬盘，可以将 innodb_flush_neighbors 设置为1，减少随机硬盘 IO ，提升系统性能。
    - 对于 IOPS 较高的固态硬盘，可以将 innodb_flush_neighbors 设置为0，缩短刷脏页操作，减少 sql 语句响应时间

### Doublewrite Buffer
