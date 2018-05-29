> [官方原文链接](http://www.django-rest-framework.org/api-guide/status-codes/)  


# 状态码



不建议在你的响应中使用裸露（直接使用数字）的状态码。 REST framework 包含一组命名常量，你可以使用它们使代码更加清晰易读。

``` python
from rest_framework import status
from rest_framework.response import Response

def empty_view(self):
    content = {'please move along': 'nothing to see here'}
    return Response(content, status=status.HTTP_404_NOT_FOUND)
```

下面列出了 `status` 模块中包含的全部 HTTP 状态码。

该模块还包含一组帮助函数，用于测试状态码是否在给定范围内。

``` python
from rest_framework import status
from rest_framework.test import APITestCase

class ExampleTestCase(APITestCase):
    def test_url_root(self):
        url = reverse('index')
        response = self.client.get(url)
        self.assertTrue(status.is_success(response.status_code))
```


## 信息 - 1xx

这类状态码表示临时响应。默认情况下，REST framework 中没有使用 1xx 状态码。

```
HTTP_100_CONTINUE
HTTP_101_SWITCHING_PROTOCOLS 
```

## 成功 - 2xx

这类状态码表明客户端的请求已被成功接收，理解和接受。

```
HTTP_200_OK
HTTP_201_CREATED
HTTP_202_ACCEPTED
HTTP_203_NON_AUTHORITATIVE_INFORMATION
HTTP_204_NO_CONTENT
HTTP_205_RESET_CONTENT
HTTP_206_PARTIAL_CONTENT
HTTP_207_MULTI_STATUS
```

## 重定向 - 3xx

这类状态码表明用户代理需要采取进一步行动来完成请求。

```
HTTP_300_MULTIPLE_CHOICES
HTTP_301_MOVED_PERMANENTLY
HTTP_302_FOUND
HTTP_303_SEE_OTHER
HTTP_304_NOT_MODIFIED
HTTP_305_USE_PROXY
HTTP_306_RESERVED
HTTP_307_TEMPORARY_REDIRECT
```

## 客户端错误 - 4xx

4xx 状态码适用于客户端出现错误的情况。除了响应 HEAD 请求之外，服务器应该包含一个错误情况说明的实体，以及它是临时的还是永久的。

```
HTTP_400_BAD_REQUEST
HTTP_401_UNAUTHORIZED
HTTP_402_PAYMENT_REQUIRED
HTTP_403_FORBIDDEN
HTTP_404_NOT_FOUND
HTTP_405_METHOD_NOT_ALLOWED
HTTP_406_NOT_ACCEPTABLE
HTTP_407_PROXY_AUTHENTICATION_REQUIRED
HTTP_408_REQUEST_TIMEOUT
HTTP_409_CONFLICT
HTTP_410_GONE
HTTP_411_LENGTH_REQUIRED
HTTP_412_PRECONDITION_FAILED
HTTP_413_REQUEST_ENTITY_TOO_LARGE
HTTP_414_REQUEST_URI_TOO_LONG
HTTP_415_UNSUPPORTED_MEDIA_TYPE
HTTP_416_REQUESTED_RANGE_NOT_SATISFIABLE
HTTP_417_EXPECTATION_FAILED
HTTP_422_UNPROCESSABLE_ENTITY
HTTP_423_LOCKED
HTTP_424_FAILED_DEPENDENCY
HTTP_428_PRECONDITION_REQUIRED
HTTP_429_TOO_MANY_REQUESTS
HTTP_431_REQUEST_HEADER_FIELDS_TOO_LARGE
HTTP_451_UNAVAILABLE_FOR_LEGAL_REASONS
```

## 服务器错误 - 5xx

5xx 表示服务器意识到其发生错误或无法执行请求的情况。除了响应 HEAD 请求之外，服务器应该包含一个错误情况说明的实体，以及它是临时的还是永久的。

```
HTTP_500_INTERNAL_SERVER_ERROR
HTTP_501_NOT_IMPLEMENTED
HTTP_502_BAD_GATEWAY
HTTP_503_SERVICE_UNAVAILABLE
HTTP_504_GATEWAY_TIMEOUT
HTTP_505_HTTP_VERSION_NOT_SUPPORTED
HTTP_507_INSUFFICIENT_STORAGE
HTTP_511_NETWORK_AUTHENTICATION_REQUIRED
```

## 帮助函数

以下帮助函数可用于识别响应代码的类别。

``` python
is_informational()  # 1xx
is_success()        # 2xx
is_redirect()       # 3xx
is_client_error()   # 4xx
is_server_error()   # 5xx
```
