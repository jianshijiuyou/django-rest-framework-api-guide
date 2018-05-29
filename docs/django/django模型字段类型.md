# 模型字段参考

## 字段选项

以下参数适用于所有字段类型。全部都是可选的。

### null

如果是 `True`，Django 将允许在数据库该字段的值为 `NULL`。默认是 `False`。

### blank

如果是 `True`，该字段被允许为空值。默认是 `False`。

!> 请注意，这不同于 `null`。`null` 纯粹是与数据库相关的，而 `blank` 与验证相关。如果一个字段有 `blank=True`，表单验证将允许输入一个空值。如果一个字段有 `blank=False`，该字段将是必需的。

### choices

一个可迭代的对象（例如，一个列表或元组），其自身包含两个项目（`[(A, B), (A, B) ...]`）的迭代项，以用作该字段的选项。如果这是给定的，默认表单小部件将是一个选择框和这些选择而不是标准文本字段。

每个元组中的第一个元素是要在模型上设置的实际值，第二个元素是人类可读的名称。例如：

``` python
YEAR_IN_SCHOOL_CHOICES = (
    ('FR', 'Freshman'),
    ('SO', 'Sophomore'),
    ('JR', 'Junior'),
    ('SR', 'Senior'),
)
```

通常，最好在模型类中定义选项，并为每个值定义一个适当命名的常量：

``` python
from django.db import models

class Student(models.Model):
    FRESHMAN = 'FR'
    SOPHOMORE = 'SO'
    JUNIOR = 'JR'
    SENIOR = 'SR'
    YEAR_IN_SCHOOL_CHOICES = (
        (FRESHMAN, 'Freshman'),
        (SOPHOMORE, 'Sophomore'),
        (JUNIOR, 'Junior'),
        (SENIOR, 'Senior'),
    )
    year_in_school = models.CharField(
        max_length=2,
        choices=YEAR_IN_SCHOOL_CHOICES,
        default=FRESHMAN,
    )

    def is_upperclass(self):
        return self.year_in_school in (self.JUNIOR, self.SENIOR)
```

您还可以将可用选项收集到可用于组织目的的命名组中：

``` python
MEDIA_CHOICES = (
    ('Audio', (
            ('vinyl', 'Vinyl'),
            ('cd', 'CD'),
        )
    ),
    ('Video', (
            ('vhs', 'VHS Tape'),
            ('dvd', 'DVD'),
        )
    ),
    ('unknown', 'Unknown'),
)
```

对于已 `choices` 设置的每个模型字段，Django 将添加一个方法来检索字段当前值的人类可读名称。请参阅 `get_FOO_display()` 数据库 API 文档。

!> 如果 `choices` 是动态的，最好换成 `ForeignKey`。

除非在字段中设置 `blank=False` ， 否则将使用选择框呈现 `default` 包含的标签 `"---------"`。要覆盖此行为，添加一个元组 `choices` 包含 `None`; 例如 `(None, 'Your String For Display')` 。或者，你可以使用空字符串而不是 `None`，这是有意义的 - 比如在 `CharField` 上。

### db_column

用于此字段的数据库列的名称。如果没有给出，Django 将使用该字段的名称。

如果您的数据库列名是 SQL 保留字，或者包含 Python 变量名中不允许使用的字符 - 特别是连字符 - 这没关系。 Django在幕后引用列名和表名。

### db_index

如果为 `True`，则会为此字段创建数据库索引。

### db_tablespace

如果此字段已编入索引，则用于此字段索引数据库表空间（ database tablespace）的名称。默认值是项目的 `DEFAULT_INDEX_TABLESPACE` setting（如果已设置）或模型的 `db_tablespace`（如果有）。如果后端不支持索引的表空间，则忽略此选项。

### default

字段的默认值。这可以是一个值或一个可调用的对象。如果是可调用对象，则每次创建新对象时都会调用它。

