# Database

## 四大特性-ACID

1. **原子性**：事务是数据库的最小逻辑工作单位，事务中包含的各操作要么全部提交成功，要么全部失败回滚。
2. **一致性**： 任务执行的结果必须是使数据库从一个一致性状态变到另一个一致性状态。因此当数据库只包含成功事务提交的结果时，就说数据库处于一致性状态。如果数据库系统 运行中发生故障，有些事务尚未完成就被迫中断，这些未完成事务对数据库所做的修改有一部分已写入物理数据库，这时数据库就处于一种不正确的状态，或者说是不一致的状态。
3. **隔离性**：一个事务的执行不能其它事务干扰。即一个事务内部的操作及使用的数据对其它并发事务是隔离的，并发执行的各个事务之间不能互相干扰。
4. **持续性**：也称永久性，指一个事务一旦提交，它对数据库中的数据的改变就应该是永久性的。接下来的其它操作或故障不应该对其执行结果有任何影响。

### ACID靠什么保证的？

- 原子性(atomicity) 由undo log日志保证，它记录了需要回滚的日志信息，事务回滚时撤销已经执行成功的sql
- 一致性(consistency) 一般由代码层面来保证
- 隔离性(isolation) 由MVCC来保证
- 持久性(durability) 由内存+redo log来保证，mysql修改数据同时在内存和redo log记录这次操作，事务提交的时候通过redo log刷盘，宕机的时候可以从redo log恢复

## 四种隔离级别

### Read Uncommitted（读取未提交内容）

在该隔离级别，所有事务都可以看到其他未提交事务的执行结果。本隔离级别很少用于实际应用，因为它的性能也不比其他级别好多少。读取未提交的数据，也被称之为**脏读**（Dirty Read）。

### Read Committed（读取提交内容）

这是大多数数据库系统的默认隔离级别（但不是MySQL默认的）。它满足了隔离的简单定义：一个事务只能看见已经提交事务所做的改变。这种隔离级别也支持所谓的**不可重复读**（Nonrepeatable Read），因为同一事务的其他实例在该实例处理其间可能会有新的commit，所以同一select可能返回不同结果。

### Repeatable Read（可重读）

这是MySQL的默认事务隔离级别，它确保同一事务的多个实例在并发读取数据时，会看到同样的数据行。不过理论上，这会导致另一个棘手的问题：**幻读**（Phantom Read）。简单的说，幻读指当用户读取某一范围的数据行时，另一个事务又在该范围内插入了新行，当用户再读取该范围的数据行时，会发现有新的“幻影” 行。InnoDB和Falcon存储引擎通过MVCC机制（快照读）解决了该问题。

> #### Multi-Version Concurrency Control（多版本并发控制）
>  
> 一种并发控制机制，在数据库中用来控制并发执行的事务，控制事务隔离进行。
> 通过**保存数据在某个时间点的快照**来进行控制的（undo log）。允许同一个数据记录拥有多个不同的版本。如果在查询时通过添加相对应的约束条件，就能获取用户想要的对应版本的数据。  
> **innodb的实现重点：**
>
> - 隐式字段
> - 把修改前的数据存放于undo log，通过回滚指针与主数据关联
> - ReadView

### Serializable（可串行化）

这是最高的隔离级别，它通过强制事务排序，使之不可能相互冲突，从而解决幻读问题。简言之，它是在每个读的数据行上加上共享锁。在这个级别，可能导致大量的超时现象和锁竞争。

## MySql两大引擎

**InnoDB**：支持外键、行锁、事务安全，适合多并发和QPS较高的场景， update insert
**MyISAM**：有索引的顺序访问方法，只支持表级锁，不支持外键， select insert

1. **自动增长**
    - myisam引擎的自动增长列必须是索引，如果是组合索引，自动增长可以不是第一列，他可以根据前面几列进行排序后递增。
    - innodb引擎的自动增长列必须是索引，如果是组合索引也必须是组合索引的第一列。
