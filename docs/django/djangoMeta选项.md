# 模型 Meta 选项

本文档解释了您可以在内部类 Meta 中为您的模型提供的所有可能的元数据选项。

## 可用的 Meta 选项

### abstract

如果 `abstract=True`，那么这个模型将是一个抽象基类。

### app_label

如果模型在 `INSTALLED_APPS` 的应用程序之外定义，则它必须声明它属于哪个应用程序：

``` python
app_label = 'myapp'
```

如果要使用格式 `app_label.object_name` 或 `app_label.model_name` 表示模型，则可以分别使用 `model._meta.label` 或 `model._meta.label_lower`。

### base_manager_name

用于模型的 `_base_manager` 的 manager 的名称。

### db_table

用于模型的数据库表的名称：

``` python
db_table = 'music_album'
```

为了节省时间，Django 自动从模型类的名称和包含它的应用程序中派生出数据库表的名称。模型的数据库表名是通过将模型的“应用标签”（您在 `manage.py startapp` 中使用的名称）与模型的类名称相加，并在它们之间加下划线来构建的。

例如，如果您有应用程序 `bookstore`（由 `manage.py startapp bookstore` 创建），则定义为 `Book` 类的模型将具有名为 `bookstore_book` 的数据库表。

要覆盖数据库表名，请使用 Meta 类中的 `db_table` 参数。

!> 强烈建议您在通过 `db_table` 覆盖表名时使用小写表名，尤其是在使用 `MySQL` 后端时。

### db_tablespace

用于此模型的数据库表空间的名称。默认值是项目的 `DEFAULT_TABLESPACE` setting（如果设置）。

### default_manager_name

用于模型 `_default_manager` 的 manager 的名称。

### default_related_name

默认情况下将相关对象关系使用的名称返回到此名称。默认值是 `<model_name>_set`。

该选项还会设置 `related_query_name`。

由于字段的反向名称应该是唯一的，因此如果打算继承模型，请小心。

要解决名称冲突问题，部分名称应该包含 `'%(app_label)s'` 和 `'%(model_name)s'`，它们分别被模型所在的应用程序的名称和模型的名称替换，两者都是小写。

### get_latest_by

模型中的字段名称或字段名称列表，通常为 `DateField`，`DateTimeField` 或 `IntegerField`。这指定了在模型管理器的 `latest()` 和 `earliest()` 方法中使用的默认字段。

例如：

``` python
# Latest by ascending order_date.
get_latest_by = "order_date"

# Latest by priority descending, order_date ascending.
get_latest_by = ['-priority', 'order_date']
```

### managed

默认为 `True`，这意味着 Django 将在迁移或作为迁移的一部分创建适当的数据库表，并将其作为 flush 管理命令的一部分移除。也就是说，Django 管理数据库表的生命周期。

如果为 `False`，则不会为此模型执行数据库表创建或删除操作。如果模型已经通过其他方式创建了现有表或数据库视图，这很有用。这是 `managed=False` 时唯一的区别。模型处理的所有其他方面与正常情况完全相同。这包括

1. 如果不声明它，则将自动主键字段添加到模型。为避免以后的代码阅读器混淆，建议在使用非托管模型时指定要建模的数据库表中的所有列。
2. 如果具有 `managed=False` 的模型包含指向另一个非托管模型的 `ManyToManyField`，则多对多连接的中间表也不会创建。但是，将创建一个托管模型和一个非托管模型之间的中间表。

如果您需要更改此默认行为，请将中间表创建为显式模型（根据需要使用托管集），并使用 `ManyToManyField.through` 属性使关系使用您的自定义模型。

对于涉及 `managed=False` 的模型的测试，您需要确保将正确的表创建为测试设置的一部分。

如果您有兴趣更改模型类的 Python 级行为，可以使用 `managed=False` 并创建现有模型的副本。但是，对于这种情况，有更好的方法：代理模型

### order_with_respect_to

使该对象相对于给定字段可排序，通常是 `ForeignKey`。这可以用来制作相关的对象或者相对于父对象的可配置性。例如，如果 `Answer` 与 `Question` 对象有关，并且问题具有多个答案，并且答案的顺序很重要，则可以这样做：

``` python
from django.db import models

class Question(models.Model):
    text = models.TextField()
    # ...

class Answer(models.Model):
    question = models.ForeignKey(Question, on_delete=models.CASCADE)
    # ...

    class Meta:
        order_with_respect_to = 'question'
```

当设置 `order_with_respect_to` 时，会提供两个附加方法来检索和设置相关对象的顺序：`get_RELATED_order()` 和 `set_RELATED_order()`，其中 `RELATED` 是小写的模型名称。例如，假设 `Question` 对象具有多个相关的 `Answer` 对象，则返回的列表将包含相关 `Answer` 对象的主键：

``` python
>>> question = Question.objects.get(id=1)
>>> question.get_answer_order()
[1, 2, 3]
```

`Question` 对象的相关 `Answer` 对象的顺序可以通过传递 `Answer` 主键列表来设置：

``` python
>>> question.set_answer_order([3, 1, 2])
```

相关的对象还有两个方法，`get_next_in_order()` 和 `get_previous_in_order()`，它们可以用来以正确的顺序访问这些对象。假设 `Answer` 对象是按 id 排序的：

