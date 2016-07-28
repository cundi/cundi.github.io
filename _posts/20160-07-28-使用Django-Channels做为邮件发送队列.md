---
layout: post
title: "使用Django Channels做为邮件发送队列"
categories: Django
tags: Django Channels 翻译
---

* content
{toc}

`原文链接：https://www.wordfugue.com/using-django-channels-email-sending-queue/`
`by phildini on April 8, 2016`

## django的channels技术
Channels is a project by led Andrew Godwin to bring native asynchronous processing to Django. Most of the tutorials for integrating Channels into a Django project focus on Channels' ability to let Django "speak WebSockets", but Channels has enormous potential as an async task runner. Channels could replace Celery or RQ for most projects, and do so in a way that feels more native.  

Channels 是一个由Andrew Godwin领导，给Django带来原生异步处理的项目。很多把Channels集成到Django项目的教程注重于Channels让Django说“Websockets”的能力，但是Channels有着巨大的做为异步任务管理器的潜力。Channels可以替换大多数项目中的Celery或者RQ，而且这样做是一种更为原生的做法。  




To demonstrate this, let's use Channels to add non-blocking email sending to a Django project. We're going to add email invitations to a pre-existing project, and then send those invitations through Channels.  

为了说明此种技术，我们使用Channels对Django项目添加非阻塞的电邮发送。我们要对预先存在的项目提添加邮件邀请，然后通过Channels发送这些邀请。  

First, we'll need an invitation model. This isn't strictly necessary, as you could instead pass the right properties through Channels itself, but having an entry in the database provides a number of benefits, like using the Django admin to keep track of what invitations have been sent.  

首先，我们需要一个邀请模型。这里没有必要限制多严格，相反你可以传递正确的属性到Channels，数据库的中的项提供了很多的好处，比如，使用Django admin跟踪所发出的邀请。  

```python
from django.db import models
from django.contrib.auth.models import User


class Invitation(models.Model):

    email = models.EmailField()
    sent = models.DateTimeField(null=True)
    sender = models.ForeignKey(User)
    key = models.CharField(max_length=32, unique=True)

    def __str__(self):
        return "{} invited {}".format(self.sender, self.email)
```

We create these invitations using a ModelForm.  

我们使用ModelForm创建这些邀请。  

```python
from django import forms
from django.utils.crypto import get_random_string

from .models import Invitation


class InvitationForm(forms.ModelForm):

    class Meta:
        model = Invitation
        fields = ['email']

    def save(self, *args, **kwargs):
        self.instance.key = get_random_string(32).lower()
        return super(InvitationForm, self).save(*args, **kwargs)
```

Connecting this form to a view is left as an exercise to the reader. What we'd like to have happen now is for the invitation to be sent in the background as soon as it's created. Which means we need to install Channels.  

将这个表单和视图对接就做为一个练习留给读者。我们所希望的是邀请被创建后尽可能快地将其发送出去。这就是说，我们需要安装Channels了。  

```shell
pip install channels
```

We're going to be using Redis as a message carrier, also called a layer in Channels-world, between our main web process and the Channels worker processes. So we also need the appropriate Redis library.  

我们使用Redis做为消息传送器，在Channels领域内也称作中间层，因为Redis工作在我们的主web进程和Channels的工作进程之间。所以我们还需要一个适用于Redis的库。  

```shell
pip install asgi-redis
```

Redis is the preferred Channels layer and the one we're going to use for our setup. (The Channels team has also provided an in-memory layer and a database layer, but use of the database layer is strongly discouraged.) If we don't have Redis installed in our development environment, we'll need instructions for installing Redis on our development OS. (This possibly means googling "install redis {OUR OS NAME}".) If we're on a Debian/Linux-based system, this will be something like:  

Redis是Channels中间层的优先选择，这也是我们要使用的设置。（Channels团队还提供了一个内存中间层，和数据库中间层，但我们强烈不鼓励你使用数据库中间层。）如果在开发环境中还没有Redis，我们需要说明一下在开发系统中Redis的安装。（这也意味着你可以谷歌一下 “install redis {OUR OS NAME}”。）如果我们使用的基于Linux的Debian系统，其安装命令如下：  

