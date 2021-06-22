# mybatis 自定义redis做二级缓存
## 前言
*如果关注功能实现，可以直接看功能实现部分*

### 何时使用二级缓存
> 一个宗旨---不常变的稳定而常用的
- 一级是默认开启的sqlsession级别的。
- 只在单表中使用，且所有的操作都是一个namespace下
- 查询多 增删改少的情况下
- 缓存并不全是优点，缺点很明显，缓存有时不是最新的数据。
### 二级缓存参数说明
*这是一个跨Sqlsession级虽的缓存，是mapper级别的，也就是可以多个sqlsession访问同一个mapper时生效*  

| 关键字 | 解读 |
| -- | -- |
|eviction|缓存回收策略|
|flushInterval|刷新时间间隔，单位毫秒|
|size|引用数目，代表缓存最多可以存多少对象|
|readOnly|是否只读默认false 如果True所有的sql返回的是一个对象，性能高并发安全性底，如果false返回的是序列化后的副本，安全高效率底|

#### 常用的回收策略（与操作系统内存回收策略差不多）
- LRU 默认，最近最少使用
- FIFO 先进先出
- SOFT 软引用 移除基于垃圾回收器状态和软引用规则的对象
- WEAK 弱引用 更积极的移除移除基于垃圾回收器状态和弱引用规则的对象
#### sql中控制是否使用缓存及刷新缓存
``` xml
<!-- 开启二级缓存 -->
<setting name="cacheEnabled" value="true"/> 
<select id="" resultMap="" useCache="false">
<update id="" parameterType="" flushCache="false" />
```
### 缓存何时会失效(意思就是构成key的因素一样，但是没有命中value)
#### 一级失效
- 不在同一个Sqlsession中，例如未开启事务，mybatis每次查询都会关闭旧的创建新的。
- 增删改操作程序会clear缓存
- 手动执行sqlsession.clearCache()
#### 二级失效
- insert ,update,delete语句的执行
- 超时
- 被回收策略回收
- 


