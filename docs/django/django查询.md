# 查询操作

一旦创建了数据模型，Django 就会自动为您提供一个数据库抽象 API，使您可以创建，检索，更新和删除对象。本文档介绍了如何使用此 API。


在本指南（和参考文献）中，我们将参考以下模型，它们构成了一个 Weblog 应用程序：

``` python
from django.db import models

class Blog(models.Model):
    name = models.CharField(max_length=100)
    tagline = models.TextField()

    def __str__(self):
        return self.name

class Author(models.Model):
    name = models.CharField(max_length=200)
    email = models.EmailField()

    def __str__(self):
        return self.name

class Entry(models.Model):
    blog = models.ForeignKey(Blog, on_delete=models.CASCADE)
    headline = models.CharField(max_length=255)
    body_text = models.TextField()
    pub_date = models.DateField()
    mod_date = models.DateField()
    authors = models.ManyToManyField(Author)
    n_comments = models.IntegerField()
    n_pingbacks = models.IntegerField()
    rating = models.IntegerField()

    def __str__(self):
        return self.headline
```

## 创建对象

为了在 Python 对象中表示数据库表数据，Django 使用直观的系统：模型类表示数据库表，该类的实例表示数据库表中的特定记录。

要创建一个对象，请使用模型类的关键字参数对其进行实例化，然后调用 `save()` 以将其保存到数据库。

假设模型存在于文件 `mysite/blog/models.py` 中，下面是一个例子：

``` python
>>> from blog.models import Blog
>>> b = Blog(name='Beatles Blog', tagline='All the latest Beatles news.')
>>> b.save()
```

幕后执行 SQL 语句 INSERT 。 在你没有调用 `save()` 方法之前， Django 不会将数据保存到数据库

`save()` 方法没有返回值。

## 保存对对象的更改

要保存对已存在于数据库中的对象的更改，请使用 `save()`。

给定一个已经保存到数据库的 `Blog` 实例 `b5`，这个例子改变它的名字并更新数据库中的记录：

``` python
>>> b5.name = 'New name'
>>> b5.save()
```

幕后执行 SQL 语句 UPDATE 。 在你没有调用 `save()` 方法之前， Django 不会将数据保存到数据库

### 保存 ForeignKey 和 ManyToManyField 字段

更新 ForeignKey 字段的方式与保存普通字段的方式完全相同 -- 只需将正确类型的对象分配给相关字段即可。

``` python
>>> from blog.models import Blog, Entry
>>> entry = Entry.objects.get(pk=1)
>>> cheese_blog = Blog.objects.get(name="Cheddar Talk")
>>> entry.blog = cheese_blog
>>> entry.save()
```

还可以通过 `add()` 方法：

``` python
>>> from blog.models import Author
>>> joe = Author.objects.create(name="Joe")
>>> entry.authors.add(joe)
```
要一次添加多个记录：

``` python
>>> john = Author.objects.create(name="John")
>>> paul = Author.objects.create(name="Paul")
>>> george = Author.objects.create(name="George")
>>> ringo = Author.objects.create(name="Ringo")
>>> entry.authors.add(john, paul, george, ringo)
```

## 检索对象

要从数据库中检索对象，请在您的模型类上通过 `Manager` 构建一个 `QuerySet`。

`QuerySet` 表示数据库中的对象集合。它可以有零个，一个或多个过滤器。过滤器根据给定的参数缩小查询结果的范围。在 SQL 术语中，`QuerySet` 等同于 SELECT 语句，过滤器是限制性子句，如 WHERE 或 LIMIT。

您可以使用模型的 `Manager` 获取 `QuerySet`。每个模型至少有一个 `Manager`，默认情况下是 `objects`。通过模型类直接访问它，如下所示：

``` python
>>> Blog.objects
<django.db.models.manager.Manager object at ...>
>>> b = Blog(name='Foo', tagline='Bar')
>>> b.objects
Traceback:
    ...
AttributeError: "Manager isn't accessible via Blog instances."
```
!> Managers 只能通过模型​​类访问，而不能从模型实例访问。

`Manager` 是模型的 `QuerySet` 的主要来源。例如，`Blog.objects.all()` 返回包含数据库中所有 `Blog` 对象的 `QuerySet`。

### 检索所有对象

从表中检索对象的最简单方法是获取所有对象。为此，请使用 `Manager` 上的 `all()` 方法：

``` python
>>> all_entries = Entry.objects.all()
```

`all()` 方法返回数据库中所有对象的 `QuerySet`。

### 使用过滤器检索特定的对象

`filter(**kwargs)`

返回包含匹配给定查找参数的对象的新 `QuerySet`。

`exclude(**kwargs)`

返回包含与给定查找参数不匹配的对象的新 `QuerySet`。

例如，要获取从 2006 年开始的博客条目的 `QuerySet`，请使用 `filter()`，如下所示：

