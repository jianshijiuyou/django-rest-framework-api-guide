> [官方原文链接](http://www.django-rest-framework.org/api-guide/pagination/)  



# 分页

REST framework 包含对可定制分页样式的支持。这使你可以将较大的结果集分成单独的数据页面。

分页 API 支持：

* 以分页链接的形式作为响应内容的一部分。
* 以分页链接的形式包含在响应的 header 中，如 `Content-Range` 或 `Link`.

内置的样式目前是以分页链接的形式作为响应内容的一部分。使用可浏览的 API 时，此样式更易于访问。

分页仅在你使用通用视图或视图集时自动执行。如果你使用的是常规 `APIView`，则需要自己调用分页 API 以确保返回分页响应。示例请参阅 `mixins.ListModelMixin` 和 `generics.GenericAPIView` 类的源代码。

可以通过将分页类设置为 `None`，关闭分页。

## 设置分页样式

分页样式可以使用 `DEFAULT_PAGINATION_CLASS` 和 `PAGE_SIZE` setting key 全局设置。例如，要使用内置的 limit/offset 分页，你可以这样做：

``` python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.LimitOffsetPagination',
    'PAGE_SIZE': 100
}
```

请注意，你需要设置分页类和应使用的页面大小。默认情况下，`DEFAULT_PAGINATION_CLASS` 和 `PAGE_SIZE` 都是 `None`。

你还可以使用 `pagination_class` 属性在单个视图上设置分页类。通常，你希望在整个 API 中使用相同的分页样式，但你可能希望在每个视图的基础上更改分页的各个方面，例如默认或最大页面大小。

## 修改分页样式

如果要修改分页样式的特定方面，则需要继承其中一个分页类，并设置要更改的属性。

``` python
class LargeResultsSetPagination(PageNumberPagination):
    page_size = 1000
    page_size_query_param = 'page_size'
    max_page_size = 10000

class StandardResultsSetPagination(PageNumberPagination):
    page_size = 100
    page_size_query_param = 'page_size'
    max_page_size = 1000
```

然后，你可以使用 `.pagination_class` 属性将新样式应用于视图：

``` python
class BillingRecordsView(generics.ListAPIView):
    queryset = Billing.objects.all()
    serializer_class = BillingRecordsSerializer
    pagination_class = LargeResultsSetPagination
```

或者使用 `DEFAULT_PAGINATION_CLASS` setting key 全局应用样式。例如：

``` python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'apps.core.pagination.StandardResultsSetPagination'
}
```

---

# API 参考

## PageNumberPagination

此分页样式在请求查询参数中接受一个页码值。

**Request**:

```
GET https://api.example.org/accounts/?page=4
```

**Response**:

``` python
HTTP 200 OK
{
    "count": 1023
    "next": "https://api.example.org/accounts/?page=5",
    "previous": "https://api.example.org/accounts/?page=3",
    "results": [
       …
    ]
}
```

#### Setup

要全局启用 `PageNumberPagination` 样式，请使用以下配置，并根据需要设置 `PAGE_SIZE`：

``` python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 100
}
```

如果使用的是 `GenericAPIView` 的子类，还可以设置 `pagination_class` 属性以在每个视图的基础上选择 `PageNumberPagination`。

#### Configuration

`PageNumberPagination` 类包含一些可以被覆盖以修改分页样式的属性。

要设置这些属性，你应该继承 `PageNumberPagination` 类，然后像上面那样启用你的自定义分页类。

* `django_paginator_class` - 要使用的 Django Paginator 类。默认是 `django.core.paginator.Paginator`，对于大多数用例来说应该没问题。
* `page_size` - 指定页面大小的数字值。如果设置，则会覆盖 `PAGE_SIZE` setting。默认值与 `PAGE_SIZE` setting key 相同。
* `page_query_param` - 一个字符串值，指定用于分页控件的查询参数的名称。
* `page_size_query_param` - 一个字符串值，指定查询参数的名称，允许客户端根据每个请求设置页面大小。默认为 `None`，表示客户端可能无法控制所请求的页面大小。
* `max_page_size` - 一个数字值，表示允许的最大页面大小。该属性仅在 `page_size_query_param` 也被设置时有效。
* `last_page_strings` - 字符串列表或元组，用于指定可能与 `page_query_param` 一起使用的值，用以请求集合中的最终页面。默认为 `('last',)`
* `template` - 在可浏览 API 中渲染分页控件时使用的模板的名称。可能会被覆盖以修改渲染样式，或设置为 `None` 以完全禁用 HTML 分页控件。默认为 `"rest_framework/pagination/numbers.html"`。

---

## LimitOffsetPagination

这种分页样式反映了查找多个数据库记录时使用的语法。客户端包含 “limit” 和 “offset” 查询参数。limit 表示要返回的 item 的最大数量，并且等同于其他样式中的 `page_size`。offset 指定查询的起始位置与完整的未分类 item 集的关系。

**Request**:

``` python
GET https://api.example.org/accounts/?limit=100&offset=400
```

**Response**:

``` python
HTTP 200 OK
{
    "count": 1023
    "next": "https://api.example.org/accounts/?limit=100&offset=500",
    "previous": "https://api.example.org/accounts/?limit=100&offset=300",
    "results": [
       …
    ]
}
```

#### Setup

要全局启用 `LimitOffsetPagination` 样式，请使用以下配置：

``` python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.LimitOffsetPagination'
}
```

或者，你也可以设置一个 `PAGE_SIZE` 键。如果使用了 `PAGE_SIZE` 参数，则 `limit` 查询参数将是可选的，并且可能会被客户端忽略。

如果使用的是 `GenericAPIView` 子类，还可以设置 `pagination_class` 属性以基于每个视图选择 `LimitOffsetPagination`。

#### Configuration

`LimitOffsetPagination` 类包含一些可以被覆盖以修改分页样式的属性。

要设置这些属性，应该继承 `LimitOffsetPagination` 类，然后像上面那样启用你的自定义分页类。

* `default_limit` - 一个数字值，指定客户端在查询参数中未提供的 limit 。默认值与 `PAGE_SIZE` setting key 相同。
* `limit_query_param` - 一个字符串值，指示 “limit” 查询参数的名称。默认为 `'limit'`。
* `offset_query_param` - 一个字符串值，指示 “offset” 查询参数的名称。默认为 `'offset'`。
* `max_limit` - 一个数字值，表示客户端可以要求的最大允许 limit。默认为 `None`。
* `template` - 在可浏览 API 中渲染分页控件时使用的模板的名称。可能会被覆盖以修改渲染样式，或设置为 `None` 以完全禁用 HTML 分页控件。默认为 `"rest_framework/pagination/numbers.html"`。

---

## CursorPagination

基于游标的分页提供了一个不透明的 “游标” 指示器，客户端可以使用该指示器来翻阅结果集。此分页样式仅提供前向和反向控件，并且不允许客户端导航到任意位置。

基于游标的分页需要在结果集中存在唯一的，不变的 item 顺序。这种排序通常可以是记录上的创建时间戳，因为这确保了排序的一致性。

基于游标的分页比其他方案更复杂。它还要求结果集渲染固定顺序，并且不允许客户端任意索引结果集。但它确实提供了以下好处：

* 提供一致的分页视图。正确使用时 `CursorPagination` 确保客户端在分页时不会看到同一个 item，即使在分页过程中其他客户端正在插入新 item。
* 支持使用非常大的数据集。使用极大数据集分页时，使用基于偏移量的分页样式可能会变得效率低下或无法使用。基于游标的分页方案具有固定时间属性，并且不会随着数据集大小的增加而减慢。

#### 细节和限制

正确使用基于游标的分页需要稍微注意细节。你需要考虑希望将该方案应用于何种顺序。默认是按 `"-created"` 排序。这假设在模型实例上必须有一个 “created” 时间戳字段，并且会渲染一个 “时间轴” 样式分页视图，其中最近添加的 item 是第一个。

你可以通过重写分页类上的 `'ordering'` 属性或者将 `OrderingFilter` 过滤器类与 `CursorPagination` 一起使用来修改排序。与 `OrderingFilter` 一起使用时，你应该考虑限制用户可以排序的字段。

正确使用游标分页应该有一个满足以下条件的排序字段：

* 在创建时应该是一个不变的值，例如时间戳，slug，或其他只设置一次的字段。
* 应该是独特的，或几乎独一无二的。毫秒精度时间戳就是一个很好的例子。这种游标分页的实现使用了一种智能的 “位置 + 偏移” 风格，允许它正确地支持非严格唯一的值作为排序。
* 应该是可以强制为字符串的非空值。
* 不应该是一个 float。精度错误很容易导致错误的结果。提示：改用小数。（如果你已经有一个 float 字段并且必须对其进行分页，则可以在此处找到[使用小数来限定精度的示例](https://gist.github.com/keturn/8bc88525a183fd41c73ffb729b8865be#file-fpcursorpagination-py)。）
* 该字段应该有一个数据库索引。

使用不满足这些约束条件的排序字段通常仍然有效，但是你将失去游标分页的一些好处。

#### Setup

要全局启用 `CursorPagination` 样式，请使用以下配置，根据需要修改 `PAGE_SIZE`：

``` python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.CursorPagination',
    'PAGE_SIZE': 100
}
```

如果使用的是 `GenericAPIView` 子类，还可以设置 `pagination_class` 属性以基于每个视图选择 `CursorPagination`。

#### Configuration

`CursorPagination` 类包含一些可以被覆盖以修改分页样式的属性。

要设置这些属性，你应该继承 `CursorPagination` 类，然后像上面那样启用你的自定义分页类。

* `page_size` = 指定页面大小的数字值。如果设置，则会覆盖 `PAGE_SIZE` 设置。默认值与 `PAGE_SIZE` setting key 相同。
* `cursor_query_param` = 一个字符串值，指定 “游标” 查询参数的名称。默认为 `'cursor'`.
* `ordering` = 这应该是一个字符串或字符串列表，指定将应用基于游标的分页的字段。例如： `ordering = 'slug'`。默认为 `-created`。该值也可以通过在视图上使用 `OrderingFilter` 来覆盖。
* `template` = 在可浏览 API 中渲染分页控件时使用的模板的名称。可能会被覆盖以修改渲染样式，或设置为 `None` 以完全禁用 HTML 分页控件。默认为 `"rest_framework/pagination/previous_and_next.html"`。

---

# 自定义分页样式

要创建自定义分页序列化类，你应该继承 `pagination.BasePagination` 并覆盖 `paginate_queryset(self, queryset, request, view=None)` 和 `get_paginated_response(self, data)` 方法：

* `paginate_queryset` 方法被传递给初始查询集，并且应该返回一个只包含请求页面中的数据的可迭代对象。
* `get_paginated_response` 方法传递序列化的页面数据，并返回一个 `Response` 实例。

请注意，`paginate_queryset` 方法可以在分页实例上设置状态，而后 `get_paginated_response` 方法可以使用它。

## 举个栗子

假设我们想用一个修改后的格式替换默认的分页输出样式，该样式包含嵌套的 “links” key（包含上一页，下一页链接）。我们可以像这样指定一个自定义分页类：

``` python
class CustomPagination(pagination.PageNumberPagination):
    def get_paginated_response(self, data):
        return Response({
            'links': {
                'next': self.get_next_link(),
                'previous': self.get_previous_link()
            },
            'count': self.page.paginator.count,
            'results': data
        })
```

然后我们需要在配置中设置自定义类：

``` python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'my_project.apps.core.pagination.CustomPagination',
    'PAGE_SIZE': 100
}
```

请注意，如果你关心如何在可浏览的 API 中显示键的顺序，则可以在构建分页响应的主体时选择使用 `OrderedDict`，这是可选的。

## 使用你的自定义分页类

要默认使用你的自定义分页类，请使用 `DEFAULT_PAGINATION_CLASS` setting：

``` python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'my_project.apps.core.pagination.LinkHeaderPagination',
    'PAGE_SIZE': 100
}
```
列表端点的 API 响应现在将包含一个 `Link` header，而不是将分页链接包含为响应主体的一部分。

## 分页和模式

通过实现 `get_schema_fields()` 方法，你还可以使分页控件可用于 REST framework 提供的模式自动生成。此方法应具有以下签名：

`get_schema_fields(self, view)`

该方法应该返回一个 `coreapi.Field` 实例列表。

---

![Link Header](http://www.django-rest-framework.org/img/link-header-pagination.png)

*自定义分页样式，使用  'Link' header*

---

# HTML 分页控件

默认情况下，使用分页类将导致 HTML 分页控件显示在可浏览的 API 中。有两种内置显示样式。 `PageNumberPagination` 和 `LimitOffsetPagination` 类显示包含上一页和下一页控件的页码列表。 `CursorPagination` 类显示更简单的样式，只显示上一页和下一页控件。

## 自定义控件

你可以覆盖渲染 HTML 分页控件的模板。这两种内置式样是：

* `rest_framework/pagination/numbers.html`
* `rest_framework/pagination/previous_and_next.html`

在全局模板目录中提供具有这些路径的模板将覆盖相关分页类的默认渲染。

或者，你可以通过在现有类的子类上完全禁用 HTML 分页控件，将 `template=None` 设置为该类的属性。然后，你需要配置你的 `DEFAULT_PAGINATION_CLASS` setting key，以将你的自定义类用作默认分页样式。

#### 低级 API

用于确定分页类是否应显示控件的低级 API 作为分页实例上的 `display_page_controls` 属性公开。如果需要显示HTML 分页控件，自定义分页类应该在 `paginate_queryset` 方法中设置为 `True` 。

`.to_html()` 和 `.get_html_context()` 方法也可以在自定义分页类中重写，以便进一步自定义控件的渲染方式。

