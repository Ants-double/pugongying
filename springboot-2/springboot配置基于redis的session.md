# springboot配置基于redis的session

- 为什么要使用session共享？

  因为在多个服务器之间我准确认识同一客户端，以方便保存会话状态。

- 只有redis这一种方案吗？或者说基于redis的Session能有什么优势？

  不是的，我们的目的是要让会话状态得以保存和准确识别，比如nginx基于hash来进行轮询，那每次会话都会到同样一台服务上，也可以保证会话状态。

  而且，本质是我们把会话状态放入nosql数据库，每次从数据库中读取还原。基于redis方案比较成熟还有就是redis内存操作等其它优势，可以去redis官网了解。

## 通过redis来实现Session共享

1. 引入必要的pom依赖

   ``` xml
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-data-redis</artifactId>
   </dependency>
   
   <dependency>
       <groupId>org.apache.commons</groupId>
       <artifactId>commons-pool2</artifactId>
   </dependency>
   <dependency>
       <groupId>org.springframework.session</groupId>
       <artifactId>spring-session-data-redis</artifactId>
   </dependency>
   ```

   

2. 编写必要的配置信息

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

   

3. 启动基于redis的session共享,在启动类上添加@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)

   ``` java
   
   @SpringBootApplication
   @EnableCaching
   @EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
   public class SaleWhiteBoardApplication {
   
       public static void main(String[] args) {
           SpringApplication.run(SaleWhiteBoardApplication.class, args);
       }
   
   }
   
   ```

   **maxInactiveIntervalInSeconds: 设置 Session 失效时间，使用 Spring Session 之后，原 Spring Boot 配置文件 `application.yml` 中的 `server.session.timeout` 属性不再生效**

