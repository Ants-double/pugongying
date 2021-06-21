# springboot配置多环境日志打印

- 为什么要配置？

  一般开发用windows ，而程序最终多在linux系统上运行，每次打包发布都要更改对应的配置信息。

- 有几种方法可以实现？

  目前常见的有两种方法，一种是基于logback-spring.xml文件中来配置，优点是只有一个文件，缺点是要共用一部分模版配置，另一种是定义不同的Logback配置文件，优点可以针对不同的条件激活不同的配置，缺点是文件过多。其实两个原理是一样的，把日志托管给spring。

- 需要引用哪些依赖？

  需要引用日志模块，但是已经包含在springboot-start里面了。

## 通过不同的文件来达到多环境配置

1. 首先在resources新建log文件夹并分在文件夹中别创建logback-dev.xml ,logback-test.xml ,logback-prod.xml文件

   -- logback-dev.xml

   ``` xml
   
   <included>
       <!--0. 日志格式和颜色渲染 -->
       <!-- 彩色日志依赖的渲染类 -->
       <conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter" />
       <conversionRule conversionWord="wex" converterClass="org.springframework.boot.logging.logback.WhitespaceThrowableProxyConverter" />
       <conversionRule conversionWord="wEx" converterClass="org.springframework.boot.logging.logback.ExtendedWhitespaceThrowableProxyConverter" />
   
       <!-- 控制台彩色日志格式  %black", "%red", "%green","%yellow","%blue", "%magenta","%cyan", "%white", "%gray", "%boldRed","%boldGreen", "%boldYellow", "%boldBlue", "%boldMagenta""%boldCyan", "%boldWhite" and "%highlight" 支持的颜色-->
       <property name="CONSOLE_LOG_PATTERN" value="%boldMagenta(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %boldYellow(${LOG_LEVEL_PATTERN:-%5p}) %boldCyan(${PID:- }){magenta} %boldGreen(---){faint} %boldGreen([%15.15t]){faint} %boldCyan(%-40.40logger{50}){cyan} %boldYellow(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}"/>
       <!--文件日志打印格式 -->
       <property name="FILE_LOG_PATTERN" value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n"/>
       <!--1. 输出到控制台 , class 用来指定输出策略-->
       <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
           <!--此日志appender是为开发使用，只配置最底级别，控制台输出的日志级别是大于或等于此级别的日志信息-->
           <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
               <level>debug</level> <!--控制台输出的日志级别-->
           </filter>
           <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
               <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
               <pattern>${CONSOLE_LOG_PATTERN}</pattern>
           </encoder>
       </appender>
   
       <!--2. 输出到文档-->
       <!-- 2.1 level为 DEBUG 日志，时间滚动输出  -->
       <appender name="DEBUG_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
           <!-- 正在记录的日志文档的路径及文档名 -->
           <file>${log.path}/white_dev_debug.log</file>
           <!--日志文档输出格式-->
           <encoder>
               <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
               <charset>UTF-8</charset> <!-- 设置字符集 -->
           </encoder>
           <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
           <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
               <!-- 日志归档 -->
               <fileNamePattern>${log.path}/white-dev-debug-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
               <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                   <maxFileSize>100MB</maxFileSize>
               </timeBasedFileNamingAndTriggeringPolicy>
               <!--日志文档保留天数-->
               <maxHistory>15</maxHistory>
           </rollingPolicy>
           <!-- 此日志文档只记录debug级别的 -->
           <filter class="ch.qos.logback.classic.filter.LevelFilter">
               <level>debug</level>
               <onMatch>ACCEPT</onMatch>
               <onMismatch>DENY</onMismatch>
           </filter>
       </appender>
   
       <!-- 2.2 level为 INFO 日志，时间滚动输出  -->
       <appender name="INFO_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
           <!-- 正在记录的日志文档的路径及文档名 -->
           <file>${log.path}/white_dev_info.log</file>
           <!--日志文档输出格式-->
           <encoder>
               <pattern>${FILE_LOG_PATTERN}</pattern>
               <charset>UTF-8</charset>
           </encoder>
           <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
           <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
               <!-- 每天日志归档路径以及格式 -->
               <fileNamePattern>${log.path}/white-dev-info-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
               <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                   <maxFileSize>100MB</maxFileSize>
               </timeBasedFileNamingAndTriggeringPolicy>
               <!--日志文档保留天数-->
               <maxHistory>15</maxHistory>
           </rollingPolicy>
           <!-- 此日志文档只记录info级别的 -->
           <filter class="ch.qos.logback.classic.filter.LevelFilter">
               <level>info</level>
               <onMatch>ACCEPT</onMatch>
               <onMismatch>DENY</onMismatch>
           </filter>
       </appender>
   
       <!-- 2.3 level为 WARN 日志，时间滚动输出  -->
       <appender name="WARN_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
           <!-- 正在记录的日志文档的路径及文档名 -->
           <file>${log.path}/white_dev_warn.log</file>
           <!--日志文档输出格式-->
           <encoder>
               <pattern>${FILE_LOG_PATTERN}</pattern>
               <charset>UTF-8</charset> <!-- 此处设置字符集 -->
           </encoder>
           <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
           <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
               <fileNamePattern>${log.path}/white-dev-warn-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
               <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                   <maxFileSize>100MB</maxFileSize>
               </timeBasedFileNamingAndTriggeringPolicy>
               <!--日志文档保留天数-->
               <maxHistory>15</maxHistory>
           </rollingPolicy>
           <!-- 此日志文档只记录warn级别的 -->
           <filter class="ch.qos.logback.classic.filter.LevelFilter">
               <level>warn</level>
               <onMatch>ACCEPT</onMatch>
               <onMismatch>DENY</onMismatch>
           </filter>
       </appender>
   
       <!-- 2.4 level为 ERROR 日志，时间滚动输出  -->
       <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
           <!-- 正在记录的日志文档的路径及文档名 -->
           <file>${log.path}/white_dev_error.log</file>
           <!--日志文档输出格式-->
           <encoder>
               <pattern>${FILE_LOG_PATTERN}</pattern>
               <charset>UTF-8</charset> <!-- 此处设置字符集 -->
           </encoder>
           <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
           <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
               <fileNamePattern>${log.path}/white-dev-error-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
               <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                   <maxFileSize>100MB</maxFileSize>
               </timeBasedFileNamingAndTriggeringPolicy>
               <!--日志文档保留天数-->
               <maxHistory>15</maxHistory>
           </rollingPolicy>
           <!-- 此日志文档只记录ERROR级别的 -->
           <filter class="ch.qos.logback.classic.filter.LevelFilter">
               <level>ERROR</level>
               <onMatch>ACCEPT</onMatch>
               <onMismatch>DENY</onMismatch>
           </filter>
       </appender>
   
       <root level="DEBUG">
           <appender-ref ref="CONSOLE" />
           <appender-ref ref="DEBUG_FILE" />
           <appender-ref ref="INFO_FILE" />
           <appender-ref ref="WARN_FILE" />
           <appender-ref ref="ERROR_FILE" />
       </root>
   
   
   </included>
   ```

   -- logback-test.xml

   ``` xml
   
   <included>
       <!--0. 日志格式和颜色渲染 -->
       <!-- 彩色日志依赖的渲染类 -->
       <conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter" />
       <conversionRule conversionWord="wex" converterClass="org.springframework.boot.logging.logback.WhitespaceThrowableProxyConverter" />
       <conversionRule conversionWord="wEx" converterClass="org.springframework.boot.logging.logback.ExtendedWhitespaceThrowableProxyConverter" />
   
       <!-- 控制台彩色日志格式  %black", "%red", "%green","%yellow","%blue", "%magenta","%cyan", "%white", "%gray", "%boldRed","%boldGreen", "%boldYellow", "%boldBlue", "%boldMagenta""%boldCyan", "%boldWhite" and "%highlight" 支持的颜色-->
       <property name="CONSOLE_LOG_PATTERN" value="%boldMagenta(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %boldYellow(${LOG_LEVEL_PATTERN:-%5p}) %boldCyan(${PID:- }){magenta} %boldGreen(---){faint} %boldGreen([%15.15t]){faint} %boldCyan(%-40.40logger{50}){cyan} %boldYellow(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}"/>
       <!--文件日志打印格式 -->
       <property name="FILE_LOG_PATTERN" value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n"/>
       <!--1. 输出到控制台 , class 用来指定输出策略-->
       <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
           <!--此日志appender是为开发使用，只配置最底级别，控制台输出的日志级别是大于或等于此级别的日志信息-->
           <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
               <level>debug</level> <!--控制台输出的日志级别-->
           </filter>
           <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
               <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
               <pattern>${CONSOLE_LOG_PATTERN}</pattern>
           </encoder>
       </appender>
   
       <!--2. 输出到文档-->
       <!-- 2.1 level为 DEBUG 日志，时间滚动输出  -->
       <appender name="DEBUG_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
           <!-- 正在记录的日志文档的路径及文档名 -->
           <file>${log.path}/white_test_debug.log</file>
           <!--日志文档输出格式-->
           <encoder>
               <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
               <charset>UTF-8</charset> <!-- 设置字符集 -->
           </encoder>
           <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
           <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
               <!-- 日志归档 -->
               <fileNamePattern>${log.path}/white-test-debug-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
               <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                   <maxFileSize>100MB</maxFileSize>
               </timeBasedFileNamingAndTriggeringPolicy>
               <!--日志文档保留天数-->
               <maxHistory>15</maxHistory>
           </rollingPolicy>
           <!-- 此日志文档只记录debug级别的 -->
           <filter class="ch.qos.logback.classic.filter.LevelFilter">
               <level>debug</level>
               <onMatch>ACCEPT</onMatch>
               <onMismatch>DENY</onMismatch>
           </filter>
       </appender>
   
       <!-- 2.2 level为 INFO 日志，时间滚动输出  -->
       <appender name="INFO_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
           <!-- 正在记录的日志文档的路径及文档名 -->
           <file>${log.path}/white_test_info.log</file>
           <!--日志文档输出格式-->
           <encoder>
               <pattern>${FILE_LOG_PATTERN}</pattern>
               <charset>UTF-8</charset>
           </encoder>
           <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
           <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
               <!-- 每天日志归档路径以及格式 -->
               <fileNamePattern>${log.path}/white-test-info-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
               <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                   <maxFileSize>100MB</maxFileSize>
               </timeBasedFileNamingAndTriggeringPolicy>
               <!--日志文档保留天数-->
               <maxHistory>15</maxHistory>
           </rollingPolicy>
           <!-- 此日志文档只记录info级别的 -->
           <filter class="ch.qos.logback.classic.filter.LevelFilter">
               <level>info</level>
               <onMatch>ACCEPT</onMatch>
               <onMismatch>DENY</onMismatch>
           </filter>
       </appender>
   
       <!-- 2.3 level为 WARN 日志，时间滚动输出  -->
       <appender name="WARN_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
           <!-- 正在记录的日志文档的路径及文档名 -->
           <file>${log.path}/white_test_warn.log</file>
           <!--日志文档输出格式-->
           <encoder>
               <pattern>${FILE_LOG_PATTERN}</pattern>
               <charset>UTF-8</charset> <!-- 此处设置字符集 -->
           </encoder>
           <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
           <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
               <fileNamePattern>${log.path}/white-test-warn-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
               <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                   <maxFileSize>100MB</maxFileSize>
               </timeBasedFileNamingAndTriggeringPolicy>
               <!--日志文档保留天数-->
               <maxHistory>15</maxHistory>
           </rollingPolicy>
           <!-- 此日志文档只记录warn级别的 -->
           <filter class="ch.qos.logback.classic.filter.LevelFilter">
               <level>warn</level>
               <onMatch>ACCEPT</onMatch>
               <onMismatch>DENY</onMismatch>
           </filter>
       </appender>
   
       <!-- 2.4 level为 ERROR 日志，时间滚动输出  -->
       <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
           <!-- 正在记录的日志文档的路径及文档名 -->
           <file>${log.path}/white_test_error.log</file>
           <!--日志文档输出格式-->
           <encoder>
               <pattern>${FILE_LOG_PATTERN}</pattern>
               <charset>UTF-8</charset> <!-- 此处设置字符集 -->
           </encoder>
           <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
           <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
               <fileNamePattern>${log.path}/white-test-error-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
               <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                   <maxFileSize>100MB</maxFileSize>
               </timeBasedFileNamingAndTriggeringPolicy>
               <!--日志文档保留天数-->
               <maxHistory>15</maxHistory>
           </rollingPolicy>
           <!-- 此日志文档只记录ERROR级别的 -->
           <filter class="ch.qos.logback.classic.filter.LevelFilter">
               <level>ERROR</level>
               <onMatch>ACCEPT</onMatch>
               <onMismatch>DENY</onMismatch>
           </filter>
       </appender>
   
       <root level="DEBUG">
           <appender-ref ref="CONSOLE" />
           <appender-ref ref="DEBUG_FILE" />
           <appender-ref ref="INFO_FILE" />
           <appender-ref ref="WARN_FILE" />
           <appender-ref ref="ERROR_FILE" />
       </root>
   
   
   </included>
   ```

   -- logback-prod.xml

   ``` xml
   <included>
       <!-- name的值是变量的名称，value的值时变量定义的值。通过定义的值会被插入到logger上下文中。定义后，可以使“${}”来使用变量。 -->
   <!--    <property name="log.path" value="/Users/andy/up/springboot2/" />-->
       <!--0. 日志格式和颜色渲染 -->
       <!-- 彩色日志依赖的渲染类 -->
       <conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter" />
       <conversionRule conversionWord="wex" converterClass="org.springframework.boot.logging.logback.WhitespaceThrowableProxyConverter" />
       <conversionRule conversionWord="wEx" converterClass="org.springframework.boot.logging.logback.ExtendedWhitespaceThrowableProxyConverter" />
   
       <!-- 控制台彩色日志格式  %black", "%red", "%green","%yellow","%blue", "%magenta","%cyan", "%white", "%gray", "%boldRed","%boldGreen", "%boldYellow", "%boldBlue", "%boldMagenta""%boldCyan", "%boldWhite" and "%highlight" 支持的颜色-->
       <property name="CONSOLE_LOG_PATTERN" value="%boldMagenta(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %boldYellow(${LOG_LEVEL_PATTERN:-%5p}) %boldCyan(${PID:- }){magenta} %boldGreen(---){faint} %boldGreen([%15.15t]){faint} %boldCyan(%-40.40logger{50}){cyan} %boldYellow(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}"/>
       <!--文件日志打印格式 -->
       <property name="FILE_LOG_PATTERN" value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n"/>
       <!--1. 输出到控制台 , class 用来指定输出策略-->
       <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
           <!--此日志appender是为开发使用，只配置最底级别，控制台输出的日志级别是大于或等于此级别的日志信息-->
           <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
               <level>info</level>
           </filter>
           <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
               <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
               <pattern>${CONSOLE_LOG_PATTERN}</pattern>
           </encoder>
       </appender>
   
       <!--2. 输出到文档-->
       <!-- 2.1 level为 DEBUG 日志，时间滚动输出  -->
       <appender name="DEBUG_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
           <!-- 正在记录的日志文档的路径及文档名 -->
           <file>${log.path}/white_prod_debug.log</file>
           <!--日志文档输出格式-->
           <encoder>
               <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
               <charset>UTF-8</charset> <!-- 设置字符集 -->
           </encoder>
           <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
           <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
               <!-- 日志归档 -->
               <fileNamePattern>${log.path}/white-prod-debug-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
               <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                   <maxFileSize>100MB</maxFileSize>
               </timeBasedFileNamingAndTriggeringPolicy>
               <!--日志文档保留天数-->
               <maxHistory>15</maxHistory>
           </rollingPolicy>
           <!-- 此日志文档只记录debug级别的 -->
           <filter class="ch.qos.logback.classic.filter.LevelFilter">
               <level>debug</level>
               <onMatch>ACCEPT</onMatch>
               <onMismatch>DENY</onMismatch>
           </filter>
       </appender>
   
       <!-- 2.2 level为 INFO 日志，时间滚动输出  -->
       <appender name="INFO_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
           <!-- 正在记录的日志文档的路径及文档名 -->
           <file>${log.path}/white_prod_info.log</file>
           <!--日志文档输出格式-->
           <encoder>
               <pattern>${FILE_LOG_PATTERN}</pattern>
               <charset>UTF-8</charset>
           </encoder>
           <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
           <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
               <!-- 每天日志归档路径以及格式 -->
               <fileNamePattern>${log.path}/white-prod-info-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
               <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                   <maxFileSize>100MB</maxFileSize>
               </timeBasedFileNamingAndTriggeringPolicy>
               <!--日志文档保留天数-->
               <maxHistory>15</maxHistory>
           </rollingPolicy>
           <!-- 此日志文档只记录info级别的 -->
           <filter class="ch.qos.logback.classic.filter.LevelFilter">
               <level>info</level>
               <onMatch>ACCEPT</onMatch>
               <onMismatch>DENY</onMismatch>
           </filter>
       </appender>
   
       <!-- 2.3 level为 WARN 日志，时间滚动输出  -->
       <appender name="WARN_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
           <!-- 正在记录的日志文档的路径及文档名 -->
           <file>${log.path}/white_prod_warn.log</file>
           <!--日志文档输出格式-->
           <encoder>
               <pattern>${FILE_LOG_PATTERN}</pattern>
               <charset>UTF-8</charset> <!-- 此处设置字符集 -->
           </encoder>
           <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
           <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
               <fileNamePattern>${log.path}/white-prod-warn-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
               <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                   <maxFileSize>100MB</maxFileSize>
               </timeBasedFileNamingAndTriggeringPolicy>
               <!--日志文档保留天数-->
               <maxHistory>15</maxHistory>
           </rollingPolicy>
           <!-- 此日志文档只记录warn级别的 -->
           <filter class="ch.qos.logback.classic.filter.LevelFilter">
               <level>warn</level>
               <onMatch>ACCEPT</onMatch>
               <onMismatch>DENY</onMismatch>
           </filter>
       </appender>
   
       <!-- 2.4 level为 ERROR 日志，时间滚动输出  -->
       <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
           <!-- 正在记录的日志文档的路径及文档名 -->
           <file>${log.path}/white_prod_error.log</file>
           <!--日志文档输出格式-->
           <encoder>
               <pattern>${FILE_LOG_PATTERN}</pattern>
               <charset>UTF-8</charset> <!-- 此处设置字符集 -->
           </encoder>
           <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
           <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
               <fileNamePattern>${log.path}/white-prod-error-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
               <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                   <maxFileSize>100MB</maxFileSize>
               </timeBasedFileNamingAndTriggeringPolicy>
               <!--日志文档保留天数-->
               <maxHistory>15</maxHistory>
           </rollingPolicy>
           <!-- 此日志文档只记录ERROR级别的 -->
           <filter class="ch.qos.logback.classic.filter.LevelFilter">
               <level>ERROR</level>
               <onMatch>ACCEPT</onMatch>
               <onMismatch>DENY</onMismatch>
           </filter>
       </appender>
   
       <!--
           <logger>用来设置某一个包或者具体的某一个类的日志打印级别、
           以及指定<appender>。<logger>仅有一个name属性，
           一个可选的level和一个可选的addtivity属性。
           name:用来指定受此logger约束的某一个包或者具体的某一个类。
           level:用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，
                 还有一个特俗值INHERITED或者同义词NULL，代表强制执行上级的级别。
                 如果未设置此属性，那么当前logger将会继承上级的级别。
           addtivity:是否向上级logger传递打印信息。默认是true。
           <logger name="org.springframework.web" level="info"/>
           <logger name="org.springframework.scheduling.annotation.ScheduledAnnotationBeanPostProcessor" level="INFO"/>
       -->
   
       <!--
           使用mybatis的时候，sql语句是debug下才会打印，而这里我们只配置了info，所以想要查看sql语句的话，有以下两种操作：
           第一种把<root level="info">改成<root level="DEBUG">这样就会打印sql，不过这样日志那边会出现很多其他消息
           第二种就是单独给dao下目录配置debug模式，代码如下，这样配置sql语句会打印，其他还是正常info级别：
           【logging.level.org.mybatis=debug logging.level.dao=debug】
        -->
   
       <!--
         root节点是必选节点，用来指定最基础的日志输出级别，只有一个level属性 ,这个日志的级别优先级比上面单个配置的优先级高
           举例， 如果root 配置的是INFO ，控制台输出配置级别为debug 则不会打印debug 日志
           level:用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，
           不能设置为INHERITED或者同义词NULL。默认是DEBUG
           可以包含零个或多个元素，标识这个appender将会添加到这个logger。
       -->
   
       <!-- 4.2 生产环境:输出到文档-->
       <root level="info">
           <appender-ref ref="CONSOLE" />
           <appender-ref ref="DEBUG_FILE" />
           <appender-ref ref="INFO_FILE" />
           <appender-ref ref="ERROR_FILE" />
           <appender-ref ref="WARN_FILE" />
       </root>
   
   </included>
   ```

   

