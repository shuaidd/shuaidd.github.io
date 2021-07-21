---
title: mybatis
date: 2021-07-07 16:09:03
tags:
---

### mybatis-3.4.x 从源码看configuration

>前提小知识
1. 数据库操作的常规步骤

```
1.加载数据库驱动
2.根据认证信息获取数据库连接
3.开启事务
4.创建statement
5.执行sql
6.处理结果集
7.提交事务
8.关闭资源
```
2. mybatis官方学习文档地址

```
http://www.mybatis.org/mybatis-3/
```

>从源码看mybatis configuration 中几个主要的配置都是什么作用

```java
/**
   * 解析mybatis配置文件，从根节点configuration开始解析
   * @param root
   */
  private void parseConfiguration(XNode root) {
    try {
      //issue #117 read properties first
      /*读取配置的属性信息 */
      propertiesElement(root.evalNode("properties"));
      /*解析setting节点*/
      Properties settings = settingsAsProperties(root.evalNode("settings"));
      loadCustomVfs(settings);
      /*实体类型别名注册*/
      typeAliasesElement(root.evalNode("typeAliases"));
      /*拦截器注册*/
      pluginElement(root.evalNode("plugins"));
      /*对象工厂*/
      objectFactoryElement(root.evalNode("objectFactory"));
      /*对象包装工厂*/
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      /*自定义反射器工厂类*/
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      settingsElement(settings);
      // read it after objectFactory and objectWrapperFactory issue #631
      environmentsElement(root.evalNode("environments"));
      /*多数据库厂商 数据库ID的生成实现类*/
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      /*注册Java类型 与 数据库字段类型的对应关系 处理器*/
      typeHandlerElement(root.evalNode("typeHandlers"));
      /*注册数据库操作的接口*/
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }
```
>typeAliases 实体别名配置 这个理解和使用都比较简单【注：大小写不敏感  别名全部会转成小写】

1.注册类的别名
```java
  /*****详见TypeAliasRegistry.java start*******/
  public void registerAlias(Class<?> type) {
    /*默认是类的简单名称*/
    String alias = type.getSimpleName();
    Alias aliasAnnotation = type.getAnnotation(Alias.class);
    if (aliasAnnotation != null) {
      /*如果有Alias注解且值不为空，则使用注解配置的别名注册*/
      alias = aliasAnnotation.value();
    } 
    registerAlias(alias, type);
  }
  
  public void registerAlias(String alias, Class<?> value) {
    if (alias == null) {
      throw new TypeException("The parameter alias cannot be null");
    }
    // issue #748
    /*转成小写*/
    String key = alias.toLowerCase(Locale.ENGLISH);
    if (TYPE_ALIASES.containsKey(key) && TYPE_ALIASES.get(key) != null && !TYPE_ALIASES.get(key).equals(value)) {
      throw new TypeException("The alias '" + alias + "' is already mapped to the value '" + TYPE_ALIASES.get(key).getName() + "'.");
    }
    TYPE_ALIASES.put(key, value);
  }
  /*****详见TypeAliasRegistry.java end*******/
```
2.别名的使用

```java
  /*****详见BaseBuilder.java start*******/
  protected Class<?> resolveAlias(String alias) {
    return typeAliasRegistry.resolveAlias(alias);
  }
   /*****详见BaseBuilder.java end*******/
  
  /*****详见TypeAliasRegistry.java start*******/
  public <T> Class<T> resolveAlias(String string) {
    try {
      if (string == null) {
        return null;
      }
      // issue #748
      String key = string.toLowerCase(Locale.ENGLISH);
      Class<T> value;
      if (TYPE_ALIASES.containsKey(key)) {
        /*如果存在别名就直接按照别名获取class*/
        value = (Class<T>) TYPE_ALIASES.get(key);
      } else {
        /*不存在别名配置则按照全路径获取class*/
        value = (Class<T>) Resources.classForName(string);
      }
      return value;
    } catch (ClassNotFoundException e) {
      throw new TypeException("Could not resolve type alias '" + string + "'.  Cause: " + e, e);
    }
  }
  /*****详见TypeAliasRegistry.java end*******/
```

```java
/*****别名注册简单示例 start*******/
package typeAlias;

import org.apache.ibatis.type.Alias;

@Alias("testAlias")
public class TestAlias {
    private String test;

    public String getTest() {
        return test;
    }

    public void setTest(String test) {
        this.test = test;
    }
}
/*****别名注册简单示例 end*******/
```

```xml
<!--在mapper xml里面可以使用别名的地方 【不限于下面这些地方可以使用别名】-->
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
    PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="test.CachedAuthorMapper">
  <!--别名方式 -->
  <parameterMap id="s" type="testAlias" >
     <parameter property="test" javaType="string" jdbcType="VARCHAR"/>
  </parameterMap>
  <resultMap id="BASE_MAP" type="testAlias">
     <result jdbcType="VARCHAR" javaType="string" property="test" column="username"/>
  </resultMap>
  <select id="testAlias" resultType="testAlias" parameterType="testAlias">
  </select>
  <!--全路径方式 -->
  <select id="selectAuthorWithInlineParams"
          parameterType="int"
          resultType="org.apache.ibatis.domain.blog.Author">
    select * from author where id = #{id}
  </select>
</mapper>
```

>plugins mybatis拦截器
1. 拦截器的注册

```xml
<!-- 配置文件方式加入拦截器 -->
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    ...
    <typeAliases>
        <package name="typeAlias"/>
    </typeAliases>
    <plugins>
       <!-- 可以直接使用别名 -->
        <plugin interceptor="testPlugin">
            <property name="testProd" value="hello  mybatis plugin"/>
        </plugin>
    </plugins>
  ...
</configuration>
```
拦截器自定义实现简单示例
```java
@Alias("testPlugin")
/*声明要拦截的类和方法【明确指定方法参数个数和类型】*/
@Intercepts({
        @Signature(type = Executor.class,method = "query",args = {MappedStatement.class,Object.class, RowBounds.class, ResultHandler.class}),
        @Signature(type = Executor.class,method = "query",args = {MappedStatement.class,Object.class, RowBounds.class, ResultHandler.class, CacheKey.class, BoundSql.class})
})
public class TestPlugin implements Interceptor {
    private final Logger logger = LoggerFactory.getLogger(TestPlugin.class);
    private String testProd;

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        logger.info(testProd);
        return invocation.getMethod().invoke(invocation.getTarget(),invocation.getArgs());
    }

    @Override
    public Object plugin(Object target) {
        /*使用mybatis为我们提供好的默认处理方式*/
        return Plugin.wrap(target,this);
    }

    @Override
    public void setProperties(Properties properties) {
       this.testProd = properties.getProperty("testProd");
    }
}
```
注册拦截器的解析入口
```java
  /************详见XMLConfigBuilder.java******************************* */
  private void pluginElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        String interceptor = child.getStringAttribute("interceptor");
        Properties properties = child.getChildrenAsProperties();
        Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).newInstance();
        interceptorInstance.setProperties(properties);
        //在InterceptorChain中注册
        configuration.addInterceptor(interceptorInstance);
      }
    }
  }
 ```
