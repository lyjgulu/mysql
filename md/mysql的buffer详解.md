## mysql的 Buffer 详解

### Buffer Pool
1. change buffer
- todo（09讲）

  log buffer区包含(redo log buffer和undo log buffer)
  binlog buffer

changebuffer merge
从磁盘读入数据页到内存（老版本的数据页）；
从 change buffer 里找出这个数据页的 change buffer 记录 (可能有多个），依次应用，得到新版数据页；
写 redo log。这个 redo log 包含了数据的变更和 change buffer 的变更。

#### buffer pool flush(内存脏页刷盘)