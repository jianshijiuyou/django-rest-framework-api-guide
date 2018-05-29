> [官方原文链接](http://www.django-rest-framework.org/api-guide/versioning/)  




# 版本控制

API 版本控制允许你更改不同客户端之间的行为。 REST framework 提供了许多不同的版本控制方案。

版本控制由传入的客户端请求决定，可能基于请求 URL 或请求 header。

有几种有效的方法来处理版本控制。非版本化的系统也可能是合适的，特别是如果你正在为超出控制范围的多个客户端的非常长期的系统进行工程设计。

## 使用 REST framework 进行版本控制

当启用 API 版本控制时， `request.version` 属性将包含一个对应于传入客户端请求的版本的字符串。

默认情况下，版本控制未启用，`request.version` 将始终返回 `None`。

#### 基于版本的变化行为

如何改变 API 的行为取决于你，但你可能通常需要的一个示例是在新版本中切换到不同的序列化样式。例如：

``` python
def get_serializer_class(self):
    if self.request.version == 'v1':
        return AccountSerializerVersion1
    return AccountSerializer
```

#### 反向解析版本化 API 的 URL

REST framework 包含的 `reverse` 函数与版本控制方案相关联。你需要确保将当前请求包含为关键字参数，如下所示。

``` python
from rest_framework.reverse import reverse

reverse('bookings-list', request=request)
```

上述功能将应用适合请求版本的任何 URL 转换。例如：

* 如果正在使用 `NamespacedVersioning`，并且 API 版本为 'v1'，则使用的 URL lookup 将为 `'v1:bookings-list'`，可能会解析为像 `http://example.org/v1/bookings/` 这样的 URL。
* 如果正在使用 `QueryParameterVersioning`，并且 API 版本为 `1.0`，则返回的 URL 可能与 `http://example.org/bookings/?version=1.0` 类似

#### 版本化的 API 和超链接序列化类

将超链接序列化样式与基于 URL 的版本控制方案一起使用时，请确保将请求作为上下文包含在序列化类中。

``` python
def get(self, request):
    queryset = Booking.objects.all()
    serializer = BookingsSerializer(queryset, many=True, context={'request': request})
    return Response({'all_bookings': serializer.data})
```

这样做将允许任何返回的 URL 包含合适的版本。

## 配置版本控制方案

版本控制方案由 `DEFAULT_VERSIONING_CLASS` setting key 定义。

``` python
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.NamespaceVersioning'
}
```

除非明确设置，否则 `DEFAULT_VERSIONING_CLASS` 的值将为 `None`。在这种情况下，`request.version` 属性将始终返回 `None`。

你还可以在单​​个视图上设置版本控制方案。通常，您不需要这样做，因为全局使用单个版本控制方案更有意义。如果你确实需要这样做，请使用 `versioning_class` 属性。

``` python
class ProfileList(APIView):
    versioning_class = versioning.QueryParameterVersioning
```

#### 其他版本设置

以下 settings key 也用于控制版本控制：

* `DEFAULT_VERSION`. 当没有版本信息存在时，用于 `request.version` 的值。默认为 `None`。
* `ALLOWED_VERSIONS`. 如果设置，则此值将限制版本控制方案可能返回的版本集，如果提供的版本不在此集中，则会引发错误。请注意，用于 `DEFAULT_VERSION` 设置的值始终被认为是 `ALLOWED_VERSIONS` 集的一部分（除非它是 `None`）。默认为 `None`。
* `VERSION_PARAM`. 应该用于任何版本控制参数的字符串，例如媒体类型或 URL 查询参数。默认为 `'version'`。

你还可以通过定义自己的版本控制方案并使用 `default_version`，`allowed_versions` 和 `version_param` 类变量，在每个视图或每个视图集的基础上设置版本控制类以及这三个值。例如，如果你想使用 `URLPathVersioning`：

``` python
from rest_framework.versioning import URLPathVersioning
from rest_framework.views import APIView

class ExampleVersioning(URLPathVersioning):
    default_version = ...
    allowed_versions = ...
    version_param = ...

class ExampleView(APIVIew):
    versioning_class = ExampleVersioning
```

---

# API 参考

## AcceptHeaderVersioning

此方案要求客户端将版本指定为 `Accept` header 中媒体类型的一部分。该版本作为媒体类型参数包含在内，它补充了主要媒体类型。

这是一个使用 accept header 版本风格的示例 HTTP 请求。

```
GET /bookings/ HTTP/1.1
Host: example.com
Accept: application/json; version=1.0
```

在上面的示例请求中 `request.version` 属性将返回字符串 `'1.0'`。

基于 Accept header 的版本控制通常被认为是最佳实践，尽管其他样式可能更适合你的客户端需求。

#### 使用 accept header 和 vendor media type

严格地说，`json` media type 不能被指定为包含附加参数。如果你正在构建精心指定的公共 API，则可以考虑使用vendor media type。为此，请将你的渲染器配置为使用基于 JSON 的渲染器和自定义 media type：

``` python
class BookingsAPIRenderer(JSONRenderer):
    media_type = 'application/vnd.megacorp.bookings+json'
```

客户端请求现在看起来像这样：

``` python
GET /bookings/ HTTP/1.1
Host: example.com
Accept: application/vnd.megacorp.bookings+json; version=1.0
```

## URLPathVersioning

该方案要求客户端将版本指定为 URL 路径的一部分。

``` python
GET /v1/bookings/ HTTP/1.1
Host: example.com
Accept: application/json
```

你的 URL conf 必须包含一个与 `'version'` 关键字参数相匹配的模式，以便版本控制方案可以使用此信息。

``` python
urlpatterns = [
    url(
        r'^(?P<version>(v1|v2))/bookings/$',
        bookings_list,
        name='bookings-list'
    ),
    url(
        r'^(?P<version>(v1|v2))/bookings/(?P<pk>[0-9]+)/$',
        bookings_detail,
        name='bookings-detail'
    )
]
```

## NamespaceVersioning

对于客户端来说，这个方案与 `URLPathVersioning` 相同。唯一的区别是，它是如何在 Django 应用程序中配置的，因为它使用 URL 命名空间而不是 URL 关键字参数。

```
GET /v1/something/ HTTP/1.1
Host: example.com
Accept: application/json
```

使用此方案，`request.version` 属性是根据与传入请求路径匹配的 `namespace` 确定的。

在下面的例子中，我们给出了一组视图两个不同的可能的 URL 前缀，每个前缀在不同的命名空间下：

``` python
# bookings/urls.py
urlpatterns = [
    url(r'^$', bookings_list, name='bookings-list'),
    url(r'^(?P<pk>[0-9]+)/$', bookings_detail, name='bookings-detail')
]

# urls.py
urlpatterns = [
    url(r'^v1/bookings/', include('bookings.urls', namespace='v1')),
    url(r'^v2/bookings/', include('bookings.urls', namespace='v2'))
]
```

如果你只需要简单的版本控制方案，那么 `URLPathVersioning` 和 `NamespaceVersioning` 都是可以的。 `URLPathVersioning`方法可能更适合于小型项目，`NamespaceVersioning` 可能更容易管理大型项目。

## HostNameVersioning

hostname 版本控制方案要求客户端将请求的版本指定为 URL 中 hostname 的一部分。

例如，以下是对 `http://v1.example.com/bookings/` URL 的 HTTP 请求：

```
GET /bookings/ HTTP/1.1
Host: v1.example.com
Accept: application/json 
```

默认情况下，这个实现期望 hostname 匹配这个简单的正则表达式：

```
^([a-zA-Z0-9]+)\.[a-zA-Z0-9]+\.[a-zA-Z0-9]+$
```

请注意，第一组用括号括起来，表示这是 hostname 的匹配部分。

`HostNameVersioning` 方案在调试模式下可能会很笨拙，因为你通常会访问诸如 `127.0.0.1` 的原始 IP 地址。

如果你有要求根据版本将传入请求路由到不同的服务器，那么基于 hostname 的版本控制就会特别有用，因为你可以为不同的 API 版本配置不同的 DNS 记录。

## QueryParameterVersioning

该方案是一种简单的风格，其中包含版本作为 URL 中的查询参数。例如：

```
GET /something/?version=0.1 HTTP/1.1
Host: example.com
Accept: application/json 
```

---

# 自定义版本控制方案

要实现自定义版本控制方案，请继承 `BaseVersioning`并覆盖 `.determine_version` 方法。

## 举个栗子

以下示例使用自定义的 `X-API-Version` header 来确定所请求的版本。

``` python
class XAPIVersionScheme(versioning.BaseVersioning):
    def determine_version(self, request, *args, **kwargs):
        return request.META.get('HTTP_X_API_VERSION', None)
```

如果你的版本控制方案基于请求 URL，则还需要更改版本化 URL 的确定方式。为了做到这一点，你应该重写类的 `.reverse()`方法。有关示例，请参阅源代码。
