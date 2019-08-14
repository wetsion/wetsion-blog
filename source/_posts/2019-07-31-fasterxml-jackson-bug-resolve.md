---
title: Fasterxml Jackson远程代码执行漏洞修复方案
description: Fasterxml Jackson远程代码执行漏洞修复
categories:
 - 后端
tags:
 - 踩坑
 - Java
---


最近fasterxml jackson爆出了同阿里fastjson类似的远程代码执行的漏洞，修复方案也是同样的升级版本，但jackson不同于fastjson，jackson被很多插件所依赖，影响面比较广，只单单的引入新的jackson-databind并不能完全有效，所幸官方也及时更新了一个pom，这里记录一下。

在根项目maven的pom.xml文件里的`dependencyManagement`中引入新的pom：

```java
            <dependency>
                <groupId>com.fasterxml.jackson.core</groupId>
                <artifactId>jackson-bom</artifactId>
                <version>2.9.9.2.20190727</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
```