2. **主键**
    - myisam允许没有任何索引和主键的表存在，myisam索引文化和数据结构是分离的,索引保存数据记录的地址。
    - innodb引擎如果没有设定主键或者非空唯一索引，就会自动生成一个6字节的主键(用户不可见)，innodb的数据是主索引的一部分，表数据文件本身就是按照B+Tree组织的一个索引结构，附加索引保存的是主索引的值。
3. **count()函数**
    - myisam保存有表的总行数，如果`select count(\*) from table`;会直接取出出该值
    - innodb没有保存表的总行数，如果使用`select count(\*) from table`；就会遍历整个表，消耗相当大，但是在加了where条件后，myisam和innodb处理的方式都一样。
4. **全文索引**
    - myisam支持 FULLTEXT类型的全文索引
    - innodb不支持FULLTEXT类型的全文索引，但是innodb可以使用sphinx插件支持全文索引，并且效果更好。（sphinx   是一个开源软件，提供多种语言的API接口，可以优化mysql的各种查询）
5. **辅助索引**
   - 在MyISAM中，主索引和辅助索引在结构上没有任何区别，只是主索引要求key是唯一的，而辅助索引的key可以重复。
   - InnoDB的所有辅助索引都引用主键作为data域。
6. **delete from table**
使用这条命令时，innodb不会从新建立表，而是一条一条的删除数据，在innodb上如果要清空保存有大量数据的表，最好不要使用这个命令。(推荐使用truncate table，不过需要用户有drop此表的权限)

## B 树与 B+ 树

- B+树内节点不存储数据，所有数据存储在叶节点导致查询时间复杂度固定为 log n
- B-树查询时间复杂度不固定，与 key 在树中的位置有关，最好为O(1)
- B+树叶节点两两相连可大大增加区间访问性，可使用在范围查询等
- B+树更适合外部存储(存储磁盘数据)。由于内节点无 data 域，每个节点能索引的范围更大更精确。

## 索引

### 聚簇索引

数据的一种存储方式，**数据与索引在一个文件中**。它的辅助索引的叶子节点存储的是指向行的指针，存放的就是整张表的行记录数据。一般情况下InnoDB会默认创建聚簇索引，按照每张表的主键构造一颗B+树，一张表只允许存在一个聚簇索引。（唯一性）

- **数据访问更快**
由于行数据和叶子节点存储在一起，同一页中会有多条数据，访问同一数据的不同行记录时，已经把也加载到了Buffer中，再次访问的时候，会在内存中完成访问，不必访问磁盘，这样由于主键和行数据时一起被载入磁盘的，找到叶子节点就可以立即将行数据返回了
- **聚簇索引对主键的排序查找和范围查找速度非常快**
聚簇索引适合用在排序的场合，非聚簇索引不适合，取出一定范围数据的时候，使用聚簇索引更快
- **插入速度严重依赖于插入顺序**
- **维护索引很昂贵**

### 非聚簇索引 （辅助索引）

**将数据存储于索引分开结构**，索引结构的叶子节点指向了数据的对应行。
辅助索引访问数据总是需要二次查找，非聚簇索引都是辅助索引，像复合索引、前缀索引、唯一索引、辅助引擎叶子节点存储的不再是行的物理位置，而是**主键值**。

### Q&A

#### 主键索引是聚集索引还是非聚集索引?

在InnoDB下主键索引是聚集索引，在MyISAM下主键索引是非聚集索引

#### 聚集索引和非聚集索引的区别?

- 聚簇索引的叶子结存存放的是主键值和数据行，支持覆盖索引； 二级索引的叶子节点存放的是主键值会在执行数据行的指针。
- 由于叶子节点（数据页）只能按照一颗B+树排序，因此一张表只能由一个聚簇索引；辅助索引的存在不影响聚簇索引中数据的组织，所以一张表可以有多个辅助索引。
- 聚集索引存储记录是物理上连续存在，⽽⾮聚集索引是逻辑上的连续，物理存储并不连续。  

### 索引类别

- **Primary Key（聚集索引）**：InnoDB存储引擎的表会存在主键（唯一非null），如果建表的时候没有指定主键，则会使用第一非空的唯一索引作为聚集索引，否则InnoDB会自动帮你创建一个不可见的、长度为6字节的row_id用来作为聚集索引。
- **单列索引**：单列索引即一个索引只包含单个列
- **组合索引**：组合索引指在表的多个字段组合上创建的索引，只有在查询条件中使用了这些字段的左边字段时，索引才会被使用。使用组合索引时**遵循最左前缀集合**
  - 一个查询可以只使用索引中的一部分，但只能是最左侧部分
