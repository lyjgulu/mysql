## mysql事务(InnoDB)

### 事务基本点
1. 事务的特性
    - ACID(原子性、一致性、隔离性、持久性)
2. 四种隔离级别
    - 读未提交：脏读，不可重复读，幻读
    - 读已提交：不可重复读，幻读
    - 可重复读：幻读(Next-Key lock解决)
    - 可串行化：无问题(一般不使用该隔离级别)
      **不可重复读的重点是修改，幻读的重点在于新增或者删除。**

3. 数据库里面会创建一个视图(快照读)，访问的时候以视图的逻辑结果为准。
    - 在“可重复读”隔离级别下，这个视图是在事务启动时创建的，整个事务存在期间都用这个视图。 
    - 在“读提交”隔离级别下，这个视图是在每个SQL语句开始执行的时候创建的。
    - “读未提交”隔离级别下直接返回记录上的最新值，没有视图概念。
    - 而“串行化”隔离级别下直接用加锁(当前读)的方式来避免并行访问。
    
4. 当前读与快照读
    - 当前读：DML语句语句(X锁，排他锁)，select lock in share mode(S锁，共享锁)
    - 快照读：MVCC
   
### MVCC(undo log部分看mysql的各种log)
- MVCC适用于读提交与可重复读的隔离级别
    1. 读提交在每个SQL语句开始执行的时候创建快照。(read view)
    2. 可重复读在事务启动时创建唯一快照。(reda view)
    3. **更新数据都是先读后写的，而这个读，只能读当前的值，称为“当前读”。（current read）**
- 判断trx_ids方法
![transactionId](https://raw.githubusercontent.com/lyjgulu/mysql/main/image/transactionId.png)
    - up_limit_id：当前已经提交的事务号 + 1，事务号 < up_limit_id ，对于当前Read View都是可见的。理解起来就是创建Read View视图的时候，之前已经提交的事务对于该事务肯定是可见的。
    - low_limit_id：当前最大的事务号 + 1，事务号 >= low_limit_id，对于当前Read View都是不可见的。理解起来就是在创建Read View视图之后创建的事务对于该事务肯定是不可见的。
    - trx_ids：为活跃事务id列表，即Read View初始化时当前未提交的事务列表。所以当进行RR读的时候，trx_ids中的事务对于本事务是不可见的（除了自身事务，自身事务对于表的修改对于自己当然是可见的）。理解起来就是创建RV时，将当前活跃事务ID记录下来，后续即使他们提交对于本事务也是不可见的。 
- MVCC详图
![transactionId](https://raw.githubusercontent.com/lyjgulu/mysql/main/image/mvcc%2Bundolog.png)

- 解决幻读(详细看mysql的锁机制)
    - 主要是依靠行级别锁。

- PostgreSQL与MySQL在MVCC实现上的区别(扩展点)
   - MySQL采用MVCC与悲观锁(Next-Key lock)结合。(具体看mysql锁机制)
      - 所有DDL的sql语句会分配唯一id(回滚id)，读操作依据当前事务的可见trx_id(//TODO)
   - PostgreSQL采用MVCC与乐观锁结合
      - 所有的事务在执行之前都会被分配一个唯一的时间戳，每一个数据项都有读写两个时间戳。
      - 不阻塞读请求，直接返回最新值。
      - 写操作在执行时，事务的时间戳一定要大或者等于数据行的读时间戳，否则就会被回滚。
