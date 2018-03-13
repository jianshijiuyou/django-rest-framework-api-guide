> [官方原文链接](http://www.django-rest-framework.org/api-guide/fields/)


# Serializer 字段

> Form 类中的每个字段不仅负责验证数据，还负责 “清洗” 它 — 将其规范化为一致的格式。  
>
> &mdash; [Django 文档][cite]

序列化字段处理基本数据类型和其他数据类型（比如自定义的类）之间的转换。它们还可以对数据进行验证，以及从其父对象中检索和设置值。

---

**注意：** 序列化字段都声明在 `fields.py`  中，但按照惯例，应该使用 `from rest_framework import serializers` ，并用 `serializers.<FieldName>` 的方式引用。

---

## 核心参数

每个序列化字段类的构造函数都需要一些参数。某些字段类需要附加特定于该字段的参数，但应始终接受以下参数：

### `read_only`

只读字段包含于输出 API 中，不应该包含在需要创建或更新操作的输入 API 中。在序列化类输入中错误的包含 'read_only' 会被忽略。

将其设置为 `True` 可确保在序列化表示时使用该字段，但在反序列化期间创建或更新实例时不使用该字段。

默认为 `False`

### `write_only`

将其设置为 `True` 以确保在更新或创建实例时可以使用该字段，但在序列化表示时不包括该字段。

默认为 `False`

### `required`

如果在反序列化过程中没有该提供字段，通常会出现错误。如果在反序列化过程中不需要此字段，则应该设置为 false。

将此设置为 `False` 还允许在序列化实例时从输出中省略对象属性或字典密钥。如果密钥不存在，它将不会包含在输出表示中。

默认为 `True`

### `allow_null`

如果把 `None` 传递给序列化字段，通常会引发错误。如果 `None` 应被视为有效值，则将此关键字参数设置为 `True` 。

请注意，将此参数设置为 `True` 将意味着序列化输出的缺省值为 `null`，但并不意味着输入反序列化的缺省值。

默认为 `False`

### `default`

如果设置，则会给出默认值，在没有提供输入值时，将使用该默认值。如果未设置，则默认行为是不填充该属性。

部分更新操作时不应该使用 `default`。因为有些情况下，只有传入数据中提供的字段才会返回验证值。

可以设置为函数或其他可调用的对象，在这种情况下，每次使用该值时都会对其进行调用。被调用时，它将不会收到任何参数。如果可调用对象具有 `set_context` 方法，那么在每次将字段实例作为参数获取值之前都会调用该方法。这与验证器的工作方式相同。

在序列化实例时，如果对象属性或字典关键字不存在于实例中，将使用缺省值。

请注意，设置默认值意味着该字段不是必需的。同时包括 `default` 和 `required` 的关键字参数都是无效的，会引发错误。

### `source`

将用于填充字段的属性的名称。可以是一个只接受 `self` 参数的方法，如 `URLField(source='get_absolute_url')`，或者使用点符号来遍历属性，如 `EmailField(source='user.email')`。在使用点符号时，如果在属性遍历期间任何对象不存在或为空，则可能需要提供缺省值。

`source ='*'` 具有特殊含义，用于表示整个对象应该传递到该字段。这对创建嵌套表示或对于需要访问完整对象以确定输出表示的字段非常有用。

默认为该字段的名称。

### `validators`

应该应用于传入字段输入的验证函数列表，该列表中的函数应该引发验证错误或仅返回。验证器函数通常应该引发 `serializers.ValidationError` ，但 Django 的内置 `ValidationError` 也支持与 Django 代码库或第三方 Django 包中定义的验证器兼容。

### `error_messages`

一个字典，key 是错误代码， value 是对应的错误信息。

### `label`

一个简短的文本字符串，可用作 HTML 表单字段或其他描述性元素中字段的名称。

### `help_text`

一个文本字符串，可用作 HTML 表单字段或其他描述性元素中字段的描述。

### `initial`

应该用于预填充 HTML 表单字段的值。你可能会传递一个可调用对象，就像你对任何常规 Django `Field` 所做的一样：

``` python
import datetime
from rest_framework import serializers
class ExampleSerializer(serializers.Serializer):
    day = serializers.DateField(initial=datetime.date.today)
```

### `style`

可用于控制渲染器渲染字段的键值对的字典。

这里有两个例子是 `'input_type'` 和 `'base_template'` ：

``` python
# Use <input type="password"> for the input.
password = serializers.CharField(
    style={'input_type': 'password'}
)

# Use a radio input instead of a select input.
color_channel = serializers.ChoiceField(
    choices=['red', 'green', 'blue'],
    style={'base_template': 'radio.html'}
)
```

---

# Boolean 字段

## BooleanField

表示一个 boolean 值。

使用 HTML 编码表单时需要注意，省略一个 boolean 值被视为将字段设置为 `False`，即使它指定了 `default=True` 选项。这是因为 HTML 复选框通过省略该值来表示未选中的状态，所以 REST framework 将省略看作是空的复选框。

请注意，将使用 `required=False` 选项生成默认的 `BooleanField` 实例（因为 Django `models.BooleanField` 始终为 `blank=True`）。如果想要更改此行为，请在序列化类上显式声明 `BooleanField`。

对应与 `django.db.models.fields.BooleanField`.

**签名：** `BooleanField()`

## NullBooleanField

表示一个布尔值，它也接受 `None` 作为有效值。

对应与 `django.db.models.fields.NullBooleanField`.

**签名：** `NullBooleanField()`

---

# 字符串字段

## CharField

表示文本。可以使用 `max_length` ， `min_length` 验证（或限定）文本的长短。

对应与 `django.db.models.fields.CharField` 或 `django.db.models.fields.TextField`.

**签名：** `CharField(max_length=None, min_length=None, allow_blank=False, trim_whitespace=True)`

- `max_length` - 验证输入所包含的字符数不超过这个数目。
- `min_length` - 验证输入所包含的字符数不少于这个数目。
- `allow_blank` - 如果设置为 `True`，则空字符串应被视为有效值。如果设置为 `False`，那么空字符串被认为是无效的并会引发验证错误。默认为 `False`。
- `trim_whitespace` - 如果设置为 `True`，则前后空白将被删除。默认为 `True`。

`allow_null` 选项也可用于字符串字段，尽管它相对于 `allow_blank` 来说不被推荐。同时设置 `allow_blank=True` 和 `allow_null=True` 是有效的，但这样做意味着字符串表示允许有两种不同类型的空值，这可能导致数据不一致和微妙的应用程序错误。

## EmailField

表示文本，将文本验证为有效的电子邮件地址。

对应与 `django.db.models.fields.EmailField`

**签名：** `EmailField(max_length=None, min_length=None, allow_blank=False)`

## RegexField

表示文本，用于验证给定的值是否与某个正则表达式匹配。

对应与 `django.forms.fields.RegexField`.

**签名：** `RegexField(regex, max_length=None, min_length=None, allow_blank=False)`

强制的 `regex` 参数可以是一个字符串，也可以是一个编译好的 Python 正则表达式对象。

使用 Django 的 `django.core.validators.RegexValidator` 进行验证。

## SlugField

一个根据模式 `[a-zA-Z0-9_-]+` 验证输入的 `RegexField` 。

对应与 `django.db.models.fields.SlugField`.

**签名：** `SlugField(max_length=50, min_length=None, allow_blank=False)`

## URLField

一个根据 URL 匹配模式验证输入的 `RegexField`。完全合格的 URL 格式为 `http://<host>/<path>`。

对应与 `django.db.models.fields.URLField`.  使用 Django 的 `django.core.validators.URLValidator` 进行验证。

**签名：** `URLField(max_length=200, min_length=None, allow_blank=False)`

## UUIDField

确保输入的字段是有效的 UUID 字符串。`to_internal_value` 方法将返回一个 `uuid.UUID` 实例。在输出时，字段将以规范的连字符格式返回一个字符串，例如：

```
"de305d54-75b4-431b-adb2-eb6b9e546013"
```

**签名：** `UUIDField(format='hex_verbose')`

- `format`: 确定 uuid 值的表示形式
    - `'hex_verbose'` - 权威的十六进制表示形式，包含连字符： `"5ce0e9a5-5ffa-654b-cee0-1238041fb31a"`
    - `'hex'` - 紧凑的十六进制表示形式， 不包含连字符：`"5ce0e9a55ffa654bcee01238041fb31a"`
    - `'int'` - 128 位整数表示形式：`"123456789012312313134124512351145145114"`
    - `'urn'` - RFC 4122 URN 表示形式： `"urn:uuid:5ce0e9a5-5ffa-654b-cee0-1238041fb31a"`
  修改 `format` 仅影响表示值。所有格式都被 `to_internal_value` 接受。

## FilePathField

一个其选项仅限于文件系统上某个目录中的文件名的字段。

对应于 `django.forms.fields.FilePathField`.

**签名：** `FilePathField(path, match=None, recursive=False, allow_files=True, allow_folders=False, required=None, **kwargs)`

- `path` - FilePathField 应该从中选择的目录的绝对文件系统路径。
- `match` - 用来过滤文件名的正则表达式，string 类型。
- `recursive` - 指定是否应该包含路径的所有子目录。默认值是 `False`。
- `allow_files` - 是否应该包含指定位置的文件。默认值为 `True`。这个参数或 `allow_folders` 必须是 `True`。（两个属性必须有一个为 `true`）
- `allow_folders` - 是否应该包含指定位置的文件夹。默认值是 `False`。这个参数或 `allow_files` 必须是 `True`。（两个属性必须有一个为 `true`）

## IPAddressField

确保输入是有效的 IPv4 或 IPv6 字符串。

对应于 `django.forms.fields.IPAddressField` 和 `django.forms.fields.GenericIPAddressField`.

**签名：** `IPAddressField(protocol='both', unpack_ipv4=False, **options)`

- `protocol` 将有效输入限制为指定的协议。接受的值是 `'both'` （默认），`'IPv4'` 或 `'IPv6'` 。匹配不区分大小写。
- `unpack_ipv4` 解压 IPv4 映射的地址，如 `::ffff:192.0.2.1`。如果启用此选项，则该地址将解压到 192.0.2.1。 默认是禁用的。只能在 `protocol` 设置为 `'both'` 时使用。

---

# 数字字段

## IntegerField

表示整数。

对应于 `django.db.models.fields.IntegerField`, `django.db.models.fields.SmallIntegerField`, `django.db.models.fields.PositiveIntegerField` 和 `django.db.models.fields.PositiveSmallIntegerField`。

**签名：** `IntegerField(max_value=None, min_value=None)`

- `max_value` 验证所提供的数字不大于这个值。
- `min_value` 验证所提供的数字不小于这个值。

## FloatField

表示浮点。

对应于 `django.db.models.fields.FloatField`.

**签名：** `FloatField(max_value=None, min_value=None)`

- `max_value` 验证所提供的数字不大于这个值。
- `min_value` 验证所提供的数字不小于这个值。

## DecimalField

表示十进制，由 Python 用 `Decimal` 实例表示。

对应于 `django.db.models.fields.DecimalField`.

**签名：** `DecimalField(max_digits, decimal_places, coerce_to_string=None, max_value=None, min_value=None)`

- `max_digits` 允许的最大位数。它必须是 `None` 或大于等于 `decimal_places` 的整数。
- `decimal_places` 小数位数。
- `coerce_to_string` 如果应返回字符串值，则设置为 `True` ;如果应返回 `Decimal` 对象，则设置为 `False` 。默认值与 `COERCE_DECIMAL_TO_STRING` settings key 的值相同，除非被覆盖，否则该值将为 `True`。如果序列化对象返回 `Decimal` 对象，则最终的输出格式将由渲染器决定。请注意，设置 `localize` 将强制该值为 `True`。
- `max_value` 验证所提供的数字不大于这个值。
- `min_value` 验证所提供的数字不小于这个值。
- `localize` 设置为 `True` 以启用基于当前语言环境的输入和输出本地化。这也会迫使 `coerce_to_string` 为 `True` 。默认为 `False` 。请注意，如果你在 settings 文件中设置了 `USE_L10N=True`，则会启用数据格式化。
- `rounding` 设置量化到配置精度时使用的舍入模式。 有效值是 `decimal` 模块舍入模式。默认为 `None`。
#### 用法示例

若要验证数字到999，精确到 2 位小数，应该使用：

    serializers.DecimalField(max_digits=5, decimal_places=2)

用10位小数来验证数字不超过10亿：

    serializers.DecimalField(max_digits=19, decimal_places=10)

这个字段还接受一个可选参数，`coerce_to_string`。如果设置为 `True`，则表示将以字符串形式输出。如果设置为 `False`，则表示将保留为 `Decimal` 实例，最终表示形式将由渲染器确定。

如果未设置，则默认设置为与 `COERCE_DECIMAL_TO_STRING` setting 相同的值，除非另行设置，否则该值为 `True`。

---

# 日期和时间字段

## DateTimeField

表示日期和时间。

对应于 `django.db.models.fields.DateTimeField`.

**签名：** `DateTimeField(format=api_settings.DATETIME_FORMAT, input_formats=None)`

* `format` - 表示输出格式的字符串。如果未指定，则默认为与 `DATETIME_FORMAT` settings key 相同的值，除非设置，否则将为 `'iso-8601'`。设置为格式化字符串则表明 `to_representation` 返回值应该被强制为字符串输出。格式化字符串如下所述。将此值设置为 `None` 表示 Python  `datetime` 对象应由 `to_representation` 返回。在这种情况下，日期时间编码将由渲染器确定。
* `input_formats` - 表示可用于解析日期的输入格式的字符串列表。 如果未指定，则将使用 `DATETIME_INPUT_FORMATS` 设置，该设置默认为 `['iso-8601']`。

#### `DateTimeField` 格式化字符串。

格式化字符串可以是明确指定的 Python strftime 格式，也可以是使用 ISO 8601 风格 datetime 的特殊字符串 `iso-8601` 。（例如 `'2013-01-29T12:34:56.000000Z'`）

当一个 `None` 值被用于格式化 `datetime` 对象时，`to_representation` 将返回，最终的输出表示将由渲染器类决定。

#### `auto_now` 和 `auto_now_add` 模型字段。

使用 `ModelSerializer` 或 `HyperlinkedModelSerializer` 时，请注意，`auto_now=True`或 `auto_now_add=True` 的模型字段默认情况下将使用 `read_only=True` 。

如果想覆盖此行为，则需要在序列化类中明确声明 `DateTimeField`。例如：

``` python
class CommentSerializer(serializers.ModelSerializer):
    created = serializers.DateTimeField()

    class Meta:
        model = Comment
```

## DateField

表示日期。

对应于 `django.db.models.fields.DateField`

**签名：** `DateField(format=api_settings.DATE_FORMAT, input_formats=None)`

* `format` - 表示输出格式的字符串。如果未指定，则默认为与 `DATE_FORMAT` settings key 相同的值，除非设置，否则将为 `'iso-8601'`。设置为格式化字符串则表明 `to_representation` 返回值应该被强制为字符串输出。格式化字符串如下所述。将此值设置为 `None` 表示 Python  `date` 对象应由 `to_representation` 返回。在这种情况下，日期时间编码将由渲染器确定。
* `input_formats` - 表示可用于解析日期的输入格式的字符串列表。 如果未指定，则将使用 `DATE_INPUT_FORMATS` 设置，该设置默认为 `['iso-8601']`。

#### `DateField` 格式化字符串

格式化字符串可以是明确指定的 Python strftime 格式，也可以是使用 ISO 8601 风格 date 的特殊字符串 `iso-8601` 。（例如 `'2013-01-29'`）

## TimeField

表示时间。

对应于 `django.db.models.fields.TimeField`

**签名：** `TimeField(format=api_settings.TIME_FORMAT, input_formats=None)`

* `format` - 表示输出格式的字符串。如果未指定，则默认为与 `TIME_FORMAT` settings key 相同的值，除非设置，否则将为 `'iso-8601'`。设置为格式化字符串则表明 `to_representation` 返回值应该被强制为字符串输出。格式化字符串如下所述。将此值设置为 `None` 表示 Python  `time` 对象应由 `to_representation` 返回。在这种情况下，日期时间编码将由渲染器确定。
* `input_formats` - 表示可用于解析日期的输入格式的字符串列表。 如果未指定，则将使用 `TIME_INPUT_FORMATS` 设置，该设置默认为 `['iso-8601']`。

#### `TimeField` 格式化字符串

格式化字符串可以是明确指定的 Python strftime 格式，也可以是使用 ISO 8601 风格 time 的特殊字符串 `iso-8601` 。（例如 `'12:34:56.000000'`）

## DurationField

表示持续时间。
对应于 `django.db.models.fields.DurationField`

这些字段的 `validated_data` 将包含一个 `datetime.timedelta` 实例。该表示形式是遵循格式 `'[DD] [HH:[MM:]]ss[.uuuuuu]'` 的字符串。

**签名：** `DurationField()`

---

# 选择字段

## ChoiceField

可以从一个有限的选择中接受值的字段。

如果相应的模型字段包含 `choices=…` 参数，则由 `ModelSerializer` 自动生成字段。

**签名：** `ChoiceField(choices)`

- `choices` - 有效值列表，或 `(key, display_name)` 元组列表。
- `allow_blank` - 如果设置为 `True`，则空字符串应被视为有效值。如果设置为 `False`，那么空字符串被认为是无效的并会引发验证错误。默认是 `False`。
- `html_cutoff` - 如果设置，这将是 HTML 选择下拉菜单中显示的选项的最大数量。可用于确保自动生成具有非常大可以选择的 ChoiceField，而不会阻止模板的渲染。默认是 `None`.
- `html_cutoff_text` - 指定一个文本指示器，在截断列表时显示，比如在 HTML 选择下拉菜单中已经截断了最大数量的项目。默认就会显示 `"More than {count} items…"`

`Allow_blank` 和 `allow_null` 都是 `ChoiceField` 上的有效选项，但强烈建议只使用一个而不是两个都用。对于文本选择，`allow_blank` 应该是首选，`allow_null` 应该是数字或其他非文本选项的首选。

## MultipleChoiceField

可以接受一组零、一个或多个值的字段，从有限的一组选择中选择。采取一个必填的参数。 `to_internal_value` 返回一个包含选定值的 `set`。

**签名：** `MultipleChoiceField(choices)`

- `choices` - 有效值列表，或 `(key, display_name)` 元组列表。
- `allow_blank` - 如果设置为 `True`，则空字符串应被视为有效值。如果设置为 `False`，那么空字符串被认为是无效的并会引发验证错误。默认是 `False`。
- `html_cutoff` - 如果设置，这将是 HTML 选择下拉菜单中显示的选项的最大数量。可用于确保自动生成具有非常大可以选择的 ChoiceField，而不会阻止模板的渲染。默认是 `None`.
- `html_cutoff_text` - 指定一个文本指示器，在截断列表时显示，比如在 HTML 选择下拉菜单中已经截断了最大数量的项目。默认就会显示 `"More than {count} items…"`

`Allow_blank` 和 `allow_null` 都是 `ChoiceField` 上的有效选项，但强烈建议只使用一个而不是两个都用。对于文本选择，`allow_blank` 应该是首选，`allow_null` 应该是数字或其他非文本选项的首选。

---

# 文件上传字段

#### 解析和文件上传。

 `FileField`和 `ImageField` 类只适用于 `MultiPartParser` 或 `FileUploadParser` 。大多数解析器，例如， e.g. JSON 不支持文件上传。 Django 的常规 `FILE_UPLOAD_HANDLERS` 用于处理上传的文件。

## FileField

表示文件。执行 Django 的标准 FileField 验证。

对应于 `django.forms.fields.FileField`.

**签名：** `FileField(max_length=None, allow_empty_file=False, use_url=UPLOADED_FILES_USE_URL)`

 - `max_length` - 指定文件名的最大长度。
 - `allow_empty_file` - 指定是否允许空文件。
- `use_url` - 如果设置为 `True`，则 URL 字符串值将用于输出表示。如果设置为 `False`，则文件名字符串值将用于输出表示。默认为 `UPLOADED_FILES_USE_URL` settings key 的值，除非另有设置，否则为 `True`。

## ImageField

表示图片。验证上传的文件内容是否匹配已知的图片格式。

对应于 `django.forms.fields.ImageField`.

**签名：** `ImageField(max_length=None, allow_empty_file=False, use_url=UPLOADED_FILES_USE_URL)`

 - `max_length` - 指定文件名的最大长度。
 - `allow_empty_file` - 指定是否允许空文件。
- `use_url` - 如果设置为 `True`，则 URL 字符串值将用于输出表示。如果设置为 `False`，则文件名字符串值将用于输出表示。默认为 `UPLOADED_FILES_USE_URL` settings key 的值，除非另有设置，否则为 `True`。

需要 `Pillow` 库或 `PIL` 库。  建议使用 `Pillow` 库。 因为 `PIL` 已经不再维护。

---

# 复合字段

## ListField

验证对象列表的字段类。

**签名：** `ListField(child=<A_FIELD_INSTANCE>, min_length=None, max_length=None)`

- `child` - 应该用于验证列表中的对象的字段实例。如果未提供此参数，则列表中的对象将不会被验证。
- `min_length` - 验证列表中包含的元素数量不少于这个数。
- `max_length` - 验证列表中包含的元素数量不超过这个数。

例如，要验证整数列表：
``` python
scores = serializers.ListField(
   child=serializers.IntegerField(min_value=0, max_value=100)
)
```
`ListField` 类还支持一种声明式风格，允许编写可重用的列表字段类。

``` python
class StringListField(serializers.ListField):
    child = serializers.CharField()
```

我们现在可以在我们的应用程序中重新使用我们自定义的 `StringListField` 类，而无需为其提供 `child` 参数。

## DictField

验证对象字典的字段类。`DictField` 中的键总是被假定为字符串值。

**签名：** `DictField(child=<A_FIELD_INSTANCE>)`

- `child` - 应该用于验证字典中的值的字段实例。如果未提供此参数，则映射中的值将不会被验证。

例如，要创建一个验证字符串到字符串映射的字段，可以这样写：

``` python
document = DictField(child=CharField())
```

你也可以像使用 `ListField` 一样使用声明式风格。例如：

``` python
class DocumentField(DictField):
    child = CharField()
```

## JSONField

验证传入的数据结构由有效 JSON 基元组成的字段类。在其二进制模式下，它将表示并验证 JSON 编码的二进制字符串。

**签名：** `JSONField(binary)`

- `binary` - 如果设置为 `True`，那么该字段将输出并验证 JSON 编码的字符串，而不是原始数据结构。默认是 `False`.

---

# 其他类型的字段

## ReadOnlyField

只是简单地返回字段的值而不进行修改的字段类。

当包含与属性相关的字段名而不是模型字段时，此字段默认与 `ModelSerializer` 一起使用。

**签名：** `ReadOnlyField()`

例如，如果 `has_expired` 是 `Account` 模型中的一个属性，则以下序列化程序会自动将其生成为 `ReadOnlyField` ：

``` python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ('id', 'account_name', 'has_expired')
```

## HiddenField

不根据用户输入获取值的字段类，而是从默认值或可调用值中获取值。

**签名：** `HiddenField()`

例如，要包含始终提供当前时间的字段作为序列化类验证数据的一部分，则可以使用以下内容：

``` python
modified = serializers.HiddenField(default=timezone.now)
```

如果需要根据某些预先提供的字段值运行某些验证，则通常只需要 `HiddenField` 类，而不是将所有这些字段公开给最终用户。

## ModelField

可以绑定到任意模型字段的通用字段。`ModelField` 类将序列化/反序列化的任务委托给其关联的模型字段。该字段可用于为自定义模型字段创建序列化字段，而无需创建新的自定义序列化字段。

`ModelSerializer` 使用此字段来对应自定义模型字段类。

**签名：** `ModelField(model_field=<Django ModelField instance>)`

`ModelField` 类通常用于内部使用，但如果需要，可以由 API 使用。为了正确实例化 `ModelField`，必须传递一个附加到实例化模型的字段。例如：`ModelField(model_field=MyModel()._meta.get_field('custom_field'))`

## SerializerMethodField

这是一个只读字段。它通过调用它所连接的序列化类的方法来获得它的值。它可用于将任何类型的数据添加到对象的序列化表示中。

**签名：** `SerializerMethodField(method_name=None)`

- `method_name` - 要调用序列化对象的方法的名称。如果不包含，则默认为 `get_<field_name>`.

由 `method_name` 参数引用的序列化方法应该接受一个参数（除了 `self`），这是要序列化的对象。它应该返回你想要包含在对象的序列化表示中的任何内容。例如：

``` python
from django.contrib.auth.models import User
from django.utils.timezone import now
from rest_framework import serializers

class UserSerializer(serializers.ModelSerializer):
    days_since_joined = serializers.SerializerMethodField()

    class Meta:
        model = User

    def get_days_since_joined(self, obj):
        return (now() - obj.date_joined).days
```



---

# 自定义字段

如果你想创建自定义字段，则需要对 `Field` 进行子类化，然后重写 `.to_representation()` 和 `.to_internal_value()` 方法中的一个或两个。这两个方法用于在初始数据类型和基本序列化数据类型之间进行转换。基本数据类型通常是 number，string， boolean， `date`/`time`/`datetime` 或 `None` 。它们也可以是任何仅包含其他基本对象的列表或字典。其他类型可能会支持，但具体取决于你使用的渲染器。

调用 `.to_representation()` 方法将初始数据类型转换为基本的可序列化数据类型。

调用 `to_internal_value()` 方法将基本数据类型恢复为其内部 python 表示形式。如果数据无效，此方法应该引发 `serializers.ValidationError` 异常。

请注意，2.x 版本中存在的 `WritableField` 类不再存在。 应此，如果字段需要支持数据输入，则应该继承 `Field` 并覆盖 `to_internal_value()`。

## 看几个栗子

### 基本的自定义字段

我们来看一个序列化代表 RGB 颜色值的类的例子：

``` python
class Color(object):
    """
    A color represented in the RGB colorspace.
    """
    def __init__(self, red, green, blue):
        assert(red >= 0 and green >= 0 and blue >= 0)
        assert(red < 256 and green < 256 and blue < 256)
        self.red, self.green, self.blue = red, green, blue

class ColorField(serializers.Field):
    """
    Color objects are serialized into 'rgb(#, #, #)' notation.
    """
    def to_representation(self, obj):
        return "rgb(%d, %d, %d)" % (obj.red, obj.green, obj.blue)

    def to_internal_value(self, data):
        data = data.strip('rgb(').rstrip(')')
        red, green, blue = [int(col) for col in data.split(',')]
        return Color(red, green, blue)
```

默认情况下，字段值被视为映射到对象的属性。如果需要自定义字段值的访问方式，则需要覆盖  `.get_attribute()` 和/或 `.get_value()`。

让我们创建一个可以用来表示被序列化对象的类名的字段：

``` python
class ClassNameField(serializers.Field):
    def get_attribute(self, obj):
        # We pass the object instance onto `to_representation`,
        # not just the field attribute.
        return obj

    def to_representation(self, obj):
        """
        Serialize the object's class name.
        """
        return obj.__class__.__name__
```

### 抛出验证错误

我们上面的 `ColorField` 类目前不执行任何数据验证。为了表示无效数据，我们应该引发一个 `serializers.ValidationError`，如下所示：

``` python
def to_internal_value(self, data):
    if not isinstance(data, six.text_type):
        msg = 'Incorrect type. Expected a string, but got %s'
        raise ValidationError(msg % type(data).__name__)

    if not re.match(r'^rgb\([0-9]+,[0-9]+,[0-9]+\)$', data):
        raise ValidationError('Incorrect format. Expected `rgb(#,#,#)`.')

    data = data.strip('rgb(').rstrip(')')
    red, green, blue = [int(col) for col in data.split(',')]

    if any([col > 255 or col < 0 for col in (red, green, blue)]):
        raise ValidationError('Value out of range. Must be between 0 and 255.')

    return Color(red, green, blue)
```

`.fail()` 方法是引发 `ValidationError` 的快捷方式，它从 `error_messages` 字典中接收消息字符串。例如：

``` python
default_error_messages = {
    'incorrect_type': 'Incorrect type. Expected a string, but got {input_type}',
    'incorrect_format': 'Incorrect format. Expected `rgb(#,#,#)`.',
    'out_of_range': 'Value out of range. Must be between 0 and 255.'
}

def to_internal_value(self, data):
    if not isinstance(data, six.text_type):
        self.fail('incorrect_type', input_type=type(data).__name__)

    if not re.match(r'^rgb\([0-9]+,[0-9]+,[0-9]+\)$', data):
        self.fail('incorrect_format')

    data = data.strip('rgb(').rstrip(')')
    red, green, blue = [int(col) for col in data.split(',')]

    if any([col > 255 or col < 0 for col in (red, green, blue)]):
        self.fail('out_of_range')

    return Color(red, green, blue)
```

这种风格让你的错误信息更清晰，并且与代码分离，应该是首选。

### 使用 `source='*'`

这里我们将举一个具有 `x_coordinate` 和 `y_coordinate` 属性的平面 `DataPoint` 模型的示例。

``` python
class DataPoint(models.Model):
    label = models.CharField(max_length=50)
    x_coordinate = models.SmallIntegerField()
    y_coordinate = models.SmallIntegerField()
```

使用自定义字段和 `source ='*'`，我们可以提供坐标对的嵌套表示形式：

``` python
class CoordinateField(serializers.Field):

    def to_representation(self, obj):
        ret = {
            "x": obj.x_coordinate,
            "y": obj.y_coordinate
        }
        return ret

    def to_internal_value(self, data):
        ret = {
            "x_coordinate": data["x"],
            "y_coordinate": data["y"],
        }
        return ret


class DataPointSerializer(serializers.ModelSerializer):
    coordinates = CoordinateField(source='*')

    class Meta:
        model = DataPoint
        fields = ['label', 'coordinates']
```

请注意，此示例不处理验证。使用 `source ='*'` 的嵌套序列化类可以更好地处理坐标嵌套，并且带有两个 `IntegerField` 实例，每个实例都有自己的 `source` 指向相关字段。

然而，这个例子的关键点是：

* `to_representation` 传递整个 `DataPoint` 对象， 并且会映射到所需的输出。

``` python
>>> instance = DataPoint(label='Example', x_coordinate=1, y_coordinate=2)
>>> out_serializer = DataPointSerializer(instance)
>>> out_serializer.data
ReturnDict([('label', 'Example'), ('coordinates', {'x': 1, 'y': 2})])
```

* 除非我们的字段是只读的，否则 `to_internal_value` 必须映射回适合更新目标对象的字典。使用 `source='*'` ，`to_internal_value` 的返回将更新根验证的数据字典，而不是单个键。

``` python
>>> data = {
...     "label": "Second Example",
...     "coordinates": {
...         "x": 3,
...         "y": 4,
...     }
... }
>>> in_serializer = DataPointSerializer(data=data)
>>> in_serializer.is_valid()
True
>>> in_serializer.validated_data
OrderedDict([('label', 'Second Example'),
             ('y_coordinate', 4),
             ('x_coordinate', 3)])
```

为了完整性，我们再次做同样的事情，但是使用上面建议的嵌套序列化方法：

``` python
class NestedCoordinateSerializer(serializers.Serializer):
    x = serializers.IntegerField(source='x_coordinate')
    y = serializers.IntegerField(source='y_coordinate')


class DataPointSerializer(serializers.ModelSerializer):
    coordinates = NestedCoordinateSerializer(source='*')

    class Meta:
        model = DataPoint
        fields = ['label', 'coordinates']
```

这里，我们在 `IntegerField` 声明中处理目标和源属性对（`x` 和 `x_coordinate`，`y` 和 `y_coordinate` ）之间的映射。这是使用了 `source ='*'` 的 `NestedCoordinateSerializer` 。

新的 `DataPointSerializer` 展现出与自定义字段方法相同的行为。

序列化：

``` python
>>> out_serializer = DataPointSerializer(instance)
>>> out_serializer.data
ReturnDict([('label', 'testing'),
            ('coordinates', OrderedDict([('x', 1), ('y', 2)]))])
```

反序列化：

``` python
>>> in_serializer = DataPointSerializer(data=data)
>>> in_serializer.is_valid()
True
>>> in_serializer.validated_data
OrderedDict([('label', 'still testing'),
             ('x_coordinate', 3),
             ('y_coordinate', 4)])
```

虽然没有编写验证，但是可以使用内置的验证方式：

``` python
>>> invalid_data = {
...     "label": "still testing",
...     "coordinates": {
...         "x": 'a',
...         "y": 'b',
...     }
... }
>>> invalid_serializer = DataPointSerializer(data=invalid_data)
>>> invalid_serializer.is_valid()
False
>>> invalid_serializer.errors
ReturnDict([('coordinates',
             {'x': ['A valid integer is required.'],
              'y': ['A valid integer is required.']})])
```

出于这个原因，可以首先尝试嵌套序列化类方法。当嵌套序列化类变得不可行或过于复杂时，可以使用自定义字段方法。


[cite]: https://docs.djangoproject.com/en/stable/ref/forms/api/#django.forms.Form.cleaned_data
[html-and-forms]: ../topics/html-and-forms.md
[FILE_UPLOAD_HANDLERS]: https://docs.djangoproject.com/en/stable/ref/settings/#std:setting-FILE_UPLOAD_HANDLERS
[strftime]: https://docs.python.org/3/library/datetime.html#strftime-and-strptime-behavior
[iso8601]: https://www.w3.org/TR/NOTE-datetime
[drf-compound-fields]: https://drf-compound-fields.readthedocs.io
[drf-extra-fields]: https://github.com/Hipo/drf-extra-fields
[djangorestframework-recursive]: https://github.com/heywbj/django-rest-framework-recursive
[django-rest-framework-gis]: https://github.com/djangonauts/django-rest-framework-gis
[django-rest-framework-hstore]: https://github.com/djangonauts/django-rest-framework-hstore
[django-hstore]: https://github.com/djangonauts/django-hstore
[python-decimal-rounding-modes]: https://docs.python.org/3/library/decimal.html#rounding-modes
