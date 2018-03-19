> [官方原文链接](http://www.django-rest-framework.org/api-guide/filtering/)  


# 过滤


REST framework 的通用列表视图的默认行为是从模型管理器返回整个查询集。通常你会希望 API 限制查询集返回的条目。

筛选 `GenericAPIView` 子类的查询集的最简单方法是重写 `.get_queryset()` 方法。

重写此方法允许你以多种不同方式自定义视图返回的查询集。

## 根据当前用户进行过滤

你可能需要过滤查询集，以确保只返回与当前通过身份验证的用户发出的请求相关的结果。

你可以基于 `request.user` 的值进行筛选来完成此操作。

例如：

``` python
from myapp.models import Purchase
from myapp.serializers import PurchaseSerializer
from rest_framework import generics

class PurchaseList(generics.ListAPIView):
    serializer_class = PurchaseSerializer

    def get_queryset(self):
        """
        This view should return a list of all the purchases
        for the currently authenticated user.
        """
        user = self.request.user
        return Purchase.objects.filter(purchaser=user)
```


## 根据 URL 进行过滤

另一种过滤方式可能涉及基于 URL 的某个部分限制查询集。

例如，如果你的 URL 配置包含这样的条目：

``` python
url('^purchases/(?P<username>.+)/$', PurchaseList.as_view()),
```

然后，你可以编写一个视图，返回由 URL 的用户名部分过滤的 purchase 查询集：

``` python
class PurchaseList(generics.ListAPIView):
    serializer_class = PurchaseSerializer

    def get_queryset(self):
        """
        This view should return a list of all the purchases for
        the user as determined by the username portion of the URL.
        """
        username = self.kwargs['username']
        return Purchase.objects.filter(purchaser__username=username)
```

## 根据查询参数进行过滤

过滤初始查询集的最后一个例子是根据 url 中的查询参数确定初始查询集。

我们可以覆盖 `.get_queryset()` 来处理诸如 `http://example.com/api/purchases?username=denvercoder9` 的URL，并且只有在 URL 中包含 `username` 参数时才过滤查询集：

``` python
class PurchaseList(generics.ListAPIView):
    serializer_class = PurchaseSerializer

    def get_queryset(self):
        """
        Optionally restricts the returned purchases to a given user,
        by filtering against a `username` query parameter in the URL.
        """
        queryset = Purchase.objects.all()
        username = self.request.query_params.get('username', None)
        if username is not None:
            queryset = queryset.filter(purchaser__username=username)
        return queryset
```

---

# 通用过滤器

除了能够覆盖默认的查询集外，REST framework 还包括对通用过滤后端的支​​持，使你可以轻松构建复杂的搜索和过滤器。

通用过滤器也可以在可浏览的 API 和管理 API 中将自己渲染为 HTML 控件。

![Filter Example](http://www.django-rest-framework.org/img/filter-controls.png)

## 设置过滤器后端

可以使用 `DEFAULT_FILTER_BACKENDS` setting 全局设置默认的过滤器后端。例如。

``` python
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': ('django_filters.rest_framework.DjangoFilterBackend',)
}
```

你还可以使用基于 `GenericAPIView` 类的视图，在每个视图或视图集的基础上设置过滤器后端。

``` python
import django_filters.rest_framework
from django.contrib.auth.models import User
from myapp.serializers import UserSerializer
from rest_framework import generics

class UserListView(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    filter_backends = (django_filters.rest_framework.DjangoFilterBackend,)
```

## 过滤和对象查找

请注意，如果为一个视图配置了一个过滤器后端，那么除了用于筛选列表视图之外，它还将用于筛选返回单个对象的查询集。

例如，根据前面的示例以及 ID 为 `4675` 的产品，以下 URL 将返回相应的对象，或返回 404 响应，具体取决于给定产品实例是否满足过滤条件：

``` python
http://example.com/api/products/4675/?category=clothing&max_price=10.00
```

## 覆盖初始查询集

请注意，你可以同时重写的 `.get_queryset()` 和通用过滤，并且所有内容都将按预期工作。例如，如果产品与用户具有多对多关系，则可能需要编写一个如下所示的视图：

``` python
class PurchasedProductsList(generics.ListAPIView):
    """
    Return a list of all the products that the authenticated
    user has ever purchased, with optional filtering.
    """
    model = Product
    serializer_class = ProductSerializer
    filter_class = ProductFilter

    def get_queryset(self):
        user = self.request.user
        return user.purchase_set.all()
```

---

# API 参考

## DjangoFilterBackend

`django-filter` 库包含一个 `DjangoFilterBackend` 类，它支持 REST framework 对字段过滤进行高度定制。

要使用 `DjangoFilterBackend`，首先安装 `django-filter`。然后将 `django_filters` 添加到 Django 的 `INSTALLED_APPS` 中

``` python
pip install django-filter
```

你现在应该将过滤器后端添加到设置中：

``` python
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': ('django_filters.rest_framework.DjangoFilterBackend',)
}
```

或者将过滤器后端添加到单个视图或视图集。

``` python
from django_filters.rest_framework import DjangoFilterBackend

class UserListView(generics.ListAPIView):
    ...
    filter_backends = (DjangoFilterBackend,)
```

如果你只需要简单的基于等式的过滤，则可以在视图或视图集上设置 `filter_fields` 属性，列出你要过滤的一组字段。

``` python
class ProductList(generics.ListAPIView):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    filter_backends = (DjangoFilterBackend,)
    filter_fields = ('category', 'in_stock')
```

这将自动为给定字段创建一个 `FilterSet` 类，并允许你发出如下请求：

``` python
http://example.com/api/products?category=clothing&in_stock=True
```

对于更高级的过滤要求，你应该在视图上在指定 `FilterSet` 类。你可以在 [django-filter 文档](https://django-filter.readthedocs.io/en/latest/index.html)中阅读有关 `FilterSet` 的更多信息。还建议你阅读 [DRF integration](https://django-filter.readthedocs.io/en/latest/guide/rest_framework.html)。


## SearchFilter

`SearchFilter` 类支持简单的基于单个查询参数的搜索，并且基于 Django 管理员的搜索功能。

在使用时，可浏览的 API 将包含一个 `SearchFilter` 控件：

![Search Filter](http://www.django-rest-framework.org/img/search-filter.png)

`SearchFilter` 类将仅在视图具有 `search_fields` 属性集的情况下应用。`search_fields` 属性应该是模型上文本类型字段的名称列表，例如 `CharField` 或 `TextField`。

``` python
class UserListView(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    filter_backends = (filters.SearchFilter,)
    search_fields = ('username', 'email')
```

这将允许客户端通过查询来过滤列表中的项目，例如：

```
http://example.com/api/users?search=russell
```

你还可以使用查找 API 双下划线表示法对 ForeignKey 或 ManyToManyField 执行相关查找：

``` python
search_fields = ('username', 'email', 'profile__profession')
```

默认情况下，搜索将使用不区分大小写的部分匹配。搜索参数可能包含多个搜索词，它们应该是空格和（或）逗号分隔的。如果使用多个搜索条件，则只有在所有提供的条件匹配的情况下，对象才会返回到列表中。

搜索行为可以通过将各种字符预先添加到 `search_fields` 来限制。

* '^' 匹配起始部分。
* '=' 完全匹配。
* '@' 全文搜索。（目前只支持 Django 的 MySQL 后端。）
* '$' 正则匹配。

例如：

``` python
search_fields = ('=username', '=email')
```

默认情况下，搜索参数被命名为 `'search'` ，但这可能会被 `SEARCH_PARAM` setting 覆盖。

有关更多详细信息，请参阅 [Django 文档](https://docs.djangoproject.com/en/stable/ref/contrib/admin/#django.contrib.admin.ModelAdmin.search_fields)。

---

## OrderingFilter

`OrderingFilter` 类支持简单查询参数控制结果的排序。

![Ordering Filter](http://www.django-rest-framework.org/img/ordering-filter.png)

默认情况下，查询参数被命名为 `'ordering'`，但这可能会被 `ORDERING_PARAM` setting 覆盖。

例如，要通过 username 对用户排序：

```
http://example.com/api/users?ordering=username
```

客户端也可以通过在字段名称前加 ' - ' 来指定反向排序，如下所示：

```
http://example.com/api/users?ordering=-username 
```

也可以指定多个排序：

```
http://example.com/api/users?ordering=account,username
```

### 指定可以根据哪些字段进行排序

建议你明确指定 API 应该允许在排序过滤器中使用哪些字段。你可以通过在视图上设置一个 `ordering_fields` 属性来完成此操作，如下所示：

``` python
class UserListView(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    filter_backends = (filters.OrderingFilter,)
    ordering_fields = ('username', 'email')
```

这有助于防止意外的数据泄漏，例如允许用户根据密码哈希字段或其他敏感数据进行排序。

如果你未在视图上指定 `ordering_fields` 属性，则过滤器类将默认允许用户过滤由 `serializer_class` 属性指定的序列化类中的任何可读字段。

如果你确信视图使用的查询集不包含任何敏感数据，则还可以通过使用特殊值 `'__all__'` 明确指定视图允许在任何模型字段或查询集聚合上进行排序。

``` python
class BookingsListView(generics.ListAPIView):
    queryset = Booking.objects.all()
    serializer_class = BookingSerializer
    filter_backends = (filters.OrderingFilter,)
    ordering_fields = '__all__'
```

### 指定默认顺序

如果在视图上设置了 `ordering` 属性，则将用作默认排序。

通常情况下，你应该通过在初始查询集上设置 `order_by` 来控制此操作，但是通过在视图上使用 `ordering` 参数，你可以指定排序方式，然后可以将其作为上下文自动传递到渲染的模板。这可以自动渲染列标题，如果它们用于排序结果。

``` python
class UserListView(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    filter_backends = (filters.OrderingFilter,)
    ordering_fields = ('username', 'email')
    ordering = ('username',)
```

`ordering` 属性可以是一个字符串或者字符串列表（元组）。

---

## DjangoObjectPermissionsFilter

`DjangoObjectPermissionsFilter` 旨在与 [`django-guardian`](https://django-guardian.readthedocs.io/) 软件包一起使用，添加了自定义 `'view'` 的权限。过滤器将确保查询集仅返回用户具有适当查看权限的对象。

如果你使用的是 `DjangoObjectPermissionsFilter`，那么你可能还需要添加适当的对象权限类，以确保用户只有在具有适当对象权限的情况下才能对实例进行操作。做到这一点的最简单方法是继承 `DjangoObjectPermissions` 并为 `perms_map` 属性添加 `'view'` 权限。

使用 `DjangoObjectPermissionsFilter` 和 `DjangoObjectPermissions` 的完整示例可能如下所示。

**permissions.py**:

``` python
class CustomObjectPermissions(permissions.DjangoObjectPermissions):
    """
    Similar to `DjangoObjectPermissions`, but adding 'view' permissions.
    """
    perms_map = {
        'GET': ['%(app_label)s.view_%(model_name)s'],
        'OPTIONS': ['%(app_label)s.view_%(model_name)s'],
        'HEAD': ['%(app_label)s.view_%(model_name)s'],
        'POST': ['%(app_label)s.add_%(model_name)s'],
        'PUT': ['%(app_label)s.change_%(model_name)s'],
        'PATCH': ['%(app_label)s.change_%(model_name)s'],
        'DELETE': ['%(app_label)s.delete_%(model_name)s'],
    }
```

**views.py**:

``` python
class EventViewSet(viewsets.ModelViewSet):
    """
    Viewset that only lists events if user has 'view' permissions, and only
    allows operations on individual events if user has appropriate 'view', 'add',
    'change' or 'delete' permissions.
    """
    queryset = Event.objects.all()
    serializer_class = EventSerializer
    filter_backends = (filters.DjangoObjectPermissionsFilter,)
    permission_classes = (myapp.permissions.CustomObjectPermissions,)
```



---

# 自定义通用过滤器

你还可以提供自己的通用过滤器后端，或者编写一个可供其他开发人员使用的可安装应用程序。

为此，请继承 `BaseFilterBackend`，并覆盖 `.filter_queryset(self, request, queryset, view)` 方法。该方法应该返回一个新的，过滤的查询集。

除了允许客户端执行搜索和过滤外，通用过滤器后端可用于限制哪些对象应该对给定的请求或用户可见。

## 举个栗子

你可能需要限制用户只能看到他们创建的对象。

``` python
class IsOwnerFilterBackend(filters.BaseFilterBackend):
    """
    Filter that only allows users to see their own objects.
    """
    def filter_queryset(self, request, queryset, view):
        return queryset.filter(owner=request.user)
```

我们可以通过覆盖视图上的 `get_queryset()`来实现相同的行为，但使用过滤器后端允许你更轻松地将此限制添加到多个视图，或者将其应用于整个 API。

## 自定义接口

通用过滤器也可以在可浏览的 API 中渲染接口。为此，你应该实现一个 `to_html()` 方法，该方法返回过滤器的渲染 HTML 表示。此方法应具有以下签名：

`to_html(self, request, queryset, view)`

该方法应该返回一个渲染的 HTML 字符串。

## Pagination & schemas

通过实现 `get_schema_fields()` 方法，你还可以使过滤器控件可用于 REST framework 提供的模式自动生成。此方法应具有以下签名：

`get_schema_fields(self, view)`

该方法应该返回一个 `coreapi.Field` 实例列表。