``` python
Entry.objects.filter(pub_date__year=2006)
```

使用默认 manager 类，它与以下内容相同：

``` python
Entry.objects.all().filter(pub_date__year=2006)
```

#### 链式过滤器

改进 `QuerySet` 的结果本身就是一个 `QuerySet`，因此可以将改进链接在一起。例如：

``` python
>>> Entry.objects.filter(
...     headline__startswith='What'
... ).exclude(
...     pub_date__gte=datetime.date.today()
... ).filter(
...     pub_date__gte=datetime.date(2005, 1, 30)
... )
```

#### 过滤后的 `QuerySet` 是独一无二的

每次您完善 `QuerySet` 时，都会获得全新的 `QuerySet`，它绝不会绑定到以前的 `QuerySet`。每个优化都会创建一个独立且不同的 `QuerySet` ，可以存储，使用和重用。

举例：

``` python
>>> q1 = Entry.objects.filter(headline__startswith="What")
>>> q2 = q1.exclude(pub_date__gte=datetime.date.today())
>>> q3 = q1.filter(pub_date__gte=datetime.date.today())
```

#### QuerySet 会延迟查询(lazy)

`QuerySets` 是懒惰的 -- 创建 `QuerySet` 的行为不涉及任何数据库活动。您可以将过滤器堆叠在一起，并且在使用 `QuerySet` 之前，Django 不会真正运行查询。看看这个例子：

``` python
>>> q = Entry.objects.filter(headline__startswith="What")
>>> q = q.filter(pub_date__lte=datetime.date.today())
>>> q = q.exclude(body_text__icontains="food")
>>> print(q)
```

看起来像是操作了三次数据库，实际上只有在执行 `print(p)` 时才会真正执行一次数据库查询操作。

### 用 get() 检索单个对象

`filter()` 总是返回一个 `QuerySet`，即使匹配的对象只有一个。

如果你知道只有一个对象和查询匹配，则可以直接使用 `get()` 方法返回单个对象：

``` python
>>> one_entry = Entry.objects.get(pk=1)
```

!> 请注意，`get()` 方法如果没有查找到对象，将引发 `DoesNotExist` 异常。查询到多个对象则会引发 `MultipleObjectReturned` 异常，这两个异常都是模型类的一个属性。

### 截取 QuerySet

可以使用 Python 中的切片语法对 `QuerySet` 的数量进行限制。这相当于 SQL LIMIT 和 OFFSET 子句。

例如，返回前 5 个对象（`LIMIT 5`）。

``` python
>>> Entry.objects.all()[:5]
```

返回第六到第十个对象（`OFFSET 5 LIMIT 5`）。

``` python
>>> Entry.objects.all()[5:10]
```

不支持负数索引（例如 `Entry.objects.all()[-1]`）。

通常切片一个 `QuerySet` 会返回一个新的 `QuerySet`，但这不会真的执行数据库操作。但是如果使用了 `step` 切片语法 例如下面这样，则是会执行数据库操作的：

``` python
>>> Entry.objects.all()[:10:2]
```

由于切片的工作原理是模糊的，所以禁止对切片后的 `queryset` 进行进一步的过滤或排序。

要检索单个对象而不是列表时，请使用下标索引而不是切片，例如：

``` python
>>> Entry.objects.order_by('headline')[0]
```

这大致相当于：

``` python
>>> Entry.objects.order_by('headline')[0:1].get()
```

请注意，如果没有检索到对应的对象，前者会引发 `indexError`，后者会引发 `DoesNotExist`。

### 字段查找

字段查找是指你如何指定 SQL WHERE 子句。它们被指定为 `QuerySet` 方法 `filter()`，`exclude()` 和 `get()` 的关键字参数。

基本查找关键字参数大概像 `field__lookuptype=value`（中间是一个双下划线）。例如：

``` python
>>> Entry.objects.filter(pub_date__lte='2006-01-01')
```

将（大致）转换为以下 SQL：

``` sql
SELECT * FROM blog_entry WHERE pub_date <= '2006-01-01';
```

在查找中指定的字段必须是模型字段的名称。但是有一个例外，在使用 `ForeignKey` 的情况下，你可以指定带 `_id` 后缀的字段名称。在这种情况下， value 参数应该包含外部模型主键的原始值。例如：

``` python
>>> Entry.objects.filter(blog_id=4)
```

如果您传递了无效的关键字参数，则会引发 `TypeError`。

一些常见的查询术语：

`exact`

精确匹配，例如：

``` python
>>> Entry.objects.get(headline__exact="Cat bites dog")
```

将（大致）转换为以下 SQL：

``` sql
SELECT ... WHERE headline = 'Cat bites dog';
```

如果您不提供查找术语 -- 也就是说，如果您的关键字参数不包含双下划线 -- 则查找类型被假定为 `exact`。

例如，以下两条语句是等价的：

