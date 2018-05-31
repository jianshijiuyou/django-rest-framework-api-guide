# URL 调度器

## Django 如何处理一个请求

当一个用户请求 Django 站点的一个页面，下面是 Django 系统决定执行哪个 Python 代码使用的算法：

1. Django 确定要使用的根 URLconf 模块。通常，这是 `ROOT_URLCONF` 设置的值，但是如果传入的 `HttpRequest` 对象具有 `urlconf` 属性（由中间件设置），则将使用其值来代替 `ROOT_URLCONF` 设置。
2. Django 加载该 Python 模块并查找变量 `urlpatterns`。这应该是 `django.urls.path()` 和/或 `django.urls.re_path()` 实例的 Python 列表。
3. Django 依次匹配每个 URL 模式，在与请求的 URL 匹配的第一个模式停下来。
4. 一旦某个 URL 模式匹配，Django 就会导入并调用给定的视图，该视图是一个简单的 Python 函数（或基于类的视图）。该视图通过以下参数传递：
    * 一个 `HttpRequest` 实例。
    * 如果匹配的 URL 模式没有返回任何命名组，则来自正则表达式的匹配作为位置参数提供。
    * 关键字参数由路径表达式匹配的所有命名部分组成，并由 `django.urls.path()` 或 `django.urls.re_path()` 的可选 `kwargs` 参数中指定的任何参数覆盖。
5. 如果没有 URL 模式匹配，或者在此过程中的任何点发生异常，Django 将调用适当的错误处理视图。见下面的错误处理。

## 例如

下面是一个简单的 URLconf:

``` python
from django.urls import path

from . import views

urlpatterns = [
    path('articles/2003/', views.special_case_2003),
    path('articles/<int:year>/', views.year_archive),
    path('articles/<int:year>/<int:month>/', views.month_archive),
    path('articles/<int:year>/<int:month>/<slug:slug>/', views.article_detail),
]
```

注意：

* 要从 URL 中捕获值，请使用尖括号。
* 捕获的值可以选择包含转换器类型。例如，使用 `<int:name>` 来捕获整数参数。如果不包含转换器，则匹配除字符 `/` 之外的任何字符串。
* 没有必要添加一个前导斜杠，因为每个 URL 都有。例如，是 `articles`，而不是 `/articles`。

一些请求的例子：

* 请求 `/articles/2005/03/` 将与列表中的第三个条目匹配。Django 会调用函数 `views.month_archive(request, year=2005, month=3)`。
* `/articles/2003/` 将匹配列表中的第一个模式，而不是第二个模式，因为这些模式是按顺序测试的，而第一个模式是第一个符合的测试。在这里，Django 会调用函数 `views.special_case_2003(request)`。
* `/articles/2003` 不符合任何这些模式，因为每个模式都要求 URL 以斜杠结尾。
* `/articles/2003/03/building-a-django-site/` 将匹配最终模式。Django 会调用函数 `views.article_detail(request, year=2003, month=3, slug="building-a-django-site")`。

## Path 转换器

以下路 Path 换器默认可用：

* `str`：匹配任何非空字符串，不包括路径分隔符 `'/'`。如果转换器不包含在表达式中，这是默认值。
* `int`：匹配零或任何正整数。返回一个 `int`。
* `slug`：匹配任何由 ASCII 字母或数字组成的字符串，加上连字符和下划线字符。例如，`building-your-1st-django-site`。
* `uuid`：匹配格式化的 UUID。为防止多个 URL 映射到同一页面，必须包含破折号，并且字母必须是小写。例如， `075194d3-6885-417e-a8a8-6c931e272f00`。返回一个 `UUID` 实例。
* `path`：匹配任何非空字符串，包括路径分隔符 `'/'`。这可以让你匹配一个完整的 URL 路径，而不仅仅是一个 URL 路径的一部分，就像 `str` 一样。

## 使用正则表达式

如果路径和转换器语法不足以定义您的 URL 模式，您还可以使用正则表达式。为此，请使用 `re_path()` 而不是 `path()`。

在 Python 正则表达式中，命名正则表达式组的语法是 `(?P<name>pattern)`，其中 `name` 是组的名称，`pattern` 是一些要匹配的模式。

以下是使用正则表达式重写的前面的示例 URLconf：