默认值不能是可变对象（模型实例，list，set 等），因为对该对象的同一实例的引用将用作所有新模型实例中的默认值。相反的，应该将所需的默认值包装为可调用的对象。例如，如果你想为 `JSONField` 指定一个默认字典，可以使用一个函数：

``` python
def contact_default():
    return {"email": "to1@example.com"}

contact_info = JSONField("ContactInfo", default=contact_default)
```

lambda 表达式不能用于默认字段选项，因为它们不能被迁移序列化。

对于像映射到模型实例的 `ForeignKey` 这样的字段，默认值应该是它们引用的字段的值（一般是 pk 除非设置了 to_field）而不是模型实例。

当创建新的模型实例并且未为该字段提供值时使用默认值。当该字段是主键时，当该字段设置为 `None` 时也使用默认值。

### editable

如果为 `False`，则该字段将不会显示在 admin 或任何其他 ModelForm 中。在模型验证过程中它们也被跳过。默认值为 `True`。

### error_messages

`error_messages` 参数允许您覆盖该字段将引发的默认消息。传入一个字典，其中包含与要覆盖的错误消息相匹配的 `key`。

错误消息 `key` 包括 `null`, `blank`, `invalid`, `invalid_choice`, `unique`, 和 `unique_for_date`。

这些错误消息通常不会传播到表单。

### help_text

用表单小部件显示额外的 “帮助” 文本。即使您的字段未用于表单，对于文档也很有用。

请注意，该值在自动生成的表单中不是 HTML 转义的。如果您愿意，可以在 `help_text` 中包含 HTML。例如：

``` python
help_text="Please use the following format: <em>YYYY-MM-DD</em>."
```

或者，您可以使用纯文本和 `django.utils.html.escape()`来转义任何 HTML 特殊字符。确保您避开可能来自不受信任的用户的任何帮助文本，以避免发生跨站点脚本攻击。

### primary_key

如果为 `True`，则此字段是模型的主键。

如果您没有为模型中的任何字段指定 `primary_key=True`，Django 将自动添加一个 `AutoField` 来保存主键，因此您不需要在任何字段上设置 `primary_key=True`，除非您想覆盖默认的主键行为。

`primary_key=True`意味着 `null=False` 且 `unique=True`。一个对象只允许有一个主键。

主键字段是只读的。如果您更改现有对象上主键的值并保存它，则会在旧对象旁边创建一个新对象。

### unique

如果为 `True`，则该字段在整个表格中必须是唯一的。

这是在数据库级和模型验证实施的。如果您尝试在唯一字段中保存具有重复值的模型，则模型的 `save()` 方法会引发 `django.db.IntegrityError`。

此选项在除 `ManyToManyField` 和 `OneToOneField` 之外的所有字段类型中都有效。

!> 请注意，当 `unique` 为 `True` 时，您不需要指定 `db_index` ，因为 `unique` 意味着创建索引。

### unique_for_date

将其设置为 `DateField` 或 `DateTimeField` 的名称，以要求该字段对于日期字段的值是唯一的。

例如，如果您有一个字段 `title` 设置了 `unique_for_date="pub_date"`，那么 Django 将不允许输入两个具有相同 `title` 和 `pub_date` 的记录。

请注意，如果您将其设置为指向 `DateTimeField`，则只会考虑字段的日期部分。另外，当 `USE_TZ` 为 `True` 时，检查将在对象保存时在当前时区执行。

这由 `Model.validate_unique()` 在模型验证期间实施，但不在数据库级。如果任何 `unique_for_date` 约束涉及不属于 `ModelForm` 的字段（例如，如果其中一个字段在 exclude 中列出或具有 `editable=False`），则 `Model.validate_unique()` 将跳过对该特定约束的验证。

### unique_for_month

与 `unique_for_date` 一样，但要求该字段相对于该月份是唯一的。

### unique_for_year

像 `unique_for_date` 和 `unique_for_month` 一样。

### verbose_name

该字段的人类可读名称。如果没有给出详细名称，Django 将使用该字段的属性名称自动创建它，并将下划线转换为空格。

### validators

要为此字段运行的验证器列表。

