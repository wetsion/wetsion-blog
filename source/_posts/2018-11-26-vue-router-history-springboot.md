---
title: Vue router history模式在SpringBoot应用中的配置
description: vue router在使用history模式时，在springboot应用又该怎么配置呢？
categories:
 - 前端
 - 后端
 - Spring
tags:
 - vue
 - spring
---

> 前面我们记录过vue router使用history模式时，项目打包扔在tomcat里时，tomcat该怎么配置来支持。这次vue项目打包后，js、css等静态资源发布在CDN上，将模版页index.html放在springboot后端项目里，再访问后端提供的url来转到index.html，那么该怎么配置呢？

如上所述，这次静态资源发布在CDN，给后端项目一个模版页，模版页中引用静态资源，通过时间戳来更新获取最新版本。springboot模版引擎用的thymeleaf，这是模版页：

```html
<!DOCTYPE html>
<html>

<head>
    <meta charset="utf-8"/>
    <meta content="width=device-width,maximum-scale=1.0,initial-scale=1.0,minimum-scale=1.0,user-scalable=yes,shrink-to-fit=no" name="viewport"/>
    <title>xxx</title>
    <link th:href="${f2eCdnUrl}+'/layout.min.css?'+${timestamp}" rel="stylesheet"/>
</head>

<body>
    <div id="app">

    </div>
    <script type="text/javascript" th:src="${f2eCdnUrl}+'/vendor.js?'+${timestamp}"></script>
	<script type="text/javascript" th:src="${f2eCdnUrl}+'/app.js?'+${timestamp}"></script>
</body>
</html>

```

springboot提供一个controller接口来访问转到模版页：

```java
@Controller
public class IndexController {

    @Value("${f2e.cdn.url}")
    private String f2eCdnUrl;

    @GetMapping("/view/")
    public String index(Model model) {
        model.addAttribute("f2eCdnUrl", f2eCdnUrl);
        model.addAttribute("timestamp", System.currentTimeMillis());
        return "index";
    }
}
```

这里的`f2eCdnUrl`是读取`application.properties`配的cdn的url，我这里对外是从`/view/`来访问index页面，而不是直接的`/`，因为如果我项目里有结合SpringSecurity，需要对`/api/**`的接口进行鉴权拦截，为了做区分，前端页面统一以`/view`开头，同样的，在vue router的配置里也需要指明`base`，如下

```js
let router = new VueRouter({
    routes: routes,
    base: '/view',
    mode: 'history'
  })
```

这个时候，启动springboot项目，可以通过/view/来访问前端项目的首页，并通过前端路由来跳转页面，但一旦刷新页面或者手动输入URL回车就会出现404页面。其实和在tomcat配置的原理类似，我们需要做的是对404做处理，当出现404的时候，再往一个URL上跳。

我们看一下`org.springframework.boot.context.embedded.EmbeddedServletContainerCustomizer`这个接口：

```java
/**
 * Strategy interface for customizing auto-configured embedded servlet containers. Any
 * beans of this type will get a callback with the container factory before the container
 * itself is started, so you can set the port, address, error pages etc.
 * <p>
 * Beware: calls to this interface are usually made from a
 * {@link EmbeddedServletContainerCustomizerBeanPostProcessor} which is a
 * {@link BeanPostProcessor} (so called very early in the ApplicationContext lifecycle).
 * It might be safer to lookup dependencies lazily in the enclosing BeanFactory rather
 * than injecting them with {@code @Autowired}.
 *
 * @author Dave Syer
 * @see EmbeddedServletContainerCustomizerBeanPostProcessor
 */
public interface EmbeddedServletContainerCustomizer {

	/**
	 * Customize the specified {@link ConfigurableEmbeddedServletContainer}.
	 * @param container the container to customize
	 */
	void customize(ConfigurableEmbeddedServletContainer container);

}
```

可以看到，这个接口是实现了策略模式的，重点是我们使用这个接口去设置`error pages`，所以我们定义一个配置类，在配置类创建一个`EmbeddedServletContainerCustomizer`这个接口的bean:

```java
@Configuration
public class ErrorPageConfig {

    @Bean
    public EmbeddedServletContainerCustomizer containerCustomizer() {
        return new EmbeddedServletContainerCustomizer() {
            @Override
            public void customize(ConfigurableEmbeddedServletContainer configurableEmbeddedServletContainer) {
                ErrorPage errorPage = new ErrorPage(HttpStatus.NOT_FOUND, "/view/");
                configurableEmbeddedServletContainer.addErrorPages(errorPage);
            }
        };
    }

}
```

可以看到，定义了一个error page，http status是not found的时候，重新跳转回之前设置的`/view/`这个路径。至此，就实现了springboot对vue router history模式的支持。