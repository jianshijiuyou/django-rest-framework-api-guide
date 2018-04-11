> [官方原文链接](http://www.django-rest-framework.org/api-guide/content-negotiation/)  

# 内容协商


内容协商是基于客户端或服务器偏好选择多种可能的表示之一以返回客户端的过程。

## 确定接受的渲染器

REST framework 根据可用的渲染器，每个渲染器的优先级以及客户端的 `Accept:` header，使用简单的内容协商风格来确定应将哪些媒体类型返回给客户端。所使用的风格部分由客户端驱动，部分由服务器驱动。

1. 更具体的媒体类型优先于较不特定的媒体类型。
2. 如果多种媒体类型具有相同的特性，则优先根据为给定视图配置的渲染器排序。

例如，给出以下 `Accept` header:

``` python
application/json; indent=4, application/json, application/yaml, text/html, */*
```

每种给定媒体类型的优先级为：

* `application/json; indent=4`
* `application/json`, `application/yaml` 和 `text/html`
* `*/*`

如果所请求的视图仅用 `YAML` 和 `HTML` 的渲染器配置，则 REST framework 将选择 `renderer_classes` 列表或 `DEFAULT_RENDERER_CLASSES` 设置中首先列出的渲染器。


---

**注意**:  确定偏好时，REST framework 不会考虑 "q" 值。使用 "q" 值会对缓存产生负面影响，作者认为这是对内容协商的一种不必要和过于复杂的方法。

---

# 自定义内容协商

你不太可能希望为 REST framework 提供自定义内容协商方案，但如果需要，你可以这样做。要实现自定义内容协商方案，请覆盖 `BaseContentNegotiation`。

REST framework 的内容协商类处理选择适当的请求解析器和适当的响应渲染器，因此你应该实现 `.select_parser(request, parsers)` 和 `.select_renderer(request, renderers, format_suffix)` 方法。

`select_parser()` 方法应从可用解析器列表中返回一个解析器实例，如果没有任何解析器可以处理传入请求，则返回 `None`。

`select_renderer()` 方法应该返回（渲染器实例，媒体类型）的二元组，或引发 `NotAcceptable` 异常。

## 举个栗子

以下是自定义内容协商类，它在选择适当的解析器或渲染器时会忽略客户端请求。

``` python
from rest_framework.negotiation import BaseContentNegotiation

class IgnoreClientContentNegotiation(BaseContentNegotiation):
    def select_parser(self, request, parsers):
        """
        Select the first parser in the `.parser_classes` list.
        """
        return parsers[0]

    def select_renderer(self, request, renderers, format_suffix):
        """
        Select the first renderer in the `.renderer_classes` list.
        """
        return (renderers[0], renderers[0].media_type)
```

## 设置内容协商

默认内容协商类可以使用 `DEFAULT_CONTENT_NEGOTIATION_CLASS` setting 全局设置。例如，以下设置将使用我们的示例 `IgnoreClientContentNegotiation` 类。

``` python
REST_FRAMEWORK = {
    'DEFAULT_CONTENT_NEGOTIATION_CLASS': 'myapp.negotiation.IgnoreClientContentNegotiation',
}
```

你还可以使用 `API​​View` 基于类的视图设置用于单个视图或视图集的内容协商。

``` python
from myapp.negotiation import IgnoreClientContentNegotiation
from rest_framework.response import Response
from rest_framework.views import APIView

class NoNegotiationView(APIView):
    """
    An example view that does not perform content negotiation.
    """
    content_negotiation_class = IgnoreClientContentNegotiation

    def get(self, request, format=None):
        return Response({
            'accepted media type': request.accepted_renderer.media_type
        })
```