2. 在resources中创建logback-spring.xml文件,主要是根据不同的spring.profiles.active引入不同的文件到logback-spring.xml中来实现控制，内容如下

   ``` xml
   <configuration scan="false">
       <contextName>logback</contextName>
       <property name="log.path" value="./logs"/>
       <property name="log.pattern" value="%-4relative \\\[%thread\\\] %-5level %logger{35} - %msg %n"/>
   
       <springProperty scope="context" name="profile" source="spring.profiles.active" defaultValue="dev"/>
       <logger level="TRACE" name="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping"/>
       <include resource="log/logback-${profile}.xml"/>
   </configuration>
   ```

   -- 说明

   a. spring.profiles.active定义在application文件中的话，并不会生效，Logback只会去加载名为 **logback-spring.profiles.active_IS_UNDEFINED.xml** 的文件，需要设置其系统环境变量方可生效。可以通过<springProperty>来设置

   b. springboot2 日志规划有所改变，只有TRACE才能提供更加详细的信息

   至此完成了相关配置。

   

   **- 也可以每个配置文件都写成完成的logback.xml格式，通过配置不同的logging.config来实现 **

   ``` yaml
   # dev
   logging:
     config: classpath:log/logback-dev.xml
     
   # test
   logging:
     config: classpath:log/logback-test.xml
     
   # prod
   logging:
     config: classpath:log/logback-prod.xml
   
   ```

   

