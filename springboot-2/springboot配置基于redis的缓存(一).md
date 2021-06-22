# springboot配置基于redis的缓存

- springboot为什么要用缓存？

  [https://docs.spring.io/spring-boot/docs/2.4.6/reference/htmlsingle/#boot-features-caching](https://docs.spring.io/spring-boot/docs/2.4.6/reference/htmlsingle/#boot-features-caching)

- springboot如何实现缓存及Redis做缓存的特性？

  [https://docs.spring.io/spring-boot/docs/2.4.6/reference/htmlsingle/#boot-features-caching-provider](https://docs.spring.io/spring-boot/docs/2.4.6/reference/htmlsingle/#boot-features-caching-provider)

## 一引入必要的maven依赖

``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
```



## 配置RedisConfig

- 在application.yaml添加对应的redis配置信息

``` yaml
  # redis 配置
spring:  
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

```

- 添加自动化配置类

``` java
/**
 * @author lyy
 * @date 2021/6/21
 */
@Configuration
@EnableCaching
@SuppressWarnings("all")
public class RedisConfig {
    @Bean
    @ConditionalOnMissingBean(name = "redisTemplate")
    public RedisTemplate<String, Object> redisTemplate(
            RedisConnectionFactory redisConnectionFactory) {

        Jackson2JsonRedisSerializer<Object> jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer<Object>(Object.class);
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        // 解决jackson2无法反序列化LocalDateTime的问题
        om.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        om.registerModule(new JavaTimeModule());
        om.activateDefaultTyping(om.getPolymorphicTypeValidator(), ObjectMapper.DefaultTyping.NON_FINAL);

        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(om);

        RedisTemplate<String, Object> template = new RedisTemplate<String, Object>();
        template.setConnectionFactory(redisConnectionFactory);
        // template.setKeySerializer(jackson2JsonRedisSerializer);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(jackson2JsonRedisSerializer);
        template.setHashKeySerializer(jackson2JsonRedisSerializer);
        template.setHashValueSerializer(jackson2JsonRedisSerializer);
        template.afterPropertiesSet();
        return template;
    }

    @Bean
    @ConditionalOnMissingBean(StringRedisTemplate.class)
    public StringRedisTemplate stringRedisTemplate(
            RedisConnectionFactory redisConnectionFactory) {
        StringRedisTemplate template = new StringRedisTemplate();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }


   //cacheManager只针对注解缓存有效
    //只使用redisTemplate不需要配置

    @Bean
    public KeyGenerator keyGenerator() {
        return (o, method, objects) -> {
            StringBuilder sb = new StringBuilder();
            sb.append(o.getClass().getName());
            sb.append("." + method.getName() + "(");
            for (Object obj : objects) {
                sb.append(obj.toString());
            }
            sb.append(")");
            return sb.toString();
        };
    }

    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.FIELD, JsonAutoDetect.Visibility.ANY);
        om.disable(SerializationFeature.FAIL_ON_EMPTY_BEANS);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        return om;
    }


    @Bean
    public Jackson2JsonRedisSerializer jackson2JsonRedisSerializer(ObjectMapper om) {
        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
        jackson2JsonRedisSerializer.setObjectMapper(om);
        return jackson2JsonRedisSerializer;
    }
    /**
     * springboot2.x中，RedisCacheManager已经没有了单参数的构造方法
     * 1.x中通过参数redisTemplate配置的方式不可行
     */
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory, RedisSerializer jackson2JsonRedisSerializer) {
        RedisCacheConfiguration cacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(1))  // 设置缓存有效期一小时
                .disableCachingNullValues()
                .computePrefixWith(cacheName -> "ants_sale_white".concat(":").concat(cacheName).concat(":"))
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(jackson2JsonRedisSerializer));

        return RedisCacheManager.builder(redisConnectionFactory)
                .cacheDefaults(cacheConfiguration)
                .build();
    }


}

```

**注意：需要先在RedisConfig加上@EnableCaching，表示开启缓存功能**

## 启用spring 注解缓存

``` java
@Service(value = "UserServer")
@CacheConfig(cacheNames = "user")
public class UserServer implements com.ants.furun.sale_white_board.servers.api.UserServer {
    @Override
    @CachePut(key = "'ants-'+#userPo.id")
    public UserPo addUser(UserPo userPo) {
        return userPo;
    }

    @Override
    @CacheEvict(key="'ants-'+#p0")
    public int deleteUser(int id) {
        return 0;
    }

    @Override
    @Cacheable(key = "'ants-'+#p0")
    public UserPo getUser(int id) {
        return null;
    }
}

```

**其中#p0是指的第一个参数，#p1是第二个参数，以此类推。**

此时我们查看redis可以看到缓存的结果

![redis-session](https://images.cnblogs.com/cnblogs_com/ants_double/1989032/o_210622012628spring-redis-session.png)

### 相关注解说明

1. @EnableCaching

   开启缓存功能，一般放在启动类上或者自定义的RedisConfig配置类上

2. @CacheConfig

   使用@CacheConfig（cacheNames="cacheName")注解在类上，用来指定统一的value值，统一管理keys,这时可以在方法上省略value,如果在方法上写了value,那么以方法上的为准。

3. @Cacheable

   根据方法对返回的结果进行缓存，下次请求时，如果缓存存在，直接返回缓存数据，如果不存在，则执行方法，并把返回结果缓存，多用于查询方法上。

   | 属性/方法名   | 解释                             |
   | ------------- | -------------------------------- |
   | value         | 缓存名，指定了缓存放在那块空间上 |
   | cacheNames    | 与value差不多，二选一            |
   | key           | 缓存key,可以用SPEL标签自定义     |
   | keyGenerator  | key生成器                        |
   | cacheManager  | 缓存管理器                       |
   | cacheResolver | 缓存解析器                       |
   | condition     | 条件符合则缓存                   |
   | unless        | 条件符合不缓存                   |
   | sync          | 是否使用异步模式，默认false      |

   

4. @CachePut

   此注解标注的方法，每次都会执行，并将结果存入指定的缓存中。其它方法则可以直接读取缓存数据。一般用在新增方法上，属性值同@Cacheable

5. @CacheEvict

   此注解标注的方法会清空缓存，一般用于更新或删除方法上，属性与@Cacheable差不多，下面是特有的

   | 属性/方法名      | 解释                                                         |
   | ---------------- | ------------------------------------------------------------ |
   | allEntries       | 是否清空所有缓存，默认false,如果指定为true,则方法调用后立即清空所有的缓存 |
   | beforeInvocation | 是否在执行方法之前就清空缓存，默认为false,如果指定为true，刚方法执行前会清空所有缓存 |

   

6. @Caching

   可以实现在同一个方法上使用多种注解