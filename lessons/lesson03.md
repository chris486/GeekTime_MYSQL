# 03、事务

### 1、事务相关概念

**事务**

事务用于保证一组数据库操作，要么全部成功，要么全部失败；
事务支持是在引擎层实现的；
不是所有引擎都支持事务，如原生的 MyISAM ；

**特性 ACID**

Atomicity       原子性
Consistency     一致性
Isolation       隔离性
Durability      持久性

### 2、隔离性与隔离级别

> 多个事务同时执行的时候，可能出现脏读（dirty read）、不可重复读（non-repeatable read）、幻读（phantom read）的问题

隔离级别：

- 读未提交（read uncommitted）：一个事务还没提交时，它做的变更就能被别的事务看到。
- 读提交（read committed）：一个事务提交之后，它做的变更才会被其他事务看到。
- 可重复读（repeatable read）：一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的。当然在可重复读隔离级别下，未提交变更对其他事务也是不可见的。
- 串行化（serializable ）：对于同一行记录，“写”会加“写锁”，“读”会加“读锁”。当出现读写锁冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行。

> 对于一些从 Oracle 迁移到 MySQL 的应用，为保证数据库隔离级别的一致，你一定要记得将 MySQL 的隔离级别设置为“读提交”。

配置的方式是，将启动参数 ` transaction-isolation ` 的值设置成` READ-COMMITTED `。

` show variables like 'transaction_isolation'; ` 查看当前值

### 3、事务隔离的实现

**多版本并发控制（MVCC）**：同一条记录在系统中可以存在多个版本

> **回滚日志什么时候删除？**
系统会判断当没有事务需要用到这些回滚日志的时候，回滚日志会被删除;

> **什么时候不需要了？**
当系统里么有比这个回滚日志更早的read-view的时候;

> **为什么尽量不要使用长事务?**
长事务意味着系统里面会存在很老的事务视图，在这个事务提交之前，回滚记录都要保留，这会导致大量占用存储空间。除此之外，长事务还占用锁资源，可能会拖垮库。

### 4、事务的启动方式

- 显式启动事务语句， `begin` 或 `start transaction`。配套的提交语句是 `commit`，回滚语句是 `rollback`。
- `set autocommit=0`，这个命令会将这个线程的自动提交关掉，直到执行 `commit` 或 `rollback` 语句，或者断开连接。

> 建议你总是使用 set autocommit=1, 通过显式语句的方式来启动事务。

在 `information_schema` 库的 `innodb_trx` 这个表中查询长事务

**eg：查找持续时间超过 60s 的事务**
```
select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>60
```