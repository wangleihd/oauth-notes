# OAuth 


## 1. OAuth Roles 角色

OAuth 定义了四个角色

**Resource Owner** User

授权应用访问自己账号的用户，应用访问账号被限于授权所允许的*scope*。

**Resource Server** API

保存受保护的用户账户。

**Authorization Server** API

提供用户许可界面，认证成功后给应用发送access token。

**Client**  应用 Applications

需要访问用户账号的应用。
它必须提前获得用户授权，而该授权必须被API验证通过。

## 2. Abstract Protocol Flow 一般流程

下图展示了一般流程，根据grant type的不同，细节略有不同。

![](imgs/abstract_flow.png)

1. 应用向用户做申请授权(authorization request)，以访问服务资源
2. 如果用户授权这个申请，则应用可以获得用户的authorization grant
3. 应用向服务提供自己的application identity和用户的authorization grant，以请求access token 
4. 如果application identity和authorization grant都有效，则服务会向应用发送一个access token。授权结束。
5. 应用从服务请求资源，并提供access token。
6. 如果access token有效，则服务向应用提供资源。


## 3. Application Registration 应用注册 

在使用OAuth之前，必须在服务里注册应用。
这个过程一般是在服务网站的注册表单(registration form)里进行的，你需要提供如下信息

- Application Name
- Application Website
- Redirect URI or Callback URL

### 3.1 Redirect URI

redirect URI是服务授权（或拒绝）应用后，将用户重定向的网址。然后应用就会处理authorization codes或access tokens。

服务仅会重定向到注册过的这个URI，以避免攻击。而任何redirect URIs必须被TLS保护（以`https`开头），这可以防止在授权过程中token被截取。

原生应用可能注册一个特殊的URL，比如demoapp://redirect。

### 3.2 Client ID and Secret

注册以后，服务会返回client credentials，包括一个client identifier和一个client secret。

- **client ID** 是一个公开字符串，被服务用来识别应用，也用来构建authorization URLs提供给用户。
- **client secret** 当应用请求访问用户账号时，在服务处验证应用的identity，必须在应用和服务间保密。如果应用不能保证保密，比如SPA或原生应用，就不使用secret。

## 4. Authorization Grant 

授权过程就是获取一个access token的过程。
在上面的流程图里，前四步包括获取authorization grant和access token. 而authorization grant type 
依赖于应用申请授权的方法和服务器提供的grant types。OAuth2 定义了4种，针对不同的场合：

- **Authorization Code** 运行在服务端的应用（server-side Applications）
- **Implicit**  移动应用(Mobile Apps)和Web应用(Web Applications)。
- **Resource Owner Password Credentials** 用于可信任应用，比如服务自己的应用。
- **Client Credentials** 应用API请求 application access

### 4.1 Grant Type: Authorization Code

这是最常见的grant type，因为它针对server-side applications做了优化。这种场合下服务器源码不公开，所以可以保证Client Secret的保密性。

