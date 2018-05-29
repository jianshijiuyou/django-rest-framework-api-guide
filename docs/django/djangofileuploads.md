# 文件上传

当 Django 处理文件上传时，文件数据最终放置在 `request.FILES` 中（有关 `request` 对象的更多信息，请参阅[请求和响应对象的文档](https://docs.djangoproject.com/en/2.0/ref/request-response/)）。本文档介绍了文件如何存储在磁盘和内存中，以及如何自定义默认行为。

!> 警告：如果您接受来自不可信用户的上传内容，则存在安全风险！有关缓解详细信息，请参阅[用户上传内容](https://docs.djangoproject.com/en/2.0/topics/security/#user-uploaded-content-security)中的安全指南主题。

## 基本文件上传

考虑一个包含 `FileField` 的简单表单：

``` python
from django import forms

class UploadFileForm(forms.Form):
    title = forms.CharField(max_length=50)
    file = forms.FileField()
```

处理此表单的视图将接收 `request.FILES` 中的文件数据，它是一个包含表单中每个 `FileField`（或 `ImageField` 或其他 `FileField` 子类）的键的字典。因此，上述表单中的数据可以作为 `request.FILES['file']` 访问。

请注意，如果请求方法是 `POST`，并且发布请求的 `<form>` 具有属性 `enctype="multipart/form-data"` ，那么 `request.FILES` 将仅包含数据。否则，`request.FILES` 将为空。

大多数情况下，您只需将 `request` 中的文件数据传递到表单中，如将绑定上传文件绑定到表单中所述。这看起来像这样：

`views.py`

``` python
from django.http import HttpResponseRedirect
from django.shortcuts import render
from .forms import UploadFileForm

# Imaginary function to handle an uploaded file.
from somewhere import handle_uploaded_file

def upload_file(request):
    if request.method == 'POST':
        form = UploadFileForm(request.POST, request.FILES)
        if form.is_valid():
            handle_uploaded_file(request.FILES['file'])
            return HttpResponseRedirect('/success/url/')
    else:
        form = UploadFileForm()
    return render(request, 'upload.html', {'form': form})
```

请注意，我们必须将 `request.FILES` 传递给 form 的构造函数;这样才能将文件数据绑定到表单中。

以下是处理上传文件的常用方法：

``` python
def handle_uploaded_file(f):
    with open('some/file/name.txt', 'wb+') as destination:
        for chunk in f.chunks():
            destination.write(chunk)
```

使用 `UploadedFile.chunks()` 而不是使用 `read()` 循环可确保大文件不会压倒系统内存。

`UploadedFile` 对象上还有其他一些方法和属性;请参阅 [UploadedFile](https://docs.djangoproject.com/en/2.0/ref/files/uploads/#django.core.files.uploadedfile.UploadedFile) 以获取完整的参考。

### 使用模型处理上传的文件

如果您使用 `FileField` 将文件保存在 `Model` 上，则使用 `ModelForm` 可以使此过程更加轻松。调用 `form.save()` 时，文件对象将被保存到相应 `FileField` 的 `upload_to` 参数指定的位置：

``` python
from django.http import HttpResponseRedirect
from django.shortcuts import render
from .forms import ModelFormWithFileField

def upload_file(request):
    if request.method == 'POST':
        form = ModelFormWithFileField(request.POST, request.FILES)
        if form.is_valid():
            # file is saved
            form.save()
            return HttpResponseRedirect('/success/url/')
    else:
        form = ModelFormWithFileField()
    return render(request, 'upload.html', {'form': form})
```

如果您手动构建对象，则可以简单地将 `request.FILES` 中的文件对象分配给模型中的文件字段：

``` python
from django.http import HttpResponseRedirect
from django.shortcuts import render
from .forms import UploadFileForm
from .models import ModelWithFileField

def upload_file(request):
    if request.method == 'POST':
        form = UploadFileForm(request.POST, request.FILES)
        if form.is_valid():
            instance = ModelWithFileField(file_field=request.FILES['file'])
            instance.save()
            return HttpResponseRedirect('/success/url/')
    else:
        form = UploadFileForm()
    return render(request, 'upload.html', {'form': form})
```

### 上传多个文件

如果要使用一个表单字段上载多个文件，请设置字段的 widget 的 `multiple` HTML 属性：

`forms.py`

``` python
from django import forms

class FileFieldForm(forms.Form):
    file_field = forms.FileField(widget=forms.ClearableFileInput(attrs={'multiple': True}))
```

然后重写 `FormView` 子类的 `post` 方法来处理多个文件上传：

`views.py`

``` python
from django.views.generic.edit import FormView
from .forms import FileFieldForm

class FileFieldView(FormView):
    form_class = FileFieldForm
    template_name = 'upload.html'  # Replace with your template.
    success_url = '...'  # Replace with your URL or reverse().

    def post(self, request, *args, **kwargs):
        form_class = self.get_form_class()
        form = self.get_form(form_class)
        files = request.FILES.getlist('file_field')
        if form.is_valid():
            for f in files:
                ...  # Do something with each file.
            return self.form_valid(form)
        else:
            return self.form_invalid(form)
```

## 上传处理程序

当用户上传文件时，Django 将文件数据传递给上传处理程序 - 一个处理文件数据上传的小类。上传处理程序最初是在 `FILE_UPLOAD_HANDLERS` setting 中定义的，该设置默认为：

``` python
["django.core.files.uploadhandler.MemoryFileUploadHandler",
 "django.core.files.uploadhandler.TemporaryFileUploadHandler"]
```

`MemoryFileUploadHandler` 和 `TemporaryFileUploadHandler` 一起提供了将小文件读入内存和将大文件读入磁盘的 Django 默认文件上载行为。

您可以编写自定义处理程序来自定义 Django 如何处理文件。例如，您可以使用自定义处理程序来执行用户级配额，即时压缩数据，呈现进度条，甚至直接将数据发送到另一个存储位置，而无需将其存储在本地。请参阅[编写自定义上传处理程序](https://docs.djangoproject.com/en/2.0/ref/files/uploads/#custom-upload-handlers)以获取有关如何自定义或完全替代上载行为的详细信息

### 上传的数据存储在哪里

在保存上传的文件之前，数据需要存储在某个地方。

默认情况下，如果上传的文件小于 2.5 兆字节，Django 将在内存中保存上传的全部内容。这意味着保存文件只涉及从内存读取和写入磁盘，因此速度非常快。

但是，如果上传的文件太大，Django 会将上传的文件写入存储在系统临时目录中的临时文件。在 类Unix 平台上，这意味着您可以期望 Django 生成一个名为 `/tmp/tmpzfp6I6.upload` 的文件。如果上传量足够大，您可以在 Django 将数据流式传输到磁盘上时观看此文件的大小。

这些细节 - 2.5 兆字节;`/tmp`;等等 - 只是 “合理的默认设置”，可以按照下一节中的描述进行定制。

### 更改上传处理程序行为

有几个设置可以控制 Django 的文件上传行为。详情请参阅[文件上传设置](https://docs.djangoproject.com/en/2.0/ref/settings/#file-upload-settings)。

### 即时修改上传处理程序

有时特定视图需要不同的上传行为。在这些情况下，您可以通过修改 `request.upload_handlers` 来基于每个请求覆盖上载处理程序。默认情况下，该列表将包含 `FILE_UPLOAD_HANDLERS` 给出的上传处理程序，但您可以像修改任何其他列表一样修改列表。

例如，假设你已经编写了一个 `ProgressBarUploadHandler`，它提供了关于上传进度到某种 AJAX 小部件的反馈。您可以将这个处理程序添加到上传处理程序中，如下所示：

``` python
request.upload_handlers.insert(0, ProgressBarUploadHandler(request))
```

你可能想在这种情况下使用 `list.insert()`（而不是 `append()`），因为进度条处理程序需要在任何其他处理程序之前运行。请记住，上传处理程序按顺序处理。

如果你想完全替换上传处理程序，你可以分配一个新的列表：

``` python
request.upload_handlers = [ProgressBarUploadHandler(request)]
```

# 文件管理

本文档描述了用户上传的文件的 Django 文件访问 API。较低级别的 API 足够普遍，您可以将它们用于其他目的。如果您想处理 “静态文件”（JS，CSS等），请参阅[管理静态文件](https://docs.djangoproject.com/zh-hans/2.0/howto/static-files/)（例如图像，JavaScript，CSS）。

默认情况下，Django 使用 `MEDIA_ROOT` 和 `MEDIA_URL` setting 在本地存储文件。下面的例子假定你正在使用这些默认值。

但是，Django 提供了编写自定义文件存储系统的方法，使您可以完全自定义 Django 存储文件的位置和方式。本文档的后半部分描述了这些存储系统的工作原理。

## 在模型中使用文件

当您使用 `FileField` 或 `ImageField` 时，Django 提供了一组可用于处理该文件的 API。

考虑使用 `ImageField` 存储照片的以下模型：

``` python
from django.db import models

class Car(models.Model):
    name = models.CharField(max_length=255)
    price = models.DecimalField(max_digits=5, decimal_places=2)
    photo = models.ImageField(upload_to='cars')
```

任何 `Car` 实例都会有一个 `photo` 属性，您可以使用它来获取所附照片的详细信息：

``` python
>>> car = Car.objects.get(name="57 Chevy")
>>> car.photo
<ImageFieldFile: chevy.jpg>
>>> car.photo.name
'cars/chevy.jpg'
>>> car.photo.path
'/media/cars/chevy.jpg'
>>> car.photo.url
'http://media.example.com/cars/chevy.jpg'
```

这个例子中的 `car.photo` 是一个 `File` 对象，这意味着它拥有下面描述的所有方法和属性。

!> 该文件作为将模型保存在数据库中的一部分进行保存，因此在保存模型之前，不能依赖磁盘上使用的实际文件名。

例如，可以通过将文件名设置为相对于文件存储位置的路径（如果使用默认 `FileSystemStorage`，则使用 `MEDIA_ROOT`）来更改文件名：

``` python
>>> import os
>>> from django.conf import settings
>>> initial_path = car.photo.path
>>> car.photo.name = 'cars/chevy_ii.jpg'
>>> new_path = settings.MEDIA_ROOT + car.photo.name
>>> # Move the file on the filesystem
>>> os.rename(initial_path, new_path)
>>> car.save()
>>> car.photo.path
'/media/cars/chevy_ii.jpg'
>>> car.photo.path == new_path
True
```

## File 对象

在内部，Django 在需要表示文件时使用 `django.core.files.File` 实例。

大多数情况下，您只需使用 Django 提供给您的文件（即，如上所述附加到模型的文件，或者可能是上传的文件）。

如果您需要自己构建 `File`，最简单的方法是使用 Python 内置 `file` 对象创建一个文件对象：

``` python
>>> from django.core.files import File

# Create a Python file object using open()
>>> f = open('/path/to/hello.world', 'w')
>>> myfile = File(f)
```

现在您可以使用 `File` 类的任何已记录的属性和方法。

请注意，以这种方式创建的文件不会自动关闭。可以使用以下方法自动关闭文件：

``` python
>>> from django.core.files import File

# Create a Python file object using open() and the with statement
>>> with open('/path/to/hello.world', 'w') as f:
...     myfile = File(f)
...     myfile.write('Hello World')
...
>>> myfile.closed
True
>>> f.closed
True
```

当访问大量对象的循环中的文件字段时，关闭文件尤为重要。如果在访问文件后没有手动关闭文件，则可能会出现文件描述符用尽的风险。这可能会导致以下错误：

``` python
IOError: [Errno 24] Too many open files
```

## 文件存储

在幕后，Django 决定如何以及在何处将文件存储到文件存储系统。这是实际理解文件系统，打开和读取文件等内容的对象。

Django 的默认文件存储由 `DEFAULT_FILE_STORAGE` 设置给出;如果您没有明确提供存储系统，则将使用该存储系统。

有关内置默认文件存储系统的详细信息，请参阅以下内容，并参阅编[写一个自定义存储系统](https://docs.djangoproject.com/zh-hans/2.0/howto/custom-file-storage/)以获取有关编写您自己的文件存储系统的信息。

### 存储对象

尽管大部分时间您都希望使用 `File` 对象（它委托该文件的正确存储），但您可以直接使用文件存储系统。您可以创建一些自定义文件存储类的实例，或者 - 通常更有用 - 您可以使用全局默认存储系统：

``` python
>>> from django.core.files.base import ContentFile
>>> from django.core.files.storage import default_storage

>>> path = default_storage.save('/path/to/file', ContentFile('new content'))
>>> path
'/path/to/file'

>>> default_storage.size(path)
11
>>> default_storage.open(path).read()
'new content'

>>> default_storage.delete(path)
>>> default_storage.exists(path)
False
```

请参阅 [文件存储 API](https://docs.djangoproject.com/zh-hans/2.0/ref/files/storage/)。

### 内置的文件系统存储类

Django 附带一个 `django.core.files.storage.FileSystemStorage` 类，它实现了基本的本地文件系统文件存储。

例如，下面的代码会将上传的文件存储在 `/media/photos` 下，而不管您的 `MEDIA_ROOT` 设置是什么：

``` python
from django.core.files.storage import FileSystemStorage
from django.db import models

fs = FileSystemStorage(location='/media/photos')

class Car(models.Model):
    ...
    photo = models.ImageField(storage=fs)
```

[自定义存储系统](https://docs.djangoproject.com/zh-hans/2.0/howto/custom-file-storage/)的工作方式相同：您可以将它们作为 `storage` 参数传递给 `FileField`。