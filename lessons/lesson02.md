# 02、日志系统

## 1、物理日志--重做日志 redo log

redo log：InnoDB引擎特有日志

innodb_flush_log_at_trx_commit 这个参数设置成 1 的时候，表示每次事务的 redo log 都直接持久化到磁盘，这样可以保证 MySQL 异常重启之后数据不丢失。

crash-safe:InnoDB 通过 redo log 来保证即使数据库发生异常重启，之前提交的记录都不会丢失

WAL（Write-Ahead logging）技术: 先写日志，再写磁盘

- 当有一条记录需要更新的时候，InnoDB 引擎就会先把记录写到 redo log 里面，并更新内存，这个时候更新就算完成了。然后，InnoDB 引擎会在适当（系统空闲）的时候，将这个操作记录更新到磁盘里面。

- InnoDB 的 redo log 是固定大小的，比如可以配置为一组 4 个文件，每个文件的大小是 1GB，那么这个文件总共就可以记录 4GB 的操作。从头开始写，写到末尾就又回到开头循环写

![ redo log ](../pic/02_1.png)

## 2、逻辑日志--归档日志 binlog

bin log：server层特有日志

sync_binlog 这个参数设置成 1 的时候，表示每次事务的 binlog 都持久化到磁盘，这样可以保证 MySQL 异常重启之后 binlog 不丢失。

redo.log & binlog 的区别：
* redo.log 是InnoDB特有，binlog 是server层实现，所有引擎可用；
* redo.log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”；
* redo log 是循环写的，空间固定会用完；binlog 是追加写的。文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

执行器和 InnoDB 引擎在执行 update 语句时的内部流程:
1. 执行器先找引擎取 ID=2 这一行。ID 是主键，引擎直接用树搜索找到这一行。如果 ID=2 这一行所在的数据页本来就在内存中，就直接返回给执行器；否则，需要先从磁盘读入内存，然后再返回。 
2. 执行器拿到引擎给的行数据，把这个值加上 1，比如原来是 N，现在就是 N+1，得到新的一行数据，再调用引擎接口写入这行新数据。 
3. 引擎将这行新数据更新到内存中，同时将这个更新操作记录到 redo log 里面，此时 redo log 处于 prepare 状态。然后告知执行器执行完成了，随时可以提交事务。 
4. 执行器生成这个操作的 binlog，并把 binlog 写入磁盘。 
5. 执行器调用引擎的提交事务接口，引擎把刚刚写入的 redo log 改成提交 commit 状态，更新完成（两阶段提交）。

![log](../pic/02_2.png)

## 3、两阶段提交

redo log 的写入有两个步骤：prepare 和 commit

redo log 和 binlog 都可以用于表示事务的提交状态，而两阶段提交就是让这两个状态保持逻辑上的一致。

#### 关于误删数据表后，数据的恢复步骤

1. 找到最近的一次全量备份，从这个备份恢复到临时库；
2. 从备份的时间点开始，将备份的 binlog 依次取出来，重放到中午误删表之前的那个时刻。这样你的临时库就跟误删之前的线上库一样了。
3. 把表数据从临时库取出来，按需要恢复到线上库去。