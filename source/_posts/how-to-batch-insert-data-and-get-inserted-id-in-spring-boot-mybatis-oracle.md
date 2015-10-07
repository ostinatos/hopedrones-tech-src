title: how to batch insert data and get inserted id in spring boot + mybatis + oracle
date: 2015-10-05 10:28:00
tags: [spring boot, mybatis, oracle, performance, oracle XE]
---

Spring Boot + Mybatis + oracle下如何批量插入数据并返回id？
How to batch insert data and get inserted id in spring boot + mybatis + oracle

# 1. 需求
一个很典型的场景：批量插入数据，并获得这些数据的id
在spring boot + mybatis + oracle这样的组合中如何实现？如何实现才是性能最好的方式？

# 2. 关键点
## 获取id并赋值到pojo
* **使用mybatis的selectKey标签**
	需要单独的select语句获取sequence值
```xml
<insert id="insertPerson" parameterType="mount.olympus.prometheus.model.Person"
	keyColumn="pid" keyProperty="pid">
	<selectKey keyProperty="pid" resultType="long" order="BEFORE">
	select person_s.nextval from dual
	</selectKey>
	insert into person(pid, name) values(#{pid, jdbcType=NUMERIC}, #{name, jdbcType=VARCHAR})
</insert>
```
* **使用mybatis的useGeneratedKeys="true"**
	这种方式在insert语句中可以直接使用sequence.nextval获取，不需要单独的select语句
```xml
<insert id="insertPerson" parameterType="mount.olympus.prometheus.model.Person" keyColumn="pid" keyProperty="pid" useGeneratedKeys="true">
	insert into person(pid, name) values(person_s.nextval, #{name, jdbcType=VARCHAR})
</insert>
```
### 分析：
单独的select语句会成为额外的开销。

## 批量插入数据
* **循环调用mybatis mapper接口方法**

* **拼接一个大sql，只调用一次mapper接口方法**
这种方式对于oracle无法获取每条插入数据的id

* **mybatis的batch模式**
org.apache.ibatis.session.ExecutorType.BATCH
ExecutorType有三种模式：SIMPLE,  REUSE,  BATCH
默认是SIMPLE模式。
打开session的时候可以指定此参数，经过试验，使用了BATCH模式的优势在于每次执行插入语句时会重用相同的preparedStatement，而SIMPLE模式则每次都会新建preparedStatement。

### 分析：
由于需求是要获取每条插入数据的id，因此拼接大sql的方式并不可行。
虽然按[这篇文章] [1]的说法，拼接大sql的性能会较好，但是由于不符合本文的需求，因此不在此讨论。

## 事务
使用spring管理的事务方式，可以实现在一个事务内重用mybatis的sqlSession对象的效果，避免了占用过多数据库连接导致异常。


# 3. 测试结果：
![batch operation performance result](http://7xn9mp.com1.z0.glb.clouddn.com/batch-op-perf.PNG)

## 说明：
可见最优的方案是使用mybatis的batch模式 + 采用useGeneratedKeys="true"的方式。
batch模式由于重用了preparedStatement并进行批量提交，因此性能较好。
而不使用独立的select语句查询id则可以进一步提高性能。


# 4. 建议解决方案：
* 使用mybatis的batch模式 + 采用useGeneratedKeys="true"的方式
* 由于非批量的操作不一定需要batch模式，所以可以采用默认使用simple模式的sqlSession对象。而针对批量操作，可以独立定义一个为batch操作而设的sqlSession对象。
在spring配置文件中的配置方式如下：
```xml
<!-- sql session for batch operations. -->
<bean id="sqlSessionForBatch" class="org.mybatis.spring.SqlSessionTemplate">
	<constructor-arg index="0" ref="sqlSessionFactory" />
	<constructor-arg index="1" value="BATCH" />
</bean>
```
sqlSessionFactory定义如下：
```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
	<property name="dataSource" ref="dataSource" />
</bean>
```
* 在服务实现层使用batch模式的sqlSession对象，调用mybatis的mapper接口进行数据库操作。样例代码片段：
```java
@Named("personService")
public class PersonBatchService implements IPersonService {

	@Inject
	private SqlSession sqlSessionForBatch;

	@Override
	@Transactional
	public List<String> batchInsertPerson() {
		...

		IPersonMapper mapper = sqlSessionForBatch.getMapper(IPersonMapper.class);

		for(Person p:pl){
			mapper.insertPerson(p);			
		}

		...
	}

	...
}
```

# 5. 实现注意点：
* spring boot中配置log level
	在项目classpath中加入一个application.properties，然后写上如下内容：
```	
debug=true
logging.level.org.mybatis=DEBUG
logging.level.org.apache.ibatis=DEBUG
logging.level.mount.olympus.prometheus=DEBUG
```

这样就可以按需要查看相应的日志。

* mybatis的@Flush注解
org.apache.ibatis.annotations.Flush
Flush注解的作用是在BATCH模式下，调用注解了Flush的mapper方法可以将存储在JDBC driver中的batch statement执行。
所以一般来说循环结束时最后调用flush即可。当然也可以根据记录数批量flush。

# 6. 参考代码：
* 源码地址：https://github.com/ostinatos/prometheus-spring-mybatis
* 数据库使用的是oracle XE(express edition)，用到的用户是testuser/oracle
用到的表是：
```sql
create table TESTUSER.PERSON
(
  PID  INTEGER not null,
  NAME VARCHAR2(50)
)
```
用到的sequence：
```sql
create sequence TESTUSER.PERSON_S
minvalue 1
maxvalue 9999999999999999999999999999
start with 551420
increment by 1
cache 20;
```

* 需要额外使用oracle jdbc driver，下载一个加到项目classpath即可
* spring boot的入口程序
mount.olympus.prometheus.Application
run as java application即可。

* 接口测试地址：
http://localhost:8080/person/batch
方法：post
参数：无

# references:
* MyBatis reference documentation
https://mybatis.github.io/mybatis-3/ 

* MyBatis Spring reference documentation
http://mybatis.github.io/spring/

* spring boot documentation
http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle

[1]: http://aijuans.iteye.com/blog/1537066 "MyBatis批量大数据测试的一些结果"
* http://aijuans.iteye.com/blog/1537066
MyBatis批量大数据测试的一些结果

* How to use MyBatis to effectively perform batch database operations?
http://pretius.com/how-to-use-mybatis-effectively-perform-batch-db-operations/

* 关于Mybatis的Batch模式性能测试及结论
http://www.blogjava.net/diggbag/articles/mybatis.html