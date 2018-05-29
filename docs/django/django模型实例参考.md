# 模型实例参考

## 创建模型

`class Model(**kwargs)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/base/#Model)

关键字参数是您在模型上定义的字段的名称。请注意，实例化模型不会触及数据库;为此，您需要 `save()`。

> 如果要在模型上自定义初始化方法，请使用以下方式之一。<br> <br>
> 1. 在模型类上添加一个 `classmethod`：
>    ``` python
>    from django.db import models
>
>    class Book(models.Model):
>        title = models.CharField(max_length=100)
>
>        @classmethod
>        def create(cls, title):
>            book = cls(title=title)
>            # do something with the book
>            return book
>
>    book = Book.create("Pride and Prejudice")
>    ```
> 2. 在自定义管理器中添加一个方法（通常是首选）：
>    ``` python
>    class BookManager(models.Manager):
>        def create_book(self, title):
>            book = self.create(title=title)
>            # do something with the book
>            return book
>    
>    class Book(models.Model):
>        title = models.CharField(max_length=100)
>    
>        objects = BookManager()
>    
>    book = Book.objects.create_book("Pride and Prejudice")
>    ```

## 刷新数据库中的对象

如果从模型实例中删除一个字段，则再次访问该字段会重新从数据库中载入值：

``` python
>>> obj = MyModel.objects.first()
>>> del obj.field
>>> obj.field  # Loads the field from the database
```

如果您需要从数据库重新加载模型的值，则可以使用该 `refresh_from_db()` 方法。

## 保存对象

要将对象保存回数据库，请调用 `save()`：

### 自动递增主键

如果一个模型有一个 `AutoField` - 一个自动递增的主键 - 那么当您第一次调用 `save()` 时，该自动递增的值将被计算并保存为对象的一个​​属性：

``` python
>>> b2 = Blog(name='Cheddar Talk', tagline='Thoughts on cheese.')
>>> b2.id     # Returns None, because b doesn't have an ID yet.
>>> b2.save()
>>> b2.id     # Returns the ID of your new object.
```

在调用 `save()` 之前，没有办法知道 id 的值是什么，因为该值是由数据库而不是由 Django 计算的。

为方便起见，除非您在模型中的字段上明确指定 `primary_key=True`，否则每个模型默认都有一个名为 `id` 的 `AutoField`。

#### pk 属性

`model.pk`

无论您是自己定义主键字段还是让 Django 为您提供主键字段，每个模型都会有一个名为 `pk` 的属性。它表现得像模型上的普通属性，但实际上是模型的主键字段的别名。您可以像读取任何其他属性一样读取和设置此值，并且它将更新模型中的正确字段。

#### 显式指定自动主键值

如果模型具有 `AutoField`，但您希望在保存时明确定义新对象的 `ID`，则只需在保存前明确定义它，而不是依赖 `ID` 的自动分配：

``` python
>>> b3 = Blog(id=3, name='Cheddar Talk', tagline='Thoughts on cheese.')
>>> b3.id     # Returns 3.
>>> b3.save()
>>> b3.id     # Returns 3.
```

如果您手动分配自动主键值，请确保不要使用已有的主键值！如果您使用数据库中已存在的显式主键值创建新对象，Django 会假定您正在更改现有记录而不是创建新记录。

### 当你调用 `save()` 时会发生什么？

1. **发出 `pre_save` 信号**。允许监听该信号的函数执行一些操作。
2. **预处理数据**。调用每个字段的 `pre_save()` 方法来执行自动数据的修改。例如，日期/时间字段覆盖 `pre_save()` 以实现 `auto_now_add` 和 `auto_now`。
3. **准备数据库的数据**。要求每个字段的 `get_db_prep_save()` 方法在可写入数据库的数据类型中提供其当前值。<br> <br>大多数字段不需要数据准备。简单的数据类型，如整数和字符串，可以作为 Python 对象“准备好写入”。但是，更复杂的数据类型通常需要进行一些修改。<br> <br>例如， `DateField` 字段使用 Python `datetime` 对象来存储数据。数据库不存储 `datetime` 对象，所以必须将字段值转换为符合 ISO 的日期字符串以便插入到数据库中。
4. **将数据插入数据库**。预处理的准备好的数据组成一个 SQL 语句，用于插入数据库。
5. **发出 `post_save` 信号**。允许监听该信号的函数执行一些操作。

### Django 如何知道 UPDATE 与 INSERT

您可能已经注意到 Django 数据库对象使用相同的 `save()` 方法来创建和更改对象。Django 抽象了使用 `INSERT` 或 `UPDATE` SQL 语句的必要性。具体来说，当你调用 `save()` 时，Django 遵循这个算法：

 * 如果对象的主键(primary key)属性设置了一个值为 `True`的值（即，除 `None` 或『空字符串』以外的值），Django 将执行 `UPDATE`。
 * 如果该对象的主键(primary key)属性未设置，或者 `UPDATE` 未更新任何内容，则 Django 将执行 INSERT。

这里的一个问题是，如果您不能保证主键值未被使用，那么在保存新对象时，您应该小心不要明确指定主键值。

#### 强制 INSERT 或 UPDATE

您可以将 `force_insert=True` 或 `force_update=True` 参数传递给 `save()` 方法以强制插入或更新。

> 你几乎不会用到这两个参数。

### 根据现有字段更新属性

有时您需要在一个字段上执行一个简单的算术任务，例如递增或递减当前值。达到此目的的显而易见的方法是执行如下操作：

``` python
>>> product = Product.objects.get(name='Venezuelan Beaver Cheese')
>>> product.number_sold += 1
>>> product.save()
```

如果 `number_sold` 从数据库中检索到的旧值是 `10`，则 `11` 将被写回数据库。

通过表示相对于原始字段值的更新，而不是作为新值的明确分配，可以使该过程变得健壮，避免竞争状况，并且稍微更快。Django 提供了 `F` 表达式来执行这种相对更新。使用 `F` 表达式，前面的例子表示为：

``` python
>>> from django.db.models import F
>>> product = Product.objects.get(name='Venezuelan Beaver Cheese')
>>> product.number_sold = F('number_sold') + 1
>>> product.save()
```

### 指定要保存的字段

如果 `save()` 在关键字参数 `update_fields` 中传递了字段名称列表，则只会更新该列表中指定的字段。如果您想更新对象上的一个或几个字段，这可能是可取的。防止所有模型字段在数据库中更新会带来轻微的性能优势。

``` python
product.name = 'Name changed again'
product.save(update_fields=['name'])
```

`update_fields` 参数可以是任何包含字符串的 iterable。一个空的 `update_fields` iterable 将跳过保存。值 `None` 将在所有字段上执行更新。

指定 `update_fields` 将强制更新。

## 删除对象

`Model.delete(using=DEFAULT_DB_ALIAS, keep_parents=False)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/base/#Model.delete)