``` python
>>> Blog.objects.get(id__exact=14)  # Explicit form
>>> Blog.objects.get(id=14)         # __exact is implied
```

这是为了方便，因为 `exact` 查找是最常见的情况。

`iexact`

不区分大小写的匹配。所以，查询：

``` python
>>> Blog.objects.get(name__iexact="beatles blog")
```

将匹配 `name` 为 `"Beatles Blog"` ，`"beatles blog"` 甚至是 `"BeAtlES blOG"` 的 `Blog` 对象。

`contains`

``` python
Entry.objects.get(headline__contains='Lennon')
```

将（大致）转换为以下 SQL：

``` sql
SELECT ... WHERE headline LIKE '%Lennon%';
```

请注意，这将匹配 `'Today Lennon honored'` 而不会匹配 `'today lennon honored'`。

还有一个不区分大小写的版本 `icontains`。

`startswith`, `endswith`

分别匹配开始和结尾部分。也有不区分大小写的版本 `istartswith` 和 `iendswith`。

### 跨越关系查找

Django 提供了一种强大且直观的方式来在查找中 “追踪” 关系，在幕后自动为您处理 SQL JOIN。要跨越关系，只需使用模型中相关字段的字段名称（用双下划线分隔），直到您到达所需的字段。

本示例使用 `name` 为 `'Beatles Blog'` 的 `Blog` 检索所有 `Entry` 对象：

``` python
>>> Entry.objects.filter(blog__name='Beatles Blog')
```

这种跨越可以尽可能的深入。

它也可以倒退。要引用“反向”关系，只需使用模型的小写名称即可。

此示例检索所有至少有一个 `headline` 包含 `'Lennon'` 的 `Entry` 的 `Blog` 对象：

``` python
>>> Blog.objects.filter(entry__headline__contains='Lennon')
```

如果您要跨多个关系进行筛选，并且其中一个中间模型的值不符合筛选条件，Django 会把它看作是空的（所有的值都是 `NULL`），但是它是有效的，在那里有对象。所有这一切意味着不会出现任何错误。例如，在这个过滤器中：

``` python
Blog.objects.filter(entry__authors__name='Lennon')
```

（如果有相关的 `Author` 模型），如果没有 `Author` 与某个条目相关联，则会将其视为没有附加 `name`，而不是由于缺少 `Author` 而引发错误。通常这正是你希望的结果。唯一可能引起混淆的情况是如果你使用 `isnull`。从而：

``` python
Blog.objects.filter(entry__authors__name__isnull=True)
```

将返回 `Author` 上具有空 `name` 的 `Blog` 对象以及 `entry` 上具有空 `Author` 的 `Blog` 对象。如果你不想要后者，你可以这样写：

``` python
Blog.objects.filter(entry__authors__isnull=False, entry__authors__name__isnull=True)
```

#### 跨越多值关系

当您基于 `ManyToManyField` 或反向 `ForeignKey` 过滤对象时，您可能会感兴趣的是两种不同类型的过滤器。考虑 `Blog`/`Entry` 关系（`Blog` to `Entry` 是一对多的关系）。我们可能会感兴趣的是找到一个有 `entry` 的 `blog`，`headline` 中有 `“Lennon”`，并于 2008 年发表。或者，我们可能想要找到在 `headline` 中有 `“Lennon”` `entry` 的 `blog`，以及 2008 年发表的一篇文章。由于有多个 `entry` 与单个 `Blog` 相关联，所以这些查询都是可能的，并且在某些情况下是有意义的。

同样的情况也出现在 `ManyToManyField` 上。例如，如果一个 `Entry` 有一个 `ManyToManyField` 的 `tags`，我们可能想要找到链接到名为 “music” 和 “bands” 的 `tag` 的 `entry`，或者我们可能需要包含 `name` 为 “music” 和 状态为 “public” 的 `tag` 的 `entry`。

为了处理这两种情况，Django 拥有一致的处理 `filter()` 调用的方法。同时应用单个 `filter()` 调用中的所有内容以筛选出满足所有这些要求的项目。连续的 `filter()` 调用进一步限制了对象集合，但对于多值关系，它们适用于链接到主模型的任何对象，而不一定是由较早的 `filter()` 调用选择的那些对象。

这听起来有点令人困惑，所以希望有一个例子可以说明。要选择包含 `headline` 为 “Lennon” 并且在 2008 年发布的 `entry`（满足这两个条件的同一 `entry`）的所有 `blog`，我们会写：

``` python
Blog.objects.filter(entry__headline__contains='Lennon', entry__pub_date__year=2008)
```

要选择 `headline` 中包含 “Lennon” `entry` 的所有 `blog` 以及 2008 年发表的 `entry`，我们将编写：

``` python
Blog.objects.filter(entry__headline__contains='Lennon').filter(entry__pub_date__year=2008)
```

