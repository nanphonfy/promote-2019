#### 事务基本概念
>事务的4属性（ACID）：原子性（atomicity）、一致性（consistency）、隔离性（isolation）、持久性（durability）。

>原子性：不可分割的工作单位，事务中诸操作要么都做，要么都不做；  
一致性：必须使数据库从一个一致性状态变到另一个一致性状态（与原子性密切相关）；  
隔离性：执行不能被其他事务干扰；  
持久性：一旦提交，改变是永久性。

#### 基本原理
>Spring事务本质：数据库对事务的支持。
- 纯JDBC操作数据库，使用事务步骤

```java 
获取连接Connection con = DriverManager.getConnection()
开启事务con.setAutoCommit(true/false);
执行CRUD
提交事务/回滚事务con.commit()/con.rollback();
关闭连接conn.close();
```

>Spring如何在CRUD之前和之后开启事务和关闭事务？解决该问题，即可整体理解Spring事务管理实现原理。    
>>简单介绍，注解案例：  
配置文件开启注解驱动，在相关类和方法上@Transactional。  
spring启动时会解析生成相关bean，查看拥有相关注解的类和方法，并为这些类和方法生成代理，根据@Transaction的相关参数进行相关配置注入，这样就在代理中为我们把相关的事务处理掉了（开启正常提交事务，异常回滚事务）；  
真正的数据库层的事务提交和回滚是通过binlog或者redolog实现的。

#### Spring事务的传播属性
>多个事务同时存在时，spring应如何处理这些事务的行为。  
这些属性在TransactionDefinition定义

- 常量解释表  
propagation 传播
required
requires
supports
mandatory


名称 | 解释
---|---
PROPAGATION_REQUIRED | 支持当前事务，若当前无事务，则新建一个事务。最常见，Spring默认。
PROPAGATION_REQUIRES_NEW|	新建事务，若当前有事务，挂起，新建事务将和被挂起事务没关系（相互独立）。外层事务失败回滚后，不能回滚内层事务；内层事务失败抛出异常，外层事务捕获，也可不处理回滚操作。
PROPAGATION_SUPPORTS|	支持当前事务，若当前无事务，用非事务执行。
PROPAGATION_MANDATORY|	支持当前事务，若当前无事务，抛异常。
PROPAGATION_NOT_SUPPORTED|	非事务执行，若当前有事务，挂起。
PROPAGATION_NEVER|	非事务执行，若当前有事务，抛异常。
PROPAGATION_NESTED|	若有活动事务，则运行在一个嵌套事务中；若无活动事务，则按REQUIRED执行。它用一个单独事务(拥有多个可回滚保存点)。内部事务回滚不会对外部事务造成影响。只对DataSourceTransactionManager事务管理器起效。

#### 数据库隔离级别

隔离级别 | 隔离级别值 | 导致问题
---|---|---
Read-Uncommitted|0|导致脏读
Read-Committed|1|避免脏读，允许不可重复读、幻读
Repeatable-Read|2|避免脏读、不可重复读，允许幻读
Serializable|3|串行化读，事务只能一个个执行，避免脏读、不可重复读、幻读（效率慢，慎用）

- 脏读(一增删改、一读)：一事务对数据增删改（未提交），另一事务可读到未提交数据。若第一个事务回滚了，那第二个事务就读到脏数据；  
- 不可重复读（二读之间有一改）：一事务中有两次读，第一次读和第二次之间，另外一个事务修改数据，这时两次读取数据不一致；  
- 幻读（一批量改一增）：第一个事务对一定范围数据批量修改，第二个事务在这个范围增加一条数据，这时第一个事务会丢失对新增数据的修改。  

- 总结
>隔离级别越高，越能保证数据完整性、一致性，但对并发性能的影响也越大。  
大多数数据库默认隔离级别为ReadCommited，eg.SqlServer、Oracle
少数数据库默认隔离级别为：RepeatableRead，eg.：MySQL-InnoDB

#### Spring隔离级别
isolation 

常量|解释
---|---
ISOLATION_DEFAULT| PlatfromTransactionManager默认隔离级别
ISOLATION_READ_UNCOMMITTED|	最低隔离级别
ISOLATION_READ_COMMITTED|	保证另外一个事务不能读取该事务未提交的数据。
ISOLATION_REPEATABLE_READ|	防止脏读、不可重复读，会有幻读
ISOLATION_SERIALIZABLE|	最高代价但最可靠的事务隔离级别。事务被顺序执行

