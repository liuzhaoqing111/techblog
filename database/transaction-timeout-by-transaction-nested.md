# 事务REQUIRES_NEW引起的数据库连接数耗尽
## 现象：
数据库写入请求，在20并发情况下，出现获取数据库链接超时失败情况。SQLTransientConnectionException: HikariPool-1 - Connection is not available, request timed out after 30005ms.

## 业务说明：
业务背景执行定时任务的业务，分成三个步骤，第一先更新状态，并将状态记录到数据库提交。第二步则是执行任务内容，第三个更新状态。

执行execute方法，要保证a方法完成后，提交数据库，再执行后续写入操作。即一个大事务中间包有3个事务包含Propagation.REQUIRES_NEW。

```java
@Transactional
public void execute(){
	setStatus();
	doTask();
	updateStatus();
}
@Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
public void setStatus(){}

@Transactional
public void doTask(){}

@Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
public void updateStatus(){}
```
## 分析

1. 执行execute会开启一个事务，获取一个数据库连接。
2. 执行setStatus Propagation.REQUIRES_NEW，会将当前事务挂起（数据库连接挂起不释放），开启一个新的事务（获取一个新的数据库连接）
3. setStatus执行完提交事务，释放一个连接，现在仍有一个数据库连接挂起。

根据上述分析可以得知，执行setStatus时候会导致挂起当前连接获取新连接。客户端默认只有10个连接，如果现在并发数大于可用数据库连接数时候，会导致多个线程持有一个连接同时去获取另一连接，而不会先释放连接。所有线程持有连接而等待回去另外链接，导致程序出现上述错误。

## 解决
避免持有一个数据库链接同时再去获取另外一个连接的情况。

```java
public void execute(){
	setStatus();
	try{
		doTask();
	} catch(Exception e){
	// log
	} finally {
		updateStatus();
	}
}
@Transactional(rollbackFor = Exception.class)
public void setStatus(){}

@Transactional(rollbackFor = Exception.class)
public void doTask(){}

@Transactional(rollbackFor = Exception.class)
public void updateStatus(){}
```
## 总结
数据库事务传播属性认识的不深刻，Propagation.REQUIRES_NEW会导致挂起事务，挂起数据库连接情况。