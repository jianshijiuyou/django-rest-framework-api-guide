> [官方原文链接](http://www.django-rest-framework.org/api-guide/renderers/)


## 渲染

REST framework 包含许多内置的渲染器类，允许您使用各种 media type 返回响应。同时也支持自定义渲染器。

### 如何确定使用哪个渲染器

视图的渲染器集合始终被定义为类列表。当调用视图时，REST framework 将对请求内容进行分析，并确定最合适的渲染器以满足请求。内容分析的基本过程包括检查请求的 `Accept` header，以确定它在响应中期望的 media type。或者，用 URL 上的格式后缀明确表示。例如，URL `http://example.com/api/users_count.json` 可能始终返回 JSON 数据。


### 设置渲染器

可以使用 `DEFAULT_RENDERER_CLASSES` 设置全局的默认渲染器集。例如，以下设置将使用JSON作为主要 media type，并且还包含自描述 API。


``` python
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': (
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',
    )
}
```

还可以使用基于 `API​​View` 的视图类来设置单个视图或视图集的渲染器。

``` python
from django.contrib.auth.models import User
from rest_framework.renderers import JSONRenderer
from rest_framework.response import Response
from rest_framework.views import APIView

class UserCountView(APIView):
    """
    A view that returns the count of active users in JSON.
    """
    renderer_classes = (JSONRenderer, )

    def get(self, request, format=None):
        user_count = User.objects.filter(active=True).count()
        content = {'user_count': user_count}
        return Response(content)
```

或者是在基于 `@api_view` 装饰器的函数视图上设置。

``` python
@api_view(['GET'])
@renderer_classes((JSONRenderer,))
def user_count_view(request, format=None):
    """
    A view that returns the count of active users in JSON.
    """
    user_count = User.objects.filter(active=True).count()
    content = {'user_count': user_count}
    return Response(content)
```

### 渲染器类的优先级

在为 API 指定渲染器类时，需要考虑它们处理每种媒体类型时的优先级，这点很重要。如果客户端没有指定接受数据的表现形式，例如发送 `Accept：*/*` header，或者根本不包含 `Accept` header，则 REST framework 将选择列表中的第一个渲染器用于响应。

例如，如果你的 API 提供 JSON 响应和可浏览的 HTML API，则可能需要将 `JSONRenderer` 作为默认渲染器，以便将 `JSON` 响应发送给未指定 `Accept` header 的客户端。

如果你的 API 包含可根据请求同时处理常规网页和 API 响应的视图，那么你可以考虑将 `TemplateHTMLRenderer` 设置为默认渲染器，以便与发送 broken accept headers 的老式浏览器很好地配合使用。

## API 参考

### JSONRenderer

使用 utf-8 编码将请求数据呈现为 `JSON`。

请注意，默认风格包含 unicode 字符，并使用紧凑风格呈现（没有多余的空白）响应：

``` python
{"unicode black star":"★","value":999}
```

客户端可能还会包含 “缩进” media type 参数，在这种情况下，返回的 `JSON` 将会缩进。

比如： `Accept: application/json; indent=4`。

``` python
{
    "unicode black star": "★",
    "value": 999
}
```

使用 `UNICODE_JSON` 和 `COMPACT_JSON` 设置键可以更改默认的 JSON 编码风格。

**.media_type**： `application/json`

**.format**： `'.json'`

**.charset**： `None`




### TemplateHTMLRenderer


使用 Django 的标准模板将数据呈现为 HTML。与其他渲染器不同，传递给 `Response` 的数据不需要序列化。另外，创建 `Response` 时可能需要包含 `template_name` 参数。

TemplateHTMLRenderer 将创建一个 `RequestContext`，使用 `response.data` 作为上下文字典，并确定用于呈现上下文的模板名称。

模板名称由（按优先顺序）确定：  
1. 传递给 response 的显式 `template_name` 参数。
2. 在此类上设置明确的 `.template_name` 属性。
3. 调用 `view.get_template_names()` 的返回结果。


使用 `TemplateHTMLRenderer` 的视图示例：

``` python
class UserDetail(generics.RetrieveAPIView):
    """
    A view that returns a templated HTML representation of a given user.
    """
    queryset = User.objects.all()
    renderer_classes = (TemplateHTMLRenderer,)

    def get(self, request, *args, **kwargs):
        self.object = self.get_object()
        return Response({'user': self.object}, template_name='user_detail.html')
```

你可以使用 `TemplateHTMLRenderer` 来使 REST framework 返回常规 HTML 页面，或者从单个端点（a single endpoint）返回 HTML 和 API 响应。

如果你正在构建使用 `TemplateHTMLRenderer` 以及其他渲染器类的网站，则应考虑将 `TemplateHTMLRenderer` 列为 `renderer_classes` 列表中的第一个类，以便即使对于发送格式错误的 `ACCEPT:` header 的浏览器，也会优先考虑它。


**.media_type**： `text/html`

**.format**： `'.html'`

**.charset**： `utf-8`


### StaticHTMLRenderer

一个简单的渲染器，它只是返回预渲染的 HTML。与其他渲染器不同，传递给响应对象的数据应该是表示要返回的内容的字符串。


使用 `StaticHTMLRenderer` 的视图示例：

``` python
@api_view(('GET',))
@renderer_classes((StaticHTMLRenderer,))
def simple_html_view(request):
    data = '<html><body><h1>Hello, world</h1></body></html>'
    return Response(data)
```

你可以使用 `StaticHTMLRenderer` 来使 REST framework 返回常规 HTML 页面，或者从单个端点（a single endpoint）返回 HTML 和 API 响应。


**.media_type**： `text/html`

**.format**： `'.html'`

**.charset**： `utf-8`



### BrowsableAPIRenderer

将数据呈现为可浏览的 HTML API：

![](https://user-gold-cdn.xitu.io/2018/3/6/161faa6cbcca4e26?w=791&h=696&f=png&s=39050)


该渲染器将确定哪个其他渲染器被赋予最高优先级，并使用该渲染器在 HTML 页面中显示 API 风格响应。


**.media_type**： `text/html`

**.format**： `'.api'`

**.charset**： `utf-8`

**.template**： `'rest_framework/api.html'`


**自定义 BrowsableAPIRenderer**

默认情况下，除 `BrowsableAPIRenderer` 之外，响应内容将使用最高优先级的渲染器渲染。如果你需要自定义此行为，例如，将 HTML 用作默认返回格式，但在可浏览的 API 中使用 JSON，则可以通过覆盖 `get_default_renderer()` 方法来实现。

例如：

``` python
class CustomBrowsableAPIRenderer(BrowsableAPIRenderer):
    def get_default_renderer(self, view):
        return JSONRenderer()
```


### AdminRenderer

将数据呈现为 HTML，以显示类似管理员的内容：

![](https://user-gold-cdn.xitu.io/2018/3/6/161faa6cbcd71d07?w=996&h=528&f=png&s=55904)


该渲染器适用于 CRUD 风格的 Web API，该 API 还应提供用于管理数据的用户友好界面。

请注意， `AdminRenderer`  对于嵌套或列出序列化输入的视图不起作用，因为 HTML 表单无法正确支持它们。

**注意**：当数据中存在正确配置的 `URL_FIELD_NAME` （默认为 `url`）属性时， `AdminRenderer` 仅能够包含指向详细页面的链接。对于 `HyperlinkedModelSerializer` ，情况就是这样，但对于 `ModelSerializer` 类或普通 `Serializer` 类，你需要确保明确包含该字段。例如，在这里我们使用模型 `get_absolute_url` 方法：

``` python
class AccountSerializer(serializers.ModelSerializer):
    url = serializers.CharField(source='get_absolute_url', read_only=True)

    class Meta:
        model = Account
```

**.media_type**： `text/html`

**.format**： `'.admin'`

**.charset**： `utf-8`

**.template**： `'rest_framework/admin.html'`



### HTMLFormRenderer

将序列化返回的数据呈现为 HTML 表单。该渲染器的输出不包含封闭的 `<form>` 标签，隐藏的 CSRF 输入或任何提交按钮。

这个渲染器不是直接使用，而是可以通过将一个序列化器实例传递给 `render_form` 模板标签来在模板中使用。

``` python
{% load rest_framework %}

<form action="/submit-report/" method="post">
    {% csrf_token %}
    {% render_form serializer %}
    <input type="submit" value="Save" />
</form>
```

**.media_type**： `text/html`

**.format**： `'.form'`

**.charset**： `utf-8`

**.template**： `'rest_framework/horizontal/form.html'`


### MultiPartRenderer

该渲染器用于呈现 HTML multipart form 数据。它不适合作为响应渲染器，而是用于创建测试请求，使用REST framework 的测试客户端和测试请求工厂。

**.media_type**： `multipart/form-data; boundary=BoUnDaRyStRiNg`

**.format**： `'.multipart'`

**.charset**： `utf-8`

## 自定义渲染器


要实现自定义渲染器，您应该继承 `BaseRenderer` ，设置 `.media_type` 和 `.format` 属性，并实现 `.render(self, data, media_type=None, renderer_context=None)` 方法。


该方法应返回一个字符串，它将用作 HTTP 响应的主体。

传递给 `.render()` 方法的参数是：


### `data`


请求数据，由 `Response()` 实例化时设置。

### `media_type=None`

可选的。如果提供，这是接受的媒体类型，由内容协商（content negotiation）阶段确定。

依赖于客户端的 `Accept:` header，它可以比渲染器的 `media_type` 属性更具体，并且可能包含媒体类型参数。比如 `"application/json; nested=true"` 。


### `renderer_context=None`


可选的。如果提供，它是视图提供的上下文信息字典。

默认情况下，这将包括以下键：`view` , `request` , `response` , `args` , `kwargs` 。


### 举个栗子
以下是一个示例纯文本渲染器，它将返回带有数据参数的响应作为响应的内容。

``` python
from django.utils.encoding import smart_unicode
from rest_framework import renderers


class PlainTextRenderer(renderers.BaseRenderer):
    media_type = 'text/plain'
    format = 'txt'

    def render(self, data, media_type=None, renderer_context=None):
        return data.encode(self.charset)
```


### 设置 charset 

默认情况下，渲染器类被假定为使用 UTF-8 编码。要使用不同的编码，请在渲染器上设置 `charset` 属性。

``` python
class PlainTextRenderer(renderers.BaseRenderer):
    media_type = 'text/plain'
    format = 'txt'
    charset = 'iso-8859-1'

    def render(self, data, media_type=None, renderer_context=None):
        return data.encode(self.charset)
```

请注意，如果渲染器类返回一个 unicode 字符串，则响应内容将被 `Response` 类强制为一个 bytestring，请在渲染器上设置 `charset` 属性用于确定编码。

如果渲染器返回代表原始二进制内容的字符串，则应将 `charset`  值设置为 `None`，这将确保响应的 `Content-Type` header 不会设置 `charset` 值。

在某些情况下，你可能还想将 `render_style` 属性设置为 `'binary'`。这样做也将确保可浏览的 API 不会尝试将二进制内容显示为字符串。

``` python
class JPEGRenderer(renderers.BaseRenderer):
    media_type = 'image/jpeg'
    format = 'jpg'
    charset = None
    render_style = 'binary'

    def render(self, data, media_type=None, renderer_context=None):
        return data
```

## 渲染器的高级用法

你可以使用 REST framework 的渲染器来做一些非常灵活的事情。例如...  
* 根据请求的 media type，提供来自同一端点的平面或嵌套（flat or nested）表示。
* 同时处理常规 HTML 网页和来自相同端点的基于 JSON 的 API 响应。
* 指定 API 客户端使用的多种 HTML 表示形式。
* 不用明确指定渲染器的 media type，例如使用 `media_type ='image/*'`，并使用 `Accept` header 改变响应的编码。


### media type 的变化

在某些情况下，可能希望视图根据接受的 media type 使用不同的序列化风格。如果需要这样做，可以访问 `request.accepted_renderer` 以确定将用于响应的协商（negotiate）渲染器。


例如：

``` python
@api_view(('GET',))
@renderer_classes((TemplateHTMLRenderer, JSONRenderer))
def list_users(request):
    """
    A view that can return JSON or HTML representations
    of the users in the system.
    """
    queryset = Users.objects.filter(active=True)

    if request.accepted_renderer.format == 'html':
        # TemplateHTMLRenderer takes a context dict,
        # and additionally requires a 'template_name'.
        # It does not require serialization.
        data = {'users': queryset}
        return Response(data, template_name='list_users.html')

    # JSONRenderer requires serialized data as normal.
    serializer = UserSerializer(instance=queryset)
    data = serializer.data
    return Response(data)
```

### 不明确的 media type

在某些情况下，可能需要渲染器来提供一系列 media type 。在这种情况下，可以通过使用 `media_type` 值（如 `image/*`或`*/*`）来指定它应该响应的 media type 。

如果没有明确指定渲染器的 media type ，则应确保在返回响应时使用 `content_type` 属性明确指定 media type 。比如：

``` python
return Response(data, content_type='image/png')
```

### 设计你的 media type


出于许多 Web API 的目的，具有超链接关系的简单 JSON 响应可能就足够了。如果你想完全融入 RESTful 设计和 HATEOAS，则需要更详细地考虑 media type 的设计和使用。


### HTML error 视图

通常情况下，无论处理常规响应还是引发异常的响应（例如 `Http404` 或 `PermissionDenied` 异常）或 `APIException` 的子类引起的响应，渲染器都会有相同的表现。

如果您使用的是 `TemplateHTMLRenderer` 或 `StaticHTMLRenderer`，并且引发异常，则行为会稍有不同，反映了 Django 对错误视图的默认处理。

由 HTML 渲染器引发和处理的异常将尝试使用以下方法之一（按优先顺序）进行渲染。  
* 加载并渲染模板 `{status_code}.html`。
* 加载并渲染模板 `api_exception.html`。
* 渲染 HTTP 状态码和文本，例如 "404 Not Found"。

模板将使用 `RequestContext` 进行渲染，其中包含 `status_code` 和 `details` 键。

> 注意：如果 `DEBUG = True`，则会显示 Django 的标准错误回溯页面，而不是显示 HTTP 状态码和文本。

