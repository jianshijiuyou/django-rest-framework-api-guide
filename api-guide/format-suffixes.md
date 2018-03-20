> [官方原文链接](http://www.django-rest-framework.org/api-guide/format-suffixes/)  

# Format 后缀



Web API 的常见模式是在 URL 上使用文件扩展名来为给定的媒体类型提供端点。 例如，`'http://example.com/api/users.json'` 用于提供 JSON 表示。

在 URLconf 中为你的 API 添加 format-suffix 模式是容易出错和非 DRY 的，因此 REST framework 提供了将这些模式添加到 URLconf 的快捷方式。

## format_suffix_patterns

**签名**: `format_suffix_patterns(urlpatterns, suffix_required=False, allowed=None)`

返回一个 URL pattern 列表，其中包含附加到每个 URL pattern 的格式后缀模式。

参数：

* **urlpatterns**: 必需。一个 URL pattern 列表。
* **suffix_required**:  可选。一个 boolean 值，指定 URL 中的后缀是否可选或强制。默认为 `False`，这意味着后缀默认是可选的。
* **allowed**:  可选。有效格式后缀的列表或元组。如果没有提供，将使用通配符格式后缀模式。

例如：

``` python
from rest_framework.urlpatterns import format_suffix_patterns
from blog import views

urlpatterns = [
    url(r'^/$', views.apt_root),
    url(r'^comments/$', views.comment_list),
    url(r'^comments/(?P<pk>[0-9]+)/$', views.comment_detail)
]

urlpatterns = format_suffix_patterns(urlpatterns, allowed=['json', 'html'])
```

在使用 `format_suffix_patterns` 时，你必须确保将 `'format'` 关键字参数添加到相应的视图。例如：

``` python
@api_view(('GET', 'POST'))
def comment_list(request, format=None):
    # do stuff...
```

或者基于类视图：

``` python
class CommentList(APIView):
    def get(self, request, format=None):
        # do stuff...

    def post(self, request, format=None):
        # do stuff...
```

所使用的 kwarg 的名称可以使用 `FORMAT_SUFFIX_KWARG` setting 进行修改。

另请注意，`format_suffix_patterns` 不支持降序包含 URL patterns。

### 与 `i18n_patterns` 一起使用

如果使用 Django 提供的 `i18n_patterns` 函数以及 `format_suffix_patterns`，则应确保将 `i18n_patterns` 函数用作最终或最外层函数。例如：

``` python
url patterns = [
    …
]

urlpatterns = i18n_patterns(
    format_suffix_patterns(urlpatterns, allowed=['json', 'html'])
)
```

---

## 查询参数 format

格式后缀的替代方法是将请求的 format 包含在查询参数中。REST framework 默认提供此选项，并且它在可浏览的 API 中用于在不同的可用表示之间切换。

要使用其短格式表示，请使用 `format` 查询参数。例如： `http://example.com/organizations/?format=csv`。

此查询参数的名称可以使用 `URL_FORMAT_OVERRIDE` 设置进行修改。将该值设置为 `None` 以禁用此行为。

