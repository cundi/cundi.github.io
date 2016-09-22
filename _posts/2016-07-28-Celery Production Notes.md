---
layout: post
title: "Celery 生成环境笔记"
categories: Celery Django
tags: Django Redis Celery RabbitMQ broker worker
---

* content
{toc}

原文链接：http://www.john2x.com/blog/celery-production-notes.html  

I’ve been using Celery a lot lately on several projects at work lately, and I think I’ve finally gotten to a point where I am more or less comfortable working with it. It was painful getting started with Celery. The official docs and guides were useful of course, but when it came time to deploy to production, I found them to be quite lacking.  

Celery gets a lot of flak for being too big, or too difficult to work with. Despite the complaints, it’s mindshare, track record, and library support with Django makes going with the alternatives1 a bit less tempting. You’ll need a better reason for choosing an alternative other than just for the sake of using something else. At the end of the day, Celery works and is good enough for most projects.  

This post will focus on using Celery (3.1) with Django (1.9).  

本文着重于Celery (3.1)和Django (1.9)的使用。  

## Choose the right broker

The official guides already cover choosing the right broker, but I’ll quickly mention it here for completeness.  

官方指南已经提到了选择正确的代理，但是

Fortunately the choice is not difficult, as there are only two:  

- Redis
- RabbitMQ

There are other supported brokers, of course, but those aren’t considered “stable” yet. Never use the Django ORM as your broker. That would be like using manage.py runserver on production. (On development, using the ORM is a nice and easy way to get started, and avoids the overhead of setting up Redis/RabbitMQ on your machine).  

当让，还有其他的被支持的代理，但是它们被认为是还不够”稳定“。绝对不要使用Django ORM做为代理。就这样像是在生成环境中使用manage.py runserver。（在进行开发时，使用ORM入手是一个很好、简单的方式，从而避免在机器上设置Redis／RabbitMQ的花费。）  

From what I’ve read, Redis is good enough for most cases, but you should probably only use it if you have a good reason to not use RabbitMQ and understand the caveats that come with using Redis2.  

据我所知道的，Redis应对大多数情况足够了，但是

RabbitMQ is the officially recommended broker and is a good default choice. The official guide has a nice walkthrough on how to configure RabbitMQ.  

RabbitMQ是官方推荐的代理，而且也是一个很好的缺省选择。官方指南在如何配置RabbitMQ上有很好的说明。  

We also enable RabbitMQ’s management web UI. It becomes especially useful when your job count starts to get high and/or when something goes wrong with the queues. Just make sure you’ve set a strong password for the vhost user before turning it on.  

我们还启用了RabbitMQ的web管理界面。

It is a good idea to have a dedicated instance for RabbitMQ, as the message database could get quite big, and you’ll soon find yourself running out of disk space if it’s running on the same machine as the webserver or data storage.  

## Routing tasks 路由任务

By default, Celery routes all your tasks to the default queue named celery. This is good enough for small apps that have less than 10 (or less) different task functions.  

默认，Celery把你的全部任务路由到默认的名叫celery的队列。

Personally I like to have each Django “app” and its tasks have a dedicated queue. This makes it easy to adjust the workers for each app without affecting others.  

我个人喜欢让每个Django的“app”都拥有一个专用的队列。

For example, let’s say we have an app called customers with the following structure:  

例如，假如我们拥有一个称作customers的包含如下结构的应用：  

```
+ project/
|- core/
|  |- models.py
|  |- tasks.py
|  `- views.py
|- customers/
|  |- models.py
|  |- views.py
|  `- tasks.py
|- settings.py
|- celery_config.py
`- manage.py
```

And we want all customers tasks to be processed by a dedicated worker (could be running on a separate machine), and have all other tasks (e.g. core tasks) to be processed by a lower frequency/priority worker.  

我们想让所有的customers任务都可以由专用的worke（可能运行在一台独立的主机上）来处理，让剩余的其他任务（例如，core的任务）由一个低频率／优先级低worker处理。  

We’ll need a router for the customers app. Create a new file customers/routers.py:  

我们需要为customers应用设置路由。创建一个新文件customers/routers.py：  

```python
# customers/routers.py

class CeleryTaskRouter(object):
    def route_for_task(self, task, args=None, kwargs=None):
        if task.startswith('customers.'):
            return 'customers'
        return None