#### 事务详解
>查询：没有事务；  
增删改：才有事务。  
事务主要用来保证数据操作的一致性。

- 插入
>插入一条记录时，不会直接操作表。  
①插入前先在临时表（放在内存）生成一条记录；  
②try实际表，实际表一切正常，则拷贝一份数据插入；  
③删除临时表记录，返回影响行数给用户；  
④若实际表已存在记录，则返回报错信息给用户，删除临时表（清空）。

- 删除  
>①先根据条件，从原始表查询符合条件的数据行；  
②将这些数据行复制到临时表；  
③执行删除，若有错误，原始数据原封不动，清空临时表数据，返回错误码；  
④若执行成功，则真正删除原始表，返回影响行数。

- 更新
>①先根据条件，从原始表查询符合条件的数据行；  
②将这些数据行复制到临时表；  
③执行更新，检查是否有外键依赖等，若有错误，原始数据原封不动，清空临时表数据，返回错误码；  
④若执行成功，则真正更新原始表，返回影响行数。

- 查询时不会有临时表。 

>delete from会锁表，如果加了where就是锁行。  
数据库事务这么设计的目的：后悔。不仅临时表可恢复，日志也可以，多种回滚机制，备份。  
优先加载临时表数据，若有，不会访问目标表。  
临时表使用单例，与实际表对应一把锁。所有事务都在该表进行。

#### 事务嵌套
- 假设外层事务ServiceOut的MethodOut()调用内层ServiceIn的MethodIn()

- PROPAGATION_REQUIRED(spring默认)  
>若ServiceIn.MethodIn()(事务级别PROPAGATION_REQUIRED)，那么执行ServiceOut.MethodOut()时spring已起事务，这时调用ServiceIn.MethodIn()会看到自己已运行在ServiceOut.MethodOut()的事务内部，不再起新事务。若ServiceIn.MethodIn()运行时没在事务中，会为自己分配一个事务。  
这样，ServiceOut.MethodOut()或ServiceIn.MethodIn()内的任何地方异常，事务都会被回滚。

- PROPAGATION_REQUIRES_NEW
>若ServiceOut.MethodOut()事务级别为PROPAGATION_REQUIRED，ServiceIn.MethodIn()事务级别为PROPAGATION_REQUIRES_NEW。    
那么执行到ServiceIn.MethodIn()时，ServiceOut.MethodOut()所在事务会挂起，ServiceIn.MethodIn()会起新事务，等待ServiceIn.MethodIn()事务完成后，才继续执行。    
这两类事务区别：回滚程度。ServiceIn.MethodIn()为新起一个事务（存在两个不同事务）。若ServiceIn.MethodIn()已提交，那么ServiceOut.MethodOut()失败回滚，ServiceIn.MethodIn()不会回滚；若ServiceIn.MethodIn()失败回滚，若抛出异常被ServiceOut.MethodOut()捕获，ServiceOut.MethodOut()事务仍可能提交(主要看B抛出的异常是不是A会回滚的异常)。  
- PROPAGATION_SUPPORTS
>若ServiceIn.MethodIn()事务级别为PROPAGATION_SUPPORTS，那执行到ServiceIn.MethodIn()时，若发现ServiceOut.MethodOut()已开启事务，则加入当前事务，若发现ServiceOut.MethodOut()没开启事务，则自己也不开启事务。这时，内部方法的事务性完全依赖于最外层事务。  
- PROPAGATION_NESTED
>情况较复杂：ServiceIn.MethodIn()事务级别为PROPAGATION_NESTED，ServiceIn.MethodIn若rollback,那内部事务将回滚到执行前的SavePoint，而外部事务可有以下两种处理方式:
>>a、捕获异常，执行异常分支逻辑  
该方式是嵌套事务最有价值之处（分支执行效果），若ServiceIn.MethodIn失败,那执行ServiceC.MethodC(),而ServiceIn.MethodIn已回滚到执行前的SavePoint,故不产生脏数据,该特性可用在某些特殊业务中,而PROPAGATION_REQUIRED和PROPAGATION_REQUIRES_NEW都无法做到；  
b、外部事务回滚/提交代码不做任何修改,那么若内部事务rollback,回滚到执行前的SavePoint,外部事务将根据具体的配置决定commit或rollback。

>另外三种事务传播属性基本用不到，可忽略。