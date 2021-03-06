# MySQL中InnoDB的锁机制

### 目录
- [为什么需要锁？](#为什么需要锁？)
- [行锁和表锁](#行锁和表锁)
- [共享锁和排他锁](#共享锁和排他锁)
- [二阶段锁协议](#二阶段锁协议)
- [死锁](#死锁)
- [乐观锁](#乐观锁)
- [悲观锁](#悲观锁)
- [乐观锁之CAS解决并发扣款数据一致性问题](#乐观锁之CAS解决并发扣款数据一致性问题)
- [乐观锁之CAS解决并发扣款数据一致性问题优化，CAS下ABA问题](#乐观锁之CAS解决并发扣款数据一致性问题优化，CAS下ABA问题)


### 为什么需要锁？
数据库的锁机制使得在对数据库进行并发访问时，可以保障数据的完整性和一致性。锁冲突也是影响数据库并发访问性能的一个重要因素。锁的各种操作包括获得锁、检测锁是否是否已解除、释放锁等都是消耗资源的，。

### 行锁和表锁
行锁和表锁的粒度不同，行锁锁住的是一行或多行记录，表锁锁住的是整张表。  
行锁：开销大，加锁慢，会出现死锁，锁的粒度小，发生锁冲突的概率低，并发高。  
表锁：开销小，加锁快，不会出现死锁，锁的粒度大，发生锁冲突的概率高，并发低。  

MyISAM只支持表锁，不支持行锁，InnoDB支持表锁和行锁。InnoDB的行锁是针对索引加的行锁，如果SQL没有用到索引，会从行锁升级为表锁。  

### 共享锁和排他锁
行锁分为**共享锁**和**排他锁**。  

共享锁，又称为读锁，简称S锁。当事务对数据加上读锁后，其他事务只能对该数据加读锁，不能做任何修改操作，也就是不能添加写锁。只有当数据上的读锁被释放后，其他事务才能对其添加写锁。共享锁主要是为了支持并发的读取数据而出现的，读取数据时，不允许其他事务对当前数据进行修改操作，从而避免"不可重读"的问题的出现。  

排他锁，又称为写锁、独占锁，简称X锁。若事务T对数据对象A加上X锁，则只允许事务T读取和修改A，其他任何事务都不能再对A加任何类型的锁，直到T释放A上的锁。这就保证了其他事务在T释放A上的锁之前其他事务不能再读取和修改A。  

普通的select语句是不加锁的，select包裹在事务中，同样也是不加锁的。  

**显式加锁：**
```sql
-- 共享锁
SELECT ... LOCK IN SHARE MODE
-- 排他锁
SELECT ... FOR UPDATE
```
**隐式加锁：**  
update和delete会对查询出的记录隐式加排他锁，加锁类型和for update类似。

准备一张数据表，并插入一条数据。
```sql
CREATE TABLE `t_goods` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `name` varchar(50) NOT NULL DEFAULT '' COMMENT '商品名称',
  `repertory` int(10) NOT NULL DEFAULT '0' COMMENT '商品库存',
  PRIMARY KEY (`id`),
  KEY `idx_name` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商品表';

INSERT INTO t_goods (`name`, `repertory`) values ('商品1', 20);
```
共享锁：
```sql
-- 会话1
mysql> begin;
mysql> select * from t_goods where id=1 lock in share mode; -- 对id为1的记录加共享锁
+----+---------+-----------+
| id | name    | repertory |
+----+---------+-----------+
|  1 | 商品1   |        20 |
+----+---------+-----------+

-- 会话2
mysql> begin;
mysql> select * from t_goods where id=1 lock in share mode; -- 可以对id为1的记录加共享锁
+----+---------+-----------+
| id | name    | repertory |
+----+---------+-----------+
|  1 | 商品1   |        20 |
+----+---------+-----------+

-- 会话2
mysql> update t_goods set repertory=10 where id=1; -- 不可以对id为1的记录加排他锁
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```
排他锁：
```sql
-- 会话1
mysql> begin;
mysql> select * from t_goods where id=1 for update; -- 对id为1的记录加排他锁
+----+---------+-----------+
| id | name    | repertory |
+----+---------+-----------+
|  1 | 商品1   |        20 |
+----+---------+-----------+

-- 会话2
mysql> select * from t_goods where id=1 for update; -- 不可以对id为1的记录加排他锁
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

-- 会话2
mysql> select * from t_goods where id=1 lock in share mode; -- 不可以对id为1的记录加共享锁
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```

### 二阶段锁协议
二阶段锁：Two-phase locking(2PL)。  
InnoDB采用的是两阶段锁协议，前一个阶段为加锁，后一个阶段为解锁。在事务的执行过程中，COMMIT和ROLLBACK是解锁阶段。加锁阶段只能加锁不能解锁，一旦开始解锁，则进入解锁阶段，不能再加锁。

MySQL的行锁是在引擎层由各个引擎自己实现的。但并不是所有的引擎都支持行锁，比如MyISAM引擎就不支持行锁。不支持行锁意味着并发控制只能使用表锁，对于这种引擎的表，同一张表上任何时刻只能有一个更新在执行，这就会影响到业务并发度。InnoDB是支持行锁的，这也是MyISAM被InnoDB替代的重要原因之一。  

在下面的例子中，事务B的update语句执行时会是什么现象呢？假设字段id是表t的主键。  

![锁冲突](https://raw.githubusercontent.com/duiying/img/master/锁冲突.png)  

事务A在执行完两条update语句后，持有id为1和2两条记录的行锁(排他锁)，此时事务B的update语句会被阻塞，直到事务A执行commit后，事务A释放了id为1和2两条记录的行锁，事务B才能继续执行。  

从这个例子我们可以理解二阶段锁协议，在InnoDB事务中，行锁是在需要的时候才加上的，但并不是不需要了就立即释放锁，而是要等到事务结束时才释放，这个就是两阶段锁协议。  

知道了这个设定，对我们使用事务有什么帮助呢？那就是，如果你的事务中需要锁多个行，要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放。看下面这个例子。  

假设你负责实现一个电影票在线交易业务，顾客A要在影院B购买电影票。我们简化一点，这个业务需要涉及到以下操作：  
1. 从顾客A账户余额中扣除电影票价；
2. 给影院B的账户余额增加这张电影票价；
3. 记录一条交易日志；

也就是说，要完成这个交易，我们需要update两条记录，并insert一条记录。当然，为了保证交易的原子性，我们要把这三个操作放在一个事务中。那么，你会怎样安排这三个语句在事务中的顺序呢？  

试想如果同时有另外一个顾客C要在影院B买票，那么这两个事务冲突的部分就是语句2了。因为它们要更新同一个影院账户的余额，需要修改同一行数据。  

根据两阶段锁协议，不论你怎样安排语句顺序，所有的操作需要的行锁都是在事务提交的时候才释放的。所以，如果你把语句2安排在最后，比如按照3、1、2这样的顺序，那么影院账户余额这一行的锁时间就最少。这就最大程度地减少了事务之间的锁等待，提升了并发度。  

### 死锁
死锁是指两个或两个以上的事务在执行过程中，因争夺资源而造成的一种互相等待的现象。
![死锁](https://raw.githubusercontent.com/duiying/img/master/死锁.png)  
上图中，事务A在等待事务B释放id为2的排他锁，事务B在等待事务A释放id为1的排他锁，事务A和事务B在互相等待对方的资源释放，就是进入了死锁状态。当出现死锁以后，有两种策略：  
1. 直接进入等待，直到超时。这个超时时间可以通过参数innodb_lock_wait_timeout来设置。
2. 发起死锁检测，发现死锁后，主动回滚死锁链条中的某个事务，让其他事务得以继续执行。将参数innodb_deadlock_detect设置为on，表示开启这个逻辑。  
```sql
-- 会话1
mysql> begin;
mysql> update t_goods set repertory=1 where id=1;

-- 会话2
mysql> begin;
mysql> update t_goods set repertory=1 where id=2;

-- 会话1
mysql> update t_goods set repertory=1 where id=2;

-- 会话2
mysql> update t_goods set repertory=1 where id=1;
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

### 乐观锁
乐观并发控制(又名“乐观锁”，Optimistic Concurrency Control，缩写“OCC”)是一种并发控制的方法。它假设多用户并发的事务在处理时不会彼此互相影响，各事务能够在不产生锁的情况下处理各自影响的那部分数据。在提交数据更新之前，每个事务会先检查在该事务读取数据后，有没有其他事务又修改了该数据。如果其他事务有更新的话，正在提交的事务会进行回滚。  

乐观锁的特点先进行业务操作，不到万不得已不去拿锁。即“乐观”的认为拿锁多半是会成功的，因此在进行完业务操作需要实际更新数据的最后一步再去拿一下锁就好。  

乐观锁是一种思想，可以通过CAS来实现乐观锁。

乐观并发控制多数用于数据争用不大、冲突较少的环境中，这种环境中，偶尔回滚事务的成本会低于读取数据时锁定数据的成本，因此可以获得比其他并发控制方法更高的吞吐量。

### 悲观锁 
悲观并发控制(又名“悲观锁”，Pessimistic Concurrency Control，缩写“PCC”)是一种并发控制的方法。它可以阻止一个事务以影响其他用户的方式来修改数据。如果一个事务执行的操作读某行数据应用了锁，那只有当这个事务把锁释放，其他事务才能够执行与该锁冲突的操作。  

悲观锁的特点是先获取锁，再进行业务操作，即“悲观”的认为获取锁是非常有可能失败的，因此要先确保获取锁成功再进行业务操作。  

共享锁和排他锁是悲观锁的不同的实现，它俩都属于悲观锁的范畴。  

悲观并发控制主要用于数据争用激烈的环境，以及发生并发冲突时使用锁保护数据的成本要低于回滚事务的成本的环境中。  

### 乐观锁之CAS解决并发扣款数据一致性问题
CAS：在set写回的时候，加上初始状态的条件compare，只有初始状态不变时，才允许set写回成功，Compare And Set(CAS)，是一种常见的降低读写锁冲突，保证数据一致性的方法。  

**扣款的场景是怎样的？**  

**第一步，从数据库查询用户现有余额**  
```sql
-- 假设id为1的用户查询的余额是100
select yue from t_account where account_id = 1;
```
**第二步，业务层实现逻辑处理**  
1. 查询商品价格，比如是80
2. 查询商品是否有活动，比如是9折
3. 余额是否足够，余额足够才往下走
```php
if ($yue > 80*0.9) {
    $newYue = $yue - 80*0.9;
} else {
    return '余额不足';
}
```
**第三步，修改用户余额**  
```sql
update t_account set yue = $newYue where account_id = 1;
```

**上面这个步骤，在并发场景下会出现什么问题？**  
在并发环境下，这种"查询加修改"的业务有一定概率出现数据不一致。  

比如有业务1和业务2并发扣款：  
1. 业务1和业务2都查询到余额是100
2. 业务1经过逻辑处理，计算出余额应该扣60；业务2经过逻辑处理，计算出余额应该扣70
3. 业务1对用户余额先修改，改为了40；业务2对用户余额后修改，改为了30
此时异常出现了，用户原余额100，业务1扣款60，业务2扣款70，最后余额为30。  

**常见的解决方案?**  
可以对每一个用户进行分布式锁互斥，例如：在redis/zk里抢到一个key才能继续操作，否则禁止操作。这种悲观锁方案确实可行，但要引入额外的组件(redis/zk)，并且会降低吞吐量。  

**对于小概率的不一致，有没有乐观锁的方案呢?**  
使用CAS机制解决：在set写回的时候，加上初始状态的条件compare，只有初始状态不变时，才允许set写回成功。  

在上面这个两个业务同时扣款的例子，只需要将：
```sql
update t_account set yue = $newYue where account_id = 1;
-- 改为
update t_account set yue = $newYue where account_id = 1 and yue = $yue;
```
并发执行时：
```sql
-- 业务1
update t_account set yue = 40 where account_id = 1 and yue = 100;
-- 业务2
update t_account set yue = 30 where account_id = 1 and yue = 100;
```
这样，两个业务并发扣款时，只有1个业务能执行成功。  

**怎么判断哪个业务执行成功，哪个业务执行失败呢？**
   
update操作，其实无所谓成功或者失败，业务能通过affect rows来判断：  
- 写回成功的，affect rows为1
- 写回失败的，affect rows为0

**总结：** 高并发"查询并修改"的场景，可以用CAS(Compare and Set)的方式解决数据一致性问题。对应到业务，即在set的时候，加上初始条件的比对即可。  

### 乐观锁之CAS解决并发扣款数据一致性问题优化，CAS下ABA问题
前面提到，用CAS机制可以在尽量不影响吞吐量的情况下，保证数据的一致性。  

**什么是ABA问题？**  
CAS机制确实能够提升吞吐，并保证一致性，但在极端情况下可能会出现ABA问题。  

考虑如下操作：
1. 并发1(上)：数据的初始值是A，后续计划实施CAS乐观锁，期望数据仍是A的时候，修改才能成功
2. 并发2：将数据修改成B
3. 并发3：将数据修改回A
4. 并发1(下)：CAS乐观锁，检测发现初始值还是A，进行数据修改

上述并发环境下，并发1在修改数据时，虽然还是A，但已经不是初始条件的A了，中间发生了A变B，B又变A的变化，此A已经非彼A，数据却成功修改，可能导致错误，这就是CAS引发的所谓的ABA问题。  

余额操作，出现ABA问题并不会对业务产生影响，因为对于"余额"属性来说，前一个A为100余额，与后一个A为100余额，本质是相同的。  

但其他场景未必是这样，举一个堆栈操作的例子：  
![ABA问题](https://raw.githubusercontent.com/duiying/img/master/ABA问题.jpeg)   
此时会出现系统错误，因为此"A1"非彼"A1"。  

**ABA问题可以怎么优化？**  

导致ABA问题的原因，是CAS过程中只简单进行了"值"的校验，在有些情况下，"值"相同不会引入错误的业务逻辑(例如余额)，有些情况下，"值"虽然相同，却已经不是原来的数据了(例如堆栈)。  

因此，CAS不能只比对"值"，还必须确保是原来的数据，才能修改成功。  

常见的实践是，将"值"比对，升级为"版本号"的比对，一个数据一个版本，版本变化，即使值相同，也不应该修改成功。  

余额并发读写例子，引入版本号的具体实践如下：  
首先，用户表要加一个version字段。t_account(id,account_id,yue,version)
```sql
-- 查询余额时，同时查询版本号
select yue,version from t_account where account_id = 1;
```
然后，设置余额时，由原来的值比对，改为版本比对。  
```sql
update t_account set yue = 30, version = version + 1 where account_id = 1 and version = $oldVersion;
```
此时假设有并发操作，业务1会修改版本号+1，业务2并发操作会执行失败。  

**总结：**   
- select&set业务场景，在并发时会出现一致性问题
- 基于"值"的CAS乐观锁，可能导致ABA问题
- CAS乐观锁，必须保证修改时的"此数据"就是"彼数据"，应该由"值"比对，优化为"版本号"比对

### 参考
- [07 | 行锁功过：怎么减少行锁对性能的影响？](https://www.cnblogs.com/a-phper/p/10313876.html)
- [并发扣款，如何保证数据的一致性？](https://mp.weixin.qq.com/s/QSpBDlW1KktJ8iHaYcO2rw)
- [并发扣款一致性优化，CAS下ABA问题](https://zhuanlan.51cto.com/art/201909/602496.htm)

