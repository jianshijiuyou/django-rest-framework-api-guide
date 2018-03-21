> [官方原文链接](http://www.django-rest-framework.org/api-guide/testing/)  


# 测试


REST framework 包含一些扩展 Django 现有测试框架的助手类，并改进对 API 请求的支持。

# APIRequestFactory

扩展了 Django 现有的 `RequestFactory` 类。

## 创建测试请求

`APIRequestFactory` 类支持与 Django 的标准 `RequestFactory` 类几乎完全相同的 API。这意味着标准的 `.get()`, `.post()`, `.put()`, `.patch()`, `.delete()`, `.head()` 和 `.options()` 方法都可用。

``` python
from rest_framework.test import APIRequestFactory

# Using the standard RequestFactory API to create a form POST request
factory = APIRequestFactory()
request = factory.post('/notes/', {'title': 'new idea'})
```

#### 使用 `format` 参数

创建请求主体（如 `post`，`put` 和 `patch`）的方法包括 `format` 参数，这使得使用除 multipart 表单数据以外的内容类型生成请求变得容易。例如：

``` python
# Create a JSON POST request
factory = APIRequestFactory()
request = factory.post('/notes/', {'title': 'new idea'}, format='json')
```

默认情况下，可用的格式是 `'multipart'` 和 `'json'` 。为了与 Django 现有的 `RequestFactory` 兼容，默认格式是 `'multipart'`。


#### 显式编码请求主体

如果你需要显式编码请求正文，则可以通过设置 `content_type` 标志来完成。例如：

``` python
request = factory.post('/notes/', json.dumps({'title': 'new idea'}), content_type='application/json')
```

#### PUT 和 PATCH 与表单数据

Django 的 `RequestFactory` 和 REST framework 的 `APIRequestFactory` 之间值得注意的一个区别是 multipart 表单数据将被编码为除 `.post()` 以外的方法。

例如，使用 `APIRequestFactory`，你可以像这样做一个表单 PUT 请求：

``` python
factory = APIRequestFactory()
request = factory.put('/notes/547/', {'title': 'remember to email dave'})
```

使用 Django 的 `RequestFactory`，你需要自己显式编码数据：

``` python
from django.test.client import encode_multipart, RequestFactory

factory = RequestFactory()
data = {'title': 'remember to email dave'}
content = encode_multipart('BoUnDaRyStRiNg', data)
content_type = 'multipart/form-data; boundary=BoUnDaRyStRiNg'
request = factory.put('/notes/547/', content, content_type=content_type)
```

## 强制认证

当使用请求工厂直接测试视图时，能够直接验证请求通常很方便，而不必构造正确的验证凭证。

要强制验证请求，请使用 `force_authenticate()` 方法。

``` python
from rest_framework.test import force_authenticate

factory = APIRequestFactory()
user = User.objects.get(username='olivia')
view = AccountDetail.as_view()

# Make an authenticated request to the view...
request = factory.get('/accounts/django-superstars/')
force_authenticate(request, user=user)
response = view(request)
```

该方法的签名是 `force_authenticate(request, user=None, token=None)`。调用时，可以设置 user 和 token 中的任一个或两个。

例如，当使用令牌强行进行身份验证时，你可能会执行以下操作：

``` python
user = User.objects.get(username='olivia')
request = factory.get('/accounts/django-superstars/')
force_authenticate(request, user=user, token=user.auth_token)
```

---

**注意**: `force_authenticate` 直接将 `request.user` 设置为内存中的 `user` 实例。如果跨多个测试重新使用同一个 `user` 实例来更新已保存的 `user` 状态，则可能需要在测试之间调用 `refresh_from_db()`。

---

**注意**: 使用 `APIRequestFactory` 时，返回的对象是 Django 的标准 `HttpRequest`，而不是 REST framework 的 `Request` 对象，只有在调用视图后才会生成该对象。

这意味着直接在请求对象上设置属性可能并不总是有你期望的效果。例如，直接设置 `.token` 将不起作用，并且仅在使用会话身份验证时直接设置 `.user` 才会起作用。

``` python
# Request will only authenticate if `SessionAuthentication` is in use.
request = factory.get('/accounts/django-superstars/')
request.user = user
response = view(request)
```

---

## 强制 CSRF 验证

默认情况下，使用 `APIRequestFactory` 创建的请求在传递给 REST framework 视图时不会应用 CSRF 验证。如果你需要明确打开 CSRF 验证，则可以通过在实例化工厂时设置 `enforce_csrf_checks` 标志来实现。

``` python
factory = APIRequestFactory(enforce_csrf_checks=True)
```

---

**注意**: 值得注意的是，Django 的标准 `RequestFactory` 不需要包含这个选项，因为当使用常规的 Django 时，CSRF 验证发生在中间件中，当直接测试视图时该中间件不运行。在使用 REST framework 时，CSRF 验证发生在视图内，因此请求工厂需要禁用视图级 CSRF 检查。

---

# APIClient

扩展了 Django 现有的 `Client` 类。

## 发出请求

`APIClient` 类支持与 Django 标准 `Client` 类相同的请求接口。这意味着标准的 `.get()`, `.post()`, `.put()`, `.patch()`, `.delete()`, `.head()` 和 `.options()` 方法都可用。例如：

``` python
from rest_framework.test import APIClient

client = APIClient()
client.post('/notes/', {'title': 'new idea'}, format='json')
```


## 认证

#### .login(**kwargs)

`login` 方法的功能与 Django 的常规 `Client` 类一样。这使你可以对任何包含 `SessionAuthentication` 的视图进行身份验证。

``` python
# Make all requests in the context of a logged in session.
client = APIClient()
client.login(username='lauren', password='secret')
```

要登出，请照常调用 `logout` 方法。

``` python
# Log out
client.logout()
```

`login` 方法适用于测试使用会话认证的 API，例如包含 AJAX 与 API 交互的网站。

#### .credentials(**kwargs)

`credentials` 方法可用于设置 header，这些 header 将包含在测试客户端的所有后续请求中。

``` python
from rest_framework.authtoken.models import Token
from rest_framework.test import APIClient

# Include an appropriate `Authorization:` header on all requests.
token = Token.objects.get(user__username='lauren')
client = APIClient()
client.credentials(HTTP_AUTHORIZATION='Token ' + token.key)
```

请注意，第二次调用 `credentials` 会覆盖任何现有凭证。你可以通过调用没有参数的方法来取消任何现有的凭证。

``` python
# Stop including any credentials
client.credentials()
```

`credentials` 方法适用于测试需要验证 header 的 API，例如 basic 验证，OAuth1 和 OAuth2 验证以及简单令牌验证方案。

#### .force_authenticate(user=None, token=None)

有时你可能想完全绕过认证，强制测试客户端的所有请求被自动视为已认证。

如果你正在测试 API 但是不想构建有效的身份验证凭据以进行测试请求，则这可能是一个有用的捷径。

``` python
user = User.objects.get(username='lauren')
client = APIClient()
client.force_authenticate(user=user)
```

要对后续请求进行身份验证，请调用 `force_authenticate` 将 user 和(或) token 设置为 `None`。

``` python
client.force_authenticate(user=None)
```

## CSRF 验证

默认情况下，使用 `APIClient` 时不应用 CSRF 验证。如果你需要明确启用 CSRF 验证，则可以通过在实例化客户端时设置 `enforce_csrf_checks` 标志来实现。

``` python
client = APIClient(enforce_csrf_checks=True)
```

像往常一样，CSRF 验证将仅适用于任何会话验证视图。这意味着 CSRF 验证只有在客户端通过调用 `login()` 登录后才会发生。

---

# RequestsClient

REST framework 还包含一个客户端，用于使用流行的 Python 库 `requests` 与应用程序进行交互。 这可能是有用的，如果：

* 你期望主要从另一个 Python 服务与 API 进行交互，并且希望在与客户端相同的级别测试该服务。
* 你希望以这样的方式编写测试，以便它们也可以在分段或实时环境中运行。 

它暴露了与直接使用请求会话完全相同的接口。

``` python
client = RequestsClient()
response = client.get('http://testserver/users/')
assert response.status_code == 200
```

请注意，requests client 要求你传递完全限定的 URL。

## `RequestsClient` 与数据库一起工作

如果你想编写仅与服务接口交互的测试，则 `RequestsClient` 类很有用。这比使用标准的 Django 测试客户端要严格一些，因为这意味着所有的交互必须通过 API。

如果你使用的是 `RequestsClient`，你需要确保测试设置和结果断言以常规 API 调用的方式执行，而不是直接与数据库模型进行交互。例如，不是检查 `Customer.objects.count() == 3`，而是列出 customers 端点，并确保它包含三条记录。

## Headers & 身份验证

自定义 header 和身份验证凭证的提供方式与使用标准 `requests.Session` 实例时的方式相同。

``` python
from requests.auth import HTTPBasicAuth

client.auth = HTTPBasicAuth('user', 'pass')
client.headers.update({'x-test': 'true'})
```

## CSRF

如果你使用 `SessionAuthentication` ，那么你需要为 `POST`, `PUT`, `PATCH` 或 `DELETE` 请求包含一个 CSRF 令牌。

你可以通过遵循基于 JavaScript 的客户端使用的相同流程来实现。首先进行 `GET` 请求以获取 CRSF 令牌，然后在以下请求中呈现该令牌。

例如...

``` python
client = RequestsClient()

# Obtain a CSRF token.
response = client.get('/homepage/')
assert response.status_code == 200
csrftoken = response.cookies['csrftoken']

# Interact with the API.
response = client.post('/organisations/', json={
    'name': 'MegaCorp',
    'status': 'active'
}, headers={'X-CSRFToken': csrftoken})
assert response.status_code == 200
```

## Live tests

使用 `RequestsClient` 和 `CoreAPIClient` 可以编写在开发环境中运行的测试用例，也可以直接根据测试服务器或生产环境运行测试用例。

使用这种风格来创建几个核心功能的基本测试是验证你的实时服务的有效方法。这样做可能需要仔细注意安装和卸载（setup and teardown），以确保测试的运行方式不会直接影响客户数据。

---

# CoreAPIClient

CoreAPIClient 允许你使用 Python `coreapi` 客户端库与你的 API 进行交互。

``` python
# Fetch the API schema
client = CoreAPIClient()
schema = client.get('http://testserver/schema/')

# Create a new organisation
params = {'name': 'MegaCorp', 'status': 'active'}
client.action(schema, ['organisations', 'create'], params)

# Ensure that the organisation exists in the listing
data = client.action(schema, ['organisations', 'list'])
assert(len(data) == 1)
assert(data == [{'name': 'MegaCorp', 'status': 'active'}])
```

## Headers & 身份验证

自定义 header 和身份验证可以与 `RequestsClient` 类似的方式和 `CoreAPIClient` 一起使用。

``` python
from requests.auth import HTTPBasicAuth

client = CoreAPIClient()
client.session.auth = HTTPBasicAuth('user', 'pass')
client.session.headers.update({'x-test': 'true'})
```

---

# API Test cases

REST framework 包含以下测试用例类，它们类似现有的 Django 测试用例类，但使用 `API​​Client` 而不是 Django 的默认 `Client`。

* `APISimpleTestCase`
* `APITransactionTestCase`
* `APITestCase`
* `APILiveServerTestCase`

## 举个栗子

你可以像使用常规 Django 测试用例类一样使用任何 REST framework 的测试用例类。 `self.client` 属性将是一个 `APIClient` 实例。

``` python
from django.urls import reverse
from rest_framework import status
from rest_framework.test import APITestCase
from myproject.apps.core.models import Account

class AccountTests(APITestCase):
    def test_create_account(self):
        """
        Ensure we can create a new account object.
        """
        url = reverse('account-list')
        data = {'name': 'DabApps'}
        response = self.client.post(url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(Account.objects.count(), 1)
        self.assertEqual(Account.objects.get().name, 'DabApps')
```

---

# URLPatternsTestCase

REST framework 还提供了一个用于隔离每个类的 `urlpatterns` 的测试用例类。请注意，它继承自 Django 的 `SimpleTestCase`，并且很可能需要与另一个测试用例类混合使用。

## 例如

``` python
from django.urls import include, path, reverse
from rest_framework.test import APITestCase, URLPatternsTestCase


class AccountTests(APITestCase, URLPatternsTestCase):
    urlpatterns = [
        path('api/', include('api.urls')),
    ]

    def test_create_account(self):
        """
        Ensure we can create a new account object.
        """
        url = reverse('account-list')
        response = self.client.get(url, format='json')
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data), 1)
```

---

# 测试响应

## 检查响应数据

在检查测试响应的有效性时，检查响应的创建数据通常比较方便，而不是检查完全渲染的响应。

例如，检查 `response.data` 更容易：

``` python
response = self.client.get('/users/4/')
self.assertEqual(response.data, {'id': 4, 'username': 'lauren'})
```

而不是检查解析 `response.content` 的结果：

``` python
response = self.client.get('/users/4/')
self.assertEqual(json.loads(response.content), {'id': 4, 'username': 'lauren'})
```

## 渲染响应

如果你使用 `API​​RequestFactory` 直接测试视图，则返回的响应将不会渲染，因为模板响应的渲染由 Django 的内部请求 - 响应循环执行。为了访问 `response.content`，你首先需要渲染响应。

``` python
view = UserDetail.as_view()
request = factory.get('/users/4')
response = view(request, pk='4')
response.render()  # Cannot access `response.content` without this.
self.assertEqual(response.content, '{"username": "lauren", "id": 4}')
```

---

# 配置

## 设置默认格式

用于创建测试请求的默认格式可以使用 `TEST_REQUEST_DEFAULT_FORMAT` setting key 进行设置。例如，默认情况下总是对测试请求使用 JSON 而不是标准的 multipart 表单请求，请在 `settings.py` 文件中设置以下内容：

``` python
REST_FRAMEWORK = {
    ...
    'TEST_REQUEST_DEFAULT_FORMAT': 'json'
}
```

## 设置可用的格式

如果你需要使用除 multipart 或 json 请求之外的其他方法来测试请求，则可以通过设置 `TEST_REQUEST_RENDERER_CLASSES` setting 来完成。

例如，要在测试请求中添加对 `format ='html'` 的支持，您可能在 `settings.py` 文件中有这样的内容。

``` python
REST_FRAMEWORK = {
    ...
    'TEST_REQUEST_RENDERER_CLASSES': (
        'rest_framework.renderers.MultiPartRenderer',
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.TemplateHTMLRenderer'
    )
}
```