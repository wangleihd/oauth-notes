# OAuth 


相关术语使用英文描述，不做翻译。

## 1. OAuth Roles 角色

OAuth 定义了四个角色

**Resource Owner** User

授权*application*访问自己账号的用户，*application*访问账号被限于授权所允许的*scope*。

**Resource Server** API 或 Service

保存受保护的用户账户。

**Authorization Server** API 或 Service

提供用户许可界面，认证成功后给application发送access token。

**Client**  Applications

需要访问用户账号的应用程序。
它必须提前获得用户授权，而该授权必须被API验证通过。

## 2. Abstract Protocol Flow 一般流程

下图展示了一般流程，根据grant type的不同，细节略有不同。

![](imgs/abstract_flow.png)

1. 应用(application)向用户做申请授权(authorization request)，以访问服务资源
2. 如果用户授权这个申请，则应用(application)可以获得用户的authorization grant
3. 应用(application)向服务提供自己的application identity和用户的authorization grant，以请求access token 
4. 如果application identity和authorization grant都有效，则服务会向应用(application)发送一个access token。授权结束。
5. 应用(application)从服务请求资源，并提供access token。
6. 如果access token有效，则服务向应用提供资源。


## 3. Application Registration 应用注册 

在使用OAuth之前，必须在服务里注册应用(application)。
这个过程一般是在服务网站的注册表单(registration form)里进行的，你需要提供如下信息

- Application Name
- Application Website
- Redirect URI or Callback URL

### 3.1 Redirect URI

redirect URI是服务授权（或拒绝）application后，将用户重定向的网址。然后application就会处理authorization codes或access tokens。

服务仅会重定向到注册过的这个URI，以避免攻击。而任何redirect URIs必须被TLS保护（以`https`开头），这可以防止在授权过程中token被截取。

原生应用可能注册一个特殊的URL，比如demoapp://redirect。

### 3.2 Client ID and Secret

注册以后，服务会返回client credentials，包括一个client identifier和一个client secret。

- **client ID** 是一个公开字符串，被服务用来识别application，也用来构建authorization URLs提供给用户。
- **client secret** 当application请求访问用户账号时，在服务处验证application的identity，必须在application和服务间保密。如果application不能保证保密，比如SPA或原生应用，就不使用secret。

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

#### Step 1: Authorization Code Link

首先给用户一个下面这样的 authorization code link 

```
https://cloud.digitalocean.com/v1/oauth/authorize?response_type=code&client_id=CLIENT_ID&redirect_uri=CALLBACK_URL&scope=read
```
- **https://cloud.digitalocean.com/v1/oauth/authorize**: API authorization endpoint
- **client_id=client_id**: application的client ID (API用来确认这个application是谁)
- **redirect_uri=CALLBACK_URL**: where the service redirects the user-agent after an authorization code is granted
- **response_type=code**: 表示application希望获取一个authorization code grant
- **scope=read**: specifies the level of access that the application is requesting

- **state** - 应用程序生成的随机字符串，后面需要验证

#### Step 2: User Authorizes Application

当用户点击上面的链接后，他必须先登录服务（或者已处于登录状态），以验证自己的身份。然后，他会看到服务给出的提示界面，用以授权或者拒绝。

![](imgs/authcode.png)

上面这个提示里，*Thedropletbook App*这个application正在申请*manicas@digitalocean.com*这个账号的**read**授权。

#### Step 3: Application Receives Authorization Code

如果用户点击"Authorize Application", 则服务会将user-agent重定向到application registration时给出的 redirect URI，并附上一个authorization code。
重定向有点类似如下的链接（假设应用是"dropletbook.com"）

```
https://dropletbook.com/callback?code=AUTHORIZATION_CODE
```

#### Step 4: Application Requests Access Token

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



#### Step 5: Application Receives Access Token

如果authorization有效，则API则会向application发送一个包括access token的response。完整的response类似下面的示例:

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
此时，application则被授权。它可以使用这个token访问在API访问被授权的用户账号，知道token过期或者重置。

如果出现错误，则会返回一个错误信息

```
{
  "error":"invalid_request"
}
```

### 4.2 浏览器应用

