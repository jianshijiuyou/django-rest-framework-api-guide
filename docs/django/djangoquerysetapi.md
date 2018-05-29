# QuerySet API

`class QuerySet(model=None, query=None, using=None)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/query/#QuerySet)

QuerySet 类具有两个公有属性用于内省：

`ordered`

如果 `QuerySet` 是排好序的则为 `Tru`e —— 例如有一个 `order_by()` 子句或者模型有默认的排序。 否则为 `False` 。

`db`

如果现在执行，则返回将使用的数据库。

## 返回新 QuerySet 的方法

Django 提供了一系列 的 `QuerySet` 筛选方法，用于改变 `QuerySet` 返回的结果类型或者 SQL 查询执行的方式。

### filter()

`filter(**kwargs)`

返回一个新的QuerySet，它包含满足查询参数的对象。

查找的参数（**kwargs）应该满足下文字段查找中的格式。 在底层的 SQL 语句中，多个参数通过 `AND` 连接。

如果你需要执行更复杂的查询（例如 `OR` 语句），你可以使用 `Q` 对象。

### exclude()

`exclude(**kwargs)`

返回一个新的 `QuerySet`，它包含不满足给定的查找参数的对象。

查找的参数（**kwargs）应该满足下文字段查找中的格式。 在底层的 `SQL` 语句中，多个参数通过 `AND` 连接，然后所有的内容放入 `NOT()` 中。

下面的示例排除所有 `pub_date` 晚于 `2005-1-3` 且 `headline` 为 `"Hello"` 的记录：

``` python
Entry.objects.exclude(pub_date__gt=datetime.date(2005, 1, 3), headline='Hello')
```

它等同于 `SQL` 语句：

``` python
SELECT ...
WHERE NOT (pub_date > '2005-1-3' AND headline = 'Hello')
```

下面的示例排除所有 `pub_date` 晚于 `2005-1-3` 或者 `headline` 为 `"Hello"` 的记录：

``` python
Entry.objects.exclude(pub_date__gt=datetime.date(2005, 1, 3)).exclude(headline='Hello')
```

它等同于 `SQL` 语句：

``` python
SELECT ...
WHERE NOT pub_date > '2005-1-3'
AND NOT headline = 'Hello'
```

### annotate()

`annotate(*args, **kwargs)`

给返回的记录添加一个额外的临时字段，这个字段不会存入数据库。

例如，如果您在操作博客列表，则可能需要确定每个博客中有多少个条目：

``` python
>>> from django.db.models import Count
>>> q = Blog.objects.annotate(Count('entry'))
# The name of the first blog
>>> q[0].name
'Blogasaurus'
# The number of entries on the first blog
>>> q[0].entry__count
42
```

Blog模型本身并未定义entry__count属性，但通过使用关键字参数指定聚合函数，您可以控制额外字段的名称：

``` python
>>> q = Blog.objects.annotate(number_of_entries=Count('entry'))
# The number of entries on the first blog, using the name provided
>>> q[0].number_of_entries
42
```

### order_by()

`order_by(*fields)`

默认情况下，`QuerySet` 返回的结果由模型 `Meta` 中的 `ordering` 选项给出的排序元组排序。您可以通过使用 `order_by` 方法在每个 `QuerySet` 的基础上覆盖此值。

例如：

``` python
Entry.objects.filter(pub_date__year=2005).order_by('-pub_date', 'headline')
```

上面的结果将按 `pub_date` 降序排序，然后按 `headline` 升序排序。 “-pub_date” 前面的负号表示降序。默认升序。要随机排序，请使用“`？`”，如下所示：

``` python
Entry.objects.order_by('?')
```

!> 注：`order_by('?')` 查询可能耗费资源且很慢，这取决于使用的数据库。

若要按照另外一个模型中的字段排序，可以使用查询关联模型时的语法。 即通过字段的名称后面跟上两个下划线（__），再跟上新模型中的字段的名称，直至你希望连接的模型。 像这样：

``` python
Entry.objects.order_by('blog__name', 'headline')
```

如果排序的字段与另外一个模型关联，Django 将使用关联的模型的默认排序，或者如果没有指定 `Meta.ordering` 将通过关联的模型的主键排序。 例如，因为 `Blog` 模型没有指定默认的排序：

``` python
Entry.objects.order_by('blog')
```

...与以下相同：

``` python
Entry.objects.order_by('blog__id')
```

如果 `Blog` 设置 `ordering = ['name']`，那么第一个 `QuerySet` 将等同于：

``` python
Entry.objects.order_by('blog__name')
```

您还可以通过在表达式上调用 `asc()` 或 `desc()` 来按查询表达式进行排序：

``` python
Entry.objects.order_by(Coalesce('summary', 'headline').desc())
```

你可以通过 `Lower` 将一个字段转换为小写来排序，它将达到大小写一致的排序：

``` python
Entry.objects.order_by(Lower('headline').desc())
```

如果您不希望将任何顺序应用于查询，甚至不需要默认排序，则可以调用不带参数的 `order_by()`。

您可以通过检查 `QuerySet.ordered` 属性来确定是否排序查询，如果 `QuerySet` 已以任何方式排序，则该属性将为 `True`。

每个 `order_by()` 调用都将清除以前的任何顺序。例如，该查询将按 `pub_date` 排序而不是 `headline`：

``` python
Entry.objects.order_by('headline').order_by('pub_date')
```

### reverse()

使用 `reverse()` 方法可以反转查询集元素的返回顺序。第二次调用 `reverse()` 会将顺序恢复到正常方向。

要检索查询集中的“最后”五个项目，可以这样做：

``` python
my_queryset.reverse()[:5]
```

请注意，这与从 Python 中的序列末尾进行切片并不完全相同。上面的例子会先返回最后一个 item，然后返回倒数第二个 item，依此类推。如果我们有一个 Python 序列并查看了 `seq[-5:]`，我们会首先看到倒数第五项。 Django 不支持这种访问模式（从最后切入），因为在 SQL 中无法高效地完成它。

此外，请注意 `reverse()` 通常只应在具有已定义顺序的 `QuerySet` 上调用（例如，在查询定义默认顺序的模型或使用 `order_by()` 时）。如果给定的查询集没有定义这样的排序，那么调用 `reverse()` 就没有实际效果（排序在调用 `reverse()` 之前是未定义的，并且之后将保持未定义）。

### distinct()

`distinct(*fields)`

返回一个在 SQL 查询中使用 `SELECT DISTINCT` 的新 `QuerySet`。 它将去除查询结果中重复的行。

默认情况下，`QuerySet` 不会去除重复的行。 在实际应用中，这一般不是个问题，因为像 `Blog.objects.all()` 这样的简单查询不会引入重复的行。 但是，如果查询跨越多张表，当对 `QuerySet` 求值时就可能得到重复的结果。 这时候你应该使用 `distinct()`。

### values()

`values(*fields, **expressions)`

作为迭代器使用时，返回一个返回字典的 `QuerySet`，而不是模型实例。

这些字典中的每一个都表示一个对象，其中的键与模型对象的属性名称相对应。

本示例将 `values()` 的字典与普通模型对象进行比较：

``` python
# This list contains a Blog object.
>>> Blog.objects.filter(name__startswith='Beatles')
<QuerySet [<Blog: Beatles Blog>]>

