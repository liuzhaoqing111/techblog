# 死锁问题-先insert后update导致

# 现象
服务日志报错，明显的发现：Deadlock found when trying to get lock; try restarting transaction，是在执行update操作时候出现的错误。这个明显是两个事务互相持有锁而又等待锁导致的，数据库隔离级别是RR。但具体是哪种情况导致的，需要仔细分析。下面看看业务逻辑。
# 背景
这个是一个获取数据库锁的逻辑，数据库中有个锁的表，多个任务有相同的ID通过往锁表中插入记录来实现分布式锁，没有记录时候，第一个插入成功的获取锁，如果没有插入成功，则进一步更新这个记录, 将失败状态更新成进行中，更新成功表示获得锁。
```sql
create table lock(
	id varchar(20) primary key not null,
	status varchar(1) not null, -- 1: 进行中，2：成功, 3: 失败
	times timestamp not null,
);
```

# 分析

## 业务逻辑
这个业务逻辑很简单，在一个事务里面，先进行insert, 如果没有成功再进行update，看代码
```java
@Transactional(rollbackFor = Exception.class)
public boolean tryLock(String key){
	if (tryLockInsert(key)) {
		return true;
	} else {
	  return tryLockUpdate(key)
	}
}
```
```sql
-- insert: 返回值>0表示成功
insert into lock (id, status, times) values('key', '1', sysdate);

-- update: 返回值>0表示成功
update lock set status = "1" where id = 'key' and status = '3'
```

## 分析1

一个事务里面两次写操作，而且修改的记录都是通过主键确定的一条记录，感觉来说，多个事务同时操作，应该是一个事务获取这个记录在主键索引上面的X锁，然后执行两次写。其他的事务等到第一个事务两次写完后释放X锁才能进行，以之前的认识来看应该不会出现死锁的可能呢。

## 分析2
继续分析数据库的死锁信息SHOW ENGINE INNODB STATUS\G
```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2020-11-10 05:41:02 0x7f872c170700
*** (1) TRANSACTION:
TRANSACTION 12399, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 182, OS thread handle 140218145097472, query id 3535 172.28.0.1 root updating
UPDATE track_lock  SET status=1,
create_date='2020-11-10 13:41:02.095'

 WHERE (id = 'test')
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 137 page no 3 n bits 96 index PRIMARY of table `pvuv`.`track_lock` trx id 12399 lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 74657374; asc test;;
 1: len 6; hex 000000003066; asc     0f;;
 2: len 7; hex 56000001a50a15; asc V      ;;
 3: SQL NULL;
 4: SQL NULL;
 5: len 5; hex 99a7d4da42; asc     B;;
 6: len 1; hex 31; asc 1;;

*** (2) TRANSACTION:
TRANSACTION 12400, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
3 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 175, OS thread handle 140218537019136, query id 3543 172.28.0.1 root updating
UPDATE track_lock  SET status=1,
create_date='2020-11-10 13:41:02.102'

 WHERE (id = 'test')
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 137 page no 3 n bits 96 index PRIMARY of table `pvuv`.`track_lock` trx id 12400 lock mode S locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 74657374; asc test;;
 1: len 6; hex 000000003066; asc     0f;;
 2: len 7; hex 56000001a50a15; asc V      ;;
 3: SQL NULL;
 4: SQL NULL;
 5: len 5; hex 99a7d4da42; asc     B;;
 6: len 1; hex 31; asc 1;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 137 page no 3 n bits 96 index PRIMARY of table `pvuv`.`track_lock` trx id 12400 lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 74657374; asc test;;
 1: len 6; hex 000000003066; asc     0f;;
 2: len 7; hex 56000001a50a15; asc V      ;;
 3: SQL NULL;
 4: SQL NULL;
 5: len 5; hex 99a7d4da42; asc     B;;
 6: len 1; hex 31; asc 1;;

*** WE ROLL BACK TRANSACTION (2)
```
从日志看出：
1. 事务trx id 12399 lock_mode X locks rec but not gap waiting。 (1) WAITING FOR THIS LOCK TO BE GRANTED
2. 事务trx id 12400 lock mode S locks rec but not gap。(2) HOLDS THE LOCK(S)
3. 事务trx id 12400 lock_mode X locks rec but not gap waiting。(2) WAITING FOR THIS LOCK TO BE GRANTED:
两个事务，事务2持有共享锁，等待获取排他锁，事务1等待获取排他锁。

