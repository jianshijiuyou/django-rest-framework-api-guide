# Django 中的身份认证

Django 带有一个用户认证系统。用于处理用户帐户，组，权限和基于 cookie 的用户会话。本文档的这一部分解释了默认实现如何开箱即用，以及如何扩展和定制它以适应您的项目需求。

## 概览

认证系统由以下部分组成：

* 用户(Users)
* 权限(Permissions)：标志指定用户是否可以执行特定任务。
* 组(Groups)：将标签和权限应用于多个用户的通用方式。
* 一个可配置的密码散列系统
* 表单和查看工具，用于登录用户或限制内容
* 可插入的后端系统

Django 中的认证系统的目标是非常通用，并且不提供 Web 认证系统中常见的一些功能。第三方软件包已经实施了一些常见问题的解决方案：

* 密码强度检查
* 限制登录尝试
* 针对第三方的身份认证（例如，OAuth）

## 安装

身份认证 ( Authentication ) 在 `django.contrib.auth` 中作为 Django contrib 模块捆绑在一起。默认情况下，所需的配置已包含在由 `django-admin startproject` 生成的 `settings.py` 中，这些配置由 `INSTALLED_APPS` setting 中列出的两个项目组成：

1. `'django.contrib.auth'` 包含身份认证 ( authentication ) 框架的核心，以及它的默认模型。
2. `'django.contrib.contenttypes'` 是 Django 内容类型系统，它允许权限与您创建的模型相关联。

以及 `MIDDLEWARE` setting 中的这些项目：

1. `SessionMiddleware` 管理跨请求的会话。
2. `AuthenticationMiddleware` 使用会话将用户与请求相关联。

使用这些设置后，运行命令 `manage.py migrate` 将为 auth 相关模型创建必要的数据库表，并为安装的应用程序中定义的模型创建权限。

# 使用 Django 身份认证系统

本文档解释了 Django 认证系统在其默认配置中的用法。这种配置已经发展到满足最常见的项目需求，处理合理范围广泛的任务，并且仔细实现了密码和权限。对于认证需求与默认不同的项目，Django 支持广泛的扩展和定制认证。

Django 中的认证系统认证和授权在一起，因为这些功能有些耦合。

## User 对象

`User` 对象是认证系统的核心。他们通常代表与您的网站进行互动的人，并用于实现限制访问，注册用户信息，将内容与创作者相关联等。在 Django 的身份验证框架中只存在一类用户，即 `'superusers'` 或 admin `'staff'` 用户是具有特殊属性集的用户对象，而不是不同类的用户对象。

默认用户的主要属性是：

* `username`
* `password`
* `email`
* `first_name`
* `last_name`

### 创建用户

创建用户最直接的方法是使用包含的 `create_user()` 辅助函数：

``` python
>>> from django.contrib.auth.models import User
>>> user = User.objects.create_user('john', 'lennon@thebeatles.com', 'johnpassword')

# At this point, user is a User object that has already been saved
# to the database. You can continue to change its attributes
# if you want to change other fields.
>>> user.last_name = 'Lennon'
>>> user.save()
```

### 创建超级用户

使用 `createsuperuser` 命令创建超级用户：

``` python
$ python manage.py createsuperuser --username=joe --email=joe@example.com
```

系统会提示您输入密码。输入之后，用户将立即被创建。如果您不使用 `--username` 或 `--email` 选项，它会提示您输入这些值。

### 更改密码

Django 不会在用户模型中存储原始（纯文本）密码，而只存储哈希值。因此，请勿尝试直接操作用户的密码属性。这就是创建用户时使用帮助函数的原因。

要更改用户的密码，您有几个选择：

`manage.py changepassword <username>` 提供了一种从命令行更改用户密码的方法。它会提示您更改您必须输入两次的给定用户的密码。如果两者都匹配，则新密码将立即更改。如果您不提供用户，则该命令将尝试更改其用户名与当前系统用户匹配的密码。

您还可以使用 `set_password()` 以编程方式更改密码：

``` python
>>> from django.contrib.auth.models import User
>>> u = User.objects.get(username='john')
>>> u.set_password('new password')
>>> u.save()
```

如果您安装了 Django admin，您还可以在认证系统的管理页面上更改用户的密码。

Django 还提供了可用于允许用户更改自己的密码的视图和表单。

更改用户的密码将注销其所有会话。

### 用户认证

`authenticate(request=None, **credentials)`

使用 `authenticate()` 来验证一组凭据。它将凭据作为关键字参数，默认情况下是 `username` 和 `password`，并针对每个验证后端进行检查，并在凭据对后端有效时返回 `User` 对象。如果凭证对任何后端无效，或者后端引发 `PermissionDenied`，则返回 `None`。例如：