# This list contains a dictionary.
>>> Blog.objects.filter(name__startswith='Beatles').values()
<QuerySet [{'id': 1, 'name': 'Beatles Blog', 'tagline': 'All the latest Beatles news.'}]>
```

`values()` 方法使用可选的位置参数 `*fields`，它们指定 SELECT 应该限制到的字段名称。如果指定了字段，每个字典将只包含您指定的字段的字段键/值。如果不指定字段，则每个字典将包含数据库表中每个字段的键和值。

例如：

``` python
>>> Blog.objects.values()
<QuerySet [{'id': 1, 'name': 'Beatles Blog', 'tagline': 'All the latest Beatles news.'}]>
>>> Blog.objects.values('id', 'name')
<QuerySet [{'id': 1, 'name': 'Beatles Blog'}]>
```

`values()` 方法还使用可选的关键字参数 `**expressions`，它们被传递给 `annotate()`：

``` python
>>> from django.db.models.functions import Lower
>>> Blog.objects.values(lower_name=Lower('name'))
<QuerySet [{'lower_name': 'beatles blog'}]>
```

`values()` 子句中的聚合应用于相同 `values()` 子句中的其他参数之前。如果需要按另一个 value 进行分组，请将其添加到较早的 `values()` 子句中。例如：

``` python
>>> from django.db.models import Count
>>> Blog.objects.values('entry__authors', entries=Count('entry'))
<QuerySet [{'entry__authors': 1, 'entries': 20}, {'entry__authors': 1, 'entries': 13}]>
>>> Blog.objects.values('entry__authors').annotate(entries=Count('entry'))
<QuerySet [{'entry__authors': 1, 'entries': 33}]>
```

值得一提的几个微妙之处：

* 如果您有一个名为 `foo` 的字段是 `ForeignKey`，则默认的 `values()` 调用将返回一个名为 `foo_id` 的字典键，因为这是存储实际值的隐藏模型属性的名称（`foo` 属性指的是相关模型）。当你调用 `values()` 并传入字段名称时，你可以传入 `foo` 或 `foo_id` ，你将得到相同的结果（字典关键字将匹配传入的字段名称）。

    例如：

    ``` python
    >>> Entry.objects.values()
    <QuerySet [{'blog_id': 1, 'headline': 'First Entry', ...}, ...]>

    >>> Entry.objects.values('blog')
    <QuerySet [{'blog': 1}, ...]>

    >>> Entry.objects.values('blog_id')
    <QuerySet [{'blog_id': 1}, ...]>
    ```

* 将 `values()` 与 `distinct()` 一起使用时，请注意排序可能会影响结果。有关详细信息，请参阅 [`distinct()`](https://docs.djangoproject.com/zh-hans/2.0/ref/models/querysets/#django.db.models.query.QuerySet.distinct) 中的注释。
* 如果在 `extra()` 调用之后使用了 `values()` 子句，那么 `extra()` 中的 `select` 参数定义的任何字段都必须显式包含在 `values()` 调用中。在 `value()` 调用后进行的任何 `extra()` 调用将忽略其额外选定的字段。
* 在 `values()` 之后调用 `only()` 和 `defer()` 没有意义，所以这样做会引发 `NotImplementedError`。

当您知道只需要少量可用字段中的值并且不需要模型实例对象的功能时，它非常有用。仅选择需要使用的字段会更有效。

最后，请注意，您可以在 `values()` 调用后调用 `filter()`，`order_by()` 等，这意味着这两个调用是相同的：

``` python
Blog.objects.values().order_by('id')
Blog.objects.order_by('id').values()
```

制作 Django 的人倾向于首先放置所有影响 SQL 的方法，然后（可选）使用任何影响输出的方法（如 `values()`），但这并不重要。这是你真正炫耀你的个人主义的机会。

您还可以通过 `OneToOneField`，`ForeignKey` 和 `ManyToManyField` 属性参考相关模型中具有反向关系的字段：

``` python
>>> Blog.objects.values('name', 'entry__headline')
<QuerySet [{'name': 'My blog', 'entry__headline': 'An entry'},
     {'name': 'My blog', 'entry__headline': 'Another entry'}, ...]>
```

### values_list()

`values_list(*fields, flat=False, named=False)`

这与 `values()` 类似，除了不是返回字典而是在迭代时返回元组。每个元组都包含传递给 `values_list()` 调用的相应字段或表达式的值 -- 因此第一个项目是第一个字段等。例如：

``` python
>>> Entry.objects.values_list('id', 'headline')
<QuerySet [(1, 'First entry'), ...]>
>>> from django.db.models.functions import Lower
>>> Entry.objects.values_list('id', Lower('headline'))
<QuerySet [(1, 'first entry'), ...]>
```

如果您只传入单个字段，则还可以传入 `flat` 参数。如果为 `True`，这将意味着返回的结果是单个值，而不是一元组。一个例子应该使差异更加清晰：

``` python
>>> Entry.objects.values_list('id').order_by('id')
<QuerySet[(1,), (2,), (3,), ...]>

>>> Entry.objects.values_list('id', flat=True).order_by('id')
<QuerySet [1, 2, 3, ...]>
```

当有多个字段时，传入 `flat` 是错误的。

你可以通过 `named=True` 来得到结果作为 `namedtuple()`：

``` python
>>> Entry.objects.values_list('id', 'headline', named=True)
<QuerySet [Row(id=1, headline='First entry'), ...]>
```

使用命名元组可能会使结果更具可读性，代价是将结果转换为命名元组的代价很小。

如果您没有将任何值传递给 `values_list()`，它将按照声明的顺序返回模型中的所有字段。

常见的需求是获取特定模型实例的特定字段值。要实现这一点，请使用 `values_list()`，后跟 `get()` 调用：

``` python
>>> Entry.objects.values_list('headline', flat=True).get(pk=1)
'First entry'
```

`values()` 和 `values_list()` 都是针对特定用例的优化：在没有创建模型实例的开销的情况下检索数据的子集。当处理多对多和其他多值关系（比如反向外键的一对多关系）时，这种隐喻会分崩离析，因为“一行一个对象”假设不成立。

例如，请注意跨 `ManyToManyField` 查询时的行为：

``` python
>>> Author.objects.values_list('name', 'entry__headline')
<QuerySet [('Noam Chomsky', 'Impressions of Gaza'),
 ('George Orwell', 'Why Socialists Do Not Believe in Fun'),
 ('George Orwell', 'In Defence of English Cooking'),
 ('Don Quixote', None)]>
