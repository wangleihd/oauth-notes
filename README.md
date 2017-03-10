# OAuth 

## 目录
- 1. Roles: Applications, APIs and Users
- 2. Creating an App
- 3. Authorization: Obtaining an access token
  - 3.1 Web Server Apps
  - 3.2 Single-Page Apps
  - 3.3 Mobile Apps
  - 3.4 Other Grant Types
- 4. Making Authenticated Requests

## 1. 角色 Roles: Applications, APIs and Users

**Client**  应用程序 Applications

第三方应用程序，它需要访问用户账号。它必须提前获得许可。

**Resource Server** 资源服务器

保存用户可访问信息的API服务器。

**The Authorization Server** 授权服务器

提供用户许可或者拒绝请求的界面。

**Resource Owner** Users 资源拥有者

用户，资源所有者，他允许自己账号的部分资源被访问。

## 2. 创建应用 Creating an App

在OAuth过程开始之前，必须在服务里注册这个app。 注册新的app时，需要提供的基本信息包括：应用名称，网址，logo以及一个redirect URI来重定向用户。

**Redirect URIs** 重定向地址

服务仅会重定向用户到注册过的一个URI，以避免攻击。
任何redirect URIs必须被TLS保护，所以服务金辉重定向到`https`开头的URI。
这可以防止在授权过程中token被截取。

Native apps may register a redirect URI with a custom URL scheme for the application, which may look like demoapp://redirect.

**Client ID and Secret**

注册完应用以后，会获得一个client ID 和一个client secret。

- **client ID** 是公开信息，用来构建登录地址，或被页面里面的js代码所引用。
- **client secret** 必须保密，如果一个应用不能保证保密，比如SPA或原生应用，这时就不使用secret。 理想情况下，一开始服务就不应该将secrect发给这些应用。

## 3. 授权过程：获取一个访问token

OAuth2 得第一步是获得用户的授权，在网页或移动应用，一般是向用户显示一个服务提供的界面。

针对不同的场景，提供了几种不同的"grant types"，如下:

- **Authorization Code** for apps running on a web server, browser-based and mobile apps
- **Password** 使用用户名密码登录
- **Client credentials** for application access
- **Implicit** was previously recommended for clients without a secret, but has been superceded by using the Authorization Code grant with no secret.

每种类型详细描述如下：

### 3.1 Web服务应用 Web Server Apps

Web服务应用是使用OAuth服务时最常见的应用类型。

Web服务应用使用服务端语言编写，并运行在服务器上，源码并不可以公开获取。这意味着，应用可以使用client secret和授权服务器通信。

#### 3.1.1 授权

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
- **state** - 服务返回你传递的同样的状态字符串

你首先需要比较state值，确保和开始那个一致。 你可以将这个值保存在cookie或者session里，等用户回来时比较。这可以保证重定向端点不能陷入随意交换授权码的问题。


#### 3.1.2 交换Token

你的服务器通过auth code获取access token:

```sh
POST https://api.oauth2server.com/token
  grant_type=authorization_code&
  code=AUTH_CODE_HERE&
  redirect_uri=REDIRECT_URI&
  client_id=CLIENT_ID&
  client_secret=CLIENT_SECRET
```

- **grant_type=authorization_code** - 这个流程的grant type是authorization_code
- **code=AUTH_CODE_HERE** - 你从query string获得的auth code
- **redirect_uri=REDIRECT_URI** - 必须与开始的那个redirect URI一致
- **client_id=CLIENT_ID** - 第一次创建应用时获取的client ID
- **client_secret=CLIENT_SECRET** - 因为这个request是从服务端发起的，所以可以带上secret

服务返回一个带失效期的access token

```
{
  "access_token":"RsT5OjbzRn430zqMLgV3Ia",
  "expires_in":3600
}
```

或者一个错误信息

```
{
  "error":"invalid_request"
}
```

处于安全考虑，服务必须要求app提前注册它们的redirect URIs。

### 3.2 浏览器应用 Single-Page Apps

浏览器应用完全运行在浏览器里。由于整个代码都可在浏览器获得，并不能保证secret的安全性，所以在这里不使用secrect。
整个流程跟上面一致，不过最后一步里，用auth code交换access token时不使用client secret。


#### 3.2.1 授权

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

#### 3.2.2 交换Token

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

### 3.3 移动应用

和浏览器应用一样，移动应用也不能在保存client secret。所以移动应用也必须使用不需要client secret的OAuth流程。此外，移动应用还要考虑一些额外的工作来确保流程的安全性。

#### 3.3.1 授权

创建一个“登录”按钮，发送用户给这个服务在手机上的原生应用或者移动网页。

* 在iPhone手机里，应用可以注册一个自定义URI协议，比如"facebook://"，从而当具备这个形式URL被访问时，都会启动facebook应用。
* 在安卓手机里，应用可以注册一个特殊的URL，当这个URL被访问时，直接启动原生应用。

##### 3.3.1.1 使用服务的原生App

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

##### 3.3.1.2 使用浏览器

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

## 3.3.2 Token Exchange

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

### 3.4 密码Password


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

### 3.5 应用访问Application access

有些场合，应用需要获得一个自己使用的access token，而不是任何具体用户的，OAuth针对这种情况提供了client_credentials类型的授权。

这是只需要做如下请求即可，返回数据里会包含一个access token。

```
POST https://api.oauth2server.com/token
    grant_type=client_credentials&
    client_id=CLIENT_ID&
    client_secret=CLIENT_SECRET
```

## 4. 进行被授权请求 Making Authenticated Requests

所有的授权类型的结果都是获得一个access token，这时便可以使用这个token访问API。

使用curl进行访问示例如下：

`curl -H "Authorization: Bearer RsT5OjbzRn430zqMLgV3Ia" https://api.oauth2server.com/1/me`

确保永远使用https提交请求，https是唯一确保请求不被截取或者修改的方法。

## 词汇表

- authorization 授权
- authorization code