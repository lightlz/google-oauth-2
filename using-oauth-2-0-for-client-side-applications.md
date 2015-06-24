## 在客户端方面应用程序上使用 OAuth 2.0

谷歌 OAuth 2.0 端点支持以 JavaScript 为中心的应用程序。这些应用程序可以在用户值守时访问谷歌 API，并且这类应用程序不能保存 secret。

该文章描述如何通过 OAuth 2.0 让 JavaScript 应用程序访问谷歌 API。 

### 内容

### 概览

这种方案下，程序会将浏览器（完整页面或者弹窗）重定向至一个谷歌 URL 并附带一系列查询参数来告知该应用程序所请求的谷歌 API 权限。和其他方案一样，谷歌会负责用户认证和用户准许（User consent），这一系列程序的结果是一个访问令牌。谷歌会将该访问令牌作为回应的一部分发回，然后客户端方面的脚本应用程序将访问令牌从回应中提取出来。

应用程序在接收到访问令牌之后就可以访问谷歌 API 了。

**注意**: 在这种方案下，您的应用程序应该总是使用 HTTPS 连接。

### 生成 URL


用于认证用户的 URL 是 [https://accounts.google.com/o/oauth2/auth](https://accounts.google.com/o/oauth2/auth) 。这个端点只能通过 SSL 访问，HTTP（非-SSL）连接会被拒绝。
 
----------

**端点：**https://accounts.google.com/o/oauth2/auth  
**描述：**这个端点是初次请求的目标。它负责处理寻找活动会话，认证用户和用户准许。

----------

对于客户端方面应用程序而言，谷歌认证服务器支持的查询串参数集为：

----------

**参数：**	response_type （响应类型）  
**值：**	 token （令牌）  
**描述：**	该值会告知谷歌认证服务器在回应片段中返回访问令牌。


**参数：**	client_id （客户端 ID）  
**值：**	你从[开发者控制台](https://console.developers.google.com/)处获得的客户端 ID。  
**描述：**	确定是哪个客户端正在发出请求。传过去的这个参数值必须要与[开发者控制台](https://console.developers.google.com/)里显示的完全一致。  


**参数：**	redirect_uri （重定向 URI）  
**值：**	 在[开发者控制台](https://console.developers.google.com/)里列出的这个工程的 redirect_urivalues 其中一个值。  
**描述：**	决定回应(Response)会发向哪里。这个参数的值必须和[谷歌开发者控制台](https://console.developers.google.com/)为这个工程所显示的值的其中一个完全一致（包括完整的 HTTP 或 HTTPS 格式、大小写、和末尾的'/'符号）。
 
 
**参数：**	scope （域）   
**值：**	用空格分隔的该应用程序所请求的权限集。  
**描述：**	确认您的应用程序请求的谷歌 API 访问权。传过去的参数值会以用户准许页面的方式向用户显示。请求的权限数量和获得用户准许的可能性有逆相关关系。若想了解可用的登陆域，请参见[登陆域](https://developers.google.com/+/api/oauth#login-scopes)。若想了解所有谷歌 API 的可用域，请访问 [API 浏览器](https://developers.google.com/apis-explorer/#p/)。基本上渐进地请求域是一项最佳实践，比起提前一次性请求所有权限，按需请求更好。举例来说，一个想要支持购买功能的应用程序应该在用户点击“购买”按钮的时候才要求谷歌钱包访问权; 详情参阅[渐进式授权](https://developers.google.com/accounts/docs/OAuth2WebServer#incrementalAuth)。
 
 
**参数：**	state （状态）  
**值：**	任意字符串  
**描述：**	当收到回应时提供任何可能对您的应用程序有用的状态。谷歌认证服务器会回传这个参数，所以您的应用程序会收到和它发出去一样的内容。可以用于将用户重定向到正确的站点上的资源，Nonce(一次性加密串)，用于防止跨站请求伪造攻击的防伪造令牌。
 
 
**参数：**	approval_prompt （准许提示）  
**值：** force 或 auto
**描述：**	决定用户是否应该再次进行用户准许。默认值是 auto（自动），表示用户只需要在第一次权限请求的时候看见用户许可页面。如果这个值是 force（强制），那么用户即使以前许可了您的应用程序的相关权限，也会再次见到用户许可页面。
 

**参数：**	login_hint （登陆提示）  
**值：**	电子邮箱地址 或 子标识符  
**描述：**	当您的应用程序知道哪个用户正在尝试进行认证时，可以提供这个参数作为对认证服务器的提示。将这个参数传过去会在登陆页面自动填写用户邮箱地址，或者选择适合的多用户登录会话，从而简化登陆工作流。


**参数：**	include_granted_scopes （包含已许可的域）  
**值：**	true 或 false  
**描述：**	如果这个参数值被设为 true，同时认证请求被许可，那么该次认证会许可所有 曾经许可给这组用户和应用程序的其他域；请参见[渐进式授权](https://developers.google.com/accounts/docs/OAuth2WebServer#incrementalAuth)。
 

----------

An example URL is shown below, with line breaks and spaces for readability.

	https://accounts.google.com/o/oauth2/auth?
	 scope=email%20profile&
	 state=%2Fprofile&
	 redirect_uri=https%3A%2F%2Foauth2-login-demo.appspot.com%2Foauthcallback&
	 response_type=token&
	 client_id=812741506391.apps.googleusercontent.com

### 处理回应

如果用户准许您的应用程序权限请求，谷歌会回应一个访问令牌给您的应用程序。访问令牌在返回的片段中的 access_token 参数中。由于片段不会整个返回给服务器，所以客户端方面脚本程序必须分析片段并且从 access_token 参数中提取值。在回应中还存在其他参数，包括 expires_in 和 token_type。这些参数描述了令牌的有效时限（单位为秒），返回的令牌的类型。如果请求中包含状态参数，那么回应中也会包含状态参数。

一个 User Agent 流回应的例子如下：

	https://oauth2-login-demo.appspot.com/oauthcallback#access_token=1/fFBGRNJru1FQd44AzqT3Zg&token_type=Bearer&expires_in=3600

>**注意**: 除了上面提到的字段之外，可能会有别的字段也会存在于回应内，您的应用程序不能将这种情况视为错误。上述字段集仅是最小集。

下面是一小段 JavaScript 脚本用于分析回应并且将参数返回给服务器。这个代码被托管在这个 URL 上：https://oauth2-login-demo.appspot.com/oauthcallback。

	// First, parse the query string
	var params = {}, queryString = location.hash.substring(1),
	    regex = /([^&=]+)=([^&]*)/g, m;
	while (m = regex.exec(queryString)) {
	  params[decodeURIComponent(m[1])] = decodeURIComponent(m[2]);
	}
	
	// And send the token over to the server
	var req = new XMLHttpRequest();
	// consider using POST so query isn't logged
	req.open('GET', 'https://' + window.location.host + '/catchtoken?' + queryString, true);
	
	req.onreadystatechange = function (e) {
	  if (req.readyState == 4) {
	     if(req.status == 200){
	       window.location = params['state']
	   }
	  else if(req.status == 400) {
	        alert('There was an error processing the token.')
	    }
	    else {
	      alert('something else other than 200 was returned')
	    }
	  }
	};
	req.send(null);

这段代码将从接收到的片段中的参数通过 XMLHttpRequest 发送给服务器，并且将访问令牌写入浏览器的本地存储中。后者是一个可选操作，这取决于应用程序是否请求其他 JavaScript 代码来调用谷歌 API。并且请注意这个代码将参数发往 /accepttoken 端点，并且他们是通过 HTTPS 通道传输的。

##### 错误回应

如果用户没有许可应用程序所请求的权限，谷歌认证服务器会回应一个错误。这个错误以片段形式返回。

一个错误回应示例如下：

	https://oauth2-login-demo.appspot.com/oauthcallback#error=access_denied

### 验证令牌

以片段形式接收的令牌**必须**被显式地确认。如果没能验证这种方式获取的令牌，将会让您的应用程序更可能遭受[责任混淆问题](http://en.wikipedia.org/wiki/Confused_deputy_problem)(Confused Deputy Problem)。

您可以通过发送一个 web 服务请求给谷歌认证服务器来确认令牌，并且对该 web 服务请求的结果进行一次字符串比对。

#### TokenInfo 验证

通过谷歌认证服务器来验证一个令牌相对来说简单。您的应用程序包含了一个访问令牌在access_token 参数，这个令牌可用于以下端点：

----------

**端点：**https://accounts.google.com/o/oauth2/auth  
**描述：**接收一个访问令牌并且返回访问令牌的相关信息，包括该令牌所授权的程序，该令牌的目标，用户已经许可的域，令牌的剩余生命期，用户 ID 等。

----------

下面是这类请求的一个示例：

	https://www.googleapis.com/oauth2/v1/tokeninfo?access_token=1/fFBGRNJru1FQd44AzqT3Zg

TokenInfo端点会回应一个 JSON 对象来描述令牌或错误。下面是在没有发生错误的情况下，该对象所包含的字段表：

----------

**参数：**	audience （受众）   
**描述：**	令牌的目标应用程序。


**参数：**	scope （域）   
**描述：**	空格为间隔符的用户已经许可的域集。
 

**参数：**	userid （用户 ID）   
**描述：**	这个字段只会在 profile 域存在于请求内时才会出现。该字段的值是一个已登录用户的一个恒定的标识符，用于创建和管理您的应用程序的用户会话。这个标识符和客户端 ID 是相互独立的。这意味着在同一个组织内多个应用程序联系同一个 profile 信息成为可能。 


**参数：**	expires_in （有效期至）   
**描述：**	令牌的剩余生命期，以秒为单位。
 
----------

下面是回应的一个示例：

	{
	  "audience":"8819981768.apps.googleusercontent.com",
	  "user_id":"123456789",
	  "scope":"profile email",
	  "expires_in":436
	}

>**注意：** 当验证一个令牌时，请务必确认回应中的 audience 字段精确吻合你在[开发者控制台](https://console.developers.google.com/)中获得的客户端 ID。这个步骤是绝对关键的，因为这是防止混淆责任问题的一个手段。如果令牌已经过期、或者被篡改、或者权限已经被废除，谷歌认证服务器都会返回一个错误。错误会包含 400 状态码，而 JSON 对象内容如下：
>
>	{"error":"invalid_token"}
>	
>认证服务器不会提供任何额外信息来指出失败原因，这是故意的。

### 调用谷歌 API

您的程序获得访问令牌之后，您可以使用令牌以用户或者服务账户的名义来对谷歌 API 进行调用。要做到这一点，请将访问令牌包含到发给 API 的请求中，可以通过包含 access_token 查询参数或者一个 Authorization: Bearer HTTP 头来实现。如果可能的话，我们更欢迎 HTTP 头的方法，因为查询串更容易在服务器记录中可见。在大多数情况下你可以使用客户端库来设置您的谷歌 API 调用（例如，当对[谷歌人脉 API 进行调用时](https://developers.google.com/+/api/latest/people/get#examples)）。

你可以在[OAuth 2.0 游乐场](https://developers.google.com/oauthplayground/)中自行尝试所有的谷歌 API 并查看他们所对应的域。

#### 示例

一个通过使用 access_token 查询串参数对 [people.get](https://developers.google.com/+/api/latest/people/get) 端点 （谷歌人脉 API）的调用会和下例相类似，当然在实际情况中您需要提供您自己的访问令牌：

	GET https://www.googleapis.com/plus/v1/people/userId?access_token=1/fFBGRNJru1FQd44AzqT3Zg

而对于已经认证的用户（me）而言，通过 Authorization: Bearer HTTP 头调用同样的 API 就会像下面这样：

	GET /plus/v1/people/me HTTP/1.1
	Authorization: Bearer 1/fFBGRNJru1FQd44AzqT3Zg
	Host: googleapis.com

您可以尝试 curl 命令行应用程序。下面是使用 HTTP 头的方法（推荐）的示例：

	curl -H "Authorization: Bearer 1/fFBGRNJru1FQd44AzqT3Zg" https://www.googleapis.com/plus/v1/people/me

或者，您也可以使用查询串参数的方法实现：

	curl https://www.googleapis.com/plus/v1/people/me?access_token=1/fFBGRNJru1FQd44AzqT3Zg

## 渐进式授权


在 OAuth 协议中，您的应用程序为了访问资源所请求的认证是以域来区分的，如果用户已经被认证而且同意权限请求，您的应用程序会接收到一个生命期比较短的访问令牌，这些令牌能让它访问目标资源，而刷新令牌（可选）可以让应用程序拥有长期的访问权限。

按需请求资源访问权限通常被认为是一项最佳实践。举例来说，一个让人们可以对音乐进行采样并且创建混音的应用程序也许在登陆时候只需要非常少量的资源，说不定除了登陆者的名字之外就没别的了。但是，要保存一个完整的混音可能会需要访问他们的谷歌网盘。大多数人都会觉得在应用程序需要存储文件时申请谷歌网盘的访问权限是非常自然的。

这种情况下，在登录时应用程序可以请求这个域：https://www.googleapis.com/auth/plus.loginto 来实现一个基本的社交登陆功能，然后在需要保存混音的时候才申请这个域：https://www.googleapis.com/auth/drive.file。

同时使用在[使用 OpenID 连接](https://developers.google.com/identity/protocols/OpenIDConnect)和[使用 OAuth 2.0 来访问谷歌 API](https://developers.google.com/accounts/docs/OAuth2) 里描述的步骤通常会让你的应用程序不得不管理两个不同的访问令牌。如果您希望避开这种复杂性，则需要在所有 OAuth 2.0 工作流的第一步时，将发送给 https://accounts.google.com/o/oauth2/auth 的认证 URI 里添加一个额外的参数。这个参数是 include_granted_scopes，其中值可以被赋为 true 或 false（默认值是 false），当这个值为 true 时，如果您的域认证要求被批准，谷歌认证服务器会将这次认证和所有之前已经成功的认证为这组用户-应用程序合并。这类请求的 URI 可能看上去会像下面这样（下例有插入换行和空格来增强可读性）：

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
https://www.googleapis.com/auth/plus.loginscope ，然后又向同一个用户的移动端发送了同样的请求，那么后面这个请求会被自动批准，因为组合认证会包含两边的域。

- 当您废除了一个代表组合认证的的令牌，其所有的认证都会被同时废除；这意味着如果你还保留着任何以往的权限的令牌，他们也会跟着失效。