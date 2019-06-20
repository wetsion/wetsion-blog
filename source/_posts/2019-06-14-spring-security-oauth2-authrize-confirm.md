---
title: Spring Security Oauth2：自定义手动授权页
description: 使用自定义的授权页覆盖默认的授权页
categories:
 - 后端
 - Spring
tags:
 - Java
 - spring
 - SpringSecurityOauth
---


在基于spring security oauth2搭建了认证授权中心之后，Spring security oauth2默认提供了授权页面，但较为简陋，于是提高用户体验，我们需要自定义授权页面，首先我们需要先了解授权的流程。

授权首先是通过`get`请求访问`/oauth/authorize`接口，并携带相关参数，例如:

> http://localhost:9001/oauth/authorize?response_type=code&client_id=8&redirect_uri=http://localhost:8092/api/redirect&username=wetsion&password=$2a$11$P3utwZyiCwy0jM1Hke49WeZVIu5Lv9z6nqhqKluhlweTbhUyWK74G&state=wetsion

`org.springframework.security.oauth2.provider.endpoint.AuthorizationEndpoint`中`authorize()`负责处理get请求的`/oauth/authorize`：

```java
    @RequestMapping(value = "/oauth/authorize")
	public ModelAndView authorize(Map<String, Object> model, @RequestParam Map<String, String> parameters,
			SessionStatus sessionStatus, Principal principal) {

		// Pull out the authorization request first, using the OAuth2RequestFactory. All further logic should
		// query off of the authorization request instead of referring back to the parameters map. The contents of the
		// parameters map will be stored without change in the AuthorizationRequest object once it is created.
		AuthorizationRequest authorizationRequest = getOAuth2RequestFactory().createAuthorizationRequest(parameters);

		Set<String> responseTypes = authorizationRequest.getResponseTypes();

		if (!responseTypes.contains("token") && !responseTypes.contains("code")) {
			throw new UnsupportedResponseTypeException("Unsupported response types: " + responseTypes);
		}

		if (authorizationRequest.getClientId() == null) {
			throw new InvalidClientException("A client id must be provided");
		}

		try {

			if (!(principal instanceof Authentication) || !((Authentication) principal).isAuthenticated()) {
				throw new InsufficientAuthenticationException(
						"User must be authenticated with Spring Security before authorization can be completed.");
			}

			ClientDetails client = getClientDetailsService().loadClientByClientId(authorizationRequest.getClientId());

			// The resolved redirect URI is either the redirect_uri from the parameters or the one from
			// clientDetails. Either way we need to store it on the AuthorizationRequest.
			String redirectUriParameter = authorizationRequest.getRequestParameters().get(OAuth2Utils.REDIRECT_URI);
			String resolvedRedirect = redirectResolver.resolveRedirect(redirectUriParameter, client);
			if (!StringUtils.hasText(resolvedRedirect)) {
				throw new RedirectMismatchException(
						"A redirectUri must be either supplied or preconfigured in the ClientDetails");
			}
			authorizationRequest.setRedirectUri(resolvedRedirect);

			// We intentionally only validate the parameters requested by the client (ignoring any data that may have
			// been added to the request by the manager).
			oauth2RequestValidator.validateScope(authorizationRequest, client);

			// Some systems may allow for approval decisions to be remembered or approved by default. Check for
			// such logic here, and set the approved flag on the authorization request accordingly.
			authorizationRequest = userApprovalHandler.checkForPreApproval(authorizationRequest,
					(Authentication) principal);
			// TODO: is this call necessary?
			boolean approved = userApprovalHandler.isApproved(authorizationRequest, (Authentication) principal);
			authorizationRequest.setApproved(approved);

			// Validation is all done, so we can check for auto approval...
			if (authorizationRequest.isApproved()) {
				if (responseTypes.contains("token")) {
					return getImplicitGrantResponse(authorizationRequest);
				}
				if (responseTypes.contains("code")) {
					return new ModelAndView(getAuthorizationCodeResponse(authorizationRequest,
							(Authentication) principal));
				}
			}

			// Place auth request into the model so that it is stored in the session
			// for approveOrDeny to use. That way we make sure that auth request comes from the session,
			// so any auth request parameters passed to approveOrDeny will be ignored and retrieved from the session.
			model.put("authorizationRequest", authorizationRequest);

			return getUserApprovalPageResponse(model, authorizationRequest, (Authentication) principal);

		}
		catch (RuntimeException e) {
			sessionStatus.setComplete();
			throw e;
		}

	}
```

