# MiCake.Authentication.MiniProgram.WeChat

## 🍧 介绍

`AspNet Core`的微信小程序登录验证支持包。

根据微信小程序官方的登陆文档所实现的`AspNet Core`身份验证方案。该包主要完成了开发者服务器同微信接口服务器进行凭证交换的过程，用户可以根据该扩展包所提供的生命周期方法进行自定义登陆态的处理。

![pic](https://res.wx.qq.com/wxdoc/dist/assets/img/api-login.2fcc9f35.jpg)

## 🍒 用法

### 所需环境版本

+ `AspNet Core` 3.0及以上版本

### 安装包

```powershell
Install-Package MiCake.Authentication.MiniProgram.WeChat
```

### 使用

```csharp
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
        .AddWeChatMiniProgram(options =>
        {
            options.WeChatAppId = Configuration["WeChatMiniProgram:appid"];  //微信appid
            options.WeChatSecret = Configuration["WeChatMiniProgram:secret"]; //微信secret_key
        });
```

通过`AddWeChatMiniProgram`扩展方法就可以添加对微信小程序验证的支持。`WeChatAppId`和`WeChatSecret`是必须要配置的参数，因为这是和微信服务器进行交换的必要信息之一。

下方解释了`AddWeChatMiniProgram`中使用的类型为`WeChatMiniProgramOptions`的配置信息：

| 参数 | 说明 |
| ---- | ---- |
| WeChatAppId   | 小程序 appId。从微信开放平台申请。   |
| WeChatSecret     | 小程序 appSecret key。从微信开放平台申请。   |
| WeChatGrantTtype   | 授权类型，该值为:authorization_code。无须更改。   |
| WeChatJsCodeQueryString   | 登录url中,携带小程序客户端获取到code的参数名。默认为:"code"。   |
| CustomerLoginState   | 根据微信服务器返回的会话密匙进行执行自定义登录态操作。   |

需要特别说明的是`WeChatJsCodeQueryString`和`CustomerLoginState`。

`WeChatJsCodeQueryString`一般与Options中的`CallbackPath`参数搭配使用，两个值指定了需要用户访问验证接口的URL地址：

比如`CallbackPath`为“/signin-wechat”，而`WeChatJsCodeQueryString`为"code"，那么当访问"https://your-host-address/signin-wechat?code=xxx"时，则将进入到微信小程序登陆验证过程中。

*注：上方的code是您通过小程序调用 wx.login()获取到的临时登录凭证code。*

默认情况下，验证登陆地址就是`“/signin-wechat?code=”`。开放该配置的缘由是为了避免和您现有的api冲突，当有冲突时，您可以通过更改这两个参数解决。

`CustomerLoginState`是一个`Func`类型，它返回了微信服务器所返回的`openid`和`session_key`信息。您可以通过建立自有逻辑对登陆进行处理，比如根据`openid`颁发`JWT TOKEN`等操作。

就像下方的代码一样（该代码可以在仓库中的Sample中看到）：

```csharp
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
        .AddJwtBearer(options =>
        {
            options.Audience = Configuration["JwtConfig:Audience"];
            options.ClaimsIssuer = Configuration["JwtConfig:Issuer"];
        })
        .AddWeChatMiniProgram(options =>
        {
            options.WeChatAppId = Configuration["WeChatMiniProgram:appid"];
            options.WeChatSecret = Configuration["WeChatMiniProgram:secret"];

            options.CustomerLoginState += CreateToken;   //添加颁发JwtToken的步骤
        });

public async Task CreateToken(CustomerLoginStateContext context)
{
    var associateUserService = context.HttpContext.RequestServices.GetService<AssociateWeChatUser>();

    if (context.ErrCode != null && !context.ErrCode.Equals("0"))
    {
        throw new Exception(context.ErrMsg);
    }

    var jwtToken = associateUserService.GetUserToken(context.OpenId);
    var response = context.HttpContext.Response;
    await response.WriteAsync(jwtToken);
}
```

上方代码结合`JwtBearer`验证方案，在微信服务器返回成功后，根据`OpenID`信息查询到了本地数据库中的用户信息，并且为该用户创建了`Token`进行返回。

### 一些小问题

+ **如何在`CustomerLoginState`里面获取到依赖注入的服务实例？**
  
  **answer** :`CustomerLoginStateContext`里面包含了`HttpContext`，您可以根据`HttpContext.RequestServices`来进行获取。该`ServiceProvider`的范围和`Controller`的范围是一样的。

+ **如果微信服务器验证失败会怎么样**

  **answer** :当微信服务器验证失败的时候，`OpenId`等信息将为空。所以无法进行后面的验证步骤，最后将返回验证失败的错误信息。如果您需要获取到微信服务器返回的错误信息，您可以使用`WeChatMiniProgramOptions.Events.OnWeChatServerCompleted`的`Func`委托注册。

  ```csharp
    .AddWeChatMiniProgram(options =>
    {
        options.WeChatAppId = Configuration["WeChatMiniProgram:appid"];
        options.WeChatSecret = Configuration["WeChatMiniProgram:secret"];

        options.Events.OnWeChatServerCompleted += async context =>
        {
            var msg = context.ErrMsg;  //此处将获取到错误信息。
        };
    }
  ```
