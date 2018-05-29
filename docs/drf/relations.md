> [官方原文链接](http://www.django-rest-framework.org/api-guide/relations/)  



# Serializer 关系

关系字段用于表示模型关系。 它们可以应用于 `ForeignKey`，`ManyToManyField` 和 `OneToOneField` 关系，反向关系以及 `GenericForeignKey` 等自定义关系。



> **注意：** 关系字段在 `relations.py` 中声明，但按照惯例，你应该从 `serializers` 模块导入它们，使用 `from rest_framework import serializers` 引入，并像 `serializers.<FieldName>` 这样引用字段。



#### 检查关系。

在使用 `ModelSerializer` 类时，将自动为你生成序列化字段和关系。检查这些自动生成的字段可以学习如何定制关系风格。

为此，使用 `python manage.py shell` 打开 Django shell，然后导入序列化类，实例化它并打印对象表示形式...

``` python
>>> from myapp.serializers import AccountSerializer
>>> serializer = AccountSerializer()
>>> print repr(serializer)  # Or `print(repr(serializer))` in Python 3.x.
AccountSerializer():
    id = IntegerField(label='ID', read_only=True)
    name = CharField(allow_blank=True, max_length=100, required=False)
    owner = PrimaryKeyRelatedField(queryset=User.objects.all())
```

# API 参考

为了解释各种类型的关系字段，我们将为我们的示例使用几个简单的模型。我们的模型将使用音乐专辑，以及每张专辑中列出的曲目。

``` python
class Album(models.Model):
    album_name = models.CharField(max_length=100)
    artist = models.CharField(max_length=100)

class Track(models.Model):
    album = models.ForeignKey(Album, related_name='tracks', on_delete=models.CASCADE)
    order = models.IntegerField()
    title = models.CharField(max_length=100)
    duration = models.IntegerField()

    class Meta:
        unique_together = ('album', 'order')
        ordering = ['order']

    def __unicode__(self):
        return '%d: %s' % (self.order, self.title)
```

## StringRelatedField

`StringRelatedField` 用于使用 `__unicode__` 方法表示关系。

例如，下面的序列化类。

``` python
class AlbumSerializer(serializers.ModelSerializer):
    tracks = serializers.StringRelatedField(many=True)

    class Meta:
        model = Album
        fields = ('album_name', 'artist', 'tracks')
```

将序列化为以下形式。

```
{
    'album_name': 'Things We Lost In The Fire',
    'artist': 'Low',
    'tracks': [
        '1: Sunflower',
        '2: Whitetail',
        '3: Dinosaur Act',
        ...
    ]
}
```

该字段是只读的。

**参数**:

* `many` - 如果是一对多的关系，就将此参数设置为 `True`.

## PrimaryKeyRelatedField

`PrimaryKeyRelatedField` 用于使用其主键表示关系。

例如，以下序列化类：

``` python
class AlbumSerializer(serializers.ModelSerializer):
    tracks = serializers.PrimaryKeyRelatedField(many=True, read_only=True)

    class Meta:
        model = Album
        fields = ('album_name', 'artist', 'tracks')
```

将序列化为这样的表示：

```
{
    'album_name': 'Undun',
    'artist': 'The Roots',
    'tracks': [
        89,
        90,
        91,
        ...
    ]
}
```

默认情况下，该字段是可读写的，但您可以使用 `read_only` 标志更改此行为。

**参数**:

* `queryset` - 验证字段输入时用于模型实例查询的查询集。必须显式地设置查询集，或设置 `read_only=True`。
* `many` - 如果应用于一对多关系，则应将此参数设置为 `True`。
* `allow_null` - 如果设置为 `True`，那么该字段将接受 `None` 值或可为空的关系的空字符串。默认为 `False`。
* `pk_field` - 设置为一个字段来控制主键值的序列化/反序列化。例如， `pk_field=UUIDField(format='hex')` 会将 UUID 主键序列化为其紧凑的十六进制表示形式。


## HyperlinkedRelatedField

`HyperlinkedRelatedField` 用于使用超链接来表示关系。

例如，以下序列化类：

``` python
class AlbumSerializer(serializers.ModelSerializer):
    tracks = serializers.HyperlinkedRelatedField(
        many=True,
        read_only=True,
        view_name='track-detail'
    )

    class Meta:
        model = Album
        fields = ('album_name', 'artist', 'tracks')
```

将序列化为这样的表示：

```
{
    'album_name': 'Graceland',
    'artist': 'Paul Simon',
    'tracks': [
        'http://www.example.com/api/tracks/45/',
        'http://www.example.com/api/tracks/46/',
        'http://www.example.com/api/tracks/47/',
        ...
    ]
}
```

默认情况下，该字段是可读写的，但您可以使用 `read_only` 标志更改此行为。

---

**注意**：该字段是为映射到接受单个 URL 关键字参数的 URL 的对象而设计的，如使用 `lookup_field` 和 `lookup_url_kwarg` 参数设置的对象。

这适用于包含单个主键或 slug 参数作为 URL 一部分的 URL。

如果需要更复杂的超链接表示，你需要自定义该字段，稍后会详解。

---

**参数**：

* `view_name` - 用作关系目标的视图名称。如果你使用的是标准路由器类，则这将是一个格式为 `<modelname>-detail` 的字符串。**必填**.
* `queryset` - 验证字段输入时用于模型实例查询的查询集。必须显式地设置查询集，或设置 `read_only=True`。
* `many` - 如果应用于一对多关系，则应将此参数设置为 `True`。
* `allow_null` - 如果设置为 `True`，那么该字段将接受 `None` 值或可为空的关系的空字符串。默认为 `False`。
* `lookup_field` - 用于查找的目标字段。对应于引用视图上的 URL 关键字参数。默认是 `'pk'`.
* `lookup_url_kwarg` - 与查找字段对应的 URL conf 中定义的关键字参数的名称。默认使用与 `lookup_field` 相同的值。
* `format` - 如果使用格式后缀，则超链接字段将使用与目标相同的格式后缀，除非使用 `format` 参数进行覆盖。

## SlugRelatedField

`SlugRelatedField` 用于使用目标上的字段来表示关系。

例如，以下序列化类：

``` python
class AlbumSerializer(serializers.ModelSerializer):
    tracks = serializers.SlugRelatedField(
        many=True,
        read_only=True,
        slug_field='title'
     )

    class Meta:
        model = Album
        fields = ('album_name', 'artist', 'tracks')
```

将序列化为这样的表示：

``` python
{
    'album_name': 'Dear John',
    'artist': 'Loney Dear',
    'tracks': [
        'Airport Surroundings',
        'Everything Turns to You',
        'I Was Only Going Out',
        ...
    ]
}
```

默认情况下，该字段是可读写的，但您可以使用 `read_only` 标志更改此行为。

将 `SlugRelatedField` 用作读写字段时，通常需要确保 slug 字段与 `unique=True` 的模型字段相对应。

**参数**：

* `slug_field` - 用来表示目标的字段。这应该是唯一标识给定实例的字段。例如， `username`。**必填**
* `queryset` - 验证字段输入时用于模型实例查询的查询集。必须显式地设置查询集，或设置 `read_only=True`。
* `many` - 如果应用于一对多关系，则应将此参数设置为 `True`。
* `allow_null` - 如果设置为 `True`，那么该字段将接受 `None` 值或可为空的关系的空字符串。默认为 `False`。

## HyperlinkedIdentityField

此字段可以作为身份关系应用，例如 `HyperlinkedModelSerializer` 上的 `'url'` 字段。它也可以用于对象的属性。例如，以下序列化类：

``` python
class AlbumSerializer(serializers.HyperlinkedModelSerializer):
    track_listing = serializers.HyperlinkedIdentityField(view_name='track-list')

    class Meta:
        model = Album
        fields = ('album_name', 'artist', 'track_listing')
```

将序列化为这样的表示：

``` python
{
    'album_name': 'The Eraser',
    'artist': 'Thom Yorke',
    'track_listing': 'http://www.example.com/api/track_list/12/',
}
```

该字段始终为只读。

**参数**：

* `view_name` - 用作关系目标的视图名称。如果你使用的是标准路由器类，则这将是一个格式为 `<modelname>-detail` 的字符串。**必填**。
* `lookup_field` - 用于查找的目标字段。对应于引用视图上的 URL 关键字参数。默认是 `'pk'`。
* `lookup_url_kwarg` - 与查找字段对应的 URL conf 中定义的关键字参数的名称。默认使用与 `lookup_field` 相同的值。
* `format` - 如果使用格式后缀，则超链接字段将使用与目标相同的格式后缀，除非使用 `format` 参数进行覆盖。

---

# 嵌套关系

嵌套关系可以通过使用序列化类作为字段来表达。

如果该字段用于表示一对多关系，则应将 `many=True` 标志添加到序列化字段。

## 举个栗子

例如，以下序列化类：

``` python
class TrackSerializer(serializers.ModelSerializer):
    class Meta:
        model = Track
        fields = ('order', 'title', 'duration')

class AlbumSerializer(serializers.ModelSerializer):
    tracks = TrackSerializer(many=True, read_only=True)

    class Meta:
        model = Album
        fields = ('album_name', 'artist', 'tracks')
```

将序列化为这样的嵌套表示：

``` python
>>> album = Album.objects.create(album_name="The Grey Album", artist='Danger Mouse')
>>> Track.objects.create(album=album, order=1, title='Public Service Announcement', duration=245)
<Track: Track object>
>>> Track.objects.create(album=album, order=2, title='What More Can I Say', duration=264)
<Track: Track object>
>>> Track.objects.create(album=album, order=3, title='Encore', duration=159)
<Track: Track object>
>>> serializer = AlbumSerializer(instance=album)
>>> serializer.data
{
    'album_name': 'The Grey Album',
    'artist': 'Danger Mouse',
    'tracks': [
        {'order': 1, 'title': 'Public Service Announcement', 'duration': 245},
        {'order': 2, 'title': 'What More Can I Say', 'duration': 264},
        {'order': 3, 'title': 'Encore', 'duration': 159},
        ...
    ],
}
```

## 可写嵌套序列化类

默认情况下，嵌套序列化类是只读的。如果要支持对嵌套序列化字段的写操作，则需要创建 `create()` 和/或 `update()` 方法，以明确指定应如何保存子关系。

``` python
class TrackSerializer(serializers.ModelSerializer):
    class Meta:
        model = Track
        fields = ('order', 'title', 'duration')

class AlbumSerializer(serializers.ModelSerializer):
    tracks = TrackSerializer(many=True)

    class Meta:
        model = Album
        fields = ('album_name', 'artist', 'tracks')

    def create(self, validated_data):
        tracks_data = validated_data.pop('tracks')
        album = Album.objects.create(**validated_data)
        for track_data in tracks_data:
            Track.objects.create(album=album, **track_data)
        return album

>>> data = {
    'album_name': 'The Grey Album',
    'artist': 'Danger Mouse',
    'tracks': [
        {'order': 1, 'title': 'Public Service Announcement', 'duration': 245},
        {'order': 2, 'title': 'What More Can I Say', 'duration': 264},
        {'order': 3, 'title': 'Encore', 'duration': 159},
    ],
}
>>> serializer = AlbumSerializer(data=data)
>>> serializer.is_valid()
True
>>> serializer.save()
<Album: Album object>
```

---

# 自定义关系字段

在极少数情况下，现有的关系类型都不符合您需要的表示形式，你可以实现一个完全自定义的关系字段，该字段准确描述应该如何从模型实例生成输出表示。

要实现自定义关系字段，您应该重写 `RelatedField`，并实现 `.to_representation(self, value)` 方法。此方法将字段的目标作为 `value` 参数，并返回应用于序列化目标的表示。`value` 参数通常是一个模型实例。

如果要实现读写关系字段，则还必须实现 `.to_internal_value(self, data)` 方法。

要提供基于 `context` 的动态查询集，还可以覆盖 `.get_queryset(self)`，而不是在类上指定 `.queryset` 或初始化该字段。

## 举个栗子

例如，我们可以定义一个关系字段，使用它的顺序，标题和持续时间将音轨序列化为自定义字符串表示。

``` python
import time

class TrackListingField(serializers.RelatedField):
    def to_representation(self, value):
        duration = time.strftime('%M:%S', time.gmtime(value.duration))
        return 'Track %d: %s (%s)' % (value.order, value.name, duration)

class AlbumSerializer(serializers.ModelSerializer):
    tracks = TrackListingField(many=True)

    class Meta:
        model = Album
        fields = ('album_name', 'artist', 'tracks')
```

将序列化为这样的表示：

``` python
{
    'album_name': 'Sometimes I Wish We Were an Eagle',
    'artist': 'Bill Callahan',
    'tracks': [
        'Track 1: Jim Cain (04:39)',
        'Track 2: Eid Ma Clack Shaw (04:19)',
        'Track 3: The Wind and the Dove (04:34)',
        ...
    ]
}
```

---

# 自定义超链接字段

在某些情况下，您可能需要自定义超链接字段的行为，以表示需要多个查询字段的 URL。

您可以通过继承 `HyperlinkedRelatedField` 来实现此目的。有两个可以被覆盖的方法：

**get_url(self, obj, view_name, request, format)**

`get_url` 方法用于将对象实例映射到其 URL 表示。

如果 `view_name` 和 `lookup_field` 属性未配置为正确匹配 URL conf，可能会引发 `NoReverseMatch` 。

**get_object(self, queryset, view_name, view_args, view_kwargs)**

如果您想支持可写的超链接字段，那么您还需要重写 `get_object`，以便将传入的 URL 映射回它们表示的对象。对于只读超链接字段，不需要重写此方法。

此方法的返回值应该是与匹配的 URL conf 参数对应的对象。

可能会引发 `ObjectDoesNotExist` 异常。

## 举个栗子

假设我们有一个带有两个关键字参数的 customer 对象的 URL，如下所示：

```
/api/<organization_slug>/customers/<customer_pk>/
```

这没办法用仅接受单个查找字段的默认实现来表示。

在这种情况下，我们需要继承 `HyperlinkedRelatedField` 并重写其中的方法来获得我们想要的行为：

``` python
from rest_framework import serializers
from rest_framework.reverse import reverse

class CustomerHyperlink(serializers.HyperlinkedRelatedField):
    # We define these as class attributes, so we don't need to pass them as arguments.
    view_name = 'customer-detail'
    queryset = Customer.objects.all()

    def get_url(self, obj, view_name, request, format):
        url_kwargs = {
            'organization_slug': obj.organization.slug,
            'customer_pk': obj.pk
        }
        return reverse(view_name, kwargs=url_kwargs, request=request, format=format)

    def get_object(self, view_name, view_args, view_kwargs):
        lookup_kwargs = {
           'organization__slug': view_kwargs['organization_slug'],
           'pk': view_kwargs['customer_pk']
        }
        return self.get_queryset().get(**lookup_kwargs)
```

请注意，如果您想将此风格与通用视图一起使用，那么您还需要覆盖视图上的 `.get_object` 以获得正确的查找行为。

一般来说，我们建议尽可能在 API 表示方式下使用平面风格，但嵌套 URL 风格在适度使用时也是合理的。

---

# 进一步说明

## `queryset` 参数

`queryset` 参数只对可写关系字段是必需的，在这种情况下，它用于执行模型实例查找，该查找从基本用户输入映射到模型实例。

在 2.x 版本中，如果正在使用 `ModelSerializer` 类，则序列化类有时会自动确定 `queryset` 参数。

此行为现​​在替换为始终为可写关系字段使用显式 `queryset` 参数。

这样做可以减少 `ModelSerializer` 提供的隐藏 “魔术” 数量（指 `ModelSerializer` 在内部帮我们完成的工作），使字段的行为更加清晰，并确保在使用 `ModelSerializer` 快捷方式（高度封装过，使用简单）或使用完全显式的 `Serializer` 类之间转换是微不足道的。

## 自定义 HTML 显示

模型内置的 `__str__` 方法用来生成用于填充 `choices` 属性的对象的字符串表示形式。这些 choices 用于在可浏览的 API 中填充选择的 HTML input。

要为这些 input 提供自定义表示，请重写 `RelatedField` 子类的 `display_value()` 方法。这个方法将接收一个模型对象，并且应该返回一个适合表示它的字符串。例如：

``` python
class TrackPrimaryKeyRelatedField(serializers.PrimaryKeyRelatedField):
    def display_value(self, instance):
        return 'Track: %s' % (instance.title)
```

## Select field cutoffs

在渲染可浏览的 API 关系字段时，默认只显示最多 1000 个可选 item。如果存在更多项目，则会显示 "More than 1000 items…" 的 disabled 选项。

此行为旨在防止由于显示大量关系而导致模板无法在可接受的时间范围内完成渲染。

有两个关键字参数可用于控制此行为：

- `html_cutoff` - 设置 HTML select 下拉菜单中显示的选项的最大数量。设置为 `None` 可以禁用任何限制。默认为 `1000`。
- `html_cutoff_text` - 设置一个文本字符串，在 HTML select 下拉菜单超出最大显示数量时显示。默认是 `"More than {count} items…"`

你还可以在 settings 中用 `HTML_SELECT_CUTOFF` 和 `HTML_SELECT_CUTOFF_TEXT` 来全局控制这些设置。

在强制执行 cutoff 的情况下，您可能希望改为在 HTML 表单中使用简单的 input 字段。你可以使用 `style` 关键字参数来做到这一点。例如：

``` python
assigned_to = serializers.SlugRelatedField(
   queryset=User.objects.all(),
   slug_field='username',
   style={'base_template': 'input.html'}
)
```

## 反向关系

请注意，反向关系不会自动包含在 `ModelSerializer` 和 `HyperlinkedModelSerializer` 类中。要包含反向关系，您必须明确将其添加到字段列表中。例如：

``` python
class AlbumSerializer(serializers.ModelSerializer):
    class Meta:
        fields = ('tracks', ...)
```

通常需要确保已经在关系上设置了适当的 `related_name` 参数，可以将其用作字段名称。例如：

``` python
class Track(models.Model):
    album = models.ForeignKey(Album, related_name='tracks', on_delete=models.CASCADE)
    ...
```

如果你还没有为反向关系设置相关名称，则需要在 `fields` 参数中使用自动生成的相关名称。例如：

``` python
class AlbumSerializer(serializers.ModelSerializer):
    class Meta:
        fields = ('track_set', ...)
```

## 通用关系

如果要序列化通用外键，则需要自定义字段，以明确确定如何序列化关系。

例如，给定一个以下模型的标签，该标签与其他任意模型具有通用关系：

``` python
class TaggedItem(models.Model):
    """
    Tags arbitrary model instances using a generic relation.

    See: https://docs.djangoproject.com/en/stable/ref/contrib/contenttypes/
    """
    tag_name = models.SlugField()
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.PositiveIntegerField()
    tagged_object = GenericForeignKey('content_type', 'object_id')

    def __unicode__(self):
        return self.tag_name
```

以下两种模式可以用相关的标签：

``` python
class Bookmark(models.Model):
    """
    A bookmark consists of a URL, and 0 or more descriptive tags.
    """
    url = models.URLField()
    tags = GenericRelation(TaggedItem)


class Note(models.Model):
    """
    A note consists of some text, and 0 or more descriptive tags.
    """
    text = models.CharField(max_length=1000)
    tags = GenericRelation(TaggedItem)
```

我们可以定义一个可用于序列化标签实例的自定义字段，并使用每个实例的类型来确定它应该如何序列化。

``` python
class TaggedObjectRelatedField(serializers.RelatedField):
    """
    A custom field to use for the `tagged_object` generic relationship.
    """

    def to_representation(self, value):
        """
        Serialize tagged objects to a simple textual representation.
        """
        if isinstance(value, Bookmark):
            return 'Bookmark: ' + value.url
        elif isinstance(value, Note):
            return 'Note: ' + value.text
        raise Exception('Unexpected type of tagged object')
```

如果你需要的关系具有嵌套表示，则可以在 `.to_representation()` 方法中使用所需的序列化类：

``` python
    def to_representation(self, value):
        """
        Serialize bookmark instances using a bookmark serializer,
        and note instances using a note serializer.
        """
        if isinstance(value, Bookmark):
            serializer = BookmarkSerializer(value)
        elif isinstance(value, Note):
            serializer = NoteSerializer(value)
        else:
            raise Exception('Unexpected type of tagged object')

        return serializer.data
```

请注意，使用 `GenericRelation` 字段表示的反向通用键可以使用常规关系字段类型进行序列化，因为关系中目标的类型总是已知的。

## 具有 Through 模型的 ManyToManyFields

默认情况下，将指定带有 `through` 模型的 `ManyToManyField` 的关系字段设置为只读。

如果你要明确指定一个指向具有 through 模型的 `ManyToManyField` 的关系字段，请确保将 `read_only` 设置为 `True` 。
