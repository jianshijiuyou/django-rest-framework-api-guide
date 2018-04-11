> [官方原文链接](http://www.django-rest-framework.org/api-guide/metadata/)  


# 元数据


REST framework 包含一个可配置的机制，用于确定 API 如何响应 `OPTIONS` 请求。这使你可以返回 API schema 或其他资源信息。

对于 HTTP `OPTIONS` 请求应该返回哪种风格的响应，目前还没有任何被广泛采用的约定，所以我们提供了一种专门的风格来返回一些有用的信息。

下面是一个示例响应，演示默认返回的信息。

``` python
HTTP 200 OK
Allow: GET, POST, HEAD, OPTIONS
Content-Type: application/json

{
    "name": "To Do List",
    "description": "List existing 'To Do' items, or create a new item.",
    "renders": [
        "application/json",
        "text/html"
    ],
    "parses": [
        "application/json",
        "application/x-www-form-urlencoded",
        "multipart/form-data"
    ],
    "actions": {
        "POST": {
            "note": {
                "type": "string",
                "required": false,
                "read_only": false,
                "label": "title",
                "max_length": 100
            }
        }
    }
}
```

## 设置元数据 scheme

你可以使用 `'DEFAULT_METADATA_CLASS'` settings key 全局设置元数据类：

``` python
REST_FRAMEWORK = {
    'DEFAULT_METADATA_CLASS': 'rest_framework.metadata.SimpleMetadata'
}
```

或者你可以单独设置一个视图的元数据类：

``` python
class APIRoot(APIView):
    metadata_class = APIRootMetadata

    def get(self, request, format=None):
        return Response({
            ...
        })
```

REST framework 包只包含一个名为 `SimpleMetadata` 的元数据类实现。如果你想使用另一种风格，你需要实现一个自定义的元数据类。

## 创建 schema 端点

如果你对创建通过常规 GET 请求访问的 schema 端点有特定要求，则可以考虑重新使用元数据 API 来实现此目的。

例如，可以在视图集上使用以下附加路由来提供可链接的 schema 端点。

``` python
@list_route(methods=['GET'])
def schema(self, request):
    meta = self.metadata_class()
    data = meta.determine_metadata(request, self)
    return Response(data)
```

有几个原因可以选择采用这种方法，包括 `OPTIONS` 响应不能缓存。

---

# 自定义元数据类

如果你想提供一个自定义的元数据类，你应该继承 `BaseMetadata` 并且实现 `determine_metadata(self, request, view)` 方法。

你可能想要做的事情包括返回 schema 信息，使用 JSON schema 等格式，或将调试信息返回给管理员用户。

## 举个栗子

以下类可用于限定返回到 `OPTIONS` 请求的信息。

``` python
class MinimalMetadata(BaseMetadata):
    """
    Don't include field and other information for `OPTIONS` requests.
    Just return the name and description.
    """
    def determine_metadata(self, request, view):
        return {
            'name': view.get_view_name(),
            'description': view.get_view_description()
        }
```

然后配置你的设置以使用此自定义类：

``` python
REST_FRAMEWORK = {
    'DEFAULT_METADATA_CLASS': 'myproject.apps.core.MinimalMetadata'
}
```

