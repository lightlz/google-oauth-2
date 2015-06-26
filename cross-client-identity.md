# 跨客户端身份凭证

当开发者构建软件时，会经常包含一些模块，例如，运行 Web 服务的模块，依赖浏览器的模块，运行 native 移动应用的模块。开发人员和使用软件的人，通常认为这些模块应该是应用程序的一部分。

谷歌的 OAuth 2.0 支持上述习惯。你必须在[谷歌开发者面板](https://console.developers.google.com/)中构建软件。[谷歌开发者面板](https://console.developers.google.com/)中的单位组织叫做 project，可以对应于一个多组件的应用程序。对每一个工程，你可以写上一些品牌信息，你必须指定应用程序将会访问的应用接口。开发者面板会生成一个独一无二的字符串 **client ID**，用来标志每一个多组件应用程序的组件。

## 跨客户端授权的目标

被一个或多个 `scope` 字符串标识的应用程序，当其使用 OAuth 2.0 授权时，它将代替用户来请求一个 OAuth 2.0 的访问令牌来访问资源。通常要由用户来批准应用程序的访问。

当应用程序被用户授予特定 `scope` 内的访问权限时，用户会留意到用户准许界面，这个准许界面包含了你在[谷歌开发者面板](https://console.developers.google.com/)中构建的工程的版本号和产品品牌信息。（关于如何构建同意（协议）界面的更多信息，请查看在开发者控制面板的 [Setting up OAuth 2.0](https://developers.google.com/console/help/generating-oauth2) ）。因此，谷歌认为在一个项目中，当用户授权任何访问特定 `scope` 的 **client ID** 时,既表示用户信任整个 `scope` 的申请。

效果就是，无论何时，当应用程序的组件被谷歌授权服务系统可靠地授权后，就不会多次提醒用户批准同一个应用访问资源。目前支持上诉效果的有 Web 客户端, JavaScript 的客户端和 Android 应用程序。

### 跨客户端访问令牌

依赖于运行代码的平台，软件能够通过多种多样的途径获取 OAuth 2.0 的访问令牌。更多信息，请查看 [Using OAuth 2.0 to Access Google APIs](https://developers.google.com/identity/protocols/OAuth2) 和 [Google Play Services Authorization](http://developer.android.com/google/play-services/auth.html)。通常应用程序获取一个访问令牌需要得到用户审批。

幸运的是，无论何时，在同一个项目中给其他组件授权，谷歌授权服务系统都能够使用到与用户批准授权的，给定项目内的 client ID 的相关信息。

效果就是，在同一个项目的同一个范围中，如果安卓应用请求了一个访问令牌，而且用户早就给 Web 组件授权了，用户就不会被再次询问授权。如下途径也是适用的：在安卓应用上，如果访问的范围已经被授权过了，用户就不用再次授权 Web 组件。

### 安卓 ID 令牌

跨客户端身份标志的一个好处就是，无需用户登录，仅利用它以及 ID 令牌，就可以让安卓应用和他们的主服务端沟通。

### 例子

假设开发者控制面板中有一个项目，这个项目包含了一个客户 ID 为 `9414861317621.apps.googleusercontent.com` 的服务端控件（一个应用程序）。 再假设在这个项目中还有一个原生安卓应用。

这个安卓应用能够代替设备（手机）上任意一个谷歌账号调用 [GoogleAuthUtil.getToken()](http://developer.android.com/google/play-services/auth.html#obtain) 方法，调用时， `audience:server:client_id:` 这个前缀会通过 Web 组件的客户 ID 来进行修正补全，也就是 `scope` 参数设置为 `audience:server:client_id:9414861317621.apps.googleusercontent.com` 。

在这个方案中，`GoogleAuthUtil` 这个对象将会察觉到在同一个项目中的安卓应用和 Web 组件的客户 ID ，不用用户同意，就可以提供谷歌验证过的 ID 令牌给应用。
这个 ID 令牌包含了 [一些字段](https://developers.google.com/identity/protocols/OpenIDConnect#obtainuserinfo)，其中一些比较重要的字段如下：

* `iss` : 总是 `accounts.google.com`

* `aud` : 项目中 Web 组件的客户 ID 

* `azp` : 项目中安卓应用的客户 ID

* `email` : 用来标识用户请求令牌的电子邮件

ID 令牌被设计成通过 HTTPS 传输，所以使用 web 组件必须按如下步骤进行：

1. 验证加密签名。因为令牌的形式是一个 JSON Web 令牌或者是 JWT，并且还有大多数流行的编程语言库来验证 JWT，这是简单而有效的。

2. 确保 `aud` 字段的值是属于你自己的客户 ID。

完成相关操作后， web 组件的令牌就有下列必然的特征：

1. 令牌是通过谷歌发布的。

2. 令牌被放置在 `email` 字段，然后发送到设备。

Web 组件可以通过安卓应用程序鉴定放置在 `azp` 字段的客户 ID。然而非兼容的或者高权限的安卓设备或许会伪造 `azp` 字段。

效果就是，Web 组件把来自安卓客户端的数据当做来自项目中的用户的数据，使用 ID 令牌来代替用户来访问和更新用户数据，而不用多次提醒用户进行身份验证。

### 安卓应用的 Web 端获得离线访问权限

再次假设一个包含了客户 ID 为 `9414861317621.apps.googleusercontent.com` 的 Web 组件，在这个事例中想代替用户访问两个不同的资源，但此时用户并不在线。然而，软件访问资源通常是通过一个安卓应用的交互来实现的。让我们更深入地假设，资源是被 [Google Drive scope](https://developers.google.com/drive/web/scopes) 内的字符串 `https://www.googleapis.com/auth/drive.file` 和 [Google+ scope](https://developers.google.com/+/api/oauth#scopes) 内的登陆字符串 `https://www.googleapis.com/auth/plus.login` 标志的，我们将上述两个字符串标记为 `resource-1` 和 `resource-2`。

在这个场景中，安卓应用就像是在开发者面板项目中的 web 组件那样，在设备上指定的任何的谷歌账户，都能够调用 [GoogleAuthUtil.getToken()](http://developer.android.com/google/play-services/auth.html#obtain) 方法，其 `scope` 参数的之为 oauth2:server:client_id:`9414861317621.apps.googleusercontent.com:api_scope:resource-1 resource-2.`。

在这个案例中，用户已经授权这个项目访问前面所说的那两个范围， `GoogleAuthUtil.getToken()`  方法将首次请求那两个范围的令牌。如果他可以这么做，那么这个方法并不会返回
 OAuth 令牌，但是会放回一个临时的授权令牌，用来更新访问令牌和刷新令牌。

安卓应用能通过 HTTPS 来发送授权码给自己的 web 组件。web 组件可以这个授权令牌返回访问令牌和刷新令牌，在 [Handling the response](https://developers.google.com/identity/protocols/OAuth2WebServer#handlingtheresponse) 中有相关描述。以下规则适用于 web 组件:

* 当 web 组件通过授权码交换令牌时，在 `POST` 请求中不应该包含 `redirect_uri` 参数。

* 当 web 组件接收到令牌时，必须检查 ID 令牌。接受来自谷歌安全通道的令牌，但是请求中并不含有签名检查。所以 web 组件还必须作如下操作：

    * 核实来自谷歌的 ID 令牌中的 `aud` 字段，是否与 [谷歌开发者面板](https://console.developers.google.com/) 中的客户 ID 一致。
    
    * 核实来自谷歌的 ID 令牌中的 `sub` 字段，是否与从客户端接受到的 ID 令牌中的 `sub` 字段一致。

* 当刷新令牌还未过期时， web 组件的任务就是安全地保存刷新令牌，使其能够长期使用。

* 重要的是，安卓应用本身不会企图使用授权码来更换刷新令牌。这将需要应用存储服务端的客户 ID 和客户密钥。这会导致你的安卓应用有安全漏洞，因为应用会极其简单地被破解。

你能够在一个较短的时间内，大概一个钟，使用访问令牌来访问请求的范围。刷新令牌不会过期，除非被明确地取消，你可以使用刷新令牌随时随地地来获取新的访问令牌。

### 建议的流程

为了有效使用资源减少不必要的工作，当安卓程序通过自己的后端开启一个网络会话时，我们推荐如下的程序流程：

1. 所有的请求都使用 HTTP 的 `POST` 请求，每个 `POST` 请求主体都要包含 ID 令牌。你获取的是在之前章节描述的 ID 令牌，要告知给用户的后端。

2. 会话的首次请求，不论它是否拥有在线访问必要范围的适当等级，通常都会有查询后端对其进行校验。

3. web 后端必须知道他是否拥有一个有效的刷新令牌。如果它没有刷新令牌或者发现了它的刷新令牌已经失效（你可以使用刷新令牌来获取新的访问令牌来测试其是否失效），web 后端就会传递缺少刷新令牌的信息给客户端。

4. 如果客户端接收到了来自后端的信息，得知了后端没有有效的刷新令牌。在安卓应用获取了授权令牌，就像前面说的那样，会将这个授权令牌传送回 web 后端来获取有效的刷新令牌。

5. 客户端和服务器继续他们之间的会话。

在这些步骤中，控制请求离线访问的功能是通过服务端的代码来维护的，最好能够知道什么时候需要离线访问。并且，如果应用总是请求刷新令牌，却不去检查服务端是否已经拥有了刷新令牌，这将导致有限的刷新令牌被消耗，这反倒可以突出用户/应用程序组合是优秀的。

上诉效果就是，用户只要经过少数的确认，就可以让项目的移动应用的组件，为其他组件取得永久的访问离线资源的权限.
