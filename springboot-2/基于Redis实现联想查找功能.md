# 基于Redis实现联想查找自动补全

- 本文的自动补全只指最前匹配

- 常用的方案有哪些？

  1. 利用数据库的模块匹配来做，利如mysql的like %这种方式来完成，虽然最前匹配能保证用到索引，但是效率不高。
  2. 利用搜索引擎，比如elasticsearch,sphinx 一般都用此方案
  3. 通过redis的有序集合来实现（本文）

   

## 一 补全原理

1. 拆分词，加入到有序集合，注意添加到redis时score都设置为0，这些字符就会按照自然排序排好。

   ``` shell
   zadd demo 0 内容
   ```

   

2. 利用zrank命令，定位关键字的位置索引，然后通过索引来获取所有以关键字开头的集合

   ``` shell
   zrank demo 关键字
   ```

   

3. 通过zrange获取数据

   ```shell
   zrange demo 1 -1
   ```

   

## 二基于java实现

1. 因为redis中不能存中文，所以需要一个转换方案，从新编解码,为了方便解码我们把字符用“-”隔开，

   ``` java 
   //unicode编码
       private String coding(String s) {
           if (s==null){
               return "";
           }
           char[] chars = s.toCharArray();
           StringBuffer buffer = new StringBuffer();
           for (char aChar : chars) {
               String s1 = Integer.toString(aChar, 16);
               buffer.append("-" + s1);
           }
           String encoding = buffer.toString();
           return encoding;
       }
   
       //unicode解码
       private String decoding(String s) {
           if (s==null){
               return "";
           }
           String[] split = s.split("-");
           StringBuffer buffer = new StringBuffer();
   
           for (String s1 : split) {
               if (!s1.trim().equals("")) {
                   char i = (char) Integer.parseInt(s1, 16);
                   buffer.append(i);
               }
           }
           return buffer.toString();
       }
   ```

2. 确定转换过的顺序，因为16进制和连接符我们确定好顺序字符标准串，然后拿前缀字符

   ``` java
     private String[] findPrefixRange(String prefix) {
            //查找出前缀字符串最后一个字符在列表中的位置
           int posn = VALID_CHARACTERS.indexOf(prefix.charAt(prefix.length() - 1));
           //找出前驱字符
           char suffix = VALID_CHARACTERS.charAt(posn > 0 ? posn - 1 : 0);
           //生成前缀字符串的前驱字符串
           String start = prefix.substring(0, prefix.length() - 1) + suffix + 'g';
           //生成前缀字符串的后继字符串
           String end = prefix + 'g';
           return new String[]{start, end};
       }
   
   ```

   

3. 实现redis查找命令

   ```java
   @Autowired
   private IGlobalCache globalCache;
   
   @Autowired
   RedisTemplate redisTemplate;
   
   private static final String REDIS_SALE_CUSTOMER = "redis_customer";
   
   private static final String VALID_CHARACTERS = "-0123456789abcdefg";
   
   public List<String> getRelateCustomerWord(String name) {
           if (StringUtils.isBlank(name)) {
               return null;
           }
           if (globalCache.hasKey(REDIS_SALE_CUSTOMER)) {
               // 拼接字段
               String[] prefixRange = findPrefixRange(coding(name));
               // 放入到redis中
               List<String> strFinds = autoFind(prefixRange);
               return strFinds;
           } 
       }
   
   
   private List<String> autoFind(String[] prefixRange) {
           List<String> list = new ArrayList<>();
           try {
               String uuid = UUID.randomUUID().toString().replaceAll("-", "");
   //	防止多个群成员可以同时操作有序集合,将相同的前驱字符串和后继字符串插入有序集合
               String start = prefixRange[0] + uuid;
               String end = prefixRange[1] + uuid;
               // 1.放入redis
               redisTemplate.opsForZSet().add(REDIS_SALE_CUSTOMER, start, 0);
               redisTemplate.opsForZSet().add(REDIS_SALE_CUSTOMER, end, 0);
               // 2.得到索引的位置
               int begin_index = redisTemplate.opsForZSet().rank(REDIS_SALE_CUSTOMER, start).intValue();
               int end_index = redisTemplate.opsForZSet().rank(REDIS_SALE_CUSTOMER, end).intValue();
               // 3.因为最多展示10个，所以计算出结束为止
               int erange = Math.min(begin_index + 9, end_index - 2);
   
               //  3.删除这两个放入的值
               redisTemplate.opsForZSet().remove(REDIS_SALE_CUSTOMER, start);
               redisTemplate.opsForZSet().remove(REDIS_SALE_CUSTOMER, end);
               // 4.获得其中的值
               Set<String> zrange = redisTemplate.opsForZSet().range(REDIS_SALE_CUSTOMER, begin_index, erange);
   
               list.addAll(zrange);
               ListIterator<String> it = list.listIterator();
               while (it.hasNext()) {
                   String next = it.next();
                   if (next.indexOf("g") != -1) {
                       it.remove();
                   } else {
                       //把16进制字符串转换回来
                       it.set(decoding(next));
                   }
               }
   
           } catch (Exception e) {
               e.printStackTrace();
           }
   
           return list;
   
       }
   
   ```

   

4. 实现redis添加命令

   ``` java
    public boolean putCustomerDataToRedis(String keyword) {
           Boolean add = redisTemplate.opsForZSet().add(REDIS_SALE_CUSTOMER, coding(keyword), 0);
           return add;
       }
   ```

   

### tips

- redis事务不支持集群，所以上附代码没有添加事务，可以考虑基于java的锁来实现对key的操作
