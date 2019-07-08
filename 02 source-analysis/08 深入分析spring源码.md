springjdbc封装了jdbc操作的一个框架，必须依赖spring。本身基于模板模式开发，JDBCTemplate。
JDBC二次开发，来封装一个NoSql框架。

1、加载驱动类；
2、获取连接（被封装到DataSource里）；
3、创建语句集（预处理语句集合、标准语句集）；
4、执行语句集（执行事务操作）；
5、获取结果集（若是增删改，拿到一个int值，影响行数，若是查询，拿到一个ResultSet）。

mybatis是半自动ORM框架，hibernate是全自动。springjdbc采用template设计模式，只定义了一个RowMapper接口，mapping方法，没有实现，主要用来做ORM，由自己实现。
不具体做ORM，但制定了一套规范。

手写jdbc框架，做两件事：1、单表操作实现nosql；2、把手动ORM变成自动ORM。

spring jdbc有没有sql注入。底层默认使用预处理语句集（PreparedStatement)

>1、制定QueryRules规范，定义很多查询规则常量、查询规则保存到方法；  
2、提供抽象DAO类给用户实现（基于单表，传两泛型值-类、主键）；  
3、把实体配置信息解析成EntityOperation对象，实现自动ORM；  
4、在抽象的DAO调用查询方法，把queryRule做为参数传入；  
>>做为框架，要给用户提供很方便的操作窗口。eg.spring，ClassPathXmlApplication   cpxa;  
cpxa.getBean获取所有bean。    
QueryRule解决所有查询操作。（框架设计经典之处）  

>5、将传入的QueryRule交给QueryRuleSqlBulider构建出SQL语句（被拆分的）；  
6、拼接CRUD SQL语句；  
7、交给JdbcTemplate执行；  
8、调用EntityOperation的ORM过程；  
9、返回结果（基于单表，也无需强转类型）。