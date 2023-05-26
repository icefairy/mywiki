```xml
<configuration>

    <property name="HOME_LOG" value="./logs"/>
    <!-- 输出到控制台 -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <!-- 输出的格式 -->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    <appender name="FILE-ROLLING-INFO" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${HOME_LOG}/info.log</file>

        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>logs/archived/info.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <!-- each archived file, size max 10MB -->
            <maxFileSize>10MB</maxFileSize>
            <!-- total size of all archive files, if total size > 1GB, it will delete old archived file -->
            <totalSizeCap>1GB</totalSizeCap>
            <!-- 10 days to keep -->
            <maxHistory>10</maxHistory>
        </rollingPolicy>

        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>

        <encoder>
            <pattern>%d %p %c{1.} [%t] %m%n</pattern>
        </encoder>
    </appender>
    <appender name="FILE-ROLLING-ERROR" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${HOME_LOG}/error.log</file>

        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>logs/archived/error.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <!-- each archived file, size max 10MB -->
            <maxFileSize>10MB</maxFileSize>
            <!-- total size of all archive files, if total size > 1GB, it will delete old archived file -->
            <totalSizeCap>1GB</totalSizeCap>
            <!-- 10 days to keep -->
            <maxHistory>10</maxHistory>
        </rollingPolicy>

        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>

        <encoder>
            <pattern>%d %p %c{1.} [%t] %m%n</pattern>
        </encoder>
    </appender>
    <root level="info">
        <appender-ref ref="FILE-ROLLING-INFO"/>
        <appender-ref ref="FILE-ROLLING-ERROR"/>
        <appender-ref ref="STDOUT"/>
    </root>

</configuration>
```
