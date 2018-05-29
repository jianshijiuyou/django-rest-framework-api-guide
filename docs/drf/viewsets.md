> [官方原文链接](http://www.django-rest-framework.org/api-guide/viewsets/)

## 视图集

> 在路由决定了哪个控制器用于请求后，控制器负责理解请求并产生适当的输出。     
> — *Ruby on Rails 文档*

Django REST framework 允许将一组相关视图的逻辑组合到一个称为 `ViewSet` 的类中。在其他框架中，您可能会发现概念上类似的实现，名为 “Resources” 或 “Controllers” 。

`ViewSet` 类只是一种基于类的 View，它不提供任何处理方法，如 `.get()` 或 `.post()`，而是提供诸如  `.list()` 和 `.create()` 之类的操作。

`ViewSet` 只在用 `.as_view()` 方法绑定到最终化视图时做一些相应操作。

通常，不是在 urlconf 中的视图集中明确注册视图，而是使用路由器类注册视图集，这会自动为您确定 urlconf。

### 举个栗子

定义一个简单的视图集，可以用来列出或检索系统中的所有用户。  

``` python
from django.contrib.auth.models import User
from django.shortcuts import get_object_or_404
from myapps.serializers import UserSerializer
from rest_framework import viewsets
from rest_framework.response import Response

class UserViewSet(viewsets.ViewSet):

    def list(self, request):
        queryset = User.objects.all()
        serializer = UserSerializer(queryset, many=True)
        return Response(serializer.data)

    def retrieve(self, request, pk=None):
        queryset = User.objects.all()
        user = get_object_or_404(queryset, pk=pk)
        serializer = UserSerializer(user)
        return Response(serializer.data)
```

如果需要，可以将这个视图集合成两个单独的视图，如下所示：

``` python
user_list = UserViewSet.as_view({'get': 'list'})
user_detail = UserViewSet.as_view({'get': 'retrieve'})
```

通常情况下，我们不会这样做，而是用路由器注册视图集，并允许自动生成 urlconf。

``` python
from myapp.views import UserViewSet
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'users', UserViewSet, base_name='user')
urlpatterns = router.urls
```

不用自己编写视图集，通常使用默认提供的现有基类。例如：

``` python
class UserViewSet(viewsets.ModelViewSet):
    """
    用于查看和编辑用户实例的视图。
    """
    serializer_class = UserSerializer
    queryset = User.objects.all()
```

使用 `ViewSet` 类比使用 View 类有两个主要优点。

 * 重复的逻辑可以合并成一个类。在上面的例子中，我们只需要指定一次查询集，它将在多个视图中使用。
 * 通过使用 routers，我们不再需要处理自己的 URL 配置。

这两者各有优缺点。使用常规视图和 URL 配置文件更加明确，并为您提供更多控制。如果想要更快速的开发出一个应用，或者需要使大型 API 的 URL 配置始终保持一致，视图集会非常有用。

### 操作视图集

REST framework 中包含的默认 routes  将为一组标准的  create / retrieve / update / destroy  风格 action 提供路由，如下所示：

``` python
class UserViewSet(viewsets.ViewSet):
    """
    这些方法将由路由器负责处理。

    如果要使用后缀，请确保加上 `format = None` 关键字参数
    """

    def list(self, request):
        pass

    def create(self, request):
        pass

    def retrieve(self, request, pk=None):
        pass

    def update(self, request, pk=None):
        pass

    def partial_update(self, request, pk=None):
        pass

    def destroy(self, request, pk=None):
        pass
```

在调度期间，当前 action 的名称可以通过 `.action` 属性获得。您可以检查 `.action` 以根据当前 action 调整行为。

例如，您可以将权限限制为只有 admin 才能访问 `list` 以外的其他 action，如下所示：

``` python
def get_permissions(self):
    """
    实例化并返回此视图所需的权限列表。
    """
    if self.action == 'list':
        permission_classes = [IsAuthenticated]
    else:
        permission_classes = [IsAdmin]
    return [permission() for permission in permission_classes]
```

### 标记额外的路由行为

如果需要路由特定方法，则可以用 `@detail_route` 或 `@list_route` 装饰器进行修饰。

`@detail_route` 装饰器在其 URL 模式中包含 `pk`，用于支持需要获取单个实例的方法。`@list_route` 修饰器适用于在对象列表上操作的方法。

举个栗子：

``` python
from django.contrib.auth.models import User
from rest_framework import status
from rest_framework import viewsets
from rest_framework.decorators import detail_route, list_route
from rest_framework.response import Response
from myapp.serializers import UserSerializer, PasswordSerializer

class UserViewSet(viewsets.ModelViewSet):
    """
    提供标准操作的视图集
    """
    queryset = User.objects.all()
    serializer_class = UserSerializer

    @detail_route(methods=['post'])
    def set_password(self, request, pk=None):
        user = self.get_object()
        serializer = PasswordSerializer(data=request.data)
        if serializer.is_valid():
            user.set_password(serializer.data['password'])
            user.save()
            return Response({'status': 'password set'})
        else:
            return Response(serializer.errors,
                            status=status.HTTP_400_BAD_REQUEST)

    @list_route()
    def recent_users(self, request):
        recent_users = User.objects.all().order('-last_login')

        page = self.paginate_queryset(recent_users)
        if page is not None:
            serializer = self.get_serializer(page, many=True)
            return self.get_paginated_response(serializer.data)

        serializer = self.get_serializer(recent_users, many=True)
        return Response(serializer.data)
```

另外，装饰器可以为路由视图设置额外的参数。例如...

``` python
    @detail_route(methods=['post'], permission_classes=[IsAdminOrIsSelf])
    def set_password(self, request, pk=None):
       ...
```

这些装饰器默认路由 `GET` 请求，但也可以使用 `methods` 参数接受其他 HTTP 方法。例如：

``` python
    @detail_route(methods=['post', 'delete'])
    def unset_password(self, request, pk=None):
       ...
```

这两个新操作将在 urls  `^users/{pk}/set_password/$` 和 `^users/{pk}/unset_password/$` 上。

### action 跳转

如果你需要获取 action 的 URL ，请使用 `.reverse_action()` 方法。这是 `.reverse()` 的一个便捷包装，它会自动传递视图的请求对象，并将 `url_name` 与 `.basename` 属性挂接。

请注意，`basename` 是在 `ViewSet` 注册过程中由路由器提供的。如果您不使用路由器，则必须提供`.as_view()` 方法的 `basename` 参数。

使用上一节中的示例：

``` python
>>> view.reverse_action('set-password', args=['1'])
'http://localhost:8000/api/users/1/set_password'
```

`url_name` 参数应该与 `@list_route` 和 `@detail_route` 装饰器的相同参数匹配。另外，这可以用来反转默认 `list` 和 `detail` 路由。


## API 参考


### ViewSet

`ViewSet` 类继承自 `APIView`。您可以使用任何标准属性（如 `permission_classes`，`authentication_classes`）来控制视图上的 API 策略。

`ViewSet` 类不提供任何 action 的实现。为了使用 `ViewSet` 类，必须继承该类并明确定义 action 实现。

### GenericViewSet

`GenericViewSet` 类继承自 `GenericAPIView`，并提供默认的 `get_object`，`get_queryset` 方法和其他通用视图基础行为，但默认情况下不包含任何操作。

为了使用 `GenericViewSet` 类，必须继承该类并混合所需的 mixin 类，或明确定义操作实现。

### ModelViewSet

`ModelViewSet` 类继承自 `GenericAPIView`，并通过混合各种 mixin 类的行为来包含各种操作的实现。

`ModelViewSet` 提供的操作有 `.list()` , `.retrieve()` , `.create()` , `.update()` , `.partial_update()`, 和 `.destroy()` 。


举个栗子：

由于 `ModelViewSet` 类继承自 `GenericAPIView`，因此通常需要提供至少 `queryset` 和 `serializer_class` 属性。例如：

``` python
class AccountViewSet(viewsets.ModelViewSet):
    """
    用于查看和编辑 Account
    """
    queryset = Account.objects.all()
    serializer_class = AccountSerializer
    permission_classes = [IsAccountAdminOrReadOnly]
```

请注意，您可以覆盖 `GenericAPIView` 提供的任何标准属性或方法。例如，要动态确定它应该操作的查询集的`ViewSet`，可以这样做：

``` python
class AccountViewSet(viewsets.ModelViewSet):
    """
    A simple ViewSet for viewing and editing the accounts
    associated with the user.
    """
    serializer_class = AccountSerializer
    permission_classes = [IsAccountAdminOrReadOnly]

    def get_queryset(self):
        return self.request.user.accounts.all()
```


但请注意，从 `ViewSet` 中删除 `queryset` 属性后，任何关联的 router 将无法自动导出模型的 `base_name`，因此您必须将 `base_name` kwarg 指定为 router 注册的一部分。

还要注意，虽然这个类默认提供了完整的  create / list / retrieve / update / destroy  操作集，但您可以通过使用标准权限类来限制可用操作。


### ReadOnlyModelViewSet

 `ReadOnlyModelViewSet` 类也从 `GenericAPIView` 继承。与 `ModelViewSet` 一样，它也包含各种操作的实现，但与 `ModelViewSet` 不同的是它只提供 “只读” 操作，`.list()` 和 `.retrieve()`。

举个栗子：

与 `ModelViewSet` 一样，您通常需要提供至少 `queryset` 和 `serializer_class` 属性。例如：

``` python
class AccountViewSet(viewsets.ReadOnlyModelViewSet):
    """
    A simple ViewSet for viewing accounts.
    """
    queryset = Account.objects.all()
    serializer_class = AccountSerializer
```

同样，与 `ModelViewSet` 一样，您可以覆盖`GenericAPIView` 可用的任何标准属性和方法。


## 自定义视图集基类


您可能需要使用没有完整 `ModelViewSet` 操作集的自定义 `ViewSet` 类，或其他自定义行为。

举个栗子：

要创建提供 `create`， `list` 和 `retrieve` 操作的基本视图集类，请从 `GenericViewSet` 继承，并混合（mixin ）所需的操作：

``` python
from rest_framework import mixins

class CreateListRetrieveViewSet(mixins.CreateModelMixin,
                                mixins.ListModelMixin,
                                mixins.RetrieveModelMixin,
                                viewsets.GenericViewSet):
    """
    A viewset that provides `retrieve`, `create`, and `list` actions.

    To use it, override the class and set the `.queryset` and
    `.serializer_class` attributes.
    """
    pass
```

通过创建自己的基本 `ViewSet` 类，能够提供可在 API 中的多个视图集中重用的常见行为。