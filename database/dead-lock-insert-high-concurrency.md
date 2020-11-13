# 并发insert死锁验证

## 数据库

Server version:		5.7.31 MySQL Community Server (GPL) 
###  隔离级别： repeatable read
###  表结构：
desc track_lock;
```
+-------------+--------------+------+-----+---------+-------+
| Field       | Type         | Null | Key | Default | Extra |
+-------------+--------------+------+-----+---------+-------+
| id          | varchar(100) | NO   | PRI | NULL    |       |
| status      | int(2)       | NO   |     | NULL    |       |
| create_date | timestamp    | YES  |     | NULL    |       |
+-------------+--------------+------+-----+---------+-------+
```
## 验证场景1：三个事务同时插入相同主键的记录。事务1正常提交，事务2，3插入失败，没有死锁问题。
| 时间线 | 事务1                                                        | 事务2                                                      | 事务3                                                     |
| ------ | ------------------------------------------------------------ | ---------------------------------------------------------- | --------------------------------------------------------- |
| 1      | begin;                                                       | begin;                                                     | begin;                                                    |
| 2      | insert into track_lock (id, status) values ('1','1');<br/>Query OK, 1 row affected (0.01 sec) |                                                            |                                                           |
| 3      |                                                              | insert into track_lock (id, status) values ('1','1'); 等待 | insert into track_lock (id, status) values ('1','1');等待 |
| 4      | commit;<br/>Query OK, 0 rows affected (0.01 sec)             |                                                            |                                                           |
| 5      |                                                              | ERROR 1062 (23000): Duplicate entry '1' for key 'PRIMARY'  | ERROR 1062 (23000): Duplicate entry '1' for key 'PRIMARY' |
| 6      |                                                              | commit;                                                    | commit;                                                   |

##  验证场景2：三个事务同时插入相同主键的记录。事务1插入后回滚，事务2，3一个插入成功，一个死锁错误。

| 时间线 | 事务1                                                        | 事务2                                                        | 事务3                                                     |
| ------ | ------------------------------------------------------------ | ------------------------------------------------------------ | --------------------------------------------------------- |
| 1      | begin;                                                       | begin;                                                       | begin;                                                    |
| 2      | insert into track_lock (id, status) values ('1','1');<br/>Query OK, 1 row affected (0.01 sec) |                                                              |                                                           |
| 3      |                                                              | insert into track_lock (id, status) values ('1','1'); 等待   | insert into track_lock (id, status) values ('1','1');等待 |
| 4      | rollback;<br/>Query OK, 0 rows affected (0.02 sec)           |                                                              |                                                           |
| 5      |                                                              | ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction | Query OK, 1 row affected (8.39 sec)                       |
| 6      |                                                              | commit;                                                      | commit;                                                   |

## 总结：
1. 高并发insert会产生死锁的可能，主键索引下插入相同的记录，事务1插入记录尚未提交，事务2，3同时也插入操作，在事务1回滚事务时候会造成事务2，3有一个插入成功，有一个死锁错误。
2. 业务系统设计时候，尽量避免高并发插入相同记录业务，可在数据库之前做过滤。如Java锁，分布式锁。



---

##  insert锁机制

