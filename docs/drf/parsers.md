> [官方原文链接](http://www.django-rest-framework.org/api-guide/parsers/)


## 解析器

REST framework 包含许多内置的解析器类，允许接受各种媒体类型（media types）的请求。还支持自定义解析器，这使你可以灵活地设计 API 接受的媒体类型。

### 如何确定使用哪个解析器

视图的有效解析器集始终定义为类列表。当访问 `request.data` 时，REST framework 将检查传入请求的 `Content-Type` ，并确定使用哪个解析器来解析请求内容。

> **注意**：在开发客户端应用程序时，请务必确保在 HTTP 请求中发送数据时设置了 `Content-Type` 。  
> 如果你不设置 content type，大多数客户端将默认使用 `'application / x-www-form-urlencoded'` ，这可能不是你想要的。
> 例如，如果你使用 jQuery 和 `.ajax()` 方法发送 `json` 数据，则应确保包含 `contentType:'application/json'` 设置。

### 设置解析器

可以使用 `DEFAULT_PARSER_CLASSES` 设置默认的全局解析器。例如，以下设置将只允许带有 `JSON` 内容的请求，而不是默认的 JSON 或表单数据。

``` python
REST_FRAMEWORK = {
    'DEFAULT_PARSER_CLASSES': (
        'rest_framework.parsers.JSONParser',
    )
}
```

还可以在基于类（`API​​View` ）的视图上设置单个视图或视图集的解析器。

``` python
from rest_framework.parsers import JSONParser
from rest_framework.response import Response
from rest_framework.views import APIView

class ExampleView(APIView):
    """
    A view that can accept POST requests with JSON content.
    """
    parser_classes = (JSONParser,)

    def post(self, request, format=None):
        return Response({'received data': request.data})
```

或者和 `@api_view` 装饰器一起使用。

```python
from rest_framework.decorators import api_view
from rest_framework.decorators import parser_classes
from rest_framework.parsers import JSONParser

@api_view(['POST'])
@parser_classes((JSONParser,))
def example_view(request, format=None):
    """
    A view that can accept POST requests with JSON content.
    """
    return Response({'received data': request.data})
```

## API 参考


### JSONParser

解析 JSON 请求内容。

**.media_type**： `application/json`



### FormParser

解析 HTML 表单内容。`request.data` 是一个 `QueryDict` 字典，包含所有表单参数。  

通常需要同时使用 `FormParser` 和 `MultiPartParser`，以完全支持 HTML 表单数据。

**.media_type**： `application/x-www-form-urlencoded`


### MultiPartParser

解析文件上传的 multipart  HTML 表单内容。 `request.data` 是一个 `QueryDict`（其中包含表单参数和文件）。



通常需要同时使用 `FormParser` 和 `MultiPartParser`，以完全支持 HTML 表单数据。

**.media_type**： `application/form-data`


### FileUploadParser

解析文件上传内容。  `request.data` 是一个 `QueryDict` （只包含一个存有文件的 `'file'` key）。


如果与 `FileUploadParser` 一起使用的视图是用 `filename` URL 关键字参数调用的，那么该参数将用作文件名。

如果在没有 `filename` URL 关键字参数的情况下调用，则客户端必须在 `Content-Disposition` HTTP header 中设置文件名。例如 `Content-Disposition: attachment; filename=upload.jpg`。

**.media_type**： `*/*`

请注意：
* `FileUploadParser` 用于本地客户端，可以将文件作为原始数据请求上传。对于基于 Web 的上传，或者对于具有分段上传支持的本地客户端，您应该使用 `MultiPartParser` 解析器。
* 由于此解析器的 `media_type` 与任何 content type 都匹配，因此 `FileUploadParser` 通常应该是在 API 视图上设置的唯一解析器。
* `FileUploadParser` 遵循 Django 的标准 `FILE_UPLOAD_HANDLERS` 设置和 `request.upload_handlers` 属性。有关更多详细信息，请参阅 Django 文档。

基本用法示例：

``` python
# views.py
class FileUploadView(views.APIView):
    parser_classes = (FileUploadParser,)

    def put(self, request, filename, format=None):
        file_obj = request.data['file']
        # ...
        # do some stuff with uploaded file
        # ...
        return Response(status=204)

# urls.py
urlpatterns = [
    # ...
    url(r'^upload/(?P<filename>[^/]+)$', FileUploadView.as_view())
]
```



## 自定义解析

要实现自定义解析器，应该继承 `BaseParser`，设置 `.media_type` 属性并实现 `.parse(self,stream,media_type,parser_context)` 方法。


该方法应该返回将用于填充 `request.data` 属性的数据。


传递给 `.parse()` 的参数是：

### stream

表示请求正文的流式对象。

### media_type

可选。如果提供，则这是传入请求内容的 media type。

根据请求的 `Content-Type:` header，可以比渲染器的 `media_type` 属性更具体，并且可能包含 media type 参数。比如 `"text/plain; charset=utf-8"` 。

### parser_context

可选。如果提供，则该参数将是一个包含解析请求内容可能需要的任何其他上下文的字典。

默认情况下，这将包括以下 key：`view`，`request`，`args`，`kwargs`。



### 举个栗子

以下是一个示例纯文本解析器，它将使用表示请求正文的字符串填充 `request.data` 属性。

``` python
class PlainTextParser(BaseParser):
    """
    Plain text parser.
    """
    media_type = 'text/plain'

    def parse(self, stream, media_type=None, parser_context=None):
        """
        Simply return a string representing the body of the request.
        """
        return stream.read()
```


## 第三方包

以下是可用的第三方包。


### YAML

[REST framework YAML](https://jpadilla.github.io/django-rest-framework-yaml/) 提供 YAML 解析和渲染支持。它以前直接包含在 REST framework 包中，现在作为第三方包。

#### 安装和配置

使用pip安装。

``` bash
$ pip install djangorestframework-yaml
```

修改 REST framework settings。

``` python
REST_FRAMEWORK = {
    'DEFAULT_PARSER_CLASSES': (
        'rest_framework_yaml.parsers.YAMLParser',
    ),
    'DEFAULT_RENDERER_CLASSES': (
        'rest_framework_yaml.renderers.YAMLRenderer',
    ),
}
```

### XML

[REST Framework XML](https://jpadilla.github.io/django-rest-framework-xml/) 提供了一种简单的非正式 XML 格式。它以前直接包含在 REST framework 包中，现在作为第三方包。



#### 安装和配置


使用pip安装。

``` bash
$ pip install djangorestframework-xml
```

修改 REST framework settings。


``` python
REST_FRAMEWORK = {
    'DEFAULT_PARSER_CLASSES': (
        'rest_framework_xml.parsers.XMLParser',
    ),
    'DEFAULT_RENDERER_CLASSES': (
        'rest_framework_xml.renderers.XMLRenderer',
    ),
}
```


### [MessagePack](https://github.com/juanriaza/django-rest-framework-msgpack)


### [CamelCase JSON](https://github.com/vbabiy/djangorestframework-camel-case)

## 友情提示

配合源码阅读效果更佳哦