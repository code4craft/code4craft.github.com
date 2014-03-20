---
layout: post
title: "使用SpringProfile和Mybatis进行多个数据源（H2和Mysql）的切换"
date: 2014-03-20 09:45
comments: true
categories: Web Spring
---
最近在做WebMagic的后台，遇到一个问题：后台用到了数据库，本来理想情况下是用Mysql，但是为了做到开箱即用，也整合了一个嵌入式数据库H2。这里面就有个问题了，如何用一套代码，提供对Mysql和H2两种方案的支持？博主收集了一些资料，也调试了很久，终于找到一套可行方案，记录下来。代码贴的有点多，主要是为了以后方便自己查找。

<!--more-->

## H2的使用

H2是一个嵌入式，纯Java实现的数据库，它各方面都要好于Java的sqlitejdbc。它可以使用内存模式，也可以使用磁盘模式。具体使用可以看攻略：

[http://www.cnblogs.com/gao241/p/3480472.html](http://www.cnblogs.com/gao241/p/3480472.html)

## 为MyBatis同时配置两套数据源

我们希望达到的效果是，不同的数据源使用不同的sql，并且这个切换最好只在配置中体现，与代码无关。所以我们选择xml的方式编写sql语句。

### MyBatis Spring的使用

同时使用Mybatis-Spring插件，这样Mybatis可以将Mapper（也就是DAO）自动配置成Bean，非常方便。它的一个完整示例可以看这个项目：[https://github.com/mybatis/jpetstore-6](https://github.com/mybatis/jpetstore-6)。这里我配置如下：

#### 配置Bean

```xml
 <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="us.codecraft.webmagic.dao" />
    </bean>

    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource" />
        <property name="mapperLocations" value="classpath*:/config/mapper/**/*.xml" />
    </bean>
```

#### 配置Mapper

对应的DAO和配置文件如下：

* us.codecraft.webmagic.dao.DynamicClassDao:

```java
public interface DynamicClassDao {

    public int add(DynamicClass dynamicClass);
}
```

* DynamicClassDao.xml

```xml
<mapper namespace="us.codecraft.webmagic.dao.DynamicClassDao">

    <insert id="add" parameterType="us.codecraft.webmagic.model.DynamicClass">
      insert into DynamicClass (`ClassName`,`SourceCode`,`AddTime`,`UpdateTime`)
      values (#{className},#{sourceCode},now(),now())
    </insert>
</mapper>
```

### 使用databaseIdProvider进行多个数据源的SQL切换

MyBatis支持根据不同的数据库名来进行SQL语句的切换。做法是初始化`SqlSessionFactoryBean`的时候，配置一个`databaseIdProvider`。

```xml
    <bean id="vendorProperties" class="org.springframework.beans.factory.config.PropertiesFactoryBean">
        <property name="properties">
            <props>
                <prop key="SQL Server">sqlserver</prop>
                <prop key="DB2">db2</prop>
                <prop key="Oracle">oracle</prop>
                <prop key="MySQL">mysql</prop>
                <prop key="H2">h2</prop>
            </props>
        </property>
    </bean>

    <bean id="databaseIdProvider" class="org.apache.ibatis.mapping.VendorDatabaseIdProvider">
        <property name="properties" ref="vendorProperties"/>
    </bean>

    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource" />
        <property name="databaseIdProvider" ref="databaseIdProvider" />
        <property name="mapperLocations" value="classpath*:/config/mapper/**/*.xml" />
    </bean>
```

然后在Mapper的xml里，把相应的语句加上`databaseId="xxx"`就可以了。

```xml
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="us.codecraft.webmagic.dao.DynamicClassDao">

    <insert id="add" parameterType="us.codecraft.webmagic.model.DynamicClass" databaseId="mysql">
      insert into DynamicClass (`ClassName`,`SourceCode`,`AddTime`,`UpdateTime`)
      values (#{className},#{sourceCode},now(),now())
    </insert>

    <insert id="add" parameterType="us.codecraft.webmagic.model.DynamicClass" databaseId="h2">
      insert into DynamicClass (`ClassName`,`SourceCode`,`AddTime`,`UpdateTime`)
      values (#{className},#{sourceCode},now(),now())
    </insert>

</mapper>
```

## Spring Profile

Profile是Spring 3.1后新增的特性，简单来说，就是根据不同的环境，读取不同的配置。这些配置可以放在一起，但是单独生效。贴个代码吧，很容易说明问题了：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:jdbc="http://www.springframework.org/schema/jdbc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
      http://www.springframework.org/schema/beans/spring-beans-4.0.xsd http://www.springframework.org/schema/jdbc http://www.springframework.org/schema/jdbc/spring-jdbc.xsd">

    <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource"
          destroy-method="close">
        <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://127.0.0.1:3306/WebMagic?characterEncoding=UTF-8"/>
        <property name="username" value="webmagic"/>
        <property name="password" value="webmagic"/>
    </bean>

    <beans profile="test">
        <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource"
              destroy-method="close">
            <property name="driverClassName" value="org.h2.Driver"/>
            <property name="url" value="jdbc:h2:mem:WebMagic;DB_CLOSE_DELAY=-1"/>
        </bean>
        <!--Refer to https://github.com/springside/springside4/wiki/H2-Database -->
        <jdbc:initialize-database data-source="dataSource" ignore-failures="ALL">
            <jdbc:script location="classpath:sql/h2/schema.sql" />
            <!--<jdbc:script location="classpath:data/h2/import-data.sql" encoding="UTF-8"/>-->
        </jdbc:initialize-database>
    </beans>

    <beans profile="standalone">
        <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource"
              destroy-method="close">
            <property name="driverClassName" value="org.h2.Driver"/>
            <property name="url" value="jdbc:h2:file:~/.h2/WebMagic;AUTO_SERVER=TRUE"/>
        </bean>
        <!--Refer to https://github.com/springside/springside4/wiki/H2-Database -->
        <jdbc:initialize-database data-source="dataSource" ignore-failures="ALL">
            <jdbc:script location="classpath:sql/h2/schema.sql" />
            <!--<jdbc:script location="classpath:data/h2/import-data.sql" encoding="UTF-8"/>-->
        </jdbc:initialize-database>
    </beans>

</beans>
```

设置Profile有不同的方式。

在JUnit里面，使用注解@ActiveProfile即可。

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"classpath*:/config/spring/applicationContext*.xml"})
@ActiveProfiles("test")
@Transactional
public abstract class AbstractTest {
}
```

Web项目则是在web.xml里设置：

```xml
<init-param>
  <param-name>spring.profiles.active</param-name>
  <param-value>product</param-value>
</init-param>
```

如此配置，就可以达到在不同的环境使用不同bean的目的！

## 参考资料

* H2数据库攻略 [http://www.cnblogs.com/gao241/p/3480472.html](http://www.cnblogs.com/gao241/p/3480472.html)
* spring+mybatis 多数据源整合 [http://blog.csdn.net/fhx007/article/details/12530735](http://blog.csdn.net/fhx007/article/details/12530735)
* 如何用Spring 3.1的Environment和Profile简化工作 [http://www.importnew.com/1099.html](http://www.importnew.com/1099.html)
