# 本地安装的应用程序中使用 OAuth 2.0

谷歌 OAuth 2.0 端点支持各种平台上的应用程序，例如计算机、移动电话、平板等。本地安装的应用程序会被分发到每个独立的机器上，而且通常被认为不能用于保存 secret。这些应用程序可以在用户值守时，或者在后台运行时访问谷歌 API。

如果您满足下述条件，那么这个文档就是为您准备的：

- 您正在编写一个本地安装的应用程序，且该程序的平台不是 Android 或 iOS，同时

- 您的本地安装的应用程序会运行在拥有良好输入性能并且系统具有浏览器的设备上，例如拥有全键盘的设备。

如果你正在 Android 或 iOS 编写应用程序，请使用[谷歌登陆](https://developers.google.com/identity/sign-in/)来认证您的用户。谷歌登陆按钮会管理 OAuth 2.0 工作流，包括认证和获取谷歌 API 授权。所有有谷歌账户的人都能用谷歌登陆，不论他们有没有升级到 Google+。若要添加谷歌登陆按钮，请根据平台分别参照 [Android](https://developers.google.com/identity/sign-in/android/) 或 [iOS](https://developers.google.com/identity/sign-in/ios/) 或[网站](https://developers.google.com/identity/sign-in/web/)。

如果您的应用程序运行在没有浏览器的设备上，或者输入性能受限（例如运行在家用游戏机、视频摄录机、打印机上的应用程序），那么请参照[在设备上使用 OAuth 2.0](https://developers.google.com/identity/protocols/OAuth2ForDevices)。

![](images/device-auth-decision-installedapp.png)

## 概览

在这个工作流中，在本地安装的应用程序必须能使用浏览器，或者有嵌入式浏览器作为 Web 视图。在本地安装的应用程序上的 OAuth 2.0 工作流如下：

1. 您的应用程序将浏览器重定向到一个谷歌 URL。URL 查询参数会指出应用程序所需要的谷歌 API 访问权限。

2. 和其他方案一样，谷歌会负责用户认证和用户准许（User consent），这一系列程序的结果是一个授权码。授权码会返回到浏览器的标题栏上或者作为一个查询串参数被返回，这取决于您的应用程序在请求中所发送的参数。
3. 您的应用程序会进行授权码交换来获得访问令牌和刷新令牌。在交换过程中，应用程序会提交他的客户端 ID 和客户端 secret（这两者都是从开发者控制台获得的）。
4. 您的应用程序会使用访问令牌来对谷歌 API 进行调用，应用程序也会储存刷新令牌以供未来使用。

![](images/webflow.png)

这个工作流和[在 Web 服务器应用程序上使用 OAuth 2.0](https://developers.google.com/identity/protocols/OAuth2WebServer)的工作流比较类似，但有以下三点不同：

- 当创建一个客户端 ID时，你需要指明您的应用程序是一个在本地安装的应用程序。这会影响 redirect_uri 参数的值的结果。
- 从开发者控制台获取的客户端 ID 和客户端 secret 是嵌入在您的应用程序源代码内的。在这种情况下，客户端 secret 显然没被当做秘密看待。 
- 授权码可以通过浏览器的标题栏，或者 http://localhost 端口的查询串来返回到您的应用程序。

## 为认证请求生成 URL

用于认证用户的 URL 是 [https://accounts.google.com/o/oauth2/auth](https://accounts.google.com/o/oauth2/auth) 。这个端点只能通过 SSL 访问，HTTP（非-SSL）连接会被拒绝。

--

**端点：**https://accounts.google.com/o/oauth2/auth  
**描述：**这个端点是初次请求的目标。它负责处理寻找活动会话，认证用户和用户准许。当您的应用程序对该端点发送请求时，回应会包含一个访问令牌，一个刷新令牌，和一个授权码。

--

对于在本地安装的应用程序，谷歌认证服务器支持的查询串参数集为：

**参数：**	response_type （响应类型）  
**值：**	 code （密码）  
**描述：**	决定 Google OAuth 2.0 端点是否要返回授权码。 对于在本地安装的应用程序，参数应该使用 code。

--

**参数：**	client_id （客户端 ID）  
**值：**	你从[开发者控制台](https://console.developers.google.com/)处获得的客户端 ID。  
**描述：**	确定是哪个客户端正在发出请求。传过去的这个参数值必须要与[开发者控制台](https://console.developers.google.com/)里显示的完全一致。  

--

**参数：**	redirect_uri （重定向 URI）  
**值：**	 在[开发者控制台](https://console.developers.google.com/)里列出的这个工程的 redirect_urivalues 其中一个值。  
**描述：**	决定回应(Response)会发向哪里。这个参数的值必须和[谷歌开发者控制台](https://console.developers.google.com/)为这个工程所显示的值的其中一个完全一致（包括完整的 HTTP 或 HTTPS 格式、大小写、和末尾的'/'符号）。您能使用 urn:ietf:wg:oauth:2.0:oob, urn:ietf:wg:oauth:2.0:oob:auto, 或者一个 http://localhost 端口。更多详情，请参照[选择一个重定向 URI](https://developers.google.com/identity/protocols/OAuth2InstalledApp#choosingredirecturi)。

-- 

**参数：**	scope （域）   
**值：**	用空格分隔的该应用程序所请求的权限集。  
**描述：**	确认您的应用程序请求的谷歌 API 访问权。传过去的参数值会以用户准许页面的方式向用户显示。请求的权限数量和获得用户准许的可能性有逆相关关系。若想了解可用的登陆域，请参见[登陆域](https://developers.google.com/+/api/oauth#login-scopes)。若想了解所有谷歌 API 的可用域，请访问 [API 浏览器](https://developers.google.com/apis-explorer/#p/)。基本上渐进地请求域是一项最佳实践，比起提前一次性请求所有权限，按需请求更好。举例来说，一个想要支持购买功能的应用程序应该在用户点击“购买”按钮的时候才要求谷歌钱包访问权; 详情参阅[渐进式授权](https://developers.google.com/accounts/docs/OAuth2WebServer#incrementalAuth)。

--
 
**参数：**	state （状态）  
**值：**	任意字符串  
**描述：**	当收到回应时提供任何可能对您的应用程序有用的状态。谷歌认证服务器会回传这个参数，所以您的应用程序会收到和它发出去一样的内容。若想防止跨站请求伪造攻击([CSRF](http://en.wikipedia.org/wiki/Cross-site_request_forgery)), 我们强烈推荐您在状态中包含防伪造令牌，并且在回应中进行确认。详情请参见[OpenID 连接](https://developers.google.com/identity/protocols/OpenIDConnect#createxsrftoken) 来获得实现这个功能的示例。

--
 
**参数：**	login_hint （登陆提示）  
**值：**	电子邮箱地址 或 子标识符  
**描述：**	当您的应用程序知道哪个用户正在尝试进行认证时，可以提供这个参数作为对认证服务器的提示。将这个参数传过去会在登陆页面自动填写用户邮箱地址，或者选择适合的多用户登录会话，从而简化登陆工作流。
 
--

**参数：**	include_granted_scopes （包含已许可的域）  
**值：**	true 或 false  
**描述：**	如果这个参数值被设为 true，同时认证请求被许可，那么该次认证会许可所有 曾经许可给这组用户和应用程序的其他域；请参见[渐进式授权](https://developers.google.com/accounts/docs/OAuth2WebServer#incrementalAuth)。
 

--

### 示例 URL

下面是一个示例 URL，包含换行和空格来加强可读性：

	https://accounts.google.com/o/oauth2/auth?
	  scope=email%20profile&
	  redirect_uri=urn:ietf:wg:oauth:2.0:oob&
	  response_type=code&
	  client_id=812741506391-h38jh0j4fv0ce1krdkiq0hfvt6n5amrf.apps.googleusercontent.com

如果用户通过类似上述的 URL 进行登陆并且批准了访问权限，那么其结果会是类似下面这样的对话框：

![](images/installedresult.png)


下面是另外一个示例 URL，包含换行和空格来加强可读性：

	https://accounts.google.com/o/oauth2/auth?
	  scope=email%20profile&
	  redirect_uri=http://localhost:9004&
	  response_type=code&
	  client_id=812741506391-h38jh0j4fv0ce1krdkiq0hfvt6n5amrf.apps.googleusercontent.com

这两个 URL 的区别只有 `redirect_uri` 参数的值。第一个会让回应通过页面的标题返回授权码，而第二个则会使授权码作为查询串的一部分返回到 `http://localhost` 地址。

### 选择一个重定向 URI

当您在[谷歌开发者控制台](https://console.developers.google.com/)中创建一个客户端 ID 时，控制台会为您创建两个 **redirect_uri** （重定向 URI）：

**urn:ietf:wg:oauth:2.0:oob** 和 **http://localhost**

您的应用程序所使用的参数值会决定授权码以何种方式传回。

#### http://localhost

这个参数值会告知谷歌认证服务器授权码应该以查询串参数的形式返回到客户端的 Web 服务器上。您可以在不需要变更[谷歌开发者控制台](https://console.developers.google.com/)配置的情况下指定端口号。如果要使用这个 URL 接收授权码，您的应用程序必须监听本地 Web 服务器。这种方案在许多平台上可行，但不是所有平台都可行。如果您的平台支持这种方式，这是推荐的获取授权码的方式。

> **注意**：在一些情况下，尽管可以进行监听，但其他软件（例如 Windows 防火墙）如果没有作出显著配置可能会对通讯照成影响。

#### urn:ietf:wg:oauth:2.0:oob

这个参数值会告知谷歌认证服务器应该用浏览器的标题栏将授权码返回到客户端，并且还带有一个文本页面提示用户将密码复制并粘贴到应用程序中（就像上面的截图一样）。这在当客户端（例如一个 Windows 应用程序）在系统不作出显著配置的前提下无法监听 HTTP 端口时十分有用。

当您使用这个参数值，您的应用程序可以检测到页面已经载入，并且可以读取 HTML 页面的标题来获得授权码。接下来就要靠您的应用程序来关闭浏览器窗口了，如果您想确保用户永远都不会看见包含授权码的页面的话，其实现方式根据平台的不同而不同。

如果您的目标平台不允许您检测页面是否已经载入，或者不允许您检测页面的标题，那么您可以要求用户手动将密码复制粘贴到您的应用程序内，就和页面本身的文字说明一样。

#### urn:ietf:wg:oauth:2.0:oob:auto

该参数和 `urn:ietf:wg:oauth:2.0:oob` 是一致的，但返回的确认页面不会有文字提示用户手动复制授权码，取而代之的是要求用户关闭浏览器窗口。

在您的应用程序能读取 HTML 标题（例如，通过在桌面上检查窗口标题）获得授权码，但是却没法自动关闭页面的时候，这种方法就显得有用。

## 处理回应并发出令牌请求

初次认证请求的回应会包含一个授权码（密码），您的应用程序可以用授权码交换获得一个访问令牌和一个刷新令牌。

若要发出这种令牌请求，您的应用程序应发送一个 HTTP `POST` 请求到 `/oauth2/v3/token`。并且包含以下参数：

**字段：**	code  
**描述：**	初次认证请求返回的授权码

--

**字段：**	client_id （客户端 ID）  
**描述：**	从[开发者控制台](https://console.developers.google.com/)处获得的客户端 ID。

--

**字段：**	client_secret （客户端 secret）  
**描述：**	从[开发者控制台](https://console.developers.google.com/)处获得的客户端 secret。（对于注册为 Android，iOS 或 Chrome 应用程序的客户端而言，这是可选项。）

--

**字段：**	redirect_uri （重定向 URI）  
**描述：**	从[开发者控制台](https://console.developers.google.com/)处获得的重定向 URI。

--

**字段：**	grant_type （许可类型）  
**描述：**	正如 OAuth 2.0 规格中定义的，这个字段必须包含 authorization_code 中的一个值。

--

一个实际的请求应该类似下例：

	POST /oauth2/v3/token HTTP/1.1
	Host: www.googleapis.com
	Content-Type: application/x-www-form-urlencoded

	code=4/v6xr77ewYqhvHSyW6UJ1w7jKwAzu&
	client_id=8819981768.apps.googleusercontent.com&
	client_secret=

	your_client_secret&
	redirect_uri=https://oauth2-login-demo.appspot.com/code&
	grant_type=authorization_code

	
针对上述请求的成功回应会包含下列字段：

**字段：**	access_token （访问令牌）  
**描述：**	可以发送给谷歌 API 的令牌。

--

**字段：**	refresh_token （刷新令牌）  
**描述：**	用于获得新的访问令牌的令牌，对于本地安装的应用程序，该令牌默认已经包含，刷新令牌会一直有效直到用户对其废除访问权。

--

**字段：**	expires_in （有效期）  
**描述：**	访问令牌剩余的生命期。

--

**字段：**	token_type （令牌类型）  
**描述：**	确认返回的令牌类型。目前，这个字段总是会拥有值 Bearer。

--

> **注意**: 有时回应可能会包含其他字段，您的程序不应该将这种情况视为错误。上面示例中的集合是最小集。

一个成功的回应会以 JSON 数组的形式发送回来，和下面的例子类似：

	{
	  "access_token":"1/fFAGRNJru1FTz70BzhT3Zg",
	  "expires_in":3920,
	  "token_type":"Bearer",
	  "refresh_token":"1/xEoDL4iW3cxlI7yDbSRFYNG01kVKM2C-259HOF2aQbI"
	}

## 调用谷歌 API

您的程序获得访问令牌之后，您可以使用令牌以用户或者服务账户的名义来对谷歌 API 进行调用。要做到这一点，请将访问令牌包含到发给 API 的请求中，可以通过包含 `access_token` 查询参数或者一个 `Authorization: Bearer` HTTP 头来实现。如果可能的话，我们更欢迎 HTTP 头的方法，因为查询串更容易在服务器记录中可见。在大多数情况下你可以使用客户端库来设置您的谷歌 API 调用（例如，当对[谷歌人脉 API 进行调用时](https://developers.google.com/+/api/latest/people/get#examples)）。

你可以在[OAuth 2.0 游乐场](https://developers.google.com/oauthplayground/)中自行尝试所有的谷歌 API 并查看他们所对应的域。

#### 示例

一个通过使用 access_token 查询串参数对 [people.get](https://developers.google.com/+/api/latest/people/get) 端点 （谷歌人脉 API）的调用会和下例相类似，当然在实际情况中您需要提供您自己的访问令牌：

	GET https://www.googleapis.com/plus/v1/people/userId?access_token=1/fFBGRNJru1FQd44AzqT3Zg

而对于已经认证的用户（me）而言，通过 `Authorization: Bearer` HTTP 头调用同样的 API 就会像下面这样：

	GET /plus/v1/people/me HTTP/1.1
	Authorization: Bearer 1/fFBGRNJru1FQd44AzqT3Zg
	Host: googleapis.com

您可以尝试 `curl` 命令行应用程序。下面是使用 HTTP 头的方法（推荐）的示例：

	curl -H "Authorization: Bearer 1/fFBGRNJru1FQd44AzqT3Zg" https://www.googleapis.com/plus/v1/people/me

或者，您也可以使用查询串参数的方法实现：

	curl https://www.googleapis.com/plus/v1/people/me?access_token=1/fFBGRNJru1FQd44AzqT3Zg

## 渐进式授权

在 OAuth 协议中，您的应用程序为了访问资源所请求的认证是以域来区分的，如果用户已经被认证而且同意权限请求，您的应用程序会接收到一个生命期比较短的访问令牌，这些令牌能让它访问目标资源，而刷新令牌（可选）可以让应用程序拥有长期的访问权限。

按需请求资源访问权限通常被认为是一项最佳实践。举例来说，一个让人们可以对音乐进行采样并且创建混音的应用程序也许在登陆时候只需要非常少量的资源，说不定除了登陆者的名字之外就没别的了。但是，要保存一个完整的混音可能会需要访问他们的谷歌网盘。大多数人都会觉得在应用程序需要存储文件时申请谷歌网盘的访问权限是非常自然的。

这种情况下，在登录时应用程序可以请求这个域：`https://www.googleapis.com/auth/plus.loginto` 来实现一个基本的社交登陆功能，然后在需要保存混音的时候才申请这个域：`https://www.googleapis.com/auth/drive.file`。

同时使用在[使用 OpenID 连接](https://developers.google.com/identity/protocols/OpenIDConnect)和[使用 OAuth 2.0 来访问谷歌 API](https://developers.google.com/accounts/docs/OAuth2) 里描述的步骤通常会让你的应用程序不得不管理两个不同的访问令牌。如果您希望避开这种复杂性，则需要在所有 OAuth 2.0 工作流的第一步时，将发送给 `https://accounts.google.com/o/oauth2/auth` 的认证 URI 里添加一个额外的参数。这个参数是 `include_granted_scopes`，其中值可以被赋为 `true` 或 `false`（默认值是 `false`），当这个值为 `true` 时，如果您的域认证要求被批准，谷歌认证服务器会将这次认证和所有之前已经成功的认证为这组用户-应用程序合并。这类请求的 URI 可能看上去会像下面这样（下例有插入换行和空格来增强可读性）：

	https://accounts.google.com/o/oauth2/auth?
	  scope=https://www.googleapis.com/auth/drive.file&
	  state=security_token%3D138r5719ru3e1%26url%3Dhttps://oa2cb.example.com/myHome&
	  redirect_uri=https%3A%2F%2Fmyapp.example.com%2Fcallback&
	  response_type=code&
	  client_id=8127352506391.apps.googleusercontent.com&
	  approval_prompt=force&
	  include_granted_scopes=true

我们现在把上面这种认证叫做“组合认证”；关于组合认证：

- 您可以使用刚才得到的访问令牌来访问任何在组合认证中已经合并的域。

- 当您使用刷新令牌来进行组合认证时，返回的新访问令牌也代表了组合认证，所以也可以用于访问其任意域。
- 组合认证包括了所有过去已经被许可的权限，即使这些权限请求曾经是从不同的客户端发出的。举例来说，如果您在桌面应用程序请求了
`https://www.googleapis.com/auth/plus.loginscope` ，然后又向同一个用户的移动端发送了同样的请求，那么后面这个请求会被自动批准，因为组合认证会包含两边的域。

- 当您废除了一个代表组合认证的的令牌，其所有的认证都会被同时废除；这意味着如果你还保留着任何以往的权限的令牌，他们也会跟着失效。

### 使用刷新令牌

若要获取访问令牌，您的应用程序应该发送一个 HTTPS `POST` 请求到 `https://www.googleapis.com/oauth2/v3/token`。
并且请求必须包含以下参数：

**字段：**	refresh_token （刷新令牌）  
**描述：**	从授权码交换返回的刷新令牌
 
--

**字段：**	client_id （客户端 ID）  
**描述：**	从[开发者控制台](https://console.developers.google.com/)处获得的客户端 ID。

--

**字段：**	client_secret （客户端 secret）  
**描述：**	从[开发者控制台](https://console.developers.google.com/)处获得的客户端 secret。

--

**字段：**	grant_type （许可类型）  
**描述：**	正如 OAuth 2.0 规格中定义的，这个字段必须包含 authorization_code 中的一个值。

--

这种请求如下例所示：

	POST /oauth2/v3/token HTTP/1.1
	Host: www.googleapis.com
	Content-Type: application/x-www-form-urlencoded

	client_id=8819981768.apps.googleusercontent.com&
	client_secret={client_secret}&
	refresh_token=1/6BMfW9j53gdGImsiyUH5kU5RsR4zwI9lUVX-tqf8JXQ&
	grant_type=refresh_token

只要用户没有废除您的应用程序的访问权限，回应中就会包括一个新的访问令牌。对于上述请求，回应应该和下例相似：

	{
	  "access_token":"1/fFBGRNJru1FQd44AzqT3Zg",
	  "expires_in":3920,
	  "token_type":"Bearer",
	}

请注意刷新令牌的发放数量是有限制的；限制的单位是每一组客户端-用户，你应该在长期存储器中保存刷新令牌并一直使用它直到过期。如果您的应用程序请求太多刷新令牌，就可能会触发限制，这种情况下最旧的令牌会失效。

## 客户端库

下面的客户端库整合了一些流行的框架，让您更简单地应用 OAuth 2.0。随着时间的推移，会有更多的新功能添加到这些库中。

- [谷歌 API 客户端库 for Java](https://developers.google.com/api-client-library/java/google-api-java-client/oauth2)
- [谷歌 API 客户端库 for Python](https://developers.google.com/api-client-library/python/guide/aaa_oauth)
- [谷歌 API 客户端库 for .NET](https://developers.google.com/api-client-library/dotnet/guide/aaa_oauth)
- [谷歌 API 客户端库 for Ruby](https://developers.google.com/api-client-library/ruby/guide/aaa_oauth)
- [谷歌 API 客户端库 for PHP](https://developers.google.com/api-client-library/php/guide/aaa_oauth2_web)
- [谷歌 API 客户端库 for JavaScript](https://developers.google.com/api-client-library/javascript/features/authentication)
- [谷歌工具箱 for Mac OAuth 2.0 控制器](https://code.google.com/p/gtm-oauth2/)