```

具有多个条目的作者多次出现，没有任何条目的作者对条目标题具有 `None`。

类似地，当查询反向外键时，对于没有任何作者的条目，出现 `None`

``` python
>>> Entry.objects.values_list('authors')
<QuerySet [('Noam Chomsky',), ('George Orwell',), (None,)]>
```

### dates()

`dates(field, kind, order='ASC')`

返回一个 `QuerySet`，该 `QuerySet` 的计算结果为一个 `datetime.date` 对象列表，该对象表示 `QuerySet` 内容中特定类型的所有可用日期。

`field` 应该是模型的 `DateField` 的名称。`kind` 应该是 `"year"`，`"month"` 或 `"day"`。结果列表中的每个 `datetime.date` 对象都被 “截断” 为给定 `type`。

* `"year"` 返回该字段的所有不同年份值的列表。
* `"month"` 返回该字段的所有不同年/月值的列表。
* `"day"` 返回该字段的所有不同年/月/日值的列表。

`order` ，默认为 `'ASC'`，可以是 `'ASC'` 或 `'DESC'`。这指定了如何排序结果。

例如：

``` python
>>> Entry.objects.dates('pub_date', 'year')
[datetime.date(2005, 1, 1)]
>>> Entry.objects.dates('pub_date', 'month')
[datetime.date(2005, 2, 1), datetime.date(2005, 3, 1)]
>>> Entry.objects.dates('pub_date', 'day')
[datetime.date(2005, 2, 20), datetime.date(2005, 3, 20)]
>>> Entry.objects.dates('pub_date', 'day', order='DESC')
[datetime.date(2005, 3, 20), datetime.date(2005, 2, 20)]
>>> Entry.objects.filter(headline__contains='Lennon').dates('pub_date', 'day')
[datetime.date(2005, 3, 20)]
```

### datetimes()

`datetimes(field_name, kind, order='ASC', tzinfo=None)`

返回一个 `QuerySet`，该 `QuerySe` t的计算结果为一个 `datetime.datetime` 对象列表，该对象表示 `QuerySet` 内容中特定类型的所有可用日期。

`field_name` 应该是模型的 `DateTimeField` 的名称。

`kind` 应该是 "year", "month", "day", "hour", "minute" 或 "second"。结果列表中的每个 `datetime.datetime` 对象都被截断为给定的 type。


`order` ，默认为 `'ASC'`，可以是 `'ASC'` 或 `'DESC'`。这指定了如何排序结果。

`tzinfo` 定义在截断之前将日期时间转换到的时区。事实上，根据使用的时区，给定的日期时间具有不同的表示形式。该参数必须是一个 `datetime.tzinfo` 对象。如果它是 `None` ，Django 将使用当前时区。当 `USE_TZ` 为 `False` 时它不起作用。

### none()

调用 `none()` 将创建一个从不返回任何对象的查询集，并且在访问结果时不会执行任何查询。 `qs.none()` `queryset` 是 `EmptyQuerySet` 的一个实例。

例如：

``` python
>>> Entry.objects.none()
<QuerySet []>
>>> from django.db.models.query import EmptyQuerySet
>>> isinstance(Entry.objects.none(), EmptyQuerySet)
True
```

### all()

返回当前 `QuerySet`（或 `QuerySet` 子类）的副本。这在您可能想要传入模型管理器或 `QuerySet` 并对结果进行进一步过滤的情况下非常有用。在任何一个对象上调用 `all()` 之后，你肯定会有一个 `QuerySet` 可以使用。

在使用 `QuerySet` 时，通常会缓存其结果。如果数据库中的数据可能因为对 `QuerySet` 进行操作而发生了更改，则可以通过调用先前使用过的 `QuerySet上` 的 `all()` 来获取同一查询的更新结果。

### union()

`union(*other_qs, all=False)`

使用 SQL 的 UNION 运算符合并两个或更多 `QuerySet` 的结果。例如：

``` python
>>> qs1.union(qs2, qs3)
```

UNION 运算符默认只选择不同的值。要允许重复值，请使用 `all=True` 参数。

即使参数是其他模型的 `QuerySet`，`union()`，`intersection()` 和 `difference()` 也会返回第一个 `QuerySet` 类型的模型实例。只要 SELECT 列表在所有 `QuerySet` 中都是相同的（至少是类型，只要类型在同一排序中，名称不重要）就可以传递不同的模型。在这种情况下，您必须使用 `QuerySet` 中第一个 `QuerySet` 中的列名应用于生成的 `QuerySet`。例如：

``` python
>>> qs1 = Author.objects.values_list('name')
>>> qs2 = Entry.objects.values_list('headline')
>>> qs1.union(qs2).order_by('name')
```

此外，在结果 `QuerySet` 上只允许使用 `LIMIT, OFFSET, COUNT(*), ORDER BY` 和指定列（即 `切片`，`count()`，`order_by()` 和 `values()` / `values_list()`）。此外，数据库对组合查询中允许的操作进行限制。例如，大多数数据库不允许组合查询中的 `LIMIT` 或 `OFFSET`。

### intersection()

`intersection(*other_qs)`

使用 SQL 的 INTERSECT 运算符来返回两个或多个 QuerySet 的共享元素。例如：

``` python
>>> qs1.intersection(qs2, qs3)
```

有关某些限制，请参阅 `union()`。

### difference()

`difference(*other_qs)`

使用 SQL 的 EXCEPT 运算符仅保留 `QuerySet` 中存在的元素，但不保存在其他某些 `QuerySet` 中。例如：

``` python
>>> qs1.difference(qs2, qs3)
```

有关某些限制，请参阅 `union()`。

### select_related()

`select_related(*fields)`

返回一个 `QuerySet`，它将“跟随”外键关系，并在执行查询时选择其他相关对象数据。这是一个性能提升者，它会导致一个更复杂的查询，但意味着以后使用外键关系不需要数据库查询。

以下示例说明了普通查找与 `select_related()` 查找之间的区别。这里是标准查找：

``` python
# Hits the database.
e = Entry.objects.get(id=5)

# Hits the database again to get the related Blog object.
b = e.blog
```

这里是 `select_related` 查找：

``` python
# Hits the database.
e = Entry.objects.select_related('blog').get(id=5)

# Doesn't hit the database, because e.blog has been prepopulated
# in the previous query.
b = e.blog
```

您可以和任何 queryset 对象一起使用 `select_related()`：

``` python
from django.utils import timezone

# Find all the blogs with entries scheduled to be published in the future.
blogs = set()

for e in Entry.objects.filter(pub_date__gt=timezone.now()).select_related('blog'):
    # Without select_related(), this would make a database query for each
    # loop iteration in order to fetch the related blog for each entry.
    blogs.add(e.blog)
```

`filter()` 和 `select_related()` 链接的顺序并不重要。这些 queryset 是等效的：

``` python
Entry.objects.filter(pub_date__gt=timezone.now()).select_related('blog')
Entry.objects.select_related('blog').filter(pub_date__gt=timezone.now())
```

您可以按照类似的方式来查询外键。如果您有以下模型：

``` python
from django.db import models

class City(models.Model):
    # ...
    pass

class Person(models.Model):
    # ...
    hometown = models.ForeignKey(
        City,
        on_delete=models.SET_NULL,
        blank=True,
        null=True,
    )

class Book(models.Model):
    # ...
    author = models.ForeignKey(Person, on_delete=models.CASCADE)
```

...然后调用 `Book.objects.select_related('author__hometown').get(id=4) ` 将缓存相关 `Person` 和相关 `City`：

``` python
# Hits the database with joins to the author and hometown tables.
b = Book.objects.select_related('author__hometown').get(id=4)
p = b.author         # Doesn't hit the database.
c = p.hometown       # Doesn't hit the database.

