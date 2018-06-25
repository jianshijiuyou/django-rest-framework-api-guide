> [官方原文链接](http://www.django-rest-framework.org/api-guide/throttling/)  

# 限流（Throttling）


限流与权限类似，因为它确定是否应该授权请求。 限流阀指示临时状态，并用于控制客户端可以对API进行的请求速率。

与权限一样，可能会使用多种限流方式。你的 API 可能对未经身份验证的请求进行限流，对经过身份验证的请求限流较少。

如果你需要对 API 的不同部分使用不同的限流策略，由于某些服务特别占用资源，你可能想要使用同时有多种限流策略的另一种方案。

如果你想要同时实现爆发限流率和持续限流率，也可以使用多个限流阀。例如，你可能希望将用户限制为每分钟最多 60 个请求，并且每天最多 1000 个请求。

限流阀不一定只限制请求频率。例如，存储服务可能还需要对带宽进行限制，而付费数据服务可能希望对正在访问的某些记录进行限制。

## 如何确定限流

与权限和身份验证一样，REST framework 中的限流始终定义为类的列表。

在运行视图的主体之前，会检查列表中的每个限流阀。如果任何限流检查失败，将引发一个 `exceptions.Throttled` 异常，并且该视图的主体将不会再执行。

## 设置限流策略

可以使用 `DEFAULT_THROTTLE_CLASSES` 和 `DEFAULT_THROTTLE_RATES` setting 全局设置默认限流策略。例如：

``` python
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': (
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ),
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',
        'user': '1000/day'
    }
}
```

`DEFAULT_THROTTLE_RATES` 中使用的频率描述可能包括 `second`，`minute` ，`hour` 或 `day` 作为限流期。

你还可以使用基于 `APIView` 类的视图，在每个视图或每个视图集的基础上设置限流策略。

``` python
from rest_framework.response import Response
from rest_framework.throttling import UserRateThrottle
from rest_framework.views import APIView

class ExampleView(APIView):
    throttle_classes = (UserRateThrottle,)

    def get(self, request, format=None):
        content = {
            'status': 'request was permitted'
        }
        return Response(content)
```

或者在基于 `@api_view` 装饰器的函数视图上设置。

``` python
@api_view(['GET'])
@throttle_classes([UserRateThrottle])
def example_view(request, format=None):
    content = {
        'status': 'request was permitted'
    }
    return Response(content)
```

## 如何识别客户端

`X-Forwarded-For` HTTP header 和 `REMOTE_ADDR` WSGI 变量用于唯一标识用于限流的客户端 IP 地址。如果存在 `X-Forwarded-For` header ，则会使用它，否则将使用 WSGI 环境中的 `REMOTE_ADDR` 变量的值。

如果你需要严格标识唯一的客户端 IP 地址，则需要先通过设置 `NUM_PROXIES` setting 来配置 API 运行的应用代理的数量。该设置应该是一个零或更大的整数。如果设置为非零，则一旦任何应用程序代理 IP 地址首先被排除，客户端 IP 将被标识为 `X-Forwarded-For` header 中的最后一个 IP 地址。如果设置为零，则 `REMOTE_ADDR` 值将始终用作识别 IP 地址。

重要的是要理解，如果你配置了 `NUM_PROXIES` 设置，那么在一个唯一的 NAT 的网关后面的所有客户端将被当作一个单独的客户机来对待。

关于 `X-Forwarded-For` header 如何工作以及识别远程客户端 IP 的更多内容可以在[这里找到](http://oxpedia.org/wiki/index.php?title=AppSuite:Grizzly#Multiple_Proxies_in_front_of_the_cluster)。

## 设置缓存

REST framework 提供的限流类使用 Django 的缓存后端。你应该确保你已经设置了适当的缓存 setting 。对于简单的设置，`LocMemCache` 后端的默认值应该没问题。有关更多详细信息，请参阅 [Django 的缓存文档](https://docs.djangoproject.com/en/stable/topics/cache/#setting-up-the-cache)。

如果你需要使用 `'default'` 以外的缓存，则可以通过创建自定义限流类并设置 `cache` 属性来实现。例如：

``` python
class CustomAnonRateThrottle(AnonRateThrottle):
    cache = get_cache('alternate')
```

您需要记住还要在 `'DEFAULT_THROTTLE_CLASSES'` settings key 中设置自定义的限流类，或者使用 `throttle_classes` 视图属性。

---

# API 参考

## AnonRateThrottle

`AnonRateThrottle` 将永远限制未认证的用户。通过传入请求的 IP 地址生成一个唯一的密钥来进行限制。

允许的请求频率由以下之一决定（按优先顺序）。

* 类的 `rate` 属性，可以通过继承 `AnonRateThrottle` 并设置属性来提供。
* `DEFAULT_THROTTLE_RATES['anon']` 设置.

如果你想限制未知来源的请求频率，`AnonRateThrottle` 是合适的。

## UserRateThrottle

`UserRateThrottle` 通过 API 将用户请求限制为给定的请求频率。用户标识用于生成一个唯一的密钥来加以限制。未经身份验证的请求将回退到使用传入请求的 IP 地址生成一个唯一的密钥来进行限制。

允许的请求频率由以下之一决定（按优先顺序）。

* 类的 `rate` 属性，可以通过继承 `UserRateThrottle` 并设置属性来提供。
* `DEFAULT_THROTTLE_RATES['user']` 设置.

一个 API 可能同时具有多个 `UserRateThrottles`。为此，请继承 `UserRateThrottle` 并为每个类设置一个唯一的“范围”。

例如，多个用户限流率可以通过使用以下类来实现......

``` python
class BurstRateThrottle(UserRateThrottle):
    scope = 'burst'

class SustainedRateThrottle(UserRateThrottle):
    scope = 'sustained'
```

...和以下设置。

``` python
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': (
        'example.throttles.BurstRateThrottle',
        'example.throttles.SustainedRateThrottle'
    ),
    'DEFAULT_THROTTLE_RATES': {
        'burst': '60/min',
        'sustained': '1000/day'
    }
}
```

如果希望对每个用户进行简单的全局速率限制，那么 `UserRateThrottle` 是合适的。

## ScopedRateThrottle

`ScopedRateThrottle` 类可用于限制对 API 特定部分的访问。只有当正在访问的视图包含 `.throttle_scope` 属性时才会应用此限制。然后通过将请求的 “范围” 与唯一的用户标识或 IP 地址连接起来形成唯一的限流密钥。

允许的请求频率由 `DEFAULT_THROTTLE_RATES` setting 使用请求 “范围” 中的一个键确定。

例如，给出以下视图...

``` python
class ContactListView(APIView):
    throttle_scope = 'contacts'
    ...

class ContactDetailView(APIView):
    throttle_scope = 'contacts'
    ...

class UploadView(APIView):
    throttle_scope = 'uploads'
    ...
```

...和以下设置。

``` python
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': (
        'rest_framework.throttling.ScopedRateThrottle',
    ),
    'DEFAULT_THROTTLE_RATES': {
        'contacts': '1000/day',
        'uploads': '20/day'
    }
}
```

用户对 `ContactListView` 或 `ContactDetailView` 的请求将被限制为每天 1000 次。用户对 `UploadView` 的请求将被限制为每天 20 次。

---

# 自定义限流

要自定义限流，请继承 `BaseThrottle` 类并实现 `.allow_request(self, request, view)` 方法。如果请求被允许，该方法应该返回 `True`，否则返回 `False`。

或者，你也可以重写 `.wait()` 方法。如果实现，`.wait()` 应该返回建议的秒数，在尝试下一次请求之前等待，或者返回 `None`。如果 `.allow_request()` 先前已经返回 `False`，则只会调用 `.wait()` 方法。

如果 `.wait()` 方法被实现并且请求受到限制，那么 `Retry-After` header 将包含在响应中。

## 举个栗子

以下是限流的一个示例，随机地控制每 10 次请求中的 1 次。

``` python
import random

class RandomRateThrottle(throttling.BaseThrottle):
    def allow_request(self, request, view):
        return random.randint(1, 10) != 1
```