```

And specify our routers in settings.py:  

然后在settings.py指定我们的路由：  

```python
CELERY_ROUTES = (
    'customers.routers.CeleryTaskRouter',
)
```

You can have as many routers as you want, or one main router with a bunch of if statements.  

随便你设置多少个路由都可以，或者设置一个包含多个if语句主路由。  

Now, all tasks defined in the "customers.tasks" module will be routed to a queue named "customers", this is because Celery task names are derived from their module name and function name (if you don’t specify the name parameter in @app.task).  

现在，所有定义在"customers.tasks" 模块中的任务都会被路由到一个称作"customers"的队列，这就是Celery的任务名称衍生自对应的模块名和函数名（如果你不在@app.task中指定name参数的话）

If we start a worker like so:  

如果我吗想这样启动一个worker：  

```shell
$ ./manage.py celery worker --app=celery_config:app --queue=customers -n "customers.%%h"
```

The worker will have a name "customers.hostname" and will only process tasks defined in the customers.tasks module.  

worker会被命名为"customers.hostname"，而且仅处理定义在customers.tasks模块中的任务。  

Any other tasks (e.g. those defined in core.tasks) will be ignored, unless we start a separate worker process to process tasks going to the “default” queue (named "celery" by default).  

任何其他的任务（例如，那些定义在core.tasks中的）都会被忽略，除非我们启动一个独立的worker进程把任务处理到“默认”队列（默认，称作“celery”）。  

```shell
$ ./manage.py celery worker --app=celery_config:app --queue=celery
```

This decouples our apps’ workers and makes monitoring and managing them easier.  

解构过的应用worker可以让对它们监控和管理更为容易。  

Notice that all we needed to set in our settings.py is the CELERY_ROUTES variable. Celery automatically creates the necessary queues when needed. So only define CELERY_QUEUES3 when you absolutely need to fine tune your routes and queues.  

注意，需要我们做的是在settings.py中设置CELERY_ROUTES变量。有需要时，Celery将自动的创建必要的队列。所以，

## 管理workers

We use Supervisor at work to manage our app processes (e.g. gunicorn, flower, celery).  

我们使用Supervisor管理应用进程（例如，gunicorn, flower, celery）。  

Here’s what a typical entry in supervisor.conf for the two workers above would look like:  

```
[program:celery_customers]
command=/path/to/project/venv/bin/celery_supervisor --app=celery_config:app worker --loglevel=INFO --concurrency=8 --queue=customers -n "customers.%%h"
directory = /path/to/project
stdout_logfile=/path/to/project/logs/celery_customers.out.log
stderr_logfile=/path/to/project/logs/celery_customers.err.log
autostart=true
autorestart=true
user=user
startsecs=10
stopwaitsecs=60
stopasgroup=true
killasgroup=true

[program:celery_default]
command=/path/to/project/venv/bin/celery_supervisor --app=celery_config:app worker --loglevel=INFO --concurrency=8 --queue=celery
directory = /path/to/project
stdout_logfile=/path/to/project/logs/celery_default.out.log
stderr_logfile=/path/to/project/logs/celery_default.err.log
autostart=true
autorestart=true
user=user
startsecs=10
stopwaitsecs=60
stopasgroup=true
killasgroup=true
```

The celery_supervisor script is a custom script that wraps manage.py celery. It activates the virtualenv and the project variables (since doing that with Supervisor never seemed to work). It looks like this:  

celery_supervisor是一个包装了manage.py celery的自定义脚步。它激活了虚拟环境和项目变量。

```shell
#!/bin/sh

# Wrapper script for running celery with virtualenv and env variables set

DJANGODIR=/path/to/project

echo "Starting celery with args $@"

# Activate the virtual environment. 激活虚拟环境
cd $DJANGODIR
. venv/bin/activate

# Set additional environment variables. 设置额外的环境变量
. venv/bin/postactivate

# Programs meant to be run under supervisor should not daemonize themselves
# (do not use --daemon). 程序打算运行在supervisor下面的话就不应该后台化处理
exec celery "$@"
```

Now, when a worker goes down for whatever reason, Supervisor will automatically restart them.  

现在，不论什么原因让worker停止工作，Supervisor都可以自动地重启它们。  

## 增加并发

Currently we have our workers’ concurrency level set at 8. By default, Celery will spawn 8 threads for each worker (a total of 16 threads for our example). For low concurrency levels, this is fine, but when you start needing to process a few hundred or more tasks concurrently, you will soon find that your machine will have a hard time keeping up.  

Celery has an alternative worker pool implementation that uses Eventlet. This allows for significantly higher concurrency levels than possible with the default prefork/threads, but with the requirement that your task must be non-blocking.  

I had a hard time determining what makes a task non-blocking, as the official guide only mentions it in passing. Eventlet docs state that you will need to monkeypatch libraries you are using for it to work, so I was left to wonder on where should I insert this monkeypatching code or if I even needed to? The official examples seem to suggest that no explicit monkeypatching is needed, but I couldn’t be sure.  

After asking on StackOverflow4 and the Google Group5, I got my answer. Celery does indeed do the monkeypatching for you, and no changes are needed for your tasks or app.  

To use the eventlet pool implementation, you will need to install the eventlet library:  

```shell
pip install eventlet
```

And specify the --pool option when starting the worker:  

```shell
$ ./manage.py celery worker --app=celery_config.app --queue=customers -n "customers.%%h" --pool=eventlet --concurrency=256
```

## 监控和管理workers

Now that we have our workers up and running, we want to monitor them, check the tasks currently on queue and restart them when we push updates.  

现在我们的worker已经启动并运行了，我们想监控它们，

The recommended tool for this is Flower, a web UI for Celery. Install it as usual:  

对于此种目的推荐的工具是Flower，一个Celery的web界面。和往常一样安装它：  

```shell
pip install flower
```

Then start the server (make sure your workers are running already or flower will have trouble picking them up):  

然后，启动服务器（请保证你的worker已经运行了，要么flower在选择它们时会遇到问题）：  

```shell
$ flower -A celery_config:app --basic_auth=flower:secretpassword --port=5555
```

Visit the web UI at http://yourdomain.com:5555 and you can see a list of workers with their status. The UI should be pretty self explanatory.  

提供http://yourdomain.com:5555可以访问到web界面，你可以看到包含自身状态的worker列表。界面很简单，一看就能明白。  

### 重启worker
One way to restart the workers is directly via supervisorctl, but this could get cumbersome if you have lots of worker processes, and if they are distributed across multiple servers.  

My preferred method is to do it via the Flower UI. You can select the workers you want (note though that on the latest version of flower, the UX for selecting workers have regressed6), and then click on the “Shut Down” button. This will cause the workers to gracefully perform a warm shutdown, which is what we want. And since the workers are supervised, they will automatically start up again, now using the latest version of the code.  