# Without select_related()...
b = Book.objects.get(id=4)  # Hits the database.
p = b.author         # Hits the database.
c = p.hometown       # Hits the database.
```

您可以在传递给 `select_related()` 的字段列表中引用任何 `ForeignKey` 或 `OneToOneField` 关系。

您也可以在传递给 `select_related` 的字段列表中引用 `OneToOneField` 的相反方向 - 也就是说，您可以将 `OneToOneField` 遍历回到定义该字段的对象。而不是指定字段名称，请使用 `related_name` 作为相关对象上的字段。

在某些情况下，你希望用很多相关的对象调用 `select_related()`，或者你不知道所有的关系。在这些情况下，可以在不带任何参数的情况下调用 `select_related()`。这将遵循它可以找到的所有非空外键 - 必须指定可为空的外键。这在大多数情况下不被推荐，因为它可能会使底层查询更复杂，并返回比实际需要更多的数据。

如果您需要清除 `QuerySet` 上过去调用 `select_related` 所添加的相关字段的列表，则可以将 `None` 作为参数传递给它：

``` python
>>> without_relations = queryset.select_related(None)
```

链式 `select_related` 调用与其他方法类似 - 即 `select_related('foo', 'bar') ` 等同于 `select_related('foo').select_related('bar')` 。

### prefetch_related()

`prefetch_related(*lookups)`

返回一个 `QuerySet`，它将自动检索单个批处理中每个指定查找的相关对象。

这与 `select_related` 具有相似的目的，因为两者都旨在阻止访问相关对象导致的大量数据库查询，但策略是完全不同的。

`select_related` 通过创建 SQL 连接并在 SELECT 语句中包含相关对象的字段来工作。为此，`select_related` 在相同的数据库查询中获取相关对象。但是，为避免通过加入 “多” 关系而导致的更大的结果集，`select_related` 仅限于单值关系 - 外键 和 一对一。

另一方面，`prefetch_related` 为每个关系进行单独的查找，并在 Python 中执行“连接”。这使得它可以预取多对多和多对一的对象，除了 `select_related` 支持的外键和一对一关系之外，这不能使用 `select_related` 来完成。它也支持 `GenericRelation` 和 `GenericForeignKey` 的预取，但是它必须限制在一组同样的结果中。例如，仅当查询限制为一个 `ContentType` 时，才支持预取 `GenericForeignKey` 引用的对象。

例如，假设你有这些模型：

``` python
from django.db import models

class Topping(models.Model):
    name = models.CharField(max_length=30)

class Pizza(models.Model):
    name = models.CharField(max_length=50)
    toppings = models.ManyToManyField(Topping)

    def __str__(self):
        return "%s (%s)" % (
            self.name,
            ", ".join(topping.name for topping in self.toppings.all()),
        )