## 字段类型

### AutoField

`class AutoField(**options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#AutoField)

一个 `IntegerField`，根据可用的 ID 自动递增。你通常不需要直接使用它;如果不另外指定，主键字段将自动添加到您的模型中。

### BigAutoField

`class BigAutoField(**options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#BigAutoField)

一个 64 位整数，与 `AutoField` 非常相似，只是保证适合 1 到 9223372036854775807 之间的数字。

### BigIntegerField

`class BigIntegerField(**options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#BigIntegerField)

一个 64 位整数，与 `IntegerField` 非常相似，只不过它保证适合从 -9223372036854775808 到 9223372036854775807 之间的数字。此字段的默认表单小部件是 `TextInput` 。

### BinaryField

`class BinaryField(**options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#BinaryField)

用于存储原始二进制数据的字段。它只支持字节分配。请注意，该字段的功能有限。例如，无法过滤 `BinaryField` 值上的查询集。在 `ModelForm` 中也不可能包含 `BinaryField`。

!> 别将静态文集存放到数据库中

### BooleanField

`class BooleanField(**options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#BooleanField)

一个 true/false 字段。

该字段的默认表单小部件是 `CheckboxInput`。

如果您需要接受 `null`，请改为使用 `NullBooleanField`。

当未定义 `default` 时，`BooleanField` 的默认值为 `None`。

### CharField

`class CharField(max_length=None, **options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#CharField)

一个字符串字段，用于小型到大型字符串。

对于大量的文本，请使用 `TextField`。

该字段的默认表单小部件是一个 `TextInput`。

`CharField` 有一个额外的必需参数：

#### max_length

字段的最大长度（以字符为单位）。`max_length` 在数据库级别和 Django 的验证中执行。

### DateField

`class DateField(auto_now=False, auto_now_add=False, **options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#DateField)

表示日期，由 Python 以 `datetime.date` 实例表示。有一些额外的可选参数：

#### auto_now

每次保存对象时自动将字段设置为当前日期。用于 `"last-modified"` 的时间戳。请注意，它始终使用当前日期;这不仅仅是您可以覆盖的默认值。

调用 `Model.save()` 时，该字段只会自动更新。在以其他方式更新其他字段（如 `QuerySet.update()`）时，字段不会更新，但您可以在更新中为字段指定自定义值。

#### auto_now_add

首次创建对象时，自动将字段设置为当前日期。用于创建时间戳。请注意，它始终使用当前日期;这不仅仅是您可以覆盖的默认值。所以即使您在创建对象时为此字段设置了值，它也会被忽略。如果您希望能够修改此字段，请设置以下字段而不是 `auto_now_add=True`：

* DateField: `default=date.today` - from `datetime.date.today()`
* DateTimeField: `default=timezone.now` - from `django.utils.timezone.now()`

该字段的默认表单小部件是一个 `TextInput`。admin 添加了 JavaScript 日历和 "Today" 的快捷方式。包含一个额外的 `invalid_date` 错误消息 key。

选项 `auto_now_add`，`auto_now` 和 `default` 是互斥的。这些选项的任何组合都会导致错误。

!> 按照当前的实现，将 `auto_now` 或 `auto_now_add` 设置为 `True` 将导致该字段具有 `editable=False` 和 `blank=True` 设置。

### DateTimeField

`class DateTimeField(auto_now=False, auto_now_add=False, **options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#DateTimeField)

表示日期和时间，由 Python 以 `datetime.datetime` 实例表示。采用与 `DateField` 相同的额外参数。

此字段的默认表单小部件是单个 `TextInput`。admin 使用两个独立的 `TextInput` 小部件和 JavaScript 快捷键。

### DecimalField

`class DecimalField(max_digits=None, decimal_places=None, **options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#DecimalField)

一个固定精度的十进制数，由 Python 用 `Decimal` 实例表示。有两个必需的参数：

#### max_digits

数字中允许的最大位数。请注意，此数字必须大于或等于 `decimal_places`。

#### decimal_places

数字中的小数位数。

例如，要将精度为 2 位小数的数字存储到 999，您可以使用：

``` python
models.DecimalField(..., max_digits=5, decimal_places=2)
```

当 `localize` 为 `False` 或 `TextInput` 时，此字段的默认表单小部件为 `NumberInput`。

### DurationField

`class DurationField(**options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#DurationField)

用于存储时间段的字段 - 由 `timedelta` 建模。

### EmailField

`class EmailField(max_length=254, **options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#EmailField)

一个 `CharField`，用于检查该值是否为有效的电子邮件地址。它使用 [`EmailValidator`](https://docs.djangoproject.com/zh-hans/2.0/ref/validators/#django.core.validators.EmailValidator) 来验证输入。

### FileField

`class FileField(upload_to=None, max_length=100, **options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/files/#FileField)

文件上传字段。

!> `primary_key` 参数不受支持，如果使用，会引发错误。

有两个可选参数：

#### upload_to

该属性提供了一种设置上传目录和文件名的方法，可以通过两种方式进行设置。在这两种情况下，该值都传递给 `Storage.save()` 方法。

如果你指定一个字符串值，它可能包含 `strftime()` 格式，它将被文件上传的 日期/时间 所取代（这样上传的文件不会填满给定的目录）。例如：

``` python
class MyModel(models.Model):
    # file will be uploaded to MEDIA_ROOT/uploads
    upload = models.FileField(upload_to='uploads/')
    # or...
    # file will be saved to MEDIA_ROOT/uploads/2015/01/30
    upload = models.FileField(upload_to='uploads/%Y/%m/%d/')
```

如果您使用的是默认的 `FileSystemStorage`，则字符串值将被附加到您的 `MEDIA_ROOT` 路径中，以在本地文件系统上形成存储上传文件的位置。如果您使用的是不同的存储，请检查存储的文档以了解它如何处理 `upload_to`。

`upload_to` 也可以是可调用的，例如函数。这将被调用来获取上传路径，包括文件名。这个可调用的方法必须接受两个参数，并返回一个 Unix 风格的路径（带正斜杠）传递给存储系统。这两个参数是：

| 参数 | 描述 |
|:-------------
| `instance` |  `FileField` 定义模型的一个实例。更具体地说，这是当前文件被附加的特定实例。 <br> <br> 在大多数情况下，该对象还没有被保存到数据库中，所以如果它使用默认的 `AutoField`，它可能还没有它的主键字段的值。
| `filename` | 最初提供给该文件的文件名。在确定最终目的地路径时，这可能会也可能不会被考虑在内。


举个栗子

``` python
def user_directory_path(instance, filename):
    # file will be uploaded to MEDIA_ROOT/user_<id>/<filename>
    return 'user_{0}/{1}'.format(instance.user.id, filename)

class MyModel(models.Model):
    upload = models.FileField(upload_to=user_directory_path)
```

该字段的默认表单小部件是 `ClearableFileInput`。

在模型中使用 `FileField` 或 `ImageField` （见下文）需要几个步骤：

1. 在您的设置文件中，您需要将 `MEDIA_ROOT` 定义为您希望 Django 存储上传文件的目录的完整路径。将 `MEDIA_URL` 定义为该目录的基本公用 URL。确保 Web 服务器的用户有该目录的写入权限。
2. 将 `FileField` 或 `ImageField` 添加到模型中，定义 `upload_to` 选项以指定用于上传文件的 `MEDIA_ROOT` 子目录。
3. 所有将存储在数据库中的文件都是一个路径（相对于 `MEDIA_ROOT`）。你很可能想使用 Django 提供的便捷 url 属性。例如，如果您的 `ImageField` 被称为 `mug_shot` ，则可以使用 {{object.mug_shot.url}} 在模板中获取图像的绝对路径。

例如，假设您的 `MEDIA_ROOT` 设置为 `'/home/media'`，`upload_to` 设置为 `'photos/%Y/%m/%d'`。`upload_to` 的 `'%Y/%m/%d'` 部分是 `strftime()` 格式;`'％Y'` 是四位数年份，`'％m'` 是两位数月份，`'％d'` 是两位数日期。如果您在 2007年1月15日 上传文件，它将被保存在 `/home/media/photos/2007/01/15` 目录中。

如果您想要检索上传文件的磁盘文件名或文件大小，可分别使用 `name` 和 `size` 属性;

!> 该文件作为将模型保存在数据库中的一部分进行保存，因此在保存模型之前，不能依赖磁盘上使用的实际文件名。

上传文件的相对 URL 可以使用 `url` 属性获取。在内部，它调用底层 `Storage` 类的 `url()` 方法。

请注意，无论何时处理上传的文件，都应密切关注您上传的文件以及它们的类型，以避免安全漏洞。验证所有上传的文件，以便确保文件是您认为的文件。例如，如果您盲目地让某人上传文件（如果没有进行验证）到 Web 服务器文档根目录中的目录中，则有人可以上传 CGI 或 PHP 脚本，并通过访问您网站上的 URL 来执行该脚本。

还要注意，即使是上传的 HTML 文件，由于它可以被浏览器执行（尽管不是由服务器执行），也可能造成与 XSS 或 CSRF 攻击等效的安全威胁。

`FileField` 实例在数据库中创建为 `varchar` 默认最大长度为 100 个字符的字段。与其他字段一样，您可以使用 `max_length` 参数更改最大长度。

### FilePathField

`class FilePathField(path=None, match=None, recursive=False, max_length=100, **options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#FilePathField)

是一个 `CharField` 。选项仅限于文件系统上某个目录中的文件名。

有三个特殊的参数，其中第一个是必需的：

#### path

必须参数。`FilePathField` 应该从中选择的目录的绝对文件系统路径。例：`"/home/images"`。

#### match

可选参数。一个正则表达式字符串，`FilePathField` 将用它来过滤文件名。请注意，正则表达式将应用于基本文件名，而不是完整路径。例如：`"foo.*\.txt$"`，它将匹配一个名为 `foo23.txt` 但不是 `bar.txt` 或 `foo23.png` 的文件。

#### recursive

可选参数。`True`或 `False`。默认值是 `False`。指定是否应该包含路径的所有子目录

#### allow_files

可选参数。`True`或 `False`。默认值是 `True`。指定是否应该包含指定位置中的文件。 `allow_files` 或 `allow_folders` 必须有一个是 `True`。

#### allow_folders

可选参数。`True`或 `False`。默认值是 `False`。指定是否应该包含指定位置的文件夹。`allow_folders` 或 `allow_files` 必须有一个是 `True`。

当然，这些参数可以一起使用。

一个潜在的问题是匹配适用于基本文件名，而不是完整路径。所以，这个例子：

``` python
FilePathField(path="/home/images", match="foo.*", recursive=True)
```

...将匹配 `/home/images/foo.png`，但不匹配 `/home/images/foo/bar.png`，因为匹配适用于基本文件名（`foo.png` 和 `bar.png`）。

`FilePathField` 实例在数据库中创建为 `varchar` 默认最大长度为 100 个字符的字段。与其他字段一样，您可以使用 `max_length` 参数更改最大长度。

### FloatField

`class FloatField(**options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#FloatField)

由 Python 中的 `float` 实例表示的浮点数。

当 `localize` 为 `False` 或 `TextInput` 时，此字段的默认表单小部件为 `NumberInput`。

### ImageField

`class ImageField(upload_to=None, height_field=None, width_field=None, max_length=100, **options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/files/#ImageField)

继承 `FileField` 的所有属性和方法，但也验证上传的对象是不是有效的 image。

除了可用于 `FileField` 的特殊属性外，`ImageField` 还具有 `height` 和 `width` 属性。

为了便于查询这些属性，`ImageField` 有两个额外的可选参数：

#### height_field

每次保存模型实例时自动填充为图像的高度。

#### width_field

每次保存模型实例时自动填充为图像的宽度。

!> 使用 `ImageField` 必须要安装 Pillow 库

`ImageField` 实例在数据库中创建为默认最大长度为 100 个字符的 varchar 字段。其他字段一样，您可以使用 `max_length` 参数更改最大长度。

该字段的默认表单小部件是 `ClearableFileInput`。

### IntegerField

`class IntegerField(**options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#IntegerField)

一个整数。在 Django 支持的所有数据库中，值从 -2147483648 到 2147483647 都是安全的。当 `localize` 为 `False` 或 `TextInput` 时，此字段的默认表单小部件为 `NumberInput`。

### GenericIPAddressField

`class GenericIPAddressField(protocol='both', unpack_ipv4=False, **options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#GenericIPAddressField)

一个 IPv4 或 IPv6 地址，采用字符串格式（例如 `192.0.2.30` 或者 `2a02:42fe::4`）。该字段的默认表单小部件是一个 `TextInput`。

#### protocol

限制有效输入的协议。接受的值是 `'both'`（默认），`'IPv4'` 或 `'IPv6'`。匹配不区分大小写。

#### unpack_ipv4

解压 IPv4 映射的地址，如 `::ffff:192.0.2.1`。如果启用此选项，则该地址将解压到 `192.0.2.1`。默认是禁用的。只能在 `protocol` 设置为 `'both'` 时使用。

### NullBooleanField

`class NullBooleanField(**options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#NullBooleanField)

像一个 `BooleanField`，但允许 `NULL` 作为其中一个选项。使用它而不是使用 `null=True` 的 `BooleanField`。此字段的默认表单小部件是 `NullBooleanSelect`。

### PositiveIntegerField

`class NullBooleanField(**options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#PositiveIntegerField)

像 `IntegerField` 一样，但必须是正数或零（0）。在 Django 支持的所有数据库中，值从 0 到 2147483647 都是安全的。出于向后兼容性的原因，接受值 0。

### PositiveSmallIntegerField

`class PositiveSmallIntegerField(**options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#PositiveSmallIntegerField)

像 `PositiveIntegerField` 一样，但只允许在某个（数据库相关）点下的值。在 Django 支持的所有数据库中，从 0 到 32767 的值都是安全的。

### SlugField

`class SlugField(max_length=50, **options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#SlugField)

作为某些东西的简短标签，只能包含字母，数字，下划线或连字符。他们一般被用于 URL。

就像 `CharField` 一样，您可以指定 `max_length`。如果未指定 `max_length`，则 Django 将使用默认长度 50。

使用此字段意味着将 `db_index` 设置为 `True`。

根据其他值的值自动预先填充 `SlugField` 通常很有用。您可以在 admin 中使用 `prepopulated_fields` 自动执行此操作。

#### allow_unicode

如果为 `True`，则该字段除 ASCII 字母外还接受 Unicode 字母。默认为 `False`。

### SmallIntegerField

`class SmallIntegerField(**options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#SmallIntegerField)

像 `IntegerField` 一样，但只允许在某个（数据库相关）点下的值。在 Django 支持的所有数据库中，-32768 到 32767 的值都是安全的。

### TextField

`class TextField(**options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#TextField)

一个大文本字段。该字段的默认表单小部件是一个 `Textarea`。

如果您指定了 `max_length` 属性，它将反映在自动生成的表单字段的 Textarea 小部件中。但是它不是在模型或数据库级别实施的。

### TimeField

`class TimeField(auto_now=False, auto_now_add=False, **options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#TimeField)

表示时间，在 Python 中由 `datetime.time` 实例表示。接受与 `DateField` 相同的自动填充选项。

该字段的默认表单小部件是一个 `TextInput`。admin 添加了一些 `JavaScript` 快捷方式。

### URLField

`class URLField(max_length=200, **options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#URLField)

一个表示 URL 的 `CharField`。

该字段的默认表单小部件是一个 `TextInput`。

像所有 `CharField` 子类一样，`URLField` 采用可选的 `max_length` 参数。如果您不指定 `max_length` ，则使用默认值 200。

### UUIDField

`class UUIDField(**options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/#UUIDField)

用于存储 UUID 的字段。使用 Python 的 `UUID` 类。在 PostgreSQL 上使用时，它将以 `uuid` 数据类型存储，否则以 `char(32)` 存储。

使用 `primary_key` 参数的 `UUIDField` 是 `AutoField` 的一个很好的替代方案。数据库不会为您生成 UUID，因此建议使用 ` default`：

``` python
import uuid
from django.db import models

class MyUUIDModel(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    # other fields
```
!> 请注意，`default` 接收的是一个可调用对象，而不是一个 `UUID` 实例。

## 关系字段

### ForeignKey

`class ForeignKey(to, on_delete, **options)` [[source]](https://docs.djangoproject.com/zh-hans/2.0/_modules/django/db/models/fields/related/#ForeignKey)

多对一的关系。需要两个位置参数：与模型相关的类和 `on_delete` 选项。

创建递归关系 -- 与自身具有多对一关系的对象 -- 使用 `models.ForeignKey('self', on_delete=models.CASCADE)` 。

如果您需要在尚未定义的模型上创建关系，则可以使用模型的名称，而不是模型对象本身：

``` python
from django.db import models

class Car(models.Model):
    manufacturer = models.ForeignKey(
        'Manufacturer',
        on_delete=models.CASCADE,
    )
    # ...

class Manufacturer(models.Model):
    # ...
    pass
```

当抽象模型被定义为具体模型的子类时，解析在抽象模型上定义的关系，并且与抽象模型的 app_label 不相关：

`products/models.py`

``` python
from django.db import models

class AbstractCar(models.Model):
    manufacturer = models.ForeignKey('Manufacturer', on_delete=models.CASCADE)

    class Meta:
        abstract = True
```

`production/models.py`

``` python
from django.db import models
from products.models import AbstractCar

class Manufacturer(models.Model):
    pass

class Car(AbstractCar):
    pass

# Car.manufacturer will point to `production.Manufacturer` here.
```

要引用另一个应用程序中定义的模型，可以使用完整的应用程序标签明确指定一个模型。例如，如果上面的Manufacturer模型是在另一个称为production的应用程序中定义的，则需要使用：

``` python
class Car(models.Model):
    manufacturer = models.ForeignKey(
        'production.Manufacturer',
        on_delete=models.CASCADE,
    )
```

当解决两个应用程序之间的循环导入依赖关系时，这种称为懒惰关系的引用可能很有用。

数据库索引是在 `ForeignKey` 上自动创建的。您可以通过将 `db_index` 设置为 `False` 来禁用此功能。如果要为一致性而不是连接创建外键，或者要创建替代索引（如部分索引或多列索引），则可能需要避免索引的开销。

**数据库表示**

在幕后，Django 将 “_id” 附加到字段名称以创建其数据库字段名称。在上例中，`Car` 模型的数据库表将有一个 `manufacturer_id` 列（你可以通过指定db_column来明确地改变它）。但是，除非您编写自定义 SQL，否则您的代码不应该处理数据库列名称。您应该始终处理模型对象的字段名称。

**参数**

#### on_delete

当由 `ForeignKey` 引用的对象被删除时，Django 将模拟由 `on_delete` 参数指定的 SQL 约束的行为。例如，如果您有一个可以为空的 `ForeignKey`，并且您希望在删除引用的对象时将其设置为 `null`：

``` python
user = models.ForeignKey(
    User,
    models.SET_NULL,
    blank=True,
    null=True,
)
```

`on_delete` 的可能值可在 `django.db.models` 中找到：

* `CASCADE`：级联删除。 Django 模拟 SQL 约束 ON DELETE CASCADE 的行为，并删除包含 `ForeignKey` 的对象。
* `PROTECT`：通过引发 `ProtectedError`（ `django.db.IntegrityError` 的子类）来防止删除引用的对象。
* `SET_NULL`：设置 `ForeignKey` 为 `null`; 这只有在 `null=True` 时才可能。
* `SET_DEFAULT`：将 `ForeignKey`设置为其默认值; 必须设置了 `ForeignKey`的 `default` 参数才行。
* `SET()`：将 `ForeignKey` 设置为传递给 `SET()` 的值，或者如果传递了可调用对象，则调用它的结果。在大多数情况下，为了避免在导入 `models.py` 时执行查询，必须传递可调用对象：  
    ``` python
    from django.conf import settings
    from django.contrib.auth import get_user_model
    from django.db import models

    def get_sentinel_user():
        return get_user_model().objects.get_or_create(username='deleted')[0]

    class MyModel(models.Model):
        user = models.ForeignKey(
            settings.AUTH_USER_MODEL,
            on_delete=models.SET(get_sentinel_user),
        )
    ```
* `DO_NOTHING`：不采取行动。如果数据库后端强制执行参照完整性，则会导致 `IntegrityError`，除非您手动将 SQL ON DELETE 约束添加到数据库字段。

#### limit_choices_to

当此字段使用 ModelForm 或 admin 呈现时，为此字段的可用选项设置限制（默认情况下，查询集中的所有对象可供选择）。可以使用字典，Q 对象或返回字典或 Q 对象的可调用对象。

举个栗子

``` python
staff_member = models.ForeignKey(
    User,
    on_delete=models.CASCADE,
    limit_choices_to={'is_staff': True},
)
```

使 ModelForm 上的相应字段仅列出具有 `is_staff=True` 的用户。这可能对 Django 管理员有帮助。

可调用表单可以是有用的，例如，当与 Python 日期时间模块结合使用时，可以按日期范围限制选择。例如：

``` python
def limit_pub_date_choices():
    return {'pub_date__lte': datetime.date.utcnow()}

limit_choices_to = limit_pub_date_choices
```

如果 `limit_choices_to` 为或返回Q对象，这对于复杂查询很有用，那么只有在该模型的 `ModelAdmin` 的 `raw_id_fields` 中未列出该字段时，它才会对管理员中可用的选择产生影响。

#### related_name

从相关对象返回到这个关系的名称。这也是 `related_query_name` 的默认值（用于来自目标模型的反向过滤器名称的名称）。请注意，在定义关系抽象模型时，您必须设置此值;当你这样做时，一些特殊的语法是可用的。

如果你希望 Django 不要反向关联，请将 `related_name` 设置为 `'+'` 或以 `'+'` 结尾。例如，这将确保用户模型不会与此模型有反向关系：

``` python
user = models.ForeignKey(
    User,
    on_delete=models.CASCADE,
    related_name='+',
)
```

#### related_query_name

用于来自目标模型的反向过滤器名称的名称。它默认设置为 `related_name` 或 `default_related_name` 的值，否则默认为模型的名称：

``` python
# Declare the ForeignKey with related_query_name
class Tag(models.Model):
    article = models.ForeignKey(
        Article,
        on_delete=models.CASCADE,
        related_name="tags",
        related_query_name="tag",
    )
    name = models.CharField(max_length=255)

# That's now the name of the reverse filter
Article.objects.filter(tag__name="important")
```

像 `related_name` 一样，`related_query_name` 支持通过一些特殊语法的应用程序标签和类插值。

#### to_field

关系所在的相关对象上的字段。默认情况下，Django 使用相关对象的主键。如果您引用不同的字段，则该字段必须具有 `unique=True`。

#### db_constraint

控制是否应在数据库中为该外键创建约束。默认值为 `True`，这几乎可以肯定你想要的; 将其设置为 `False` 可能对数据完整性非常不利。也就是说，以下是您可能需要执行此操作的一些场景：

* 您的遗留数据无效。
* 你正在分解你的数据库。

如果设置为 `False`，则访问不存在的相关对象将引发其 `DoesNotExist` 异常。

#### swappable

没看懂

### ManyToManyField

略

### OneToOneField

略
