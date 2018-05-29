> [官方原文链接](http://www.django-rest-framework.org/api-guide/schemas/)  


# Schemas

API schema 是一个非常有用的工具，它允许一系列用例，包括生成参考文档，或者驱动可以与 API 交互的动态客户端库。

## 安装 Core API

你需要安装 `coreapi` 软件包才能为 REST framework 添加 schema 支持。

``` bash
pip install coreapi
```

## 内部 schema 表示

REST framework 使用 Core API 以便以独立于格式的表示对 schema 信息建模。这些信息可以被渲染成各种不同的 schema 格式，或者用于生成 API 文档。

在使用 Core API 时，schema 表示为 `Document`，它是有关 API 信息的顶级容器对象。可用的 API 交互使用 `Link` 对象表示。每个链接都包含一个 URL，HTTP 方法，并且可能包含一个 `Field` 实例列表，它描述了 API 端点可以接受的任何参数。`Link` 和 `Field` 实例还可能包含描述，允许将 API schema 渲染到用户文档中。

以下是包含单个搜索端点的 API 说明示例：

``` python
coreapi.Document(
    title='Flight Search API',
    url='https://api.example.org/',
    content={
        'search': coreapi.Link(
            url='/search/',
            action='get',
            fields=[
                coreapi.Field(
                    name='from',
                    required=True,
                    location='query',
                    description='City name or airport code.'
                ),
                coreapi.Field(
                    name='to',
                    required=True,
                    location='query',
                    description='City name or airport code.'
                ),
                coreapi.Field(
                    name='date',
                    required=True,
                    location='query',
                    description='Flight date in "YYYY-MM-DD" format.'
                )
            ],
            description='Return flight availability and prices.'
        )
    }
)
```

## Schema 输出格式

为了呈现在 HTTP 响应中，必须将内部表示渲染为响应中使用的实际字节。

Core JSON 被设计为与 Core API 一起使用的规范格式。REST framework 包含用于处理此媒体类型的渲染器类，该渲染器类可用作 `renderers.CoreJSONRenderer`。

### 备用 schema 格式

其他 schema 格式（如 Open API（“Swagger”），JSON HyperSchema 或 API Blueprint）也可以通过实现处理将 `Document` 实例转换为字符串表示形式的自定义渲染器类来支持。

如果有一个 Core API 编解码器包支持将编码格式化为你要使用的格式，则可以使用编解码器来实现渲染器类。

#### 举个栗子

例如，`openapi_codec` 包提供对 Open API（“Swagger”）格式的编码或解码支持：

``` python
from rest_framework import renderers
from openapi_codec import OpenAPICodec

class SwaggerRenderer(renderers.BaseRenderer):
    media_type = 'application/openapi+json'
    format = 'swagger'

    def render(self, data, media_type=None, renderer_context=None):
        codec = OpenAPICodec()
        return codec.dump(data)
```




## Schemas vs 超媒体

值得指出的是，Core API 也可以用来模拟超媒体响应，它为 API schema 提供了另一种交互风格。

通过 API schema，整个可用接口作为单个端点呈现在前端。然后，对各个 API 端点的响应通常会以普通数据的形式呈现，而不会在每个响应中包含任何进一步的交互。

使用超媒体，客户端会看到包含数据和可用交互的文档。每次交互都会生成一个新文档，详细说明当前状态和可用交互。


---

# 创建一个 schema

REST framework 包含用于自动生成 schema 的功能，或者允许你明确指定。

## 手动 Schema 规范

要手动指定 schema，请创建一个 Core API `Document`，与上例类似。

``` python
schema = coreapi.Document(
    title='Flight Search API',
    content={
        ...
    }
)
```


## 自动 Schema 生成

自动 schema 生成由 `SchemaGenerator` 类提供。

`SchemaGenerator` 处理路由 URL pattterns 列表并编译适当结构化的 Core API 文档。

基本用法只是为 schema 提供标题并调用 `get_schema()`：

``` python
generator = schemas.SchemaGenerator(title='Flight Search API')
schema = generator.get_schema()
```

## 按视图 schema 自定义

默认情况下，查看自省是由可通过 `APIView` 上的 `schema` 属性访问的 `AutoSchema` 实例执行的。这为视图，请求方法和路径提供了适当的 Core API `Link` 对象：

``` python
auto_schema = view.schema
coreapi_link = auto_schema.get_link(...)
```

（在编译模式时，`SchemaGenerator` 为每个视图，允许的方法和路径调用 `view.schema.get_link()`。）

---

**注意**: 对于基本的 `APIView` 子类，默认内省本质上仅限于 URL kwarg 路径参数。对于包含所有基于类的视图的 `GenericAPIView` 子类，`AutoSchema` 将尝试自省列化器，分页和过滤器字段，并提供更丰富的路径字段描述。（这里的关键钩子是相关的 `GenericAPIView` 属性和方法：`get_serializer`，`pagination_class`，`filter_backends` 等。）

---

要自定义 `Link` 生成，你可以：

1）使用 `manual_fields` kwarg 在你的视图上实例化 `AutoSchema`：

``` python
from rest_framework.views import APIView
from rest_framework.schemas import AutoSchema

class CustomView(APIView):
    ...
    schema = AutoSchema(
        manual_fields=[
            coreapi.Field("extra_field", ...),
        ]
    )
```

这允许扩展最常见的情况而不需要子类化。

2）提供具有更复杂定制的 `AutoSchema` 子类：

``` python
from rest_framework.views import APIView
from rest_framework.schemas import AutoSchema

class CustomSchema(AutoSchema):
    def get_link(...):
        # Implement custom introspection here (or in other sub-methods)

class CustomView(APIView):
    ...
    schema = CustomSchema()
```

这提供了对查看内省的完全控制。

3）在你的视图上实例化 `ManualSchema`，显式为视图提供 Core API `Fields`：

``` python
from rest_framework.views import APIView
from rest_framework.schemas import ManualSchema

class CustomView(APIView):
    ...
    schema = ManualSchema(fields=[
        coreapi.Field(
            "first_field",
            required=True,
            location="path",
            schema=coreschema.String()
        ),
        coreapi.Field(
            "second_field",
            required=True,
            location="path",
            schema=coreschema.String()
        ),
    ])
```

这允许手动指定某些视图的 schema，同时在别处保持自动生成。

通过将 `schema` 设置为 `None`，你可以禁用视图的 schema 生成：

``` python
    class CustomView(APIView):
        ...
        schema = None  # Will not appear in schema
```



# 添加 schema 视图

有几种不同的方式可以将 schema 视图添加到你的 API 中，具体取决于你需要的内容。

## get_schema_view 快捷方式

在你的项目中包含 schema 的最简单方法是使用 `get_schema_view()` 函数。

``` python
from rest_framework.schemas import get_schema_view

schema_view = get_schema_view(title="Server Monitoring API")

urlpatterns = [
    url('^$', schema_view),
    ...
]
```

添加视图后，你将能够通过 API 请求来检索自动生成的 schema 定义。

``` python
$ http http://127.0.0.1:8000/ Accept:application/coreapi+json
HTTP/1.0 200 OK
Allow: GET, HEAD, OPTIONS
Content-Type: application/vnd.coreapi+json

{
    "_meta": {
        "title": "Server Monitoring API"
    },
    "_type": "document",
    ...
}
```

`get_schema_view()` 的参数是：

#### `title`

可用于为 schema 定义提供描述性标题。

#### `url`

可用于为 schema  传递规范 URL。

``` python
schema_view = get_schema_view(
    title='Server Monitoring API',
    url='https://www.example.org/api/'
)
```

#### `urlconf`

表示要为其生成 API schema 的 URL conf 的导入路径的字符串。这默认为 Django 的 ROOT_URLCONF setting 的值。

``` python
schema_view = get_schema_view(
    title='Server Monitoring API',
    url='https://www.example.org/api/',
    urlconf='myproject.urls'
)
```

#### `renderer_classes`

可用于传递渲染 API 根端点的渲染器类列表。

``` python
from rest_framework.schemas import get_schema_view
from rest_framework.renderers import CoreJSONRenderer
from my_custom_package import APIBlueprintRenderer

schema_view = get_schema_view(
    title='Server Monitoring API',
    url='https://www.example.org/api/',
    renderer_classes=[CoreJSONRenderer, APIBlueprintRenderer]
)
```

#### `patterns`

将 schema 内省限定为 url patterns 列表。如果你只想将 `myproject.api` url 公开在 schema 中：

``` python
schema_url_patterns = [
    url(r'^api/', include('myproject.api.urls')),
]

schema_view = get_schema_view(
    title='Server Monitoring API',
    url='https://www.example.org/api/',
    patterns=schema_url_patterns,
)
```

#### `generator_class`

可用于指定要传递给 `SchemaView` 的 `SchemaGenerator` 子类。

#### `authentication_classes`

可用于指定将应用于 schema 端点的认证类列表。默认为 `settings.DEFAULT_AUTHENTICATION_CLASSES`

#### `permission_classes`

可用于指定将应用于 schema 端点的权限类列表。默认为 `settings.DEFAULT_PERMISSION_CLASSES`


## 使用显式 schema 视图

如果你需要比 `get_schema_view()` 快捷方式更多的控制权，那么你可以直接使用 `SchemaGenerator` 类来自动生成 `Document` 实例，并从视图中返回该实例。

此选项使你可以灵活地设置 schema 端点，并使用你想要的任何行为。例如，你可以将不同的权限，限流或身份验证策略应用于 schema 端点。

以下是使用 `SchemaGenerator` 和视图一起返回 schema 的示例。

**views.py:**

``` python
from rest_framework.decorators import api_view, renderer_classes
from rest_framework import renderers, response, schemas

generator = schemas.SchemaGenerator(title='Bookings API')

@api_view()
@renderer_classes([renderers.CoreJSONRenderer])
def schema_view(request):
    schema = generator.get_schema(request)
    return response.Response(schema)
```

**urls.py:**

``` python
urlpatterns = [
    url('/', schema_view),
    ...
]
```

你也可以为不同的用户提供不同的 schema，具体取决于他们拥有的权限。这种方法可以用来确保未经身份验证的请求以不同的模式呈现给已验证的请求，或者确保 API 的不同部分根据角色对不同用户可见。

为了呈现一个 schema，其中包含由用户权限过滤的端点，你需要将 `request` 参数传递给 `get_schema()` 方法，如下所示：

``` python
@api_view()
@renderer_classes([renderers.CoreJSONRenderer])
def schema_view(request):
    generator = schemas.SchemaGenerator(title='Bookings API')
    return response.Response(generator.get_schema(request=request))
```

## 显式 schema 定义

自动生成方法的替代方法是通过在代码库中声明 `Document` 对象来明确指定 API schema 。这样做会多一点工作，但确保你完全控制 schema 表示。

``` python
import coreapi
from rest_framework.decorators import api_view, renderer_classes
from rest_framework import renderers, response

schema = coreapi.Document(
    title='Bookings API',
    content={
        ...
    }
)

@api_view()
@renderer_classes([renderers.CoreJSONRenderer])
def schema_view(request):
    return response.Response(schema)
```

## 静态 schema 文件

最后的选择是使用 Core JSON 或 Open API 等可用格式之一将你的 API schema 编写为静态文件。

然后你可以：

* 将模式定义写为静态文件，并直接提供静态文件。
* 编写一个使用 `Core API` 加载的 schema 定义，然后根据客户端请求将其渲染为多种可用格式之一。

---

# 作为文档的 Schemas

API schemas 的一个常见用法是使用它们来构建文档页面。

REST framework 中的 schema 生成使用文档字符串来自动填充 schema 文档中的描述。

这些描述将基于：

* 对应方法的文档字符串（如果存在）。
* 类文档字符串中的命名部分，可以是单行或多行。
* 类文档字符串。

## 举个栗子

一个 `APIView`，带有明确的方法文档字符串。

``` python
class ListUsernames(APIView):
    def get(self, request):
        """
        Return a list of all user names in the system.
        """
        usernames = [user.username for user in User.objects.all()]
        return Response(usernames)
```

一个 `ViewSet`，带有一个明确的 action 文档字符串。

``` python
class ListUsernames(ViewSet):
    def list(self, request):
        """
        Return a list of all user names in the system.
        """
        usernames = [user.username for user in User.objects.all()]
        return Response(usernames)
```

类文档字符串中带有 action 的通用视图，使用单行样式。

``` python
class UserList(generics.ListCreateAPIView):
    """
    get: List all the users.
    post: Create a new user.
    """
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = (IsAdminUser,)
```

使用多行样式的类文档字符串中带有 action 的通用视图集。

``` python
class UserViewSet(viewsets.ModelViewSet):
    """
    API endpoint that allows users to be viewed or edited.

    retrieve:
    Return a user instance.

    list:
    Return all users, ordered by most recently joined.
    """
    queryset = User.objects.all().order_by('-date_joined')
    serializer_class = UserSerializer
```

---

# API 参考

## SchemaGenerator

一个遍历路由 URL patterns 列表的类，为每个视图请求 schema 并整理生成的 CoreAPI 文档。

通常你会用一个参数实例化 `SchemaGenerator`，如下所示：

``` python
generator = SchemaGenerator(title='Stock Prices API')
```

参数：

* `title` **必需** - API 的名称。
* `url` - API schema 的 root URL。除非 schema 包含在路径前缀下，否则此选项不是必需的。
* `patterns` - 生成 schema 时要检查的 URL 列表。默认为项目的 URL conf。
* `urlconf` - 生成 schema 时使用的 URL conf 模块名称。 默认为 `settings.ROOT_URLCONF`.

### get_schema(self, request)

返回表示 API schema 的 `coreapi.Document` 实例。

``` python
@api_view
@renderer_classes([renderers.CoreJSONRenderer])
def schema_view(request):
    generator = schemas.SchemaGenerator(title='Bookings API')
    return Response(generator.get_schema())
```

`request` 参数是可选的，如果你希望将每个用户的权限应用于生成的 schema ，则可以使用该参数。

### get_links(self, request)

返回一个嵌套的字典，其中包含在 API schema 中的所有链接。

如果要修改生成的 schema 的结构，重写该方法很合适，因为你可以使用不同的布局构建新的字典。


## AutoSchema

一个处理 schema 生成的个别视图内省的类。

`AutoSchema` 通过 `schema` 属性附加到 `APIView`。

`AutoSchema` 构造函数接受一个关键字参数  `manual_fields`。

**`manual_fields`**: 将添加到生成的字段的 `coreapi.Field` 实例 `list`。具有匹配名称的生成字段将被覆盖。

``` python
class CustomView(APIView):
    schema = AutoSchema(manual_fields=[
        coreapi.Field(
            "my_extra_field",
            required=True,
            location="path",
            schema=coreschema.String()
        ),
    ])
```

对于通过继承 `AutoSchema` 来自定义 schema 生成。

``` python
class CustomViewSchema(AutoSchema):
    """
    Overrides `get_link()` to provide Custom Behavior X
    """

    def get_link(self, path, method, base_url):
        link = super().get_link(path, method, base_url)
        # Do something to customize link here...
        return link

class MyView(APIView):
  schema = CustomViewSchema()
```

以下方法可覆盖。

### get_link(self, path, method, base_url)

返回与给定视图相对应的 `coreapi.Link` 实例。

这是主要的入口点。如果你需要为特定视图提供自定义行为，则可以覆盖此内容。

### get_description(self, path, method)

返回用作链接描述的字符串。默认情况下，这基于上面的 “作为文档的 Schemas” action 中描述的视图文档字符串。

### get_encoding(self, path, method)

与给定视图交互时返回一个字符串，以指定任何请求主体的编码。 例如 `'application/json'`。可能会返回一个空白字符串，以便查看不需要请求主体的视图。

### get_path_fields(self, path, method):

返回 `coreapi.Link()` 实例列表。用于 URL 中的每个路径参数。

### get_serializer_fields(self, path, method)

返回 `coreapi.Link()` 实例列表。用于视图使用的序列化类中的每个字段。

### get_pagination_fields(self, path, method)

返回 `coreapi.Link()` 实例列表，该列表由 `get_schema_fields()` 方法返回给视图使用的分页类。

### get_filter_fields(self, path, method)

返回 `coreapi.Link()` 实例列表，该列表是由视图所使用的过滤器类的 `get_schema_fields()` 方法返回的。

### get_manual_fields(self, path, method)

返回 `coreapi.Field()` 实例列表以添加或替换生成的字段。默认为（可选）传递给 `AutoSchema` 构造函数的 `manual_fields`。

可以通过 `path` 或 `method` 覆盖自定义 manual field。例如，每个方法的调整可能如下所示：

``` python
def get_manual_fields(self, path, method):
    """Example adding per-method fields."""

    extra_fields = []
    if method=='GET':
        extra_fields = # ... list of extra fields for GET ...
    if method=='POST':
        extra_fields = # ... list of extra fields for POST ...

    manual_fields = super().get_manual_fields(path, method)
    return manual_fields + extra_fields
```

### update_fields(fields, update_with)

实用的 `staticmethod`。封装逻辑以通过 `Field.name` 添加或替换列表中的字段。可能会被覆盖以调整替换标准。


## ManualSchema

允许手动为 schema 提供 `coreapi.Field` 实例的列表，以及一个可选的描述。

``` python
class MyView(APIView):
  schema = ManualSchema(fields=[
        coreapi.Field(
            "first_field",
            required=True,
            location="path",
            schema=coreschema.String()
        ),
        coreapi.Field(
            "second_field",
            required=True,
            location="path",
            schema=coreschema.String()
        ),
    ]
  )
```

`ManualSchema` 构造函数有两个参数：

**`fields`**: `coreapi.Field` 实例列表。必需。

**`description`**: 字符串描述。可选的。


---

## Core API

本文档简要介绍了用于表示 API schema 的 `coreapi` 包内的组件。

请注意，这些类是从 `coreapi` 包导入的，而不是从 `rest_framework` 包导入的。

### Document

表示 API schema 的容器。

#### `title`

API 的名称。

#### `url`

API 的规范 URL。

#### `content`

一个字典，包含 schema 的 `Link` 对象。

为了向 schema  提供更多结构，`content` 字典可以嵌套，通常是二层。例如：

``` python
content={
    "bookings": {
        "list": Link(...),
        "create": Link(...),
        ...
    },
    "venues": {
        "list": Link(...),
        ...
    },
    ...
}
```

### Link

代表一个单独的 API 端点。

#### `url`

端点的 URL。可能是一个 URI 模板，例如 `/users/{username}/`。

#### `action`

与端点关联的 HTTP 方法。请注意，支持多个 HTTP 方法的 url 应该对应于每个 HTTP 方法的单个链接。

#### `fields`

`Field` 实例列表，描述输入上的可用参数。

#### `description`

对端点的含义和用途的简短描述。

### Field

表示给定 API 端点上的单个输入参数。

#### `name`

输入的描述性名称。

#### `required`

boolean 值，表示客户端是否需要包含值，或者参数是否可以省略。

#### `location`

确定如何将信息编码到请求中。应该是以下字符串之一：

**"path"**

包含在模板化的 URI 中。例如，`/products/{product_code}/` 的 `url` 值可以与 `"path"` 字段一起使用，以处理 URL 路径中的 API 输入，例如 `/products/slim-fit-jeans/`。

这些字段通常与项目 URL conf 中的命名参数对应。

**"query"**

包含为 URL 查询参数。例如 `?search=sale`。通常用于 `GET` 请求。

这些字段通常与视图上的分页和过滤控件相对应。

**"form"**

包含在请求正文中，作为 JSON 对象或 HTML 表单的单个 item。例如 `{"colour": "blue", ...}`。通常用于`POST`，`PUT` 和 `PATCH` 请求。多个 `"form"` 字段可能包含在单个链接上。

这些字段通常与视图上的序列化类字段相对应。

**"body"**

包含完整的请求主体。通常用于 `POST`, `PUT` 和 `PATCH` 请求。链接上不得存在超过一个 `"body"` 字段。不能与 `"form"` 字段一起使用。

这些字段通常对应于使用 `ListSerializer`来验证请求输入或使用文件上载的视图。

#### `encoding`

**"application/json"**

JSON编码的请求内容。对应于使用 `JSONParser` 的视图。仅在 `Link` 中包含一个或多个 `location="form"` 字段或单个 `location="body"` 字段时有效。

**"multipart/form-data"**

Multipart 编码的请求内容。对应于使用 `MultiPartParser` 的视图。仅在 `Link` 中包含一个或多个 `location="form"` 字段时有效。

**"application/x-www-form-urlencoded"**

URL encode 的请求内容。对应于使用 `FormParser` 的视图。仅在 `Link` 中包含一个或多个 `location="form"` 字段时有效。

**"application/octet-stream"**

二进制上传请求内容。对应于使用 `FileUploadParser` 的视图。仅在 `Link` 中包含 `location="body"` 字段时有效。

#### `description`

对输入字段的含义和用途的简短描述。
