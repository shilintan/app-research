以下来自网络资料, 非官方正式

# web 服务器

tomcat 替换为 undertow

```
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-tomcat</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-undertow</artifactId>
        </dependency>
```



```
server.undertow.accesslog.enabled=false
server.undertow.io-threads=16
server.undertow.worker-threads=500
server.undertow.max-http-post-size=0
server.undertow.direct-buffers=true
```



# 数据库连接池

druid

```
spring.datasource.druid.initial-size=10
spring.datasource.druid.min-idle=10
spring.datasource.druid.max-active=200
spring.datasource.druid.max-wait=60000
spring.datasource.druid.time-between-eviction-runs-millis=60000
spring.datasource.druid.min-evictable-idle-time-millis=300000
spring.datasource.druid.validation-query=SELECT 1 FROM DUAL
spring.datasource.druid.test-while-idle=true
spring.datasource.druid.test-on-borrow=false
spring.datasource.druid.test-on-return=false
spring.datasource.druid.max-pool-prepared-statement-per-connection-size=0
spring.datasource.druid.stat-view-servlet.enabled=false
```



# actuator

```
management.endpoints.web.exposure.include=healt
```

明天需要测试一下