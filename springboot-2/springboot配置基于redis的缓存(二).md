# springboot配置基于redis的缓存(二)

[上文（一)](https://www.cnblogs.com/ants_double/p/14917037.html#autoid-1-2-0)描写了基于注解的缓存，其实底层逻辑就是redis的get 、set操作，将操作结果返回redis中。因此我们可以使用RedisUtils来进行自己设置，比如序列化的设置，key的设置都更加灵活一些。

## 在RedisConfig中添加redisTemplate的bean和 RedisUtils的初始化构造

``` java
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
  @Bean
    IGlobalCache cache(RedisTemplate redisTemplate) {
        return new RedisUtils(redisTemplate);
    }
```



- 接口IGlobalCache

  ``` java
  
  public interface IGlobalCache {
  
  
      boolean flushdb();
      boolean deleteStratKey(String startKey);
  
      /**
       * 指定缓存失效时间
       *
       * @param key  键
       * @param time 时间(秒)
       * @return
       */
      boolean expire(String key, long time);
  
      /**
       * @param key 键 不能为null
       * @return 时间(秒) 返回0代表为永久有效
       */
      long getExpire(String key);
  
      /**
       * 判断key是否存在
       *
       * @param key 键
       * @return true 存在 false不存在
       */
      boolean hasKey(String key);
  
      /**
       * 删除缓存
       *
       * @param key 可以传一个值 或多个
       */
      void del(String... key);
  // ============================String=============================
  
      /**
       * 普通缓存获取
       *
       * @param key 键
       * @return 值
       */
      Object get(String key);
  
      /**
       * 普通缓存放入
       *
       * @param key   键
       * @param value 值
       * @return true成功 false失败
       */
      boolean set(String key, Object value);
  
      /**
       * 普通缓存放入并设置时间
       *
       * @param key   键
       * @param value 值
       * @param time  时间(秒) time要大于0 如果time小于等于0 将设置无限期
       * @return true成功 false 失败
       */
      boolean set(String key, Object value, long time);
  
      /**
       * 递增
       *
       * @param key   键
       * @param delta 要增加几(大于0)
       * @return
       */
      long incr(String key, long delta);
  
      /**
       * 递减
       *
       * @param key   键
       * @param delta 要减少几(小于0)
       * @return
       */
      long decr(String key, long delta);
  
      /**
       * HashGet
       *
       * @param key  键 不能为null
       * @param item 项 不能为null
       * @return 值
       */
      Object hget(String key, String item);
  
      /**
       * 获取hashKey对应的所有键值
       *
       * @param key 键
       * @return 对应的多个键值
       */
      Map<Object, Object> hmget(String key);
  
      /**
       * HashSet
       *
       * @param key 键
       * @param map 对应多个键值
       * @return true 成功 false 失败
       */
      boolean hmset(String key, Map<String, Object> map);
  
      /**
       * HashSet 并设置时间
       *
       * @param key  键
       * @param map  对应多个键值
       * @param time 时间(秒)
       * @return true成功 false失败
       */
      boolean hmset(String key, Map<String, Object> map, long time);
  
      /**
       * 向一张hash表中放入数据,如果不存在将创建
       *
       * @param key   键
       * @param item  项
       * @param value 值
       * @return true 成功 false失败
       */
      boolean hset(String key, String item, Object value);
  
      /**
       * 向一张hash表中放入数据,如果不存在将创建
       *
       * @param key   键
       * @param item  项
       * @param value 值
       * @param time  时间(秒) 注意:如果已存在的hash表有时间,这里将会替换原有的时间
       * @return true 成功 false失败
       */
      boolean hset(String key, String item, Object value, long time);
  
      /**
       * 删除hash表中的值
       *
       * @param key  键 不能为null
       * @param item 项 可以使多个 不能为null
       */
      void hdel(String key, Object... item);
  
      /**
       * 判断hash表中是否有该项的值
       *
       * @param key  键 不能为null
       * @param item 项 不能为null
       * @return true 存在 false不存在
       */
      boolean hHasKey(String key, String item);
  
      /**
       * hash递增 如果不存在,就会创建一个 并把新增后的值返回
       *
       * @param key  键
       * @param item 项
       * @param by   要增加几(大于0)
       * @return
       */
      double hincr(String key, String item, double by);
  
      /**
       * hash递减
       *
       * @param key  键
       * @param item 项
       * @param by   要减少记(小于0)
       * @return
       */
      double hdecr(String key, String item, double by);
  
      /**
       * 根据key获取Set中的所有值
       *
       * @param key 键
       * @return
       */
      Set<Object> sGet(String key);
  
      /**
       * 根据value从一个set中查询,是否存在
       *
       * @param key   键
       * @param value 值
       * @return true 存在 false不存在
       */
      boolean sHasKey(String key, Object value);
  
      /**
       * 将数据放入set缓存
       *
       * @param key    键
       * @param values 值 可以是多个
       * @return 成功个数
       */
      long sSet(String key, Object... values);
  
      /**
       * 将set数据放入缓存
       *
       * @param key    键
       * @param time   时间(秒)
       * @param values 值 可以是多个
       * @return 成功个数
       */
      long sSetAndTime(String key, long time, Object... values);
  
  
      /**
       * 获取set缓存的长度
       *
       * @param key 键
       * @return
       */
      long sGetSetSize(String key);
  
      /**
       * 移除值为value的
       *
       * @param key    键
       * @param values 值 可以是多个
       * @return 移除的个数
       */
      long setRemove(String key, Object... values);
  
      /**
       * 获取list缓存的内容
       *
       * @param key   键
       * @param start 开始
       * @param end   结束 0 到 -1代表所有值
       * @return
       */
      List<Object> lGet(String key, long start, long end);
  
      /**
       * 获取list缓存的长度
       *
       * @param key 键
       * @return
       */
      long lGetListSize(String key);
  
      /**
       * 通过索引 获取list中的值
       *
       * @param key   键
       * @param index 索引 index>=0时， 0 表头，1 第二个元素，依次类推；index<0时，-1，表尾，-2倒数第二个元素，依次类推
       * @return
       */
      Object lGetIndex(String key, long index);
  
      /**
       * 将list放入缓存
       *
       * @param key   键
       * @param value 值
       * @return
       */
      boolean lSet(String key, Object value);
  
      /**
       * 将list放入缓存
       *
       * @param key   键
       * @param value 值
       * @return
       */
      boolean lSet(String key, Object value, long time);
  
      /**
       * 将list放入缓存
       *
       * @param key   键
       * @param value 值
       * @param time  时间(秒)
       * @return
       */
      boolean lSetAll(String key, List<Object> value);
  
  
      /**
       * 将list放入缓存
       *
       * @param key   键
       * @param value 值
       * @param time  时间(秒)
       * @return
       */
      boolean lSetAll(String key, List<Object> value, long time);
  
      /**
       * 将list放入缓存
       *
       * @param key   键
       * @param value 值
       * @return
       */
  
      boolean rSet(String key, Object value);
  
      /**
       * 将list放入缓存
       *
       * @param key   键
       * @param value 值
       * @param time  时间(秒)
       * @return
       */
  
      boolean rSet(String key, Object value, long time);
  
      /**
       * 将list放入缓存
       *
       * @param key   键
       * @param value 值
       * @return
       */
      boolean rSetAll(String key, List<Object> value);
  
      /**
       * 将list放入缓存
       *
       * @param key   键
       * @param value 值
       * @param time  时间(秒)
       * @return
       */
      boolean rSetAll(String key, List<Object> value, long time);
  
      /**
       * 根据索引修改list中的某条数据
       *
       * @param key   键
       * @param index 索引
       * @param value 值
       * @return
       */
      boolean lUpdateIndex(String key, long index, Object value);
  
      /**
       * 移除N个值为value
       *
       * @param key   键
       * @param count 移除多少个
       * @param value 值
       * @return 移除的个数
       */
      long lRemove(String key, long count, Object value);
  
      /**
       * 从redis集合中移除[start,end]之间的元素
       *
       * @param key
       * @param stard
       * @param end
       * @return
       */
      void rangeRemove(String key, Long stard, Long end);
  
      /**
       * 返回当前redisTemplate
       *
       * @return
       */
      RedisTemplate getRedisTemplate();
  
  }
  
  ```

  

- 实现接口

``` java
@Getter
@AllArgsConstructor
public final class RedisUtils implements IGlobalCache {

    private RedisTemplate<String, Object> redisTemplate;

    @Override
    public boolean flushdb() {
        try {
            redisTemplate.execute((RedisCallback) connection -> {
                connection.flushDb();
                return null;
            });
//            Set<String> keys = redisTemplate.keys("*");
//            redisTemplate.delete(keys);
            return true;
        }
        catch (Exception ex){
            throw  ex;
        }

    }

    @Override
    public boolean deleteStratKey(String startKey) {
        try {
            Set<String> keys = redisTemplate.keys(startKey);
            redisTemplate.delete(keys);
            return true;
        }
        catch (Exception ex){
            throw  ex;
        }
    }

    @Override
    public boolean expire(String key, long time) {
        try {
            if (time > 0) {
                redisTemplate.expire(key, time, TimeUnit.SECONDS);
            }
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    @Override
    public long getExpire(String key) {
        return redisTemplate.getExpire(key, TimeUnit.SECONDS);
    }

    @Override
    public boolean hasKey(String key) {
        try {
            return redisTemplate.hasKey(key);
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    @Override
    @SuppressWarnings("unchecked")
    public void del(String... key) {
        if (key != null && key.length > 0) {
            if (key.length == 1) {
                redisTemplate.delete(key[0]);
            } else {
                redisTemplate.delete((Collection<String>) CollectionUtils.arrayToList(key));
            }
        }
    }

    @Override
    public Object get(String key) {
        return key == null ? null : redisTemplate.opsForValue().get(key);
    }

    @Override
    public boolean set(String key, Object value) {
        try {
            redisTemplate.opsForValue().set(key, value);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    @Override
    public boolean set(String key, Object value, long time) {
        try {
            if (time > 0) {
                redisTemplate.opsForValue().set(key, value, time, TimeUnit.SECONDS);
            } else {
                set(key, value);
            }
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    @Override
    public long incr(String key, long delta) {
        if (delta < 0) {
            throw new RuntimeException("递增因子必须大于0");
        }
        return redisTemplate.opsForValue().increment(key, delta);
    }

    @Override
    public long decr(String key, long delta) {
        if (delta < 0) {
            throw new RuntimeException("递减因子必须大于0");
        }
        return redisTemplate.opsForValue().increment(key, -delta);
    }

    @Override
    public Object hget(String key, String item) {
        return redisTemplate.opsForHash().get(key, item);
    }

    @Override
    public Map<Object, Object> hmget(String key) {
        return redisTemplate.opsForHash().entries(key);
    }

    @Override
    public boolean hmset(String key, Map<String, Object> map) {
        try {
            redisTemplate.opsForHash().putAll(key, map);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    @Override
    public boolean hmset(String key, Map<String, Object> map, long time) {
        try {
            redisTemplate.opsForHash().putAll(key, map);
            if (time > 0) {
                expire(key, time);
            }
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    @Override
    public boolean hset(String key, String item, Object value) {
        try {
            redisTemplate.opsForHash().put(key, item, value);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    @Override
    public boolean hset(String key, String item, Object value, long time) {
        try {
            redisTemplate.opsForHash().put(key, item, value);
            if (time > 0) {
                expire(key, time);
            }
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    @Override
    public void hdel(String key, Object... item) {
        redisTemplate.opsForHash().delete(key, item);
    }

    @Override
    public boolean hHasKey(String key, String item) {
        return redisTemplate.opsForHash().hasKey(key, item);
    }

    @Override
    public double hincr(String key, String item, double by) {
        return redisTemplate.opsForHash().increment(key, item, by);
    }

    @Override
    public double hdecr(String key, String item, double by) {
        return redisTemplate.opsForHash().increment(key, item, -by);
    }

    @Override
    public Set<Object> sGet(String key) {
        try {
            return redisTemplate.opsForSet().members(key);
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }

    @Override
    public boolean sHasKey(String key, Object value) {
        try {
            return redisTemplate.opsForSet().isMember(key, value);
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    @Override
    public long sSet(String key, Object... values) {
        try {
            return redisTemplate.opsForSet().add(key, values);
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }

    @Override
    public long sSetAndTime(String key, long time, Object... values) {
        try {
            Long count = redisTemplate.opsForSet().add(key, values);
            if (time > 0) {
                expire(key, time);
            }
            return count;
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }

    @Override
    public long sGetSetSize(String key) {
        try {
            return redisTemplate.opsForSet().size(key);
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }

    @Override
    public long setRemove(String key, Object... values) {
        try {
            Long count = redisTemplate.opsForSet().remove(key, values);
            return count;
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }

    @Override
    public List<Object> lGet(String key, long start, long end) {
        try {
            return redisTemplate.opsForList().range(key, start, end);
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }

    @Override
    public long lGetListSize(String key) {
        try {
            return redisTemplate.opsForList().size(key);
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }

    @Override
    public Object lGetIndex(String key, long index) {
        try {
            return redisTemplate.opsForList().index(key, index);
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }

    @Override
    public boolean lSetAll(String key, List<Object> value) {
        try {
            redisTemplate.opsForList().leftPushAll(key, value);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    @Override
    public boolean lSet(String key, Object value) {
        try {
            redisTemplate.opsForList().leftPush(key, value);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    @Override
    public boolean lSet(String key, Object value, long time) {
        try {
            redisTemplate.opsForList().leftPush(key, value);
            if (time > 0) {
                expire(key, time);
            }
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }

    }

    @Override
    public boolean lSetAll(String key, List<Object> value, long time) {
        try {
            redisTemplate.opsForList().leftPushAll(key, value);
            if (time > 0)
                expire(key, time);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    @Override
    public boolean rSet(String key, Object value) {
        try {
            redisTemplate.opsForList().rightPush(key, value);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    @Override
    public boolean rSet(String key, Object value, long time) {
        try {
            redisTemplate.opsForList().rightPush(key, value);
            if (time > 0) {
                expire(key, time);
            }
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }

    }

    @Override
    public boolean rSetAll(String key, List<Object> value) {
        try {
            redisTemplate.opsForList().rightPushAll(key, value);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }

    }

    @Override
    public boolean rSetAll(String key, List<Object> value, long time) {
        try {
            redisTemplate.opsForList().rightPushAll(key, value);
            if (time > 0)
                expire(key, time);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    @Override
    public boolean lUpdateIndex(String key, long index, Object value) {
        try {
            redisTemplate.opsForList().set(key, index, value);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    @Override
    public long lRemove(String key, long count, Object value) {
        try {
            Long remove = redisTemplate.opsForList().remove(key, count, value);
            return remove;
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }

    @Override
    public void rangeRemove(String key, Long stard, Long end) {
        try {
            redisTemplate.opsForList().trim(key, stard, end);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}

```

- 使用缓存

``` java
private static final String ONE_DAY_CONTENT = "SYNCHRONY_CONTENT";
    @Autowired
    ContentServer contentServer;

// 获取

   @GetMapping(value = "getCacheContent")
    public Object getCacheContent(HttpServletRequest request) {
        HttpSession session = request.getSession(false);

        if (session != null && session.getAttribute("user") != null) {
            UserPo user = (UserPo) session.getAttribute("user");
            List<Object> list = globalCache.lGet(ONE_DAY_CONTENT + user.getUserName(), 0, -1);
            return list;
        } else {
            return new ArrayList<>();
        }
    }
// 添加
  globalCache.lSet(ONE_DAY_CONTENT + user.getUserName(), content);
```

其实就是简单的redis操作，key的设置和过期时间都可以在交互时直接设置。本质还是操作redis的基本类型