假设只有一个 `blog` 有包含 “Lennon” 的 `entry` 和 2008 年的 `entry` ，但是 2008 年的 `entry` 中没有一个包含 “Lennon”。第一个查询不会返回任何 `blog`，但第二个查询将返回该 `blog`。

在第二个示例中，第一个过滤器将查询集限制为链接到 `headline` 中带有 “Lennon”  `entry` 的所有 `blog`。第二个过滤器将这组 `blog` 进一步限制为那些也链接到 2008 年发布的 `entry` 的 `blog`。第二个过滤器选择的 `entry` 可能与第一个过滤器中的 `entry` 相同或不同。我们使用每个过滤器语句过滤 `Blog` item，而不是 `Entry` item。


如上所述，跨越多值关系的查询的 `filter()` 行为对于 `exclude()` 不等效。相反，单个 `exclude()` 调用中的条件不一定引用相同的 item。

例如，以下查询将排除同时包含 2008 年发布的 `entry` 和 `headline` 中包含 “Lennon” `entry` 的 `blog`：

``` python
Blog.objects.exclude(
    entry__headline__contains='Lennon',
    entry__pub_date__year=2008,
)
```

但是，与使用 `filter()` 时的行为不同，这不会限制基于满足这两个条件的 `entry` 的 `blog`。为了做到这一点，例如，选择所有不包含 2008 年出版的 “Lennon” `entry` 的 `blog`，你需要做两个查询：

``` python
Blog.objects.exclude(
    entry__in=Entry.objects.filter(
        headline__contains='Lennon',
        pub_date__year=2008,
    ),
)
```

### 过滤器可以引用模型上的字段

在迄今为止给出的例子中，我们构建了一个过滤器，它将模型字段的值与常量进行比较。但是如果您想要将模型字段的值与同一模型中的另一个字段进行比较呢？

