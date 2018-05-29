> [官方原文链接](http://www.django-rest-framework.org/api-guide/generic-views/)

## 通用视图

基于类的视图的一个主要优点是它们允许你编写可重复使用的行为。 REST framework 通过提供大量预构建视图来提供常用模式，从而充分利用了这一点。

REST framework  提供的通用视图允许您快速构建紧密映射到数据库模型的 API 视图。

如果通用视图不符合需求，可以使用常规的 `APIView` 类，或者利用 mixin 特性和基类组合出可重用的视图。

### 举个栗子

通常，在使用通用视图时，您需要继承该视图，并设置几个类属性。  

``` python
from django.contrib.auth.models import User
from myapp.serializers import UserSerializer
from rest_framework import generics
from rest_framework.permissions import IsAdminUser

class UserList(generics.ListCreateAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = (IsAdminUser,)
```

对于更复杂的情况，您可能还想重写视图类中的各种方法。例如。

``` python
class UserList(generics.ListCreateAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = (IsAdminUser,)

    def list(self, request):
        # 注意使用`get_queryset()`而不是`self.queryset`
        queryset = self.get_queryset()
        serializer = UserSerializer(queryset, many=True)
        return Response(serializer.data)
```

对于非常简单的情况，您可能想要使用 `.as_view()` 方法来传递类属性。例如，您的 URLconf 可能包含类似于以下条目的内容：

``` python
url(r'^/users/', ListCreateAPIView.as_view(queryset=User.objects.all(), serializer_class=UserSerializer), name='user-list')
```

> 直接在 URLconf 中设置相关属性参数，这样连视图类都不用写了。



## API 参考

### GenericAPIView

`GenericAPIView` 类继承于 REST framework 的 `APIView` 类，为标准列表和详细视图添加了常见的行为。

内置的每一个具体的通用视图都是通过将 `GenericAPIView` 类和一个或多个 minxin 类相互结合来构建的。

#### 属性

##### 基本设置

以下属性控制基本视图行为。

* `queryset` - 用于从此视图返回对象的查询集。通常，您必须设置此属性，或覆盖 `get_queryset()`方法。如果你重写了一个视图方法，在视图方法中，你应该调用 `get_queryset()` 而不是直接访问这个属性，这一点很重要！因为 REST 会在内部对 `queryset` 的结果进行缓存用于后续所有请求。
* `serializer_class` - 用于验证和反序列化输入以及序列化输出的序列化类。通常，您必须设置此属性，或覆盖 `get_serializer_class()` 方法。
* `lookup_field` - 用于执行各个模型实例的对象查找的模型字段。默认为 `'pk'`。请注意，使用 hyperlinked  API 时，如果需要使用自定义值，则需要确保 API 视图和序列化类设置了  lookup field。
* `lookup_url_kwarg` - 用于对象查找的 URL 关键字参数。URL conf 应该包含与此值相对应的关键字参数。如果未设置，则默认使用与 `lookup_field` 相同的值。

关于后两个参数，我这里附上对象查询的源码大家就应该了解了。

省略部分代码，方便理解

``` python
def get_object(self):
	# 先获取数据集
    queryset = self.filter_queryset(self.get_queryset())
	# 拿到查询参数的 key
    lookup_url_kwarg = self.lookup_url_kwarg or self.lookup_field
	# 组装成 {key:value} 的形式
    filter_kwargs = {self.lookup_field: self.kwargs[lookup_url_kwarg]}
    # 查询
    obj = get_object_or_404(queryset, **filter_kwargs)
	# 最后返回
    return obj
```

##### 分页

与列表视图一起使用时，以下属性用于控制分页。

 * `pagination_class` - 对列表进行分页时使用的分页类。默认值与 `DEFAULT_PAGINATION_CLASS` 设置相同，即 `rest_framework.pagination.PageNumberPagination`。设置 `pagination_class = None` 将禁用此视图的分页。

##### 过滤

 * `filter_backends` - 用于过滤查询集的过滤器类的列表。默认值与 `DEFAULT_FILTER_BACKENDS` 设置的值相同。


#### 方法

##### 基本方法


`get_queryset(self)`

应该返回列表视图的查询集，并应该将其用作查看详细视图的基础。默认返回由 `queryset` 属性指定的查询集。

应该始终使用此方法， 而不是直接访问 `self.queryset`，因为 REST 会在内部对 `self.queryset` 的结果进行缓存用于后续所有请求。

可以覆盖以提供动态行为，例如针对不同用户的请求返回不同的数据。

举个栗子：
``` python
def get_queryset(self):
    user = self.request.user
    return user.accounts.all()
```

`get_object(self)`

应该返回详细视图的对象实例。默认使用 `lookup_field` 参数来过滤基本查询集。

可以覆盖以提供更复杂的行为，例如基于多个 URL kwarg 的对象查找。

举个栗子：
``` python
def get_object(self):
    queryset = self.get_queryset()
    filter = {}
    for field in self.multiple_lookup_fields:
        filter[field] = self.kwargs[field]

    obj = get_object_or_404(queryset, **filter)
    self.check_object_permissions(self.request, obj)
    return obj
```

请注意，如果您的 API 不包含任何对象级权限，您可以选择排除 `self.check_object_permissions`，并简单地从 `get_object_or_404` 中查找返回对象。


`filter_queryset(self, queryset)`

给定一个查询集，使用过滤器进行过滤，返回一个新的查询集。

举个栗子：

``` python
def filter_queryset(self, queryset):
    filter_backends = (CategoryFilter,)

    if 'geo_route' in self.request.query_params:
        filter_backends = (GeoRouteFilter, CategoryFilter)
    elif 'geo_point' in self.request.query_params:
        filter_backends = (GeoPointFilter, CategoryFilter)

    for backend in list(filter_backends):
        queryset = backend().filter_queryset(self.request, queryset, view=self)

    return queryset
```

`get_serializer_class(self)`

返回用于序列化的类。默认返回 `serializer_class` 属性。

可以被覆盖以提供动态行为，例如使用不同的序列化器进行读写操作，或为不同类型的用户提供不同的序列化器。

举个栗子：

``` python
def get_serializer_class(self):
    if self.request.user.is_staff:
        return FullAccountSerializer
    return BasicAccountSerializer
```

##### 保存和删除钩子（hook）

以下方法由 mixin 类提供，可以很轻松的重写对象的保存和删除行为。

 * `perform_create(self, serializer)` - 保存新对象实例时由 `CreateModelMixin` 调用。
 * `perform_update(self, serializer)` - 在保存现有对象实例时由 `UpdateModelMixin` 调用。
 * `perform_destroy(self, instance)` - 删除对象实例时由 `DestroyModelMixin` 调用。

这些钩子（hook）对设置请求中隐含的但不属于请求数据的属性特别有用。例如，您可以根据请求用户或基于 URL 关键字参数在对象上设置属性。

``` python
def perform_create(self, serializer):
    serializer.save(user=self.request.user)
```
这些覆盖点对于添加保存对象之前或之后发生的行为（如发送确认电子邮件或记录更新）也特别有用。

``` python
def perform_create(self, serializer):
    queryset = SignupRequest.objects.filter(user=self.request.user)
    if queryset.exists():
        raise ValidationError('You have already signed up')
    serializer.save(user=self.request.user)
```

注意：这些方法替代旧式版本2.x `pre_save`，`post_save`，`pre_delete` 和 `post_delete` 方法，这些方法不再可用。


##### 其他方法

通常不需要重写以下方法，但如果使用 `GenericAPIView` 编写自定义视图，则可能需要调用它们。

 * `get_serializer_context(self)` - 返回包含应该提供给序列化的任何额外上下文的字典。默认包括 `'request'`, `'view'` 和 `'format'` 键。
 * `get_serializer(self, instance=None, data=None, many=False, partial=False)` - 返回一个序列化器实例。
 * `get_paginated_response(self, data)` - 返回分页样式的 `Response` 对象。
 * `paginate_queryset(self, queryset)` - 根据需要为查询集分页，或者返回一个页面对象；如果没有为该视图配置分页，则为 `None`。
 * `filter_queryset(self, queryset)` - 给定一个查询集，使用过滤器进行过滤，返回一个新的查询集。


## Mixins 

mixin 类用于提供基本视图行为的操作。请注意，mixin 类提供了操作方法，而不是直接定义处理方法，如 `.get()` 和 `.post()`。这允许更灵活的行为组合。

mixin 类可以从 `rest_framework.mixins` 中导入。

### ListModelMixin

提供一个 `.list(request, *args, **kwargs)` 方法，实现了列出一个查询集。

如果查询集已填充，则返回 `200 OK` 响应，并将 queryset 的序列化表示形式作为响应的主体。响应数据可以设置分页。


### CreateModelMixin

提供 `.create(request, *args, **kwargs)` 方法，实现创建和保存新模型实例。

如果创建了一个对象，则会返回一个 `201 Created` 响应，并将该对象的序列化表示形式作为响应的主体。如果表示包含名为 url 的键，则响应的 `Location` header  将填充该值。

如果为创建对象提供的请求数据无效，则将返回 `400 Bad Request` 响应，并将错误细节作为响应的主体。

### RetrieveModelMixin

提供 `.retrieve(request, *args, **kwargs)` 方法，该方法实现在响应中返回现有的模型实例。

如果可以获取对象，则返回 `200 OK` 响应，并将对象的序列化表示作为响应的主体。否则，将返回一个 `404 Not Found`。



### UpdateModelMixin

提供 `.update(request, *args, **kwargs)` 方法，实现更新和保存现有模型实例。还提供了一个 `.partial_update(request, *args, **kwargs)` 方法，它与更新方法类似，只是更新的所有字段都是可选的。这允许支持 HTTP `PATCH` 请求。

如果成功更新对象，则返回 `200 OK` 响应，并将对象的序列化表示形式作为响应的主体。

如果提供的用于更新对象的请求数据无效，则将返回 `400 Bad Request` 响应，并将错误细节作为响应的主体。


### DestroyModelMixin

提供一个 `.destroy(request, *args, **kwargs)` 方法，实现现有模型实例的删除。

如果一个对象被删除，则返回一个 `204 No Content` ，否则它将返回一个 `404 Not Found`。



## 内置视图类列表

以下类是具体的通用视图。通常情况下，你应该都是使用的它们，除非需要高度的自定义行为。


这些视图类可以从 `rest_framework.generics` 中导入。


### CreateAPIView

仅用于创建实例。

提供一个 `post` 请求的处理方法。

继承自： `GenericAPIView `, `CreateModelMixin`

### ListAPIView

仅用于读取模型实例列表。

提供一个 `get` 请求的处理方法。

继承自： `GenericAPIView `, `ListModelMixin`

### RetrieveAPIView

仅用于查询单个模型实例。

提供一个 `get` 请求的处理方法。

继承自： `GenericAPIView `, `RetrieveModelMixin`

### DestroyAPIView

仅用于删除单个模型实例。

提供一个 `delete` 请求的处理方法。

继承自： `GenericAPIView `, `DestroyModelMixin`


### UpdateAPIView

仅用于更新单个模型实例。

提供 `put` 和 `patch` 请求的处理方法。

继承自： `GenericAPIView `, `UpdateModelMixin`

### ListCreateAPIView

既可以获取也可以实例集合，也可以创建实例列表

提供 `get` 和 `post` 请求的处理方法。

继承自： `GenericAPIView `, ` ListModelMixin`，`CreateModelMixin`

### RetrieveUpdateAPIView

既可以查询也可以更新单个实例

提供 `get`，`put` 和 `patch` 请求的处理方法。

继承自： `GenericAPIView `, ` RetrieveModelMixin`，`UpdateModelMixin`

### RetrieveDestroyAPIView

既可以查询也可以删除单个实例

提供 `get` 和 `delete` 请求的处理方法。

继承自： `GenericAPIView `, ` RetrieveModelMixin`，`DestroyModelMixin`

### RetrieveUpdateDestroyAPIView

同时支持查询，更新，删除

提供 `get`，`put`，`patch` 和 `delete` 请求的处理方法。

继承自： `GenericAPIView `, ` RetrieveModelMixin`，`UpdateModelMixin`，`DestroyModelMixin`

## 自定义通用视图类

通常你会想使用现有的通用视图，然后稍微定制一下行为。如果您发现自己在多个地方重复使用了一些自定义行为，则可能需要将行为重构为普通类，然后根据需要将其应用于任何视图或视图集。



### 自定义 mixins

例如，如果您需要根据 URL conf 中的多个字段查找对象，则可以创建一个 mixin 类。

举个栗子：

``` python
class MultipleFieldLookupMixin(object):
    """
    将此 mixin 应用于任何视图或视图集以获取多个字段过滤
    基于`lookup_fields`属性，而不是默认的单个字段过滤。
    """
    def get_object(self):
        queryset = self.get_queryset()             # 获取基本的查询集
        queryset = self.filter_queryset(queryset)  # 使用过滤器
        filter = {}
        for field in self.lookup_fields:
            if self.kwargs[field]: # 忽略空字段
                filter[field] = self.kwargs[field]
        obj = get_object_or_404(queryset, **filter)  # 查找对象
        self.check_object_permissions(self.request, obj)
        return obj
```

随后可以在需要应用自定义行为的任​​何时候，将该 mixin 应用于视图或视图集。

``` python
class RetrieveUserView(MultipleFieldLookupMixin, generics.RetrieveAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    lookup_fields = ('account', 'username')
```

如果您需要使用自定义行为，那么使用自定义 mixins 是一个不错的选择。


### 自定义基类

如果您在多个视图中使用 mixin，您可以进一步创建自己的一组基本视图，然后在整个项目中使用它们。例如：

``` python
class BaseRetrieveView(MultipleFieldLookupMixin,
                       generics.RetrieveAPIView):
    pass

class BaseRetrieveUpdateDestroyView(MultipleFieldLookupMixin,
                                    generics.RetrieveUpdateDestroyAPIView):
    pass
```

如果自定义行为始终需要在整个项目中的大量视图中重复使用，那么使用自定义基类是一个不错的选择。