``` python
>>> answer = Answer.objects.get(id=2)
>>> answer.get_next_in_order()
<Answer: 3>
>>> answer.get_previous_in_order()
<Answer: 1>
```

!> `order_with_respect_to` 和 `ordering` 不能一起使用

!> 设置了 `order_with_respect_to` 后需要执行迁移

### ordering

对象的默认排序，用于获取对象列表时使用：

``` python
ordering = ['-order_date']
```

这是一个元组或列表 和/或 查询表达式。每个字符串都是带有可选 `"-"` 前缀的字段名称，它表示降序。没有前缀 `"-"` 的字段将按照升序排列。使用字符串 `"?"` 随机排序。

例如，要按 `pub_date` 字段升序排序，请使用以下命令：

``` python
ordering = ['pub_date']
```

要按 `pub_date` 降序排序，请使用以下命令：

``` python
ordering = ['-pub_date']
```

要按 `pub_date` 降序排序，然后按 `author` 升序排序，请使用以下命令：

``` python
ordering = ['-pub_date', 'author']
```

您也可以使用查询表达式。要按 `author` 升序排序并使空值最后排序，请使用以下命令：

``` python
from django.db.models import F

ordering = [F('author').asc(nulls_last=True)]
```

### permissions

创建此对象时需要额外的权限才能进入权限表。为每个模型自动创建添加，删除和更改权限。这个例子指定了一个额外的权限，`can_deliver_pizzas`：

``` python
permissions = (("can_deliver_pizzas", "Can deliver pizzas"),)
```

这是格式为 `(permission_code, human_readable_permission_name)` 的 2 元组的列表或元组。

### default_permissions

默认为 `('add', 'change', 'delete')`。您可以自定义此列表，例如，如果您的应用不需要任何默认权限，则可以将其设置为空列表。在通过迁移创建模型之前，必须在模型上指定它，以防止创建任何省略的权限。

### proxy

如果 `proxy=True`，则将另一个模型子类的模型视为代理模型。

### required_db_features

列出当前连接应具有的数据库功能，以便在迁移阶段考虑模型。例如，如果您将此列表设置为 `['gis_enabled']`，则只会在启用 GIS 的数据库上同步模型。在使用多个数据库后端进行测试时，跳过某些模型也很有用。避免模型之间的关系，因为 ORM 无法处理这些关系。

### required_db_vendor

该模型特定于的受支持数据库供应商的名称。当前内置的供应商名称是：sqlite，postgresql，mysql，oracle。如果此属性不为空且当前连接供应商与其不匹配，则模型将不会同步。

### select_on_save

确定 Django 是否会使用 1.6 之前的 `django.db.models.Model.save()` 算法。旧算法使用 SELECT 来确定是否存在要更新的现有行。新算法直接尝试 UPDATE。在一些罕见的情况下，Django 不会看到现有行的 UPDATE。一个例子是 PostgreSQL ON UPDATE 触发器返回 NULL。在这种情况下，即使数据库中存在一行，新算法也会执行 INSERT 操作。


通常不需要设置该属性。默认值是 `False`。

### indexes

您想要在模型上定义的索引列表：

``` python
from django.db import models

class Customer(models.Model):
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100)

    class Meta:
        indexes = [
            models.Index(fields=['last_name', 'first_name']),
            models.Index(fields=['first_name'], name='first_name_idx'),
        ]
```

### unique_together

一起组合的字段名称必须是唯一的：

``` python
unique_together = (("driver", "restaurant"),)
```

这是一个元组元组，它们在一起考虑时必须是唯一的。它在 Django admin 中使用，并在数据库级别执行（即，适当的 UNIQUE 语句包含在 CREATE TABLE 语句中）。

为了方便起见，在处理一组字段时，unique_together可以是单个元组：

``` python
unique_together = ("driver", "restaurant")
```

`ManyToManyField` 不能包含在 `unique_together` 中。（目前还不清楚这意味着什么！）如果您需要验证与 `ManyToManyField` 相关的唯一性，请尝试使用信号或明确 through 模型。

在违反约束的模型验证期间引发的 `ValidationError` 具有 `unique_together` 错误代码。

### index_together

!>　较新的 `indexes` 选项提供比 `index_together` 更多的功能。 `index_together` 将来可能会被弃用。

一起放入索引的字段名称集合：

``` python
index_together = [
    ["pub_date", "deadline"],
]
```

该字段列表将被编入索引（即将发布适当的 CREATE INDEX 语句。）

为了方便起见，当处理一组字段时，`index_together` 可以是单个列表：

``` python
index_together = ["pub_date", "deadline"]
```

### verbose_name

对象的人类可读名称，单数：

``` python
verbose_name = "pizza"
```

如果没有给出，Django 将使用类名称的通用版本：CamelCase 变成 camel case。

### verbose_name_plural

对象的复数名称：

``` python
verbose_name_plural = "stories"
```

如果没有给出，Django 将使用 verbose_name +“s”。

!> 中文就直接把单复数设置成一样的就行了

## 只读 Meta 属性

### label

对象的表示，返回 `app_label.object_name`，例如 `'polls.Question'`。

### label_lower

模型的表示，返回 `app_label.model_name`，例如 `'polls.question'`。