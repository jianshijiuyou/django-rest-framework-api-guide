> [官方原文链接](http://www.django-rest-framework.org/api-guide/views/)

## 基于类的视图

REST framework  提供了一个 `APIView` 类，它继承于 Django 的 `View` 类。

`APIView` 类与不同的 `View` 类有所不同：  

* 传递给处理方法的 request 对象是 REST framework 的 `Request` 实例，而不是 Django 的 `HttpRequest` 实例。
* 处理方法可能返回 REST framework 的 `Response`，而不是 Django 的 `HttpResponse` 。该视图将管理内容协商，并在响应中设置正确的渲染器。
* 任何 `APIException` 异常都会被捕获并进行适当的响应。
* 传入的请求会进行认证，在请求分派给处理方法之前将进行适当的权限检查（允许或限制）。


像往常一样，使用 `API​​View` 类与使用常规 `View` 类非常相似，传入的请求被分派到适当的处理方法，如 `.get()` 或`.post()`。此外，可以在类上设置许多属性（AOP）。

举个栗子：
``` python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import authentication, permissions
from django.contrib.auth.models import User

class ListUsers(APIView):
    """
    列出系统中的所有用户

    * 需要 token 认证。
    * 只有 admin 用户才能访问此视图。
    """
    authentication_classes = (authentication.TokenAuthentication,)
    permission_classes = (permissions.IsAdminUser,)

    def get(self, request, format=None):
        """
        Return a list of all users.
        """
        usernames = [user.username for user in User.objects.all()]
        return Response(usernames)
```

> 注意：REST Framework 中的 `APIView`，`GenericAPIView`，各种 `Mixins` 和 `Viewsets` 包含许多方法和属性，刚开始要全部理解是比较困难的。这里除了文档，有一个 [Classy Django REST Framework](http://www.cdrf.co/) 资源，它提供了一个可以在线浏览的参照，包含所有属性和方法。

### API 策略属性（policy attributes）

以下属性用于增加扩展视图的功能，AOP。

.renderer_classes  
设置渲染器

.parser_classes  
设置解析器

.authentication_classes  
设置认证器

.throttle_classes

.permission_classes  
设置权限验证器

.content_negotiation_class

### API 策略实例方法（policy instantiation methods）

以下策略实例方法通常不需要我们重写。

.get_renderers(self)

.get_parsers(self)

.get_authenticators(self)

.get_throttles(self)

.get_permissions(self)

.get_content_negotiator(self)

.get_exception_handler(self)


### API 策略实现方法（policy implementation methods）

在分派到处理方法之前会调用以下方法。

.check_permissions(self, request)

.check_throttles(self, request)

.perform_content_negotiation(self, request, force=False)


### 方法调度

以下方法由视图的 `.dispatch()` 方法直接调用。它们在调用处理方法（`.get()`, `.post()`, `put()`, `patch()` 和 `.delete()`）之前或者之后被调用。

**.initial(self, request, \*args, \*\*kwargs)**

用于执行处理方法被调用之前需要的任何操作。此方法用于强制执行权限和限流，并执行内容协商。

**.handle_exception(self, exc)**

处理方法抛出的任何异常都将传递给此方法，该方法返回一个 `Response` 实例，或者重新引发异常。

默认实现处理 `rest_framework.exceptions.APIException` 的任何子类，以及 Django 的 `Http404` 和`PermissionDenied` 异常，并返回相应的错误响应。

如果需要自定义 API 返回的错误响应，应该重写此方法。


**.initialize_request(self, request, \*args, \*\*kwargs)**

确保传递给处理方法的请求对象是 `Request` 的一个实例，而不是通常的 Django `HttpRequest`。

通常不需要重写此方法。

**.finalize_response(self, request, response, \*args, \*\*kwargs)**

确保从处理方法返回的任何 `Response` 对象将被呈现为正确的内容类型，这由内容协商确定。

通常不需要重写此方法。



## 基于方法的视图

REST framework 也允许使用基于函数的视图。它提供了一套简单的装饰器来包装你的函数视图，以确保它们接收 `Request`（而不是 Django `HttpRequest`）实例并允许它们返回 `Response`（而不是 Django `HttpResponse`），并允许你配置该请求的处理方式。

### @api_view()

签名：`@api_view(http_method_names=['GET'])`

`api_view` 是一个装饰器，用 `http_method_names` 来设置视图允许响应的 HTTP 方法列表，举个栗子，编写一个简单的视图，手动返回一些数据。

``` python
from rest_framework.decorators import api_view

@api_view()
def hello_world(request):
    return Response({"message": "Hello, world!"})
```

该视图将使用 `settings` 中指定的默认渲染器，解析器，认证类等。

默认情况下，只有 `GET` 方法会被接受。其他方法将以 `"405 Method Not Allowed"` 进行响应。要改变这种行为，请指定视图允许的方法，如下所示：

``` python
@api_view(['GET', 'POST'])
def hello_world(request):
    if request.method == 'POST':
        return Response({"message": "Got some data!", "data": request.data})
    return Response({"message": "Hello, world!"})
```


### API 策略装饰器 (policy decorators)

为了覆盖默认设置，REST framework 提供了一系列可以添加到视图中的附加装饰器。这些必须在 `@api_view` 装饰器之后（下方）。例如，要创建一个使用 `throttle` 来确保它每天只能由特定用户调用一次的视图，请使用 `@throttle_classes` 装饰器，传递一个 `throttle` 类列表：

``` python
from rest_framework.decorators import api_view, throttle_classes
from rest_framework.throttling import UserRateThrottle

class OncePerDayUserThrottle(UserRateThrottle):
        rate = '1/day'

@api_view(['GET'])
@throttle_classes([OncePerDayUserThrottle])
def view(request):
    return Response({"message": "Hello for today! See you tomorrow!"})
```

这些装饰器对应于 `APIView` 上设置的策略属性。

可用的装饰器有：

 * `@renderer_classes(...)`
 * `@parser_classes(...)`
 * `@authentication_classes(...)`
 * `@throttle_classes(...)`
 * `@permission_classes(...)`


每个装饰器都有一个参数，它必须是一个类列表或者一个类元组。


### View schema 装饰器

要覆盖函数视图的默认 模式生成（schema generation），可以使用 `@schema` 装饰器。这必须在 `@api_view` 装饰器之后（下方）。例如：
``` python
from rest_framework.decorators import api_view, schema
from rest_framework.schemas import AutoSchema

class CustomAutoSchema(AutoSchema):
    def get_link(self, path, method, base_url):
        # override view introspection here...

@api_view(['GET'])
@schema(CustomAutoSchema())
def view(request):
    return Response({"message": "Hello for today! See you tomorrow!"})
```

该装饰器将采用一个 `AutoSchema` 实例，一个 `AutoSchema` 子类实例或 `ManualSchema` 实例，如[ Schemas 文档](http://www.django-rest-framework.org/api-guide/schemas/)（先放官链）中所述。您也可以传 `None` 以从 模式生成（schema generation） 中排除视图。

``` python
@api_view(['GET'])
@schema(None)
def view(request):
    return Response({"message": "Will not appear in schema!"})
```
