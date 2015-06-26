# 谷歌应用程序默认凭证

为调用谷歌应用接口，谷歌应用程序的默认凭证提供了一个简单途径来获取授权证书。

他们有最适合这些事例的解决方案，当应用程序独立于用户去调用接口时，需要拥有相同的的身份标识和授权等级。这里推荐授予应用调用谷歌云应用接口的权限，特别是在你要发布到谷歌应用引擎或者谷歌计算引擎虚拟机上的应用程序时。

## 内容

### 何时使用应用程序的默认凭证

出现下列任一情形时，我们推荐你使用应用程序的默认凭证：

* 你的代码运行在谷歌应用引擎或者谷歌计算引擎上。当部署应用程序时，其默认凭证提供了访问内置服务账号的权限，但是也提供了部署之前，测试应用程序时可供选择的凭证。

* 为避免在应用程序代码中嵌入的鉴定信息。通常一个最佳使用方式是，以避免包含密钥认证的相关源代码，如测试和生产时，需要在不同的上下文中使用不同的凭证。使用环境变量，在应用程序外可以定义凭证。

* 正在访问的应用接口，其中的数据是关联到一个云项目或整个应用程序的,而不是个人用户数据。所以最好使用一个认证流程，最终用户明确同意访问（查看 [使用 OAuth 2.0 访问谷歌应用接口](https://developers.google.com/identity/protocols/OAuth2)）。

### 应用程序的默认凭证如何工作

通过调用客户端代码库，你可以获取到应用程序的默认凭证。返回的凭证是由环境中运行的代码决定的。按照以下顺序进行检查:

1. 检测环境变量 GOOGLE_APPLICATION_CREDENTIALS 。如果环境变量是规定的，它应该指明了定义凭证的文件。通过这个简单的方法可以获取到凭证，其目的是使用 [谷歌开发者面板](https://console.developers.google.com/) 的 **APIs & Auth** 选项中的子选项 **Credentials** 来创建一个服务帐户。创建一个服务账号或者选择一个现存的，选中 **Generate new JSON key** 选项。为下载的 JSON 文件设置环境变量。

2. 如果你在你的设备上安装了谷歌云 SDK ，需要运行命令 [`gcloud auth login`](https://cloud.google.com/sdk/gcloud/reference/auth/login) ，你的身份可以用作代理，从那台机器测试代码调用应用接口。

3. 如果你的应用程序时运行在谷歌应用引擎上的，内置的服务账号要关联到应用程序。

4. 如果你的应用程序时运行在谷歌计算引擎上的，内建的服务账号必须关联到虚拟机。

5. 如果不遵循上述约束，就会发生错误。

### 在应用程序代码中调用应用程序的默认凭证

默认凭证被集成在谷歌应用接口客户端库中。支持的环境有安装在 Linux ， Windows 和 Mac OS 上的应用，以及 GCE 虚拟机。

#### Java

版本为 1.19 的 [谷歌应用接口客户端库 Java 版](https://developers.google.com/api-client-library/java/) 中集成了默认凭证。

```
import com.google.api.client.googleapis.auth.oauth2.GoogleCredential;

...

GoogleCredential credential = GoogleCredential.getApplicationDefault();
```

如下，它可以用来访问一个应用接口服务:

```
Compute compute = new Compute.Builder
    (transport, jsonFactory, credential).build();
```

一些凭证类型向你请求某些范围，但是服务实体入口点并不受理。如果你偶尔遭遇了这个情节，就像下面所做的那样，你需要注入范围：

```
Collection COMPUTE_SCOPES =
    Collections.singletonList(ComputeScopes.COMPUTE);
if (credential.createScopedRequired()) {
    credential = credential.createScoped(COMPUTE_SCOPES);
}
```

参考：[谷歌应用接口客户端库 Java 版文档](https://developers.google.com/api-client-library/java/google-api-java-client/reference).

#### Python

版本为 1.3 的 [谷歌应用接口客户端库 Python 版](https://developers.google.com/api-client-library/java/) 中集成了默认凭证。

```
from oauth2client.client import GoogleCredentials
credentials = GoogleCredentials.get_application_default()
```

可以这样使用服务:

```
from googleapiclient.discovery import build

...

service = build('compute', 'v1', credentials=credentials)
```

`build()` 方法在给定的服务内留意了注入的适当范围，虽然 `create_scoped` 方法可以显式地做到这一点。

参考：[谷歌应用接口客户端库 Python 版文档](http://google.github.io/oauth2client/source/oauth2client.html#oauth2client.client.GoogleCredentials).

### 工具支持

#### 谷歌云 SDK

自谷歌云 SDK 0.9.51 以来，当在本地测试应用程序代码时，使用命令 [`gcloud auth login`](https://cloud.google.com/sdk/gcloud/reference/auth/login) ，可以让你的身份凭证用来当做应用程序默认凭证的代理。

这是最快速便捷地在本地校验代码的方法。有一点需要注意，你的个人身份凭证通常有很多的权限，所以你该找一个场景，这个场景是一个应用接口在本地被调用而不是在发布之后被调用。它只能在谷歌云应用接口许可范围内工作。如果这个方法引发了问题，考虑下载服务账号密钥来设置环境变量 [environment variable](https://developers.google.com/identity/protocols/application-default-credentials#howtheywork) 。

#### Google App Engine SDK

谷歌应用引擎 SDK

自谷歌应用引擎SDK 1.9.18 以来，当在本地运行开发用应用程序服务，可以使用应用程序的默认凭证。注意，这个仅仅支持通过使用命令 [`gcloud preview app run`](https://cloud.google.com/sdk/gcloud/reference/preview/app/run) 来完成上诉效果，不支持在老旧的特定语言的开发服务器。

### 发现并修理故障

在运行环境的上下文中，应用程序的默认凭证允许你使用不同的身份凭证。这有时意味着需要调整来确保所有正在使用的身份拥有权限。

#### 权限

使用应用程序的默认凭证的代码，可以像用户或者服务账号那样的身份凭证运行，包括内置的服务账号的身份凭证。如果使用了一个以上的身份凭证，他们必须都有权限调用。配置权限，请打开[谷歌开发者面板](https://console.developers.google.com/)的 **Permissions** 选项。

#### 范围

他们采取 uri 的形式,名字一组给定的 API 的功能， OAuth2 的权限范围的定义是在一个给定的上下文中，可被调用的应用接口。他们采取 uri 的形式,为给定的应用接口命名,例如， "https://www.googleapis.com/auth/compute.readonly"。不同类型的身份凭证处理不同的范围。

下载的服务账号的密钥和谷歌应用引擎内建的服务账号，其范围必须在代码中指定。 应用接口的封装或许该这么做，但是在使用这个证书时，会出错。获取更多信息如何在你的代码中注入范围，请查看 [Calling the Application Default Credentials in application code](https://developers.google.com/identity/protocols/application-default-credentials#calling)。

谷歌计算引擎虚拟机的服务账号，其支持的范围必须在虚拟机创建时指定。如果使用谷歌云 SDK ，就要使用命令 [`gcloud compute instances create`](https://cloud.google.com/sdk/gcloud/reference/compute/instances/create) 以及指定 **--scopes** 参数来完成。

如果使用谷歌云 SDK 的命令 [`gcloud auth login`](https://cloud.google.com/sdk/gcloud/reference/auth/login) ，在本地提供自己的身份凭证，范围就会被固定，即使范围包含了所有的谷歌云应用接口的范围。如果你需要不在这里谈及的范围，建议你下载服务中的密钥。
