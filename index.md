# Kvorm
by Kovi Chen

## 一、 简介

1.1 轻量级ORM工具；

1.2 无需配置，直接使用；

1.3 支持动态生成SQL语句；

1.4 支持日志输出SQL语句（现仅支持log4j）；

1.5 支持事务管理；

1.6 支持全表关键字查询；

> ORM: Object Relational Mapping 对象关系映射

## 二、 注意

2.1 不支持级联，每次数据库操作仅关心一张表、一个对象;

2.2 不支持缓存（或许在未来会支持）;

2.3 输出SQL日志时，将会输出所有取值，不存在“?”占位符；

## 三、 简单例子

> 测试代码https://github.com/KoviChen/Kvorm/tree/HEAD/test

> 使用Mysql数据库，使用Spring注入数据源

### 3.1 使用的数据表（People）

列名 | 类型 | 说明
---|---|---
id | int | 主键
name | varchar | 姓名
age | int | 年龄
birthday | datetime | 生日

> 该表忽略了一些暂不重要的描述，如非空等

### 3.2 创建映射对象

```JAVA
impoty java.util.Date;

public class People{
    
    @Key
    private Integer id;
    
    @Mapping("name")
    private String n;
    
    private String age;
    
    private Date birthday;
    
    @Irrelevant
    private String test;
    
    ......
    
}
```

> 此处隐藏getter、setter方法

> Key注解用于表示该属性对应的列是候选键

> Mapping注解用于映射数据字段

> Mapping注解写在类名上则映射表名，这里不展示

> Irrelevant注解用于表示该属性与数据库无任何关系

### 3.3 配置Spring上下文环境（例子所需）

```XML
<!-- 本例子使用c3p0数据源 -->
<bean id="c3p0dateSource" class="com.mchange.v2.c3p0.ComboPooledDataSource" destroy-method="close" lazy-init="false">
    <property name="driverClass" value="${jdbc.driverClassName}" />
    <property name="jdbcUrl" value="${jdbc.url}" />
    <property name="user" value="${jdbc.username}" />
    <property name="password" value="${jdbc.password}" />
    <property name="initialPoolSize" value="${jdbc.initialPoolSize}" />
    <property name="minPoolSize" value="${jdbc.minPoolSize}" />
    <property name="maxPoolSize" value="${jdbc.maxPoolSize}" />
    <property name="acquireIncrement" value="${jdbc.acquireIncrement}" />
    <property name="maxIdleTime" value="${jdbc.maxIdleTime}" />
</bean>

<!-- 为kvorm注入数据源，设置log4j输出 -->
<bean id="sessionBean" class="com.github.kovichen.kvorm.SessionBean">
    <property name="dataSource">
        <ref bean="c3p0dateSource" />
    </property>
    <property name="log4j" value="true" />
</bean>
```

> 此处隐藏了部分xml信息

### 3.4 保存数据

```JAVA
import java.text.ParseException;
import java.text.SimpleDateFormat;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

import com.github.kovichen.kvorm.Session;
import com.github.kovichen.test.bean.People;

public class TestSave {

    public static void main(String[] args) throws ParseException {

        // 取得会话
        ApplicationContext context = new ClassPathXmlApplicationContext("config/ConfigContext.xml");
        Session session = (Session) context.getBean("sessionBean");

        // 保存数据的值
        People people = new People();
        people.setId(1);
        people.setN("Kovi");
        people.setAge("22");
        people.setBirthday(new SimpleDateFormat("yyyyMMdd").parse("19970410"));

        // 进行保存
        boolean saveResult = session.save(people);
        if (saveResult) {
            System.out.println("保存成功");
        } else {
            System.out.println("保存失败");
        }
    }
}
```

#### 日志输出

```
SQL: INSERT INTO PEOPLE (ID,name,AGE,BIRTHDAY) VALUES (1,'Kovi','22','1997-04-10 00:00:00')
```

> 属性n使用mapping注解映射为name，其他未使用mapping注解的属性其属性名均转为全大写

#### 运行结果

```
保存成功
```

#### 获取Session的其他方式
```JAVA
// dataSource实现java.sql.DataSource
Session session = new SessionBean(dataSource);
```

### 3.5 读取数据

