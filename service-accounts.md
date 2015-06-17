# 为服务类应用使用 OAuth 2.0 

谷歌的 OAuth 2.0 系统支持服务端到服务端的交互，类似于 Web 应用到谷歌服务那样的交互。为了使用这类方案，你需要一个服务账号，这个账号是属于你的应用程序的，用以代替个人终端用户。你的应用程序被服务账号代替调用谷歌的应用接口，所以用户不用直接参与。这个方案有时能调用 two-legged OAuth 或者 2LO 。（相关的词汇， three-legged OAuth 指的是一个种应用程序代替终端用户调用谷歌应用接口的方案，用户有时会允许这种做法）

特别地，当应用程序调用谷歌应用接口时，应用程序使用服务账号，更多是处理应用程序本身的数据而不是用户的数据。例如应用程序使用谷歌的云存储服务来储存持久化数据时，将会通过服务账号来鉴定应用的调用。

如果你拥有一个谷歌应用域名，又或者你使用谷歌应用来工作，例如，谷歌应用的管理员可以在的应用域名里授权应用代表用户访问用户数据。再例如，调用谷歌日历接口的应用，将在域名内代替用户添加事项到所有户的日历中去。在域名内，服务账号被授权代替用户访问数据，有时称为 delegating domain-wide authority 。

> 注意：当你使用谷歌应用市场来安装应用程序到你的设备上时，请求的权限将会自动授予给应用。你不需要手动为应用使用的服务账号授权。

以下文档描述了一个应用程序如何通过使用谷歌应用接口的客户端程序库或者 http ，来完成服务端到服务端的 OAuth 2.0 协议。

## 内容

### 概述

为了支持服务端到服务端的交互，首先在开发者面板中为你的项目创建一个服务账号。如果你想在谷歌应用中为用户访问用户数据，然后代理就会为服务账号使用域内访问。

接着，准备你的应用程序，通过服务账号的证书来调用授权接口，向 OAuth 2.0 auth 服务发出请求，获取访问令牌。

最后，你的应用程序能够使用访问令牌来调用谷歌应用接口。

> 建议：你的应用程序可以通过你熟悉的编程语言的谷歌接口库或者通过使用 OAuth 2.0 系统和 HTTP 来完成这些任务。然而，服务端到服务端的授权交互的体系结构会要求应用创建用于存储密钥的 JSON Web Tokens ，这种做法非常容易出现错误，严重影响到你应用的安全。
> 因为这个原因，我们强烈建议你使用封装好的库，例如 **Google APIs client libraries** ，它从你的代码中抽象了加密过程。

### 创建服务账号

服务账号的证书包含了唯一的电子邮件地址，一个客户 ID ，至少一对公共/私人密钥。

如果你的程序运行在谷歌应用引擎上，当你创建项目时就会自动建立起一个服务账号。

