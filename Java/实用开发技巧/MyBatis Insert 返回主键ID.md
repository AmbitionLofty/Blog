#MyBatis Insert 返回主键ID

@(实用开发技巧)


[toc]


#
#MyBatis Insert 返回主键ID

***

###需求：使用MyBatis往MySQL数据库中插入一条记录后，需要返回该条记录的自增主键值


```java

User user = new User();
user.setUserName("chenzhou");
user.setPassword("xxxx");
user.setComment("测试插入数据返回主键功能");
System.out.println("插入前主键为："+user.getUserId());
userDao.insertAndGetId(user);//插入操作
System.out.println("插入后主键为："+user.getUserId());

```


方式一：

在实体类的映射文件 "*Mapper.xml" 这样写,在mapper中指定keyProperty属性:

```sql
<insert id="insertAndGetId" useGeneratedKeys="true" keyProperty="userId" parameterType="com.chenzhou.mybatis.User">
insert into user(userName,password,comment)
values(#{userName},#{password},#{comment})
</insert>

```

useGeneratedKeys="true" 表示给主键设置自增长
keyProperty="userId" 表示将自增长后的Id赋值给实体类中的userId字段。
parameterType="com.chenzhou.mybatis.User" 这个属性指向传递的参数实体类

这里提醒下，<insert></insert> 中没有resultType属性，不要乱加。

实体类中uerId 要有getter() and setter(); 方法
第二种方式：

同样在实体类的映射文件 "*Mapper.xml" 但是要这样写:

```sql

<insert id="insertProduct" parameterType="domain.model.ProductBean" >
<selectKey resultType="java.lang.Long" order="AFTER" keyProperty="productId">
SELECT LAST_INSERT_ID()
</selectKey>
INSERT INTO t_product(productName,productDesrcible,merchantId)values(#{productName},#{productDesrcible},#{merchantId});
</insert>
```

<insert></insert> 中没有resultType属性，但是<selectKey></selectKey> 标签是有的。

order="AFTER" 表示先执行插入语句，之后再执行查询语句。

可被设置为 BEFORE 或 AFTER。

如果设置为 BEFORE,那么它会首先选择主键,设置 keyProperty 然后执行插入语句。

如果设置为 AFTER,那么先执行插入语句,然后是 selectKey 元素-这和如 Oracle 数据库相似,可以在插入语句中嵌入序列调用
keyProperty="userId" 表示将自增长后的Id赋值给实体类中的userId字段。

SELECT LAST_INSERT_ID() 表示MySQL语法中查询出刚刚插入的记录自增长Id.

实体类中uerId 要有getter() and setter(); 方法