``` python
from django.contrib.auth import authenticate
user = authenticate(username='john', password='secret')
if user is not None:
    # A backend authenticated the credentials
else:
    # No backend authenticated the credentials
```

`request` 是在身份验证后端的 `authenticate()` 方法上传递的可选 `HttpRequest`。

> 这是验证一组凭据的低级别方法;例如，它由 `RemoteUserMiddleware` 使用。除非你正在编写你自己的认证系统，否则你可能不会使用它。


!> 后面内容如果使用 DRF 框架，基本用不到了，所以省略咯～

# django.contrib.auth 模块

## User 模型

### 字段

`username`

必须。150 个字符或更少。用户名可能包含字母数字，`_`，`@`，`+`，。和 `-` 字符。如果您使用的 MySQL，请指定 `max_length=191`，因为默认情况下，MySQL 只能创建 191 个字符的唯一索引。

`first_name`

可选 (`blank=True`)。 30 个字符或更少。

`last_name`

可选 (`blank=True`)。 150 个字符或更少。

`email`

可选 (`blank=True`)。邮箱地址。

`password`

必须。密码的散列和元数据。（Django 不存储原始密码。）原始密码可以是任意长的，并且可以包含任何字符。

`groups`

多对多关系 `Group`

`user_permissions`

多对多关系 `Permission`

`is_staff`

Bollean 类型。指定此用户是否可以访问 admin 站点。

`is_active`

Bollean 类型。指定是否应将此用户帐户视为活动用户。我们建议您将此标志设置为 `False` 而不是删除帐户;这样，如果您的应用程序对用户有任何外键，外键不会中断。

`is_superuser`

Bollean 类型。指定该用户具有所有权限而不明确分配它们。

`last_login`

用户上次登录的日期时间。

`date_joined`

指定帐户何时创建的日期时间。在创建帐户时默认设置为当前日期/时间。

### 属性

`is_authenticated`

只读属性始终为 `True`（与 `AnonymousUser.is_authenticated` 相反，它始终为 `False`）。这是判断用户是否已通过身份验证的一种方式。这并不意味着任何权限，也不会检查用户是否处于活动状态或是否有有效的会话。

`is_anonymous`

只读属性始终为 `False`。这是区分 `User` 和 `AnonymousUser` 对象的一种方式。通常，您应该更喜欢使用 `is_authenticated` 属性。

### 方法

`get_username()`

返回用户的用户名。由于 `User` 模型可以被换出，您应该使用此方法而不是直接引用 `username` 属性。

`get_full_name()`

返回 `first_name` 加上 `last_name`，之间有一个空格。

`get_short_name()`

返回 `first_name`。

`set_password(raw_password)`

将用户的密码设置为给定的原始字符串，注意密码散列。不保存用户对象。

当 `raw_password` 为 `None` 时，密码将被设置为不可用的密码，就像使用 `set_unusable_password()` 一样。

`check_password(raw_password)`

如果给定的原始字符串是用户的正确密码，则返回 `True`。（在进行比较时，会将密码哈希处理。）

`set_unusable_password()`

标记用户没有设置密码。这与为密码输入空白字符串不同。该用户的 `check_password()` 将永远不会返回 `True`。不保存 `User` 对象。

如果您的应用程序的身份验证是针对现有的外部源（例如 LDAP 目录）进行的，您可能需要使用此功能。

`has_usable_password()`

如果为此用户调用了 `set_unusable_password()`，则返回 `False`。

`email_user(subject, message, from_email=None, **kwargs)`

向用户发送电子邮件。

### Manager 方法

`class models.UserManager`

`User` 模型有一个自定义管理器，它具有以下辅助方法（除了由 `BaseUserManager` 提供的方法外）：

`create_user(username, email=None, password=None, **extra_fields)`

创建，保存并返回 `User`。

`username` 和 `password` 设置为给定。 `email` 的域部分将自动转换为小写，并且返回的 `User` 对象将 `is_active` 设置为 `True`。

如果未提供密码，则将调用 `set_unusable_password()`。

`extra_fields` 关键字参数传递给 `User` 的 `__init__` 方法，以允许在自定义用户模型上设置任意字段。

`create_superuser(username, email, password, **extra_fields)`

与 `create_user()` 相同，但将 `is_staff` 和 `is_superuser` 设置为 `True`。

## 实用函数

`get_user(request)`

返回与给定请求的会话关联的用户模型实例。

它检查存储在会话中的认证后端是否存在于 `AUTHENTICATION_BACKENDS` 中。如果存在，它使用后端的 `get_user()` 方法来检索用户模型实例，然后通过调用用户模型的 `get_session_auth_hash()` 方法来验证会话。

如果存储在会话中的认证后端不再位于 `AUTHENTICATION_BACKENDS` 中，如果用户未由后端的 `get_user()` 方法返回，或者如果会话认证哈希未验证，则返回 `AnonymousUser` 的实例。