## 基于logback-spring.xml实现多环境配置

``` xml
<configuration scan="false" debug="true">
    <contextName>logback</contextName>
    <property name="log.path" value="./logs"/>
    <property name="log.pattern" value="%-4relative \[%thread\] %-5level %logger{35} - %msg %n"/>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${log.pattern}</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.path}/log.log</file>
        <encoder>
            <pattern>${log.pattern}</pattern>
            <charset>UTF-8</charset>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.path}/file/log-%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>15</maxHistory>
        </rollingPolicy>
    </appender>

    <springProfile name="test">
        <root level="DEBUG">
            <appender-ref ref="FILE"/>
        </root>
    </springProfile>
    <springProfile name="dev">
        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>
</configuration>
```

从上述配置中，可以看到定义了两个appender：一个叫做CONSOLE，用于把日志输出到控制台；另一个叫做FILE，用于把日志输出到文件中。同时还定义了两个不同环境的root配置：dev环境下，给root添加了叫做CONSOLE的appender；test环境下，给root添加了叫做FILE的appender。

通过指定application配置文件中的 `spring.profiles.active=dev` 或者通过指令 `java -jar xx.jar --spring.profiles.active=dev` 运行程序时，会发现日志只在控制台输出。同样的，当指定为test环境时，日志也只在文件中输出。

