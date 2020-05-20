# mysql

## mysql存储引擎

+ InnoDB 支持事务，支持行级别锁定，支持 B-tree、Full-text 等索引，不支持 Hash 索引；<br>
+ MyISAM 不支持事务，支持表级别锁定，支持 B-tree、Full-text 等索引，不支持 Hash 索引；<br>
+ Memory 不支持事务，支持表级别锁定，支持 B-tree、Hash 等索引，不支持 Full-text 索引；<br>
+ NDB 支持事务，支持行级别锁定，支持 Hash 索引，不支持 B-tree、Full-text 等索引；<br>
+ Archive 不支持事务，支持表级别锁定，不支持 B-tree、Hash、Full-text 等索引；<br>


## 常用命令

+ 创建索引：create (index_type) index index_name on table_name<br>
+ 加表锁：LOCK TABLE tablename locktype, tablename2 locktype<br>
+ locktype：READ只能读表, read LOCAL, write, low priority write<br>
+ 解锁：UNLOCK TABLE[S],解锁不需要表名，会解除当前用户所有的表锁<br>


## 数据库连接池的连接数

通用公式：cpu * 2 + 有效磁盘数（ssd， hhd或其他）
    

## mysql性能调优

1. 拆分复杂查询，是一个大的复杂查询还是多个简单查询好；
2. 分解关联查询，将join的on拆解为多个表的简单where查询，然后组合；优点：让缓存效率更高；查询分解后，单个查询可以减少锁的竞争；
3. 排序，尽量通过索引进行排序，要避免对大量数据进行排序，数据量大，则需要使用磁盘，否则会使用内存；limit时尽量保证使用索引
4. 子查询，union的时候如果需要排序或者limit，那么在union的各个子查询里面使用；子查询尽量使用关联查询代替；
5. 使用辅助索引时，尽量减少范围查询的范围；尽可能选择唯一性较强的字段建立索引
    

## MyISAM与InnoDB的区别

1. InnoDB支持事物ACID(Atomicity原子性、Isolation隔离性、Consistency一致性、Durability持久性;事务并发会引起脏读（读到未提交的数据，如果回滚或再次修改提交，则读到的是假数据）、不可重复读（A读，B改，A在读，不一样）、幻读（A读没读到，B加，A在读，读到了，删除也是一样）问题;事务隔离级别，读未提交read uncommitted（均不能避免），读提交read committed（避免脏读），可重复读repeatable read（默认，避免脏读和不可重复读），串行序列化serializable（均可阻止）)，而MyISAM不支持事物；mysql默认是采用auto coomit自动提交的，不手动的开启事务，那么每个sql都会当做一个事务提交，
2. InnoDB支持行级锁，而MyISAM只支持表级锁
3. InnoDB支持MVCC(多版本控制), 而MyISAM不支持
4. InnoDB支持外键，而MyISAM不支持
5. InnoDB不支持全文索引，而MyISAM支持

2者selectcount(*)哪个更快？<br>
myisam更快，因为myisam内部维护了一个计数器，可以直接调取。

mysql的日志：错误日志、查询日志、慢查询日志、binlog、中继日志（也是二进制日志）、事务日志（redo和undo）
    

## InnoDB的MVCC多版本控制原理与流程

MVCC流程：
1. 事务开启，获得一个事务版本号，有mysql分发，自增
2. 获得一个read view
3. 查询到数据，进行read view事务版本号进行匹配
4. 不符合read view规则的，则从undo log中获取历史版本数据
5. 返回符合规则的数据

他有以下几个核心：
1. 事务版本号；
2. 表的隐藏列；
3. undo log；
4. read view；

+ 表隐藏列：<br>
    DB_TRX_ID: 记录操作该数据事务的事务ID；<br>
    DB_ROLL_PTR：指向上一个版本数据在undo log 里的位置指针；<br>
    DB_ROW_ID: 隐藏ID ，当创建表没有合适的索引作为聚集索引时，会用该隐藏ID创建聚集索引;

+ undo log:<br>
    undo log 主要用于记录数据被修改之前的日志，在表信息修改之前先会把数据拷贝到undo log 里，当事务进行回滚时可以通过undo log里的日志进行数据还原。用于MVCC快照读的数据，在MVCC多版本控制中，通过读取undo log的历史版本数据可以实现不同事务版本号都拥有自己独立的快照数据版本;

+ read view:<br>
    在innodb 中每个SQL语句执行前都会得到一个read_view。副本主要保存了当前数据库系统中正处于活跃（没有commit）的事务的ID号，其实简单的说这个副本中保存的是系统中当前不应该被本事务看到的其他事务id列表；<br>
    
    他的四个属性与匹配规则：
    + trx_ids: 当前系统活跃(未提交)事务版本号集合。
    + low_limit_id: 创建当前read view 时“当前系统最大事务版本号##1“；当前事务id<up_limit_id时显示；如果满足，可以肯定该数据是在当前事务启之前就已经存在了的,所以可以显示。
    + up_limit_id: 创建当前read view 时“系统正处于活跃事务最小版本号”；数据事务ID>low_limit_id则不显示；如果满足，则说明该数据是在当前read view创建之后才产生的，所以数据不予显示;
    + creator_trx_id: 创建当前read view的事务版本号；
    当处于最大和最小之间时，这个数据有可能是在当前事务开始的时候还没有提交的；这时需结合trx_ids进行匹配；<br>1.如果事务ID不存在于trx_ids 集合（则说明read view产生的时候事务已经commit了），这种情况数据则可以显示；<br>2.如果当前事务id存在集合中，且等于creator_trx_id，说明是自己产生的，肯定就可见了；否则产生rw的时候，数据没提交，也不是自己的，就不可见；不满足，则从undo log取历史数据，然后数据历史版本事务号回头再来和read view 条件匹配 ，直到找到一条满足条件的历史数据，或者找不到则返回空结果；

mysql读取时分两种：
快照读，读undo log，无读写时加锁的问题；当前读，读当前数据库最新的数据，很明显需要加锁；
    

## InnoDB的存储结构
  
InnoDB分为聚集索引和辅助索引；他有页的概念，页管理磁盘的一些块；每张表只能有一个聚合索引，它使用此结构来实现的：

BTree是一种平衡树，他的每个节点包含多个kv组，k为键，v为值；索引定义一个大于1的值为他的'度'，h为它的高度，每个非叶子节点由n-1个k与n个指针，其中n在[d,2d]之间；每个叶子节点至少包含一个k与两个指针，指针均为null；所有的叶子节点高度相同，为h；k与指针相互间隔，两端为指针；树满足中序遍历，节点内，k递增；
新插入的数据会影响当前树的结构，所以会涉及分裂、合并、转移等操作；

B+Tree为BTree的优化变种，变化为非叶子节点不存储v，只存储k；每个叶子节点都有指向下一个和上一个相邻节点的指针；通常在B+Tree上有两个头指针，一个指向根节点，一个指向k最小的叶子节点，这样可以方便主键的分页、范围查找、或者从根节点的随机查找；

辅助索引也是用的B+Tree，但是他不存储整行的数据，只存储哪个字段的，但是他有一个标签bookmark字段，会存储当前行所在的页的位置；
  