```

并允许：

``` python
>>> Pizza.objects.all()
["Hawaiian (ham, pineapple)", "Seafood (prawns, smoked salmon)"...
```

这个问题是每次 `Pizza.__str__()` 都要求 `self.toppings.all()` 它必须查询数据库，所以 `Pizza.objects.all() ` 会在 `Toppings` 表上为每个 item 运行查询 Pizza `QuerySet`。

我们可以使用 `prefetch_related` 简化为两个查询：

``` python
>>> Pizza.objects.all().prefetch_related('toppings')
```

这意味着一个 `self.toppings.all()` 为每个 Pizza; 现在每次调用 `self.toppings.all()` 时，都不需要访问数据库中的项目，它会在预填的 `QuerySet` 缓存中找到它们，并将其填充到单个查询中。

也就是说，所有相关的配料都将在单个查询中提取，并用于使 `QuerySet` 具有预填充相关结果的缓存;这些查询集然后在 `self.toppings.all()` 调用中使用。

`prefetch_related()` 中的附加查询在 `QuerySet` 已开始评估并且主查询已执行后执行。

如果有一个可迭代的模型实例，则可以使用 `prefetch_related_objects()` 函数在这些实例上预取相关属性。

请注意，主 `QuerySet` 的结果缓存和所有指定的相关对象将被完全加载到内存中。这改变了 `QuerySets` 的典型行为，即使在数据库中执行查询之后，它通常会尽量避免在需要之前将所有对象加载到内存中。

请记住，与 `QuerySet` 一样，任何隐含的不同数据库查询的后续链接方法都将忽略先前缓存的结果，并使用新的数据库查询检索数据。所以，如果你写下以下内容：

``` python
>>> pizzas = Pizza.objects.prefetch_related('toppings')
>>> [list(pizza.toppings.filter(spicy=True)) for pizza in pizzas]
```

...那么事实上 `pizza.toppings.all()` 已被预取不会对你有帮助。`prefetch_related('toppings')` 意味着 `pizza.toppings.all()`，但 `pizza.toppings.filter()` 是一个新的不同的查询。预取缓存在这里无法帮助;事实上，它损害了性能，因为你已经完成了一个你没有使用过的数据库查询。所以请谨慎使用此功能！

另外，如果在相关管理器上调用数据库更改方法 add(), remove(), clear() 或 set()，则 related 的任何预取缓存都将被清除。

您还可以使用常规链式语法来执行相关字段的相关字段。假设我们在上面的例子中有一个额外的模型：

``` python
class Restaurant(models.Model):
    pizzas = models.ManyToManyField(Pizza, related_name='restaurants')
    best_pizza = models.ForeignKey(Pizza, related_name='championed_by', on_delete=models.CASCADE)
```

以下都是合法的：

``` python
>>> Restaurant.objects.prefetch_related('pizzas__toppings')
```

这将预取所有属于 `餐馆` 的 `比萨饼` 以及属于这些比萨饼的所有 `配料`。这将导致总共 `3` 个数据库查询 - 一个用于 `餐馆` ，一个用于 `比萨饼`，另一个用于 `配料`。

``` python
>>> Restaurant.objects.prefetch_related('best_pizza__toppings')
```

这将为每家 `餐厅` 提供最好的 `比萨饼` 和所有最好的 `披萨配料`。这将在 3 个数据库查询中完成 - 一个用于 `餐厅`，一个用于 `“最好的披萨”`，另一个 `配料`。

当然，也可以使用 `select_related` 来获取 `best_pizza` 关系，以将查询计数减少到 2：

``` python
>>> Restaurant.objects.select_related('best_pizza').prefetch_related('best_pizza__toppings')
```

由于预取是在主查询（其中包括 `select_related` 所需的连接）之后执行的，因此它能够检测到 `best_pizza` 对象已被提取，并且会跳过再次提取它们。

链式 `prefetch_related` 调用将累积预取的查找。要清除任何 `prefetch_related` 行为，请将 `None` 作为参数传递：

``` python
>>> non_prefetched = qs.prefetch_related(None)
```

此处省略一万字......

### extra()

`extra(select=None, where=None, params=None, tables=None, order_by=None, select_params=None)`

有时候，Django 查询语法本身并不能很容易地表达一个复杂的 `WHERE` 子句。对于这些边缘情况，Django 提供了 `extra() QuerySet` 修饰符 - 一个用于将特定子句注入由 `QuerySet` 生成的 SQL 中的钩子。

!> 旧 API，以后可能会弃用，省略一万字......

### defer()

`defer(*fields)`

在一些复杂的数据建模情况下，您的模型可能包含很多字段，其中一些可能包含大量数据（例如文本字段），或者需要昂贵的处理将其转换为 Python 对象。如果您在最初获取数据时不知道是否需要这些特定字段的情况下使用查询集的结果，那么可以告诉 Django 不要从数据库中检索它们。

这是通过传递字段的名称来不加载到 `defer()` 来完成的：

``` python
Entry.objects.defer("headline", "body")
```

具有延迟字段的查询集仍将返回模型实例。如果您访问该字段（一次一个，而不是所有延迟字段一次），则将从数据库中检索每个延迟字段。

您可以进行多次调用 `defer().`。每次调用都会将新字段添加到延期集合中：

``` python
# Defers both the body and headline fields.
Entry.objects.defer("body").filter(rating=5).defer("headline")
```

字段添加到延期集合的顺序无关紧要。使用已经延期的字段名称调用 `defer()` 是无害的（该字段仍将被延迟）。

您可以推迟使用标准双下划线表示法来加载相关模型中的字段（如果相关模型是通过 `select_related()` 加载的）来分隔相关字段：

``` python
Blog.objects.select_related().defer("entry__headline", "entry__body")
```

如果要清除延迟字段集，请将 `None` 作为参数传递给 `defer()`：

``` python
# Load all fields immediately.
my_queryset.defer(None)
```

即使您要求，模型中的某些字段也不会被延期。你永远不能推迟加载主键。如果您使用 `select_related()` 来检索相关模型，则不应延迟从主模型连接到相关模型的字段的加载，否则会导致错误。


!> 坑多，慎用，省略一万字......

### only()

`only(*fields)`

`only()` 方法或多或少与 `defer()` 相反。您可以在检索模型时使用不应推迟的字段进行调用。如果您有一个几乎需要延迟所有字段的模型，则使用 `only()` 指定补充字段集可以生成更简单的代码。

假设你有一个带有字段 `name`, `age` 和 `biography` 的模型。根据延迟字段，以下两个查询集是相同的：

``` python
Person.objects.defer("age", "biography")
Person.objects.only("name")
```

无论何时调用 `only()`，它都会替换要立即加载的一组字段。该方法的名称是助记符的：`only` 那些字段被立即加载;其余部分被推迟。因此，`only()` 的连续调用才会导致只考虑最终的字段：

``` python
# This will defer all fields except the headline.
Entry.objects.only("body", "rating").only("headline")
```

由于 `defer()` 递增（将字段添加到延迟列表中），因此可以将调用组合为 `only()` 和 `defer()`，并且事物将按逻辑运行：

``` python
# Final result is that everything except "headline" is deferred.
Entry.objects.only("headline", "body").defer("body")

# Final result loads headline and body immediately (only() replaces any
# existing set of fields).
Entry.objects.defer("body").only("headline", "body")
```

### using()

`using(alias)`

如果您使用多个数据库，则此方法用于控制 `QuerySet` 将针对哪个数据库进行评估。此方法唯一的参数是数据库的别名，如 `DATABASES` 中所定义。

例如：

``` python
# queries the database with the 'default' alias.
>>> Entry.objects.all()

# queries the database with the 'backup' alias
>>> Entry.objects.using('backup')
```

### select_for_update()

`select_for_update(nowait=False, skip_locked=False, of=())`

返回将锁定行直到事务结束的查询集，在受支持的数据库上生成 `SELECT ... FOR UPDATE` SQL 语句。

例如：

``` python
entries = Entry.objects.select_for_update().filter(author=request.user)
```

省略一万字......

### raw()

`raw(raw_query, params=None, translations=None)`

接受原始 SQL 查询，执行该查询，并返回一个 `django.db.models.query.RawQuerySet` 实例。这个RawQuerySet实例可以像正常的QuerySet一样迭代以提供对象实例。

## 不返回 QuerySet 的方法

这些方法不使用缓存。相反，他们每次调用数据时都会查询数据库。

### get()

`get(**kwargs)`

返回匹配给定查找参数的对象

如果找到多个对象，`get()` 会引发 `MultipleObjectsReturned`。 `MultipleObjectsReturned` 异常是模型类的一个属性。

如果没有找到给定参数的对象，`get()` 会引发一个 `DoesNotExist` 异常。这个异常是模型类的一个属性。例：

``` python
Entry.objects.get(id='foo') # raises Entry.DoesNotExist
```

`DoesNotExist` 异常继承自 `django.core.exceptions.ObjectDoesNotExist`，因此您可以定位多个 `DoesNotExist` 异常。例：

``` python
from django.core.exceptions import ObjectDoesNotExist
try:
    e = Entry.objects.get(id=3)
    b = Blog.objects.get(id=1)
except ObjectDoesNotExist:
    print("Either the entry or blog doesn't exist.")
```

如果您希望查询集返回一行，则可以使用不带任何参数的 `get()` 来返回该行的对象：

``` python
entry = Entry.objects.filter(...).exclude(...).get()
```

### create()

`create(**kwargs)`

创建对象并将其全部保存在一个步骤中的便捷方法。从而：

``` python
p = Person.objects.create(first_name="Bruce", last_name="Springsteen")
```

和：

``` python
p = Person(first_name="Bruce", last_name="Springsteen")
p.save(force_insert=True)
```

是等同的。

`force_insert=True` 意味着总是会创建一个新对象。通常你不需要担心这一点。但是，如果您的模型包含您设置的手动主键值，并且该值已存在于数据库中，则由于主键必须是唯一的，因此对 `create()` 的调用将因 `IntegrityError` 失败。如果您使用手动主键，请准备好处理异常。

### get_or_create()

`get_or_create(defaults=None, **kwargs)`

一种用给定的 `kwargs` 查找对象的简便方法（如果您的模型具有所有字段的默认值，则可能为空），如果需要则创建一个。

返回 `(object, created)` 的元组，其中 `object` 是检索或创建的对象，`created` 是一个布尔值，指定是否创建新对象。

这是代码的一种捷径。例如：

``` python
try:
    obj = Person.objects.get(first_name='John', last_name='Lennon')
except Person.DoesNotExist:
    obj = Person(first_name='John', last_name='Lennon', birthday=date(1940, 10, 9))
    obj.save()
```

随着模型中字段数量的增加，这种模式变得相当不便。上面的例子可以使用 `get_or_create()` 来重写，如下所示：

``` python
obj, created = Person.objects.get_or_create(
    first_name='John',
    last_name='Lennon',
    defaults={'birthday': date(1940, 10, 9)},
)
```

传递给 `get_or_create()` 的任何关键字参数（除了可选的 `defaults`）都将用于 `get()` 调用。如果找到一个对象，`get_or_create()` 将返回一个元组 `(找到的对象, False)` 。如果找到多个对象，则 `get_or_create` 会引发 `MultipleObjectsReturned`。如果找不到对象， `get_or_create()` 将实例化并保存一个新对象，并返回`(新对象, True)`。新对象将大致根据此算法创建：

``` python
params = {k: v for k, v in kwargs.items() if '__' not in k}
params.update({k: v() if callable(v) else v for k, v in defaults.items()})
obj = self.model(**params)
obj.save()
```

此处省略一万字......

### update_or_create()

`update_or_create(defaults=None, **kwargs)`

找到对象就更新，否则就创建。

`update_or_create` 方法尝试从基于给定 `kwargs` 的数据库中获取对象。如果找到匹配项，它会更新 `defaults` 字典中传递的字段。

这是一种捷径。例如：

``` python
defaults = {'first_name': 'Bob'}
try:
    obj = Person.objects.get(first_name='John', last_name='Lennon')
    for key, value in defaults.items():
        setattr(obj, key, value)
    obj.save()
except Person.DoesNotExist:
    new_values = {'first_name': 'John', 'last_name': 'Lennon'}
    new_values.update(defaults)
    obj = Person(**new_values)
    obj.save()
```

可以换成：

``` python
obj, created = Person.objects.update_or_create(
    first_name='John', last_name='Lennon',
    defaults={'first_name': 'Bob'},
)
```

### bulk_create()

`bulk_create(objs, batch_size=None)`

此方法以高效的方式将提供的对象列表插入数据库（通常只有1个查询，不管有多少个对象）：

``` python
>>> Entry.objects.bulk_create([
...     Entry(headline='This is a test'),
...     Entry(headline='This is only a test'),
... ])
```

尽管如此，还是有一些注意事项：

* 模型的 `save()` 方法不会被调用，`pre_save` 和 `post_save` 信号将不会被发送。
* 它不适用于多表继承方案中的子模型。
* 如果模型的主键是一个 `AutoField`，它不会像 `save()` 那样检索和设置主键属性，除非数据库后端支持它（当前是 PostgreSQL）。
* 它不适用于多对多的关系。
* 它将 `objs` 转换为一个列表，如果它是一个生成器，它将完全评估 `objs`。该转换允许检查所有对象，以便可以首先插入具有手动设置的主键的任何对象。如果您想要批量插入对象而不一次评估整个生成器，只要对象没有任何手动设置的主键，就可以使用此技术：
    ``` python
    from itertools import islice

    batch_size = 100
    objs = (Entry(headline='Test %s' % i) for i in range(1000))
    while True:
        batch = list(islice(objs, batch_size))
        if not batch:
            break
        Entry.objects.bulk_create(batch, batch_size)
    ```

`batch_size` 参数控制在单个查询中创建多少个对象。缺省情况是在一个批处理中创建所有对象，SQLite 除外，默认情况下，每个查询最多使用 999 个变量。

### count()

返回一个整数，表示与 `QuerySet` 匹配的数据库中的对象数。 `count()` 方法从不引发异常。

例：

``` python
# Returns the total number of entries in the database.
Entry.objects.count()

# Returns the number of entries whose headline contains 'Lennon'
Entry.objects.filter(headline__contains='Lennon').count()
```

`count()` 调用在后台执行 `SELECT COUNT(*)`，所以您应该始终使用 `count()`，而不是将所有记录加载到 Python 对象中，并对结果调用 `len()`（除非你需要将对象加载到内存中，在这种情况下 `len()` 会更快）。

请注意，如果您需要 `QuerySet` 中的项目数并且还从中检索模型实例（例如，通过遍历它），使用 `len(queryset)` 可能会更有效率，它不会像 `count()` 那样引起额外的数据库查询。

### in_bulk()

`in_bulk(id_list=None, field_name='pk')`

为这些值获取字段值列表 `(id_list) ` 和 `field_name`，并返回一个字典，将每个值映射到具有给定字段值的对象实例。如果未提供 `id_list`，则返回查询集中的所有对象。 `field_name` 必须是唯一的字段，并且它默认为主键。

例：

``` python
>>> Blog.objects.in_bulk([1])
{1: <Blog: Beatles Blog>}
>>> Blog.objects.in_bulk([1, 2])
{1: <Blog: Beatles Blog>, 2: <Blog: Cheddar Talk>}
>>> Blog.objects.in_bulk([])
{}
>>> Blog.objects.in_bulk()
{1: <Blog: Beatles Blog>, 2: <Blog: Cheddar Talk>, 3: <Blog: Django Weblog>}
>>> Blog.objects.in_bulk(['beatles_blog'], field_name='slug')
{'beatles_blog': <Blog: Beatles Blog>}
```

### iterator()

`iterator(chunk_size=2000)`

此处省略一万字

### latest()

`latest(*fields)`

根据给定 `field(s)` 返回表中的最新对象。

根据 `pub_date` 字段，此示例返回表中的最新 `Entry`：

``` python
Entry.objects.latest('pub_date')
```

您也可以根据几个字段选择最新的。例如，要在两个 entry 具有相同的 `pub_date` 时选择具有最早 `expire_date` 的 `Entry`：

``` python
Entry.objects.latest('pub_date', '-expire_date')
```
### earliest()

`earliest(*fields)`

除了方向改变之外，其他方面像 `latest()` 一样工作。

### first()

返回查询集匹配的第一个对象，如果没有匹配的对象，则返回 `None`。如果 `QuerySet` 没有定义的顺序，那么查询集由主键自动排序。

``` python
p = Article.objects.order_by('title', 'pub_date').first()
```

请注意，`first()` 是一个方便的方法，下面的代码示例等同于上面的示例：

``` python
try:
    p = Article.objects.order_by('title', 'pub_date')[0]
except IndexError:
    p = None
```

### last()

像 `first()` 一样工作，但返回查询集中的最后一个对象。

### aggregate()

`aggregate(*args, **kwargs)`

返回通过 `QuerySet` 计算的聚合值（平均值，总和等）的字典。 `aggregate()` 的每个参数指定一个将包含在返回的字典中的值。

Django 提供的聚合函数在下面的聚合函数中进行了描述 。由于聚合也是查询表达式，因此您可以将聚合与其他聚合或值组合以创建复杂聚合。

使用关键字参数指定的聚合将使用关键字作为注释的名称。匿名参数将根据聚合函数的名称和正在聚合的模型字段为它们生成一个名称。复杂的聚合不能使用匿名参数，而必须指定一个关键字参数作为别名。

例如，当您使用博客条目时，您可能想知道贡献了博客条目的作者数量：

``` python
>>> from django.db.models import Count
>>> q = Blog.objects.aggregate(Count('entry'))
{'entry__count': 16}
```

通过使用关键字参数指定聚合函数，可以控制返回的聚合值的名称：

``` python
>>> q = Blog.objects.aggregate(number_of_entries=Count('entry'))
{'number_of_entries': 16}
```

### exists()

如果 `QuerySet` 包含任何结果，则返回 `True`，否则返回 `False`。这尽可能以最简单和最快的方式执行查询，但它确实执行与普通 `QuerySet` 查询几乎相同的查询。

`exists()` 对于与 `QuerySet` 中的对象成员资格以及 `QuerySet` 中的任何对象（特别是大型 `QuerySet` 的上下文）中存在的相关搜索都很有用。

查找具有唯一字段（例如 `primary_key`）的模型是否为 `QuerySet` 的成员的最有效方法是：

``` python
entry = Entry.objects.get(pk=123)
if some_queryset.filter(pk=entry.pk).exists():
    print("Entry contained in queryset")
```

这将比以下要求更快，需要对整个查询集进行评估和迭代：

```
if entry in some_queryset:
   print("Entry contained in QuerySet")
```

查找查询集是否包含任何项目：

``` python
if some_queryset.exists():
    print("There is at least one object in some_queryset")
```

这将比以下更快：

``` python
if some_queryset:
    print("There is at least one object in some_queryset")
```

...但不是很大程度上（因此需要大量查询来提高效率）。

### update()

`update(**kwargs)`

对指定的字段执行SQL更新查询，并返回匹配的行数（如果某些行已具有新值，则可能不等于更新的行数）。

例如，要关闭 2010 年发布的所有博客条目的评论，您可以这样做：

``` python
>>> Entry.objects.filter(pub_date__year=2010).update(comments_on=False)
```

（这假设您的 `Entry` 模型具有 `pub_date` 和 `comments_on` 字段。）

您可以更新多个字段 - 对多少字段没有限制。例如，在这里我们更新 `comments_on`和 `headline` 字段：

``` python
>>> Entry.objects.filter(pub_date__year=2010).update(comments_on=False, headline='This is old')
```

`update()` 方法立即应用，并且更新后的 `QuerySet` 的唯一限制是它只能更新模型主表中的列，而不能更新相关模型中的列。你不能这样做，例如：

``` python
>>> Entry.objects.update(blog__name='foo') # Won't work!
```

虽然基于相关领域的过滤仍然是可能的：

``` python
>>> Entry.objects.filter(blog__id=1).update(comments_on=True)
```

您无法在已经创建切片或不能再被过滤的 `QuerySet` 上调用 `update()`。

`update()` 方法返回受影响的行数：

``` python
>>> Entry.objects.filter(id=64).update(comments_on=True)
1

>>> Entry.objects.filter(slug='nonexistent-slug').update(comments_on=True)
0

>>> Entry.objects.filter(pub_date__year=2010).update(comments_on=False)
132
```

如果你只是更新记录而不需要对模型对象做任何事情，最有效的方法是调用 `update()`，而不是将模型对象加载到内存中。例如，而不是这样做：

``` python
e = Entry.objects.get(id=10)
e.comments_on = False
e.save()
```

而是这样做

``` python
Entry.objects.filter(id=10).update(comments_on=False)
```

使用 `update()` 还可以防止在加载对象和调用 `save()` 之间的短时间内数据库中可能发生变化的争用情况。

最后，认识到 `update()` 在 SQL 级别执行更新，因此不会在模型上调用任何 `save()` 方法，也不会发出 `pre_save` 或 `post_save` 信号（这是调用 `Model.save()` 的结果）。如果您想更新具有自定义 `save()` 方法的模型的一堆记录，请遍历它们并调用 `save()`，如下所示：

``` python
for e in Entry.objects.filter(pub_date__year=2010):
    e.comments_on = False
    e.save()
```

### delete()

对 `QuerySet` 中的所有行执行 `SQL` 删除查询，并返回删除的对象数和每个对象类型具有删除次数的字典。

`delete()` 立即应用。您无法在已经创建切片或无法再过滤的 `QuerySet` 上调用 `delete()`。

例如，要删除特定博客中的所有条目：

``` python
>>> b = Blog.objects.get(pk=1)

# Delete all the entries belonging to this Blog.
>>> Entry.objects.filter(blog=b).delete()
(4, {'weblog.Entry': 2, 'weblog.Entry_authors': 2})
```

默认情况下，Django 的 `ForeignKey` 模拟 `ON DELETE CASCADE` 的 SQL 约束 - 换句话说，任何外键指向要删除的对象的对象都将被删除。例如：

``` python
>>> blogs = Blog.objects.all()

# This will delete all Blogs and all of their Entry objects.
>>> blogs.delete()
(5, {'weblog.Blog': 1, 'weblog.Entry': 2, 'weblog.Entry_authors': 2})
```

此级联行为可通过 `ForeignKey` 的 `on_delete` 参数进行自定义。

### as_manager()

省略几十字......

## 字段查找

字段查找是您如何指定 `SQL WHERE` 子句的内容。它们被指定为 `QuerySet` 方法 `filter()`，`exclude()` 和 `get()` 的关键字参数。

下面列出了 Django 的内置查找。也可以为模型字段编写自定义查找。

### exact

完全匹配。如果提供的用于比较的值为 `None`，则它将被解释为 SQL NULL。

``` python
Entry.objects.get(id__exact=14)
Entry.objects.get(id__exact=None)
```

相当于：

``` sql
SELECT ... WHERE id = 14;
SELECT ... WHERE id IS NULL;
```

### iexact

不区分大小写的精确匹配。如果为比较提供的值为 `None`，则它将被解释为SQL NULL

``` python
Blog.objects.get(name__iexact='beatles blog')
Blog.objects.get(name__iexact=None)
```

等价于：

``` sql
SELECT ... WHERE name ILIKE 'beatles blog';
SELECT ... WHERE name IS NULL;
```

### contains

区分大小写的内容匹配。

``` python
Entry.objects.get(headline__contains='Lennon')
```

等价于：

``` sql
SELECT ... WHERE headline LIKE '%Lennon%';
```

### icontains

不区分大小写的内容匹配。

``` python
Entry.objects.get(headline__icontains='Lennon')
```

等价于：

``` sql
SELECT ... WHERE headline ILIKE '%Lennon%';
```

### in

``` python
Entry.objects.filter(id__in=[1, 3, 4])
```

等价于：

``` sql
SELECT ... WHERE id IN (1, 3, 4);
```

您还可以使用查询集动态评估值列表，而不是提供文字值列表：

``` python
inner_qs = Blog.objects.filter(name__contains='Cheddar')
entries = Entry.objects.filter(blog__in=inner_qs)
```

类似于：

``` sql
SELECT ... WHERE blog.id IN (SELECT id FROM ... WHERE NAME LIKE '%Cheddar%')
```

还可以这样：

``` python
inner_qs = Blog.objects.filter(name__contains='Ch').values('name')
entries = Entry.objects.filter(blog__name__in=inner_qs)
```

但请不要这样：

``` python
# Bad code! Will raise a TypeError.
inner_qs = Blog.objects.filter(name__contains='Ch').values('name', 'id')
entries = Entry.objects.filter(blog__name__in=inner_qs)
```

### gt

大于

``` python
Entry.objects.filter(id__gt=4)
```

等价于：

``` sql
SELECT ... WHERE id > 4;
```

### gte

大于等于

### lt

小于

### lte

小于等于

### startswith

从开始部分匹配，区分大小写

``` python
Entry.objects.filter(headline__startswith='Lennon')
```

等价于；

``` python
SELECT ... WHERE headline LIKE 'Lennon%';
```

### istartswith

从开始部分匹配，不区分大小写

``` python
Entry.objects.filter(headline__istartswith='Lennon')
```

等价于；

``` python
SELECT ... WHERE headline ILIKE 'Lennon%';
```

### endswith

从结尾部分匹配，区分大小写

``` python
Entry.objects.filter(headline__endswith='Lennon')
```

等价于；

``` python
SELECT ... WHERE headline LIKE '%Lennon';
```

### iendswith

从结尾部分匹配，不区分大小写

``` python
Entry.objects.filter(headline__iendswith='Lennon')
```

等价于；

``` python
SELECT ... WHERE headline ILIKE '%Lennon'
```

### range

范围匹配

``` python
import datetime
start_date = datetime.date(2005, 1, 1)
end_date = datetime.date(2005, 3, 31)
Entry.objects.filter(pub_date__range=(start_date, end_date))
```

等价于：

``` sql
SELECT ... WHERE pub_date BETWEEN '2005-01-01' and '2005-03-31';
```

### date

日期匹配

``` python
Entry.objects.filter(pub_date__date=datetime.date(2005, 1, 1))
Entry.objects.filter(pub_date__date__gt=datetime.date(2005, 1, 1))
```

### year

年份匹配

``` python
Entry.objects.filter(pub_date__year=2005)
Entry.objects.filter(pub_date__year__gte=2005)
```

等价于：

``` sql
SELECT ... WHERE pub_date BETWEEN '2005-01-01' AND '2005-12-31';
SELECT ... WHERE pub_date >= '2005-01-01';
```

### month

月份匹配

``` python
Entry.objects.filter(pub_date__month=12)
Entry.objects.filter(pub_date__month__gte=6)
```

等价于：

``` sql
SELECT ... WHERE EXTRACT('month' FROM pub_date) = '12';
SELECT ... WHERE EXTRACT('month' FROM pub_date) >= '6';
```

### day

每月第几天匹配

```
Entry.objects.filter(pub_date__day=3)
Entry.objects.filter(pub_date__day__gte=3)
```

等价于：

``` sql
SELECT ... WHERE EXTRACT('day' FROM pub_date) = '3';
SELECT ... WHERE EXTRACT('day' FROM pub_date) >= '3';
```


### week

第几周匹配

``` python
Entry.objects.filter(pub_date__week=52)
Entry.objects.filter(pub_date__week__gte=32, pub_date__week__lte=38)
```

### week_day

星期几匹配

1 表示星期日，不是星期一

``` python
Entry.objects.filter(pub_date__week_day=2)
Entry.objects.filter(pub_date__week_day__gte=2)
```


### quarter

季度匹配

第二季度（4月1日至6月30日）检索条目的示例：

``` python
Entry.objects.filter(pub_date__quarter=2)
```

### time

时间匹配

``` python
Entry.objects.filter(pub_date__time=datetime.time(14, 30))
Entry.objects.filter(pub_date__time__range=(datetime.time(8), datetime.time(17)))
```

### hour

小时匹配，采用 0 到 23 之间的整数。

``` python
Event.objects.filter(timestamp__hour=23)
Event.objects.filter(time__hour=5)
Event.objects.filter(timestamp__hour__gte=12)
```

等价于：

``` sql
SELECT ... WHERE EXTRACT('hour' FROM timestamp) = '23';
SELECT ... WHERE EXTRACT('hour' FROM time) = '5';
SELECT ... WHERE EXTRACT('hour' FROM timestamp) >= '12';
```

### minute

分钟匹配，采用 0 到 59 之间的整数。

``` python
Event.objects.filter(timestamp__minute=29)
Event.objects.filter(time__minute=46)
Event.objects.filter(timestamp__minute__gte=29)
```

等价于：

``` sql
SELECT ... WHERE EXTRACT('minute' FROM timestamp) = '29';
SELECT ... WHERE EXTRACT('minute' FROM time) = '46';
SELECT ... WHERE EXTRACT('minute' FROM timestamp) >= '29';
```

### second

秒数匹配，采用0到59之间的整数。

``` python
Event.objects.filter(timestamp__second=31)
Event.objects.filter(time__second=2)
Event.objects.filter(timestamp__second__gte=31)
```

等价于：

``` sql
SELECT ... WHERE EXTRACT('second' FROM timestamp) = '31';
SELECT ... WHERE EXTRACT('second' FROM time) = '2';
SELECT ... WHERE EXTRACT('second' FROM timestamp) >= '31';
```

### isnull

``` python
Entry.objects.filter(pub_date__isnull=True)
```

等价于：

``` sql
SELECT ... WHERE pub_date IS NULL;
```

### regex

正则匹配

``` python
Entry.objects.get(title__regex=r'^(An?|The) +')
```

等价于：

``` sql
SELECT ... WHERE title REGEXP BINARY '^(An?|The) +'; -- MySQL

SELECT ... WHERE REGEXP_LIKE(title, '^(An?|The) +', 'c'); -- Oracle

SELECT ... WHERE title ~ '^(An?|The) +'; -- PostgreSQL

SELECT ... WHERE title REGEXP '^(An?|The) +'; -- SQLite
```

### iregex

不区分大小写的正则表达式匹配。

``` python
Entry.objects.get(title__iregex=r'^(an?|the) +')
```

等价于：

``` sql
SELECT ... WHERE title REGEXP '^(an?|the) +'; -- MySQL

SELECT ... WHERE REGEXP_LIKE(title, '^(an?|the) +', 'i'); -- Oracle

SELECT ... WHERE title ~* '^(an?|the) +'; -- PostgreSQL

SELECT ... WHERE title REGEXP '(?i)^(an?|the) +'; -- SQLite
```

## 聚合函数

Django 在 `django.db.models` 模块中提供了以下聚合函数。有关如何使用这些聚合函数的详细信息，请参阅 [关于聚合的主题指南](https://docs.djangoproject.com/zh-hans/2.0/topics/db/aggregation/)。

expression，
output_field，
filter，
**extra，
Avg，
Count，
Max，
Min，
StdDev，
Sum，
Variance，

# 查询相关工具

## Q() 对象

`Q()` 对象与 `F` 对象一样，将 SQL 表达式封装在可用于数据库相关操作的 Python 对象中。

通常，`Q()` 对象可以定义和重用条件。这允许使用 `| (OR)` 和 `& (AND)` 操作符构建复杂的数据库查询;尤其是，在 `QuerySet` 中不能使用 `OR`。

## Prefetch() 对象

`class Prefetch(lookup, queryset=None, to_attr=None)`

`Prefetch()` 对象可用于控制 `prefetch_related()` 的操作。

`lookup` 参数描述了遵循的关系，并且与传递给 `prefetch_related()` 的基于字符串的查找相同。例如：

``` python
>>> from django.db.models import Prefetch
>>> Question.objects.prefetch_related(Prefetch('choice_set')).get().choice_set.all()
<QuerySet [<Choice: Not much>, <Choice: The sky>, <Choice: Just hacking again>]>
# This will only execute two queries regardless of the number of Question
# and Choice objects.
>>> Question.objects.prefetch_related(Prefetch('choice_set')).all()
<QuerySet [<Question: What's up?>]>
```

`queryset` 参数为给定的查找提供基本的 `QuerySet`。这对进一步过滤预取操作或从预取关系中调用 `select_related()` 很有用，因此可以进一步减少查询数量：

``` python
>>> voted_choices = Choice.objects.filter(votes__gt=0)
>>> voted_choices
<QuerySet [<Choice: The sky>]>
>>> prefetch = Prefetch('choice_set', queryset=voted_choices)
>>> Question.objects.prefetch_related(prefetch).get().choice_set.all()
<QuerySet [<Choice: The sky>]>
```

`to_attr` 参数将预取操作的结果设置为自定义属性：

``` python
>>> prefetch = Prefetch('choice_set', queryset=voted_choices, to_attr='voted_choices')
>>> Question.objects.prefetch_related(prefetch).get().voted_choices
<QuerySet [<Choice: The sky>]>
>>> Question.objects.prefetch_related(prefetch).get().choice_set.all()
<QuerySet [<Choice: Not much>, <Choice: The sky>, <Choice: Just hacking again>]>
```

## prefetch_related_objects()

## FilteredRelation() 对象