## 分析3 insert锁机制

这个为什么出现共享锁呢，网上找到这个大牛的文章跟这里的问题是一样的[记一次神奇的Mysql死锁排查](https://mp.weixin.qq.com/s/ZjKor6bKv9ak_RhYMUqg0Q)。但insert操作具体加锁步骤说明，我看后还是不理解。又找到一篇文章[Insert into 加锁机制](https://blog.csdn.net/and1kaney/article/details/51214001) 

> 官方文档对于insert 加锁的描述：
> INSERT sets an exclusive lock on the inserted row. This lock is an index-record lock, not a next-key lock (that is, there is no gap lock) and does not prevent other sessions from inserting into the gap before the inserted row.Prior to inserting the row, a type of gap lock called an insertion intention gap lock is set. This lock signals the intent to insert in such a way that multiple transactions inserting into the same index gap need not wait for each other if they are not inserting at the same position within the gap.If a duplicate-key error occurs, a shared lock on the duplicate index record is set. This use of a shared lock can result in deadlock should there be multiple sessions trying to insert the same row if another session already has an exclusive lock. 
>
> 大体的意思是：insert会对插入成功的行加上排它锁，这个排它锁是个记录锁，而非next-key锁（当然更不是gap锁了），不会阻止其他并发的事务往这条记录之前插入记录。在插入之前，会先在插入记录所在的间隙加上一个插入意向gap锁（简称I锁吧），并发的事务可以对同一个gap加I锁。如果insert 的事务出现了duplicate-key error ，事务会对duplicate index record加共享锁。这个共享锁在并发的情况下是会产生死锁的，比如有两个并发的insert都对要对同一条记录加共享锁，而此时这条记录又被其他事务加上了排它锁，排它锁的事务提交或者回滚后，两个并发的insert操作是会发生死锁的。

还有这边文章说的[Insert语句的加锁流程](http://mysql.taobao.org/monthly/2020/09/06/) :
>隐式锁主要用在插入场景中。在Insert语句执行过程中，必须检查两种情况，一种是如果记录之间加有间隙锁，为了避免幻读，此时是不能插入记录的，另一中情况如果Insert的记录和已有记录存在唯一键冲突，此时也不能插入记录。除此之外，insert语句的锁都是隐式锁，但跟踪代码发现，insert时并没有调用lock_rec_add_to_queue函数进行加锁, 其实所谓隐式锁就是在Insert过程中不加锁。
只有在特殊情况下，才会将隐式锁转换为显示锁。这个转换动作并不是加隐式锁的线程自发去做的，而是其他存在行数据冲突的线程去做的。例如事务1插入记录且未提交，此时事务2尝试对该记录加锁，那么事务2必须先判断记录上保存的事务id是否活跃，如果活跃则帮助事务1建立一个锁对象，而事务2自身进入等待事务1的状态

以及[INSERT 加锁流程](https://www.aneasystone.com/archives/2017/12/solving-dead-locks-three.html)
> 1. 首先对插入的间隙加插入意向锁（Insert Intension Locks）
>  - 如果该间隙已被加上了 GAP 锁或 Next-Key 锁，则加锁失败进入等待；
>  - 如果没有，则加锁成功，表示可以插入；
> 2. 然后判断插入记录是否有唯一键，如果有，则进行唯一性约束检查
>  - 如果不存在相同键值，则完成插入
>  - 如果存在相同键值，则判断该键值是否有锁
>    - 如果没有锁， 判断该记录是否被标记为删除
>      - 如果标记为删除，说明事务已经提交，还没来得及 purge，这时等待行S锁，重新进行唯一性约束检查。
>      - 如果没有标记删除，则报 1062 duplicate key 错误；
>        - 如果这个记录有活跃事务则给活跃事务加上X记录锁。当前事务等待行 S 锁，重新进行唯一性约束检查。
>        - 如果没有活跃事务，当前事务返回唯一键冲突错误。
>    - 如果有锁，说明该记录正在处理（新增、删除或更新），且事务还未提交，等待行S锁，重新进行唯一性约束检查。
> 3. 插入记录并对记录加 X 记录锁；

   

总结下 用具体3个事务插入相同记录 在主键字段，模拟死锁情况。

1. 事务1，2，3找到相应的gap添加插入意向间隙锁I。并发的事务可以对同一个gap加I锁。为了防止幻读，如果记录之间加有 GAP 锁或 Next-Key 锁，此时不能 INSERT。
2. 事务1插入成功，尚未提交事务。
3. 事务2插入相同记录，出现了duplicate-key error ，事务2会给事务1对duplicate index record加排他锁X。事务2等待共享锁S
4. 事务3插入相同记录，发现有X锁，等待共享锁S
5. 如果事务1提交，则释放X锁。
   1. 事务2，事务3同时获得S锁，重新进行唯一性约束检查。
   2. 事务2，事务3同时获得S锁，然后发现唯一冲突结束。没有死锁。
6. 如果事务1回滚，则释放X锁。
   1. 事务2，事务3同时获得S锁，重新进行唯一性约束检查。
   2. 事务2，事务3发现可以插入记录，先去获取X锁。
   3. 事务2，事务3各自持有S锁，又同时获取X锁，导致死锁。
   4. 事务2，事务3有一个报死锁错误后释放锁，另外一个插入成功。

按照出现死锁的情况8分析，死锁出现情况是在一个事务insert未完成前，至少有2个insert事务同时对这个记录进行insert操作才会产生。


### 思考：Insert Intention Locks作用

Insert Intention Locks的引入，我理解是为了提高数据插入的并发能力。
如果没有Insert Intention Locks的话，可能就需要使用Gap Locks来代替。



## 分析4： insert update 死锁流程

回到最初的问题上来，出现死锁情况是高并发下的对相同记录insert update,分析下加锁流程，日志记录中说明锁在update过程中产生，并且不是gap lock，按照日志中事务1等待X锁，事务2持有S锁，等待X锁分析可能情况：

2. 事务0执行insert，X锁尚未提交事务
2. 事务1执行insert后duplicate-key error，等待S锁
3. 事务2执行insert后duplicate-key error返回，等待S锁
4. 事务0commit。
5. 事务1，事务2同时获得S锁，
6. 事务1，事务2执行update操作
7. 事务1，事务2获取相同记录的X锁等待，等待对方释放S锁。
8. 事务1，事务2一个发现死锁后释放锁，另一个事务获得X锁执行update。

# 解决
1. 将RR隔离级别，降低成RC隔离级别。这里RC隔离级别会用快照读，从而不会加S锁。
2. 再插入的时候使用select * for update,加X锁，从而不会加S锁。
3. 可以提前加上分布式锁，可以利用Redis,或者ZK等等，
4. 根据业务情况，将insert和update分成两个事务执行。不影响业务正确性。

第一种方法不太现实，毕竟隔离级别不能轻易的修改。第三种方法又比较麻烦。所以第二种方法简单些。

上述解决方案1，2，3是按照[记一次神奇的Mysql死锁排查](https://mp.weixin.qq.com/s/ZjKor6bKv9ak_RhYMUqg0Q)上面来的，另外添加的第四条是我按照业务情况找到另外的方案。

# 总结

1. 避免对相同的记录，进行高并发的写操作。
   1. 把数据库唯一索引的特性做成分布式锁使用，容易死锁
   2. 把数据库唯一索引的特性实现接口冪等性，容易死锁。
2. 避免一个事务里面对一个记录，进行多次修改，这样会增加锁的复杂性。