浏览器应用完全运行在浏览器里。由于整个代码都可在浏览器获得，并不能保证secret的安全性，所以在这里不使用secrect。
整个流程跟上面一致，不过最后一步里，用auth code交换access token时不使用client secret。


#### 4.2.1 授权

创建一个“登录”链接并发送给用户

`https://oauth2server.com/auth?response_type=code&client_id=CLIENT_ID&redirect_uri=REDIRECT_URI&scope=photos&state=1234zyx`

- **code** - 表示服务希望获得一个授权码(authorization code)
- **client_id** - 第一次创建应用时获取的client ID
- **redirect_uri** - 授权结束以后返回给用户的URI
- **scope** -  一个或多个范围值，表示你希望访问用户的哪些资源
- **state** - 应用程序生成的随机字符串，后面需要验证

然后，用户看到如下提示界面：

![OAuth Authorization Prompt](imgs/oauth-authorization-prompt.png)

如果用户点击“Allow”则服务重定向用户，则服务会重定向到你的网站，并附上一个auth code

`https://oauth2client.com/cb?code=AUTH_CODE_HERE&state=1234zyx`

- **code** - 服务在查询字符串里返回授权码
- **state** -  服务返回你传递的同样的状态字符串

你首先需要比较state值，确保和开始那个一致。 你可以将这个值保存在cookie或者session里，等用户回来时比较。这可以保证重定向端点不能陷入随意交换授权码的问题。

#### 4.2.2 交换Token

```
POST https://api.oauth2server.com/token
  grant_type=authorization_code&
  code=AUTH_CODE_HERE&
  redirect_uri=REDIRECT_URI&
  client_id=CLIENT_ID
```
- **grant_type=authorization_code** - 这个流程的grant type是authorization_code
- **code=AUTH_CODE_HERE** - 你从query string获得的auth code
- **redirect_uri=REDIRECT_URI** - 必须与开始的那个redirect URI一致
- **client_id=CLIENT_ID** - 第一次创建应用时获取的client ID

### 4.3 移动应用

和浏览器应用一样，移动应用也不能在保存client secret。所以移动应用也必须使用不需要client secret的OAuth流程。此外，移动应用还要考虑一些额外的工作来确保流程的安全性。

#### 4.3.1 授权

创建一个“登录”按钮，发送用户给这个服务在手机上的原生应用或者移动网页。

* 在iPhone手机里，应用可以注册一个自定义URI协议，比如"facebook://"，从而当具备这个形式URL被访问时，都会启动facebook应用。
* 在安卓手机里，应用可以注册一个特殊的URL，当这个URL被访问时，直接启动原生应用。

##### 4.3.1.1 使用服务的原生App

如果用户安装了Facebook的原生应用，将它定向到下述URL

`fbauth2://authorize?response_type=code&client_id=CLIENT_ID&redirect_uri=REDIRECT_URI&scope=email&state=1234zyx`

- **response_type=code** -  说明你的服务希望获得一个authorization code
- **client_id=CLIENT_ID** -  第一次创建应用时获取的client ID
- **redirect_uri=REDIRECT_URI** - 授权完成以后用户需要访问的URI, 比如fb00000000://authorize
- **scope=email** -  一个或多个范围值，表示你希望访问用户的哪些资源
- **state*=1234zyx** - 应用程序生成的随机字符串，后面需要验证