拦截器的实际注册类
 ```java
  public class InterceptorChain {

  private final List<Interceptor> interceptors = new ArrayList<Interceptor>();

  /*为拦截对象返回代理对象*/
  public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }
  
  /**
  *注册拦截器
  */
  public void addInterceptor(Interceptor interceptor) {
    interceptors.add(interceptor);
  }
  
  public List<Interceptor> getInterceptors() {
    return Collections.unmodifiableList(interceptors);
  }
}
```
2.拦截器的使用
mybatis默认会在以下四个对象上使用plugin
- ParameterHandler
- ResultSetHandler
- StatementHandler
- Executor

```java
 /***************详见Configuration.java start**************************/
 /**
 *为ParameterHandler对象生成代理对象
 */
 public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
    ParameterHandler parameterHandler = mappedStatement.getLang().createParameterHandler(mappedStatement, parameterObject, boundSql);
    parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);
    return parameterHandler;
  }
  /***************详见Configuration.java end**************************/
  
  /***************InterceptorChain.java start**************************/
  /**
  * 调用拦截器plugin方法生成代理对象 默认实现为Plugin.wrap 详见下文
  */
  public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }
  /***************InterceptorChain.java end**************************/
  
```
拦截器代理对象InvocationHandler实现，真正处理切面逻辑的地方
```java
package org.apache.ibatis.plugin;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;

import org.apache.ibatis.reflection.ExceptionUtil;

/**
 * @author Clinton Begin
 */
public class Plugin implements InvocationHandler {

  private final Object target;
  private final Interceptor interceptor;
  private final Map<Class<?>, Set<Method>> signatureMap;

  private Plugin(Object target, Interceptor interceptor, Map<Class<?>, Set<Method>> signatureMap) {
    this.target = target;
    this.interceptor = interceptor;
    this.signatureMap = signatureMap;
  }

  /**
   * 返回对象的代理对象
   * @param target
   * @param interceptor
   * @return
   */
  public static Object wrap(Object target, Interceptor interceptor) {
    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    Class<?> type = target.getClass();
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
    if (interfaces.length > 0) {
      return Proxy.newProxyInstance(
          type.getClassLoader(),
          interfaces,
          new Plugin(target, interceptor, signatureMap));
    }
    return target;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    /*如果方法是拦截器要拦截的方法，则调用拦截器的拦截方法*/
    try {
      Set<Method> methods = signatureMap.get(method.getDeclaringClass());
      if (methods != null && methods.contains(method)) {
        return interceptor.intercept(new Invocation(target, method, args));
      }
      /*不在拦截列表则不做任何处理*/
      return method.invoke(target, args);
    } catch (Exception e) {
      throw ExceptionUtil.unwrapThrowable(e);
    }
  }

  /**
   * 获取拦截器拦截的方法列表
   * @param interceptor
   * @return
   */
  private static Map<Class<?>, Set<Method>> getSignatureMap(Interceptor interceptor) {
    Intercepts interceptsAnnotation = interceptor.getClass().getAnnotation(Intercepts.class);
    // issue #251
    if (interceptsAnnotation == null) {
      throw new PluginException("No @Intercepts annotation was found in interceptor " + interceptor.getClass().getName());      
    }
    Signature[] sigs = interceptsAnnotation.value();
    Map<Class<?>, Set<Method>> signatureMap = new HashMap<Class<?>, Set<Method>>();
    for (Signature sig : sigs) {
      Set<Method> methods = signatureMap.get(sig.type());
      if (methods == null) {
        methods = new HashSet<Method>();
        signatureMap.put(sig.type(), methods);
      }
      try {
        Method method = sig.type().getMethod(sig.method(), sig.args());
        methods.add(method);
      } catch (NoSuchMethodException e) {
        throw new PluginException("Could not find method on " + sig.type() + " named " + sig.method() + ". Cause: " + e, e);
      }
    }
    return signatureMap;
  }

  private static Class<?>[] getAllInterfaces(Class<?> type, Map<Class<?>, Set<Method>> signatureMap) {
    Set<Class<?>> interfaces = new HashSet<Class<?>>();
    while (type != null) {
      for (Class<?> c : type.getInterfaces()) {
        if (signatureMap.containsKey(c)) {
          interfaces.add(c);
        }
      }
      type = type.getSuperclass();
    }
    return interfaces.toArray(new Class<?>[interfaces.size()]);
  }
}

```


>objectFactory 返回结果对象生成工厂 下面是默认实现

```java
package org.apache.ibatis.reflection.factory;

import java.io.Serializable;
import java.lang.reflect.Constructor;
import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Properties;
import java.util.Set;
import java.util.SortedSet;
import java.util.TreeSet;

import org.apache.ibatis.reflection.ReflectionException;

/**
 * @author Clinton Begin
 */
public class DefaultObjectFactory implements ObjectFactory, Serializable {

  private static final long serialVersionUID = -8855120656740914948L;

  @Override
  public <T> T create(Class<T> type) {
    return create(type, null, null);
  }

  @SuppressWarnings("unchecked")
  @Override
  public <T> T create(Class<T> type, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    Class<?> classToCreate = resolveInterface(type);
    // we know types are assignable
    return (T) instantiateClass(classToCreate, constructorArgTypes, constructorArgs);
  }

  @Override
  public void setProperties(Properties properties) {
    // no props for default
  }

  private  <T> T instantiateClass(Class<T> type, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    try {
      Constructor<T> constructor;
      if (constructorArgTypes == null || constructorArgs == null) {
        constructor = type.getDeclaredConstructor();
        if (!constructor.isAccessible()) {
          constructor.setAccessible(true);
        }
        return constructor.newInstance();
      }
      constructor = type.getDeclaredConstructor(constructorArgTypes.toArray(new Class[constructorArgTypes.size()]));
      if (!constructor.isAccessible()) {
        constructor.setAccessible(true);
      }
      return constructor.newInstance(constructorArgs.toArray(new Object[constructorArgs.size()]));
    } catch (Exception e) {
      StringBuilder argTypes = new StringBuilder();
      if (constructorArgTypes != null && !constructorArgTypes.isEmpty()) {
        for (Class<?> argType : constructorArgTypes) {
          argTypes.append(argType.getSimpleName());
          argTypes.append(",");
        }
        argTypes.deleteCharAt(argTypes.length() - 1); // remove trailing ,
      }
      StringBuilder argValues = new StringBuilder();
      if (constructorArgs != null && !constructorArgs.isEmpty()) {
        for (Object argValue : constructorArgs) {
          argValues.append(String.valueOf(argValue));
          argValues.append(",");
        }
        argValues.deleteCharAt(argValues.length() - 1); // remove trailing ,
      }
      throw new ReflectionException("Error instantiating " + type + " with invalid types (" + argTypes + ") or values (" + argValues + "). Cause: " + e, e);
    }
  }

  protected Class<?> resolveInterface(Class<?> type) {
    Class<?> classToCreate;
    if (type == List.class || type == Collection.class || type == Iterable.class) {
      classToCreate = ArrayList.class;
    } else if (type == Map.class) {
      classToCreate = HashMap.class;
    } else if (type == SortedSet.class) { // issue #510 Collections Support
      classToCreate = TreeSet.class;
    } else if (type == Set.class) {
      classToCreate = HashSet.class;
    } else {
      classToCreate = type;
    }
    return classToCreate;
  }

  @Override
  public <T> boolean isCollection(Class<T> type) {
    return Collection.class.isAssignableFrom(type);
  }

}

```