```shell
apt-get install redis-server
```

If we're on a Mac, we're going to use Homebrew, then install Redis through Homebrew:  

如果我们使用的是Mac，我们要用到Homebrew，通过Homebrew来安装Redis：  

```shell
brew install redis
```

The rest of this tutorial is going to assume we have Redis installed and running in our development environment.  

本教程的剩余部分会假设我们已经在开发环境安装并运行了Redis。  

With Channels, redis, and asgi-redis installed, we can start adding Channels to our project. In our project's settings.py, add 'channels' to INSTALLED_APPS and add the channels configuration block.  

随着Channels, redis, 和asgi-redis的安装，我们可以开始对项目添加Channels了。在项目的settings.py文件中，将'channels'加入到INSTALLED_APPS，并添加channels的专属配置语句。  

```python
INSTALLED_APPS = (
    ...,
    'channels',
)

CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "asgi_redis.RedisChannelLayer",
        "CONFIG": {
            "hosts": [os.environ.get('REDIS_URL', 'redis://localhost:6379')],
        },
        "ROUTING": "myproject.routing.channel_routing",
    },
}
```

Let's look at the CHANNEL_LAYERS block. If it looks like Django's database settings, that's not an accident. Like we have a default database defined elsewhere in our settings, here we're defining a default Channels configuration. Our configuration uses the Redis backend, specifies the url of the Redis server, and points at a routing configuration. The routing configuration works like our project's urls.py. (We're also assuming our project is called 'myproject', you should replace that with your project's actual package name)  

我们看一看CHANNEL_LAYERS语句块。如果它看上去类似于Django的数据库设置，那就不回有问题。就像我们在设置中的其他地方定义的默认数据库一样，这里我们定义了一个Channels的默认配置。我们的配置使用Redis做为后端，为Redis服务器指定了url，并说明了路由配置。路由配置的工作类似于项目的urls.py。（我们还假设项目的名称为“myproject”， 你应该用自己项目的真实包名称去替换它。）

Since we're just using Channels to send email in the background, our routing.py is going to be pretty short.  

因为我们使用Channels在后端发送电邮，所以我们的routing.py会非常的短。  

```python
from channels.routing import route

from .consumers import send_invite

channel_routing = [
    route('send-invite',send_invite),
]
```

Hopefully this structure looks somewhat like how we define URLs. What we're saying here is that we have one route, 'send-invite', and what we receive on that channel should be consumed by the 'send_invite' consumer in our invitations app. The consumers.py file in our invitations app is similar to a views.py in a standard Django app, and it's where we're going to handle the actual email sending.  

希望这个结构看上去和我们定义的URL有些相似之处。这里我们要说的是，我们有一个路由，'send-invite'，在邀请应用中我们接受到的通道内容应该被send_invite的消费者所消费。邀请应用中的consumers.py文件类似于标准Django应用中的views.py，它是我们处理真实邮件发送的地方。  

```python
import logging
from django.contrib.sites.models import Site
from django.core.mail import EmailMessage
from django.utils import timezone

from invitations.models import Invitation

logger = logging.getLogger('email')


def send_invite(message):
    try:
        invite = Invitation.objects.get(
            id=message.content.get('id'),
        )
    except Invitation.DoesNotExist:
        logger.error("Invitation to send not found")
        return
    
    subject = "You've been invited!"
    body = "Go to https://%s/invites/accept/%s/ to join!" % (
            Site.objects.get_current().domain,
            invite.key,
        )
    try:
        message = EmailMessage(
            subject=subject,
            body=body,
            from_email="Invites <invites@%s.com>" % Site.objects.get_current().domain,
            to=[invite.email,],
        )
        message.send()
        invite.sent = timezone.now()
        invite.save()
    except:
        logger.exception('Problem sending invite %s' % (invite.id))
```

Consumers consume messages from a given channel, and messages are wrapper objects around blocks of data. That data must reduce down to a JSON blob, so it can be stored in a Channels layer and passed around. In our case, the only data we're using is the ID of the invite to send. We fetch the invite object from the database, build an email message based on that invite object, then try to send the email. If it's successful, we set a 'sent' timestamp on the invite object. If it fails, we log an error.  