## 功能实现
### 添加依赖
``` xml
<!-- 集成了lettuce的连接方式也可以用jedis方式看自己建议用集成的说明稳定些 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <!-- 序列化 -->
        <dependency>
            <groupId>de.javakaffee</groupId>
            <artifactId>kryo-serializers</artifactId>
        </dependency>
        <!-- 连接池 -->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
        </dependency>
        <!-- 自定义获取Bean -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
        </dependency>
        <!-- 断言判断，正式环境中可以使用 -->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
        </dependency>
```
### 启动二级缓存
- 可以配置在调用mapper的项目中，方便以后维护
``` yml
spring: 
  redis:
    host: 192.168.16.104
    port: 6379
    database: 0
    password: 1234
    jedis:
      pool:
        max-active: 100
        max-wait: -1
        max-idle: 100
        min-idle: 5
    timeout: 3000
  data:
    redis:
      repositories:
        enabled: false
#集群的方式
  redis:
    database: 0
    cluster:
      max-redirects: 3
      nodes:
        - 192.168.15.208:7001
        - 192.168.15.208:7002
        - 192.168.15.208:7003
        - 192.168.15.208:7004
        - 192.168.15.208:7005
        - 192.168.15.208:7006

    #password: 1234
    lettuce:
      pool:
        max-active: 1000
        max-wait: -1
        max-idle: 10
        min-idle: 5
    timeout: 3000
  data:
    redis:
      repositories:
        enabled: false


# MyBatis Config properties
mybatis:
  type-aliases-package: 
  mapper-locations: classpath:mapper/*.xml
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
    cache-enabled: true
```
### 自定义相关代码
- 确保所有POJO都实现了序列化并声明了序列号
``` java
  private static final long serialVersionUID =-1L;
```
- 自定义实现缓存接口
``` java
package com.ants.sale.config;

import com.ants.sale.aop.ApplicationContextHolder;
import org.apache.commons.codec.digest.DigestUtils;
import org.apache.commons.lang3.RandomUtils;
import org.apache.ibatis.cache.Cache;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.ValueOperations;
import java.util.Set;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;


/**
 * @author lyy
 * @Deprecated
 * @date 2021/4/8
 */

public class RedisCache  implements Cache {

//    @Autowired
//    private IGlobalCache globalCache;

    private static final Logger logger = LoggerFactory.getLogger(RedisCache.class);
    private static final String COMMON_CACHE_KEY = "ANTS:" ;
    /**
     * 统一字符集
     */
    private static final String CHARSET = "utf-8";
    /**
     * key摘要算法
     */
    private static final String ALGORITHM = "SHA-256";
    private static final int MIN_EXPIRE_MINUTES = 6;
    private static final int MAX_EXPIRE_MINUTES = 24;

    private final ReadWriteLock readWriteLock = new ReentrantReadWriteLock();
    // cache instance id
    private final String id;
    private RedisTemplate redisTemplate;
    // redis过期时间
    //private static final long EXPIRE_TIME_IN_MINUTES = 30;

    public RedisCache(String id) {
        if (id == null) {
            throw new IllegalArgumentException("Cache instances require an ID");
        }
        this.id = id;
    }


    /**
     * redis key规则前缀
     */

    private String getKeys() {

        return COMMON_CACHE_KEY + this.id + ":*";

    }

    /**
     * 按照一定规则标识key
     */

    private String getKey(Object key) {

        StringBuilder accum = new StringBuilder();

        accum.append(COMMON_CACHE_KEY);

        accum.append(this.id).append(":");

        accum.append(DigestUtils.md5Hex(String.valueOf(key)));

        return accum.toString();

    }

    @Override
    public String getId() {
        return id;
    }

    /**
     * Put query result to redis
     *
     * @param key
     * @param value
     */
    @Override
    public void putObject(Object key, Object value) {
        try {
            RedisTemplate redisTemplate = getRedisTemplate();
            ValueOperations opsForValue = redisTemplate.opsForValue();
            String strKey = getKey(key);
            // 有效期为1~2小时之间随机，防止雪崩
            int expireMinutes = RandomUtils.nextInt(MIN_EXPIRE_MINUTES, MAX_EXPIRE_MINUTES);
            opsForValue.set(strKey, value, expireMinutes, TimeUnit.HOURS);
            logger.debug("Put query result to redis");
        } catch (Throwable t) {
            logger.error("Redis put failed", t);
        }
    }

    /**
     * Get cached query result from redis
     *
     * @param key
     * @return
     */
    @Override
    public Object getObject(Object key) {
        try {
            RedisTemplate redisTemplate = getRedisTemplate();
            ValueOperations opsForValue = redisTemplate.opsForValue();
            String strKey = getKey(key);
            logger.info("Get cached query result from redis {}",this.id);
            return opsForValue.get(strKey);
        } catch (Throwable t) {
            logger.error("Redis get failed, fail over to db", t);
            return null;
        }
    }

    /**
     * Remove cached query result from redis
     *
     * @param key
     * @return
     */
    @Override
    @SuppressWarnings("unchecked")
    public Object removeObject(Object key) {
        try {
            RedisTemplate redisTemplate = getRedisTemplate();
            String strKey = getKey(key);
            redisTemplate.delete(strKey);
            logger.info("Remove cached query result from redis");
        } catch (Throwable t) {
            logger.error("Redis remove failed", t);
        }
        return null;
    }

    /**
     * Clears this cache instance
     */
    @Override
    public void clear() {
        RedisTemplate redisTemplate = getRedisTemplate();
        logger.debug("clear cache, id={}", id);
        String hsKey = getKeys();
        Set<String> keys = redisTemplate.keys(hsKey);
           redisTemplate.delete(keys);
        //Map idMap = redisTemplate.opsForHash().entries(hsKey);
//        // 获取CacheNamespace所有缓存key
//        if (!idMap.isEmpty()) {
//            Set<Object> keySet = idMap.keySet();
//            Set<String> keys = new HashSet<>(keySet.size());
//            keySet.forEach(item -> keys.add(item.toString()));
//            // 清空CacheNamespace所有缓存
//            redisTemplate.delete(keys);
//            // 清空CacheNamespace
//            redisTemplate.delete(hsKey);
//        }
        logger.info("Clear  the cached  from redis which key is"+hsKey);
    }

    /**
     * This method is not used
     *
     * @return
     */
    @Override
    public int getSize() {
        return 0;
    }

    @Override
    public ReadWriteLock getReadWriteLock() {
        return readWriteLock;
    }



    private RedisTemplate getRedisTemplate() {
        if (redisTemplate == null) {
           // redisTemplate = globalCache.getRedisTemplate();
            redisTemplate = ApplicationContextHolder.getBean("redisTemplate");
        }
        return redisTemplate;
    }
}

```
- 定义ApplicationContextHolder来实现Bean的注入和获取
``` java

/**
 * @author lyy
 * @description
 * @date 2019/9/2
 */
@Component
public class ApplicationContextHolder implements ApplicationContextAware, DisposableBean {
    private static final Logger logger = LoggerFactory.getLogger(ApplicationContextHolder.class);

    private static ApplicationContext applicationContext;


    public static ApplicationContext getApplicationContext() {
        if (applicationContext == null) {
            throw new IllegalStateException(
                    "'applicationContext' property is null,ApplicationContextHolder not yet init.");
        }
        return applicationContext;
    }

    /**
     *      * 根据bean的id来查找对象
     *      * @param id
     *      * @return
     *      
     */
    public static Object getBeanById(String id) {
        checkApplicationContext();
        return applicationContext.getBean(id);
    }

    /**
     *      * 通过名称获取bean
     *      * @param name
     *      * @return
     *      
     */
    @SuppressWarnings("unchecked")
    public static <T> T getBean(String name) {
        checkApplicationContext();
        Object object = applicationContext.getBean(name);
        return (T) object;
    }

    /**
     *      * 根据bean的class来查找对象
     *      * @param c
     *      * @return
     *      
     */
    @SuppressWarnings("all")
    public static <T> T getBeanByClass(Class<T> c) {
        checkApplicationContext();
        return (T) applicationContext.getBean(c);
    }

    /**
     *      * 从静态变量ApplicationContext中取得Bean, 自动转型为所赋值对象的类型.
     * 如果有多个Bean符合Class, 取出第一个.
     *      * @param cluss
     *      * @return
     *      
     */
    public static <T> T getBean(Class<T> cluss) {
        checkApplicationContext();
        return (T) applicationContext.getBean(cluss);
    }

    /**
     *      * 名称和所需的类型获取bean
     *      * @param name
     *      * @param cluss
     *      * @return
     *      
     */
    public static <T> T getBean(String name, Class<T> cluss) {
        checkApplicationContext();
        return (T) applicationContext.getBean(name, cluss);
    }

    public static <T> Map<String, T> getBeansOfType(Class<T> type) throws BeansException {
        checkApplicationContext();
        return applicationContext.getBeansOfType(type);
    }

    /**
     * 检查ApplicationContext不为空.
     */
    private static void checkApplicationContext() {
        if (applicationContext == null) {
            throw new IllegalStateException("applicaitonContext未注入,请在applicationContext.xml中定义ApplicationContextHolderGm");
        }
    }

    @Override
    public void destroy() throws Exception {
        checkApplicationContext();
    }

    /**
     * 清除applicationContext静态变量
     */
    public static void cleanApplicationContext() {
        applicationContext = null;
    }


    @Override
    public void setApplicationContext(ApplicationContext context) throws BeansException {
        //checkApplicationContext();
        applicationContext = context;
        logger.info("holded applicationContext,显示名称:" + applicationContext.getDisplayName());
    }
}
```
#### 启用缓存
- 注解的方式,注解在mapper接口上
``` java
@CacheNamespace(implementation = RedisCache.class)
```
- xml的方式
``` xml
 <cache type="***.RedisCache">
    <property name="eviction" value="LRU"/>
    <property name="flushInterval" value="60000000"/>
    <property name="size" value="2048"/>
    <property name="readOnly" value="false"/>
  </cache>
```
### 为什么使用redis?除了redis还有其它什么？
1. redis是一个高性能nosql库，配上集群稳定高效，因为默认的二级缓存是在内存中的。这样可以把缓存独立出来，也是所有第三方map结构做二级缓存的优点
2. 可以更好的适应分布式
3. 还有ehcache,还有理论上的所有nosql库应该都可以

## 注意事项
### 缓存不生效（或者叫缓存穿透）
- 注解的sql使用注解开启，xml配置的sql要在xml中开启，混用不生效
- windows版的redis要配置一下允许外网连接，即使把redis.windows.conf中的bind 127.0.0.1注释掉了也不行
  
``` conf
# Examples:
#
# bind 192.168.1.100 10.0.0.1
bind 0.0.0.0
```
- intellij idea 生成序列号,下面选中后放光标放在类是，不是接口上按alt+enter
```
File -> Settings -> Inspections -> Serialization issues -> Serialization class without ‘serialVersionUID’
```