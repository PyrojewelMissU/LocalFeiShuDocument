[JavaGui如何解决幻读](https://javaguide.cn/database/mysql/transaction-isolation-level.html#%E8%A7%A3%E5%86%B3%E5%B9%BB%E8%AF%BB%E7%9A%84%E6%96%B9%E6%B3%95)

https://zhuanlan.zhihu.com/p/1990791355539154787

https://chatgpt.com/c/6964e14e-5f18-8321-b5d2-aa45da4f3b96

https://developer.aliyun.com/article/1646000

https://cloud.tencent.com/developer/article/2364816



RR（Repeatable Read，可重复读）隔离级别在大多数情况下解决了幻读，但并非完全解决。 [<span style="color: rgb(36,91,219); background-color: inherit">InnoDB 存储引擎</span>](https://zhida.zhihu.com/search?content_id=268504436\&content_type=Article\&match_order=1\&q=InnoDB+%E5%AD%98%E5%82%A8%E5%BC%95%E6%93%8E\&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NjgzOTE2NDcsInEiOiJJbm5vREIg5a2Y5YKo5byV5pOOIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MjY4NTA0NDM2LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.ArtJexIYboeV2YkTmoic3GikuoKM3_sd5IGchC50Rbg\&zhida_source=entity)通过 [<span style="color: rgb(36,91,219); background-color: inherit">MVCC</span>](https://zhida.zhihu.com/search?content_id=268504436\&content_type=Article\&match_order=1\&q=MVCC\&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NjgzOTE2NDcsInEiOiJNVkNDIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MjY4NTA0NDM2LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.8eDxY2prDcpxABXvqQAAxG-2Mbg7CL_wsreXPLLSMsk\&zhida_source=entity) + [<span style="color: rgb(36,91,219); background-color: inherit">间隙锁</span>](https://zhida.zhihu.com/search?content_id=268504436\&content_type=Article\&match_order=1\&q=%E9%97%B4%E9%9A%99%E9%94%81\&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NjgzOTE2NDcsInEiOiLpl7TpmpnplIEiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjg1MDQ0MzYsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.YfWHa8t0FC-WXMA7jsVfXQtbpszQoby7ISktm6eo5_E\&zhida_source=entity) 的组合机制，在绝大多数场景下防止了幻读，但在某些边缘情况下仍然可能出现。

MySQL RR 如何"解决"幻读

1. MVCC（多版本并发控制）

```plain&#x20;text
-- MVCC 防止快照读的幻读
BEGIN;  -- 事务开始，创建Read View
SELECT * FROM users WHERE age > 20;  -- 快照读，基于Read View
-- 此时其他事务的INSERT不会影响这个Read View
SELECT * FROM users WHERE age > 20;  -- 还是相同的结果集
COMMIT;
```

* 间隙锁（Gap Lock）

```plain&#x20;text
-- 间隙锁防止当前读的幻读
BEGIN;
-- 当前读（加锁读）
SELECT * FROM users WHERE age > 20 FOR UPDATE;
-- InnoDB会在age>20的范围内加间隙锁
-- 其他事务无法在这个范围内插入新记录
COMMIT;
```

间隙锁的问题：

* 只在某些索引条件下生效

* 如果是全表扫描，可能无法加间隙锁

* 某些索引结构可能无法完全覆盖所有可能的插入位置

- [<span style="color: rgb(36,91,219); background-color: inherit">Next-Key Lock</span>](https://zhida.zhihu.com/search?content_id=268504436\&content_type=Article\&match_order=1\&q=Next-Key+Lock\&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NjgzOTE2NDcsInEiOiJOZXh0LUtleSBMb2NrIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MjY4NTA0NDM2LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.nkv5GrJYvWFZdf3_Y5MO1eVDF1fVdUzb6l1jc7ZNpbY\&zhida_source=entity)（临键锁）

```plain&#x20;text
Next-Key Lock = 记录锁（Record Lock） + 间隙锁（Gap Lock）

示例：表中有记录 10, 20, 30
SELECT * FROM t WHERE id > 15 FOR UPDATE;
锁定的范围：
- 记录锁：锁住id=20的记录
- 间隙锁：锁住(10,20)和(20,30)的间隙
- 效果：阻止id>15的范围内插入新记录
```

RR隔离级别仍然可能出现幻读的情况

情况1：先快照读，后当前读

```plain&#x20;text
-- 会话1（事务A）
BEGIN;
-- 1. 快照读
SELECT * FROM users WHERE age > 20;  -- 返回3条记录

-- 会话2（事务B）
INSERT INTO users (age) VALUES (25);  -- 成功插入

-- 会话1（事务A）继续
-- 2. 当前读（同一事务内混合读取模式）
SELECT * FROM users WHERE age > 20 FOR UPDATE;  
-- 返回4条记录！出现了幻读
```

原因分析：

* 第一次是快照读，基于MVCC的Read View

* 第二次是当前读，读取最新数据，并加锁

* 当前读能看到快照读之后插入的数据

情况2：特殊的INSERT...SELECT

```plain&#x20;text
-- 会话1（事务A）
BEGIN;
-- 假设users表有唯一索引或主键
INSERT INTO users_backup 
SELECT * FROM users WHERE age > 20;
-- 这个SELECT是当前读，可能会读取到其他事务插入的数据

-- 如果此时会话2插入了一条age>20的记录
-- 那么INSERT...SELECT可能会报唯一键冲突
```

情况3：UPDATE语句的副作用

```plain&#x20;text
-- 会话1（事务A）
BEGIN;
SELECT * FROM users WHERE age > 20;  -- 返回3条记录

-- 会话2（事务B）
INSERT INTO users (age) VALUES (25);  -- 成功

-- 会话1（事务A）
UPDATE users SET status = 1 WHERE age > 20;
-- 这个UPDATE会看到新插入的记录，并修改它

SELECT * FROM users WHERE age > 20;  -- 返回4条记录
-- UPDATE后的SELECT看到了新记录
```

五、持久性（Durability） 定义：事务提交后，其对数据库的修改就是永久性的，即使系统发生故障也不会丢失。

额外知识点：

* MYSQL 锁：解决多个事务同时更新一行数据。

  * 锁定读：读取当前版本，读取数据加锁，阻塞其他的事物同时的修改，避免安全问题 select \* from. for update。

  * 共享锁和排他锁。

  * 表锁：LOCK TABLES XX READ表级共享锁， LOCK TABLES XX WRITE表级独占锁。

  * 行锁：命中索引后，才会加行锁。

  * 表锁：确定不了具体要插入多少条数据。

  * 意向锁：

  * 间隙锁：会锁上前面和后面间隙的ID锁，需命中索引。

  * DDL：元数据锁。

* 隐式提交语句：DDL。当在一个会话里面，一个事物还没有提交或者回滚就有开启了一个事物，会隐式的提交上一个事物。

* binlog 归档日志，记录的是偏向于逻辑性的日志，是mysql server自己的日志文件。刷盘策略 sync\_binlog 默认值0，binlog写入的时候，先写入到os cache中。1 会直接写入到磁盘文件。

