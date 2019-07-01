---
title: SpringBoot：整合Actuator实现动态修改日志级别
description: SpringBoot整合Actuator实现动态修改日志级别
categories:
 - 后端
 - Spring
tags:
 - Java
 - spring
---


&emsp;&emsp;由于线上服务器是没有权限进行操作的，所以只能靠打印日志来追踪问题，但如果所有日志都使用info级别也不行，这样日志文件就会太大了，占硬盘空间。对于有的日志要使用debug级别，而如果修改配置文件，项目还得重启才能生效，所以就想到要动态修改日志级别。

引入actuator依赖：

```java
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```

`application.properties`文件中加入：

```java
management.security.enabled=false
```

关闭安全认证。

通过GET请求访问`/loggers`接口，获取到所有的日志级别，类似如下：

```java
{
    "levels": [
        "OFF",
        "ERROR",
        "WARN",
        "INFO",
        "DEBUG",
        "TRACE"
    ],
    "loggers": {
        "ROOT": {
            "configuredLevel": "INFO",
            "effectiveLevel": "INFO"
        },
        "com": {
            "configuredLevel": null,
            "effectiveLevel": "INFO"
        },
        // 省略n行
    }
}
```

loggers中的每个key即为ROOT和各个包路径对应的日志级别。

通过POST请求访问`/loggers/{key}`接口，key即为上面获取到的结果中loggers的key，也就是ROOT和各个包路径，请求体如下：


```java
{
	"configuredLevel": "DEBUG"
}
```

这样，就将key对应的日志级别设置成了debug。

注意，系统重启后，修改的将会恢复。