这个为什么出现共享锁呢，网上找到这个大牛的文章跟这里的问题是一样的[记一次神奇的Mysql死锁排查](https://mp.weixin.qq.com/s/ZjKor6bKv9ak_RhYMUqg0Q)。但insert操作具体加锁步骤说明，我看后还是不理解。又找到一篇文章[Insert into 加锁机制](https://blog.csdn.net/and1kaney/article/details/51214001) 

> 官方文档对于insert 加锁的描述：
> INSERT sets an exclusive lock on the inserted row. This lock is an index-record lock, not a next-key lock (that is, there is no gap lock) and does not prevent other sessions from inserting into the gap before the inserted row.Prior to inserting the row, a type of gap lock called an insertion intention gap lock is set. This lock signals the intent to insert in such a way that multiple transactions inserting into the same index gap need not wait for each other if they are not inserting at the same position within the gap.If a duplicate-key error occurs, a shared lock on the duplicate index record is set. This use of a shared lock can result in deadlock should there be multiple sessions trying to insert the same row if another session already has an exclusive lock. 
>
> 大体的意思是：insert会对插入成功的行加上排它锁，这个排它锁是个记录锁，而非next-key锁（当然更不是gap锁了），不会阻止其他并发的事务往这条记录之前插入记录。在插入之前，会先在插入记录所在的间隙加上一个插入意向gap锁（简称I锁吧），并发的事务可以对同一个gap加I锁。如果insert 的事务出现了duplicate-key error ，事务会对duplicate index record加共享锁。这个共享锁在并发的情况下是会产生死锁的，比如有两个并发的insert都对要对同一条记录加共享锁，而此时这条记录又被其他事务加上了排它锁，排它锁的事务提交或者回滚后，两个并发的insert操作是会发生死锁的。

还有这边文章说的[Insert语句的加锁流程](http://mysql.taobao.org/monthly/2020/09/06/) :

>隐式锁主要用在插入场景中。在Insert语句执行过程中，必须检查两种情况，一种是如果记录之间加有间隙锁，为了避免幻读，此时是不能插入记录的，另一中情况如果Insert的记录和已有记录存在唯一键冲突，此时也不能插入记录。除此之外，insert语句的锁都是隐式锁，但跟踪代码发现，insert时并没有调用lock_rec_add_to_queue函数进行加锁, 其实所谓隐式锁就是在Insert过程中不加锁。
>只有在特殊情况下，才会将隐式锁转换为显示锁。这个转换动作并不是加隐式锁的线程自发去做的，而是其他存在行数据冲突的线程去做的。例如事务1插入记录且未提交，此时事务2尝试对该记录加锁，那么事务2必须先判断记录上保存的事务id是否活跃，如果活跃则帮助事务1建立一个锁对象，而事务2自身进入等待事务1的状态

以及[INSERT 加锁流程](https://www.aneasystone.com/archives/2017/12/solving-dead-locks-three.html)

> 1. 首先对插入的间隙加插入意向锁（Insert Intension Locks）
>
>  - 如果该间隙已被加上了 GAP 锁或 Next-Key 锁，则加锁失败进入等待；
>  - 如果没有，则加锁成功，表示可以插入；
>
> 2. 然后判断插入记录是否有唯一键，如果有，则进行唯一性约束检查
>
>  - 如果不存在相同键值，则完成插入
>  - 如果存在相同键值，则判断该键值是否有锁
>    - 如果没有锁， 判断该记录是否被标记为删除
>      - 如果标记为删除，说明事务已经提交，还没来得及 purge，这时等待行S锁，重新进行唯一性约束检查。
>      - 如果没有标记删除，则报 1062 duplicate key 错误；
>        - 如果这个记录有活跃事务则给活跃事务加上X记录锁。当前事务等待行 S 锁，重新进行唯一性约束检查。
>        - 如果没有活跃事务，当前事务返回唯一键冲突错误。
>    - 如果有锁，说明该记录正在处理（新增、删除或更新），且事务还未提交，等待行S锁，重新进行唯一性约束检查。
>
> 3. 插入记录并对记录加 X 记录锁；

   

总结下 用具体3个事务插入相同记录 在主键字段，模拟死锁情况。

1. 事务1，2，3找到相应的gap添加插入意向间隙锁I。并发的事务可以对同一个gap加I锁。为了防止幻读，如果记录之间加有 GAP 锁或 Next-Key 锁，此时不能 INSERT。
2. 事务1插入成功，尚未提交事务。
3. 事务2插入相同记录，出现了duplicate-key error ，事务2会给事务1对duplicate index record加排他锁X。事务2等待共享锁S
4. 事务3插入相同记录，发现有X锁，等待共享锁S
5. 如果事务1提交，则释放X锁。
   6. 事务2，事务3同时获得S锁，重新进行唯一性约束检查。
   2. 事务2，事务3同时获得S锁，然后发现唯一冲突结束。没有死锁。
6. 如果事务1回滚，则释放X锁。
   1. 事务2，事务3同时获得S锁，重新进行唯一性约束检查。
   2. 事务2，事务3发现可以插入记录，先去获取X锁。
   3. 事务2，事务3各自持有S锁，又同时获取X锁，导致死锁。
   4. 事务2，事务3有一个报死锁错误后释放锁，另外一个插入成功。

## 总结

高并发insert会产生死锁的可能，主键索引下插入相同的记录，事务1插入记录尚未提交，事务2，3同时也插入操作，在事务1回滚事务后，事务2，3有一个死锁错误而释放S锁，从而另外一个事务则可以获取锁成功执行。