为对象发出 SQL DELETE。这只会删除数据库中的对象;Python 实例仍然存在，并且在其字段中仍然有数据。此方法返回删除的对象数和每个对象类型具有删除次数的字典。

有时在多表继承时，您可能只想删除一个子模型的数据。指定 `keep_parents=True` 将保留父模型的数据。

## 其他模型实例方法

一些对象方法有特殊用途。

### `__str__()`

Django `str(obj)` 在很多地方使用。最值得注意的是，在 Django 管理站点中显示对象，并在显示对象时将其作为插入到模板中的值。因此，您应该始终从 `__str__()` 方法中返回一个漂亮的，人类可读的模型表示。

``` python
from django.db import models

class Person(models.Model):
    first_name = models.CharField(max_length=50)
    last_name = models.CharField(max_length=50)

    def __str__(self):
        return '%s %s' % (self.first_name, self.last_name)
```

### `__eq__()`

相等方法的定义是，除了主键值为 `None` 的实例不等于除自身以外的任何实例外，具有相同主键值和相同具体类的实例被认为是相同的。对于代理模型，具体类被定义为模型的第一个非代理父代;对于所有其他模型，它只是模型的类。

``` python
from django.db import models

class MyModel(models.Model):
    id = models.AutoField(primary_key=True)

class MyProxyModel(MyModel):
    class Meta:
        proxy = True

class MultitableInherited(MyModel):
    pass

# Primary keys compared
MyModel(id=1) == MyModel(id=1)
MyModel(id=1) != MyModel(id=2)
# Primay keys are None
MyModel(id=None) != MyModel(id=None)
# Same instance
instance = MyModel(id=None)
instance == instance
# Proxy model
MyModel(id=1) == MyProxyModel(id=1)
# Multi-table inheritance
MyModel(id=1) != MultitableInherited(id=1)
```

## 额外的实例方法

除 `save()`，`delete()` 之外，模型对象可能具有以下某些方法：

`Model.get_FOO_display()`

对于设置了 `choices` 的每个字段，对象都有一个 `get_FOO_display()` 方法，其中 `FOO` 是该字段的名称。该方法返回该字段的“人类可读”值。

``` python
from django.db import models

class Person(models.Model):
    SHIRT_SIZES = (
        ('S', 'Small'),
        ('M', 'Medium'),
        ('L', 'Large'),
    )
    name = models.CharField(max_length=60)
    shirt_size = models.CharField(max_length=2, choices=SHIRT_SIZES)
```

``` python
>>> p = Person(name="Fred Flintstone", shirt_size="L")
>>> p.save()
>>> p.shirt_size
'L'
>>> p.get_shirt_size_display()
'Large'
```

## 其他属性

`DoesNotExist`

ORM 在几个地方引发了这个异常，例如 `QuerySet.get()` 在给定查询参数未找到对象时。Django 提供了一个 `DoesNotExist` 异常作为每个模型类的属性来标识无法找到的对象的类，并允许您用 `try/except` 来捕获特定的模型类。异常是 `django.core.exceptions.ObjectDoesNotExist` 的子类。