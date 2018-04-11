> [官方原文链接](http://www.django-rest-framework.org/api-guide/permissions/)  


# 权限


与 authentication 和 throttling 一起，permission 决定是应该接受还是拒绝访问请求。

权限检查总是在视图的最开始处运行，在任何其他代码被允许进行之前。权限检查通常会使用 `request.user` 和 `request.auth` 属性中的认证信息来确定是否允许传入请求。

权限用于授予或拒绝不同类别的用户访问 API 的不同部分。

最简单的权限是允许通过身份验证的用户访问，并拒绝未经身份验证的用户访问。这对应于 REST framework 中的 `IsAuthenticated` 类。

稍微宽松的权限会允许通过身份验证的用户完全访问，而未通过身份验证的用户只能进行只读访问。这对应于 REST framework 中的 `IsAuthenticatedOrReadOnly` 类。

## 如何确定权限

REST framework 中的权限总是被定义为权限类的列表。

在运行视图的主体之前，检查列表中的每个权限。如果任何权限检查失败，则会引发 `exceptions.PermissionDenied` 或 `exceptions.NotAuthenticated` 异常，并且视图的主体不会再运行。

当权限检查失败时，根据以下规则，将返回 “403 Forbidden” 或 “401 Unauthorized” 响应：

* 该请求已成功通过身份验证，但权限被拒绝。 *&mdash; 将返回 403 Forbidden 响应。*
* 该请求未成功通过身份验证，并且最高优先级身份验证类未添加 `WWW-Authenticate` header。*&mdash; 将返回 403 Forbidden 响应。*
* 该请求未成功通过身份验证，不过最高优先级身份验证类添加了 `WWW-Authenticate` header。*&mdash; 返回一个 HTTP 401 Unauthorized 响应，并会带上一个适当的 `WWW-Authenticate` header。*

## 对象级权限

REST framework 权限还支持对象级权限。对象级权限用于确定是否允许用户对特定对象进行操作，该特定对象通常是指模型实例。

`.get_object()` 被调用时，对象级权限由 REST framework 的通用视图执行。与视图级权限一样，如果用户不被允许对给定对象进行操作，则会引发 `exceptions.PermissionDenied` 异常。

如果您正在编写自己的视图并希望强制执行对象级权限，或者如果您在通用视图上重写了 `get_object` 方法，那么将需要显式地在你检索该对象时调用 `.check_object_permissions(request, obj)` 方法。

这将引发 `PermissionDenied` 或 `NotAuthenticated` 异常，或者只是在视图具有适当的权限时才返回。

例如：

``` python
def get_object(self):
    obj = get_object_or_404(self.get_queryset(), pk=self.kwargs["pk"])
    self.check_object_permissions(self.request, obj)
    return obj
```

#### 对象级权限的限制

出于性能原因，通用视图在返回对象列表时不会自动将对象级权限应用于查询集中的每个实例。

通常，当您使用对象级权限时，您还需要适当地过滤查询集，以确保用户只能看到他们被允许查看的实例。

## 设置权限策略

默认权限策略可以使用 `DEFAULT_PERMISSION_CLASSES` setting 全局设置。例如。

``` python
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': (
        'rest_framework.permissions.IsAuthenticated',
    )
}
```

如果未指定，则此设置默认为允许无限制访问：

``` python
'DEFAULT_PERMISSION_CLASSES': (
   'rest_framework.permissions.AllowAny',
)
```

您还可以在基于 `APIView` 类的视图上设置身份验证策略。

``` python
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework.views import APIView

class ExampleView(APIView):
    permission_classes = (IsAuthenticated,)

    def get(self, request, format=None):
        content = {
            'status': 'request was permitted'
        }
        return Response(content)
```

或者在基于 `@api_view` 装饰器的函数视图上设置。

``` python
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response

@api_view(['GET'])
@permission_classes((IsAuthenticated, ))
def example_view(request, format=None):
    content = {
        'status': 'request was permitted'
    }
    return Response(content)
```

__注意：__ 当你通过类属性或装饰器设置新的权限类时，`settings.py` 文件中的默认设置会被忽略。

---

# API 参考

## AllowAny

`AllowAny` 权限类将允许不受限制的访问，而不管该请求是否已通过身份验证或未经身份验证。

也不一定非要用此权限，可以通过为权限设置空列表或元组来实现相同的结果，但是你会发现，使用此权限使意图更加清晰。

## IsAuthenticated

`IsAuthenticated` 权限类将拒绝任何未通过身份验证的用户的访问。

 如果你希望 API 只能由注册用户访问，则可以使用此权限。

## IsAdminUser

`IsAdminUser` 权限仅允许 `user.is_staff` 为 `True` 用户访问，其他任何用户都将被拒绝。

如果你希望 API 只能被部分受信任的管理员访问，则可以使用此权限。

## IsAuthenticatedOrReadOnly

`IsAuthenticatedOrReadOnly` 允许通过身份验证的用户执行任何请求。未通过身份验证的用户只能请求 “安全” 的方法： `GET`， `HEAD` 或 `OPTIONS`。

如果你希望 API 允许匿名用户拥有读取权限，并且只允许对已通过身份验证的用户执行写入权限，则可以使用此权限。

## DjangoModelPermissions

此权限类与 Django 的标准 `django.contrib.auth` 模型权限绑定。此权限只能应用于具有 `.queryset` 属性集的视图。只有在用户通过身份验证并分配了相关模型权限的情况下，才有权限访问。

* `POST` 请求要求用户在模型上具有 `add` 权限。
* `PUT` 和 `PATCH` 请求要求用户在模型上具有 `change` 权限。
* `DELETE` 请求要求用户在模型上具有 `delete` 权限。

默认行为也可以被重写以支持自定义模型权限。例如，你可能想要包含 GET 请求的 `view` 模型权限。

要自定义模型权限，请继承 `DjangoModelPermissions` 并设置 `.perms_map` 属性。有关详细信息，请参阅源代码。

#### 使用不包含 `queryset` 属性的视图。

如果你将此权限与重写 `get_queryset()` 方法的视图一起使用，则视图上可能没有 `queryset` 属性。在这种情况下，我们建议使用 sentinel 查询集标记视图，以便此类可以确定所需的权限。例如：

``` python
queryset = User.objects.none()  # Required for DjangoModelPermissions
```

## DjangoModelPermissionsOrAnonReadOnly

与 `DjangoModelPermissions` 类似，但也允许未经身份验证的用户对 API 进行只读访问。

## DjangoObjectPermissions

该权限类与 Django 的标准对象权限框架绑定，该框架允许对每个模型对象进行权限验证。为了使用此权限类，你还需要添加支持对象级权限的权限后端，例如 [django-guardian](https://github.com/lukaszb/django-guardian)。

与 `DjangoModelPermissions` 一样，此权限只能应用于具有 `.queryset` 属性或 `.get_queryset()` 方法的视图。只有在用户通过身份验证并且具有相关的每个对象权限和相关的模型权限后，才有权限访问。

* `POST` 请求要求用户对模型实例具有 `add` 权限。
* `PUT` 和 `PATCH` 请求要求用户对模型实例具有 `change` 权限。
* `DELETE` 请求要求用户对模型实例具有 `delete` 权限。

请注意，`DjangoObjectPermissions` 不需要 `django-guardian` 软件包，并且同样支持其他对象级别的后端。

与 `DjangoModelPermissions` 一样，你可以通过继承 `DjangoObjectPermissions` 并设置 `.perms_map` 属性来自定义模型权限。有关详细信息，请参阅源代码。

---

**注意**: 如果你需要获取 `GET`，`HEAD` 和 `OPTIONS` 请求的对象级 `view` 权限，则还需要考虑添加 `DjangoObjectPermissionsFilter` 类，以确保列表端点只返回包含用户具有查看权限的对象的结果。

---

---

# 自定义权限

要实现自定义权限，请继承 `BasePermission` 并实现以下方法中的一个或两个：

* `.has_permission(self, request, view)`
* `.has_object_permission(self, request, view, obj)`

如果请求被授予访问权限，则方法应该返回 `True`，否则返回 `False`。

如果你需要测试一个请求是一个读操作还是一个写操作，你应该根据常量 `SAFE_METHODS` 检查请求方法， `SAFE_METHODS` 是一个包含 `'GET'`，`'OPTIONS'` 和 `'HEAD'` 的元组。例如：

``` python
if request.method in permissions.SAFE_METHODS:
    # Check permissions for read-only request
else:
    # Check permissions for write request
```

---

**Note**: 只有在视图级别 `has_permission` 检查已通过时才会调用实例级别的 `has_object_permission` 方法。还要注意，为了运行实例级检查，视图代码应该显式调用 `.check_object_permissions(request, obj)`。如果你使用的是通用视图，那么默认情况下会为您处理。（基于函数的视图将需要明确检查对象权限，在失败时引发 `PermissionDenied`。）

---

如果测试失败，自定义权限将引发 `PermissionDenied` 异常。要更改与异常相关的错误消息，请直接在你的自定义权限上实现 `message` 属性。否则将使用 `PermissionDenied` 的 `default_detail` 属性。

``` python
from rest_framework import permissions

class CustomerAccessPermission(permissions.BasePermission):
    message = 'Adding customers not allowed.'

    def has_permission(self, request, view):
         ...
```

## 举个栗子

以下是一个权限类的示例，该权限类将传入请求的 IP 地址与黑名单进行比对，并在 IP 被列入黑名单时拒绝该请求。

``` python
from rest_framework import permissions

class BlacklistPermission(permissions.BasePermission):
    """
    Global permission check for blacklisted IPs.
    """

    def has_permission(self, request, view):
        ip_addr = request.META['REMOTE_ADDR']
        blacklisted = Blacklist.objects.filter(ip_addr=ip_addr).exists()
        return not blacklisted
```

除了针对所有传入请求运行的全局权限，还可以创建对象级权限，这些权限仅针对影响特定对象实例的操作执行。例如：

``` python
class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    Object-level permission to only allow owners of an object to edit it.
    Assumes the model instance has an `owner` attribute.
    """

    def has_object_permission(self, request, view, obj):
        # Read permissions are allowed to any request,
        # so we'll always allow GET, HEAD or OPTIONS requests.
        if request.method in permissions.SAFE_METHODS:
            return True

        # Instance must have an attribute named `owner`.
        return obj.owner == request.user
```

请注意，通用视图将检查适当的对象级权限，但如果你正在编写自己的自定义视图，则需要确保检查自己的对象级权限。您可以通过在拥有对象实例后从视图中调用 `self.check_object_permissions(request, obj)` 来完成此操作。如果任何对象级权限检查失败，此调用将引发适当的 `APIException`，否则将简单地返回。

另请注意，通用视图将仅检查单个模型实例的视图的对象级权限。如果你需要列表视图的对象级过滤，则需要单独过滤查询集。
