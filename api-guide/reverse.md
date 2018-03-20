> [官方原文链接](http://www.django-rest-framework.org/api-guide/reverse/)  


# 返回 URL


通常，从 Web API（例如 `http://example.com/foobar`）返回绝对 URI 可能是更好的做法，而不是返回相对 URI，例如 `/foobar`。

这样做的好处有：

* 它更明确。
* 它为你的 API 客户端留下更少的工作。
* 当字符串在诸如 JSON 这样的表示中没有本地 URI 类型时，它的含义是没有歧义的。
* 它使得使用超链接标记 HTML 表示等事情变得很容易。

REST framework 提供了两个实用函数，可以更简单地从 Web API 返回绝对 URI。

使用它们不是必须的，但是如果你这样做，自描述 API 将能够自动为你输出超链接，这使得浏览 API 变得更容易。

## reverse

**签名:** `reverse(viewname, *args, **kwargs)`

具有与 `django.urls.reverse` 相同的行为，除了它返回一个完全限定的 URL，使用 request 来确定主机和端口。

你应该将 request 作为关键字参数包含在该函数中，例如：

``` python
from rest_framework.reverse import reverse
from rest_framework.views import APIView
from django.utils.timezone import now

class APIRootView(APIView):
    def get(self, request):
        year = now().year
        data = {
            ...
            'year-summary-url': reverse('year-summary', args=[year], request=request)
        }
        return Response(data)
```

## reverse_lazy

**签名:** `reverse_lazy(viewname, *args, **kwargs)`

具有与 `django.urls.reverse_lazy` 相同的行为，除了它返回一个完全限定的 URL，使用 request 来确定主机和端口。

与 `reverse` 函数一样，你应该将 `request` 作为关键字参数包含在函数中，例如：

``` python
api_root = reverse_lazy('api-root', request=request)
```