如果isApproved为false，则调用`getUserApprovalPageResponse`：

```java
private ModelAndView getUserApprovalPageResponse(Map<String, Object> model,
			AuthorizationRequest authorizationRequest, Authentication principal) {
		if (logger.isDebugEnabled()) {
			logger.debug("Loading user approval page: " + userApprovalPage);
		}
		model.putAll(userApprovalHandler.getUserApprovalRequest(authorizationRequest, principal));
		return new ModelAndView(userApprovalPage, model);
	}
```

这里： `String userApprovalPage = "forward:/oauth/confirm_access"`

即转发到`/oauth/confirm_access`

默认是由`org.springframework.security.oauth2.provider.endpoint.WhitelabelApprovalEndpoint`类处理`/oauth/confirm_access`请求，将用户引导到默认授权页（这里默认授权页是由代码渲染的html页面）：

```java
@FrameworkEndpoint
@SessionAttributes("authorizationRequest")
public class WhitelabelApprovalEndpoint {

	@RequestMapping("/oauth/confirm_access")
	public ModelAndView getAccessConfirmation(Map<String, Object> model, HttpServletRequest request) throws Exception {
		final String approvalContent = createTemplate(model, request);
		if (request.getAttribute("_csrf") != null) {
			model.put("_csrf", request.getAttribute("_csrf"));
		}
		View approvalView = new View() {
			@Override
			public String getContentType() {
				return "text/html";
			}

			@Override
			public void render(Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) throws Exception {
				response.setContentType(getContentType());
				response.getWriter().append(approvalContent);
			}
		};
		return new ModelAndView(approvalView, model);
	}

	// 省略其他
}
```

看到这里，我们就已经可以知道，想要自定义授权页，只需要替换掉默认的`WhitelabelApprovalEndpoint`即可。

 - 引入thymeleaf依赖，在resource下创建templates文件夹，再在templates文件夹创建一个grant.html文件：

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>授权</title>
</head>
<!-- 省略样式 -->
<body style="margin: 0px">
<div class="title">
    <div class="title-right">大数据RMS统一管理平台 授权</div>
    <div class="title-left">
        <a href="#help">help</a>
    </div>
</div>
<div class="container">
    <h3 th:text="'应用 '+${clientName}+' 请求授权，该应用将获取你的以下信息'"></h3>
    <p>昵称，账号</p>
    授权后表明你已同意
    <form method="post" action="/oauth/authorize">
        <input type="hidden" name="user_oauth_approval" value="true">
        <input type="hidden" name="scope.all" value="true">
        <input type="hidden" name="_csrf" th:value="${_csrf.getToken()}"/>
        <button class="btn" type="submit"> 同意/授权</button>
    </form>
</div>
</body>
</html>
```

 - 在`applicaiton.properties`中配置thymeleaf:

```java
spring.thymeleaf.prefix=classpath:/templates/
spring.thymeleaf.cache=false
spring.thymeleaf.suffix=.html
```

 - 创建处理`/oauth/confirm_access`请求的controller覆盖默认的`WhitelabelApprovalEndpoint`:

```java
@Controller
@SessionAttributes("authorizationRequest")
public class GrantController {

    @Autowired
    private PlatformService platformService;

    @RequestMapping("/oauth/confirm_access")
    public ModelAndView getAccessConfirmation(Map<String, Object> model, HttpServletRequest request) {
        AuthorizationRequest authorizationRequest = (AuthorizationRequest) model.get("authorizationRequest");
        ModelAndView view = new ModelAndView();
        view.setViewName("grant");
        view.addObject("clientId", authorizationRequest.getClientId());
        PlatformDO platformDO = platformService.getById(Long.parseLong(authorizationRequest.getClientId()));
        view.addObject("clientName", platformDO.getNameCn());
        return view;
    }
}
```


至此，需要用户授权的`/oauth/authorize`请求将会跳转到自定义的授权页面，就完成了自定义授权页面的开发。