# 自定义 Django 中的认证机制

## 其他身份认证来源

### 指定身份认证后端

在幕后，Django 维护着一个 “身份认证后端” 列表，用于检查身份认证。当有人调用 `django.contrib.auth.authenticate()` 时，Django 尝试在所有身份验证后端进行身份验证。如果第一个验证方法失败，Django 会尝试第二个验证方法，依此类推，直到尝试完所有后端。

在 `AUTHENTICATION_BACKENDS` setting 中指定要使用的身份验证后端列表。这应该是一个 Python 路径名列表，指向知道如何进行身份验证的 Python 类。这些类可以在你的 Python 路径上的任何地方。

默认情况下，`AUTHENTICATION_BACKENDS` 被设置为：

``` python
['django.contrib.auth.backends.ModelBackend']
```

这是检查 Django 用户数据库并查询内置权限的基本身份验证后端。它不提供通过任何速率限制机制防止暴力攻击的保护。您可以在自定义身份验证后端实现自己的速率限制机制，也可以使用大多数 Web 服务器提供的机制。

`AUTHENTICATION_BACKENDS` 的顺序很重要，所以如果相同的用户名和密码在多个后端有效，Django 将在第一次正确匹配时停止处理。

如果后端引发 `PermissionDenied` 异常，认证将立即失败。 Django 不会检查后面的后端。

### 编写身份认证后端

认证后端是一个实现两个必需方法的类：`get_user(user_id)` 和 `authenticate(request, **credentials)`，以及一组可选的与权限相关的授权方法。

`get_user` 方法需要一个 `user_id` - 可以是一个用户名，数据库 ID 或其他，但必须是用户对象的主键，并返回一个用户对象。

`authenticate` 方法将 `request` 参数和凭据作为关键字参数。大多数情况下，它看起来像这样：

``` python
class MyBackend:
    def authenticate(self, request, username=None, password=None):
        # Check the username/password and return a user.
        ...
```

但它也可以验证 token，如下所示：

``` python
class MyBackend:
    def authenticate(self, request, token=None):
        # Check the token and return a user.
        ...
```

无论使用哪种方式，`authenticate()` 都应该检查它获取的凭据，并在凭证有效时返回与这些凭证相匹配的用户对象。如果它们无效，则应返回 `None`。

`request` 是一个 `HttpRequest`，如果没有提供给 `authenticate()`（将其传递给后端），它可能是 `None`。

Django admin 与 Django `User` 对象紧密耦合。处理这个问题的最好方法是为每个存在于后端的用户创建一个 Django `User` 对象（例如，在您的 LDAP 目录，外部 SQL 数据库等中）。您可以编写脚本来提前执行此操作，或者您的 `authenticate` 方法可以在用户第一次登录时执行。

以下是一个示例后端，它会根据 `settings.py` 文件中定义的用户名和密码变量进行身份验证，并在用户首次进行身份验证时创建一个 Django `User` 对象：

``` python
from django.conf import settings
from django.contrib.auth.hashers import check_password
from django.contrib.auth.models import User

class SettingsBackend:
    """
    Authenticate against the settings ADMIN_LOGIN and ADMIN_PASSWORD.

    Use the login name and a hash of the password. For example:

    ADMIN_LOGIN = 'admin'
    ADMIN_PASSWORD = 'pbkdf2_sha256$30000$Vo0VlMnkR4Bk$qEvtdyZRWTcOsCnI/oQ7fVOu1XAURIZYoOZ3iq8Dr4M='
    """

    def authenticate(self, request, username=None, password=None):
        login_valid = (settings.ADMIN_LOGIN == username)
        pwd_valid = check_password(password, settings.ADMIN_PASSWORD)
        if login_valid and pwd_valid:
            try:
                user = User.objects.get(username=username)
            except User.DoesNotExist:
                # Create a new user. There's no need to set a password
                # because only the password from settings.py is checked.
                user = User(username=username)
                user.is_staff = True
                user.is_superuser = True
                user.save()
            return user
        return None

    def get_user(self, user_id):
        try:
            return User.objects.get(pk=user_id)
        except User.DoesNotExist:
            return None
```

## 扩展现有的 User 模型

有两种方法可以扩展默认 `User` 模型，而不用替换自己的模型。如果您需要的更改是纯粹的行为，并且不需要对存储在数据库中的内容进行任何更改，则可以基于 `User` 创建代理模型。这允许代理模型提供的任何功能，包括默认排序，自定义管理器或自定义模型方法。

如果您希望存储与 `User` 相关的信息，则可以将 `OneToOneField` 用于包含字段的模型以获取更多信息。这种一对一模式通常称为 profile 模型，因为它可能存储有关站点用户的非 auth 相关信息。例如，您可以创建一个 `Employee` 模型：

