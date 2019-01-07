# Transaction
---
## 一 隔离级别
### 1.1 ACID
- Atomicity 

    整个事务中的所有操作，要么全部完成，要么全部不完成
- Consistency
    
    一个事务可以封装状态改变（除非它是一个只读的）。事务必须始终保持系统处于一致的状态，不管在任何给定的时间并发事务有多少。   
- Isolation

    事务是并发控制机制，他们交错使用时也能提供一致性。隔离让我们隐藏来自外部世界未提交的状态变化，一个失败的事务不应该破坏系统的状态。隔离是通过用悲观或乐观锁机制实现的
- Durability 

    在事务完成以后，该事务对数据库所作的更改便持久的保存在数据库之中，并不会被回滚
### 1.2 数据库问题
- 脏读

    事务A修改了一个数据，但未提交，事务B读到了事务A未提交的更新结果，如果事务A提交失败，事务B读到的就是脏数据。
- 不可重复
    
    在同一个事务中，对于同一份数据读取到的结果不一致。比如，事务B在事务A提交前读到的结果，和提交后读到的结果可能不同。通过MVCC可以在无锁的情况下，避免不可重复读
- 幻读

    在同一个事务中，同一个查询多次返回的结果不一致。事务A新增了一条记录，事务B在事务A提交前后各执行了一次查询操作，发现后一次比前一次多了一条记录。幻读是由于并发事务增加记录导致的，这个不能像不可重复读通过记录加锁解决，因为对于新增的记录根本无法加锁。需要将事务串行化，才能避免幻读。

### 1.3 四种隔离级别
- Read Uncommitted 

    一个事务可以读到另一个事务未提交的结果，所有并发事务问题都会发生
- Read Committed

    只有在事务提交后，其更新结果才会被其他事务看见，可以解决脏读
- Repeated Read

    在一个事务中，对于同一份数据的读取结果总是相同的，无论是否有其他事务对这份数据进行操作，以及这个事务是否提交。可以解决脏读、不可重复读
- Serialization

    事务串行化执行，隔离级别最高，牺牲并发性。可以解决并发事务的所有问题

## 二 事物传播机制
- PROPAGATION_REQUIRED：如果当前没有事务，就创建一个新事务，如果当前存在事务，就加入该事务，该设置是最常用的设置
- PROPAGATION_SUPPORTS：支持当事务，如果当前存在事务，就加入该事务，如果当前不存在事务，就以非事务执行
- PROPAGATION_MANDATORY：支持当前事务，如果当前存在事务，就加入该事务，如果当前不存在事务，就抛出异常
- PROPAGATION_REQUIRES_NEW：创建新事务，无论当前存不存在事务，都创建新事务
- PROPAGATION_NOT_SUPPORTED：以非事务方式执行操作，如果当前存在事务，就把当前事务挂起
- PROPAGATION_NEVER：以非事务方式执行，如果当前存在事务，则抛出异常
- PROPAGATION_NESTED：如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与PROPAGATION_REQUIRED类似的操作

## 三 @Transaction 
### 3.1 默认设置
- The propagation setting is PROPAGATION_REQUIRED.
- The isolation level is ISOLATION_DEFAULT.
- The transaction is read-write.
- The transaction timeout defaults to the default timeout of the underlying transaction system, or to none if timeouts are not supported.
- Any RuntimeException triggers rollback, and any checked Exception does not.
### 3.2 其他设置
点击[Spring Transactional Settings](https://docs.spring.io/spring/docs/5.1.3.RELEASE/spring-framework-reference/data-access.html#tx-propagation)