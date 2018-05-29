> [官方原文链接](http://www.django-rest-framework.org/api-guide/exceptions/)  


# 异常



## REST framework 视图中的异常处理

REST framework 的视图处理各种异常，并返回适当的错误响应。

需要处理的异常情况有：

* 在 REST framework 内引发的 `APIException`的子类。
* Django 的 `Http404` 异常。
* Django 的 `PermissionDenied` 异常。

在每种情况下，REST framework 都会返回一个带有适当状态码和内容类型的响应。响应的主体将包含有关错误性质的其他细节。

大多数错误响应将包含响应正文中的关键 `detail`。

例如，以下请求：

```
DELETE http://api.example.com/foo/bar HTTP/1.1
Accept: application/json
```

可能会收到错误响应，指出在该资源上不允许使用 `DELETE` 方法：

```
HTTP/1.1 405 Method Not Allowed
Content-Type: application/json
Content-Length: 42

{"detail": "Method 'DELETE' not allowed."}
```

验证错误的处理方式稍有不同，但都是将字段名称作为响应中的关键字。如果验证错误不是特定于某个字段的，那么它将使用 “non_field_errors” 键，或者为 `NON_FIELD_ERRORS_KEY` setting 设置的字符串值。

示例验证错误可能如下所示：

```
HTTP/1.1 400 Bad Request
Content-Type: application/json
Content-Length: 94

{"amount": ["A valid integer is required."], "description": ["This field may not be blank."]}
```

## 自定义异常处理

你可以通过创建处理函数来实现自定义异常处理，该函数将 API 视图中引发的异常转换为响应对象。这使你可以控制 API 错误响应的样式。

该函数必须带有一对参数，第一个是要处理的异常，第二个是包含任何额外上下文（例如当前正在处理的视图）的字典。异常处理函数应该返回一个 `Response` 对象，或者如果无法处理异常，则返回 `None`。如果处理程序返回 `None`，那么异常将被重新抛出，Django 将返回一个标准的 HTTP 500 'server error' 响应。

例如，你可能希望确保所有错误响应都包含响应正文中的 HTTP 状态码，如下所示：

```
HTTP/1.1 405 Method Not Allowed
Content-Type: application/json
Content-Length: 62

{"status_code": 405, "detail": "Method 'DELETE' not allowed."}
```

为了改变响应的风格，你可以编写下面的自定义异常处理程序：

``` python
from rest_framework.views import exception_handler

def custom_exception_handler(exc, context):
    # Call REST framework's default exception handler first,
    # to get the standard error response.
    response = exception_handler(exc, context)

    # Now add the HTTP status code to the response.
    if response is not None:
        response.data['status_code'] = response.status_code

    return response
```

context 参数不被默认处理程序使用，但是如果异常处理程序需要更多信息，例如当前正在处理的视图（可以作为 `context['view']` 访问），则该参数可能很有用。

异常处理程序还必须使用 `EXCEPTION_HANDLER` setting key 在你的设置中进行配置。例如：

``` python
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'my_project.my_app.utils.custom_exception_handler'
}
```

如果未指定，则 `'EXCEPTION_HANDLER'` setting 默认为由 REST framework 提供的标准异常处理程序：

``` python
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'rest_framework.views.exception_handler'
}
```

请注意，异常处理程序只会根据由异常产生的响应调用。它不会用于视图直接返回的任何响应，例如在序列化验证失败时通用视图返回的 `HTTP_400_BAD_REQUEST` 响应。

---

# API 参考

## APIException

**签名:** `APIException()`

在 `APIView` 类或 `@api_view` 中引发的所有异常的基类。

要自定义异常，请继承 `APIException`，并在该类上设置 `.status_code`，`.default_detail` 和 `default_code` 属性。

例如，如果你的 API 依赖于可能无法访问的第三方服务，则可能需要为 "503 Service Unavailable" HTTP 响应码封装异常。你可以这样做：

``` python
from rest_framework.exceptions import APIException

class ServiceUnavailable(APIException):
    status_code = 503
    default_detail = 'Service temporarily unavailable, try again later.'
    default_code = 'service_unavailable'
```

#### 检查 API 异常

有许多不同的属性可用于检查 API 异常的状态。你可以使用它们为你的项目构建自定义异常处理。

可用的属性和方法有：

* `.detail` - 返回错误的文本描述。
* `.get_codes()` - 返回错误的代码标识符。
* `.get_full_details()` - 返回文本描述和代码标识符。

在大多数情况下，错误详情将是一个简单的 item：

``` bash
>>> print(exc.detail)
You do not have permission to perform this action.
>>> print(exc.get_codes())
permission_denied
>>> print(exc.get_full_details())
{'message':'You do not have permission to perform this action.','code':'permission_denied'}
```

在验证错误的情况下，错误详情将是 item 列表或字典：

``` bash
>>> print(exc.detail)
{"name":"This field is required.","age":"A valid integer is required."}
>>> print(exc.get_codes())
{"name":"required","age":"invalid"}
>>> print(exc.get_full_details())
{"name":{"message":"This field is required.","code":"required"},"age":{"message":"A valid integer is required.","code":"invalid"}}
```

## ParseError

**签名:** `ParseError(detail=None, code=None)`

在访问 `request.data` 时包含格式错误的数据则会引发此异常。

默认情况下，此异常会导致 HTTP 状态码 "400 Bad Request" 的响应。

## AuthenticationFailed

**签名:** `AuthenticationFailed(detail=None, code=None)`

当传入的请求包含不正确的身份验证时引发。

默认情况下，此异常会导致 HTTP 状态码 "401 Unauthenticated" 的响应，但也可能会导致 "403 Forbidden" 响应，具体取决于所使用的身份验证方案。

## NotAuthenticated

**签名:** `NotAuthenticated(detail=None, code=None)`

当未经身份验证的请求未通过权限检查时引发。

默认情况下，此异常会导致 HTTP 状态码 "401 Unauthenticated" 的响应，但也可能会导致 "403 Forbidden" 响应，具体取决于所使用的身份验证方案。

## PermissionDenied

**签名:** `PermissionDenied(detail=None, code=None)`

当经过身份验证的请求未通过权限检查时引发。

默认情况下，此异常会导致 HTTP 状态码 "403 Forbidden" 的响应。

## NotFound

**签名:** `NotFound(detail=None, code=None)`

当资源不存在于给定的 URL 时引发。这个异常相当于标准的 `Http404` Django 异常。

默认情况下，此异常会导致 HTTP 状态码为 "404 Not Found" 的响应。

## MethodNotAllowed

**签名:** `MethodNotAllowed(method, detail=None, code=None)`

当请求发生时，找不到视图上对应的处理方法时引发。

默认情况下，此异常会导致 HTTP 状态码为 "405 Method Not Allowed" 的响应。

## NotAcceptable

**签名:** `NotAcceptable(detail=None, code=None)`

当请求发生时，任何可用渲染器都不符合 `Accept` header 时引发。

默认情况下，此异常会导致 HTTP 状态码为 "406 Not Acceptable" 的响应。

## UnsupportedMediaType

**签名:** `UnsupportedMediaType(media_type, detail=None, code=None)`

如果在访问 `request.data` 时没有可以处理请求数据的内容类型的解析器，就会引发。

默认情况下，此异常会导致 HTTP 状态码 "415 Unsupported Media Type" 的响应。

## Throttled

**签名:** `Throttled(wait=None, detail=None, code=None)`

传入的请求未通过限流检查时引发。

默认情况下，此异常会导致 HTTP 状态码 "429 Too Many Requests" 的响应。

## ValidationError

**签名:** `ValidationError(detail, code=None)`

`ValidationError` 异常与其他 `APIException` 类略有不同：

* `detail` 参数是必需的，不是可选的。
* `detail` 参数可以是错误详情列表或字典，也可以是嵌套的数据结构。
* 按照惯例，你应该导入 serializers 模块并使用完全限定的 `ValidationError` 样式，以区别于 Django 内置的验证错误。例如： `raise serializers.ValidationError('This field must be an integer value.')`

`ValidationError` 类应该用于序列化类和字段验证以及验证器类。使用 `raise_exception` 关键字参数调用 `serializer.is_valid` 时也会引发此问题：

``` python
serializer.is_valid(raise_exception=True)
```

通用视图使用 `raise_exception=True` 标志，意味着你可以在 API 中全局覆盖验证错误响应的样式。为此，请使用自定义异常处理程序，如上所述。

默认情况下，此异常会导致 HTTP 状态码 "400 Bad Request" 的响应。