``` python
from django.contrib.auth.models import User

class Employee(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    department = models.CharField(max_length=100)
```

假设现有员工 Fred Smith 拥有 `User` 和 `Employee` 模型，则可以使用 Django 的标准相关模型约定访问相关信息：

``` python
>>> u = User.objects.get(username='fsmith')
>>> freds_department = u.employee.department
```

要将 profile 模型的字段添加到 admin 的用户页面，请在应用程序的 `admin.py` 中定义 `InlineModelAdmin`（对于本示例，我们将使用 `StackedInline`），并将其添加到 `UserAdmin` 类，该类用 `User` 类注册：

``` python
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin as BaseUserAdmin
from django.contrib.auth.models import User

from my_user_profile_app.models import Employee

# Define an inline admin descriptor for Employee model
# which acts a bit like a singleton
class EmployeeInline(admin.StackedInline):
    model = Employee
    can_delete = False
    verbose_name_plural = 'employee'

# Define a new User admin
class UserAdmin(BaseUserAdmin):
    inlines = (EmployeeInline, )

# Re-register UserAdmin
admin.site.unregister(User)
admin.site.register(User, UserAdmin)
```

这些 profile 模型在任何方面都不是特别的 - 它们只是恰好与用户模型具有一对一链接的 Django 模型。因此，它们不会在创建用户时自动创建，但可以根据需要使用 `django.db.models.signals.post_save` 来创建或更新相关模型。

使用相关模型会产生额外的查询或连接来检索相关数据。根据您的需要，包含相关字段的自定义用户模型可能是您更好的选择，但是，与项目应用程序中默认用户模型的现有关系可能会证明额外的数据库负载。

## 替换自定义 `User` 模型

某些类型的项目可能具有身份验证要求，而 Django 的内置 `User` 模型并不总是适合的。例如，在一些网站上，使用电子邮件地址作为您的身份标记而不是用户名更有意义。

Django 允许您通过为引用自定义模型的 `AUTH_USER_MODEL` setting 提供值来覆盖默认用户模型：

``` python
AUTH_USER_MODEL = 'myapp.MyUser'
```

myapp 表示 Django 应用程序的名称（它必须位于 `INSTALLED_APPS` 中），MyUser 是用作用户模型的 Django 模型的名称。

### 启动项目时使用自定义用户模型

如果您正在开始一个新项目，强烈建议设置一个自定义用户模型，即使默认的用户模型对您来说已经足够。此模型的行为与默认用户模型的行为相同，但如果需要，您可以在将来自定义它：


``` python
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    pass
```

不要忘记指向 `AUTH_USER_MODEL`。在创建任何迁移或首次运行 `manage.py migrate` 之前执行此操作。

另外，请在应用程序的 `admin.py` 中注册模型：

``` python
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin
from .models import User

admin.site.register(User, UserAdmin)
```

### 引用 User 模型

如果直接引用 `User` （例如，通过在外键中引用），那么您的代码将不适用于 `AUTH_USER_MODEL` setting 已更改为不同用户模型的项目。

`get_user_model()`

不要直接引用 `User`，而应该使用 `django.contrib.auth.get_user_model()` 引用 `User` 模型。此方法将返回当前活动的用户模型 - 如果指定了用户模型，则返回自定义用户模型，否则返回 `User`。

在为用户模型定义外键或多对多关系时，应使用 `AUTH_USER_MODEL` setting 指定自定义模型。例如：

``` python
from django.conf import settings
from django.db import models

class Article(models.Model):
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
    )
```

连接到用户模型发送的信号时，应使用 `AUTH_USER_MODEL` setting 指定自定义模型。例如：

``` python
from django.conf import settings
from django.db.models.signals import post_save

def post_save_receiver(sender, instance, created, **kwargs):
    pass

post_save.connect(post_save_receiver, sender=settings.AUTH_USER_MODEL)
```

一般来说，用导入时执行的代码中的 `AUTH_USER_MODEL` setting 来引用用户模型是最容易的，但是，也可以在 Django 导入模型时调用 `get_user_model()`，以便使用 `models.ForeignKey(get_user_model(), ...)`。

如果您的应用使用多个用户模型进行测试，例如使用 `@override_settings(AUTH_USER_MODEL=...)`，并且将 `get_user_model()` 的结果缓存在模块级变量中，则可能需要侦听 `setting_changed` 信号以清除缓存。例如：

``` python
from django.apps import apps
from django.contrib.auth import get_user_model
from django.core.signals import setting_changed
from django.dispatch import receiver

@receiver(setting_changed)
def user_model_swapped(**kwargs):
    if kwargs['setting'] == 'AUTH_USER_MODEL':
        apps.clear_cache()
        from myapp import some_module
        some_module.UserModel = get_user_model()
```

