#MyBatis里字段到枚举类型的转换/映射
[toc]

#
#MyBatis里字段到枚举类型的转换/映射


###简单介绍

我们在用`MyBatis`里，很多时间有这样一个需求：**bean里有个属性是枚举，在DB存储时我们想存的枚举的代号，从DB拿出来时想直接映射成目标枚举类型，也即代号字段与Java枚举类的相互类型转换**。

当然，你可以为每个枚举写一个`MyEnumTypeHandler`，但问题是要为每个类都写一个`TypeHandler`，过于繁琐。

有了泛型，一个通用的`TypeHandler`直接搞定。


###源码


源码详见：https://github.com/waterystone/spring-mybatis-test


EnumTypeHandler
```java

package com.adu.spring_test.mybatis.typehandler;
import java.sql.CallableStatement;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import com.adu.spring_test.mybatis.util.CodeEnumUtil;
import org.apache.ibatis.type.BaseTypeHandler;
import org.apache.ibatis.type.JdbcType;
import com.adu.spring_test.mybatis.enums.CodeBaseEnum;
/**
* mapper里字段到枚举类的映射。
* 用法一:
* 入库：#{enumDataField, typeHandler=com.adu.spring_test.mybatis.typehandler.EnumTypeHandler}
* 出库：
* <resultMap>
* <result property="enumDataField" column="enum_data_field" javaType="com.xxx.MyEnum" typeHandler="com.adu.spring_test.mybatis.typehandler.EnumTypeHandler"/>
* </resultMap>
*
* 用法二：
* 1）在mybatis-config.xml中指定handler:
* <typeHandlers>
* <typeHandler handler="com.adu.spring_test.mybatis.typehandler.EnumTypeHandler" javaType="com.xxx.MyEnum"/>
* </typeHandlers>
* 2)在MyClassMapper.xml里直接select/update/insert。
*/
public class EnumTypeHandler<E extends Enum<?> & CodeBaseEnum> extends BaseTypeHandler<CodeBaseEnum> {
private Class<E> clazz;
public EnumTypeHandler(Class<E> enumType) {
if (enumType == null)
throw new IllegalArgumentException("Type argument cannot be null");
this.clazz = enumType;
}
@(实用开发技巧)Override
public void setNonNullParameter(PreparedStatement ps, int i, CodeBaseEnum parameter, JdbcType jdbcType)
throws SQLException {
ps.setInt(i, parameter.code());
}
@Override
public E getNullableResult(ResultSet rs, String columnName) throws SQLException {
return CodeEnumUtil.codeOf(clazz, rs.getInt(columnName));
}
@Override
public E getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
return CodeEnumUtil.codeOf(clazz, rs.getInt(columnIndex));
}
@Override
public E getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
return CodeEnumUtil.codeOf(clazz, cs.getInt(columnIndex));
}
}
```

CodeBaseEnum

```java
package com.adu.spring_test.mybatis.enums;
public interface CodeBaseEnum {
int code();
}

```


```java

package com.adu.spring_test.mybatis.util;
import com.adu.spring_test.mybatis.enums.CodeBaseEnum;
public class CodeEnumUtil {
/**
* @param enumClass
* @param code
* @param <E>
* @return
*/
public static <E extends Enum<?> & CodeBaseEnum> E codeOf(Class<E> enumClass, int code) {
E[] enumConstants = enumClass.getEnumConstants();
for (E e : enumConstants) {
if (e.code() == code)
return e;
}
return null;
}
}

```



###用法

1.有枚举得类里实现`CodeBaseEnum`接口

2./**
* mapper里字段到枚举类的映射。
* 用法一:
* 入库：#{enumDataField, typeHandler=com.adu.spring_test.mybatis.typehandler.EnumTypeHandler}
* 出库：
* <resultMap>
* <result property="enumDataField" column="enum_data_field" javaType="com.xxx.MyEnum" typeHandler="com.adu.spring_test.mybatis.typehandler.EnumTypeHandler"/>
* </resultMap>
*
* 用法二：
* 1）在mybatis-config.xml中指定handler:
* <typeHandlers>
* <typeHandler handler="com.adu.spring_test.mybatis.typehandler.EnumTypeHandler" javaType="com.xxx.MyEnum"/>
* </typeHandlers>
* 2)在MyClassMapper.xml里直接select/update/insert。
*/


3.如果你是spirngBoot 得话可以在 配置文件中 `spring.mybatis.xxxtypehandler`


4.然后自己测试下吧，我反正实践过了觉得好使。






