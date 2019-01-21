---
title: SpringCloud踩的坑：eureka接入security后unknown server
description: eureka注册中心在接入Spring secrity时出现了unknown server无法注册的问题
categories:
 - 后端
 - SpringCloud
tags:
 - SpringCloud
 - 踩坑
 - eureka
 - SpringSecurity
---

> 在使用SpringCloud eureka注册中心，接入Spring security时出现了unknown server找不到注册中心无法注册的问题

最开始我的eureka server的配置是

```java
spring.securiy.basic.enabled=true
spring.security.user.name=weixin
spring.security.user.password=weixin
```

通过查找资料，有网友说高版本不再需要这些配置项，于是新建了个配置类

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.sessionManagement().sessionCreationPolicy(SessionCreationPolicy.NEVER);
        http.authorizeRequests().anyRequest().authenticated().and().httpBasic();
    }

    @Autowired
    public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication().passwordEncoder(NoOpPasswordEncoder.getInstance())
            //admin
            .withUser("weixin").password("weixin").roles("EUREKA-CLIENT").and()
            //eureka-security-client
            .withUser("eureka-security-client").password("eureka-security-client").roles("EUREKA-CLIENT")
        ;
    }
}
```

再次启动，发现问题依旧存在，继续查找，有网友说，高版本security默认开启了csrf检验，需要手动关闭csrf检验，于是在配置类中增加了
> http.csrf().disable()

于是配置类最终是这样
```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.sessionManagement().sessionCreationPolicy(SessionCreationPolicy.NEVER);
        http.csrf().disable();
        http.authorizeRequests().anyRequest().authenticated().and().httpBasic();
    }

    @Autowired
    public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication().passwordEncoder(NoOpPasswordEncoder.getInstance())
            //admin
            .withUser("weixin").password("weixin").roles("EUREKA-CLIENT").and()
            //eureka-security-client
            .withUser("eureka-security-client").password("eureka-security-client").roles("EUREKA-CLIENT")
        ;
    }
}
```

再次启动，问题依旧存在。此时，戏剧性的，令人无语的原因终于发现了，竟然是eureka server与client中的`eureka.client.service-url.defaultZone`该配置项的问题，我写的是`eureka.client.service-url.default-zone`，不是驼峰命名，导致了这个问题，所以`defaultZone`一定要是驼峰命名，通过查看源码我们可以知道原因，找到`org.springframework.cloud.netflix.eureka.EurekaClientConfigBean`这个类，找到下面这段代码：

```java
public List<String> getEurekaServerServiceUrls(String myZone) {
    String serviceUrls = (String)this.serviceUrl.get(myZone);
    if (serviceUrls == null || serviceUrls.isEmpty()) {
        serviceUrls = (String)this.serviceUrl.get("defaultZone");
    }

    if (!StringUtils.isEmpty(serviceUrls)) {
        String[] serviceUrlsSplit = StringUtils.commaDelimitedListToStringArray(serviceUrls);
        List<String> eurekaServiceUrls = new ArrayList(serviceUrlsSplit.length);
        String[] var5 = serviceUrlsSplit;
        int var6 = serviceUrlsSplit.length;

        for(int var7 = 0; var7 < var6; ++var7) {
            String eurekaServiceUrl = var5[var7];
            if (!this.endsWithSlash(eurekaServiceUrl)) {
                eurekaServiceUrl = eurekaServiceUrl + "/";
            }

            eurekaServiceUrls.add(eurekaServiceUrl.trim());
        }

        return eurekaServiceUrls;
    } else {
        return new ArrayList();
    }
}
```

由此我们可以看到源码中就是`get("defaultZone")`，而不是default-zone

好了，以上就是无法注册服务这个坑的逐步debug的过程，以后再遇到相似问题可以借鉴下