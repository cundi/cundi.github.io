---
layout: post
title: "Django使用Channels和Celery示例"
categories: Django Channels Celery
tags: Django Channels Celery 翻译
---

* content
{toc}

`原文链接：http://vincenttide.com/blog/1/django-channels-and-celery-example/`  

In this tutorial, I will go over how to setup a Django Channels project to work with Celery and have instant notification when task starts and completes. Django Channels uses WebSockets to enable two-way communication between the server and browser client. It is assumed that the reader is comfortable with how to setup a normal Django project and we will only cover the parts relating to Channels and Celery.  

在这份教程中，我们详细讨论如何设置一个使用Celery的Django Channels项目，并在任务启动和完成时发送即时通知。Django Channels使用WebSockets启动介于服务器端和浏览器端之间的双向通信。这里假设读者

You can find the [Github repository here](https://github.com/VincentTide/django-channels-celery-example) and a similar deployment at http://tasker.vincenttide.com. Note that this deployment contains some extra stuff not covered in this tutorial such as a cancel functionality. The front end of the sample deployment is also running the React library whereas we will only be using JavaScript in this demo.  

To get started let’s install some dependencies which we will need. We will need a Channels layer backend which is what Channels uses to pass and store messages. We will also need a Celery broker transport backend. As it turns out, we can use Redis for both of these tasks so that is what we will use.  

```shell
# Add Chris Lea’s redis ppa - he maintains the ppa for many open source projects
$ sudo add-apt-repository ppa:chris-lea/redis-server
$ sudo apt-get update
$ sudo apt-get install redis-server
# Now check that redis-server is up and running 检查redis服务器是否在运行
$ redis-cli ping
# PONG
```

Setup a new Django project in a virtualenv and install the following libraries:  

在虚拟环境中设置一个新的Django项目，然后安装下面的库：  

```shell
$ pip install django
$ pip install channels  # the channels library
$ pip install asgi_redis  # the redis channel layer we are using
$ pip install celery  # Celery task queue
```

Let’s take a look at the settings.py file first.  

首先，我们来看一看settings.py文件。  

```python
# Add our new app to installed apps
INSTALLED_APPS = [
#…
  ‘jobs’,
]
# Channels settings
CHANNEL_LAYERS = {
   "default": {
       "BACKEND": "asgi_redis.RedisChannelLayer",  # use redis backend
       "CONFIG": {
           "hosts": [os.environ.get('REDIS_URL', 'redis://localhost:6379')],  # set redis address
       },
       "ROUTING": "django_channels_celery_tutorial.routing.channel_routing",  # load routing from our routing.py file
   },
}
# Celery settings
BROKER_URL = 'redis://localhost:6379/0'  # our redis address
# use json format for everything
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
```

First we add our new app to the INSTALLED_APPS list. The Channels setting simply tells Channels what backend we are using, in this case Redis. The ROUTING option tells Channels where to look for our WebSockets routes which will be found in a file called routing.py. The Celery setting tells Celery where to look for our broker and that we want to use json format for everything.  

首先，我们把新应用添加到了INSTALLED_APPS列表。Channels设置简明的告诉Channels我们正在使用的后端，这个例子中我们使用的是Redis。ROUTING选项告诉Channels在哪里可以找到WebSockets路由，它可以在称作routing.py的一个文件中找到。Celery设置告诉Celery在哪里查找代理器，而且我们想要使用的是json数据格式。  

Next let’s look at the routing.py file:  

```python
from channels import route
from jobs import consumers
channel_routing = [
   # Wire up websocket channels to our consumers:
   route("websocket.connect", consumers.ws_connect),
   route("websocket.receive", consumers.ws_receive),
]
```

Here we simply hook up what functions we want to handle the connect and receive messages. We could also add a function to handle a disconnect message but for our purposes, that is not needed. We tell Channels to look for our functions in our jobs/consumers.py file.  

Here’s the main parts of the consumers.py file:  

```python
@channel_session
def ws_connect(message):
   message.reply_channel.send({
       "text": json.dumps({
           "action": "reply_channel",
           "reply_channel": message.reply_channel.name,
       })
   })

   
@channel_session
def ws_receive(message):
   try:
       data = json.loads(message['text'])
   except ValueError:
       log.debug("ws message isn't json text=%s", message['text'])
       return
   if data:
       reply_channel = message.reply_channel.name
       if data['action'] == "start_sec3":
           start_sec3(data, reply_channel)
```

In our ws_connect function, we will simply echo back to the client what their reply channel address is. The reply channel is the unique address that gets assigned to every browser client that connects to our websockets server. This value which can be retrieved from message.reply_channel.name can be saved or passed on to a different function such as a Celery task so that they can also send a message back. In fact this is what we will be doing. message.reply_channel.send is a convenient shortcut that Channels provides for us to reply back to the same client. If you only have the reply_channel name, you will have to use the below method to send a message:  

```python
Channel(reply_channel_name).send({
   "text": json.dumps({
       "action": "started",
       "job_id": job.id,
       "job_name": job.name,
       "job_status": job.status,
   })
})
```

In our ws_receive function, we look at the action parameter to check what the client wants us to do. If you wanted to do different things, you could have multiple action commands. In our example, we only have the one command which is to run a function called start_sec3. start_sec3 simply sleeps for 3 seconds and then sends a reply back to the client that it has completed. Note that we pass the reply_channel address so it knows where to send the response.  

The last important piece is the javascript handling the client side functions.  

```js
$(function() {
   // When we are using HTTPS, use WSS too.
   var ws_scheme = window.location.protocol == "https:" ? "wss" : "ws";
   var ws_path = ws_scheme + '://' + window.location.host + '/dashboard/';
   console.log("Connecting to " + ws_path)
   var socket = new ReconnectingWebSocket(ws_path);
   socket.onmessage = function(message) {
       console.log("Got message: " + message.data);
       var data = JSON.parse(message.data);
       // if action is started, add new item to table
       if (data.action == "started") {
           var task_status = $("#task_status");
           var ele = $('<tr></tr>');
           ele.attr("id", data.job_id);
           var item_id = $("<td></td>").text(data.job_id);
           ele.append(item_id);
           var item_name = $("<td></td>").text(data.job_name);
           ele.append(item_name);
           var item_status = $("<td></td>");
           item_status.attr("id", "item-status-"+data.job_id);
           var span = $('<span class="label label-primary"></span>').text(data.job_status);
           item_status.append(span);
           ele.append(item_status);
           task_status.append(ele);
       }
       // if action is completed, just update the status
       else if (data.action == "completed"){
           var item = $('#item-status-' + data.job_id + ' span');
           item.attr("class", "label label-success");
           item.text(data.job_status);
       }
   };
   $("#taskform").on("submit", function(event) {
       var message = {
           action: "start_sec3",
           job_name: $('#task_name').val()
       };
       socket.send(JSON.stringify(message));
       $("#task_name").val('').focus();
       return false;
   });
});
```

Here we first create the websockets object then we assign the socket.onmessage function to handle what we should do for each websockets message. If the action parameter is “started”, we will add a new entry to the table. If it action is completed, we simply change the corresponding column status to completed.  

这里，我们首先创建websockets对象，然后赋值socket.onmessage函数以处理每条websockets消息。如果action参数为“started”，我们对表格填海一个新条目。如果action已完成，我们简单改变对应列的状态。

The one form that we have is wired up to send a websockets message to the server that tells it to run the action “start_sec3”.  

To see the entire project files, visit the [Github repository](https://github.com/VincentTide/django-channels-celery-example). To run the Github repository code, first make sure you have Redis installed then run the following commands:  

```shell
pip install -r requirements.txt
python manage.py makemigrations
python manage.py migrate
python manage.py runserver  # Start daphne and workers
celery worker -A example -l info  # Start celery workers
```

That should start the development server on http://localhost:8000. Again, you can find a similar deployment on http://tasker.vincenttide.com.  

