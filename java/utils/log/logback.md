# logback



## logback 的配置

application.yml

logback-spring.xml



## application.yml

```yml
logging:
	pattern:
		# 设置控制台区域日志的输出格式，日期 - 信息 换行
		console: "%d - %msg%n"
	# 只配置输出文件的路径
    # path: /var/log/tomcat/
    # 输出文件的路径，名字
    file: /var/log/tomcat/sell
    # 日志级别
    # level: debug
    # 具体到路径，如果是具体到类，最后加上类名
    level:
   		me:
      		jmix:
        		brothertakeaway:
          			dao:
            			mapper: trace
```



## logback-spring.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!--appender 是一个小的配置项，class 是处理需要使用的类-->
    <appender name="consoleLog" class="ch.qos.logback.core.ConsoleAppender">
        <!--layout 是输出的格式-->
        <layout class="ch.qos.logback.classic.PatternLayout">
            <pattern>%d - %msg%n</pattern>
        </layout>
    </appender>

    <!--滚动输出（隔一段时间输出一个文件）。因为并不是向控制台输出，所以从 layout 改为了 encoder-->
    <appender name="fileInfoLog" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!--根据指定的级别，过滤其余输出-->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <!--输出的格式-->
        <encoder>
            <pattern>%msg%n</pattern>
        </encoder>
        <!--滚动策略，以时间为单位进行滚动，每天一个日志文件-->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!--路径-->
            <fileNamePattern>/Users/qianshijie/logs/brother-takeaway/info.%d.log</fileNamePattern>
        </rollingPolicy>
    </appender>

    <appender name="fileWarnLog" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>WARN</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <encoder>
            <pattern>%msg%n</pattern>
        </encoder>
        <!--滚动策略-->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!--路径-->
            <fileNamePattern>/Users/qianshijie/logs/brother-takeaway/warn.%d.log</fileNamePattern>
        </rollingPolicy>
    </appender>

    <appender name="fileErrorLog" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <encoder>
            <pattern>%msg%n</pattern>
        </encoder>
        <!--滚动策略-->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!--路径-->
            <fileNamePattern>/Users/qianshijie/logs/brother-takeaway/error.%d.log</fileNamePattern>
        </rollingPolicy>
    </appender>

    <!--指定使用的配置项-->
    <root level="info">
        <appender-ref ref="consoleLog" />
        <appender-ref ref="fileInfoLog" />
        <appender-ref ref="fileWarnLog" />
        <appender-ref ref="fileErrorLog" />
    </root>
</configuration>
```

日志级别（等级从大到小）

```
ERROR, WARN, INFO, DEBUG, TRACE;
```