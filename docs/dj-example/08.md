# 8 管理支付和订单

在上一章中，你创建了一个包括商品目录和订单系统的在线商店。你还学习了如何用 Celery 启动异步任务。在这一章中，你会学习如何在网站中集成支付网关。你还会扩展管理站点，用于管理订单和导出不同格式的订单。

我们会在本章覆盖以下知识点：

- 在项目中集成支付网关
- 管理支付通知
- 导出订单到 CSV 文件中
- 为管理站点创建自定义视图
- 动态生成 PDF 单据

## 8.1 集成支付网关

支付网关允许你在线处理支付。你可以使用支付网关管理用户订单，以及通过可靠的，安全的第三方代理处理支付。这意味着你不用考虑在自己的系统中存储信用卡。

有很多支付网关可供选择。我们将集成 PayPal，它是最流行的支付网关之一。

PayPal 提供了几种方法在网站中集成它的网关。标准集成包括一个`Buy now`按钮，你可能在其它网站见过。这个按钮把顾客重定向到 PayPal 来处理支付。我们将在网站中集成包括一个自定义`Buy now`按钮的`PayPal Payments Standard`。PayPal 会处理支付，并发送一条支付状态的信息到我们的服务器。

### 8.1.1 创建 PayPal 账户

你需要一个 PayPal 商家账户，才能在网站中集成支付网关。如果你还没有 PayPal 账户，在[这里](https://www.paypal.com/signup/account)注册。确保你选择了商家账户。

在注册表单填写详细信息完成注册。PayPal 会给你发送一封邮件确认账户。

### 8.1.2 安装 django-paypal

`django-paypal`是一个第三方 Django 应用，可以简化在 Django 项目中集成 PayPal。我们将用它在我们的商店中集成`PayPal Payments Standard`。你可以在[这里](http://django-paypal.readthedocs.io/en/stable/)查看 django-paypal 的文档。

在终端使用以下命令安装 django-paypal：

```py
pip install django-paypal
```

编辑项目的`settings.py`文件，在`INSTALLED_APPS`设置中添加`paypal.standard.ipn`：

```py
INSTALLED_APPS = [
	# ...
	'paypal.standard.ipn',
]
```

这个应用是 django-paypal 提供的，通过`Instant Payment Notification(IPN)`集成`PayPal Payments Standard`。我们之后会处理支付通知。

在`myshop`的`settings.py`文件添加以下设置来配置 django-paypal：

```py
# django-paypal settings
PAYPAL_RECEIVER_EMAIL = 'mypaypalemail@myshop.com'
PAYPAL_TEST = True
```

这些设置分别是：

- `PAYPAL_RECEIVER_EMAIL`：你 PayPal 账户的邮箱地址。用你创建 PayPal 账户的邮箱替换`mypaypalemail@myshop.com`。
- `PAYPAL_TEST`：一个布尔值，表示是否用 PayPal 的 Sandbox 环境处理支付。在迁移到生产环境之前，你可以用 Sandbox 测试 PayPal 集成。

打开终端执行以下命令，同步 django-paypal 的模型到数据库中：

```py
python manage.py migrate
```

你会看到类似这样结尾的输出：

```py
Running migrations:
  Applying ipn.0001_initial... OK
  Applying ipn.0002_paypalipn_mp_id... OK
  Applying ipn.0003_auto_20141117_1647... OK
  Applying ipn.0004_auto_20150612_1826... OK
  Applying ipn.0005_auto_20151217_0948... OK
  Applying ipn.0006_auto_20160108_1112... OK
  Applying ipn.0007_auto_20160219_1135... OK
```

现在 django-paypal 的模型已经同步到数据库中。你还需要添加 django-paypal 的 URL 模式到项目中。编辑`myshop`项目的主`urls.py`文件，并添加以下 URL 模式。记住，把它放在`shop.urls`模式之前，避免错误的模式匹配：

```py
url(r'^paypal/', include('paypal.standard.ipn.urls')),
```

让我们把支付网关添加到结账过程中。

### 8.1.3 添加支付网关

结账流程是这样的：

1. 用户添加商品到购物车中。
2. 用户结账购物车。
3. 重定向用户到 PayPal 进行支付。
4. PayPal 发送支付通知到我们的服务器。
5. PayPal 重定向用户返回我们的网站。

使用以下命令在项目中创建一个新应用：

```py
python manage.py startapp payment
```

我们将使用这个应用管理结账流程和用户支付。

编辑项目的`settings.py`文件，在`INSTALLED_APP`设置中添加`payment`：

```py
INSTALLED_APPS = [
	# ...
	'paypal.standard.ipn',
	'payment',
]
```

现在`payment`应用已经在项目中激活了。编辑`orders`应用的`views.py`文件，添加以下导入：

```py
from django.shortcuts import render, redirect
from django.core.urlresolvers import reverse
```

找到`order_create`视图中的以下代码：

```py
# launch asynchronous task
order_created.delay(order.id)
return render(request, 'orders/order/created.html', {'order': order})
```

替换为下面的代码：

```py
# launch asynchronous task
order_created.delay(order.id)
request.session['order_id'] = order.id
return redirect(reverse('payment:process'))
```

创建订单成功之后，我们用`order_id`会话键在当前会话中设置订单 ID。然后我们把用户重定向到接下来会创建的`payment:process` URL。

编辑`payment`应用的`views.py`文件，并添加以下代码：

```py
from decimal import Decimal
from django.conf import settings
from django.core.urlresolvers import reverse
from django.shortcuts import render, get_object_or_404
from paypal.standard.forms import PayPalPaymentsForm
from orders.models import Order

def payment_process(request):
    order_id = request.session.get('order_id')
    order = get_object_or_404(Order, id=order_id)
    host = request.get_host()

    paypal_dict = {
        'business': settings.PAYPAL_RECEIVER_EMAIL,
        'amount': '%.2f' % order.get_total_cost().quantize(Decimal('.01')),
        'item_name': 'Order {}'.format(order.id),
        'invoice': str(order.id),
        'currency_code': 'USD',
        'notify_url': 'http://{}{}'.format(host, reverse('paypal-ipn')),
        'return_url': 'http://{}{}'.format(host, reverse('payment:done')),
        'cancel_return': 'http://{}{}'.format(host, reverse('payment:canceled')),
    }
    form = PayPalPaymentsForm(initial=paypal_dict)
    return render(request, 'payment/process.html', {'order': order, 'form': form})
```

在`payment_process`视图中，我们生成了一个自定义 PayPal 的`Buy now`按钮用于支付。首先我们从`order_id`会话键中获得当前订单，这个键值之前在`order_create`视图中设置过。我们获得指定 ID 的`Order`对象，并创建了包括以下字段的`PayPalPaymentForm`：

- `business`：处理支付的 PayPal 商家账户。在这里我们使用`PAYPAL_RECEIVER_EMAIL`设置中定义的邮箱账户。
- `amount`：向顾客收取的总价。
- `item_name`：出售的商品名。我们使用商品 ID，因为订单里可能包括多个商品。
- `invoice`：单据 ID。每次支付对应的这个 ID 应用是唯一的。我们使用订单 ID。
- `currency_code`：这次支付的货币。我们设置为`USD`使用美元。使用与 PayPal 账户中设置的相同货币（`EUR`对应欧元）。
- `notify_url`：PayPal 发送 IPN 请求到这个 URL。我们使用 django-paypal 提供的`paypal-ipn` URL。这个 URL 关联的视图处理负责支付通知和在数据库中保存支付通知。
- `return_url`：支付成功后重定向用户到这个 URL。我们使用之后会创建的`payment:done` URL。
- `cancel_return`：如果支付取消，或者遇到其它问题，重定向用户到这个 URL。我们使用之后会创建的`payment:canceled` URL。

`PayPalPaymentForm`会被渲染为带隐藏字典的标准表单，用户只能看到`Buy now`按钮。点用户点击这个按钮，表单会通过 POST 提交到 PayPal。

让我们创建一个简单的视图，当支付完成，或者因为某些原因取消支付，让 PayPal 重定向用户。在同一个`views.py`文件中添加以下代码：

```py
from django.views.decorators.csrf import csrf_exempt

@csrf_exempt
def payment_done(request):
    return render(request, 'payment/done.html')

@csrf_exempt
def payment_canceled(request):
    return render(request, 'payment/canceled.html')
```

因为 PayPal 可以通过 POST 重定向用户到这些视图的任何一个，所以我们用`csrf_exempt`装饰器避免 Django 期望的 CSRF 令牌。在`payment`应用目录中创建`urls.py`文件，并添加以下代码：

```py
from django.conf.urls import url
from . import views

urlpatterns = [
    url(r'^process/$', views.payment_process, name='process'),
    url(r'^done/$', views.payment_done, name='done'),
    url(r'^canceled/$', views.payment_canceled, name='canceled'),
]
```

这些是支付流程的 URL。我们包括了以下 URL 模式：

- `process`：用于生成带`Buy now`按钮的 PayPal 表单的视图
- `done`：当支付成功后，用于 PayPal 重定向用户
- `canceled`：当支付取消后，用于 PayPal 重定向用户

编辑`myshop`项目的主`urls.py`文件，引入`payment`应用的 URL 模式：

```py
url(r'^payment/', include('payment.urls', namespace='payment')),
```

记住把它放在`shop.urls`模式之前，避免错误的模式匹配。

在`payment`应用目录中创建以下文件结构：

```py
templates/
	payment/
		process.html
		done.html
		canceled.html
```

编辑`payment/process.html`模板，添加以下代码：

```py
{% extends "shop/base.html" %}

{% block title %}Pay using PayPal{% endblock title %}

{% block content %}
    <h1>Pay using PayPal</h1>
    {{ form.render }}
{% endblock content %}
```

这个模板用于渲染`PayPalPaymentForm`和显示`Buy now`按钮。

编辑`payment/done.html`模板，添加以下代码：

```py
{% extends "shop/base.html" %}

{% block content %}
    <h1>Your payment was successful</h1>
    <p>Your payment has been successfully received.</p>
{% endblock content %}
```

用户支付成功后，会重定向到这个模板页面。

编辑`payment/canceled.html`模板，并添加以下代码：

```py
{% extends "shop/base.html" %}

{% block content %}
    <h1>Your payment has not been processed</h1>
    <p>There was a problem processing your payment.</p>
{% endblock content %}
```

处理支付遇到问题，或者用户取消支付时，会重定向到这个模板页面。

让我们尝试完整的支付流程。

### 8.1.4 使用 PayPal 的 Sandbox

在浏览器中打开`http://developer.paypal.com`，并用你的 PayPal 商家账户登录。点击`Dashboard`菜单项，然后点击`Sandbox`下的`Accounts`选项。你会看到你的 sandbox 测试账户列表，如下图所示：

![](http://ooyedgh9k.bkt.clouddn.com/%E5%9B%BE8.1.png)

最初，你会看到一个商家账户和一个 PayPal 自动生成的个人测试账户。你可以点击`Create Account`按钮创建新的 sandbox 测试账户。

点击列表中`Type`为`PERSONAL`的账户，然后点击`Pofile`链接。你会看到测试账户的信息，包括邮箱地址和个人资料信息，如下图所示：

![](http://ooyedgh9k.bkt.clouddn.com/%E5%9B%BE8.2.png)

在`Funding`标签页中，你会看到银行账户，信用卡数据，以及 PayPal 贷方余额。

当你的网站使用 sandbox 环境时，测试账户可以用来处理支付。导航到`Profile`标签页，然后点击修改`Change password`链接。为这个测试账户创建一个自定义密码。

在终端执行`python manage.py runserver`命令启动开发服务器。在浏览器中打开`http://127.0.0.1:8000/`，添加一些商品到购物车中，然后填写结账表单。当你点击`Place order`按钮时，订单会存储到数据库中，订单 ID 会保存在当前会话中，然后会重定向到支付处理页面。这个页面从会话中获得订单，并渲染带`Buy now`按钮的 PayPal 表单，如下图所示：

![](http://ooyedgh9k.bkt.clouddn.com/%E5%9B%BE8.3.png)

> **译者注：**启动开发服务器后，还需要启动 RabbitMQ 和 Celery，因为我们要用它们异步发送邮件，否则会抛出异常。

你可以看一眼 HTML 源码，查看生成的表单字段。

点击`Buy now`按钮。你会被重定向到 PayPal，如下图所示：

![](http://ooyedgh9k.bkt.clouddn.com/%E5%9B%BE8.4.png)

输入顾客测试账号的邮箱地址和密码，然后点击登录按钮。你会被重定向到以下页面：

![](http://ooyedgh9k.bkt.clouddn.com/%E5%9B%BE8.5.png)

> **译者注：**即之前修改过密码的个人账户。

现在点击`立即付款`按钮。最后，你会看到一个包括交易 ID 的确认页面，如下图所示：

![](http://ooyedgh9k.bkt.clouddn.com/%E5%9B%BE8.6.png)

点击`返回商家`按钮。你会被重定向到`PayPalPaymentForm`的`return_url`字段指定的 URL。这是`payment_done`视图的 URL，如下图所示：

![](http://ooyedgh9k.bkt.clouddn.com/%E5%9B%BE8.7.png)

支付成功！但是因为我们在本地运行项目，127.0.0.1 不是一个公网 IP，所以 PayPal 不能给我们的应用发送支付状态通知。我们接下来学习如何让我们的网站可以从 Internet 访问，从而接收 IPN 通知。

### 8.1.5 获得支付通知

IPN 是大部分支付网关都会提供的方法，用于实时跟踪购买。当网关处理完一个支付后，会立即给你的服务器发送一个通知。该通知包括所有支付细节，包括状态和用于确认通知来源的支付签名。这个通知作为独立的 HTTP 请求发送到你的服务器。出现问题的时候，PayPal 会多次尝试发送通知。

`django-payapl`自带两个不同的 IPN 信号，分别是：

- `valid_ipn_received`：当从 PayPal 接收的 IPN 消息是正确的，并且不会与数据库中现在消息重复时触发
- `invalid_ipn_received`：当从 PayPal 接收的消息包括无效数据或者格式不对时触发

我们将创建一个自定义接收函数，并把它连接到`valid_ipn_received`信号来确认支付。

在`payment`应用目录中创建`signals.py`文件，并添加以下代码：

```py
from django.shortcuts import get_object_or_404
from paypal.standard.models import ST_PP_COMPLETED
from paypal.standard.ipn.signals import valid_ipn_received
from orders.models import Order

def payment_notification(sender, **kwargs):
    ipn_obj = sender
    if ipn_obj.payment_status == ST_PP_COMPLETED:
        # payment was successful
        order = get_object_or_404(Order, id=ipn_obj.invoice)
        # mark the order as paid
        order.paid = True
        order.save()

valid_ipn_received.connect(payment_notification)
```

我们把`payment_notification`接收函数连接到 django-paypal 提供的`valid_ipn_received`信号。接收函数是这样工作的：

1. 我们接收`sender`对象，它是在`paypal.standard.ipn.models`中定义的`PayPalPN`模型的一个实例。
2. 我们检查`paypal_status`属性，确保它等于 django-paypal 的完成状态。这个状态表示支付处理成功。
3. 接着我们用`get_object_or_404`快捷函数获得订单，这个订单的 ID 必须匹配我们提供给 PayPal 的`invoice`参数。
4. 我们设置订单的`paid`属性为`True`，标记订单状态为已支付，并把`Order`对象保存到数据库中。

当`valid_ipn_received`信号触发时，你必须确保信号模块已经加载，这样接收函数才会被调用。最好的方式是在包括它们的应用加载的时候，加载你自己的信号。可以通过定义一个自定义的应用配置来实现，我们会在下一节中讲解。

### 8.1.6 配置我们的应用

你已经在第六章学习了应用配置。我们将为`payment`应用定义一个自定义配置，用来加载我们的信号接收函数。

在`payment`应用目录中创建`apps.py`文件，并添加以下代码：

```py
from django.apps import AppConfig

class PaymentConfig(AppConfig):
    name = 'payment'
    verbose_name = 'Payment'

    def ready(self):
        # improt signal handlers
        import payment.signals
```

在这段代码中，我们为`payment`应用定义了一个`AppConfig`类。`name`参数是应用的名字，`verbose_name`是一个可读的名字。我们在`ready()`方法中导入信号模板，确保应用初始化时会加载信号模块。

编辑`payment`应用的`__init__.py`文件，并添加这一行代码：

```py
default_app_config = 'payment.apps.PaymentConfig'
```

这会让 Django 自动加载你的自定义应用配置类。你可以在[这里](https://docs.djangoproject.com/en/1.11/ref/applications/)阅读更多关于应用配置的信息。

### 8.1.7 测试支付通知

因为我们在本地环境开发，所以我们需要让 PayPal 可以访问我们的网站。有几个应用程序可以让开发环境通过 Internet 访问。我们将使用 Ngrok，是最流行的之一。

从[这里](https://ngrok.com/)下载你的操作系统版本的 Ngrok，并使用以下命令运行：

```py
./ngrok http 8000
```

这个命令告诉 Ngrok 在 8000 端口为你的本地主机创建一个链路，并为它分配一个 Internet 可访问的主机名。你可以看到类似这样的输入：

```py
Session Status                online
Account                       lakerszhy (Plan: Free)
Update                        update available (version 2.2.4, Ctrl-U to update)
Version                       2.1.18
Region                        United States (us)
Web Interface                 http://127.0.0.1:4040
Forwarding                    http://c0f17d7c.ngrok.io -> localhost:8000
Forwarding                    https://c0f17d7c.ngrok.io -> localhost:8000

Connections                   ttl     opn     rt1     rt5     p50     p90
                              0       0       0.00    0.00    0.00    0.00
```

Ngrok 告诉我们，我们网站使用的 Django 开发服务器在本机的 8000 端口运行，现在可以通过`http://c0f17d7c.ngrok.io`和`https://c0f17d7c.ngrok.io`（分别对应 HTTP 和 HTTPS 协议）在 Internet 上访问。Ngrok 还提供了一个网页 URL，这个网页显示发送到这个服务器的信息。在浏览器中打开 Ngrok 提供的 URL，比如`http://c0f17d7c.ngrok.io`。在购物车中添加一些商品，下单，然后用 PayPal 测试账户支付。此时，PayPal 可以访问`payment_process`视图中`PayPalPaymentForm`的`notify_url`字段生成的 URL。如果你查看渲染的表单，你会看类似这样的 HTML 表单：

```py
<input id="id_notify_url" name="notify_url" type="hidden" value="http://c0f17d7c.ngrok.io/paypal/">
```

完成支付处理后，在浏览器中打开`http://127.0.0.1:8000/admin/ipn/paypalipn/`。你会看到一个`IPN`对象，对应状态是`Completed`的最新一笔支付。这个对象包括支付的所有信息，它由 PayPal 发送到你提供给 IPN 通知的 URL。

> **译者注：**如果通过`http://c0f17d7c.ngrok.io`访问在线商店，则需要在项目的`settings.py`文件的`ALLOWED_HOSTS`设置中添加`c0f17d7c.ngrok.io`。

> **译者注：**我在后台看到的一直都是`Pending`状态，一直没有找出原因。哪位朋友知道的话，请给我留言，谢谢。

你也可以在[这里](https://developer.paypal.com/developer/ipnSimulator/)使用 PayPal 的模拟器发送 IPN。模拟器允许你指定通知的字段和类型。

除了`PayPal Payments Standard`，PayPal 还提供了`Website Payments Pro`，它是一个订购服务，可以在你的网站接收支付，而不用重定向到 PayPal。你可以在[这里](http://django-paypal.readthedocs.io/en/latest/pro/index.html)查看如何集成`Website Payments Pro`。

## 8.2 导出订单到 CSV 文件

有时你可能希望把模型中的信息导出到文件中，然后把它导入到其它系统中。其中使用最广泛的格式是`Comma-Separated Values(CSV)`。CSV 文件是一个由若干条记录组成的普通文本文件。通常一行包括一条记录和一些定界符号，一般是逗号，用于分割记录的字段。我们将自定义管理站点，让它可以到处订单到 CSV 文件。

### 8.2.1 在管理站点你添加自定义操作

Django 提供了大量自定义管理站点的选项。我们将修改对象列表视图，在其中包括一个自定义的管理操作。

一个管理操作是这样工作的：用户在管理站点的对象列表页面用复选框选择对象，然后选择一个在所有选中选项上执行的操作，最后执行操作。下图显示了操作位于管理站点的哪个位置：

![](http://ooyedgh9k.bkt.clouddn.com/%E5%9B%BE8.8.png)

> 创建自定义管理操作允许工作人员一次在多个元素上进行操作。

你可以编写一个常规函数来创建自定义操作，该函数需要接收以下参数：

- 当前显示的`ModelAdmin`
- 当前请求对象——一个`HttpRequest`实例
- 一个用户选中对象的`QuerySet`

当在管理站点触发操作时，会执行这个函数。

我们将创建一个自定义管理操作，来下载一组订单的 CSV 文件。编辑`orders`应用的`admin.py`文件，在`OrderAdmin`类之前添加以下代码：

```py
import csv
import datetime
from django.http import HttpResponse

def export_to_csv(modeladmin, request, queryset):
    opts = modeladmin.model._meta
    response = HttpResponse(content_type='text/csv')
    response['Content-Disposition'] = 'attachment;filename={}.csv'.format(opts.verbose_name)
    writer = csv.writer(response)

    fields = [field for field in opts.get_fields() if not field.many_to_many and not field.one_to_many]
    # Write a first row with header information
    writer.writerow([field.verbose_name for field in fields])
    # Write data rows
    for obj in queryset:
        data_row = []
        for field in fields:
            value = getattr(obj, field.name)
            if isinstance(value, datetime.datetime):
                value = value.strftime('%d/%m/%Y')
            data_row.append(value)
        writer.writerow(data_row)
    return response
export_to_csv.short_description = 'Export to CSV'
```

在这段代码中执行了以下任务：

1. 我们创建了一个`HttpResponse`实例，其中包括定制的`text/csv`内容类型，告诉浏览器该响应看成一个 CSV 文件。我们还添加了`Content-Disposition`头部，表示 HTTP 响应包括一个附件。
2. 我们创建了 CSV 的`writer`对象，用于向`response`对象中写入数据。
3. 我们用模型的`_meta`选项的`get_fields()`方法动态获得模型的字段。我们派出了对多对和一对多关系。
4. 我们用字段名写入标题行。
5. 我们迭代给定的`QuerySet`，并为`QuerySet`返回的每个对象写入一行数据。因为 CSV 的输出值必须为字符串，所以我们格式化`datetime`对象。
6. 我们设置函数的`short_description`属性，指定这个操作在模板中显示的名字。

我们创建了一个通用的管理操作，可以添加到所有`ModelAdmin`类上。

最后，如下添加`export_to_csv`管理操作到`OrderAdmin`类上：

```py
calss OrderAdmin(admin.ModelAdmin):
	# ...
	actions = [export_to_csv]
```

在浏览器中打开`http://127.0.0.1:8000/admin/orders/order/`，管理操作如下图所示：

![](http://ooyedgh9k.bkt.clouddn.com/%E5%9B%BE8.9.png)

选中几条订单，然后在选择框中选择`Export to CSV`操作，接着点击`Go`按钮。你的浏览器会下载生成的`order.csv`文件。用文本编辑器打开下载的文件。你会看到以下格式的内容，其中包括标题行，以及你选择的每个`Order`对象行：

```py
ID,first name,last name,email,address,postal code,city,created,updated,paid
1,allen,iverson,lakerszhy@gmail.com,北京市朝阳区,100012,北京市,11/05/2017,11/05/2017,False
2,allen,kobe,lakerszhy@gmail.com,北京市朝阳区,100012,北京市,11/05/2017,11/05/2017,False
```

正如你所看到的，创建管理操作非常简单。

## 8.3 用自定义视图扩展管理站点

有时，你可能希望通过配置`ModelAdmin`，创建管理操作和覆写管理目标来定制管理站点。这种情况下，你需要创建自定义的管理视图。使用自定义视图，可以创建任何你需要的功能。你只需要确保只有工作人员能访问你的视图，以及让你的模板继承自管理模板来维持管理站点的外观。

让我们创建一个自定义视图，显示订单的相关信息。编辑`orders`应用的`views.py`文件，并添加以下代码：

```py
from django.contrib.admin.views.decorators import staff_member_required
from django.shortcuts import get_object_or_404
from .models import Order

@staff_member_required
def admin_order_detail(request, order_id):
    order = get_object_or_404(Order, id=order_id)
    return render(request, 'admin/orders/order/detail.html', {'order': order})
```

`staff_member_required`装饰器检查请求这个页面的用户的`is_active`和`is_staff`字段是否为`True`。这个视图中，我们用给定的 ID 获得`Order`对象，然后渲染一个模板显示订单。

现在编辑`orders`应用的`urls.py`文件，添加以下 URL 模式：

```py
url(r'^admin/order/(?P<order_id>\d+)/$', views.admin_order_detail, name='admin_order_detail'),
```

在`orders`应用的`templates`目录中创建以下目录结构：

```py
admin/
	orders/
		order/
			detail.html
```

编辑`detail.html`模板，添加以下代码：

```py
{% extends "admin/base_site.html" %}
{% load static %}

{% block extrastyle %}
    <link rel="stylesheet" type="text/css" href="{% static "css/admin.css" %}" />
{% endblock extrastyle %}

{% block title %}
    Order {{ order.id }} {{ block.super }}
{% endblock title %}

{% block breadcrumbs %}
    <div class="breadcrumbs">
        <a href="{% url "admin:index" %}">Home</a> $rsaquo;
        <a href="{% url "admin:orders_order_changelist" %}">Orders</a> $rsaquo;
        <a href="{% url "admin:orders_order_change" order.id %}">Order {{ order.id }}</a> 
        $rsaquo; Detail
    </div>
{% endblock breadcrumbs %}

{% block content %}
    <h1>Order {{ order.id }}</h1>
    <ul class="object-tools">
        <li>
            <a href="#" onclick="window.print();">Print order</a>
        </li>
    </ul>
    <table>
        <tr>
            <th>Created</th>
            <td>{{ order.created }}</td>
        </tr>
        <tr>
            <th>Customer</th>
            <td>{{ order.first_name }} {{ order.last_name }}</td>
        </tr>
        <tr>
            <th>E-mail</th>
            <td><a href="mailto:{{ order.email }}">{{ order.email }}</a></td>
        </tr>
        <tr>
            <th>Address</th>
            <td>{{ order.address }}, {{ order.postal_code }} {{ order.city }}</td>
        </tr>
        <tr>
            <th>Total amount</th>
            <td>${{ order.get_total_cost }}</td>
        </tr>
        <tr>
            <th>Status</th>
            <td>{% if order.paid %}Paid{% else %}Pending payment{% endif %}</td>
        </tr>
    </table>

    <div class="module">
        <div class="tabular inline-related last-related">
            <table>
                <h2>Items bought</h2>
                <thead>
                    <tr>
                        <th>Product</th>
                        <th>Price</th>
                        <th>Quantity</th>
                        <th>Total</th>
                    </tr>
                </thead>
                <tbody>
                    {% for item in order.items.all %}
                        <tr class="row{% cycle "1" "2" %}">
                            <td>{{ item.product.name }}</td>
                            <td class="num">${{ item.price }}</td>
                            <td class="num">{{ item.quantity }}</td>
                            <td class="num">${{ item.get_cost }}</td>
                        </tr>
                    {% endfor %}
                    <tr class="total">
                        <td colspan="3">Total</td>
                        <td class="num">${{ order.get_total_cost }}</td>
                    </tr>
                </tbody>
            </table>
        </div>
    </div>
{% endblock content %}
```

这个模板用于在管理站点显示订单详情。模板扩展自 Django 管理站点的`admin/base_site.html`模板，其中包括主 HTML 结构和管理站的 CSS 样式。我们加载自定义的静态文件`css/admin.css`。

为了使用静态文件，我们可以从本章的示例代码中获得它们。拷贝`orders`应用的`static/`目录中的静态文件，添加到你项目中的相同位置。

我们使用父模板中定义的块引入自己的内容。我们显示订单信息和购买的商品。

当你想要扩展一个管理模板时，你需要了解它的结构，并确定它存在哪些块。你可以在[这里](https://github.com/django/django/tree/1.11/django/contrib/admin/templates/admin)查看所有管理模板。

如果需要，你也可以覆盖一个管理模板。把要覆盖的模板拷贝到`templates`目录中，保留一样的相对路径和文件。Django 的管理站点会使用你自定义的模板代替默认模板。

最后，让我们为管理站点的列表显示页中每个`Order`对象添加一个链接。编辑`orders`应用的`amdin.py`文件，在`OrderAdmin`类之前添加以下代码：

```py
from django.core.urlresolvers import reverse

def order_detail(obj):
    return '<a href="{}">View</a>'.format(reverse('orders:admin_order_detail', args=[obj.id]))
order_detail.allow_tags = True
```

这个函数接收一个`Order`对象作为参数，并返回一个`admin_order_detail`的 HTML 链接。默认情况下，Django 会转义 HTML 输出。我们必须设置函数的`allow_tags`属性为`True`，从而避免自动转义。

> 在任何`Model`方法，`ModelAdmin`方法，或者可调用函数中设置`allow_tags`属性为`True`可以避免 HTML 转义。使用`allow_tags`时，确保转义用户的输入，以避免跨站点脚本。

然后编辑`OrderAdmin`类来显示链接：

```py
class OrderAdmin(admin.ModelAdmin):
    list_display = [... order_detail]
```

在浏览器中打开`http://127.0.0.1:8000/admin/orders/order/`，现在每行都包括一个`View`链接，如下图所示：

![](http://ooyedgh9k.bkt.clouddn.com/%E5%9B%BE8.10.png)

点击任何一个订单的`View`链接，会加载自定义的订单详情页面，如下图所示：

![](http://ooyedgh9k.bkt.clouddn.com/%E5%9B%BE8.11.png)

## 8.4 动态生成 PDF 单据

我们现在已经有了完成的结账和支付系统，可以为每个订单生成 PDF 单据了。有几个 Python 库可以生成 PDF 文件。一个流行的生成 PDF 文件的 Python 库是 Reportlab。你可以在[这里](https://docs.djangoproject.com/en/1.11/howto/outputting-pdf/)查看如果使用 Reportlab 输出 PDF 文件。

大部分情况下，你必须在 PDF 文件中添加自定义样式和格式。你会发现，让 Python 远离表现层，渲染一个 HTML 模板，然后把它转换为 PDF 文件更加方便。我们将采用这种方法，在 Django 中用模块生成 PDF 文件。我们会使用 WeasyPrint，它是一个 Python 库，可以从 HTML 模板生成 PDF 文件。

### 8.4.1 安装 WeasyPrint

首先，为你的操作系统安装 WeasyPrint 的依赖，请访问[这里](http://weasyprint.readthedocs.io/en/latest/install.html)。

然后用以下命令安装 WeasyPrint：

```py
pip install WeasyPrint
```

### 8.4.2 创建 PDF 模板

我们需要一个 HTML 文档作为 WeasyPrint 的输入。我们将创建一个 HTML 模板，用 Django 渲染它，然后把它传递给 WeasyPrint 生成 PDF 文件。

在`orders`应用的`templates/orders/order/`目录中创建`pdf.html`文件，并添加以下代码：

```py
<html>
<body>
    <h1>My Shop</h1>
    <p>
        Invoice no. {{ order.id }}</br>
        <span class="secondary">
            {{ order.created|date:"M d, Y" }}
        </span>
    </p>

    <h3>Bill to</h3>
    <p>
        {{ order.first_name }} {{ order.last_name }}</br>
        {{ order.email }}</br>
        {{ order.address }}</br>
        {{ order.postal_code }}, {{ order.city }}
    </p>

    <h3>Items bought</h3>
    <table>
        <thead>
            <tr>
                <th>Product</th>
                <th>Price</th>
                <th>Quantity</th>
                <th>Cost</th>
            </tr>
        </thead>
        <tbody>
            {% for item in order.items.all %}
                <tr class="row{% cycle "1" "2" %}">
                    <td>{{ item.product.name }}</td>
                    <td class="num">${{ item.price }}</td>
                    <td class="num">{{ item.quantity }}</td>
                    <td class="num">${{ item.get_cost }}</td>
                </tr>
            {% endfor %}
            <tr class="total">
                <td colspan="3">Total</td>
                <td class="num">${{ order.get_total_cost }}</td>
            </tr>
        </tbody>
    </table>

    <span class="{% if order.paid %}paid{% else %}pending{% endif %}">
        {% if order.paid %}Paid{% else %}Pending payment{% endif %}
    </span>
</body>
</html>
```

这是 PDF 单据的模板。在这个模板中，我们显示所有订单详情和一个包括商品的 HTML 的`<table>`元素。我们还包括一个消息，显示订单是否支付。

### 8.4.3 渲染 PDF 文件

我们将创建一个视图，在管理站点中生成已存在订单的 PDF 单据。编辑`orders`应用的`views.py`文件，并添加以下代码：

```py
from django.conf import settings
from django.http import HttpResponse
from django.template.loader import render_to_string
import weasyprint

@staff_member_required
def admin_order_pdf(request, order_id):
    order = get_object_or_404(Order, id=order_id)
    html = render_to_string('orders/order/pdf.html', {'order': order})
    response = HttpResponse(content_type='application/pdf')
    response['Content-Disposition'] = 'filename="order_{}.pdf"'.format(order.id)
    weasyprint.HTML(string=html).write_pdf(response, 
        stylesheets=[weasyprint.CSS(settings.STATIC_ROOT + 'css/pdf.css')])
    return response
```

这个视图用于生成订单的 PDF 单据。我们用`staff_member_required`装饰器确保只有工作人员可以访问这个视图。我们用给定的 ID 获得`Order`对象，并用 Django 提供的`render_to_string()`函数渲染`orders/order/pdf.html`文件。被渲染的 HTML 保存在`html`变量中。然后，我们生成一个新的`HttpResponse`对象，指定`application/pdf`内容类型，并用`Content-Disposition`指定文件名。我们用 WeasyPrint 从被渲染的 HTML 代码生成一个 PDF 文件，并把文件写到`HttpResponse`对象中。我们用`css/pdf.css`静态文件为生成的 PDF 文件添加 CSS 样式。我们从`STATIC_ROOT`设置中的本地路径加载它。最后返回生成的响应。

因为我们需要使用`STATIC_ROOT`设置，所以需要把它添加到我们项目中。这是项目的静态文件存放的路径。编辑`myshop`项目的`settings.py`文件，添加以下设置：

```py
STATIC_ROOT = os.path.join(BASE_DIR, 'static/')
```

接着执行`python manage.py collectstatic`命令。你会看到这样结尾的输出：

```py
You have requested to collect static files at the destination
location as specified in your settings:

    /Users/lakerszhy/Documents/GitHub/Django-By-Example/code/Chapter 8/myshop/static

This will overwrite existing files!
Are you sure you want to do this?
```

输入`yes`并按下`Enter`。你会看到一条消息，显示静态文件已经拷贝到`STATIC_ROOT`目录中。

`collectstatic`命令拷贝应用中所有静态文件到`STATIC_ROOT`设置中定义的目录。这样每个应用可以在`static/`目录中包括静态文件。你还可以在`STATICFILES_DIRS`设置中提供其它静态文件源。执行`collectstatic`命令时，`STATICFILES_DIRS`中列出的所有目录都会被拷贝到`STATIC_ROOT`目录中。

编辑`orders`应用中的`urls.py`文件，添加以下 URL 模式：

```py
url(r'admin/order/(?P<order_id>\d+)/pdf/$', views.admin_order_pdf, name='admin_order_pdf'),
```

现在，我们可以编辑管理列表显示页面，为`Order`模型的每条记录添加一个 PDF 文件链接。编辑`orders`应用的`admin.py`文件，在`OrderAdmin`类之前添加以下代码：

```py
def order_pdf(obj):
    return '<a href="{}">PDF</a>'.format(reverse('orders:admin_order_pdf', args=[obj.id]))
order_pdf.allow_tags = True
order_pdf.short_description = 'PDF bill'
```

把`order_pdf`添加到`OrderAdmin`类的`list_display`属性中，如下所示：

```py
class OrderAdmin(admin.ModelAdmin):
    list_display = [..., order_detail, order_pdf]
```

如果你为可调用对象指定了`short_description`属性，Django 将把它作为列名。

在浏览器中打开`http://127.0.0.1:8000/admin/orders/order`。每行都会包括一个 PDF 链接，如下图所示：

![](http://ooyedgh9k.bkt.clouddn.com/%E5%9B%BE8.12.png)

点击任意一条订单的 PDF 链接。你会看到生成的 PDF 文件，下图是未支付的订单：

![](http://ooyedgh9k.bkt.clouddn.com/%E5%9B%BE8.13.png)

已支付订单如下图所示：

![](http://ooyedgh9k.bkt.clouddn.com/%E5%9B%BE8.14.png)

### 8.4.4 通过邮件发送 PDF 文件

当收到支付时，让我们给顾客发送一封包括 PDF 单据的邮件。编辑`payment`应用的`signals.py`文件，并添加以下导入：

```py
from django.template.loader import render_to_string
from django.core.mail import EmailMessage
from django.conf import settings
import weasyprint
from io import BytesIO
```

然后在`order.save()`行之后添加以下代码，保持相同的缩进：

```py
# create invoice e-mail
subject = 'My Shop - Invoice no. {}'.format(order.id)
message = 'Please, find attached the invoice for your recent purchase.'
email = EmailMessage(subject, message, 'admin@myshop.com', [order.email])

# generate PDF
html = render_to_string('orders/order/pdf.html', {'order': order})
out = BytesIO()
stylesheets = [weasyprint.CSS(settings.STATIC_ROOT + 'css/pdf.css')]
weasyprint.HTML(string=html).write_pdf(out, stylesheets=stylesheets)
# attach PDF file
email.attach('order_{}.pdf'.format(order.id), out.getvalue(), 'application/pdf')
# send e-mail
email.send()
```

在这个信号中，我们用 Django 提供的`EmailMessage`类创建了一个邮件对象。然后把模板渲染到`html`变量中。我们从渲染的模板中生成 PDF 文件，并把它输出到一个`BytesIO`实例（内存中的字节缓存）中。接着我们用`EmailMessage`对象的`attach()`方法，把生成的 PDF 文件和`out`缓存中的内容添加到`EmailMessage`对象中。

记得在项目`settings.py`文件中设置发送邮件的`SMTP`设置，你可以参考第二章。

现在打开 Ngrok 提供的应用 URL，完成一笔新的支付，就能在邮件中收到 PDF 单据了。

## 8.5 总结

在这一章中，你在项目中集成了支付网关。你自定义了 Django 管理站点，并学习了如果动态生成 CSV 和 PDF 文件。

下一章会深入了解 Django 项目的国际化和本地化。你还会创建一个优惠券系统和商品推荐引擎。