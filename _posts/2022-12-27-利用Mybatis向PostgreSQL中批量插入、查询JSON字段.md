---
title: 利用Mybatis向PostgreSQL中批量插入、查询JSON字段
categories: [编程, Java ]
tags: [mybatis,postgresql]
---

这里我使用的是TimescaleDB，加了一个时间戳字段，不过没差。
关于PostgreSQL中Json数据类型的操作，可以[参考官网](https://www.postgresql.org/docs/current/functions-json.html)。
### 应用场景介绍
将TCP发过来的数据包（通过消息队列发过来）解析出数据（一个数据包含有多帧，一帧中含有多条信息），并和本地规则表的格式对应起来。以`JsonLineMsg`实体类代表对应的一帧数据：

```java
package tsdb.entity;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.ToString;
import java.sql.Timestamp;

@Data
@AllArgsConstructor
@NoArgsConstructor
@ToString
public class JsonLineMsg {
    private Timestamp timeStamp; // 时间戳
	
    private String keyAndRuleData; // key value,key为根据规则表生成的唯一标识，value为TCP解析出的对应的数据。这个字段对应数据库中的Json类型字段，String类型进入数据库还需转换为Json格式。
}
```
对应psql的表结构为：
![](/assets/2022/12/27/1.png)
上面`JsonLineMsg`实体类的一个对象就代表的一帧中的所有数据项`many（key:value）`，`keyAndRuleData`字段用来存储所有数据项，在`psql`中对应一个类型为`json`（或`jsonb`）的字段。

### 数据insert
为了查询JSON中的字段，在insert的过程中有些注意事项，==如果插入时JSON格式不正确，查询JSON字段是总返回`null`==。记录一下：
为了降低数据库打开关闭的耗时，每积累20帧持久化一次。
==note==:
* foreach批量插入 、 mybatis ExecutorType.BATCH模式插入 、 for循环insert
* 其实实际意义上来说，包括在程序里面for循环还是在sql里面for循环都不算是批量操作。只有将ExecutorType设置为BATCH模式才是真正意义上的批量操作。 并且事实证明在sql循环时设置batch与否其实执行时间差别不是很大，几乎可以忽略不计。所以其实如果不是特别要求性能。可以直接在sql中使用for循环即可 。谨慎使用batch，如果需要使用batch，请在需要的函数上面设置batch，不要全局使用。因为batch也是有副作用的。 比如在Insert操作时，在事务没有提交之前，是没有办法获取到自增的id，；此外，对于update、delete无法返回更新、插入条数。这在某型情形下是不符合业务要求的。**上面的是搬运的，不过后来有看了看，还是应该用BATCH的Executor来批量导入，实际项目中foreach不可控，指不定啥时候就报错了，文章最后记录了ExecutorType为BATCH写法的关键部分）** foreach的xml拼接sql是最不推荐的方式，使用时有大段的xml和sql语句要写，很容易出错，工作效率很低。更关键点是，**虽然效率尚可，但是真正需要效率的时候你挂了，要你何用？** 批处理执行是有大数据量插入时推荐的做法，使用起来也比较方便。
* 关于批处理的方式的具体说明，可以参考推文[MyBatis 三种批量插入方式的比较，我推荐第3个！](https://mp.weixin.qq.com/s/iwAlQwo810AaSy4JD4_0TQ)或者去StackOverFlow查一下，讲解的比较全面，总之，还是用ExecutorType为BATCH写法比较靠谱。


一帧中包含多条信息，一条信息对应一个`key:value`，所以每次从规则表生成的key和TCP解析出的value都要加到一个代表一帧所有数据的JSON串中。要注意的代码如下：
```java
                 // 存储一帧的所有key:value
                StringBuilder json = new StringBuilder();
                json.append("{");
                // frmLen 帧中信息个数
                for (int j = 0; j < frmLen; j++) {
                    StatRule stat = frm.getStat(j);
                    assert stat != null;
                    // 一条stat的key和value
                    int key = stat.getKey();
                    long value = System.nanoTime();
//                    String value = ParseStat.Parse(datas, stat);
                    json.append("\"");  // key左右必须加引号，key必为String类型
                    json.append(key);
                    json.append("\"");
                    json.append(":");
//                    json.append("\"");
                    json.append(value); // value左右不是必须加引号，若是String则加
//                    json.append("\"");
                    if ((j != statLen - 1)) {
                        json.append(",");
                    }
                }
                json.append("}");
               
                JsonLineMsg jsonLineMsg = new JsonLineMsg(new Timestamp(System.currentTimeMillis()), json.toString());
```
要注意的就是这个`key`和`value`加入数据库的类型如果为text(即java字符串)就要加`引号`，所以`key`两头必须加，`value`看情况。
对应的XML中的语句：

```xml
    <insert id="batchInsertJsonLineMsg"
            useGeneratedKeys="true" >
        insert into jsonlinemsg (timestamp ,keyandruledata ) values
        <foreach item="item" collection="list" separator="," close=";">
            (#{item.timeStamp},(#{item.keyAndRuleData})::json)
        </foreach>
    </insert>

```
这个`::json`就是将非json类型转为json类型，否则JAVA中String类型会对应其他的数据库字段类型，插入会报错。

==note:==     psql 4种类型转换 [https://www.postgresql.org/docs/14/sql-syntax-lexical.html](https://www.postgresql.org/docs/14/sql-syntax-lexical.html)


1. `type 'string'`   只能用于字面常量转换、且不能用于数组中

2. `typename ( 'string' )`  可用于运行时类型转换
3. `'string'::type`   可用于数组，可用于运行时类型转换
4. `CAST ( 'string' AS type )` 可用于数组，可用于运行时类型转换

插入后用Navicat查看：
![](/assets/2022/12/27/2.png)

如果查看到类似于 `"{"1":"1_234"}"`、`{\"1\":\"1_123\"}`这样，格式就是不正确的，查询JSON中字段会返回null。

### 数据select

```xml
  <select id="selectValueData" resultType="String">
        select keyandruledata::json ->>#{key}  from jsonlinemsg where timestamp = (#{time}::timestamp)
    </select>
```
要注意的就是这个`::json`，至于 `->` 还是 `->>`可以参考开头的官网链接。

ps: timescaledb官网推荐用jsonb，但是我测试发现jsonb查询插入都比不上json，不知道为啥
ps: 发现了，原来是转换为tsdb时，索引没建立起来，重新建表又测试了一遍，确实jsonb读取快。

### BATCH 批量插入

```java
// 获取连接的方法，设置ExecutorType.BATCH以及关闭自动提交
    public static SqlSession getSessionForBatch(String xmlPath, Properties properties) throws IOException {
        return MybatisUtil.getSqlSessionFactory(xmlPath, properties).openSession(ExecutorType.BATCH,false);
    }
```

```java
    public void update(List<PropUrl> propUrlLst) throws IOException {
        // ExecutorType.BATCH
        try (SqlSession session = MybatisUtil.getSessionForBatch(myBatisConfigXmlPath)) {
            InitTsdbUrlTableMapper mapper = (InitTsdbUrlTableMapper) session.getMapper(mapperClazz);

            for (int i = 0; i < propUrlLst.size(); i++) {
                mapper.updatePropMatchRule(propUrlLst.get(i));
                // 每50次提交一次防止内存溢出
                if ((i+1) % 50 == 0) {
                    session.commit();
                    session.clearCache();
                }
            }
            session.commit();
            session.clearCache();

            log.info("update successfully ->{}", propUrlLst);
        }
    }
```

### SpringBoot中BATCH批量插入
配置类，注入batch的`SqlSessionTemplate`的bean，另外`SqlSessionTemplate`不支持手动`commit`，而是通过`@Transactional`注解配置`commit`
```java
@Configuration
public class MybatisConfig {

    @Bean
    @Primary
    public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sessionFactory){
        return new SqlSessionTemplate(sessionFactory);
    }

    @Bean
    public SqlSessionTemplate batchSqlSessionTemplate(SqlSessionFactory sessionFactory){
        return new SqlSessionTemplate(sessionFactory,ExecutorType.BATCH);
    }
}
```
```java
@Service
public class TestService {
    private final SqlSessionTemplate batchSqlSessionTemplate;
    private final SqlSessionTemplate sqlSessionTemplate;

    public TestService(@Qualifier("batchSqlSessionTemplate") SqlSessionTemplate batchSqlSessionTemplate,
                       @Qualifier("sqlSessionTemplate")SqlSessionTemplate sqlSessionTemplate) {
        this.batchSqlSessionTemplate = batchSqlSessionTemplate;
        this.sqlSessionTemplate = sqlSessionTemplate;
    }

    /**
     * ExecutorType.SIMPLE
     * 插入 names.size()=10000 耗时 约 6000ms
     * @param names
     * @throws InterruptedException
     */
    @Transactional
    public void insertTests(List<String> names) {
        TestMapper mapper = sqlSessionTemplate.getMapper(TestMapper.class);
        long start = System.currentTimeMillis();
        for (int i = 0; i < names.size(); i++) {
            System.out.println(i);
            mapper.insertTests(names.get(i));
        }
        long end = System.currentTimeMillis();
        System.out.println(end-start);
    }

    /**
     * ExecutorType.BATCH
     * 插入 names.size()=10000 耗时 约 130ms
     * @param names
     * @throws InterruptedException
     */
    @Transactional
    public void batchInsertTests(List<String> names){
        TestMapper mapper = batchSqlSessionTemplate.getMapper(TestMapper.class);
        long start = System.currentTimeMillis();
        for (int i = 0; i < names.size(); i++) {
            System.out.println(i);
            mapper.insertTests(names.get(i));
        }
        long end = System.currentTimeMillis();
        System.out.println(end-start);

    }


}

```
测试插入10000条数据：
- `ExecutorType.SIMPLE`耗时 `6000ms`左右
- `ExecutorType.BATCH`耗时 `130ms`左右 