- **Unique（唯一索引）**：索引列的值必须唯一，但**允许空值**。若是组合索引，则列值的组合必须唯一。主键索引是一种特殊的唯一索引，不允许有空值
- **Key（普通索引）**：是MySQL中的基本索引类型，允许在定义索引的列中插入重复值和空值
- **FULLTEXT（全文索引）**：全文索引类型为FULLTEXT，在定义索引的列上支持值的全文查找，允许在这些索引列中插入重复值和空值。全文索引可以在CHAR、VARCHAR或者TEXT类型的列上创建
- **SPATIAL（空间索引）**：空间索引是对空间数据类型的字段建立的索引，MySQL中的空间数据类型有4种，分别是GEOMETRY、POINT、LINESTRING和POLYGON。MySQL使用SPATIAL关键字进行扩展，使得能够用于创建正规索引类似的语法创建空间索引。创建空间索引的列必须声明为NOT NULL

#### 命中索引但还是执行慢的原因？

- 命中的索引可能不是最优的索引，需要重新调整索引的设置
- 索引字段重复或者空置太多
- 查询范围太广，形成全索引扫描
- 没有利用到覆盖索引导致产生回表现象
- 索引字段数据分布太随机，导致即便没有回表查询但是大量随机IO

## 日志

日志是mysql数据库的重要组成部分，记录着数据库运行期间各种状态信息。mysql日志主要包括错误日志、查询日志、慢查询日志、事务日志、二进制日志几大类。
**逻辑日志**：可以简单理解为记录的就是sql语句。
**物理日志**：因为mysql数据最终是保存在数据页中的，物理日志记录的就是数据页变更。

### binlog（归档日志）

binlog属于MySQL Server层面，属于逻辑日志，记录所有更新且提交了数据或者已经潜在更新提交了数据，以二进制的形式记录这个语句的原始逻辑，依靠binlog是没有crash-safe能力的。

**作用**：**主从复制**：在Master端开启binlog，然后将binlog发送到各个Slave端，Slave端重放binlog，实现主从同步。

### redo log（重做日志）

redo log是InnoDB存储引擎层的日志，通常是**物理日志**，记录的是数据页的物理修改，不管事务是否提交都会记录下来，它用来恢复提交后的物理数据页(恢复数据页，且只能恢复到最后一次提交的位置)。

**作用**：a)**确保事务的持久性**。防止在发生故障的时间点（断电），尚有脏页未写入磁盘，在重启mysql服务的时候，可以根据redo log日志进行恢复，也就达到了**crash-safe**，从而达到事务的持久性。b)**数据恢复**

### undo log（回滚日志）

undo用来回滚行记录到某个版本(执行反向操作)。undo log一般是**逻辑日志**，根据每行记录进行记录。可实现事务的**原子性**。

**作用**：保存了事务发生之前的数据的一个版本，可以用于回滚，同时可以提供多版本并发控制下的读（MVCC），也即非锁定读。

### redo log和binlog区别

- redo log是属于innoDB层面，binlog属于MySQL Server层面的，这样在数据库用别的存储引擎时可以达到一致性的要求。
- redo log是物理日志，记录该数据页更新的内容；binlog是逻辑日志，记录的是这个更新语句的原始逻辑
- redo log是循环写，日志空间大小固定；binlog是追加写，是指一份写到一定大小的时候会更换下一个文件，不会覆盖。
- binlog可以作为恢复数据使用，主从复制搭建，redo log作为异常宕机或者介质故障后的数据恢复使用。
- crash safe是靠redo log来保障的，并不靠binlog。只不过为了奔溃恢复后的数据与binlog 一致，某些有commit 缺失的redo log 在恢复时要根据binlog 是否写入来决定是否恢复

## 从准备更新一条数据到事务的提交的流程

![更新数据流程](asset/更新数据流程.png)