This is a redirection-based flow, which means that the application must be capable of interacting with the user-agent (i.e. the user's web browser) and receiving API authorization codes that are routed through the user-agent.

authorization code流程如下图所示:

![](imgs/auth_code_flow.png)

#### 4.1.1 Step 1: Authorization Code Link

首先给用户一个下面这样的 authorization code link 

```
https://cloud.digitalocean.com/v1/oauth/authorize?
  response_type=code&
  client_id=CLIENT_ID&
  redirect_uri=CALLBACK_URL&
  scope=read
```
- **https://cloud.digitalocean.com/v1/oauth/authorize**: API authorization endpoint
- **response_type=code**: 表示应用希望获取一个authorization code grant
- **client_id=client_id**: 应用的client ID (API用来确认这个应用是谁)
- **redirect_uri=CALLBACK_URL**: where the service redirects the user-agent after an authorization code is granted
- **scope=read**: 应用请求访问的范围

#### 4.1.2 Step 2: User Authorizes Application

当用户点击上面的链接后，他必须先登录服务（或者已处于登录状态），以验证自己的身份。然后，他会看到服务给出的提示界面，用以授权或者拒绝。

![](imgs/authcode.png)

上面这个提示里，*Thedropletbook App*这个应用正在申请*manicas@digitalocean.com*这个账号的**read**授权。

#### 4.1.3 Step 3: Application Receives Authorization Code

如果用户点击"Authorize Application", 则服务会将user-agent重定向到application registration时给出的 redirect URI，并附上一个authorization code。
重定向有点类似如下的链接（假设应用是"dropletbook.com"）

```
https://dropletbook.com/callback?code=AUTHORIZATION_CODE
```

#### 4.1.4 Step 4: Application Requests Access Token

应用向API token endpoint发送authorization code以及其他authentication details（包括client secret）以请求access token。

下面是一个发向 DigitalOcean's token endpoint的POST例子:

```
https://cloud.digitalocean.com/v1/oauth/token?
  client_id=CLIENT_ID&
  client_secret=CLIENT_SECRET&
  grant_type=authorization_code&
  code=AUTHORIZATION_CODE&
  redirect_uri=CALLBACK_URL
```

- **grant_type=authorization_code** - 这个流程的grant type是authorization_code
- **code=AUTHORIZATION_CODE** - 你从query string获得的authorization code
- **redirect_uri=CALLBACK_URL** - 必须与开始的那个redirect URI一致
- **client_id=CLIENT_ID** - 第一次创建应用时获取的client ID
- **client_secret=CLIENT_SECRET** - 因为这个request是从服务端发起的，所以可以带上secret

#### 4.1.5 Step 5: Application Receives Access Token

如果authorization有效，则API则会向应用发送一个包括access token的response。完整的response类似下面的示例:

```
{
  "access_token":"ACCESS_TOKEN",
  "token_type":"bearer",
  "expires_in":2592000,
  "refresh_token":"RsT5OjbzRn430zqMLgV3Ia",
  "scope":"read",
  "uid":100101,
  "info":{
    "name":"Mark E. Mark",
    "email":"mark@thefunkybunch.com"
  }
}
```
此时，应用则被授权。它可以使用这个token访问在API访问被授权的用户账号，知道token过期或者重置。

如果出现错误，则会返回一个错误信息

```
{
  "error":"invalid_request"
}
```


### 4.2 Grant Type: Implicit

这种类型主要用于mobile apps和web applications, 在这种场合下client secret的安全性不能够被保证，所以不使用client secrect。

这种类型是一个基于重定向的流程，access token被给予user-agent来传递给应用，所以统一设备的其他用户或者应用可能会获取这一信息。
另外， 这个流程不验证应用的identity，而是依赖于注册到服务的redirect URI来完成这一目的.

这种类型不支持refresh tokens.

这类流程基本如下所示：用户被要求授权应用，然后authorization server将access token发送给user-agent，它再传给应用。

![](imgs/implicit_flow.png)

#### 4.2.1 Step 1: Implicit Authorization Link

首先给用户一个下面这样的authorization link，用以请求API的token。这个链接和authorization code link类似，
只不过它是请求一个token而不是code(注意response type是"token"):

```
https://cloud.digitalocean.com/v1/oauth/authorize?
  response_type=token&
  client_id=CLIENT_ID&
  redirect_uri=CALLBACK_URL&
  scope=read
```

#### 4.2.2 Step 2: User Authorizes Application

当用户点击link时，他必须先登录服务来验证自己。然后他会看到服务给出的提示界面，用以授权或者拒绝。

![](imgs/authcode.png)

上面这个提示里，*Thedropletbook App*这个应用正在申请*manicas@digitalocean.com*这个账号的**read**授权。

#### 4.2.3 Step 3: User-agent Receives Access Token with Redirect URI

如果用户点击"Authorize Application",服务则重定向user-agent到application redirect URI, 并在URI里面包含access token。
比如

```
https://dropletbook.com/callback#token=ACCESS_TOKEN
```

#### 4.2.4 Step 4: User-agent Follows the Redirect URI

user-agent访问这个redirect URI并保持access token.

#### 4.2.5 Step 5: Application Sends Access Token Extraction Script

应用返回一个页面，包含一个脚本，可以从完整redirect URI里提取access token.

#### 4.2.6 Step 6: Access Token Passed to Application

user-agent执行脚本，并将access token传给应用.

最终应用就被授权了。他可以使用token在API里访问用户账号，直到过期或重置。


### 4.3 Grant Type: Resource Owner Password Credentials


用户直接向应用提供用户名和密码，进而向服务获取access token。
在可以使用其他流程时尽量不要使用这个流程，由于这种方式需要采集用户密码，所以只能用于服务自身创建的应用程序

用户将credentials提供给应用后，应用向authorization server申请access token，这个POST请求如下所示：

```
https://oauth.example.com/token?
  grant_type=password&
  username=USERNAME&
  password=PASSWORD&
  client_id=CLIENT_ID
```

如果user credentials有效, 则authorization server向应用返回一个access token，并完成授权。

注意，这里并不使用client secret，因为多数情况下是移动或桌面应用，secrect并不会被保护。

### 4.4 Grant Type: Client Credentials

有些场合，应用需要获得一个自己使用的access token，而不是任何具体用户的，OAuth针对这种情况提供了client_credentials类型的授权。

应用只需要向服务发送它的credentials,包括client ID和client secret。 一个示例请求如下:

```
https://oauth.example.com/token?
  grant_type=client_credentials&
  client_id=CLIENT_ID&
  client_secret=CLIENT_SECRET
```

如果验证成功就会返回一个token。


## 5. Example Access Token Usage

所有的授权类型的结果都是获得一个access token，这时便可以使用这个token访问API。

使用curl进行访问示例如下，注意它包含了access token：

```
curl -X POST -H "Authorization: Bearer ACCESS_TOKEN" "https://api.digitalocean.com/v2/$OBJECT" 
```
如果access token有效，再回返回相应结果。如果token失效或者非法，则会返回"invalid_request"错误。

## 6. Refresh Token Flow

使用过期的token访问API会获得 "Invalid Token Error"。

At this point, if a refresh token was included when the original access token was issued, 
it can be used to request a fresh access token from the authorization server.

Here is an example POST request, using a refresh token to obtain a new access token:

```
https://cloud.digitalocean.com/v1/oauth/token?
  grant_type=refresh_token&
  client_id=CLIENT_ID&
  client_secret=CLIENT_SECRET&
  refresh_token=REFRESH_TOKEN
```

## 7. 参考资料

- [OAuth 2 Simplified](https://aaronparecki.com/oauth-2-simplified/)
- [An Introduction to OAuth 2](https://www.digitalocean.com/community/tutorials/an-introduction-to-oauth-2)
- [OAuth 2.0 by David Rice](https://www.youtube.com/playlist?list=PL675281900139F609) 使用银行的例子比较生动的图解
