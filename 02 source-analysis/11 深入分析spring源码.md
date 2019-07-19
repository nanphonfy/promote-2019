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