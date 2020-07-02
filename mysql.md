##Mysql

###1、mysql的事务隔离级别
* 脏读，事务未提交，其做的变更别的事务能看到
* 可重复读，视图和事务id，一个事务执行过程中看到的数据跟启动时看到的数据一致，如果他没有修改这个数据的话。可重复读的场景是在事务过程中，其他事务对数据的修改不影响已经修改的状态，即认为事务启动时的视图是静态的。实现方式是通过readview视图和事务id。
* 读提交，事务提交后其做的变更才能被其他事务看到
* 串行化，一个事务完成后才能进行下一个事务
* MVCC，多版本控制协议
* 乐观锁和悲观锁
	* 操作共享数据时，“悲观锁”认为数据出现冲突的可能性很大，而“乐观锁”认为大部分情况不会出现冲突，进而觉得是否采用排他性措施。

	
###2、一条sql语句是怎么执行的
* 客户端连接到mysql的server-->连接器
* 连接器从缓存查询是否有结果，有结果返回，没有的话就进入分析器-->分析器
* 分析器进行词法分析，语法分析。-->优化器
* 优化器进行执行计划生成，索引的选择。-->执行器
* 执行器操作存储引擎，查询返回结果。-->innodb


###3、mysql的update语句
* redo-log（Write-Ahead Logging），关键点先写日志，在写磁盘。当需要更新操作的时候，先把记录写入redo-log，然后更新内存，这时候就算是更新完成了。当系统不忙的时候，在将redo-log的变更写入到磁盘里。
* binlog。归档日志，分为statement和row两个级别，也有混合级别。redo-log是循环写的，而binlog是一直追加的。
* 两阶段提交
	* redo-log和binlog都可以表示事物的提交状态，两阶段提交就是为了让两个状态保持逻辑上的一致性，
	* redo-log刷盘有不同的配置选项，可以直接写入，也可以生成一个group以后一起写入
	* innodb_flush_log_at_trx_commit参数表明redo-log的刷盘策略，默认是1，表示每次都写入磁盘；0表示事物执行中日志放到redo-log的bugger中，事务提交以后直接写如果redo-log的文件中；2表示事务提交后也写入redo-log的buffer中，异步去刷盘
	
###4、数据库索引
* 等值查询和范围查询对不同的索引模型有不同的性能反应，等值查询适合hash索引方式，访问时间复杂度为O(1)，范围查询则更需要有序数组。
* InnoDB是以主键顺序一索引的形式存储，使用B+树模型
* 基于主键索引查询和基于普通索引查询的区别
	* 主键索引不需要回表，查询结果后直接返回
	* 普通索引不包含值，索引查询到的只是对应的主键值，需要再次通过主键索引去查找数据
	* 如果普通索引的结果满足需要的查询返回值，那么也会直接返回

* 如果索引数据页数据满了，需要重新申请数据页，然后将部分数据移动过去，这样也会造成瓶颈
* 如果数据页数据有删除，也会将数据页合并，然后删除分页
* 主键越小，普通索引节点的叶子节点就越小，那么普通索引所占的空间也就越小
* 覆盖索引：如果普通索引上查到的值可以直接提供查询结果，那么就不需要回表，索引结果覆盖了查询结果，所以这样就称为覆盖索引。
* B+树这种索引结构，可以利用索引的“最左前缀”来标记记录
* 建立联合索引的时候，索引内部的字段顺序的安排，应该是以索引的复用能力为准则。有了最左前缀的条件，当有了(a,b)这个索引以后，就没必要再字段a上创建索引
* 索引下推：可以再索引查找的过程中，对索引中包含的字段先判断，直接过滤掉不满足条件的记录，减少回表次数。

###5、MySql的锁
* 全局锁，表级锁和行级锁
* 全局锁，整库加锁，全局备份的时候
* 表级锁，为表加锁，还有一种是元数据锁（metadata lock），MDL不会显示使用，但是在访问表的时候会自动加上。可以保证写的一致性，保证在变更过程中该表的元数据不会发生变更。对所有表的增删改查都是需要先申请MDL锁的
* 给小表加字段的时候导致整个表不可用
	* 如果该表的查询很多，那么，当有一个事务开启后进行了一次查询，在事务没提交的时候，其拿到的MDL读锁并不会释放，那么如果此时发生了一个alter操作，该操作会阻塞住（因为拿不到DBL写锁），如果此时再过来一个查询，就同样只能被锁住，那么此时这个表就相当于不可用了。
	* 要安全给小标加字段就需要
	* 尽量减小长事务的出现
	* 给alter的变更语句加等待时间，超时后退出重试
