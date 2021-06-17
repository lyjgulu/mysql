## mysql的 Buffer 详解

### Buffer Pool

- Buffer Pool淘汰策略(LRU，与redis的LRU实现思想一致，方式大不相同)
![BufferPoolLRU](https://raw.githubusercontent.com/lyjgulu/mysql/main/image/BufferPoolLRU.png)
    - 新插入的页加入到list中间，分成两部分，在头部。
    - 最近新访问形成年轻代 New Sublist 占总list 5/8
    - 在尾部，访问过的页形成 Old Sublist，占总list 3/8
    - 在 Old Sublist 中的页被再次访问，会将此页放入到 New Sublist的头部。
        - 特例一，有一些读取到Buffer Pool中被访问几次后就再也不会被访问了，可以设置innodb_old_blocks_time参数来控制多长时间内被访问不会放到 New Sublist。
        - 特例二，不执行严格的LRU淘汰策略，将热点数据长时间存储在Buffer Pool。

- Buffer Pool预读
    - 预读是需要将大量的叶加载到 Buffer Pool 中，提高 IO 性能
        1. 线性预读，按Buffer Pool中顺序访问的页来预测即将访问的页。
        2. 随机预读，根据Buffer Pool中已有页，不管顺序，预读可能需要的页(比方一个索引在物理上总共15页，Buffer Pool中有14页，剩下一页会采取预读)

- Buffer Pool监视器部分，需要可自查

- Buffer Pool 配置(主要提高系统性能)
    1. 将Buffer Pool尽可能的调整更大。
    2. 将Buffer Pool拆分为多个部分，解决线程瓶颈问题。
    3. 将热点数据长时间保存在Buffer Pool中，不进行淘汰。
    4. 使用预读策略，异步加载页到Buffer Pool中。
    5. 控制刷盘策略(详细看下面的flush)
    6. 正常退出的mysql Buffer Pool机制(详细看下面的flush，由后台线程执行)

#### buffer pool flush(内存脏页刷盘)(redo log部分看mysql的各种log)
1. InnoDB的redo log写满了。这时候系统会停止所有更新操作，把checkpoint往前推进，redo log留出空间可以继续写。
![flushByRedoLog](https://raw.githubusercontent.com/lyjgulu/mysql/main/image/flushByRedoLog.png)
2. 系统内存不足。当需要新的内存页，而内存不够用的时候，就要淘汰一些数据页，空出内存给别的数据页使用。如果淘汰的是“脏页”，就要先将脏页写到磁盘。刷脏页一定会写盘，就保证了每个数据页有两种状态：
    - 一种是内存里存在，内存里就肯定是正确的结果，直接返回；
    - 另一种是内存里没有数据，就可以肯定数据文件上是正确的结果，读入内存后返回。这样的效率最高。
3. 系统空闲时，后台将内存脏页刷盘
4. mysql正常关闭，会将内存脏页刷盘(可配置)

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

### change buffer
- change buffer 存在原因: 执行DML操作时，索引列通常是未排序的，需要大量 I/O 操作更新，一些辅助索引不在 buffer pool 中，使用 change buffer,可以避免大量 I/O 操作。
- 在 change buffer 中记录的页被加载到 buffer pool 中，主线程会将被更新的页合并，稍后会刷新到磁盘。合并时机：
    1. 服务器空闲时间
    2. 服务器缓慢关闭时
- 优点：
    1. 有大量DML操作的应用程序适合使用，减少大量 I/O 操作。(读多写少)
    2. 适合有较多二级索引的表。
- 缺点：
    1. 占用 buffer pool 空间。
    2. 不适合二级索引较少的表
- 默认占用 buffer pool 25%空间，最高可调到50%

### log buffer(redo log buffer和undo log buffer)
- 日志缓冲区是保存要写入磁盘上日志文件的数据的内存区域。

### Doublewrite Buffer(2MB) 
- Doublewrite 包括2MB大小的 Doublewrite Buffer 和磁盘共享表空间中2MB大小连续的128个页(2个区)
- buffer pool 脏页刷盘，由于数据库 I/O 最小单位是16k，文件系统 I/O 最小单位为 4Kb，I/O 写入存在 page 损坏风险，redo log 只能恢复整个脏块，需要 Doublewrite Buffer 辅助
![pageDamage](https://raw.githubusercontent.com/lyjgulu/mysql/main/image/pageDamage.png)
- 工作流程
![pageDamage](https://raw.githubusercontent.com/lyjgulu/mysql/main/image/DoublewriteProcess.png)
    1. 当一系列机制触发数据缓冲池中的脏页刷新时，并不直接写入磁盘数据文件中，而是先拷贝至内存中的doublewrite buffer中；
    2. 接着从两次写缓冲区分两次写入磁盘共享表空间中(连续存储，顺序写，性能很高)，每次写1MB；
    3. 待第二步完成后，再将 doublewrite buffer 中的脏页数据写入实际的各个表空间文件(离散写)；(脏页数据固化后，即进行标记对应doublewrite数据可覆盖)
- 优点：保证数据的完整性；缺点：增加 fsync 操作，增加写负载






changebuffer merge
从磁盘读入数据页到内存（老版本的数据页）；
从 change buffer 里找出这个数据页的 change buffer 记录 (可能有多个），依次应用，得到新版数据页；
写 redo log。这个 redo log 包含了数据的变更和 change buffer 的变更。