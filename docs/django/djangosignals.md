# 信号

Django 包含一个 “信号调度器”，它可以帮助解耦的应用程序在框架中的其他地方发生操作时得到通知。简而言之，信号允许某些发送者通知一组接收者发生了某些操作。当许多代码可能对相同事件感兴趣时，它们特别有用。

Django 提供了一组内置的信号，让用户代码可以通过 Django 自己通知某些操作。这些包括一些有用的通知：

* `django.db.models.signals.pre_save` & `django.db.models.signals.post_save`

    在模型的 `save()` 方法被调用之前或之后发送。

* `django.db.models.signals.pre_delete` & `django.db.models.signals.post_delete`

    在模型的 `delete()` 方法或 queryset 的 `delete()` 方法被调用之前或之后发送。

* `django.db.models.signals.m2m_changed`

    当模型上的 `ManyToManyField` 发生更改时发送。

* `django.core.signals.request_started` & `django.core.signals.request_finished`

    当 Django 开始或完成 HTTP 请求时发送。

## 监听信号

要接收信号，请使用 `Signal.connect()` 方法注册接收器函数。接收器函数在信号发送时被调用。

`Signal.connect(receiver, sender=None, weak=True, dispatch_uid=None)`

Parameters:

* `receiver` -- 将被连接到这个信号的回调函数。
* `sender` -- 指定一个特定的发送者接收信号。
* `weak` -- Django 默认将信号处理程序存储为弱引用。因此，如果您的接收器是本地函数，则可能会被回收。为了防止这种情况发生，当你调用信号的 `connect()` 方法时，传递 `weak=False`。
* `dispatch_uid` -- 在可能发送重复信号的情况下，信号接收器的唯一标识符。

让我们通过注册一个在每个 HTTP 请求完成后调用的信号来看看它是如何工作的。我们将连接到 `request_finished` 信号。

### 接收器函数

首先，我们需要定义一个接收器函数。接收器可以是任何 Python 函数或方法：

``` python
def my_callback(sender, **kwargs):
    print("Request finished!")
```

注意该函数带有一个 `sender` 参数，以及通配符关键字参数（`**kwargs`）;所有的信号处理程序必须采用这些参数。

稍后我们会看看 `sender`，但现在看看 `**kwargs` 参数。所有信号都会发送关键字参数，并可能随时更改这些关键字参数。在 `request_finished` 的情况下，它被记录为不发送参数，这意味着我们可能会试图将我们的信号处理写为 `my_callback(sender)`。

这将是错误的 - 事实上，如果你这样做，Django 会抛出一个错误。这是因为在任何时候参数都可能被添加到信号中，并且接收器必须能够处理这些新的参数。

### 连接接收器函数

有两种方法可以将接收器连接到信号。您可以采取手动连接路线：

``` python
from django.core.signals import request_finished

request_finished.connect(my_callback)
```

或者，您可以使用 `receiver()` 装饰器：

以下是你如何与装饰器连接：

``` python
from django.core.signals import request_finished
from django.dispatch import receiver

@receiver(request_finished)
def my_callback(sender, **kwargs):
    print("Request finished!")
```

现在，我们的 `my_callback` 函数将在每次请求结束时被调用。

> 这代码应该写在哪？ <br> <br>
> 严格地说，信号处理和注册代码可以写在任何你喜欢的地方，虽然建议避免应用程序的根模块及其 `models` 模块来最小化导入代码的副作用。 <br> <br>
> 实际上，信号处理程序通常在与其相关的应用程序的 `signals` 子模块中定义。信号接收器连接在应用程序配置类（`apps.py`）的 `ready()` 方法中。如果您使用的是 `receiver()` 装饰器，只需在 `ready()` 中导入信号子模块。

!> `ready()` 方法可能会在测试过程中多次执行，因此您可能希望防止重复信号，特别是如果您计划在测试中发送它们。

### 连接到特定 sender 发送的信号

一些信号会被多次发送，但您只会对接收这些信号的某个子集感兴趣。例如，考虑保存模型之前发送的 `django.db.models.signals.pre_save` 信号。大多数情况下，您不需要知道任何模型何时保存 - 只需保存一个特定模型即可。

在这些情况下，您可以注册接收仅由特定 sender 发送的信号。在 `django.db.models.signals.pre_save` 的情况下，sender 将是保存的模型类，因此您可以指示您只需要某个模型发送的信号：

``` python
from django.db.models.signals import pre_save
from django.dispatch import receiver
from myapp.models import MyModel


@receiver(pre_save, sender=MyModel)
def my_handler(sender, **kwargs):
    ...
```

只有当 `MyModel` 的一个实例被保存时才会调用 `my_handler` 函数。

### 防止重复信号

在某些情况下，将接收器连接到信号的代码可能会运行多次。这可能会导致您的接收器功能被多次注册，因此会针对单个信号事件多次调用。

如果此行为有问题（例如，保存模型时使用信号发送电子邮件），请传递唯一标识符作为 `dispatch_uid` 参数以标识您的接收器功能。这个标识符通常是一个字符串，尽管任何可哈希对象都足够了。最终的结果是，每个唯一的 `dispatch_uid` 值只会将接收器函数绑定到信号一次：

``` python
from django.core.signals import request_finished

request_finished.connect(my_callback, dispatch_uid="my_unique_identifier")
```

## 定义和发送信号

您的应用程序可以利用信号基础设施并提供自己的信号。

### 定义信号

``` python
import django.dispatch

pizza_done = django.dispatch.Signal(providing_args=["toppings", "size"])
```

声明一个 `pizza_done` 信号将提供 `toppings` 和 `size` 参数的接收器。

请记住，您可以随时更改此参数列表，因此在第一次尝试时获得 API 并非必要。

### 发送信号

在 Django 中发送信号有两种方式。

`Signal.send(sender, **kwargs)`

`Signal.send_robust(sender, **kwargs)`

要发送信号，请调用 `Signal.send()`（所有内置信号均使用此信号）或 `Signal.send_robust()`。你必须提供 `sender` 参数（这是大多数时候的类），并且可以提供尽可能多的其他关键字参数。

例如，以下是发送 `pizza_done` 信号的例子：

``` python
class PizzaStore:
    ...

    def send_pizza(self, toppings, size):
        pizza_done.send(sender=self.__class__, toppings=toppings, size=size)
        ...
```

`send()` 和 `send_robust()` 都返回一组元组对 `[(receiver, response), ... ]` 的列表，表示被调用的接收函数及其响应值的列表。

 `send()` 与 `send_robust()` 的不同之处在于如何处理接收函数引发的异常。`send()` 不会捕获接收者引发的任何异常;它只是允许错误传播。因此，在出现错误时，并非所有接收机都可能被通知信号。

`send_robust()` 捕获从 Python 的 `Exception` 类派生的所有错误，并确保所有接收器都被通知信号。如果发生错误，则错误实例会返回到引发错误的接收方的元组对中。

回调函数存在于调用 `send_robust()` 时返回的错误的 `__traceback__` 属性中。

## 断开信号

`Signal.disconnect(receiver=None, sender=None, dispatch_uid=None)`

要从信号中断开接收器，请调用 `Signal.disconnect()`。参数如 `Signal.connect()` 中所述。如果接收器断开，该方法返回 `True`，否则返回 `False`。

`receiver` 参数指示注册的接收者断开连接。如果使用 `dispatch_uid` 来标识接收者，它可能是 `None`。