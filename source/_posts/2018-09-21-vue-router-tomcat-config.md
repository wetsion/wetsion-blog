---
title: Vue router history模式在tomcat的配置
description: vue router在使用history时，部署时需要在服务器再进一步配置
categories:
 - 前端
tags:
 - vue
---

> vue router在使用history时，部署时需要在服务器再进一步配置

vue中，router使用history模式，即mode: history，可以让项目在浏览器的地址显示和正常网页URL一样，不会出现/#/这样奇奇怪怪的东西，

但使用这种history模式需要再进一步配置，否则就会发现一回车或者刷新页面就会404。而官网只介绍了Apache、Nginx的配置方法，由于

我目前是将项目build之后部署在tomcat上，所以需要在tomcat上增加一些配置。

以往我们使用Java写web项目部署在tomcat时，通常都会有一个WEB-INF文件夹，并包含一个web.xml文件，而vue项目build之后只是纯静

态资源项目，所以我们需要在build之后的dist文件夹里新增一个WEB-INF文件夹，并新建web.xml文件，文件内容如下：

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<web-app xmlns="http://java.sun.com/xml/ns/javaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd" version="3.0" metadata-complete="true">

  <display-name>Your Project Name</display-name>
  <description>
     Your Project Description
  </description>
  <error-page>  
   <error-code>404</error-code>  
   <location>/</location>  
  </error-page>  
</web-app>
```

至此再启动tomcat，再访问，就不会再出现刷新或回车后404的现象