* 行锁
	* 两阶段锁。

###6、事务隔离
* mvcc中的快照，readView和行数据事务，利用undo-log（回滚日志）和行数据版本。
* 事务一致性视图，数据版本可见性规则如下
![](./picture/trx.png)
* 事务启动的时候同样保存了当前活动事务的id，在高低水位之间判断时同样会对比活动事务id

###7、实战问题
（1）唯一索引和普通索引的选择

* 唯一索引不能使用change buffer，唯一索引找到满足条件的语句后就不会再查找，普通索引会继续查找
* 普通索引加change buffer可能提升系统性能，redo-log节省磁盘随机写造成的性能影响，而change buffer主要节省随机读造成的io消耗

（2）Mysql为什么会选错索引

* 优化器的逻辑：
	* 扫描行数的判读，统计信息判断，如果统计信息不准确，那么有可能会造成优化器判断失误，造成选错索引
	* 1、可以通过force index强制指定索引
	* 2、通过修改mysql语句，引导mysql选择正确的索引
	* 3、创建合适的索引

（3）字符串加索引

* mysql支持前缀索引，当然前缀索引可能造成查询扫描的数据量增加
* 合理的前缀索引可以节省空间，同时加快查询速度
* 使用前缀索引就没法使用覆盖索引带来的优化效果了
* 可以使用倒叙索引
* 使用hash字段

（4）数据库为什么会抖一下？

* 刷脏页，将内存中的数据刷入磁盘永久保存
* 什么情况下会刷脏页呢？
	* redo-log写满了
	* 系统内存不足，需要淘汰脏页，这些淘汰的脏页就被刷进内存
	* mysql认为系统空闲的时候，也就主动进行刷脏页的动作
	* mysql正常关闭的时候需要把内存中的脏页刷入到磁盘中

（5）数据删了一半，文件大小确没变

* 数据页的重用造成的
* 删除数据、插入数据都可能造成数据页空洞
* 解决？重建表

（6）count(*)太慢？

* 全表扫描
* InnoDB没法存储行数据
* 解决方式：
	* 缓存计数保存
	* 数据库保存计数

（7）order by怎么工作的

* sort buffer。查询数据，放到sort buffer缓存
* 如果buffer放不下，分段缓存合并
* 如果单行数据太大，可以只排序需要的字段列和主键字段
* rowid排序

（8）如果现实随机数据

* 如果表没有主键id，那么InnoDB会自动创建rowid作为主键
* order by rand() 使用了内存临时表，内存临时表排序的时候使用了 rowid 排序方法。
* 解决？修改sql，通过计算随机位置以后再去查找数据。

``` sql
	select count(*) into @C from t;
	set @Y1 = floor(@C * rand());
	set @Y2 = floor(@C * rand());
	set @Y3 = floor(@C * rand());
	select * from t limit @Y1，1； //在应用代码里面取Y1、Y2、Y3值，拼出SQL后执行
	select * from t limit @Y2，1；
	select * from t limit @Y3，1；
```

（9）为啥逻辑相同，但是性能差距很大？

* 对索引字段做的函数操作会破坏原有的字段的有序性，会让优化器放弃使用索引
* 隐式类型转换也是函数操作，也会有这样的问题
* 隐式字符转换
* 连接过程中要求在被驱动表的索引字段上加函数操作，是直接导致对被驱动表做全表扫描的原因

（10）为什么执行一行也很慢？

* 长时间不返回，表被锁住了
* 等MDL锁，一个线程正在表t上请求或持有MSL锁，从而把select语句阻塞住了
* 等flush，flush单表或者flush全表
* 等行锁
* 慢查询，执行了全表扫描

（11）mysql怎么保证数据不丢失，崩溃恢复

* binlog写入
* redo-log写入

（12）查询大量数据会不会把数据库打爆

* 不会，结果集会存在netbuffer中，而net buffer是有大小限制的，而且不是一次性存取然后写入，是获取一部分就返回一部分
* 全表扫描可能会影响缓存命中率

（13）mysql的join

* 选择合适的驱动表
* 如果join_buffer放不下，那就分段放入
* 尽量使用小表做驱动表
* 优化join：
	* 如果按照主键的递增顺序查询的话，对磁盘的读比较接近顺序读，能够提升读性能
	
（14）自增主键为什么不连续

（15）insert语句锁为什么这么多

（16）怎么样快速复制一个表

（17）自增id用完了怎么办？