``` python
from django.urls import path, re_path

from . import views

urlpatterns = [
    path('articles/2003/', views.special_case_2003),
    re_path(r'^articles/(?P<year>[0-9]{4})/$', views.year_archive),
    re_path(r'^articles/(?P<year>[0-9]{4})/(?P<month>[0-9]{2})/$', views.month_archive),
    re_path(r'^articles/(?P<year>[0-9]{4})/(?P<month>[0-9]{2})/(?P<slug>[\w-]+)/$', views.article_detail),
]
```

这完成了与前面的例子大致相同的事情，除了：

* 匹配的确切 URL 会受到一些限制。例如，年份 `10000` 将不再匹配，因为年份整数限制为四位数字。
* 无论正则表达式匹配什么类型，每个捕获的参数都以字符串的形式发送到视图。

从使用 `path()` 切换到 `re_path()` 或反过来切换时，注意视图参数的类型可能会更改尤为重要，因此您可能需要调整视图。

### 使用未命名的正则表达式组

除了命名组语法，例如 `(?P<year>[0-9]{4})`，您也可以使用较短的未命名组，例如 `([0-9]{4})`。

!> 不推荐使用

### 嵌套参数

正则表达式允许嵌套参数，Django 将解析它们并将它们传递给视图。当反转时，Django 将尝试填充所有外部捕获的参数，而忽略任何嵌套的捕获参数。考虑下面的 URL 模式，可以选择使用页面参数：

``` python
from django.urls import re_path

urlpatterns = [
    re_path(r'^blog/(page-(\d+)/)?$', blog_articles),                  # bad
    re_path(r'^comments/(?:page-(?P<page_number>\d+)/)?$', comments),  # good
]
```

两种模式都使用嵌套参数并解决：例如，`blog/page-2/` 会导致与具有两个位置参数的 `blog_articles` 匹配：`page-2/` 和 `2`。第二种 `comments` 模式将匹配 `comments/page-2/`，关键字参数 `page_number` 设置为 `2`。这种情况下的外部参数是一个非捕获参数 `(?:...)`。

## URLconf 搜索的内容

请求的 URL 被看做是一个普通的 Python 字符串， URLconf 在其上查找并匹配。进行匹配时将不包括 GET 或 POST 请求方式的参数以及域名。

例如， `https://www.example.com/myapp/` 请求中，URLconf 将查找 `myapp/`

在 `https://www.example.com/myapp/?page=3` 请求中，URLconf 仍将查找 `myapp/` 。

URLconf 不检查使用了哪种请求方法。换句话讲，所有的请求方法 —— 即，对同一个 URL 的无论是 POST 请求 、 GET 请求 、或 HEAD 请求方法等等 —— 都将路由到相同的函数。

## 指定视图参数的默认值

一个方便的技巧是为视图的参数指定默认参数。这里有一个 URLconf 和 view 的例子：

``` python
# URLconf
from django.urls import path

from . import views

urlpatterns = [
    path('blog/', views.page),
    path('blog/page<int:num>/', views.page),
]

# View (in blog/views.py)
def page(request, num=1):
    # Output the appropriate page of blog entries, according to num.
    ...
```

在上面的例子中，两个 URL 模式指向相同的 view - `views.page` - 但是第一个模式并没有从 URL 中捕获任何东西。如果第一个模式匹配，则 `page()` 函数将使用其默认参数 `num`，1。如果第二个模式匹配，`page()` 将使用捕获的任何 `num` 值。

## 错误处理

当 Django 无法为请求的 URL 找到匹配项或者引发异常时，Django 会调用错误处理视图。

这些情况发生时使用的视图通过4个变量指定。它们的默认值应该满足大部分项目，但是通过赋值给它们以进一步的自定义也是可以的。