Django提供了 [F 表达式](https://docs.djangoproject.com/zh-hans/2.0/ref/models/expressions/#django.db.models.F) 来允许这样的比较。`F()` 的实例充当对查询中的模型字段的引用。然后可以在查询过滤器中使用这些引用来比较同一模型实例上两个不同字段的值。

例如，要查找比 `pingbacks` 有更多注释的所有 `blog` `entry` 的列表，我们构造一个 `F()` 对象来引用 `pingback` 计数，并在查询中使用该 `F()` 对象：

``` python
>>> from django.db.models import F
>>> Entry.objects.filter(n_comments__gt=F('n_pingbacks'))
```

Django 支持对 `F()` 对象使用常量和其他 `F()` 对象的加法，减法，乘法，除法，模和幂运算。为了找到所有比 `pingbacks` 多两倍的评论，我们修改查询：

``` python
>>> Entry.objects.filter(n_comments__gt=F('n_pingbacks') * 2)
```

要查找 `entry` 评级小于 `pingback` 计数和评论计数总和的所有条目，我们将发出查询：

``` python
>>> Entry.objects.filter(rating__lt=F('n_comments') + F('n_pingbacks'))
```

您还可以使用双下划线表示法来跨越 `F()` 对象中的关系。具有双下划线的 `F()` 对象将引入访问相关对象所需的任何连接。例如，要检索作者姓名与博客名称相同的所有条目，我们可以发出查询：

``` python
>>> Entry.objects.filter(authors__name=F('blog__name'))
```

对于 date 和 date/time 字段，可以添加或减去 `timedelta` 对象。以下内容将返回发布后超过 3 天修改的所有 `entry`：

``` python
>>> from datetime import timedelta
>>> Entry.objects.filter(mod_date__gt=F('pub_date') + timedelta(days=3))
```

`F()` 对象支持 `.bitand()`，`.bitor()`，`.bitrightshift()` 和 `.bitleftshift()` 的按位运算。例如：

``` python
>>> F('somefield').bitand(16)
```

### 通过 pk 查找

为了方便起见，Django 提供了一个 pk 查找快捷方式，代表 "primary key"。

在示例 Blog 模型中，primary key 是 id 字段，所以这三个语句是等同的：

``` python
>>> Blog.objects.get(id__exact=14) # Explicit form
>>> Blog.objects.get(id=14) # __exact is implied
>>> Blog.objects.get(pk=14) # pk implies id__exact
```

pk 的使用不限于 `__exact` 查询 -- 任何查询术语都可以与 pk 结合来对模型的主键执行查询：

``` python
# Get blogs entries with id 1, 4 and 7
>>> Blog.objects.filter(pk__in=[1,4,7])

# Get all blog entries with id > 14
>>> Blog.objects.filter(pk__gt=14)
```

pk 查找也可以在连接中使用。例如，这三个语句是等同的：

``` python
>>> Entry.objects.filter(blog__id__exact=3) # Explicit form
>>> Entry.objects.filter(blog__id=3)        # __exact is implied
>>> Entry.objects.filter(blog__pk=3)        # __pk implies __id__exact
```

### 在 LIKE 语句中转义百分号和下划线

等同于 LIKE SQL 语句（iexact，contains，icontains，startswith，istartswith，endswith 和 iendswith）的字段查找将自动转义为 LIKE 语句中使用的两个特殊字符 -- 百分号和下划线。（在 LIKE 语句中，百分号表示多字符通配符，下划线表示单字符通配符。）

这意味我们可以专注于业务逻辑，而不用考虑 SQL 语法。例如，要检索包含百分号的所有条目，只需要直接使用百分号：

``` python
>>> Entry.objects.filter(headline__contains='%')
```

Django 生成的 SQL 将如下所示：

``` sql
SELECT ... WHERE headline LIKE '%\%%';
```

下划线也是如此。

### 缓存和 QuerySet

每个 `QuerySet` 都包含一个缓存以最大限度地减少数据库访问。了解它的工作原理将使您能够编写最高效的代码。

在新创建的 `QuerySet` 中，缓存为空。当 `QuerySet` 第一次被使用 -- 并因此发生数据库查询 -- Django 将查询结果保存在 `QuerySet` 的缓存中，并返回已明确请求的结果（例如，如果 `QuerySet` 正在迭代，则返回下一个元素）。随后对 `QuerySet` 的操作重新使用缓存的结果。

如果没有正确使用 `QuerySet` ，它可能会消耗里的资源。比如以下方式将创建两个 `QuerySet` ，对其进行操作，之后丢弃它们：

``` python
>>> print([e.headline for e in Entry.objects.all()])
>>> print([e.pub_date for e in Entry.objects.all()])
```

这意味着相同的数据库查询将执行两次，显著地加倍了数据库负载。另外，这两个列表可能不包含相同的数据库记录，因为在两个请求之间可能已经添加或删除了一个 `Entry`。

为了避免这个问题，只需保存 `QuerySet` 并重用它：

``` python
>>> queryset = Entry.objects.all()
>>> print([p.headline for p in queryset]) # Evaluate the query set.
>>> print([p.pub_date for p in queryset]) # Re-use the cache from the evaluation.
```

#### 当 QuerySet 没有被缓存时

查询集并不总是缓存它们的结果。在仅操作部分查询集时，会检查缓存，但如果未填充，则后续查询返回的项不会被缓存。具体而言，这意味着使用数组切片或索引截取查询集不会填充缓存。

例如，重复获取 `queryset` 对象中的某个索引将每次查询数据库：

``` python
>>> queryset = Entry.objects.all()
>>> print(queryset[5]) # Queries the database
>>> print(queryset[5]) # Queries the database again
```

但是，如果整个查询集已经被执行（读取使用过），记录将被缓存：

``` python
>>> queryset = Entry.objects.all()
>>> [entry for entry in queryset] # Queries the database
>>> print(queryset[5]) # Uses cache
>>> print(queryset[5]) # Uses cache
```

以下是将导致整个查询集被使用并因此填充缓存的其他操作的一些示例：

``` python
>>> [entry for entry in queryset]
>>> bool(queryset)
>>> entry in queryset
>>> list(queryset)
```

!> 简单地打印查询集不会填充缓存。这是因为调用 `__repr__()` 仅返回整个查询集的一部分。

## 使用 Q 对象进行复杂查询

在 `filter()` 中使用多个关键字参数查询，对应到数据库中是使用 `AND` 连接起来的。如果你想使用更复杂的查询（例如，使用 `OR` 语句的查询），可以使用 `Q` 对象。

`Q` 对象（`django.db.models.Q`）是一个用于封装关键字参数集合的对象。这些关键字参数在上面的 “字段查找” 中指定。

例如，这个 `Q` 对象封装了一个 `LIKE` 查询：

``` python
from django.db.models import Q
Q(question__startswith='What')
```

 `Q` 对象可以使用 `＆` 和 `|` 进行组合运算。当一个操作符用于两个 `Q` 对象时，它会生成一个新的 `Q` 对象。

例如，这个语句产生一个 `Q` 对象，它表示两个 `"question__startswith"` 查询的 `"OR"`：

``` python
Q(question__startswith='Who') | Q(question__startswith='What')
```

这相当于以下 SQL WHERE 子句：

``` sql
WHERE question LIKE 'Who%' OR question LIKE 'What%'
```

通过将 `Q` 对象与 `＆` 和 `|` 相结合，您可以编写任意复杂的语句运算符并使用括号分组。此外，可以使用 `〜` 运算符来否定 `Q` 对象，从而允许将普通查询和否定（`NOT`）查询组合在一起的组合查找：

``` python
Q(question__startswith='Who') | ~Q(pub_date__year=2005)
```

每个使用关键字参数的查找函数（例如 `filter()`，`exclude()`，`get()`）也可以传递一个或多个 `Q` 对象作为位置（未命名）参数。如果您为查找函数提供多个 `Q` 对象参数，则参数将一起 “`AND`”。例如：

``` python
Poll.objects.get(
    Q(question__startswith='Who'),
    Q(pub_date=date(2005, 5, 2)) | Q(pub_date=date(2005, 5, 6))
)
```

底层的 SQL 大概是这样：

``` python
SELECT * from polls WHERE question LIKE 'Who%'
    AND (pub_date = '2005-05-02' OR pub_date = '2005-05-06')
```

查找函数可以混合使用 `Q` 对象和关键字参数。提供给查找函数的所有参数（不管它们是关键字参数还是 `Q` 对象）都是 “`AND`” 一起编辑的。如果提供了一个 `Q` 对象，它必须在任何关键字参数的定义之前。例如：

``` python
Poll.objects.get(
    Q(pub_date=date(2005, 5, 2)) | Q(pub_date=date(2005, 5, 6)),
    question__startswith='Who',
)
```

...将是一个有效的查询，相当于前面的例子;但：

``` python
# INVALID QUERY
Poll.objects.get(
    question__startswith='Who',
    Q(pub_date=date(2005, 5, 2)) | Q(pub_date=date(2005, 5, 6))
)
```

...将无效。

> Django 单元测试中的 [`OR` 查找示例](https://github.com/django/django/blob/master/tests/or_lookups/tests.py)显示了 `Q` 的一些可能的用法。

## 比较对象

要比较两个模型实例，只需使用标准的 Python 比较运算符双等号: `==`。在底层，Django 将比较了两个模型的 primary key。

使用上面的 `Entry` 例子，以下两个语句是等价的：

``` python
>>> some_entry == other_entry
>>> some_entry.id == other_entry.id
```

如果模型的 primary key 不是 `id`，也没有问题。不管它叫什么，比较总是使用 primary key。例如，如果模型的 primary key 字段被称为 `name`，则这两个语句是等同的：

``` python
>>> some_obj == other_obj
>>> some_obj.name == other_obj.name
```

## 删除对象


方便的删除方法被命名为 `delete()`。此方法立即删除对象并返回删除的对象数量和每个对象类型具有删除次数的字典。例：

``` python
>>> e.delete()
(1, {'weblog.Entry': 1})
```

您也可以批量删除对象。每个 `QuerySet` 都有一个 `delete()` 方法，该方法删除该 `QuerySet` 的所有成员。

例如，这将删除 `pub_date` 年份为 `2005` 年的所有 `Entry` 对象：

``` python
>>> Entry.objects.filter(pub_date__year=2005).delete()
(5, {'webapp.Entry': 5})
```

当 Django 删除一个对象时，默认情况下它会模拟 SQL 约束 `ON DELETE CASCADE` 的行为 -- 换句话说，任何有外键指向要删除的对象的对象都将被删除。例如：

``` python
b = Blog.objects.get(pk=1)
# This will delete the Blog and all of its Entry objects.
b.delete()
```

此级联行为可通过 `ForeignKey` 的 `on_delete` 参数进行自定义。

请注意，`delete()` 是唯一不在 `Manager` 本身公开的 `QuerySet` 方法。这是一种安全机制，可防止您意外地请求 `Entry.objects.delete()` 并删除所有条目。如果您确实要删除所有对象，则必须明确请求一个完整的查询集：

``` python
Entry.objects.all().delete()
```

## 复制模型实例

虽然没有用于复制模型实例的内置方法，但可以轻松创建复制了所有字段值的新实例。在最简单的情况下，你可以将 `pk`设置为 `None`。使用我们的博客示例：

``` python
blog = Blog(name='My blog', tagline='Blogging is easy')
blog.save() # blog.pk == 1

blog.pk = None
blog.save() # blog.pk == 2
```

如果你使用继承，事情变得更加复杂。考虑一下 `Blog` 的一个子类：

``` python
class ThemeBlog(Blog):
    theme = models.CharField(max_length=200)

django_blog = ThemeBlog(name='Django', tagline='Django is easy', theme='python')
django_blog.save() # django_blog.pk == 3
```

由于继承的工作原理，你必须将 `pk` 和 `id` 都设置为 `None`：

``` python
django_blog.pk = None
django_blog.id = None
django_blog.save() # django_blog.pk == 4
```

此过程不会复制不属于模型数据库表的关系。`Entry` 有一个 `ManyToManyField` to `Author`。复制 `entry` 后，您必须设置新 `entry` 的多对多关系：

``` python
entry = Entry.objects.all()[0] # some previous entry
old_authors = entry.authors.all()
entry.pk = None
entry.save()
entry.authors.set(old_authors)
```

对于 `OneToOneField`，您必须复制相关对象并将其分配给新对象的字段，以避免违反一对一唯一约束。例如，假设 `entry` 已经被重复如上：

``` python
detail = EntryDetail.objects.all()[0]
detail.pk = None
detail.entry = entry
detail.save()
```

## 一次更新多个对象

有时你想为 `QuerySet` 中的所有对象设置一个字段为特定的值。你可以用 `update()` 方法做到这一点。例如：

``` python
# Update all the headlines with pub_date in 2007.
Entry.objects.filter(pub_date__year=2007).update(headline='Everything is the same')
```

您只能使用此方法设置非关系字段和 `ForeignKey` 字段。要更新非关系字段，请将新值作为常量提供。要更新ForeignKey字段，请将新值设置为要指向的新模型实例。例如：

``` python
>>> b = Blog.objects.get(pk=1)

# Change every Entry so that it belongs to this Blog.
>>> Entry.objects.all().update(blog=b)
```

该 `update()` 方法立即应用并返回查询匹配的行数（如果某些行已具有新值，则该行数不计算在更新的行数内）。`QuerySet` 被更新的唯一限制是它只能访问一个数据库表：模型的主表。您可以根据相关字段进行过滤，但只能更新模型主表中的列。例：

``` python
>>> b = Blog.objects.get(pk=1)

# 更新属于这个博客的所有标题。
>>> Entry.objects.select_related().filter(blog=b).update(headline='Everything is the same')
```

请注意 `update()` 方法直接转换为 SQL 语句。这是直接更新的批量操作。它不会在模型上运行任何 `save()` 方法，或者发出 `pre_save` 或 `post_save` 信号（这是调用 `save()` 的结果），或兑现 `auto_now` 字段选项。如果要将每个 item 保存在 `QuerySet` 中并确保在每个实例上调用 `save()` 方法，则不需要任何特殊函数来处理该 item。只需循环它们并调用 `save()`：

``` python
for item in my_queryset:
    item.save()
```

调用更新也可以使用 `F` 表达式根据模型中另一个字段的值更新一个字段。这对于根据计数器的当前值递增计数器特别有用。例如，要增加博客中每个条目的 `pingback` 计数：

``` python
>>> Entry.objects.all().update(n_pingbacks=F('n_pingbacks') + 1)
```

但是，与 `filter` 和 `exclude` 子句中的 `F()` 对象不同，在 `update` 中使用 `F()` 对象时不能引入连接 -- 您只能引用正在更新的模型的本地字段。如果您尝试使用 `F()` 对象引入联接，则会引发 `FieldError`：

``` python
# This will raise a FieldError
>>> Entry.objects.update(headline=F('blog__name'))
```

## 关系对象

当您在模型中定义关系（即 `ForeignKey`，`OneToOneField` 或 `ManyToManyField` ）时，该模型的实例将具有访问相关对象的方便的 API。

本节中的所有示例均使用本页顶部定义的示例 `Blog`，`Author` 和 `Entry` 模型。

### 一对多的关系

#### 正向访问

如果模型具有 `ForeignKey` ，则该模型的实例将通过模型的简单属性访问相关（外部）对象。

举例：

``` python
>>> e = Entry.objects.get(id=2)
>>> e.blog # Returns the related Blog object.
```

您可以通过外键属性操作外键对象。在调用 `save()` 之前，对外键的更改不会保存到数据库。例：

``` python
>>> e = Entry.objects.get(id=2)
>>> e.blog = some_blog
>>> e.save()
```

如果 `ForeignKey` 字段具有 `null=True`（即，它允许 `NULL` 值），则可以分配 `None` 以移除关系。例：

``` python
>>> e = Entry.objects.get(id=2)
>>> e.blog = None
>>> e.save() # "UPDATE blog_entry SET blog_id = NULL ...;"
```

第一次访问相关对象时，缓存对一对多关系的正向访问。随后访问同一对象实例上的外键将被缓存。例：

``` python
>>> e = Entry.objects.get(id=2)
>>> print(e.blog)  # Hits the database to retrieve the associated Blog.
>>> print(e.blog)  # Doesn't hit the database; uses cached version.
```

请注意，`select_related()` `QuerySet` 方法会提前预先填充所有一对多关系的缓存。例：

``` python
>>> e = Entry.objects.select_related().get(id=2)
>>> print(e.blog)  # Doesn't hit the database; uses cached version.
>>> print(e.blog)  # Doesn't hit the database; uses cached version.
```

#### 反向访问

如果模型具有 `ForeignKey`，则外键模型的实例将有权访问 `Manager`，该 `Manager` 返回第一个模型的所有实例。默认情况下，此 `Manager` 名为 `FOO_set`，其中 `FOO` 是源模型名称，小写。该 `Manager` 返回 `QuerySet`，可以按照上述 “检索对象” 一节中的描述进行过滤和操作。

举例：

``` python
>>> b = Blog.objects.get(id=1)
>>> b.entry_set.all() # Returns all Entry objects related to Blog.

# b.entry_set is a Manager that returns QuerySets.
>>> b.entry_set.filter(headline__contains='Lennon')
>>> b.entry_set.count()
```

您可以通过在 `ForeignKey` 定义中设置 `related_name` 参数来覆盖 `FOO_set` 名称。例如，如果 `Entry` 模型被更改为 `blog=ForeignKey(Blog, on_delete=models.CASCADE, related_name='entries')`，上面的示例代码将如下所示：

``` python
>>> b = Blog.objects.get(id=1)
>>> b.entries.all() # Returns all Entry objects related to Blog.

# b.entries is a Manager that returns QuerySets.
>>> b.entries.filter(headline__contains='Lennon')
>>> b.entries.count()
```

#### 使用自定义反向 manager

默认情况下，用于反向关系的 `RelatedManager` 是该模型的默认 manager 的子类。如果您想为给定的查询指定不同的 manager，可以使用以下语法：

``` python
from django.db import models

class Entry(models.Model):
    #...
    objects = models.Manager()  # Default Manager
    entries = EntryManager()    # Custom Manager

b = Blog.objects.get(id=1)
b.entry_set(manager='entries').all()
```

如果 `EntryManager` 在其 `get_queryset()` 方法中执行了默认过滤，则该过滤将应用于 `all()` 调用。

当然，指定一个自定义反向 manager 也可以让你调用它的自定义方法：

``` python
b.entry_set(manager='entries').is_published()
```

#### 处理关系对象的其他方法

除了上面 “检索对象” 中定义的 `QuerySet` 方法之外，`ForeignKey` `Manager` 还具有用于处理关系对象集的附加方法。

`add(obj1, obj2, ...)`

将特定的模型对象加入关联对象集合。

`create(**kwargs)`

创建一个新对象，将其保存并放入关系对象集中。返回新创建的对象。

`remove(obj1, obj2, ...)`

从关系对象集中删除指定的模型对象。

`clear()`

从关系对象集中删除所有对象。

`set(objs)`

替换一组相关的对象。

要分配相关集的成员，请将 `set()` 方法与可迭代的对象实例一起使用。例如，如果 `e1` 和 `e2` 是 `Entry` 实例：

``` python
b = Blog.objects.get(id=1)
b.entry_set.set([e1, e2])
```

如果 `clear()` 方法可用，则在 iterable（本例中为列表）中的所有对象都添加到 set 之前，将从 `entry_set` 中删除任何预先存在的对象。如果 `clear()` 方法不可用，则会添加迭代中的所有对象而不删除任何现有元素。

本节中描述的每个 “反向” 操作对数据库都有直接影响。每一次添加，创建和删除都会立即自动保存到数据库中。

### 多对多的关系

多对多关系的两端都可以自动访问另一端的 API。API 的作用类似于上面的 “反向” 一对多关系。

一个区别在于属性命名：定义 `ManyToManyField` 的模型使用该字段本身的属性名称，而 “反向” 模型使用原始模型的小写模型名称和 “`_set`”（就像反向一对多关系）。

一个例子使得这更容易理解：

``` python
e = Entry.objects.get(id=3)
e.authors.all() # Returns all Author objects for this Entry.
e.authors.count()
e.authors.filter(name__contains='John')

a = Author.objects.get(id=5)
a.entry_set.all() # Returns all Entry objects for this Author.
```

像 `ForeignKey` 一样，`ManyToManyField` 可以指定 `related_name`。在上面的例子中，如果 `Entry` 中的 `ManyToManyField` 指定了 `related_name='entries'`，那么每个 `Author` 实例将具有 `entries` 属性而不是 `entry_set`。

与一对多关系的另一个区别是，除了模型实例之外，多对多关系中的 `add()`，`set()` 和 `remove()` 方法接受主键值。例如，如果 `e1` 和 `e2` 是 `Entry` 实例，那么这些 `set()` 调用的工作原理是相同的：

``` python
a = Author.objects.get(id=5)
a.entry_set.set([e1, e2])
a.entry_set.set([e1.pk, e2.pk])
```

### 一对一的关系

一对一关系与多对一关系非常相似。如果您在模型上定义了 `OneToOneField`，则该模型的实例将通过模型的简单属性访问相关对象。

``` python
class EntryDetail(models.Model):
    entry = models.OneToOneField(Entry, on_delete=models.CASCADE)
    details = models.TextField()

ed = EntryDetail.objects.get(id=2)
ed.entry # Returns the related Entry object.
```

差异出现在 “反向” 查询中。一对一关系中的相关模型也可以访问 `Manager` 对象，但该 `Manager` 表示一个对象，而不是一组对象：

``` python
e = Entry.objects.get(id=2)
e.entrydetail # returns the related EntryDetail object
```

如果没有对象被分配给这个关系，Django 将引发一个 `DoesNotExist` 异常。

实例可以按照与分配转发关系相同的方式分配给反向关系：

``` python
e.entrydetail = ed
```