**PS：输出到文件的appender，只要其配置生效，即使在当前的环境下，并未使用，也会生成相应的目录和文件信息。**

##  配置属性说明

1. logging.file，设置文件，可以是绝对路径，也可以是相对路径。如：`logging.file=my.log`

2. logging.path，设置目录，会在该目录下创建spring.log文件，并写入日志内容，如：`logging.path=/var/log`

   *如果只配置 logging.file，会在项目的当前路径下生成一个 xxx.log 日志文件。
   如果只配置 logging.path，在 /var/log文件夹生成一个日志文件为 spring.log*

   > 注：二者不能同时使用，如若同时使用，则只有logging.file生效

3. Spring Boot官方推荐优先使用带有`-spring`的文件名作为你的日志配置（如使用`logback-spring.xml`，而不是`logback.xml`），命名为logback-spring.xml的日志配置文件，spring boot可以为它添加一些spring boot特有的配置项,上面是默认的命名规则，并且放在`src/main/resources`下面即可。

4. 如果你即想完全掌控日志配置，但又不想用`logback.xml`作为`Logback`配置的名字，可以在`application.properties`配置文件里面通过logging.config属性指定自定义的名字：虽然一般并不需要改变配置文件的名字，但是如果你想针对不同运行时Profile使用不同的日志配置

5. logback相关字段说明

   - `addtivity`:是否向上级logger传递打印信息。默认是true。
   - scan:当此属性设置为true时，配置文件如果发生改变，将会被重新加载，默认值为true。
   - scanPeriod:设置监测配置文件是否有修改的时间间隔，如果没有给出时间单位，默认单位是毫秒。当scan为true时，此属性生效。默认的时间间隔为1分钟。
   - debug:当此属性设置为true时，将打印出logback内部日志信息，实时查看logback运行状态。默认值为false。

   