完整的细节请参见 [自定义错误视图](https://docs.djangoproject.com/zh-hans/2.0/topics/http/views/#customizing-error-views) 。

这些值得在你的根 URLconf 中设置。在其它 URLconf 中设置这些变量将不会生效果。

它们的值必须是可调用的或者是表示视图的 Python 完整导入路径的字符串，可以方便地调用它们来处理错误情况。

这些值是：

* handler400 -- 查看 [django.conf.urls.handler400](https://docs.djangoproject.com/zh-hans/2.0/ref/urls/#django.conf.urls.handler400).
* handler403 -- 查看 [django.conf.urls.handler403](https://docs.djangoproject.com/zh-hans/2.0/ref/urls/#django.conf.urls.handler403).
* handler404 -- 查看 [django.conf.urls.handler404](https://docs.djangoproject.com/zh-hans/2.0/ref/urls/#django.conf.urls.handler404).
* handler500 -- 查看 [django.conf.urls.handler500](https://docs.djangoproject.com/zh-hans/2.0/ref/urls/#django.conf.urls.handler500).

## 包含其他 URLconf

在任何时候，你的 `urlpatterns` 都可以 “包含” 其他 URLconf 模块。这实质上是将一组网址 “植根于” 其他网址之下。

例如，下面是 Django 网站本身的 URLconf 的摘录。它包含许多其他 URLconf：

``` python
from django.urls import include, path

urlpatterns = [
    # ... snip ...
    path('community/', include('aggregator.urls')),
    path('contact/', include('contact.urls')),
    # ... snip ...
]
```

每当 Django 遇到 `include()` 时，它会截断与该点匹配的 URL 部分，并将剩余的字符串发送到包含的 URLconf 以供进一步处理。

另一种可能性是通过使用 `path()` 实例列表来包含其他 URL 模式。例如，考虑这个 URLconf：

``` python
from django.urls import include, path

from apps.main import views as main_views
from credit import views as credit_views

extra_patterns = [
    path('reports/', credit_views.report),
    path('reports/<int:id>/', credit_views.report),
    path('charge/', credit_views.charge),
]

urlpatterns = [
    path('', main_views.homepage),
    path('help/', include('apps.help.urls')),
    path('credit/', include(extra_patterns)),
]
```

在这个例子中，`/credit/reports/` URL 将由 `credit_views.report()` Django 视图处理。

这可以用来消除重复使用单个模式前缀的 URLconf 中的冗余。例如，考虑这个 URLconf：

``` python
from django.urls import path
from . import views

urlpatterns = [
    path('<page_slug>-<page_id>/history/', views.history),
    path('<page_slug>-<page_id>/edit/', views.edit),
    path('<page_slug>-<page_id>/discuss/', views.discuss),
    path('<page_slug>-<page_id>/permissions/', views.permissions),
]
```

我们可以通过只声明一次公共路径前缀并对不同的后缀进行分组来改善这一点：

``` python
from django.urls import include, path
from . import views

urlpatterns = [
    path('<page_slug>-<page_id>/', include([
        path('history/', views.history),
        path('edit/', views.edit),
        path('discuss/', views.discuss),
        path('permissions/', views.permissions),
    ])),
]
```

### 捕获的参数

包含的 URLconf 从父 URLconf 接收任何捕获的参数，因此以下示例是有效的：

``` python
# In settings/urls/main.py
from django.urls import include, path

urlpatterns = [
    path('<username>/blog/', include('foo.urls.blog')),
]

# In foo/urls/blog.py
from django.urls import path
from . import views

urlpatterns = [
    path('', views.blog.index),
    path('archive/', views.blog.archive),
]
```

在上面的例子中，如预期的那样，捕获的 “username” 变量被传递给包含的 URLconf。

## 传递额外的选项到视图函数

URLconf 有一个钩子，允许你将额外的参数作为 Python 字典传递给你的视图函数。

`path()` 函数可以接受一个可选的第三个参数，它应该是一个额外的关键字参数字典传递给视图函数。

``` python
from django.urls import path
from . import views

urlpatterns = [
    path('blog/<int:year>/', views.year_archive, {'foo': 'bar'}),
]
```

在这个例子中，对于 `/blog/2005/` 的请求，Django 将调用 `views.year_archive(request, year=2005, foo='bar')`。

### 传递额外选项到 include()

同样，您可以将额外选项传递给 `include()`，并且包含的 ​​URLconf 中的每一行都将传递额外的选项。

例如，这两个 URLconf 集在功能上是相同的：

第一种：

``` python
# main.py
from django.urls import include, path

urlpatterns = [
    path('blog/', include('inner'), {'blog_id': 3}),
]

# inner.py
from django.urls import path
from mysite import views

urlpatterns = [
    path('archive/', views.archive),
    path('about/', views.about),
]
```

第二种：

``` python
# main.py
from django.urls import include, path
from mysite import views

urlpatterns = [
    path('blog/', include('inner')),
]

# inner.py
from django.urls import path

urlpatterns = [
    path('archive/', views.archive, {'blog_id': 3}),
    path('about/', views.about, {'blog_id': 3}),
]
```

请注意，无论该行的视图是否实际接受这些选项都是有效的，总是会将其他选项传递给包含的URLconf中的每一行。出于这个原因，只有在确定包含的URLconf中的每个视图都接受了您传递的额外选项时，此技术才有用。

## URL 的反向解析

Django 提供了用于执行 URL 反转的工具，以匹配需要 URL 的不同层：
* 在模板中：使用 url 模板标签。
* 在 Python 代码中：使用该 `reverse()` 函数。
* 在与处理 Django 模型实例的 URL 相关的更高级别的代码中：`get_absolute_url()` 方法。

### 举个栗子

再考虑一下这个 URLconf 条目：

``` python
from django.urls import path

from . import views

urlpatterns = [
    #...
    path('articles/<int:year>/', views.year_archive, name='news-year-archive'),
    #...
]
```

根据此设计，对应于年 `nnnn` 的档案的 URL 是 `/articles/<nnnn>/`。

您可以使用以下方式在模板代码中获得这些内容

``` html
<a href="{% url 'news-year-archive' 2012 %}">2012 Archive</a>
{# Or with the year in a template context variable: #}
<ul>
{% for yearvar in year_list %}
<li><a href="{% url 'news-year-archive' yearvar %}">{{ yearvar }} Archive</a></li>
{% endfor %}
</ul>
```

或者在 Python 代码中：

``` python
from django.http import HttpResponseRedirect
from django.urls import reverse

def redirect_to_year(request):
    # ...
    year = 2006
    # ...
    return HttpResponseRedirect(reverse('news-year-archive', args=(year,)))
```

如果出于某种原因决定应该更改每年发布文章内容的 URL，那么您只需要更改 URLconf 中的条目。

在某些情况下，视图具有通用性，因此 URL 和视图之间可能存在多对一的关系。对于这些情况，视图名称在倒转 URL 时不是一个足够好的标识符。阅读下一节以了解 Django 为此提供的解决方案。

## 命名 URL 模式

为了执行 URL 反转，您需要使用上述示例中所做的命名的 URL 模式。用于 URL 名称的字符串可以包含您喜欢的任何字符。不限于有效的 Python 名称。

命名 URL 模式时，请选择不太可能与其他应用程序的名称选择冲突的名称。如果您调用 URL 模式 `comment` 并且其他应用程序执行相同操作，则 `reverse()` 查找的 URL 取决于项目的 `urlpatterns` 列表中最后一个模式。

在您的 URL 名称上加上一个前缀（可能来自应用程序名称（例如 `myapp-comment`，而不是 `comment`））会降低碰撞的可能性。

如果您想覆盖视图，您可以故意选择与另一个应用程序相同的 URL 名称。例如，一个常见的用例是重写 `LoginView`。部分 Django 和大多数第三方应用程序假定此视图具有名称 `login` 的 URL 模式。如果你有一个自定义的登录视图，并给它的 URL 名称是 `login`，`reverse()` 会找到你的自定义视图，只要它在包含 `django.contrib.auth.urls`（如果包括它的话）后的 `urlpatterns`。

如果参数不同，您也可以为多个 URL 模式使用相同的名称。除了 URL 名称，`reverse()` 匹配参数的数量和关键字参数的名称。

## URL 命名空间

看个例子

`urls.py`

``` python
from django.urls import include, path

urlpatterns = [
    path('author-polls/', include('polls.urls', namespace='author-polls')),
    path('publisher-polls/', include('polls.urls', namespace='publisher-polls')),
]
```

`polls/urls.py`

``` python
from django.urls import path

from . import views

app_name = 'polls'
urlpatterns = [
    path('', views.IndexView.as_view(), name='index'),
    path('<int:pk>/', views.DetailView.as_view(), name='detail'),
    ...
]
```

使用

``` python
reverse('author-polls:index')
```

### URL 命名空间和包含的 URLconf

可以通过两种方式指定包含的 URLconf 的应用名称空间。

第一种，你可以在包含的 URLconf 模块中设置与 urlpatterns 属性相同级别的 app_name 属性。 你必须将实际模块或模块的字符串引用传递到 `include()` ，而不是 `urlpatterns` 本身的列表。

`polls/urls.py`

``` python
from django.urls import path

from . import views

app_name = 'polls'
urlpatterns = [
    path('', views.IndexView.as_view(), name='index'),
    path('<int:pk>/', views.DetailView.as_view(), name='detail'),
    ...
]
```

`urls.py`

``` python
from django.urls import include, path

urlpatterns = [
    path('polls/', include('polls.urls')),
]
```

`polls.urls` 中定义的 URL 将具有应用名称空间 `polls`。

第二种，你可以 include 一个包含嵌套命名空间数据的对象。 如果你 `include()` 一个 `url()` 实例的列表，那么该对象中包含的 URL 将添加到全局命名空间。 但是，你也可以 `include()` 一个 2 元组，其中包含：

```
(<list of path()/re_path() instances>, <application namespace>)
```

例如：

``` python
from django.urls import include, path

from . import views

polls_patterns = ([
    path('', views.IndexView.as_view(), name='index'),
    path('<int:pk>/', views.DetailView.as_view(), name='detail'),
], 'polls')

urlpatterns = [
    path('polls/', include(polls_patterns)),
]
```

这将 include 指定的 URL 模式到给定的应用命名空间。

可以使用 `include()` 的 namespace 参数指定实例命名空间。 如果未指定，则实例命名空间默认为 URLconf 的应用命名空间。 这意味着它也将是该命名空间的默认实例。

## 静态资源配置

### 用户上传的文件

可以像下面这样

``` python
from django.conf import settings
from django.urls import re_path
from django.views.static import serve

# ... the rest of your URLconf goes here ...

if settings.DEBUG:
    urlpatterns += [
        re_path(r'^media/(?P<path>.*)$', serve, {
            'document_root': settings.MEDIA_ROOT,
        }),
    ]
```

或者像下面这样

``` python
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    # ... the rest of your URLconf goes here ...
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

`settings.py` 中：

``` python
MEDIA_URL = '/media/'

MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
```

### Web 静态资源

图片，JavaScript，CSS 等

1. 确保 `django.contrib.staticfiles` 包含在你的 `INSTALLED_APPS`。

2. 在您的 settings.py 文件中，定义 `STATIC_URL`，例如：

    ``` python
    STATIC_URL = '/static/'
    ```

3. 在您的模板中，使用 `static` 模板标签，使用配置的 `STATICFILES_STORAGE` 为给定的相对路径构建 URL。

    ``` html
    {% load static %}
    <img src="{% static "my_app/example.jpg" %}" alt="My image"/>
    ```

4. 将您的静态文件存储在应用程序中名为 `static` 的文件夹中。例如
    `my_app/static/my_app/example.jpg`

您的项目可能还会有不与特定应用绑定的静态资产。除了在应用程序中使用 `static/` 目录之外，您还可以在 setting 文件中定义一个目录列表（`STATICFILES_DIRS`），Django 也将查找其中的静态文件。例如：

``` python
STATICFILES_DIRS = [
    os.path.join(BASE_DIR, "static"),
    '/var/www/static/',
]
```

#### 在开发过程中提供静态文件

例如，如果您的 `STATIC_URL` 定义为 `/static/`，则可以通过将以下代码段添加到您的 `urls.py` 中来完成此操作：

``` python
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    # ... the rest of your URLconf goes here ...
] + static(settings.STATIC_URL, document_root=settings.STATIC_ROOT)
```

#### 部署静态文件

`django.contrib.staticfiles` 提供了一个便捷的管理命令，用于在单个目录中收集静态文件，以便您可以轻松地为其提供服务。

1. 将 `STATIC_ROOT` setting 设置为您想要从中提供这些文件的目录，例如：

    ``` python
    STATIC_ROOT = "/var/www/example.com/static/"
    ```

2. 运行 `collectstatic` 管理命令：

    ```
    $ python manage.py collectstatic
    ```

    这会将静态文件夹中的所有文件复制到 `STATIC_ROOT` 目录中。

3. 使用您选择的 Web 服务器来提供静态文件访问，如 Nginx。