消费者消费了来自已有通道的消息，消息是一个含有多一个数据块的包装器对象。数据必须削减为JSON blob，这样它才能够被存储在Channels的中间层，并传送出去。在我们的这个例子中，唯一要使用的数据是发出邀请的ID。我们从数据库获取invite对象，编写了一条基于invite对象的电邮消息，然后试着发送电邮。如果发送成功的话，我们对invite对象使者一个sent时间戳。如果发送失败的话，我们把错误记录到日志。  

The last piece to set in motion is sending a message to the 'send-invite' channel at the right time. To do this, we modify our InvitationForm  

最后一个要设置的动作是在合适的时机发送消息到'send-invite'通道。要这样做的话，我们需要修改InvitationForm  

```python
from django import forms
from django.utils.crypto import get_random_string

from channels import Channel

from .models import Invitation


class InvitationForm(forms.ModelForm):

    class Meta:
        model = Invitation
        fields = ['email']

    def save(self, *args, **kwargs):
        self.instance.key = get_random_string(32).lower()
        response = super(InvitationForm, self).save(*args, **kwargs)
        notification = {
            'id': self.instance.id,
        }
        Channel('send-invite').send(notification)
        return response
```

We import Channel from the channels package, and send a data blob on the 'send-invite' channel when our invite is saved.  

我们从channels包导入Channel，然后再在我们的邀请被保存时在'send-invite'通道上发送blob数据。  

Now we're ready to test! Assuming we've wired the form up to a view, and set the correct email host settings in our settings.py, we can test sending an invite in the background of our app using Channels. The amazing thing about Channels in development is that we start our devserver normally, and, in my experience at least, It Just Works.  

现在，我们准备做测试了！假设我们已经把表单连接到了视图，并在settings.py中的设置了正确的邮件主机设置，我们可以在使用了Channels的应用后台发送一次邀请。在开发过程中关于Channels令人振奋的事情是，我们正常启动了开发服务器，至少就我的经历而言，Channels是可以运行的。

```shell
python manage.py runserver
```

Congratulations! We've added background tasks to our Django application, using Channels!  

恭喜了！我们在Django应用使用中添加了后台任务，而且使用的是Channels！

Now, I don't believe something is done until it's shipped, so let's talk a bit about deployment. The Channels docs make a great start at covering this, but I use Heroku, so I'm adapting the excellent tutorial written by Jacob Kaplan-Moss for this project.  
 
现在，可以使用之前，我是不相信事情已经做完了的，所以我们来聊聊部署。Channels的文档对于部署说的很好了，但我用的是Heroku，所以改编了这份由Jacob Kaplan-Moss为此项目编写的教程。  

We start by creating an asgi.py, which lives in the same directory as the wsgi.py Django created for us.  

我们开始创建一个asgi.py，它放置在和Django为我们创建的wsgi.py同一个目录。  

```python
import os
import channels.asgi

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "myproject.settings")
channel_layer = channels.asgi.get_channel_layer()
```

(Again, remembering to replace "myproject" with the actual name of our package directory)  

（再说一遍，记得替换"myproject"为包目录的真实名称）

Then, we update our Procfile to include the main Channels process, running under Daphne, and a worker process.  

然后，我们更新Procfile以包括主要的Channels进程，

```python
web: daphne myproject.asgi:channel_layer --port $PORT --bind 0.0.0.0 -v2
worker: python manage.py runworker --settings=myproject.settings -v2
```

We can use Heroku's free Redis hosting to get started, deploy our application, and enjoy sending email in the background without blocking our main app serving requests.  

我们可以免费使用Heroku的Redis服务来部署应用，享受在后台发送邮件，而不会阻塞到主应用的请求。  

Hopefully this tutorial has inspired you to explore Channels' background-task functionality, and think about getting your apps ready for when Channels lands in Django core. I think we're heading towards a future where Django can do even more out-of-the-box, and I'm excited to see what we build!  

希望这份教程启发了你去研究Channels的后台任务功能，当Channels被用到Django核心时可以考虑为你的应用。我认为我们正在迈向Django可以实现更多开箱即用功能的未来，而且我很希望见到那一天！