## SQL语句

### 确认查询是否使用索引

```sql
explain select surname,first_name from a,b where a.id=b.id
```

- `type` 反应查询语句的性能，应保证查询至少达到 `range` 级别
- `possible_keys`: SQL查询时用到的索引
- `key` 显示SQL实际决定查询结果使用的键(索引)。如果没有使用索引，值为NULL
- `rows` 显示MySQL认为它执行查询时必须检查的行数

### 索引操作

```sql
CREATE INDEX创建索引:CREATE [UNIQUE|FULLTEXT|SPATIAL] INDEX index_name ON  table_name (col_name[length],...) [ASC|DESC]
ALTER TABLE 创建索引:ALTER TABLE table_name ADD [UNIQUE|FULLTEXT|SPATIAL][INDEX|KEY][index_name] (col_name[length],...) [ASC|DESC]
ALTER TABLE 删除索引:ALTER TABLE table_name DROP INDEX index_name
DROP INDEX  删除索引:DROP INDEX index_name ON table_name
```

### 存储引擎操作

```sql
查看mysql支持的引擎：show engines;
查看当前默认的引擎： show variables like 'default_storage_engine';
查看指定表的引擎：   show create table xf_card;
修改指定表的引擎：   alter table xf_card engine=innodb;
```

### 日志操作

```sql
日志是否启用: show variables like 'log%';
当前日志:    show master status;
```

#### for update

for update是一种行级锁，排它锁，一旦用户对某个行施加了行级加锁，则该用户可以查询更新被加锁的数据行，其它用户只能查询但不能更新被加锁的数据行。select for update 是为了在查询时，对这条数据进行加锁，避免其他用户以该表进行插入，修改或删除等操作，造成表的不一致性。

### 索引失效的条件

- where子句中使用!=或<>操作符
- where 进行 null 值判断（where num is null）
- where 子句中使用 or 来连接条件，可以使用union all方案代替
- where 中in 和 not in 也要慎用，能用 between 就不要用 in
- where like 不能前置百分号
- where 子句中使用参数（where num=@num）
- where 子句中对字段进行表达式操作（where num/2=100）
- where子句中对字段进行函数操作（where substring(name,1,3)=’abc’）
- order by 涉及的列没有索引

### 存储过程

存储过程是一个预编译的SQL语句，优点是允许模块化的设计，就是说只需创建一次，以后在该程序中就可以调用多次。如果某次操作需要执行多次SQL，使用存储过程比单纯SQL语句执行要快。可以用一个命令对象来调用存储过程。

### SQL语句优化

- Where子句中：where表之间的连接必须写在其他Where条件之前，那些可以过滤掉最大数量记录的条件必须写在Where子句的末尾.HAVING最后。
- 用EXISTS替代IN、用NOT EXISTS替代NOT IN。
- 避免在索引列上使用计算。
- 避免在索引列上使用IS NULL和IS NOT NULL
- 对查询进行优化，应尽量避免全表扫描，首先应考虑在 where 及 order by 涉及的列上建立索引。
- 应尽量避免在 where 子句中对字段进行 null 值判断，否则将导致引擎放弃使用索引而进行全表扫描。
- 应尽量避免在 where 子句中对字段进行表达式操作，这将导致引擎放弃使用索引而进行全表扫描。

### 避免SQL注入

1. **过滤输入内容，校验字符串**
过滤输入内容就是在数据提交到数据库之前，就把用户输入中的不合法字符剔除掉。可以使用编程语言提供的处理函数或自己的处理函数来进行过滤，还可以使用正则表达式匹配安全的字符串。

2. **参数化查询**
预防 SQL 注入攻击最有效的方法。参数化查询是指在设计与数据库连接并访问数据时，在需要填入数值或数据的地方，使用参数（Parameter）来给值。数据库服务器不会将参数的内容视为 SQL 语句的一部分来进行处理，而是在数据库完成 SQL 语句的编译之后，才套用参数运行。因此就算参数中含有破坏性的指令，也不会被数据库所运行。
MySQL 的参数格式是以“?”字符加上参数名称而成，如下所示：

    ```sql
    UPDATE myTable SET c1 = ?c1, c2 = ?c2, c3 = ?c3 WHERE c4 = ?c4
    ```

