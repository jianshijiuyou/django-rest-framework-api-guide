> [官方原文链接](http://www.django-rest-framework.org/api-guide/validators/)  


# 验证器

大多数情况下，您在 REST framework 中处理验证时，只需依赖默认的字段验证，或者在序列化类或字段类上编写明确的验证方法。

但是，有时你会希望将验证逻辑放置到可重用组件中，以便在整个代码库中轻松地重用它。这可以通过使用验证器函数和验证器类来实现。

## REST framework 中的验证

Django REST framework 序列化器中的验证与 Django `ModelForm` 类中验证的工作方式有点不同。

使用 `ModelForm`，验证一部分在表单上执行，一部分在模型实例上执行。使用 REST framework ，验证完全在序列化类上执行。这是有优势的，原因如下：

* 它使问题适当的分离，让代码行为变的更加清晰。
* 使用快捷的 `ModelSerializer` 类和使用显式的 `Serializer` 类可以轻松切换。任何用于 `ModelSerializer` 的验证行为都很容易复制。
* 打印序列化类实例的 `repr` 将会显示它应用的验证规则。在模型实例上没有额外的隐藏验证行为（因为全在序列化类上）。

当你使用 `ModelSerializer` 时，所有验证都会自动为你处理。如果你想要改为使用 `Serializer` 类，那么你需要明确定义验证规则。

#### 举个栗子

作为 REST framework 如何使用显式验证的示例，我们将采用一个简单的模型类，该类具有唯一性约束的字段。

``` python
class CustomerReportRecord(models.Model):
    time_raised = models.DateTimeField(default=timezone.now, editable=False)
    reference = models.CharField(unique=True, max_length=20)
    description = models.TextField()
```

下面是一个基本的 `ModelSerializer` ，我们可以使用它来创建或更新 `CustomerReportRecord` 的实例：

``` python
class CustomerReportSerializer(serializers.ModelSerializer):
    class Meta:
        model = CustomerReportRecord
```

现在使用 `manage.py shell` 打开 Django shell

``` python
>>> from project.example.serializers import CustomerReportSerializer
>>> serializer = CustomerReportSerializer()
>>> print(repr(serializer))
CustomerReportSerializer():
    id = IntegerField(label='ID', read_only=True)
    time_raised = DateTimeField(read_only=True)
    reference = CharField(max_length=20, validators=[<UniqueValidator(queryset=CustomerReportRecord.objects.all())>])
    description = CharField(style={'type': 'textarea'})
```

这里有趣的是 `reference` 字段。我们可以看到唯一性约束由序列化字段上的验证器明确执行。

由于这种更明确的风格，REST framework 包含一些在核心 Django 中没有的验证器类。这些类在下面详细说明。

---

## UniqueValidator

该验证器可用于在模型字段上强制实施 `unique=True` 约束。它需要一个必需的参数和一个可选的 `messages` 参数：

* `queryset` *必须* - 这是验证唯一性的查询集。
* `message` - 验证失败时使用的错误消息。
* `lookup` - 用于查找已验证值的现有实例。默认为 `'exact'`。

这个验证器应该应用于序列化字段，如下所示：

``` python
from rest_framework.validators import UniqueValidator

slug = SlugField(
    max_length=100,
    validators=[UniqueValidator(queryset=BlogPost.objects.all())]
)
```

## UniqueTogetherValidator

此验证器可用于在模型实例上强制实施 `unique_together` 约束。它有两个必需的参数和一个可选的 `messages` 参数：

* `queryset` *必须* - 这是验证唯一性的查询集。
* `fields` *必须* - 一个存放字段名称的列表或者元组，这个集合必须是唯一的（意思是集合中的字段代表的一组值不能同时出现在两条数据中）。这些字段必须都是序列化类中的字段。
* `message` - 验证失败时使用的错误消息。

验证器应该应用于序列化类，如下所示：

``` python
from rest_framework.validators import UniqueTogetherValidator

class ExampleSerializer(serializers.Serializer):
    # ...
    class Meta:
        # ToDo items belong to a parent list, and have an ordering defined
        # by the 'position' field. No two items in a given list may share
        # the same position.
        validators = [
            UniqueTogetherValidator(
                queryset=ToDoItem.objects.all(),
                fields=('list', 'position')
            )
        ]
```

---

**注意**: `UniqueTogetherValidation` 类总是施加一个隐式约束，即它所应用的所有字段都是按需处理的。具有 `default` 值的字段是一个例外，因为它们总是提供一个值，即使在用户输入中省略了这个值。

---

## UniqueForDateValidator

## UniqueForMonthValidator

## UniqueForYearValidator

这些验证器可用于强制实施模型实例上的 `unique_for_date`，`unique_for_month` 和 `unique_for_year` 约束。他们有以下参数：

* `queryset` *必须* - 这是验证唯一性的查询集。
* `field` *必须* - 在给定日期范围内需要被验证唯一性的字段的名称。该字段必须是序列化类中的字段。
* `date_field` *必须* - 将用于确定唯一性约束的日期范围的字段名称。该字段必须是序列化类中的字段。
* `message` - 验证失败时使用的错误消息。

验证器应该应用于序列化类，如下所示：

``` python
from rest_framework.validators import UniqueForYearValidator

class ExampleSerializer(serializers.Serializer):
    # ...
    class Meta:
        # Blog posts should have a slug that is unique for the current year.
        validators = [
            UniqueForYearValidator(
                queryset=BlogPostItem.objects.all(),
                field='slug',
                date_field='published'
            )
        ]
```

> 我解释下，上面例子的意思是，在 `published` 日期所在的年份中，`slug` 字段的值必须唯一，注意，不是要和 `published` 完全相等的日期，而是年份相等。`unique_for_date`，`unique_for_month` 同理。


用于验证的日期字段应该始终存在于序列化类中。你不能简单地依赖模型类 `default=...`，因为默认值在验证运行之后才会生成。

你可能需要使用几种样式，具体取决于你希望 API 如何展现。如果你使用的是 `ModelSerializer` ，可能只需依赖 REST framework 为你生成的默认值，但如果你使用的是 `Serializer` 或需要更明确的控制，请使用下面演示的样式。

#### 与可写日期字段一起使用。

如果你希望日期字段是可写的，唯一值得注意的是你应该确保它始终可用于输入数据中，可以通过设置 `default` 参数或通过设置 `required=True` 来实现。

``` python
published = serializers.DateTimeField(required=True)
```

#### 与只读日期字段一起使用。

如果你希望日期字段可见，但用户无法编辑，请设置 `read_only=True` 并另外设置 `default=...` 参数。

``` python
published = serializers.DateTimeField(read_only=True, default=timezone.now)
```

该字段对用户不可写，但默认值仍将传递给 `validated_data`。

#### 与隐藏的日期字段一起使用。

如果你希望日期字段对用户完全隐藏，请使用 `HiddenField`。该字段类型不接受用户输入，而是始终将其默认值返回到序列化类中的 `validated_data`。

``` python
published = serializers.HiddenField(default=timezone.now)
```

---

**注意**: `UniqueFor<Range>Validation` 类总是施加一个隐式约束，即它所应用的所有字段都是按需处理的。具有 `default` 值的字段是一个例外，因为它们总是提供一个值，即使在用户输入中省略了这个值。

---

# Advanced field defaults

在序列化类的多个字段中应用的验证器有时不需要由 API 客户端提供的字段输入，但它可以用作验证器的输入。

有两种模式可能需要这种验证：

* 使用 `HiddenField` 。该字段将出现在 `validated_data` 中，但不会用在序列化输出表示中。
* 使用 `read_only=True` 的标准字段，同时也包含 `default=...` 参数。该字段将用于序列化输出表示中，但不能由用户直接设置。

REST framework 包含一些在这种情况下可能有用的默认值。

#### CurrentUserDefault

可用于表示当前用户的默认类。为了使用它，在实例化序列化类时，`'request'` 必须作为上下文字典的一部分提供。

``` python
owner = serializers.HiddenField(
    default=serializers.CurrentUserDefault()
)
```

#### CreateOnlyDefault

可用于在 create 操作期间仅设置默认参数的默认类。在 update 期间，该字段被省略。

它接受一个参数，这是在 create 操作期间应该使用的默认值或可调用参数。

``` python
created_at = serializers.DateTimeField(
    read_only=True,
    default=serializers.CreateOnlyDefault(timezone.now)
)
```

---

# 验证器的限制

有一些不明确的情况，你需要显示处理验证，而不是依赖 `ModelSerializer` 生成的默认序列化类。

在这些情况下，你可能希望通过为序列化类 `Meta.validators` 属性指定一个空列表来禁用自动生成的验证器。

## 可选字段

默认情况下 "unique together" 验证强制所有字段都是 `required=True`。在某些情况下，你可能希望显式将 `required=False` 应用于其中一个字段，在这种情况下，验证所需的行为是不明确的。

在这种情况下，你通常需要从序列化类中排除验证器，并且在 `.validate()` 方法中或在视图中显式地编写验证逻辑。

例如：

``` python
class BillingRecordSerializer(serializers.ModelSerializer):
    def validate(self, data):
        # Apply custom validation either here, or in the view.

    class Meta:
        fields = ('client', 'date', 'amount')
        extra_kwargs = {'client': {'required': 'False'}}
        validators = []  # Remove a default "unique together" constraint.
```

## 更新嵌套序列化类

将更新应用于现有实例时，唯一性验证器将从唯一性检查中排除当前实例。当前实例在唯一性检查的上下文中可用，因为它作为序列化程序中的一个属性存在，最初在实例化序列化类时已使用 `instance=...` 传递。

在嵌套序列化类上进行更新操作时，无法应用此排除，因为该实例不可用。

你可能又一次需要明确地从序列化类中移除验证器，并将验证约束的代码显式写入 `.validate()` 方法或视图中。

## 调试复杂的案例

如果你不确定 `ModelSerializer` 类的默认行为，那么运行 `manage.py shell` 并打印序列化类实例通常是一个好主意，以便你可以检查它自动生成的字段和验证器。

``` python
>>> serializer = MyComplexModelSerializer()
>>> print(serializer)
class MyComplexModelSerializer:
    my_fields = ...
```

还要记住，在复杂情况下，明确定义序列化类通常会更好，而不是依赖默认的 `ModelSerializer` 行为。虽然这样会写更多的代码，但确保了最终的行为更加透明。

---

# 编写自定义验证器

你可以使用 Django 现有的验证器，也可以编写自定义验证器。

## 基于函数

验证器可以是任何可调用对象，在失败时引发 `serializers.ValidationError`。

``` python
def even_number(value):
    if value % 2 != 0:
        raise serializers.ValidationError('This field must be an even number.')
```

#### 字段级验证

你可以通过向 `Serializer` 子类添加 `.validate_<field_name>`方法来指定自定义字段级验证。

## 基于类

要编写一个基于类的验证器，请使用 `__call__` 方法。基于类的验证器很有用，因为它们允许参数化和重用行为。

``` python
class MultipleOf(object):
    def __init__(self, base):
        self.base = base

    def __call__(self, value):
        if value % self.base != 0:
            message = 'This field must be a multiple of %d.' % self.base
            raise serializers.ValidationError(message)
```

#### 使用 `set_context()`

在一些高级的情况下，你可能想要在验证器中获取正在被验证的序列化字段。这时，你可以通过在基于类的验证器上声明 `set_context` 方法来完成此操作。

``` python
def set_context(self, serializer_field):
    # Determine if this is an update or a create operation.
    # In `__call__` we can then use that information to modify the validation behavior.
    self.is_update = serializer_field.parent.instance is not None
```