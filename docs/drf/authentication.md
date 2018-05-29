> [官方原文链接](http://www.django-rest-framework.org/api-guide/authentication/)  
> [前往掘金阅读](https://juejin.im/post/5aaa7d11518825558154b69a)  

# 身份验证

身份验证是将传入请求与一组识别凭证（例如请求的用户或其签名的令牌）相关联的机制。然后，权限和限制策略可以使用这些凭据来确定请求是否应该被允许。

REST framework 提供了许多开箱即用的身份验证方案，同时也允许你实施自定义方案。

身份验证始终在视图的开始处运行，在执行权限和限制检查之前，在允许继续执行任何其他代码之前。

`request.user` 属性通常会设置为 `contrib.auth` 包的 `User` 类的一个实例。

`request.auth` 属性用于其他身份验证信息，例如，它可以用来表示请求已签名的身份验证令牌。

---

**注意：** 不要忘记， **身份验证本身不会（允许或不允许）传入的请求**，它只是标识请求的凭据。

---

## 如何确定身份验证

认证方案总是被定义为一个类的列表。  REST framework 将尝试使用列表中的每个类进行认证，并将使用成功认证的第一个类的返回值来设置 `request.user` 和 `request.auth` 。

如果没有类进行身份验证，则将 `request.user` 设置为 `django.contrib.auth.models.AnonymousUser` 的实例，并将 `request.auth` 设置为 `None`.

可以使用 `UNAUTHENTICATED_USER` 和 `UNAUTHENTICATED_TOKEN` 设置修改未经身份验证的请求的 `request.user` 和 `request.auth` 的值。

## 设置认证方案

默认的认证方案可以使用 `DEFAULT_AUTHENTICATION_CLASSES`  setting 全局设置。例如。

``` python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework.authentication.BasicAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    )
}
```

您还可以使用基于 `APIView` 类的视图，在每个视图或每个视图集的基础上设置身份验证方案。

``` python
from rest_framework.authentication import SessionAuthentication, BasicAuthentication
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework.views import APIView

class ExampleView(APIView):
    authentication_classes = (SessionAuthentication, BasicAuthentication)
    permission_classes = (IsAuthenticated,)

    def get(self, request, format=None):
        content = {
            'user': unicode(request.user),  # `django.contrib.auth.User` instance.
            'auth': unicode(request.auth),  # None
        }
        return Response(content)
```

或者，如果您将 `@api_view` 装饰器与基于函数的视图一起使用。

``` python
@api_view(['GET'])
@authentication_classes((SessionAuthentication, BasicAuthentication))
@permission_classes((IsAuthenticated,))
def example_view(request, format=None):
    content = {
        'user': unicode(request.user),  # `django.contrib.auth.User` instance.
        'auth': unicode(request.auth),  # None
    }
    return Response(content)
```

## 未经授权和禁止响应

当未经身份验证的请求被拒绝时，有两种不同的错误代码可能是合适的。

* [HTTP 401 Unauthorized][http401]
* [HTTP 403 Permission Denied][http403]

HTTP 401 响应必须始终包含 `WWW-Authenticate` header，该 header 指示客户端如何进行身份验证。 HTTP 403 响应不包含 `WWW-Authenticate` header。

将使用哪种响应取决于认证方案。尽管可能正在使用多种认证方案，但只能使用一种方案来确定响应的类型。 **在确定响应类型时使用视图上设置的第一个认证类**。

请注意，当请求可以成功进行身份验证时，仍然可能会因为权限而被拒绝，在这种情况下，将始终使用 `403 Permission Denied` 响应，而不管身份验证方案如何。

## Apache mod_wsgi 特定的配置

请注意，如果使用 mod_wsgi 部署到 Apache，授权 header 默认情况下不会传递到 WSGI 应用程序，因为它假定认证将由 Apache 处理，而不是在应用程序级别处理。

如果您正在部署到 Apache 并使用任何基于非会话的身份验证，则需要明确配置 mod_wsgi 以将所需的 headers 传递给应用程序。这可以通过在适当的上下文中指定 `WSGIPassAuthorization` 指令并将其设置为 `'On'` 来完成。

``` python
# this can go in either server config, virtual host, directory or .htaccess
WSGIPassAuthorization On
```

---

# API 参考

## BasicAuthentication

该认证方案使用 HTTP Basic Authentication，并根据用户的用户名和密码进行签名。Basic Authentication 通常只适用于测试。

如果成功通过身份验证，`BasicAuthentication` 将提供以下凭据。

* `request.user` 是一个 Django `User` 实力.
* `request.auth` 是 `None`.

未经身份验证的响应被拒绝将导致 `HTTP 401 Unauthorized` 的响应和相应的 WWW-Authenticate header。例如：

``` python
WWW-Authenticate: Basic realm="api"
```

**注意：** 如果您在生产环境中使用 `BasicAuthentication`，则必须确保您的 API 仅可通过 `https` 访问。您还应该确保您的 API 客户端将始终在登录时重新请求用户名和密码，并且永远不会将这些详细信息存储到持久化存储中。

## TokenAuthentication

此认证方案使用简单的基于令牌的 HTTP 认证方案。令牌身份验证适用于 client-server 架构，例如本机桌面和移动客户端。

要使用 `TokenAuthentication` 方案，您需要将认证类配置为包含 `TokenAuthentication` ，并在 `INSTALLED_APPS` 设置中另外包含 `rest_framework.authtoken` ：

``` python
INSTALLED_APPS = (
    ...
    'rest_framework.authtoken'
)
```

---

**注意：** 确保在更改设置后运行 `manage.py migrate` 。 `rest_framework.authtoken` 应用程序提供 Django 数据库迁移。

---

您还需要为您的用户创建令牌。

``` python
from rest_framework.authtoken.models import Token

token = Token.objects.create(user=...)
print token.key
```

对于客户端进行身份验证，令牌密钥应包含在 `Authorization` HTTP header 中。关键字应以字符串文字 “Token” 为前缀，用空格分隔两个字符串。例如：

``` python
Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b
```

**注意：** 如果您想在 header 中使用不同的关键字（例如 `Bearer`），只需子类化 `TokenAuthentication` 并设置 `keyword` 类变量。

如果成功通过身份验证，`TokenAuthentication` 将提供以下凭据。

* `request.user` 是一个 Django `User` 实例.
* `request.auth` 是一个 `rest_framework.authtoken.models.Token` 实例.

未经身份验证的响应被拒绝将导致 `HTTP 401 Unauthorized` 的响应和相应的 WWW-Authenticate header。例如：

``` python
WWW-Authenticate: Token
```

`curl` 命令行工具可能对测试令牌认证的 API 有用。例如：

``` python
curl -X GET http://127.0.0.1:8000/api/example/ -H 'Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b'
```

---

**注意：** 如果您在生产中使用 `TokenAuthentication`，则必须确保您的 API 只能通过 `https` 访问。

---

#### 生成令牌

##### 通过使用信号

如果您希望每个用户都拥有一个自动生成的令牌，则只需捕捉用户的 `post_save` 信号即可。

``` python
from django.conf import settings
from django.db.models.signals import post_save
from django.dispatch import receiver
from rest_framework.authtoken.models import Token

@receiver(post_save, sender=settings.AUTH_USER_MODEL)
def create_auth_token(sender, instance=None, created=False, **kwargs):
    if created:
        Token.objects.create(user=instance)
```

请注意，您需要确保将此代码片段放置在已安装的 `models.py` 模块或 Django 启动时将导入的其他某个位置。

如果您已经创建了一些用户，则可以为所有现有用户生成令牌，例如：

``` python
from django.contrib.auth.models import User
from rest_framework.authtoken.models import Token

for user in User.objects.all():
    Token.objects.get_or_create(user=user)
```

##### 通过暴露一个 API 端点

使用 `TokenAuthentication` 时，您可能希望为客户提供一种机制，以获取给定用户名和密码的令牌。  REST framework 提供了一个内置的视图来支持这种行为。要使用它，请将 `obtain_auth_token` 视图添加到您的 URLconf 中：

``` python
from rest_framework.authtoken import views
urlpatterns += [
    url(r'^api-token-auth/', views.obtain_auth_token)
]
```

请注意，模式的 URL 部分可以是任何你想使用的。

当使用表单数据或 JSON 将有效的 `username` 和 `password` 字段发布到视图时， `obtain_auth_token` 视图将返回 JSON 响应：

``` python
{ 'token' : '9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b' }
```

请注意，缺省的 `obtain_auth_token` 视图显式使用 JSON 请求和响应，而不是使用你设置的默认的渲染器和解析器类。

默认情况下，没有权限或限制应用于 `obtain_auth_token` 视图。 如果您希望应用 throttling ，则需要重写视图类，并使用 `throttle_classes` 属性包含它们。

如果你需要自定义 `obtain_auth_token` 视图，你可以通过继承 `ObtainAuthToken` 视图类来实现，并在你的 url conf 中使用它。

例如，您可能会返回超出 `token` 值的其他用户信息：

``` python
from rest_framework.authtoken.views import ObtainAuthToken
from rest_framework.authtoken.models import Token
from rest_framework.response import Response

class CustomAuthToken(ObtainAuthToken):

    def post(self, request, *args, **kwargs):
        serializer = self.serializer_class(data=request.data,
                                           context={'request': request})
        serializer.is_valid(raise_exception=True)
        user = serializer.validated_data['user']
        token, created = Token.objects.get_or_create(user=user)
        return Response({
            'token': token.key,
            'user_id': user.pk,
            'email': user.email
        })
```

还有 `urls.py`:

``` python
urlpatterns += [
    url(r'^api-token-auth/', CustomAuthToken.as_view())
]
```


##### 使用 Django admin

也可以通过管理界面手动创建令牌。如果您使用的用户群很大，我们建议您对 `TokenAdmin` 类进行修补以根据需要对其进行定制，更具体地说，将 `user` 字段声明为 `raw_field`。

`your_app/admin.py`:

``` python
from rest_framework.authtoken.admin import TokenAdmin

TokenAdmin.raw_id_fields = ('user',)
```


#### 使用 Django manage.py 命令

从版本 3.6.4 开始，可以使用以下命令生成用户令牌：

``` python
./manage.py drf_create_token <username>
```

此命令将返回给定用户的 API 令牌，如果它不存在则创建它：

``` python
Generated token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b for user user1
```

如果您想重新生成令牌（例如，它已被泄漏），则可以传递一个附加参数：

``` python
./manage.py drf_create_token -r <username>
```


## SessionAuthentication

此认证方案使用 Django 的默认 session 后端进行认证。Session 身份验证适用于与您的网站在同一会话环境中运行的 AJAX 客户端。

如果成功通过身份验证，则 `SessionAuthentication` 会提供以下凭据。

* `request.user` 是一个 Django `User` 实例.
* `request.auth` 是 `None`.

未经身份验证的响应被拒绝将导致 `HTTP 403 Forbidden` 响应。

如果您在 SessionAuthentication 中使用 AJAX 风格的 API，则需要确保为任何 “不安全” 的 HTTP 方法调用（例如 `PUT`，`PATCH`，`POST` 或 `DELETE` 请求）包含有效的 CSRF 令牌。

**警告**: 创建登录页面时应该始终使用 Django 的标准登录视图。这将确保您的登录视图得到适当的保护。

REST framework 中的 CSRF 验证与标准 Django 略有不同，因为需要同时支持基于 session 和非基于 session 的身份验证。这意味着只有经过身份验证的请求才需要 CSRF 令牌，并且可以在没有 CSRF 令牌的情况下发送匿名请求。此行为不适用于应始终应用 CSRF 验证的登录视图。


## RemoteUserAuthentication

这种身份验证方案允许您将身份验证委托给您的 Web 服务器，该服务器设置 `REMOTE_USER` 环境变量。

要使用它，你必须在你的 `AUTHENTICATION_BACKENDS` 设置中有 `django.contrib.auth.backends.RemoteUserBackend` （或者一个子类）。默认情况下，`RemoteUserBackend` 为不存在的用户名创建 `User` 对象。要改变这个和其他行为，请参考 Django 文档。

如果成功通过身份验证，`RemoteUserAuthentication` 将提供以下凭据：

* `request.user` 是一个 Django `User` 实例.
* `request.auth` 是 `None`.

有关配置验证方法的信息，请参阅您的 Web 服务器的文档，例如：

* [Apache Authentication How-To](https://httpd.apache.org/docs/2.4/howto/auth.html)
* [NGINX (Restricting Access)](https://www.nginx.com/resources/admin-guide/#restricting_access)


# 自定义身份认证

要实现自定义身份验证方案，请继承 `BaseAuthentication` 并重写 `.authenticate(self, request)` 方法。如果认证成功，该方法应返回 `(user, auth)` 的二元组，否则返回 `None`。

在某些情况下，您可能想要从 `.authenticate()` 方法引发 `AuthenticationFailed` 异常而不是返回 `None`。

通常你应该采取的方法是：

* 如果不尝试认证，则返回 `None`。任何其他正在使用的身份验证方案仍将被检查。
* 如果尝试身份验证但失败了，请引发 `AuthenticationFailed` 异常。无论是否进行任何权限检查，都将立即返回错误响应，并且不再检查任何其他身份验证方案。

您也 *可以* 重写 `.authenticate_header(self, request)` 方法。如果实现，它应该返回一个字符串，该字符串将用作 `HTTP 401 Unauthorized` 响应中的 WWW-Authenticate header 的值。

如果未覆盖 `.authenticate_header()` 方法，那么当未经身份验证的请求被拒绝访问时，身份验证方案将返回 `HTTP 403 Forbidden` 响应。

---

**Note:** 当请求对象的 `.user` 或 `.auth` 属性调用您的自定义身份验证器时，您可能会看到 `AttributeError` 作为 `WrappedAttributeError` 被重新引发。这对于防止原始异常被外部属性访问所抑制是必要的。Python 不会识别 `AttributeError` 来自您的自定义身份验证器，而是会假设请求对象没有 `.user` 或 `.auth` 属性。这些错误应该由您的验证器修复或以其他方式处理。

---

## 举个栗子

下面的示例将根据名为 “X_USERNAME” 的自定义请求头中的用户名对任何传入请求进行身份验证。

``` python
from django.contrib.auth.models import User
from rest_framework import authentication
from rest_framework import exceptions

class ExampleAuthentication(authentication.BaseAuthentication):
    def authenticate(self, request):
        username = request.META.get('X_USERNAME')
        if not username:
            return None

        try:
            user = User.objects.get(username=username)
        except User.DoesNotExist:
            raise exceptions.AuthenticationFailed('No such user')

        return (user, None)
```


