> [官方原文链接](http://www.django-rest-framework.org/api-guide/requests/)



> 如果你正在开发基于 REST 的 web API 服务...... 应该忽略 request.POST。 
> — *Malcom Tredinnick，Django 开发组*

REST framework 的 `Request` 类扩展与标准的 `HttpRequest`，并做了相应的增强，比如更加灵活的请求解析（request parsing）和认证（request authentication）。

## Request 解析

REST framwork 的 `Request` 对象提供了灵活的请求解析，允许你使用 JSON data 或 其他 media types 像通常处理表单数据一样处理请求。

### .data

`request.data` 返回请求主题的解析内容。这跟标准的 `request.POST` 和 `request.FILES` 类似，并且还具有以下特点：
*  包括所有解析的内容，文件（file） 和 非文件（non-file inputs）。
*  支持解析 `POST` 以外的 HTTP method ， 比如 `PUT`， `PATCH`。
*  更加灵活，不仅仅支持表单数据，传入同样的 JSON 数据一样可以正确解析，并且不用做额外的处理（意思是前端不管提交的是表单数据，还是 JSON 数据，`.data` 都能够正确解析）。

*.data 具体操作，以后再说～*

### .query_params

`request.query_params` 等同于 `request.GET`，不过其名字更加容易理解。

为了代码更加清晰可读，推荐使用 `request.query_params` ，而不是 Django 中的 `request.GET`，这样那够让你的代码更加明显的体现出 ----- 任何 HTTP method 类型都可能包含查询参数（query parameters），而不仅仅只是 'GET' 请求。

### .parsers

`APIView` 类或者 `@api_view` 装饰器将根据视图上设置的 `parser_classes` 或 `settings` 文件中的 `DEFAULT_PARSER_CLASSES` 设置来确保此属性（`.parsers`）自动设置为 `Parser` 实例列表。

**通常不需要关注该属性......**

如果你非要看看它里面是什么，可以打印出来看看，大概长这样：
``` bash
[<rest_framework.parsers.JSONParser object at 0x7fa850202d68>, <rest_framework.parsers.FormParser object at 0x7fa850202be0>, <rest_framework.parsers.MultiPartParser object at 0x7fa850202860>]
```
恩，包含三个解析器 `JSONParser`，`FormParser`，`MultiPartParser`。

> 注意： 如果客户端发送格式错误的内容，则访问 `request.data` 可能会引发 `ParseError` 。默认情况下， REST framework 的 `APIView` 类或者 `@api_view` 装饰器将捕获错误并返回 `400 Bad Request` 响应。
> 如果客户端发送的请求内容无法解析（不同于格式错误），则会引发 `UnsupportedMediaType` 异常，默认情况下会被捕获并返回 `415 Unsupported Media Type` 响应。


## 内容协商

该请求公开了一些属性，允许你确定内容协商阶段的结果。这使你可以实施一些行为，例如为不同媒体类型选择不同的序列化方案。

### .accepted_renderer

渲染器实例是由内容协商阶段选择的。

### .accepted_media_type

表示内容协商阶段接受的 media type 的字符串。

## 认证（Authentication）

REST framework 提供了灵活的认证方式：
* 可以在 API 的不同部分使用不同的认证策略。
* 支持同时使用多个身份验证策略。
* 提供与传入请求关联的用户（user）和令牌（token）信息。

### .user

`request.user` 通常会返回 `django.contrib.auth.models.User` 的一个实例，但其行为取决于正在使用的身份验证策略。

如果请求未经身份验证，则 `request.user` 的默认值是 `django.contrib.auth.models.AnonymousUser` 的实例（就是匿名用户）。

*关于 `.user` 的更多内容，以后再说～*

### .auth

`request.auth` 返回任何附加的认证上下文（authentication context）。`request.auth` 的确切行为取决于正在使用的身份验证策略，但它通常可能是请求经过身份验证的令牌（token）实例。

如果请求未经身份验证，或者没有附加上下文（context），则 `request.auth` 的默认值为 `None`。

*关于 `.auth` 的更多内容，以后再说～*

### .authenticators

`APIView` 类或 `@api_view` 装饰器将确保根据视图上设置的 `authentication_classes` 或基于 `settings` 文件中的 `DEFAULT_AUTHENTICATORS` 设置将此属性（`.authenticators`）自动设置为 `Authentication` 实例列表。

**通常不需要关注该属性...... **

> 注意：调用 `.user` 或 `.auth` 属性时可能会引发 `WrappedAttributeError` 异常。这些错误源于 authenticator  作为一个标准的 `AttributeError` ，为了防止它们被外部属性访问修改，有必要重新提升为不同的异常类型。Python 无法识别来自 authenticator  的 `AttributeError`，并会立即假定请求对象没有 `.user` 或 `.auth` 属性。authenticator 需要修复。

多说几句

`.authenticators` 其实存的就是当前使用的认证器（authenticator）列表，打印出来大概是这样：

``` bash
[<rest_framework.authentication.SessionAuthentication object at 0x7f8ae4528710>, <rest_framework.authentication.BasicAuthentication object at 0x7f8ae45286d8>]
```
可以看到这里使用的认证器（authenticator）包括 `SessionAuthentication` 和 `BasicAuthentication`。

## 浏览器增强

REST framework 支持基于浏览器的 `PUT`，`PATCH`，`DELETE` 表单。

### .method

`request.method` 返回请求 HTTP 方法的大写字符串表示形式。如 `GET`,`POST`...。

透明地支持基于浏览器的 `PUT`，`PATCH` 和 `DELETE` 表单。

*更多相关信息以后再说～*

### .content_type

`request.content_type` 返回表示 HTTP 请求正文的媒体类型（media type）的字符串对象（比如： `text/plain` , `text/html` 等），如果没有提供媒体类型，则返回空字符串。

通常不需要直接访问此属性，一般都依赖与 REST 框架的默认请求解析行为。

不建议使用 `request.META.get('HTTP_CONTENT_TYPE')` 来获取 content type 。

*更多相关信息以后再说～*

### .stream

`request.stream` 返回一个代表请求主体内容的流。

通常不需要直接访问此属性，一般都依赖与 REST 框架的默认请求解析行为。

## 标准的 HttpRequest 属性

由于 REST framework 的 `Request ` 扩展于 Django 的 `HttpRequest`，所有其他标准属性和方法也可用。例如`request.META` 和 `request.session` 字典都可以正常使用。

请注意，由于实现原因，`Request` 类不会从 `HttpRequest` 类继承，而是使用组合扩展类（优先使用组合，而非继承，恩，老铁没毛病 0.0）
