> [官方原文链接](http://www.django-rest-framework.org/api-guide/routers/)

## 路由


一些 Web 框架（如 Rails）提供了一种能够自动确定应用程序的 URL 如何映射到处理请求的功能。

REST framework 增加了对 Django 自动 URL 路由的支持，并提供了一种将视图逻辑连接到一组 URL 的简单，高效和一致的方式。

### 用法

下面是一个使用 `SimpleRouter` 的简单 URL 配置示例。

``` python
from rest_framework import routers

router = routers.SimpleRouter()
router.register(r'users', UserViewSet)
router.register(r'accounts', AccountViewSet)
urlpatterns = router.urls
```

`register()` 方法有两个必须参数：  
 * `prefix` - 设置这组路由的前缀。
 * `viewset` - 设置对应的视图集类。

或者，您也可以指定一个附加参数：  

 * `base_name` - 用于创建的 URL 名称的基础。如果未设置，将根据视图集的 `queryset` 属性自动生成。请注意，如果视图集不包含 `queryset` 属性，则在注册视图集时必须设置 `base_name`。


上面的例子会生成以下 URL 模式：  

* URL pattern: `^users/$` Name:  `'user-list'`  
* URL pattern: `^users/{pk}/$` Name:  `'user-detail'`  
* URL pattern: `^accounts/$` Name:  `'account-list'`  
* URL pattern: `^accounts/{pk}/$` Name:  `'account-detail'`  

> 注意：`base_name` 参数用于指定视图名称模式的初始部分。在上面的例子中，是 `user` 或 `account` 部分。

通常，您不需要指定 `base_name` 参数，但是如果您有一个视图集定义了自定义 `get_queryset` 方法，那么该视图集可能没有设置 `.queryset` 属性。如果此时尝试注册该视图，则会看到如下所示的错误：

``` text
'base_name' argument not specified, and could not automatically determine the name from the viewset, as it does not have a '.queryset' attribute.
```
> `'base_name'` 参数未指定，并且无法自动确定视图中的名称，因为它没有' `.queryset'` 属性。

这时候就需要在注册视图集时显式设置 `base_name` 参数，因为它无法从模型名称中自动确定。


#### 使用 `include` 与路由

路由实例上的 `.urls` 属性是一个标准的 URL patterns。关于如何包含这些 URL，有许多不同的样式。

例如，可以将 `router.urls` 附加到现有视图的列表中...


``` python
router = routers.SimpleRouter()
router.register(r'users', UserViewSet)
router.register(r'accounts', AccountViewSet)

urlpatterns = [
    url(r'^forgot-password/$', ForgotPasswordFormView.as_view()),
]

urlpatterns += router.urls
```

另外，你也可以使用 Django 的 `include` 函数，比如...

``` python
urlpatterns = [
    url(r'^forgot-password/$', ForgotPasswordFormView.as_view()),
    url(r'^', include(router.urls)),
]
```
还可以设置 namespace。

``` python
urlpatterns = [
    url(r'^forgot-password/$', ForgotPasswordFormView.as_view()),
    url(r'^api/', include(router.urls, namespace='api')),
]
```

如果对超链接序列化器使用命名空间，则还需要确保序列化器上的任何 `view_name` 参数都能正确反映命名空间。在上面的示例中，您需要为超链接到用户详细信息视图的序列化程序字段包含诸如 `view_name='api:user-detail'` 之类的参数。

#### 额外的链接和操作

用 `@detail_route` 或 `@list_route` 装饰的 视图上的任何方法 也将被路由。例如，在 `UserViewSet` 类中给出这样的方法：

``` python
from myapp.permissions import IsAdminOrIsSelf
from rest_framework.decorators import detail_route

class UserViewSet(ModelViewSet):
    ...

    @detail_route(methods=['post'], permission_classes=[IsAdminOrIsSelf])
    def set_password(self, request, pk=None):
        ...
```

会生成以下URL模式：
* URL pattern:  `^users/{pk}/set_password/$` Name:  `'user-set-password'`


如果您不想使用默认生成的 URL 模式，则可以使用 url_path 参数对其进行自定义。

例如，如果您想将我们的自定义操作的URL更改为 `^users/{pk}/change-password/$`，则可以编写：

``` python
from myapp.permissions import IsAdminOrIsSelf
from rest_framework.decorators import detail_route

class UserViewSet(ModelViewSet):
    ...

    @detail_route(methods=['post'], permission_classes=[IsAdminOrIsSelf], url_path='change-password')
    def set_password(self, request, pk=None):
        ...
```

上面的例子现在将生成以下URL模式：

* URL pattern:  `^users/{pk}/change-password/$` Name:  `'user-change-password'`

如果您不想使用生成的默认名称，则可以使用 url_name 参数对其进行自定义。

例如，如果您想将自定义操作的名称更改为 `'user-change-password'`，则可以编写：

``` python
from myapp.permissions import IsAdminOrIsSelf
from rest_framework.decorators import detail_route

class UserViewSet(ModelViewSet):
    ...

    @detail_route(methods=['post'], permission_classes=[IsAdminOrIsSelf], url_name='change-password')
    def set_password(self, request, pk=None):
        ...
```

上面的例子现在将生成以下URL模式：

* URL pattern:  `^users/{pk}/set_password/$`  Name: `'user-change-password'`


可以同时使用 `url_path` 和 `url_name` 参数。

更多相关信息请看 [视图集：标记额外的路由行为](https://juejin.im/post/5a991807518825558a060a77#heading-3)。


## API 参考

### SimpleRouter

`SimpleRouter` 包含标准的 `list`，`create`，`retrieve`，`update`，`partial_update` 和 `destroy` action。`SimpleRouter` 还支持视图集使用 `@detail_route` 或 `@list_route` 装饰器标记其他要路由的方法。

<table border="0.8">
    <tr><th>URL Style</th><th>HTTP Method</th><th>Action</th><th>URL Name</th></tr>
    <tr><td rowspan=2>{prefix}/</td><td>GET</td><td>list</td><td rowspan=2>{basename}-list</td></tr></tr>
    <tr><td>POST</td><td>create</td></tr>
    <tr><td>{prefix}/{methodname}/</td><td>GET, 或者由 `methods` 参数指定</td><td>`@list_route` 装饰的方法</td><td>{basename}-{methodname}</td></tr>
    <tr><td rowspan=4>{prefix}/{lookup}/</td><td>GET</td><td>retrieve</td><td rowspan=4>{basename}-detail</td></tr></tr>
    <tr><td>PUT</td><td>update</td></tr>
    <tr><td>PATCH</td><td>partial_update</td></tr>
    <tr><td>DELETE</td><td>destroy</td></tr>
    <tr><td>{prefix}/{lookup}/{methodname}/</td><td>GET, 或者由 `methods` 参数指定</td><td>`@detail_route` 装饰的方法</td><td>{basename}-{methodname}</td></tr>
</table>


默认情况下，由 `SimpleRouter` 创建的 URL 附加了尾部斜杠。在实例化路由器时，可以通过将 `trailing_slash` 参数设置为 `False` 来修改此行为。例如：

``` python
router = SimpleRouter(trailing_slash=False)
```

尾部斜杠在 Django 中是常规的，但在其他一些框架（如 Rails）中默认不使用。选择使用哪种风格在很大程度上是一个偏好问题，尽管一些 JavaScript 框架可能会期望特定的路由风格。

`SimpleRouter` 将匹配包含除斜杠和句点字符以外的任何字符的 lookup 值。对于更严格（或宽松）的 lookup pattern，请在视图集上设置 `lookup_value_regex` 属性。例如，您可以将 lookup  限制为有效的 UUID：

``` python
class MyModelViewSet(mixins.RetrieveModelMixin, viewsets.GenericViewSet):
    lookup_field = 'my_model_id'
    lookup_value_regex = '[0-9a-f]{32}'
```

### DefaultRouter

`DefaultRouter` 与上面的 `SimpleRouter` 相似，但还包含一个默认的 API 根视图，该视图返回一个包含指向所有列表视图的超链接的响应。它还为可选的 `.json` 风格格式后缀生成路由。


<table border="0.8">
    <tr><th>URL Style</th><th>HTTP Method</th><th>Action</th><th>URL Name</th></tr>
    <tr><td>[.format]</td><td>GET</td><td>自动生成的根视图</td><td>api-root</td></tr></tr>
    <tr><td rowspan=2>{prefix}/[.format]</td><td>GET</td><td>list</td><td rowspan=2>{basename}-list</td></tr></tr>
    <tr><td>POST</td><td>create</td></tr>
    <tr><td>{prefix}/{methodname}/[.format]</td><td>GET, 或者由 `methods` 参数指定</td><td>`@list_route` 装饰的方法</td><td>{basename}-{methodname}</td></tr>
    <tr><td rowspan=4>{prefix}/{lookup}/[.format]</td><td>GET</td><td>retrieve</td><td rowspan=4>{basename}-detail</td></tr></tr>
    <tr><td>PUT</td><td>update</td></tr>
    <tr><td>PATCH</td><td>partial_update</td></tr>
    <tr><td>DELETE</td><td>destroy</td></tr>
    <tr><td>{prefix}/{lookup}/{methodname}/[.format]</td><td>GET, 或者由 `methods` 参数指定</td><td>`@detail_route` 装饰的方法</td><td>{basename}-{methodname}</td></tr>
</table>

> 注意：我在使用 3.7.7 版本时，发现要写成 `{prefix}[.format]/` 风格才能访问，`{prefix}/[.format]` 风格会报 404，不知道是我设置问题还是官方更新了。

与 `SimpleRouter` 一样，通过在实例化 `DefaultRouter` 时将 `trailing_slash` 参数设置为 `False`，可以删除 URL 路径上的尾部斜杠。

``` python
router = DefaultRouter(trailing_slash=False)
```


## 自定义路由

自定义路由并不是你经常需要做的事情，但是如果你对 API 的 URL 是如何构建的有特定的要求的话，它会很有用。这样做可以让你以可重用的方式封装 URL 结构，确保你不必为每个新视图明确编写 URL 模式。

实现自定义路由的最简单方法是对现有路由类之一进行子类化。`.routes` 属性用于对将映射到每个视图集的 URL 模式进行模板化。`.Routes` 属性是一个 `Route` 列表（`Route` 的是一个 namedtuple）。

`Route` 命名元组的参数：

**url** ： 表示要路由的 URL 的字符串。可以包含以下格式字符串：  
 * `{prefix}` - 路由的 URL 前缀。
 * `{lookup}` - 匹配单个实例的 lookup field。
 *  `{trailing_slash}` - 可以是 '/' 或空字符串，具体取决于 `trailing_slash` 参数。

**mapping** ： HTTP 方法名称与视图方法的映射


**name**： 用于调用 `reverse` 时的 URL 的名称。可能包含以下格式字符串：  
* `{basename}` - 创建的 URL 名称的基础。


**initkwargs**： 实例化视图时应传递的任何其他参数的字典。注意，`suffix` 参数被保留用于标识视图集类型，在生成视图名称和 breadcrumb  链接时使用。


### 自定义动态路由

你还可以自定义 `@list_route` 和 `@detail_route` 装饰器的路由方式。要路由这两个装饰器中的一个或两个，请在 `.routes` 列表中包含一个 `DynamicListRoute` 和/或 `DynamicDetailRoute`（别忘了类型是 namedtuple）。


`DynamicListRoute` 和 `DynamicDetailRoute` 的参数是：  

**url**： 表示要路由的 URL 的字符串。可以包含与 `Route` 相同的格式字符串，并且还接受 `{methodname}` 和 `{methodnamehyphen}` 格式的字符串。

**name**： 用于调用 `reverse` 时的名称。可以包含以下格式字符串：`{basename}` , `{methodname} ` 和 ` {methodnamehyphen}` 。

**initkwargs**： 实例化视图时应传递的任何其他参数的字典。



### 举个栗子


以下示例只会路由 `list` 和 `retrieve` action，并且不使用尾部斜杠约定。

``` python
from rest_framework.routers import Route, DynamicDetailRoute, SimpleRouter

class CustomReadOnlyRouter(SimpleRouter):
    """
    A router for read-only APIs, which doesn't use trailing slashes.
    """
    routes = [
        Route(
            url=r'^{prefix}$',
            mapping={'get': 'list'},
            name='{basename}-list',
            initkwargs={'suffix': 'List'}
        ),
        Route(
            url=r'^{prefix}/{lookup}$',
            mapping={'get': 'retrieve'},
            name='{basename}-detail',
            initkwargs={'suffix': 'Detail'}
        ),
        DynamicDetailRoute(
            url=r'^{prefix}/{lookup}/{methodnamehyphen}$',
            name='{basename}-{methodnamehyphen}',
            initkwargs={}
        )
    ]
```
让我们来看看 `CustomReadOnlyRouter` 为一个简单的视图集生成的路由。

`views.py`：

``` python
class UserViewSet(viewsets.ReadOnlyModelViewSet):
    """
    A viewset that provides the standard actions
    """
    queryset = User.objects.all()
    serializer_class = UserSerializer
    lookup_field = 'username'

    @detail_route()
    def group_names(self, request, pk=None):
        """
        Returns a list of all the group names that the given
        user belongs to.
        """
        user = self.get_object()
        groups = user.groups.all()
        return Response([group.name for group in groups])
```

`urls.py`：

```python
router = CustomReadOnlyRouter()
router.register('users', UserViewSet)
urlpatterns = router.urls
```

将生成以下映射...


| URL | HTTP Method | Action | URL Name|
|-|-|-|-|
|/users | GET | list | user-list |
|/users/{username} | GET | retrieve| user-detail |
|/users/{username}/group-names | GET | group_names | user-group-names |


有关设置 `.routes` 属性的另一个示例，请参阅 `SimpleRouter` 类的源代码。


### 自定义路由器进阶

如果想提供完全自定义的行为，可以继承 `BaseRouter` 并覆盖 `get_urls(self)` 方法。该方法应检查已注册的视图集并返回一组 URL 模式。可以通过访问 `self.registry` 属性来检查注册的 prefix，viewset 和 basename tuples。

你可能还想覆盖 `get_default_base_name（self，viewset）`方法，或者在向路由注册视图集时始终显式设置 `base_name` 参数。



## 第三方软件包

以下是可用的第三方包。


### [DRF Nested Routers](https://github.com/alanjds/drf-nested-routers)


### [ModelRouter (wq.db.rest)](https://wq.io/wq.db)


### [DRF-extensions](https://chibisov.github.io/drf-extensions/docs/)