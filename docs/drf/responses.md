> [官方原文链接](http://www.django-rest-framework.org/api-guide/responses/)

## Responses

> 与基本的 HttpResponse 对象不同，TemplateResponse 对象保留了视图提供的用于计算响应的上下文的详细信息。直到需要时才会计算最终的响应输出，也就是在后面的响应过程中进行计算。
> — *Django 文档*

REST framework 通过提供一个 `Response` 类来支持 HTTP 内容协商，该类允许你根据客户端请求返回不同的表现形式（如： JSON ，HTML 等）。

`Response` 是 Django 的 `SimpleTemplateResponse` 的子类。`Response` 对象使用数据进行初始化，数据应由 Python 对象（native Python primitives）组成。然后 REST framework 使用标准的 HTTP 内容协商来确定它应该如何渲染最终响应的内容。


当然，您也可以不使用 `Response` 类，直接返回常规 `HttpResponse` 或 `StreamingHttpResponse` 对象。 使用 `Response` 类只是提供了一个更好的交互方式，它可以返回多种格式。


除非由于某种原因需要大幅度定制 REST framework ，否则应该始终对返回 `Response` 对象的视图使用 `APIView` 类或 `@api_view` 装饰器。这样做可以确保视图执行内容协商，并在视图返回之前为响应选择适当的渲染器。


## 创建 response

### Response()

与普通 `HttpResponse` 对象不同，您不会使用渲染的内容实例化 `Response` 对象。相反，您传递的是未渲染的数据，可能包含任何 Python 对象。

由于 `Response` 类使用的渲染器不能处理复杂的数据类型（比如 Django 的模型实例），所以需要在创建 `Response` 对象之前将数据序列化为基本的数据类型。

你可以使用 REST framework 的 `Serializer ` 类来执行序列化的操作，也可以用自己的方式来序列化。


**构造方法**: `Response(data, status=None, template_name=None, headers=None, content_type=None)`  


参数：
 * `data`： 响应的序列化数据。
 * `status`： 响应的状态代码。默认为200。
 * `template_name`： 选择 `HTMLRenderer` 时使用的模板名称。
 * `headers`： 设置 HTTP header，字典类型。
 * `content_type`： 响应的内容类型，通常渲染器会根据内容协商的结果自动设置，但有些时候需要手动指定。


## 属性

### .data

还没有渲染，但已经序列化的响应数据。

### .status_code

状态码

### .content

将会返回的响应内容，必须先调用 `.render()` 方法，才能访问 `.content` 。

### .template_name

只有在 response 的渲染器是 `HTMLRenderer` 或其他自定义模板渲染器时才需要提供。

### .accepted_renderer

用于将会返回的响应内容的渲染器实例。

从视图返回响应之前由 `APIView` 或 `@api_view` 自动设置。

### .accepted_media_type

内容协商阶段选择的媒体类型。

从视图返回响应之前由 `APIView` 或 `@api_view` 自动设置。

### .renderer_context

将传递给渲染器的 `.render()` 方法的附加的上下文信息字典。

从视图返回响应之前由 `APIView` 或 `@api_view` 自动设置。

## 标准 HttpResponse 属性

`Response` 类扩展于 `SimpleTemplateResponse`，并且响应中也提供了所有常用的属性和方法。例如，您可以用标准方式在响应中设置 header：
``` python
response = Response()
response['Cache-Control'] = 'no-cache'
```

### .render()

与其他任何 `TemplateResponse` 一样，调用此方法将响应的序列化数据呈现为最终响应内容。响应内容将设置为在 `accepted_renderer` 实例上调用 `.render(data，accepted_media_type，renderer_context)` 方法的结果。

通常不需要自己调用 `.render()` ，因为它是由 Django 处理的。

