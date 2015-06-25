# 在设备上使用 OAuth 2.0

**目录**

- 概览
- 获取用户码
- 显示用户码
- 获取访问令牌和刷新令牌
- 调用谷歌 API
- 使用刷新令牌
- 允许的域

如果您：

正在编写一个在本地安装的应用程序，并且该程序不在 Android 或 iOS 上运行。
您的应用程序会运行在一个没有系统浏览器、或者输入能力有限的设备上，例如家用游戏机、摄像机、打印机等。
您的应用仅需要访问[这种工作流所允许的域](https://developers.google.com/identity/protocols/OAuth2ForDevices#allowedscopes)。

如果您正在为 Android 或 iOS 编写应用程序，请使用[谷歌登陆](https://developers.google.com/identity/sign-in/)来认证您的用户。
谷歌登陆按钮会管理 OAuth 2.0 工作流，包括认证和获取谷歌 API 授权。所有有谷歌账户的人都能用谷歌登陆，不论他们有没有升级到谷歌+。若要添加谷歌登陆按钮，请根据平台分别参照 [Android](https://developers.google.com/identity/sign-in/android/) 或 [iOS](https://developers.google.com/identity/sign-in/ios/)。

如果您正在为其他平台开发程序，并且应用程序会运行在拥有系统浏览器和强输入能力的设备（例如拥有键盘的设备）上，那么请参照[在本地安装的应用程序上使用 OAuth 2.0](https://developers.google.com/identity/protocols/OAuth2InstalledApp)。

![](images/device-auth-decision-deviceflow.png)

## 概览

运行在输入能力有限的设备（例如家用游戏机、摄像机、打印机等）上的应用程序可以以用户名义访问谷歌 API，不过用户必须通过电脑或者强输入能力的设备来完成剩余操作。工作流如下：

- 您的应用程序发送一个带参数的请求到谷歌 URL 来开始这一工作流。回应会包含一个设备码、一个用户码、一个 URL、一个有效期、和一个建议轮询间隔时间。
- 接收到上述回应之后，您的应用程序将 URL 和用户码展示给用户，并且指引用户打开浏览器，导航到该 URL，并输入代码。
- 用户这时切换到一个拥有更强输入能力的设备或电脑，启动浏览器，导航到先前设备指引的 URL，登陆账户，然后输入代码。
- 在后台，您的应用程序会轮询一个谷歌端点来获取访问令牌和刷新令牌。这些令牌只会在用户登陆和许可请求之后才会被返回到您的应用程序。

使用这种工作流，您只能获取[有限的域集](https://developers.google.com/identity/protocols/OAuth2ForDevices#allowedscopes)。

![](images/deviceflow.png)

考虑到这些设备的能力多样性和运行环境多样性，本文档的示例会使用 `curl` 命令行工具。因为将 `curl` 命令移植到不同的编程语言和运行环境通常都是简单琐碎的。

## 获取用户码

和所有 OAuth 2.0 方案一样，你首先需要在[谷歌开发者控制台](https://console.developers.google.com/)建立一个工程并且获取一个客户端 ID 和客户端 secret。

获得客户端 ID 和客户端 secret 之后，请发送一个 HTTP `POST` 给 OAuth 2.0 设备端点：`https://accounts.google.com/o/oauth2/device/code` 请求中请包含你的 `client_id`（客户端 ID）和域的列表。
和其他 OAuth 2.0 工作流不一样的是，`response_type`（回应种类）和 `redirect_uri`（重定向 URI）在设备工作流中式不需要的。

下面是请求用户码的一个示例：

	POST /o/oauth2/device/code HTTP/1.1
	Host: accounts.google.com
	Content-Type: application/x-www-form-urlencoded

	client_id=812741506391-h38jh0j4fv0ce1krdkiq0hfvt6n5amrf.apps.googleusercontent.com&
	scope=email%20profile

或者你也可以使用 `curl`:

	curl -d "client_id=812741506391-h38jh0j4fv0ce1krdkiq0hfvt6n5amrf.apps.googleusercontent.com&scope=email profile" https://accounts.google.com/o/oauth2/device/code

回应会以 JSON 对象的形式返回：

	{
	  "device_code" : "4/4-GMMhmHCXhWEzkobqIHGG_EnNYYsAkukHspeYUk9E8",
	  "user_code" : "GQVQ-JKEC",
	  "verification_url" : "https://www.google.com/device",
	  "expires_in" : 1800,
	  "interval" : 5
	}

JSON 对象内的 `user_code`（用户码）和 `verification_url`（验证 URL）值应该被展示给用户。整个概念就是让用户自己去浏览器并导航到 `verification_url` 中的 URL，然后输入 `user_code` 里面的代码。`user_code` 是大小写敏感的，所以用户需要准确地输入回应中给出的代码。`device_code`、`expires_in` 和 `interval` 字段会在[获取访问令牌和刷新令牌](https://developers.google.com/identity/protocols/OAuth2ForDevices#obtainingatoken) 章节中使用到。

## 显示用户码

`verification_url` 和 `user_code` 的值可能会变更。所以请将您的用户界面设计成有能力应对以下限制：
Design your UI in a way that can handle the following limits:

- `user_code` 必须在一个可以显示 15 个 ‘W’ 字符宽度的区域中显示。换句话说，如果你能正确显示 `WWWWWWWWWWWWWWW`，那么您的用户界面是有效的。

- 显示 `verification_url `的区域需要有足够宽度来应对一个 40 个字符长的 URL 串。

上述两个字符串可能会包含任意可列印的 US-ASCII 字符集中的字符。不要做出任何关于字符串内容的多余的假设。特别的，不要以任何形式修改 `user_code` 字符串（例如变更大小写或者加入格式化字符），这样做可能会导致未来我们修改代码的时候连同破坏您的程序正常运作。也请不要修改 `verification_url`，除非选择性的去除 URL 方案（scheme）来优化显示，如果您打算从 URL 中去除 URL 方案（例如“http://”），请确保您的应用程序可以处理 “http” 和 “https” 两种协议。

>**备注**：为了防止您的应用程序在未来无法正常工作，请用 15 个字符长度的'W' 用户码来测试您的应用程序，并且在显示时不要修改字符串。

当用户导航到 verification_url 中的 URL 时，他们会见到类似下图的页面：

![](images/usercodedevice.png)

在用户输入 user_code 之后，他们会被要求登录谷歌，并且会看见用户准许页面（如下图所示）。

![](images/approvaldevice.png)

如果用户点击“许可权限”，那么您的应用程序就可以获得一个访问令牌和刷新令牌，这两个令牌可以让您的应用程序访问一个或多个谷歌 API。和所有 OAuth 2.0 方案一样，域（`scope`）参数会控制您的应用程序拥有哪些访问谷歌 API 的权限。


## 获取访问令牌和刷新令牌

当您的应用程序把 `user_code` 和 `verification_url` 展示给用户之后，就可以用当时和 `user_code` 和 `verification_url` 一同返回的 `device_code` 参数来轮询谷歌端点。被轮询的端点 URL 是
`https://www.googleapis.com/oauth2/v3/token` ，而每次请求之间的轮询间隔在 `interval` 参数中定义，单位为秒。

这种请求如下例所示：

	POST /oauth2/v3/token HTTP/1.1
	Host: www.googleapis.com
	Content-Type: application/x-www-form-urlencoded

	client_id=812741506391-h38jh0j4fv0ce1krdkiq0hfvt6n5amrf.apps.googleusercontent.com&
	client_secret={clientSecret}&
	code=4/4-GMMhmHCXhWEzkobqIHGG_EnNYYsAkukHspeYUk9E8&
	grant_type=http://oauth.net/grant_type/device/1.0

或者使用 curl:

	curl -d "client_id=812741506391-h38jh0j4fv0ce1krdkiq0hfvt6n5amrf.apps.googleusercontent.com&
	client_secret={clientSecret}&
	code=4/4-GMMhmHCXhWEzkobqIHGG_EnNYYsAkukHspeYUk9E8&
	grant_type=http://oauth.net/grant_type/device/1.0"
	https://www.googleapis.com/oauth2/v3/token

如果用户还没有同意请求，那么回应会像下面这样：

	{
	  "error" : "authorization_pending"
	}

那么您的应用程序应该以不快于 interval 字段值的速率重复发送请求。如果您的应用程序以过快的速度轮询，那么回应会变成下面这样：

	{
	  "error" : "slow_down"
	}

一旦用户登陆谷歌并且许可您的应用程序访问谷歌 API，那么下一次轮询请求会得到一个访问令牌和刷新令牌（如下所示）。

	{
	  "access_token" : "ya29.AHES6ZSuY8f6WFLswSv0HELP2J4cCvFSj-8GiZM0Pr6cgXU",
	  "token_type" : "Bearer",
	  "expires_in" : 3600,
	  "refresh_token" : "1/551G1yXUqgkDGnkfFk6ZbjMLMDIMxo3JFc8lY8CAR-Q"
	}

接到这个回应之后，您的应用程序就可以在谷歌 API 请求中使用访问令牌了。访问令牌的生命期是有限的。如果您的应用程序需要在长时间之后访问 API，那么可以使用刷新令牌来获得一个新的访问令牌（请参见[使用刷新令牌](https://developers.google.com/identity/protocols/OAuth2ForDevices#refreshtoken)）。
如果您的应用程序需要这种权限，那么就需要将刷新令牌保存下来以便日后使用。

## 调用谷歌 API

您的程序获得访问令牌之后，您可以使用令牌以用户或者服务账户的名义来对谷歌 API 进行调用。要做到这一点，请将访问令牌包含到发给 API 的请求中，可以通过包含 `access_token` 查询参数或者一个 `Authorization: Bearer` HTTP 头来实现。如果可能的话，我们更欢迎 HTTP 头的方法，因为查询串更容易在服务器记录中可见。在大多数情况下你可以使用客户端库来设置您的谷歌 API 调用（例如，当对[谷歌人脉 API 进行调用时](https://developers.google.com/+/api/latest/people/get#examples)）。

你可以在[OAuth 2.0 游乐场](https://developers.google.com/oauthplayground/)中自行尝试所有的谷歌 API 并查看他们所对应的域。

### 示例

一个通过使用 access_token 查询串参数对 [people.get](https://developers.google.com/+/api/latest/people/get) 端点 （谷歌人脉 API）的调用会和下例相类似，当然在实际情况中您需要提供您自己的访问令牌：

	GET https://www.googleapis.com/plus/v1/people/userId?access_token=1/fFBGRNJru1FQd44AzqT3Zg

而对于已经认证的用户（me）而言，通过 Authorization: Bearer HTTP 头调用同样的 API 就会像下面这样：

	GET /plus/v1/people/me HTTP/1.1
	Authorization: Bearer 1/fFBGRNJru1FQd44AzqT3Zg
	Host: googleapis.com

您可以尝试 `curl` 命令行应用程序。下面是使用 HTTP 头的方法（推荐）的示例：

	curl -H "Authorization: Bearer 1/fFBGRNJru1FQd44AzqT3Zg" https://www.googleapis.com/plus/v1/people/me

或者，您也可以使用查询串参数的方法实现：

	curl https://www.googleapis.com/plus/v1/people/me?access_token=1/fFBGRNJru1FQd44AzqT3Zg

## 使用刷新令牌

访问令牌终会过期。API 会通过返回 `401` 状态码来指示一个访问令牌已经过期。要获得一个新的访问令牌，发送一个请求给令牌端点，并且包含`client_id`、
`client_secret`、`refresh_token` 和 `grant_type` 参数（如下所示）。

	POST /oauth2/v3/token HTTP/1.1
	Host: www.googleapis.com
	Content-Type: application/x-www-form-urlencoded

	client_id=812741506391-h38jh0j4fv0ce1krdkiq0hfvt6n5amrf.apps.googleusercontent.com&
	client_secret={clientSecret}&
	refresh_token=1/551G1yXUqgkDGnkfFk6ZbjMLMDIMxo3JFc8lY8CAR-Q&
	grant_type=refresh_token

请注意刷新令牌的发放数量是有限制的；限制的单位是每一组客户端-用户，你应该在长期存储器中保存刷新令牌并一直使用它直到过期。如果您的应用程序请求太多刷新令牌，就可能会触发限制，这种情况下最旧的令牌会失效。

## 允许的域

当您在设备上使用这种 OAuth 2.0 工作流，你只能获取以下域：

[分析工具配置和回报 API](https://developers.google.com/analytics/)

https://www.googleapis.com/auth/analytics

https://www.googleapis.com/auth/analytics.readonly

[日历 API](https://developers.google.com/google-apps/calendar/auth)

https://www.googleapis.com/auth/calendar

https://www.googleapis.com/auth/calendar.readonly

[通讯录 API (只读)](https://developers.google.com/google-apps/contacts/v3/#authorizing_requests_to_the_api_name_service)

https://www.googleapis.com/auth/contacts

[云打印 API](https://developers.google.com/cloud-print/)

https://www.googleapis.com/auth/cloudprint

[云存储 API](https://cloud.google.com/storage/docs/authentication)

https://www.googleapis.com/auth/devstorage.full_control

https://www.googleapis.com/auth/devstorage.read_write

[健身 REST API](https://developers.google.com/fit/rest/v1/authorization)

https://www.googleapis.com/auth/fitness.activity.read

https://www.googleapis.com/auth/fitness.activity.write

https://www.googleapis.com/auth/fitness.body.read

https://www.googleapis.com/auth/fitness.body.write

https://www.googleapis.com/auth/fitness.location.read

https://www.googleapis.com/auth/fitness.location.write

[聚合表 API](https://developers.google.com/fusiontables/docs/v2/using#auth)

https://www.googleapis.com/auth/fusiontables

[谷歌登陆](https://developers.google.com/+/api/oauth#scopes)

email

profile

https://www.googleapis.com/auth/plus.me

[YouTube 数据和直播流 API](https://developers.google.com/youtube/)

https://www.googleapis.com/auth/youtube

https://www.googleapis.com/auth/youtube.readonly

https://www.googleapis.com/auth/youtube.upload