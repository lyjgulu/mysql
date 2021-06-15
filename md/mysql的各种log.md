## mysql的各种log
**MySQL 是 WAL（Write-Ahead Logging）机制，也就是写操作会先存入日志，然后再写入磁盘，这样可以避开高峰，提高数据库的可用性。**

### 三种日志概述

#### undo log 与 MVCC(并发版本控制)
- undo log是 Innodb 引擎专属的日志，是记录每行数据事务执行前的数据。主要作用是用于实现 MVCC 版本控制，undo log 保证事务隔离级别的读已提交和读未提交级别，保证了原子性。(幻读问题依靠锁来实现)
![undo log抽象表示](https://raw.githubusercontent.com/lyjgulu/mysql/main/image/undo%20log.png)

#### redo log 与 Buffer Pool(buffer一块关注 mysql的各种buffer )
- InnoDB 内部维护了一个缓冲池，用于减少对磁盘数据的直接IO操作，并配合 redo log、内部的 change buffer 来实现异步的落盘，保证程序的高效执行。
- redo log 是记录修改操作，防止断电丢失写操作，降低随机写磁盘 IO 消耗（转成顺序写）；Change Buffer 是为了将写操作延迟更新到缓冲池，降低随机读磁盘 IO 的消耗。（不需要频繁从磁盘读数据页)
- innodb_flush_log_at_trx_commit这个参数设置成1的时候，表示每次事务的redo log都直接持久化到磁盘。这个参数设置成1，这样可以保证MySQL异常重启之后数据不丢失。
- redo log 中的 LSN 是日志序列号，可以简单理解为日志的记录时机。
- 例子
    - 执行sql 
    ``` sql
    insert into t(id,k) values(id1,k1),(id2,k2);
    ```
  - 假设当前k索引树的状态，查找到位置后，k1所在的数据页在内存(InnoDB buffer pool)中，k2所在的数据页不在内存中。如图。
  ![undo log抽象表示](https://raw.githubusercontent.com/lyjgulu/mysql/main/image/redoAndBuffer.png)
  - 涉及了四个部分：内存、redo log（ib_log_fileX）、 数据表空间（t.ibd）、系统表空间（ibdata1）。
  - 3，4步是将两个动作记入redo log中
- 组提交 group commit


#### bin log(所有引擎都有，二进制)
- redo log 因为大小固定，所以不能存储过多的数据，它只能用于未更新的数据落盘，而数据操作的备份恢复、以及主从复制是靠 bin log。（如果数据库误删需要还原，那么需要某个时间点的数据备份以及bin log）
- 在更新到数据页缓存或者 Change Buffer 后，首先进行 redo log 的编写，编写完成后将 redo log设为prepare状态，随后再进行binlog的编写，等到binlog也编写完成后再将redo log设置为commit状态。这是为了防止数据库宕机导致binlog没有将修改记录写入，后面数据恢复、主从复制时数据不一致。
- 断电回滚
    先检查redo log记录的事务操作是否为commit状态：
    1、如果是commit状态说明没有数据丢失，判断下一个。
    2、如果是prepare状态，检查binlog记录的对应事务操作（redo log 与 binlog 记录的事务操作有一个共同字段XID，redo log就是通过这个字段找到binlog中对应的事务的）是否完整（这点在前面 binlog三种格式分析过，每种格式记录的事务结尾都有特定的标识），如果完整就将redo log设为commit状态，然后结束；不完整就回滚redo log的事务，结束。
- 三种日志格式
    1.Row（5.7默认）。记录操作语句对具体行的操作以及操作前的整行信息。缺点是占空间大。优点是能保证数据安全，不会发生遗漏。
    2.Statement。记录修改的 sql。缺点是在 mysql 集群时可能会导致操作不一致从而使得数据不一致（比如在操作中加入了Now()函数，主从数据库操作的时间不同结果也不同）。优点是占空间小，执行快。
    3.Mixed。会针对于操作的 sql 选择使用Row 还是 Statement。缺点是还是可能发生主从不一致的情况。

#### 三种日志的比较
1. undo log是用于事务的回滚、保证事务隔离级别读已提交、可重复读实现的。redo log是用于对暂不更新到磁盘上的操作进行记录，使得其可以延迟落盘，保证程序的效率。bin log是对数据操作进行备份恢复（并不能依靠 bin log 直接完成数据恢复）。

2. undo log 与 redo log 是存储引擎层的日志，只能在 InnoDB 下使用；而bin log 是 Server 层的日志，可以在任何引擎下使用。

3. redo log 大小有限(4GB)，超过后会循环写；另外两个大小不会。
![执行过程](https://raw.githubusercontent.com/lyjgulu/mysql/main/image/redo%20log.png)

4. undo log 记录的是行记录变化前的数据；redo log 记录的是 sql 的数据页修改逻辑以及 change buffer 的变更；bin log记录操作语句对具体行的操作以及操作前的整行信息（5.7默认）或者sql语句。

5. 单独的 binlog 没有 crash-safe 能力，也就是在异常断电后，之前已经提交但未更新的事务操作到磁盘的操作会丢失，也就是主从复制的一致性无法保障，而 redo log 有 crash-safe 能力，通过与 redo log 的配合实现 "步提交"，就可以让主从库的数据也能保证一致性。(**crash-safe能力是保证即使数据库发生异常重启，之前提交的记录都不会丢失**)

6. redo log 是物理日志，它记录的是数据页修改逻辑以及 change buffer 的变更，只能在当前存储引擎下使用，而 binlog 是逻辑日志，它记录的是操作语句涉及的每一行修改前后的值，在任何存储引擎下都可以使用。

#### 日志刷盘(fsync)