For servers that support the PKCE extension (and if you're building a server, you should support the PKCE extension), you'll also include the following parameters. First, create a "code verifier" which is a random string that the app stores locally.

- **code_challenge=XXXXXXX** - This is a base64-encoded version of the sha256 hash of the code verifier string
- **code_challenge_method=S256** - Indicates the hashing method used to compute the challenge, in this case, sha256.
Note that your redirect URI will probably look like fb00000000://authorize where the protocol is a custom URL scheme that your app has registered with the OS.

##### 4.3.1.2 使用浏览器

If the service does not have a native application, you can launch a mobile browser to the standard web authorization URL. Note that you should never use an embedded web view in your own application, as this provides the user no guarantee that they are actually are entering their password in the service's website rather than a phishing site.

You should either launch the native mobile browser, or use the new iOS "SafariViewController" to launch an embedded browser in your application. This API was added in iOS 9, and provides a mechanism to launch a browser inside the application that both shows the address bar so the user can confirm they're on the correct website, and also shares cookies with the real Safari browser. It also prevents the application from inspecting and modifying the contents of the browser, so can be considered secure.

`https://facebook.com/dialog/oauth?response_type=code&client_id=CLIENT_ID&redirect_uri=REDIRECT_URI&scope=email&state=1234zyx`

Again, if the service supports PKCE, then those parameters should be included as well as described above.

- **response_type=code** - indicates that your server expects to receive an authorization code
- **client_id=CLIENT_ID** - The client ID you received when you first created the application
- **redirect_uri=REDIRECT_URI** - Indicates the URI to return the user to after authorization is complete, such as fb00000000://authorize
- **scope=email** - One or more scope values indicating which parts of the user's account you wish to access
- **state=1234zyx** - A random string generated by your application, which you'll verify later
The user will see the authorization prompt

![Facebook Authorization Prompt](imgs/everyday-city-auth.png)

## 4.3.2 Token Exchange

After clicking "Approve", the user will be redirected back to your application with a URL like

`fb00000000://authorize#code=AUTHORIZATION_CODE&state=1234zyx`

Your mobile application should first verify that the state corresponds to the state that was used in the initial request, and can then exchange the authorization code for an access token.

The token exchange will look the same as exchanging the code in the web server app case, except that the secret is not sent. If the server supports PKCE, then you will need to include an additional parameter as described below.

```
POST https://api.oauth2server.com/token
  grant_type=authorization_code&
  code=AUTH_CODE_HERE&
  redirect_uri=REDIRECT_URI&
  client_id=CLIENT_ID&
  code_verifier=VERIFIER_STRING
```  

- **grant_type=authorization_code** - The grant type for this flow is authorization_code
- **code=AUTH_CODE_HERE** - This is the code you received in the query string
- **redirect_uri=REDIRECT_URI** - Must be identical to the redirect URI provided in the original link
- **client_id=CLIENT_ID** - 第一次创建应用时获取的client ID
- **code_verifier=VERIFIER_STRING** - The plaintext string that you previously hashed to create the code_challenge

The authorization server will verify this request and return an access token.

If the server supports PKCE, then the authorization server will recognize that this code was generated with a code challenge, and will hash the provided plaintext and confirm that the hashed version corresponds with the hashed string that was sent in the initial authorization request. This ensures the security of using the authorization code flow with clients that don't support a secret.

### 4.4 密码


OAuth 2 也提供密码授权类型，用户可以通过用户名和密码直接获得access token。 由于这种方式需要采集用户密码，所以只能用于服务自身创建的应用程序。比如原生的Twitter应用可以使用用户密码登录手机和桌面应用。

要使用密码方式，只需要做一个如下简单的POST 请求

```
POST https://api.oauth2server.com/token
  grant_type=password&
  username=USERNAME&
  password=PASSWORD&
  client_id=CLIENT_ID
```

- **grant_type=password** - 授权类型是密码
- **username=USERNAME** - 用户名
- **password=PASSWORD** - 密码
- **client_id=CLIENT_ID** - 第一次创建应用时获取的client ID

服务器则会和其他类型一样返回access token。

注意，这里并不使用client secret，因为多数情况下是移动或桌面应用，secrect并不会被保护。

### 4.5 应用访问

有些场合，应用需要获得一个自己使用的access token，而不是任何具体用户的，OAuth针对这种情况提供了client_credentials类型的授权。

这是只需要做如下请求即可，返回数据里会包含一个access token。

```
POST https://api.oauth2server.com/token
    grant_type=client_credentials&
    client_id=CLIENT_ID&
    client_secret=CLIENT_SECRET
```

## 5. 请求服务

所有的授权类型的结果都是获得一个access token，这时便可以使用这个token访问API。

使用curl进行访问示例如下：

`curl -H "Authorization: Bearer RsT5OjbzRn430zqMLgV3Ia" https://api.oauth2server.com/1/me`

确保永远使用https提交请求，https是唯一确保请求不被截取或者修改的方法。

## 参考资料

- [OAuth 2 Simplified](https://aaronparecki.com/oauth-2-simplified/)
- [An Introduction to OAuth 2](https://www.digitalocean.com/community/tutorials/an-introduction-to-oauth-2)