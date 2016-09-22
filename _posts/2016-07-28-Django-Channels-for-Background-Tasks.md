---
layout: post
title: "使用Django Channels处理后台任务"
categories: Django Channels
tags: Django Channels
---

* content
{toc}

`原文链接：http://albertoconnor.ca/blog/2016/May/18/django-channels-background-tasks`

>#### 译者注：
本文讨论了使用Channels处理一般后台任务，但看原文的评论来说，若是希望使用更强大的工具还得用Celery

Django Channels is the most exciting thing to happen to Django since well Django :).  

This little tutorial is what you need to add a background task processor to Django using channels. Our task for this example will just be outputting "Hello, Channels!", but you could image running a subprocess on some data or sending an email.  

NOTE: channels works on an at-most-once delivery model, so it is possible a message in a channel could be lost, delivery isn't guaranteed. That also means consumers don't have to worry about duplicates. Read more in the docs.

This example will be stripped down to the basic code without much error checking. There are detailed examples one [here](https://github.com/andrewgodwin/channels-examples) and [here](https://github.com/jacobian/channels-example).  

We will start with a simple Django 1.9 app without channels  

```python
# urls.py
from django.conf.urls import url
from . import views

urlpatterns = [
    url(r'^$', views.home, name='home'),
]


# views.py
from django.shortcuts import render

def home(request, template="home.html"):
    print("Hello, Channels!")  # long running task of printing.
    return render(
        request,
        template,
        dict(),
    )
```

You will need to define a home.html template where Django can find it, but besides that this simple site should work and synchronously render what is in your home.html and output "Hello, Channels!" on your terminal.  

## Now for channelification!
First, you will need redis, you can use a built-in broker but redis is pretty easy to install on Linux or Mac (using homebrew at least).  

```shell
# MacOS
homebrew install redis

# Debian
apt-get install redis-server
```

Once that is done you will need to install channels and asgi-redis with pip. This apparently also installs Twisted and zope.interface. The past returns :)  

```shell
pip install channels
```

Add 'channels' to your INSTALLED_APPS and add the following to your settings.py to tell channel how to run:  

```python
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

Where you replace "myproject" with the name of your project.  

You need to create the routing.py file—beside your settings.py and root url.py file. In it add the channel_routing list, but for now we will leave it empty.  

```python
# routing.py
channel_routing = []
```

This is actually enough to get what we already had to run.  

```shell
python manage.py runserver
```

This runs workers and the front end server daphne and it will handle Django requests and responses as you are used to.  

## Workers
Stop the server for now with Control-C. Django has grown a new command: runworker  

```shell
python manage.py runworker
```

This won't run on port 8000, it will just run a worker listening on the default channel layer so you can't connect to it directly. Kill it for now with Control-C.  

Channels has a front end server you can run in a separate process called daphne. To use it you need to create a asgi.py file—beside the wsgi.py file Django created for you, or your settings.py file.  

```python
# asgi.py
import os
import channels.asgi

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "myproject.settings")
channel_layer = channels.asgi.get_channel_layer()
```

Here is how you run daphne, should be familiar if you have used gunicorn or similar.  

```shell
daphne myproject.asgi:channel_layer --port 8000
```

Now you can connect to http://127.0.0.1:8000/ but it just times out because there aren't any workers. In a separate terminal window run use the runworker command to start a worker and try connecting again.  

Boom! You are back to where you were, but now there are two processes running. Now for the interesting bit, doing something asynchronously.  

## Background Task
Websockets are the killer feature of channels and other tutorials cover how to use them. Here we are going to make our own channel for our background task of saying hello.  

It starts in routing.py where we will connect our channel to our new consumer. A consumer is a function which takes a message much like how a view takes a request.  

```python
# routing.py again
from channels.routing import route
from .consumers import hello

channel_routing = [
    route('background-hello', hello),
]
```

This means messages put into the background-hello channel will be handled by the hello callable found in the local consumers.py file—place it beside your views.py file in your app folder. We need to write our first consumer:  

```python
# consumers.py
def hello(message):
    print("Hello, Channels!")  # long running task or printing
```

Unlike views, consumers don't have to return a response, but they can send any number of messages down any number of channels they like.  

Finally, we need to use our new background channel in our view, so let's edit views.py  

```python
# views.py
from django.shortcuts import render
from channels import Channel

def home(request, template="home.html"):
    message = {}
    Channel('background-hello').send(message)
    return render(
        request,
        template,
        dict(),
    )
```

That should be enough to get "Hello, Channels!" to be printed asynchronously to your terminal.  

What is interesting is you didn't have to do anything to get the HTTP request and response channel to work. Including and initializing the channels library is enough for it to create those channels and route based on url.py. Everything Django works as it always did.  

This just scratches the surface of what you can do with channels, so start using it today!  

You can view the code from this blog post on [GitHub](https://github.com/albertoconnor/channelsbackground)  

>## Comments

>Mark
I currently use Celery for this, also in conjunction with Redis. Could you explain the advantages of Channels over such pre-existing solutions?

>Antoine
The main advantage, compared to Celery, is the integration with Django, there won't be any pip package to install. But it can't replace all the use cases that Celery can handle (especially the complex ones).

>Actually Channels is not only about tasks, it's also about websocket and being able to develop some modern app which can push data to the user.


>Adylzhan
Hi,
Using Channels for executing background task is not recommended practice due to no guarantee of task execution/delivery.


>Albert
Depends on the nature of the background task, if it is mission critical, than no. But a lot of things like notification or even some kind of image process it is ok if it works 99% of the time.
Celery is definitely the right answer a lot of the time, but channel is much easier to get started with, then you can move to Celery if you need the power.