>environments 环境 顾名思义 可以配置多个隔离的环境 -> 开发/ 测试/ 预发/ 生产

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    ...
     <!-- 默认环境-->
    <environments default="dev">
        <!-- 开发-->
        <environment id="dev">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://xxx.xxx.xxx:3306/test?useUnicode=true&amp;characterEncoding=UTF-8&amp;useSSL=false"/>
                <property name="username" value="root"/>
                <property name="password" value="123456"/>
            </dataSource>
        </environment>
         <!--测试 -->
        <environment id="test">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://xxx.xxx.xxx:3306/test?useUnicode=true&amp;characterEncoding=UTF-8&amp;useSSL=false"/>
                <property name="username" value="root"/>
                <property name="password" value="123456"/>
            </dataSource>
        </environment>
         <!--生产 -->
        <environment id="prod">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://xxx.xxx.xxx:3306/test?useUnicode=true&amp;characterEncoding=UTF-8&amp;useSSL=false"/>
                <property name="username" value="root"/>
                <property name="password" value="123456"/>
            </dataSource>
        </environment>
    </environments>
   ...
</configuration>
```
简单使用示例

```java
 public static void main(String[] args) {
        InputStream inputStream;
        try {
            inputStream = Resources.getResourceAsStream("mybatis.xml");
            //开发 
            SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream,"dev");
            // 测试 
    //SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream,"test");
    //生产
       // SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream,"prod");
            SqlSession sqlSession = sqlSessionFactory.openSession();
            CachedAuthorMapper cachedAuthorMapper = sqlSession.getMapper(CachedAuthorMapper.class);
            Author author = cachedAuthorMapper.selectAllAuthors(1);

            sqlSession.close();
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
```

>databaseIdProvider 生成数据库厂商标识

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://xxx.xxx.xxx:3306/test?useUnicode=true&amp;characterEncoding=UTF-8&amp;useSSL=false"/>
                <property name="username" value="root"/>
                <property name="password" value="123456"/>
            </dataSource>
        </environment>
    </environments>
    <!--DB_VENDOR 为 VendorDatabaseIdProvider 别名 -->
    <databaseIdProvider type="DB_VENDOR">
        <!-- mysql数据库标识-->
        <property name="MySQL" value="mysql"/>
        <!-- Oracle数据库标识-->
        <property name="oracle" value="oracle"/>
    </databaseIdProvider>
    <mappers>
        <mapper resource="test/CachedAuthorMapper.xml"/>
    </mappers>
</configuration>
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
    PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="ddshuai.CachedAuthorMapper">
  <!--当数据库为MySQL 执行这一条-->
  <select id="searchNow" databaseId="mysql" resultType="date">
    select now() from dual
  </select>
<!--当数据库为Oracle 执行这一条-->
  <select id="searchNow" databaseId="oracle" resultType="date">
    select sysdate() from dual
  </select>
</mapper>
```

>typeHandler  java类型 与 数据库类型 映射处理器

简单实现一个typeHandler 下面的typeHandler负责加密数据库的自增主键 并实现可逆转换
```java
@Alias("idHandler")
public final class IdTypeHandler extends BaseTypeHandler<String> {

	@Override
	public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType)
			throws SQLException {
		ps.setLong(i, IDEncodeUtil.decode(parameter));
	}

	@Override
	public String getNullableResult(ResultSet rs, String columnName) throws SQLException {
		final long l = rs.getLong(columnName);
		return IDEncodeUtil.encode(l);
	}

	@Override
	public String getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
		final long l = rs.getLong(columnIndex);
		return IDEncodeUtil.encode(l);
	}

	@Override
	public String getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
		final long l = cs.getLong(columnIndex);
		return IDEncodeUtil.encode(l);
	}
	
}
```

```java
public abstract class IDEncodeUtil {

	public static String encode(long l) {
		if (l < 0) {
			return Long.toString(l);
		} else{
			l = mix(l);
			return Long.toString(l, 36);
		}
	}

	public static long decode(String s) {
		if(s.startsWith("-")) {
			return Long.parseLong(s);
		} else {
			return demix(Long.parseLong(s, 36));
		}
	}

	private static long mix(long l) {
		final long[] vs = doMix(l);
		return setVersion(vs);
	}

	private static long[] doMix(long l) {
		final long version = 1L;
		long ret = l;
		int digit = 0;
		while (ret > 0) {
			digit++;
			ret = ret >> 3;
		}
		int i = 0, md = (digit - 1) / 5 + 1;
		final int mix = (int) (l & ((1 << (3 * md)) - 1));
		ret = 0;
		while (digit > 0) {
			ret += (((l & ((1 << 15) - 1)) + ((mix & (((1 << 3) - 1) << (3 * --md))) << (15 - 3 * md))) << i);
			l = (l >> 15);
			digit -= 5;
			i += 18;
		}
		l = ret;

		return new long[] { version, l };
	}

	private static long demix(long l) {
		final long[] vs = getVersion(l);
		l = vs[1];
		switch ((int) vs[0]) {
		case 1:
			long dig = 0,
			ret = 0;
			while (l > 0) {
				ret += ((l & ((1 << 15) - 1)) << dig);
				l = (l >> 18);
				dig += 15;
			}
			l = ret;
			break;
		}
		return l;
	}

	private static long setVersion(long[] vs) {
		// return vs[1] / 256 * 4096 + vs[0] * 256 + vs[1] % 256;
		return ((vs[1] >> 8) << 12) + (vs[0] << 8) + (vs[1] & 255);
	}

	private static long[] getVersion(long l) {
		// return new long[] { (l / 256) % 16, (l / 4096) * 256 + l % 256 };
		return new long[] { (l >> 8) & 15, ((l >> 12) << 8) + (l & 255) };
	}
}
```
mybatis注册上面的typeHandler

```java
/******************详见Configuration.java****************************/
private void typeHandlerElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        if ("package".equals(child.getName())) {
          String typeHandlerPackage = child.getStringAttribute("name");
          typeHandlerRegistry.register(typeHandlerPackage);
        } else {
          String javaTypeName = child.getStringAttribute("javaType");
          String jdbcTypeName = child.getStringAttribute("jdbcType");
          String handlerTypeName = child.getStringAttribute("handler");
          Class<?> javaTypeClass = resolveClass(javaTypeName);
          JdbcType jdbcType = resolveJdbcType(jdbcTypeName);
          Class<?> typeHandlerClass = resolveClass(handlerTypeName);
          if (javaTypeClass != null) {
            if (jdbcType == null) {
              typeHandlerRegistry.register(javaTypeClass, typeHandlerClass);
            } else {
              typeHandlerRegistry.register(javaTypeClass, jdbcType, typeHandlerClass);
            }
          } else {
            typeHandlerRegistry.register(typeHandlerClass);
          }
        }
      }
    }
  }
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
...
    <typeAliases>
        <package name="typeAlias"/>
    </typeAliases>
    <typeHandlers>
        <typeHandler handler="idHandler" javaType="string" jdbcType="long"/>
    </typeHandlers>
 ...
</configuration>
```
结果
```java
 @Override
  public String toString() {
    return "Author : " + id + " : " + username + " : " + email;
  }
  
  数据库记录
  id    username  email            bio
  1 	ddshuai	  ddshuai@139.com	sdssd
  

DEBUG [main] - ==>  Preparing: select * from author where id = ?; 
DEBUG [main] - ==> Parameters: 1(Integer)
DEBUG [main] - <==      Total: 1
Author : b8qp : ddshuai : ddshuai@139.com
```







>mapper sql的映射接口 mybatis的接口是如何与xml的sql关联的
sql映射现在有两种方式
1. 注解方式
2. xml配置方法

注册mapper接口

```java
  /*******解析mapper xml 详见XMLMapperBuilder.java*************/
   private void configurationElement(XNode context) {
    try {
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.equals("")) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      //设定正在解析的mapper名称空间
      builderAssistant.setCurrentNamespace(namespace);
      /*解析引用的缓存*/
      cacheRefElement(context.evalNode("cache-ref"));
      /*解析自己名称空间的缓存*/
      cacheElement(context.evalNode("cache"));
      /*解析参数映射的map*/
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      /*解析结果集映射*/
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      /*解析sql模板*/
      sqlElement(context.evalNodes("/mapper/sql"));
      /*解析增删改查的sql*/
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
  }
```

---

mapper接口执行逻辑分析

1. mapper 接口MapperProxyFactory生成动态代理对象MapperProxy
2. MapperProxy 执行接口方法Method 映射的MapperMethod方法获取方法执行结果
3. MapperMethod对象调用sqlSession对象执行数据库 增删改查操作
4. sqlSession将操作代理给Executor执行
5. Executor根据接口映射的MappedStatement对象执行底层数据库操作
6. MappedStatement 获取sqlSource,并根据参数生成最终的sql语句，GenericTokenParser【${} 直接替换成参数值,#{} 替换成 ？】 解析替换sql内的参数表达式
7. MappedStatement 获取到Statement ，如果是PreparedStatement,则跟根据参数类型选择合适的typeHandler，为PreparedStatement设置查询的参数值，优先已参数上设置的typeHandler为准，不设置，则自动判断来获取
8. Statement执行sql，结果集交给ResultSetHandler处理，自动转换成需要的Pojo对象
9. 获取到结果，如果存在ResultHandler,则交给ResultHandler处理结果
10. 处理事务，关闭资源

---
mapper生成代理对象
```java
public class MapperProxyFactory<T> {

  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<Method, MapperMethod>();

  public MapperProxyFactory(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  public Class<T> getMapperInterface() {
    return mapperInterface;
  }

  public Map<Method, MapperMethod> getMethodCache() {
    return methodCache;
  }

  /*获取mapper接口的动态代理对象*/
  @SuppressWarnings("unchecked")
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }

}
```
代理对象的实际执行逻辑
```java
public class MapperProxy<T> implements InvocationHandler, Serializable {

  private static final long serialVersionUID = -6424540398559729838L;
  private final SqlSession sqlSession;
  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache;

  public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethod> methodCache) {
    this.sqlSession = sqlSession;
    this.mapperInterface = mapperInterface;
    this.methodCache = methodCache;
  }

  //mapper 接口的实际执行逻辑
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      //如果是object对象方法则直接调用
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        //如果是接口默认方法则直接调用
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    //接口声明的sql映射类方法，执行对应的MapperMethod方法
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }

  /*方法解析后缓存已解析好的MapperMethod*/
  private MapperMethod cachedMapperMethod(Method method) {
    MapperMethod mapperMethod = methodCache.get(method);
    if (mapperMethod == null) {
      mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
      methodCache.put(method, mapperMethod);
    }
    return mapperMethod;
  }

  /*调用接口的默认实现*/
  @UsesJava7
  private Object invokeDefaultMethod(Object proxy, Method method, Object[] args)
      throws Throwable {
    final Constructor<MethodHandles.Lookup> constructor = MethodHandles.Lookup.class
        .getDeclaredConstructor(Class.class, int.class);
    if (!constructor.isAccessible()) {
      constructor.setAccessible(true);
    }
    final Class<?> declaringClass = method.getDeclaringClass();
    return constructor
        .newInstance(declaringClass,
            MethodHandles.Lookup.PRIVATE | MethodHandles.Lookup.PROTECTED
                | MethodHandles.Lookup.PACKAGE | MethodHandles.Lookup.PUBLIC)
        .unreflectSpecial(method, declaringClass).bindTo(proxy).invokeWithArguments(args);
  }

  /**
   * Backport of java.lang.reflect.Method#isDefault()
   */
  /*是否默认方法*/
  private boolean isDefaultMethod(Method method) {
    return (method.getModifiers()
        & (Modifier.ABSTRACT | Modifier.PUBLIC | Modifier.STATIC)) == Modifier.PUBLIC
        && method.getDeclaringClass().isInterface();
  }
}
```
接口方法映射的MapperMethod，实际的sql执行的路由逻辑，根据SqlCommand方式路由到SqlSession中执行对应的方法
```java
public class MapperMethod {
  //sql的类型 update/delete/insert/select/flush
  private final SqlCommand command;
  //mapper方法的元信息
  private final MethodSignature method;

  public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
    this.command = new SqlCommand(config, mapperInterface, method);
    this.method = new MethodSignature(config, mapperInterface, method);
  }

  public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
      case INSERT: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          //处理带有ResultHandler参数方式的接口
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          //处理返回列表类型的接口
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          //处理返回Map集合的接口
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          //处理返回游标的接口
          result = executeForCursor(sqlSession, args);
        } else {
          //处理只有一条记录返回的接口
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName() 
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
  
  /*
  * INSERT UPDATE  DELETE 返回值只有四种 void，int,long,boolean
  * */
  private Object rowCountResult(int rowCount) {
    final Object result;
    if (method.returnsVoid()) {
      result = null;
    } else if (Integer.class.equals(method.getReturnType()) || Integer.TYPE.equals(method.getReturnType())) {
      result = rowCount;
    } else if (Long.class.equals(method.getReturnType()) || Long.TYPE.equals(method.getReturnType())) {
      result = (long)rowCount;
    } else if (Boolean.class.equals(method.getReturnType()) || Boolean.TYPE.equals(method.getReturnType())) {
      result = rowCount > 0;
    } else {
      throw new BindingException("Mapper method '" + command.getName() + "' has an unsupported return type: " + method.getReturnType());
    }
    return result;
  }
  ...
```

```java
public static class MethodSignature {
    //是否返回多条记录
    private final boolean returnsMany;
    //是否返回map
    private final boolean returnsMap;
    //是否没有返回值
    private final boolean returnsVoid;
    //是否返回游标
    private final boolean returnsCursor;
    //返回类型
    private final Class<?> returnType;
    //返回值为Map是作为key的属性
    private final String mapKey;
    //resultHandler参数的参数索引位置
    private final Integer resultHandlerIndex;
    //rowBounds参数的参数索引位置
    private final Integer rowBoundsIndex;
    //参数名称解析实现类
    private final ParamNameResolver paramNameResolver;

    public MethodSignature(Configuration configuration, Class<?> mapperInterface, Method method) {
      Type resolvedReturnType = TypeParameterResolver.resolveReturnType(method, mapperInterface);
      if (resolvedReturnType instanceof Class<?>) {
        this.returnType = (Class<?>) resolvedReturnType;
      } else if (resolvedReturnType instanceof ParameterizedType) {
        this.returnType = (Class<?>) ((ParameterizedType) resolvedReturnType).getRawType();
      } else {
        this.returnType = method.getReturnType();
      }
      this.returnsVoid = void.class.equals(this.returnType);
      this.returnsMany = configuration.getObjectFactory().isCollection(this.returnType) || this.returnType.isArray();
      this.returnsCursor = Cursor.class.equals(this.returnType);
      this.mapKey = getMapKey(method);
      this.returnsMap = this.mapKey != null;
      this.rowBoundsIndex = getUniqueParamIndex(method, RowBounds.class);
      this.resultHandlerIndex = getUniqueParamIndex(method, ResultHandler.class);
      this.paramNameResolver = new ParamNameResolver(configuration, method);
    }
    ...
```
sqlSession 根据commandName获取到对应的MappedStatement，交给executor执行

```java
public final class MappedStatement {
  //资源文件
  private String resource;
  //核心配置类
  private Configuration configuration;
  //唯一标识
  private String id;
  //sql设置的fetchSize
  private Integer fetchSize;
  private Integer timeout;
  //Statement 类型
  private StatementType statementType;
  private ResultSetType resultSetType;
  //sql的信息
  private SqlSource sqlSource;
  //对应的缓存地址
  private Cache cache;
  //配置的参数映射集合
  private ParameterMap parameterMap;
  //结果集映射
  private List<ResultMap> resultMaps;
  //是否刷新缓存
  private boolean flushCacheRequired;
  //是否使用缓存
  private boolean useCache;
  private boolean resultOrdered;
  //sql的类型
  private SqlCommandType sqlCommandType;
  //主键生成策略
  private KeyGenerator keyGenerator;
  private String[] keyProperties;
  private String[] keyColumns;
  private boolean hasNestedResultMaps;
  private String databaseId;
  private Log statementLog;
  private LanguageDriver lang;
  private String[] resultSets;
  ...
```
executor查询
```java
  /*详见BaseExecutor.java*/
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    //根据参数获取需要执行的sql
    BoundSql boundSql = ms.getBoundSql(parameter);
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
 }
```

```java
 /***详见MappedStatement.java***/
  public BoundSql getBoundSql(Object parameterObject) {
    //根据参数获取需要执行的sql,将${},#{}处理掉，处理掉条件语句，组装成最终的SQL
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings == null || parameterMappings.isEmpty()) {
      boundSql = new BoundSql(configuration, boundSql.getSql(), parameterMap.getParameterMappings(), parameterObject);
    }

    // check for nested result maps in parameter mappings (issue #30)
    for (ParameterMapping pm : boundSql.getParameterMappings()) {
      String rmId = pm.getResultMapId();
      if (rmId != null) {
        ResultMap rm = configuration.getResultMap(rmId);
        if (rm != null) {
          hasNestedResultMaps |= rm.hasNestedResultMaps();
        }
      }
    }

    return boundSql;
  }
```

```java
/*详见CachingExecutor.java*/
@Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    //获取mapper对应的缓存
    Cache cache = ms.getCache();
    if (cache != null) {
      //如果需要刷新缓存就清掉二级缓存
      flushCacheIfRequired(ms);
      //如果使用缓存，且没有resultHandler则先试着从缓存读取结果
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          //没有缓存，则执行后面的代理操作
          list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```

```java
/*详见BaseExecutor.java*/
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      //从一级缓存读取查询结果
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      // issue #601
      deferredLoads.clear();
      //如果LocalCacheScope为STATEMENT，则不缓存
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        clearLocalCache();
      }
    }
    return list;
  }
```





### mybatis-3.4.x 从源码看延迟加载
> mybatis获取结果并映射结果集代码

#### DefaultResultSetHandler.java

```java
private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, ResultLoaderMap lazyLoader, String columnPrefix) throws SQLException {
    this.useConstructorMappings = false; // reset previous mapping result
    final List<Class<?>> constructorArgTypes = new ArrayList<Class<?>>();
    final List<Object> constructorArgs = new ArrayList<Object>();
    Object resultObject = createResultObject(rsw, resultMap, constructorArgTypes, constructorArgs, columnPrefix);
    if (resultObject != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
      final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
      for (ResultMapping propertyMapping : propertyMappings) {
        // issue gcode #109 && issue #149
        if (propertyMapping.getNestedQueryId() != null && propertyMapping.isLazy()) {
          //如果是嵌套查询并且设置的是懒加载则生成代理对象
          resultObject = configuration.getProxyFactory().createProxy(resultObject, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
          break;
        }
      }
    }
    this.useConstructorMappings = resultObject != null && !constructorArgTypes.isEmpty(); // set current mapping result
    return resultObject;
  }

```

#### 代理对象执行真正查询的触发时机
```java

    @Override
    public Object invoke(Object enhanced, Method method, Method methodProxy, Object[] args) throws Throwable {
      final String methodName = method.getName();
      try {
        synchronized (lazyLoader) {
          if (WRITE_REPLACE_METHOD.equals(methodName)) {
            //处理对象序列化问题
            Object original;
            if (constructorArgTypes.isEmpty()) {
              original = objectFactory.create(type);
            } else {
              original = objectFactory.create(type, constructorArgTypes, constructorArgs);
            }
            PropertyCopier.copyBeanProperties(type, enhanced, original);
            if (lazyLoader.size() > 0) {
              return new JavassistSerialStateHolder(original, lazyLoader.getProperties(), objectFactory, constructorArgTypes, constructorArgs);
            } else {
              return original;
            }
          } else {
            if (lazyLoader.size() > 0 && !FINALIZE_METHOD.equals(methodName)) {
              if (aggressive || lazyLoadTriggerMethods.contains(methodName)) {
                //如果配置了全部获取或者调用的方法在触发加载的方法列表内这加载全部的延迟对象
                lazyLoader.loadAll();
              } else if (PropertyNamer.isSetter(methodName)) {
                //set方法直接移除
                final String property = PropertyNamer.methodToProperty(methodName);
                lazyLoader.remove(property);
              } else if (PropertyNamer.isGetter(methodName)) {
                //如果是配置了延迟加载的get方法对应的属性则加载对应的延迟加载数据
                final String property = PropertyNamer.methodToProperty(methodName);
                if (lazyLoader.hasLoader(property)) {
                  lazyLoader.load(property);
                }
              }
            }
          }
        }
        return methodProxy.invoke(enhanced, args);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    }

```

### mybatis-3.4.x 设计模式的使用
### 设计模式概览
#### 行为类

```
中介者模式
命令模式
备忘录模式
状态模式
策略模式
解释器模式
迭代器模式
观察者模式
访问者模式
模板方法模式
责任链模式
```
#### 创建类

```
单例模式
工厂模式
抽象工厂模式
建造者模式
原型模式
```

#### 结构类

```
适配器模式
桥接模式
组合模式
装饰模式
门面模式
享元模式
代理模式
```


### mybatis使用到的模式
#### 建造者模式
> mybatis中建造者模式用的还是非常之多的
```
SqlSessionFactoryBuilder 构建 SqlSessionFactory对象
XMLConfigBuilder 构建复杂的Configuration对象
MappedStatement.Builder 构建复杂的MappedStatement对象
。。。
```
#### 抽象工厂模式

```
DefaultObjectFactory生产mybatis查询后的实体对象
```

#### 装饰模式

```
1.mybatis的执行器Executor 使用的就是装饰模式来增强功能，比如CachingExecutor
2.mubatis的Cache缓存实现，也是使用装饰模式来增强cache的功能，比如BlockingCache,FifoCache,LoggingCache...
```
#### 代理模式
>这个设计模式就用的更加普遍啦
```
1.mapper接口的使用，用jdk/cglib的动态代理实现
2.懒加载模式使用动态代理，为查询出来的对象增强功能，拦截普通方法的调用，达到懒加载效果
3.plugin的实现
```
#### 过滤器链模式

```
plugin 的实现也结合了过滤器链模式，把客户端配置的n个plugin链式的作用在对象上
```
#### 模板方法模式

```
  Executor的实现 使用了模板方法模式

  /**
  * 详见BaseExecutor.java  下面都是模板方法，具体实现交给具体子类
  */
  protected abstract int doUpdate(MappedStatement ms, Object parameter)
      throws SQLException;

  protected abstract List<BatchResult> doFlushStatements(boolean isRollback)
      throws SQLException;

  protected abstract <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)
      throws SQLException;

  protected abstract <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql)
      throws SQLException;

```
#### 策略模式

```
按照mapper接口对应的操作类型，利用策略模式，路由到正确的操作逻辑上
```

### mybatis-3.4.x 从源码看缓存的使用

>从源码看mybatis缓存

1. 简单看下SqlSession的创建

```java
  //DefaultSqlSessionFactory.java
  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      //事务管理器
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      //执行器 由Executor处理缓存，见下文
      final Executor executor = configuration.newExecutor(tx, execType);
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```
通过装饰器模式，包装Executor，丰富Executor的功能
```java
  /*详见Configuration.java*/
  public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;

    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    //默认为true，包装成缓存执行器
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    //成为拦截器代理对象
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
```
CachingExecutor对查询的处理，处理二级缓存
```java
  /*详见CachingExecutor.java*/
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    //获取mapper对应的缓存
    Cache cache = ms.getCache();
    if (cache != null) {
      //如果需要刷新缓存就清掉二级缓存
      flushCacheIfRequired(ms);
      //如果使用缓存，且没有resultHandler则先试着从缓存读取结果
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          //没有缓存，则由代理继续执行后续步骤
          list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
  
  
```
基类 BaseExecutor 对查询的处理【处理一级缓存】
```java
 /*详见BaseExecutor.java**/
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      //从一级缓存读取查询结果
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      // issue #601
      deferredLoads.clear();
      //如果LocalCacheScope为STATEMENT，则不缓存
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        clearLocalCache();
      }
    }
    return list;
  }
```
缓存的的key  CacheKey
```java
/*默认实现*/
public class PerpetualCache implements Cache {

  private final String id;
  //存放缓存的数据
  private Map<Object, Object> cache = new HashMap<Object, Object>();
  ...
```
hashMap判断key是否相等

---

```
 if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
 ...
 hash值相等 并且 内存地址相等 或者 equals返回true
```
mybatis CacheKey 实现
```java
package org.apache.ibatis.cache;

import java.io.Serializable;
import java.util.ArrayList;
import java.util.List;

import org.apache.ibatis.reflection.ArrayUtil;

/**
 * @author Clinton Begin
 */
public class CacheKey implements Cloneable, Serializable {

  private static final long serialVersionUID = 1146682552656046210L;

  public static final CacheKey NULL_CACHE_KEY = new NullCacheKey();

  private static final int DEFAULT_MULTIPLYER = 37;
  private static final int DEFAULT_HASHCODE = 17;

  private final int multiplier;
  private int hashcode;
  private long checksum;
  private int count;
  // 8/21/2017 - Sonarlint flags this as needing to be marked transient.  While true if content is not serializable, this is not always true and thus should not be marked transient.
  private List<Object> updateList;

  public CacheKey() {
    this.hashcode = DEFAULT_HASHCODE;
    this.multiplier = DEFAULT_MULTIPLYER;
    this.count = 0;
    this.updateList = new ArrayList<Object>();
  }

  public CacheKey(Object[] objects) {
    this();
    updateAll(objects);
  }

  public int getUpdateCount() {
    return updateList.size();
  }

  public void update(Object object) {
    int baseHashCode = object == null ? 1 : ArrayUtil.hashCode(object); 

    count++;
    checksum += baseHashCode;
    baseHashCode *= count;

    hashcode = multiplier * hashcode + baseHashCode;

    updateList.add(object);
  }

  public void updateAll(Object[] objects) {
    for (Object o : objects) {
      update(o);
    }
  }

 /*重写equals*/
  @Override
  public boolean equals(Object object) {
    if (this == object) {
      return true;
    }
    if (!(object instanceof CacheKey)) {
      return false;
    }

    final CacheKey cacheKey = (CacheKey) object;

    if (hashcode != cacheKey.hashcode) {
      return false;
    }
    if (checksum != cacheKey.checksum) {
      return false;
    }
    if (count != cacheKey.count) {
      return false;
    }

    for (int i = 0; i < updateList.size(); i++) {
      Object thisObject = updateList.get(i);
      Object thatObject = cacheKey.updateList.get(i);
      if (!ArrayUtil.equals(thisObject, thatObject)) {
        return false;
      }
    }
    return true;
  }

  /*重写hashCode*/
  @Override
  public int hashCode() {
    return hashcode;
  }

  @Override
  public String toString() {
    StringBuilder returnValue = new StringBuilder().append(hashcode).append(':').append(checksum);
    for (Object object : updateList) {
      returnValue.append(':').append(ArrayUtil.toString(object));
    }
    return returnValue.toString();
  }

  @Override
  public CacheKey clone() throws CloneNotSupportedException {
    CacheKey clonedCacheKey = (CacheKey) super.clone();
    clonedCacheKey.updateList = new ArrayList<Object>(updateList);
    return clonedCacheKey;
  }

}

```

```java
 /**详见BaseExecutor.java*/
  @Override
  public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    CacheKey cacheKey = new CacheKey();
    //sql的编号
    cacheKey.update(ms.getId());
    //获取的数据位置
    cacheKey.update(rowBounds.getOffset());
    cacheKey.update(rowBounds.getLimit());
    //查询的sql
    cacheKey.update(boundSql.getSql());
    //查询的参数
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    // mimic DefaultParameterHandler logic
    for (ParameterMapping parameterMapping : parameterMappings) {
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;
        String propertyName = parameterMapping.getProperty();
        if (boundSql.hasAdditionalParameter(propertyName)) {
          value = boundSql.getAdditionalParameter(propertyName);
        } else if (parameterObject == null) {
          value = null;
        } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
          value = parameterObject;
        } else {
          MetaObject metaObject = configuration.newMetaObject(parameterObject);
          value = metaObject.getValue(propertyName);
        }
        cacheKey.update(value);
      }
    }
    if (configuration.getEnvironment() != null) {
      // issue #176
      //查询的环境
      cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
  }
```


2. 从上面的源码中简单看下一级缓存，二级缓存的区别
>作用域

executor 由sqlSession持有，所以localCache是在session内共享的
```java
public abstract class BaseExecutor implements Executor {

  private static final Log log = LogFactory.getLog(BaseExecutor.class);

  protected Transaction transaction;
  protected Executor wrapper;

  protected ConcurrentLinkedQueue<DeferredLoad> deferredLoads;
  //一级缓存
  protected PerpetualCache localCache;
  protected PerpetualCache localOutputParameterCache;
  protected Configuration configuration;
  ...
```
从上文中 【CachingExecutor对查询的处理，处理二级缓存】可以发现二级缓存来源于MappedStatement，这个对象只跟mapper相关，必须位于同一个命名空间或者指定一个引用的名称空间的缓存



```
所以二级缓存的作用域会比一级缓存的小，在mapper范围内
```
>启用方式

一级缓存

```java
public class Configuration {

  ...
  //一级缓存 默认作用域SESSION范围 
  protected LocalCacheScope localCacheScope = LocalCacheScope.SESSION;
  ...
```
如果设置为 localCacheScope = LocalCacheScope.STATEMENT;一级缓存就会失效，从上文的【基类 BaseExecutor 对查询的处理【处理一级缓存】】中可以看到处理的源码

---

二级缓存

```java
public class Configuration {

  ...
  //二级缓存默认开启
  protected boolean cacheEnabled = true;
  ...
```
从上文【通过装饰器模式，包装Executor，丰富Executor的功能】中看到只有cacheEnabled为true时才会使用二级缓存的包装类


---

3.简单使用示例

一级缓存

```java
/*公共测试类**/
public class BaseTest {

    protected SqlSessionFactory sqlSessionFactory;
    protected SqlSession sqlSession;

    @Before
    public void init(){
        InputStream inputStream;
        try {
            System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles","true");
            inputStream = Resources.getResourceAsStream("mybatis.xml");
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
            sqlSession = sqlSessionFactory.openSession();
        } catch (IOException e) {
            //nothing to do
        }
    }

    @After
    public void close(){
        sqlSession.close();
    }
}
```
测试使用一级缓存
###### 关闭二级缓存

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
...
    <settings>
        <setting name="cacheEnabled" value="false"/>
    </settings>
   ...
</configuration>
```

```java
public class CacheTest extends BaseTest {
    /*
    * 测试一级缓存
    * */
    @Test
    public void testCache1(){
        CachedAuthorMapper cachedAuthorMapper = sqlSession.getMapper(CachedAuthorMapper.class);
        cachedAuthorMapper.search(1,1);
        cachedAuthorMapper.search(1,1);
    }
}
```
###### 执行结果
```sql
DEBUG [main] - ==>  Preparing: select p.id as post_id,a.id,a.author_id,a.title,r.username,p.`comment` from article a,author r,post p WHERE 1 = 1 and a.author_id = r.id and p.article_id = a.id and p.article_id = ? 
DEBUG [main] - ==> Parameters: 1(Long)
DEBUG [main] - <==      Total: 2

查询两次 只执行了一次数据库操作
```
测试关闭一级缓存

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <settings>
        <setting name="localCacheScope" value="STATEMENT"/>
        <setting name="cacheEnabled" value="false"/>
    </settings>
</configuration>
```
###### 执行结果

```sql
DEBUG [main] - ==>  Preparing: select p.id as post_id,a.id,a.author_id,a.title,r.username,p.`comment` from article a,author r,post p WHERE 1 = 1 and a.author_id = r.id and p.article_id = a.id and p.article_id = ? 
DEBUG [main] - ==> Parameters: 1(Long)
DEBUG [main] - <==      Total: 2
DEBUG [main] - ==>  Preparing: select p.id as post_id,a.id,a.author_id,a.title,r.username,p.`comment` from article a,author r,post p WHERE 1 = 1 and a.author_id = r.id and p.article_id = a.id and p.article_id = ? 
DEBUG [main] - ==> Parameters: 1(Long)
DEBUG [main] - <==      Total: 2

查询了两次
```
测试二级缓存的使用
###### 关闭一级缓存

```xml

<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <settings>
        <setting name="localCacheScope" value="STATEMENT"/>
    </settings>
</configuration>

```
###### 配置mapper启用缓存
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
    PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="test.CachedAuthorMapper">
 ...
  <cache/>
  ...
 </mapper>
```
###### 执行结果

```
DEBUG [main] - ==>  Preparing: select * from post where id >0 
DEBUG [main] - ==> Parameters: 
DEBUG [main] - <==      Total: 4

DEBUG [main] - Cache Hit Ratio [ddshuai.CachedAuthorMapper]: 0.5

查询两次 只执行了一次数据库操作 缓存命中率50%
```

使用二级缓存稍有区别

```java
public class CacheTest extends BaseTest {
    
    /*
     * 测试二级缓存
     * */
    @Test
    public void testCache2(){
        CachedAuthorMapper cachedAuthorMapper = sqlSession.getMapper(CachedAuthorMapper.class);
        cachedAuthorMapper.queryPosts();
         //必须执行，否则二级缓存不会生效
        sqlSession.commit();
        cachedAuthorMapper.queryPosts();

    }
}

```

为什么需要执行commit缓存才会生效，个人理解是避免缓存脏数据

```java
package org.apache.ibatis.cache.decorators;

import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.locks.ReadWriteLock;

import org.apache.ibatis.cache.Cache;
import org.apache.ibatis.logging.Log;
import org.apache.ibatis.logging.LogFactory;

public class TransactionalCache implements Cache {

  private static final Log log = LogFactory.getLog(TransactionalCache.class);

  //真正的缓存对象
  private final Cache delegate;
  //是否提交事务的时候清空缓存
  private boolean clearOnCommit;
  //待添加到缓存的数据
  private final Map<Object, Object> entriesToAddOnCommit;
  //缓存里没有的key
  private final Set<Object> entriesMissedInCache;

  public TransactionalCache(Cache delegate) {
    this.delegate = delegate;
    this.clearOnCommit = false;
    this.entriesToAddOnCommit = new HashMap<Object, Object>();
    this.entriesMissedInCache = new HashSet<Object>();
  }

  @Override
  public String getId() {
    return delegate.getId();
  }

  @Override
  public int getSize() {
    return delegate.getSize();
  }

  @Override
  public Object getObject(Object key) {
    // issue #116
    Object object = delegate.getObject(key);
    if (object == null) {
      entriesMissedInCache.add(key);
    }
    // issue #146
    if (clearOnCommit) {
      return null;
    } else {
      return object;
    }
  }

  @Override
  public ReadWriteLock getReadWriteLock() {
    return null;
  }

  /**
   * 添加到entriesToAddOnCommit集合
   * @param key Can be any object but usually it is a {@link CacheKey}
   * @param object
   */
  @Override
  public void putObject(Object key, Object object) {
    entriesToAddOnCommit.put(key, object);
  }

  @Override
  public Object removeObject(Object key) {
    return null;
  }

  @Override
  public void clear() {
    clearOnCommit = true;
    entriesToAddOnCommit.clear();
  }

  /**
   * 提交的时候刷新之前的待缓存数据到实际缓存中
   */
  public void commit() {
    if (clearOnCommit) {
      delegate.clear();
    }
    flushPendingEntries();
    reset();
  }

  public void rollback() {
    unlockMissedEntries();
    reset();
  }

  private void reset() {
    clearOnCommit = false;
    entriesToAddOnCommit.clear();
    entriesMissedInCache.clear();
  }

  /**
   * 添加到实际缓存
   */
  private void flushPendingEntries() {
    for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
      delegate.putObject(entry.getKey(), entry.getValue());
    }
    for (Object entry : entriesMissedInCache) {
      if (!entriesToAddOnCommit.containsKey(entry)) {
        delegate.putObject(entry, null);
      }
    }
  }

  private void unlockMissedEntries() {
    for (Object entry : entriesMissedInCache) {
      try {
        delegate.removeObject(entry);
      } catch (Exception e) {
        log.warn("Unexpected exception while notifiying a rollback to the cache adapter."
            + "Consider upgrading your cache adapter to the latest version.  Cause: " + e);
      }
    }
  }

}

```
mapper配置缓存有两种方式 cache-ref,cache
> cache 上面使用了，一般都是这种方式，那么cache-ref有什么应用场景呢

很多时候我们的操作可能不是那么单一，也不是唯一一个地方能引起缓存的变化，比如有些中间表，可能就会出现在不同的mapper映射中，那么这时候如果单独放在自己的名称空间的缓存下势必会产生一些数据不一致问题【小注：一级缓存不会产生这种问题，因为任何的mapper操作数据库的更新，都会引起缓存的刷新】，那么这些个有关联性的mapper映射就可以引用同一个缓存，来达到缓存一致性，因为无论是哪个mapper的更新操作都会刷新他们共有的缓存