3. **安全测试、安全审计**
除了开发规范，还需要合适的工具来确保代码的安全。我们应该在开发过程中应对代码进行审查，在测试环节使用工具进行扫描，上线后定期扫描安全漏洞。在开发过程中避免 SQL 注入:
   1. **避免使用动态SQL**
   避免将用户的输入数据直接放入 SQL 语句中，最好使用准备好的语句和参数化查询，这样更安全。
   1. **不要将敏感数据保留在纯文本中**
   加密存储在数据库中的私有/机密数据，这样可以提供了另一级保护，以防攻击者成功地排出敏感数据。
   1. **限制数据库权限和特权**
   将数据库用户的功能设置为最低要求；这将限制攻击者在设法获取访问权限时可以执行的操作。
   1. **避免直接向用户显示数据库错误**
   攻击者可以使用这些错误消息来获取有关数据库的信息。

## Redis

Redis是一种支持key-value等多种数据结构的存储系统。可用于缓存，事件发布或订阅，高速队列等场景。支持网络，提供字符串，哈希，列表，队列，集合结构直接存取，基于内存，可持久化。

- 读写性能优异
  - Redis能读的速度是110000次/s,写的速度是81000次/s
- 数据类型丰富
  - Redis支持二进制案例的 Strings, Lists, Hashes, Sets 及 Ordered Sets 数据类型操作。
- 原子性
  - Redis的所有操作都是原子性的，同时Redis还支持对几个操作全并后的原子性执行。
- 丰富的特性
  - Redis支持 publish/subscribe, 通知, key 过期等特性。
- 持久化
  - Redis支持RDB, AOF等持久化方式
- 发布订阅
  - Redis支持发布/订阅模式
- 分布式

### 为什么Redis是单线程 (为什么Redis这么快)？

- redis完全基于内存,绝大部分请求是纯粹的内存操作,非常快速。
- 数据结构简单,对数据操作也简单,redis中的数据结构是专门进行设计的
- 采用单线程模型, 避免了不必要的上下文切换和竞争条件, 也不存在多线程或者多线程切换而消耗CPU, 不用考虑各种锁的问题, 不存在加锁, 释放锁的操作, 没有因为可能出现死锁而导致性能消耗
- 使用了多路IO复用模型,非阻塞IO
- 使用底层模型不同,它们之间底层实现方式及与客户端之间的 通信的应用协议不一样,Redis直接构建了自己的VM机制,因为一般的系统调用系统函数的话,会浪费一定的时间去移动和请求

### 缓存穿透

- 问题来源: 缓存穿透是指**缓存和数据库中都没有的数据**，而用户不断发起请求。由于缓存是不命中时被动写的，并且出于容错考虑，如果从存储层查不到数据则不写入缓存，这将导致这个不存在的数据每次请求都要到存储层去查询，失去了缓存的意义。
- 解决方案
  - 接口层增加校验，如用户鉴权校验，id做基础校验，id<=0的直接拦截；
  - 从缓存取不到的数据，在数据库中也没有取到，这时也可以将key-value对写为key-null，缓存有效时间可以设置短点，如30秒（设置太长会导致正常情况也没法使用）。这样可以防止攻击用户反复用同一个id暴力攻击
  - 布隆过滤器。bloomfilter就类似于一个hash set，用于快速判某个元素是否存在于集合中，其典型的应用场景就是快速判断一个key是否存在于某容器，不存在就直接返回。布隆过滤器的关键就在于hash算法和容器大小

### 缓存雪崩

- 问题来源: 缓存雪崩是指**缓存中数据大批量到过期时间，而查询数据量巨大，引起数据库压力过大甚至down机**。和缓存击穿不同的是，缓存击穿指并发查同一条数据，缓存雪崩是不同数据都过期了，很多数据都查不到从而查数据库。
- 解决方案
  - 缓存数据的过期时间设置随机，防止同一时间大量数据过期现象发生。
  - 缓存数据库分布式部署，将热点数据均匀分布在不同的缓存数据库中。
  - 设置热点数据永远不过期。
