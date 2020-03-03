---
title: Spring Security Oauth2：实现授权码Redis模式
description: 扩展授权码使用redis存取
categories:
 - 后端
 - Spring
tags:
 - Java
 - spring
 - SpringSecurityOauth
---


&emsp;&emsp;在之前rms-auth尝试集群部署的时候出现过一些问题，其中一个问题就是rms-auth认证中心其中一个服务生成的授权码在其他服务中无法通过认证从而无法进一步生成token，后来暂时放弃了认证中心的集群部署，近期着手处理了这个问题。

&emsp;&emsp;在Spring security oauth2授权码模式中，默认情况下生成的授权码生成后存在JVM内存中，也就是说在集群部署时，一个服务生成的授权码只存在当前服务应用的内存中，所以无法被其他服务所认证。可以通过查看源码发现：

在`AuthorizationServerEndpointsConfigurer`中：

```java
    private AuthorizationCodeServices authorizationCodeServices() {
		if (authorizationCodeServices == null) {
			authorizationCodeServices = new InMemoryAuthorizationCodeServices();
		}
		return authorizationCodeServices;
	}
```

以及在授权码模式的endpoint（`AuthorizationEndpoint`）里：

```java
private AuthorizationCodeServices authorizationCodeServices = new InMemoryAuthorizationCodeServices();
```

&emsp;&emsp;对于授权码的生成、写和读，SpringSecurityOauth2提供了接口：`AuthorizationCodeServices`，通过实现这个接口，可以自定义授权码操作，默认提供了`RandomValueAuthorizationCodeServices`、`JdbcAuthorizationCodeServices`和`InMemoryAuthorizationCodeServices`三个实现类，后两者继承了`RandomValueAuthorizationCodeServices`，`RandomValueAuthorizationCodeServices`提供生成随机授权码字符串的功能，所以我们同样只需要继承该类即可。

&emsp;&emsp;为了解决集群部署情况下的问题，我这里选择将授权码存储在redis中。

自定义一个`InRedisAuthorizationCodeServices`类：

```java
public class InRedisAuthorizationCodeServices extends RandomValueAuthorizationCodeServices {

    private String prefix = "rms:authorization_code:";

    private final RedisTemplate<String, Object> redisTemplate;

    private final JdkSerializationStrategy serializationStrategy = new JdkSerializationStrategy();

    public InRedisAuthorizationCodeServices(RedisTemplate redisTemplate) {
        Assert.notNull(redisTemplate, "redis connection factory required");
        this.redisTemplate = redisTemplate;
    }

    @Override
    protected void store(String code, OAuth2Authentication authentication) {
        String key = prefix + code;
        redisTemplate.opsForValue().set(key, serializationStrategy.serialize(authentication));
    }

    @Override
    protected OAuth2Authentication remove(String code) {
        OAuth2Authentication auth2Authentication =
                serializationStrategy.deserialize(
                        (byte[])redisTemplate.opsForValue().get(prefix + code), OAuth2Authentication.class);
        redisTemplate.delete(prefix + code);
        return auth2Authentication;
    }

    public void setPrefix(String prefix) {
        this.prefix = prefix;
    }
}
```

然后在授权服务器配置中将自定义扩展的service配置上：

```java
@EnableAuthorizationServer
@Configuration
public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {
    // 省略其他配置

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints
        // 省略其他配置
                .authorizationCodeServices(inRedisAuthorizationCodeServices(redisTemplate));
        // 省略其他配置
    }
    
    @Bean
    public AuthorizationCodeServices inRedisAuthorizationCodeServices(RedisTemplate redisTemplate) {
        return new InRedisAuthorizationCodeServices(redisTemplate);
    }
}
```

至此就完成了授权码的redis模式，实现起来还是比较简单的。