```JAVA
import java.text.SimpleDateFormat;
import java.util.List;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

import com.github.kovichen.kvorm.Session;
import com.github.kovichen.test.bean.People;

@SuppressWarnings("resource")
public class TestQuery {

    public static void main(String[] args) {

        // 取得会话
        ApplicationContext context = new ClassPathXmlApplicationContext("config/ConfigContext.xml");
        Session session = (Session) context.getBean("sessionBean");

        // 读取数据
        List<People> peoples = session.query(People.class);
        if (peoples != null) {
            for (int i = 0; i < peoples.size(); i++) {
                People people = peoples.get(i);
                System.out.print(people.getId() + " | ");
                System.out.print(people.getN() + " | ");
                System.out.print(people.getAge() + " | ");
                System.out.println(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(people.getBirthday()));
            }
        }
    }
}
```

#### 日志输出

```
SQL: SELECT ID,name,AGE,BIRTHDAY FROM PEOPLE
```

#### 运行结果

```
1 | Kovi | 22 | 1997-04-10 00:00:00
```

> 由此可知第4步的保存操作成功

#### 例子结束

## 四、 Session方法

> 该工具核心是Session

### 4.1 Query相关

> 读取数据，并映射到对象

返回类型|方法|说明
---|---|---
<T> List<T>|query(CLass<T> clazz)|返回T映射的数据表所有数据（下同）
<T> List<T>|query(Class<T> clazz, String from)|同上，from为关联表和附加查询范围（下同）
<T> List<T>|query(Class<T> clazz, String from, boolean distinct)|同上，distinct决定是否去重（下同）
<T> List<T>|query(T t)|返回t映射的数据表中与其非空属性值相符的数据<br>（下同）
<T> List<T>|query(T t, boolean distinct)|同上
<T> List<T>|query(T t, String from)|同上
<T> List<T>|query(T t, String from, boolean distinct)|同上
<T> List<T>|query(String sql, Class<T> clazz)|执行sql进行查询，并映射到T集合
Data|query(String sql)|执行sql进行查询，返回Data对象
<T> List<T>|search(Class<T> clazz, String keyword)|对全表所有字段模糊查询keyword关键字（下同），并映射到T集合
<T> List<T>|search(Class<T> clazz, String keyword, boolean distinct)|同上
<T> List<T>|search(Class<T> clazz, String from, String keyword)|同上
<T> List<T>|search(Class<T> clazz, String from, String keyword, boolean distinct)|同上
<T> List<T>|search(T t, String keyword)|同上
<T> List<T>|search(T t, String keyword, boolean distinct)|同上
<T> List<T>|search(T t, String from, String keyword)|同上
<T> List<T>|search(T t, String from, String keyword, boolean distinct)|同上

> from参数：附加关联表和查询范围的SQL语句，显然，从关键字from开始书写；

> Data类型：在无指定映射对象时的临时映射对象；继承ArrayList<Row>，Row类型继承HashMap<String, Object>，String值列名，Object值列值；

### 4.2 Save相关

> 保存数据

返回类型|方法|说明
---|---|---
boolean|save(Object obj)|将obj中的非空属性值保存到其映射的数据表中，返回保存结果
int|save(List<Object> list)|同上所述，保存多行数据，返回保存成功的行数
boolean|save(String sql)|执行sql进行保存，返回保存结果

### 4.3 Update相关

> 更新数据

返回类型|方法|说明
---|---|---
int|update(Object obj)|更新obj映射表中满足obj中@Key标识属性值相符的数据为obj的数据，返回成功的数量
int|update(Object obj, String where)|更新obj映射表中满足where条件的数据值为obj的数据值，返回成功的数量
int|update(String sql)|执行sql进行更新，返回成功的数量

> where参数：附加查询范围的SQL语句，显然，从关键字where开始书写；

### 4.4 Remove相关

> 删除数据

返回类型|方法|说明
---|---|---
int|remove(Object obj)|删除obj映射表中满足obj的非空属性值的数据，返回删除的数量
int|remove(String sql)|执行sql进行删除，返回删除的数量

### 4.5 Transaction相关

> 事务管理

返回类型|方法|说明
---|---|---
void|beginTransaction()|开启事务
void|commit()|提交（仅在开启事务后有效），并关闭事务
void|rollback()|回滚（仅在开启事务后有效），并关闭事务

## 五、 版本日志

### 0.0.1-SNAPSHOT

1. 最初版本，不稳定
2. 支持log4j输出SQL语句
3. 支持大部分关系型数据库，如Mysql
4. 支持任意数据源（实现java.sql.DataBase）

### 0.0.2-SNAPSHOT

1. 新增全表搜索功能
2. 修复一些漏洞

### 0.0.3-SNAPSHOT

1. 修复一些漏洞

### 0.0.4-SNAPSHOT

1. 修复一些漏洞
2. 更新包名