如果你的应用运行在谷歌计算引擎上，当你创建项目时也会自动建立起一个服务账号，但是在你创建谷歌计算引擎实例时，你必须指定应用可以访问的范围。浏览更多信息，请查看 [Preparing an instance to use service accounts](https://cloud.google.com/compute/docs/authentication#using) 。

如果你的应用并没有运行在谷歌应用引擎或者谷歌计算引擎上时，你必须在谷歌开发者面板中获取证书。为了生成一个服务账号的证书，或者浏览一个你早已生成公共证书，按如下步骤执行：

1. 访问[谷歌开发者面板](https://console.developers.google.com/)。

2. 选中一个项目或者新建一个项目。

3. 在滑动条的左边，展开 **APIs & auth** 选项。然后点击 **APIs** 。在 API 部分选中 **Enabled APIs** 来罗列所有可用的 APIs 。 如果你没有让 API 处于可用状态，在 APIs 列表中选中 **Enabled API** 按钮 ，然后使其可用。

4. 在滑动条的左边，选中证书。

5. 为了建立一个新的服务账号，按如下步骤：

    a. 在 **OAuth** 下，选中 **Create new Client ID** 。

    b. 当出现提示时，选中 **Service Account** 和 **Create Client ID** 。

    c.  弹出一个对话框，点击 **OK** ，就行了。

你创建了新的密钥，然后你就可以下载到你的机器上了。下载的仅仅只是副本。你负责安全地保存它。控制台只会在你初始化服务账号时显示你的私有密钥的密码，然后密码就不会再次显示了。你现在获取了一个 **Generate New JSON Key** 和 **Generate New P12 Key** 以及删除密钥的权利。

在任何时刻你都可以返回[开发者面板](https://console.developers.google.com/)去查看 client ID ，电子邮件地址，公共密钥指纹或者生成额外的公共密钥/私有钥对。了解更多服务账号证书的细节请打开[开发者面板](https://console.developers.google.com/)，参阅 [Service accounts](https://developers.google.com/console/help/service-accounts) 的帮助文件。

留意服务账号的电子邮箱地址，存储服务账号的 P12 私有密钥到本地，让你的应用程序可以访问到这些东西。你的应用程序在调用授权接口时需要用到它们。

> 注意：在开发环境或者产品环境中，你都必须存储以及严密地管理自己的私有密钥。谷歌不会保存你的私有密钥，只会保存公共密钥。

### 授权服务账号域内授权

如果你的应用访问了用户数据，你创建的服务账号需要访问到那些你想要访问的用户的谷歌应用数据。

以下步骤必须由管理员执行:

1. 前往你的谷歌应用的[管理员面板](http://admin.google.com/)。

2. 在列表中选中 **Security** 。如果你不能看见 **Security** 列表，在页面底部的灰色栏中选中 **More controls** ，然后在列表中选中 **Security** ， 确保你是以管理员身份执行。

3. 选中 **Show more** ，然后在选项中选中 **Advanced settings**。 

4. 在 **Authentication** 中选中 **Manage API client access** 。

5. 在 **Client Name** 字段中输入服务帐户的 **Client ID** 。

6. 在 **One or More API Scopes** 字段输入你应用可以访问的范围。例如，你的应用需要域内访问谷歌的导航应用接口以及谷歌日历应用接口，便可以输入：
https://www.googleapis.com/auth/drive , https://www.googleapis.com/auth/calendar 。

7. 点击 **Authorize** 。

您的应用程序现在有权使用应用接口调用，就像用户(模仿用户)。每当你准备使用授权接口时，指定了用户在模仿。

### 准备做一个授权 API 调用

#### Java

在你从开发者面板获取客户电子邮箱和私有密钥之后， 使用 Java 版的 [Google APIs Client Library](https://code.google.com/p/google-api-java-client/wiki/OAuth2#Service_Accounts) ，根据服务账号的证书以及你的应用需要访问的范围来创建 ```GoogleCredential ``` 对象。例如：

平台： Google App Engine

代码：

```

import com.google.api.client.googleapis.extensions.appengine.auth.oauth2.AppIdentityCredential;
import com.google.api.services.sqladmin.SQLAdminScopes;

// ...

AppIdentityCredential credential =
    new AppIdentityCredential(SQLAdminScopes.SQLSERVICE_ADMIN);

```

> 注意：如果你的你应用运行在 **Google App Engine** ，你只能使用 **AppIdentityCredential** 证书对象。如果你的应用需要在其他运行环境上运行，在本地测试你的应用时，必须检测环境以及使用不同的证书机制（查看其他平台）。

---

平台：Google Compute Engine

代码：

```

import com.google.api.client.googleapis.compute.ComputeCredential;
import com.google.api.client.googleapis.javanet.GoogleNetHttpTransport;
import com.google.api.client.http.HttpTransport;
import com.google.api.client.json.JsonFactory;
import com.google.api.client.json.jackson2.JacksonFactory;

// ...

JsonFactory JSON_FACTORY = JacksonFactory.getDefaultInstance();
HttpTransport httpTransport = GoogleNetHttpTransport.newTrustedTransport();
ComputeCredential credential = new ComputeCredential.Builder()
    .setTransport(httpTransport)
    .setJsonFactory(JSON_FACTORY)
    .build();

```

> 注意：如果你的应用程序运行在 ```Google Compute Engine``` ，你只能使用 ```ComputeCredential``` 证书对象。如果你的应用需要在其他运行环境上运行，在本地测试你的应用时，必须检测环境以及使用不同的证书机制（查看其他平台）。你可以使用默认证书来简化流程。

---

平台：其他平台

代码：

```

import com.google.api.client.googleapis.auth.oauth2.GoogleCredential;
import com.google.api.client.googleapis.javanet.GoogleNetHttpTransport;
import com.google.api.client.http.HttpTransport;
import com.google.api.client.json.JsonFactory;
import com.google.api.client.json.jackson2.JacksonFactory;
import com.google.api.services.sqladmin.SQLAdminScopes;

// ...

String emailAddress = "123456789000-abc123def456@developer.gserviceaccount.com";
JsonFactory JSON_FACTORY = JacksonFactory.getDefaultInstance();
HttpTransport httpTransport = GoogleNetHttpTransport.newTrustedTransport();
GoogleCredential credential = new GoogleCredential.Builder()
    .setTransport(httpTransport)
    .setJsonFactory(JSON_FACTORY)
    .setServiceAccountId(emailAddress)
    .setServiceAccountPrivateKeyFromP12File(new File("MyProject.p12"))
    .setServiceAccountScopes(Collections.singleton(SQLAdminScopes.SQLSERVICE_ADMIN))
    .build();

```

如果你委托域内访问给服务账号，然后你想模仿一个用户账号，要指定用户的电子邮箱给 ```GoogleCredential``` 对象的 ```setServiceAccountUser``` 方法。例如:

```

GoogleCredential credential = new GoogleCredential.Builder()
    .setTransport(httpTransport)
    .setJsonFactory(JSON_FACTORY)
    .setServiceAccountId(emailAddress)
    .setServiceAccountPrivateKeyFromP12File(new File("MyProject.p12"))
    .setServiceAccountScopes(Collections.singleton(SQLAdminScopes.SQLSERVICE_ADMIN))
    .setServiceAccountUser("user@example.com")
    .build();

```

在你的应用中使用 ```GoogleCredential``` 对象来调用谷歌应用接口。

#### Python

在你从开发者面板获取客户电子邮箱和私有密钥之后， 使用 Python 版的 [Google APIs Client Library](https://developers.google.com/api-client-library/python/guide/aaa_oauth) 完成如下步骤：

1.  根据服务账号的证书以及你的应用需要访问的范围来创建 ```GoogleCredential ``` 对象。例如：

    平台： Google App Engine

    代码：

    ```

    from oauth2client.appengine import AppAssertionCredentials

    credentials = AppAssertionCredentials(
        'https://www.googleapis.com/auth/sqlservice.admin')

    ```

    > 注意：如果你的你应用运行在**谷歌应用引擎**，你只能使用 **AppIdentityCredential** 证书对象。如果你的应用需要在其他运行环境上运行，在本地测试你的应用时，必须测试环境以及使用不同的证书机制（查看其他平台）。

    ---

    平台： Google Compute Engine

    代码：

    ```

    from oauth2client.gce import AppAssertionCredentials

    credentials = AppAssertionCredentials(
        'https://www.googleapis.com/auth/sqlservice.admin')

    ```

    > 注意：如果你的你应用运行在 ```Google Compute Engine``` ，你只能使用 ```ComputeCredential``` 证书对象。如果你的应用需要在其他运行环境上运行，在本地测试你的应用时，必须测试环境以及使用不同的证书机制（查看其他平台）。你可以使用默认的证书来简化流程。

    ---

    平台： Other

    代码：

    ```

    from oauth2client.client import SignedJwtAssertionCredentials

    client_email = '123456789000-abc123def456@developer.gserviceaccount.com'
    with open("MyProject.p12") as f:
      private_key = f.read()

    credentials = SignedJwtAssertionCredentials(client_email, private_key,
        'https://www.googleapis.com/auth/sqlservice.admin')

    ```

    如果你委托域内访问给服务账号，然后你想模仿一个用户账号，在创建 ```Credentials``` 对象时要指定用户的电子邮箱给到 ```sub``` 参数，例如：

    ```

    credentials = SignedJwtAssertionCredentials(client_email, private_key,
        'https://www.googleapis.com/auth/sqlservice.admin',
        sub='user@example.org')

    ```

2. 使用 ```Credentials``` 对象的 ```authorize``` 方法来适配 ```httplib2.Http``` 实例所有请求的必要证书头。

    ```

    from httplib2 import Http

    http_auth = credentials.authorize(Http())

    ```

在你的应用中使用被授权后的 ```Http``` 对象来调用谷歌应用接口。

#### HTTP/REST

> 建议：你的应用程序可以通过使用 OAuth 2.0 系统和 HTTP 来完成这些任务。然而，服务端到服务端的授权交互的体系结构会要求应用创建用于存储密钥的 JSON Web Tokens ，这种做法非常容易出现错误，严重影响到你应用程序的安全。

> 因为这个原因，我们强烈建议你使用封装好的库，例如 **Google APIs client libraries** ，他从你的代码中抽象了加密过程。

在你从开发者面板获取到 client ID 和私人密钥后，你需要完成如下步骤：

1. [一个 JSON Web Token ](https://developers.google.com/identity/protocols/OAuth2ServiceAccount#creatingjwt)，包含了一个头部，一个设置，一个签名。

2. 从 Google OAuth 2.0 授权服务[请求一个访问码](https://developers.google.com/identity/protocols/OAuth2ServiceAccount#makingrequest)。

3. [处理授权服务返回的 JSON 响应](https://developers.google.com/identity/protocols/OAuth2ServiceAccount#handlingresponse)。

如果响应包含了一个访问码，你可以使用这个访问码来调用谷歌应用接口。（如果响应没有包含访问令牌，你的JSON Web Token 和令牌请求或许不符合格式，或者服务账号没有权限去访问被请求的范围。）

当访问码的过了有效期，你的应用会生成另一个 JWT ，记下他，然后使用它来请求另一个访问码。

![serviceaccount](images/serviceaccount.png)

本章节余下内容包括如下：创建一个 JWT 的细节，记录 JWT ，格式化访问令牌请求，处理响应。

##### 创建一个 JWT

一个 JWT 是由三部分组成：一个头部，一个要求设置，一个签名。头部和要求设置是 JSON 对象，这些 JSON 对象被序列化成 UTF-8 字节，然后通过 Base64url 进行编码。由于重复编码操作对编码的变化提供了弹性。头部，要求设置，签名通过 （.） 字符来串接在一起。

一个 JWT 的组成如下：

```

{Base64url encoded header}.{Base64url encoded claim set}.{Base64url encoded signature}

```

签名的组成如下：

```

{Base64url encoded header}.{Base64url encoded claim set}

```

##### 格式化 JWT 的头部

头部由两个字段组成，一个是签名的算法，一个是格式的类型。全部的字段都是强制性的，每个字段只有一个值。说明了其他的算法和格式，头部也会相应的变化。

服务账号依赖于 RSA SHA-256 算法和 JWT token 格式。一个头部的 JSON 示例如下：

```

{"alg":"RS256","typ":"JWT"}

```

Base64url 编码后的展示如下：

```

eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9

```

##### 格式化 JWT 的要求设置

JWT 的要求设置包含了 JWT 的相关信息，包含了请求的权限，目标的令牌，发行者，令牌被生产时的时间，令牌的有效期。大部分字段是必要的，就像 JWT 的头部， JWT 的要求设置是一个 JSON 对象，用于签名的计算。

###### 请求要求

在下面展示了请求要求的字段。他们可以以任意的顺序出现。

| 字段名 | 描述 |
|:----:|:-----------:|
| iss | 服务账号的电子邮箱 |
| scope | 一个用空格分隔应用程序请求的权限列表。 |
| aud | 目标的描述符的断言。做一个访问令牌请求时这个值总是https://www.googleapis.com/oauth2/v3/token. |
| exp | 断言的过期时间,指定为秒就是从UTC,1970年1月1日。这个值最多1小时后发布时间。|
| iat | 断言时,指定为秒就是从UTC,1970年1月1日。 |

一个 JWT 的要求设置的 JSON 如下：

```
{
  "iss":"761326798069-r5mljlln1rd4lrbhg75efgigp36m78j5@developer.gserviceaccount.com",
  "scope":"https://www.googleapis.com/auth/devstorage.readonly",
  "aud":"https://www.googleapis.com/oauth2/v3/token",
  "exp":1328554385,
  "iat":1328550785
}

```

###### 额外的要求

在一些企业的案例中，在组织内应用程序能代表一个特定的用户请求权限。在允许执行这种模拟类型的操作前，通常是由域管理员授权应用程序模拟一个用户。查看更多有关于域管理员的信息，请前往 [Managing API client access](http://support.google.com/a/bin/answer.py?hl=en&answer=162106) 。

为了获取访问码，使应用可以访问到包括用户的电子邮箱在内的资源， JWT 的要求设置的 ```sub``` 字段描述如下：

| 字段名 | 描述 |
|:----:|:-----------:|
| sub | 应用程序请求授权访问用户的电子邮件地址。 |

如果应用程序没有权限去模仿用户，那么响应的访问码包含的 ```sub``` 字段将是一个错误。

包含 ```sub``` 字段的 JWT 要求设置如下：

```

{
  "iss":"761326798069-r5mljlln1rd4lrbhg75efgigp36m78j5@developer.gserviceaccount.com",
  "sub":"some.user@example.com",
  "scope":"https://www.googleapis.com/auth/prediction",
  "aud":"https://www.googleapis.com/oauth2/v3/token",
  "exp":1328554385,
  "iat":1328550785
}

```

###### 对 JWT 的要求设置进行编码

就像 JWT 的头部，要求设置应该被序列化成 UTF-8 字节，然后通过 Base64url 进行编码。下面的就是一个 JWT 的要求设置的 JSON 例子和 Base64url-safe 例子：

```

{
  "iss":"761326798069-r5mljlln1rd4lrbhg75efgigp36m78j5@developer.gserviceaccount.com",
  "scope":"https://www.googleapis.com/auth/prediction",
  "aud":"https://www.googleapis.com/oauth2/v3/token",
  "exp":1328554385,
  "iat":1328550785
}

```

```

eyJpc3MiOiI3NjEzMjY3OTgwNjktcjVtbGpsbG4xcmQ0bHJiaGc3NWVmZ2lncDM2bTc4ajVAZGV2ZWxvcGVyLmdzZXJ2aWNlYWNjb3VudC5jb20iLCJzY29wZSI6Imh0dHBzOi8vd3d3Lmdvb2dsZWFwaXMuY29tL2F1dGgvcHJlZGljdGlvbiIsImF1ZCI6Imh0dHBzOi8vYWNjb3VudHMuZ29vZ2xlLmNvbS9vL29hdXRoMi90b2tlbiIsImV4cCI6MTMyODU1NDM4NSwiaWF0IjoxMzI4NTUwNzg1fQ

```

###### 计算签名

[JSON Web Signature](http://self-issued.info/docs/draft-ietf-jose-json-web-signature.html) 是用来为 JWT 生成签名的。输入的签名就如下列的字符数组：

```

{Base64url encoded header}.{Base64url encoded claim set}

```

在计算签名的时候，必须使用 JWT 头部中指定的签名算法。Google OAuth 2.0 授权服务只支持使用 SHA-256 哈希算法的 RSA 。这个算法在 JWT 头部的 ```alg``` 字段表示成 RS256 。 

使用 SHA256withRSA （就是使用 SHA-256 哈希函数的 RSASSA-PKCS1-V1_5-SIGN ）和从开发者面板获取的私人密钥计算输入的 UTF-8 序列。将输出字符数组。

签名必须被 Base64url 编码。头部，要求设置，签名，通过使用实心圆点 （.） 进行串接，其结果就是 JWT 。他应该像如下展示那样：

```

{Base64url encoded header}.
{Base64url encoded claim set}.
{Base64url encoded signature}

```

下面是 JWT 被 Base64url 编码前的例子：

```

{"alg":"RS256","typ":"JWT"}.
{
"iss":"761326798069-r5mljlln1rd4lrbhg75efgigp36m78j5@developer.gserviceaccount.com",
"scope":"https://www.googleapis.com/auth/prediction",
"aud":"https://www.googleapis.com/oauth2/v3/token",
"exp":1328554385,
"iat":1328550785
}.
[signature bytes]

```

下面是 JWT 被 Base64url 编码后，并准备传输的例子：

```

eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.
eyJpc3MiOiI3NjEzMjY3OTgwNjktcjVtbGpsbG4xcmQ0bHJiaGc3NWVmZ2lncDM2bTc4ajVAZGV2ZWxvcGVyLmdzZXJ2aWNlYWNjb3VudC5jb20iLCJzY29wZSI6Imh0dHBzOi8vd3d3Lmdvb2dsZWFwaXMuY29tL2F1dGgvcHJlZGljdGlvbiIsImF1ZCI6Imh0dHBzOi8vYWNjb3VudHMuZ29vZ2xlLmNvbS9vL29hdXRoMi90b2tlbiIsImV4cCI6MTMyODU1NDM4NSwiaWF0IjoxMzI4NTUwNzg1fQ.
ixOUGehweEVX_UKXv5BbbwVEdcz6AYS-6uQV6fGorGKrHf3LIJnyREw9evE-gs2bmMaQI5_UbabvI4k-mQE4kBqtmSpTzxYBL1TCd7Kv5nTZoUC1CmwmWCFqT9RE6D7XSgPUh_jF1qskLa2w0rxMSjwruNKbysgRNctZPln7cqQ

```

###### 创建访问令牌请求

在生成一个 JWT 之后，应用可以使用这个 JWT 来请求访问码。访问码请求是一个 HTTPS 的 ```POST``` 请求， POST 的数据包会被 URL 编码。 URL 是：

```

https://www.googleapis.com/oauth2/v3/token

```

在 HTTPS ```POST``` 请求中的参数如下：

The following parameters are required in the HTTPS ```POST``` request:

| 字段名 | 描述 |
|:----:|:-----------:|
| grant_type | 使用如下的字符串，如有必要请进行 URL 编码：urn:ietf:params:oauth:grant-type:jwt-bearer 。 |
| assertion | JWT,包括签名。 |

下面是用来请求访问码的 HTTPS ```POST``` 的原生数据：

```

POST /oauth2/v3/token HTTP/1.1
Host: www.googleapis.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer&assertion=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiI3NjEzMjY3OTgwNjktcjVtbGpsbG4xcmQ0bHJiaGc3NWVmZ2lncDM2bTc4ajVAZGV2ZWxvcGVyLmdzZXJ2aWNlYWNjb3VudC5jb20iLCJzY29wZSI6Imh0dHBzOi8vd3d3Lmdvb2dsZWFwaXMuY29tL2F1dGgvcHJlZGljdGlvbiIsImF1ZCI6Imh0dHBzOi8vYWNjb3VudHMuZ29vZ2xlLmNvbS9vL29hdXRoMi90b2tlbiIsImV4cCI6MTMyODU3MzM4MSwiaWF0IjoxMzI4NTY5NzgxfQ.ixOUGehweEVX_UKXv5BbbwVEdcz6AYS-6uQV6fGorGKrHf3LIJnyREw9evE-gs2bmMaQI5_UbabvI4k-mQE4kBqtmSpTzxYBL1TCd7Kv5nTZoUC1CmwmWCFqT9RE6D7XSgPUh_jF1qskLa2w0rxMSjwruNKbysgRNctZPln7cqQ

```

下面使用使用 ```curl``` 做相同的请求：

```

curl -d 'grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer&assertion=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiI3NjEzMjY3OTgwNjktcjVtbGpsbG4xcmQ0bHJiaGc3NWVmZ2lncDM2bTc4ajVAZGV2ZWxvcGVyLmdzZXJ2aWNlYWNjb3VudC5jb20iLCJzY29wZSI6Imh0dHBzOi8vd3d3Lmdvb2dsZWFwaXMuY29tL2F1dGgvcHJlZGljdGlvbiIsImF1ZCI6Imh0dHBzOi8vYWNjb3VudHMuZ29vZ2xlLmNvbS9vL29hdXRoMi90b2tlbiIsImV4cCI6MTMyODU3MzM4MSwiaWF0IjoxMzI4NTY5NzgxfQ.RZVpzWygMLuL-n3GwjW1_yhQhrqDacyvaXkuf8HcJl8EtXYjGjMaW5oiM5cgAaIorrqgYlp4DPF_GuncFqg9uDZrx7pMmCZ_yHfxhSCXru3gbXrZvAIicNQZMFxrEEn4REVuq7DjkTMyCMGCY1dpMa8aWfTQFt3Eh7smLchaZsU
' https://www.googleapis.com/oauth2/v3/token

```

###### 处理响应

如果 JWT 和访问码请求被适当的编码，以及服务账号拥有执行的权限，从授权服务返回 JSON 响应包含一个访问码，一个响应例子如下：

```

{
  "access_token" : "1/8xbJqaOZXSUZbHLl5EOtu1pxz3fmmetKx9W8CV4t79M",
  "token_type" : "Bearer",
  "expires_in" : 3600
}

```

访问码的有效期为一个小时，在有限期内可以被有效使用的。

### 调用谷歌应用接口

#### JAVA

使用 ```GoogleCredential``` 对象完成调用谷歌应用接口的步骤如下：

1. 通过 ```GoogleCredential``` 对象创建一个服务对象调用你想调用的接口。例如：

    ```

    SQLAdmin sqladmin =
        new SQLAdmin.Builder(httpTransport, JSON_FACTORY, credential).build();

    ```

2. 使用 [interface provided by the service object](https://code.google.com/p/google-api-java-client/) 为应用接口服务创建一个请求。例如，列出 exciting-example-123 工程的实例化的云数据库：

    ```
    SQLAdmin.Instances.List instances =
        sqladmin.instances().list("exciting-example-123").execute();

    ```

#### Python
 
使用被授权了的 ```HTTP``` 对象完成调用谷歌应用接口，步骤如下：

1. 创建一个服务对象调用你想调用的接口。通过创建函数以及名字、接口的版本以及被授权的 HTTP 对象来创建一个服务对象。例如，调用版本号为 1beta3 的云数据库管理员接口的代码如下：

    ```

    from apiclient.discovery import build

    sqladmin = build('sqladmin', 'v1beta3', http=http_auth)

    ```

2. 使用 [interface provided by the service object](https://code.google.com/p/google-api-java-client/) 为应用接口服务创建一个请求。例如，列出 exciting-example-123 工程的实例化的云数据库：

    ```

    response = sqladmin.instances().list(project='exciting-example-123').execute()

    ```

#### HTTP/REST

在你的应用获取到访问令牌之后，你能使用这个令牌代替用户或者服务账号来调用谷歌应用接口。为了完成上诉功能，在调用应用接口的请求通过 ```access_token``` 字段查询参数或者 ```Authorization: Bearer``` HTTP 头部包含访问码。如果允许，使用 HTTP 头部包含访问令牌会更好，因为查询字符串容易被泄露。你可以通过封装的库来建立更多的谷歌应用调用的事例。（例如调用 [People API](https://developers.google.com/+/api/latest/people/get#examples)）。

你可以尝试所有的谷歌应用接口，在 [OAuth 2.0 Playground](https://developers.google.com/oauthplayground/) 查看他们的应用范围。

##### 例子

如下，调用 [people.get](https://developers.google.com/+/api/latest/people/get) 的终端通过使用 ```access_token``` 查询字符串参数，你需要指定属于你自己的访问码：

```

GET https://www.googleapis.com/plus/v1/people/userId?access_token=1/fFBGRNJru1FQd44AzqT3Zg

```

下面是授权用户使用 ```Authorization: Bearer``` 头部完成相同的调用：

```

GET /plus/v1/people/me HTTP/1.1
Authorization: Bearer 1/fFBGRNJru1FQd44AzqT3Zg
Host: googleapis.com

```

你可以尝试在控制台应用中使用 curl 。下面是使用 HTTP 头部完成相同的调用功能：

```

curl -H "Authorization: Bearer 1/fFBGRNJru1FQd44AzqT3Zg" https://www.googleapis.com/plus/v1/people/me

```

或者，作为一种完成相同的调用功能选择，使用查询字符串参数：

```

curl https://www.googleapis.com/plus/v1/people/me?access_token=1/fFBGRNJru1FQd44AzqT3Zg

```

##### 当访问码过期时

通过 Google OAuth 2.0 Authorization Server 发布的访问码的有效期是一个小时。当访问码失效后，应用应该生成另一个 JWT 来请求另一个新的访问码。

