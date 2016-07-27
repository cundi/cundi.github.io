---
layout: post
title:  "使用Celery和Redis自动化Django任务队列"
date:   2015-02-10 15:14:54
categories: Django
tags:   Django  Celery  Redis
---

* content
{:toc}

Task Queue: a robust, asynchronous job handler and message broker enabling you to run and consume back-end functions based on system events rather than human intervention. A Django and Celery powered application can quickly respond to user requests while the task automation manager handles longer running requests through the use of a task queuing system.  




## Automate the Django Task Queue with Celery and Redis
`原始链接，http://www.craigderington.me/configure-django-task-queue-with-celery-and-redis/`  

Task Queue: a robust, asynchronous job handler and message broker enabling you to run and consume back-end functions based on system events rather than human intervention. A Django and Celery powered application can quickly respond to user requests while the task automation manager handles longer running requests through the use of a task queuing system.  

>Celery uses a message broker to pass messages between your application and the Celery worker process. In this tutorial, I am going to use Redis as the message broker and Celery as the task manager. As Redis is a persistent, in-memory data structure server, it should not be used to manage application state; as the state may not be maintained during a reboot or other system type event requiring the server to be re-started.

## 开始前的准备
I am using a clean AWS instance running Ubuntu 14.04 in a free T2-micro tier for this demonstration. The first step is to add Celery to our Django application inside our virtualenv. Our project directory is located at /var/www/ and the Django project is called django_dev.  

## 安装Redis
At our Ubuntu terminal, update the package manager.  

```shell
$ sudo apt-get update
$ sudo apt-get install redis-server
```

Then check the Redis version to ensure the server is listening.  

```shell
$ redis-server --version
Redis server v=2.8.4 sha=0000000:0...  
```

Is Redis up?  

```shell
$ redis-cli
> PING
PONG  
```

Next, activate the virtual environment and install Celery.  

```shell
/var/www/django_dev~$ source .env/bin/activate
(.env)/var/www/django_dev~$ pip install celery[redis]
(...)
Successfully installed pytz celery billiard kombu redis anyjson amqp  
Cleaning up...  
```

## 对Django应用添加Celery支持
We have to add several Celery specific settings to our Django applications settings.py. I prefer to keep my settings outside of the named inner project directory.  

```
config/settings/common.py
```

```
# Celery
BROKER_URL = 'redis://localhost:6379/0'  
CELERY_ACCEPT_CONTENT = ['json']  
CELERY_TASK_SERIALIZER = 'json'  
CELERY_RESULT_SERIALIZER = 'json' 
```

Next, create celery.py and save it in your config.settings directory. This will start Celery and create a Celery application.  

```python
from __future__ import absolute_import

import os  
from celery import Celery  
from config.settings import common as settings

# set Django settings module for celery program.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')

app = Celery('hello_django')

# Using a string here means the worker will not have to
# pickle the object when using Windows.
app.config_from_object('config.settings')  
app.autodiscover_tasks(lambda: settings.INSTALLED_APPS) 
```

To ensure that the Celery application starts everytime the Django app is started, we have to add an `__init.py__` to our project folder.  

```python
from __future__ import absolute_import  
from .celery import app as celery_app
```

### 创建第一个Celery任务
Add a new app to our Django project and ass some tasks. First, create the application.  

```shell
(.env)/var/www/django_dev~$ django-admin startapp taskapp
```

Add the new app to our config settings common.py file.  

```python
LOCAL_APPS = (  
    ...
    'taskapp',
)
```

Now, in our taskapp app folder, add a new file tasks.py and add the following code to create the first Celery task.  

```python
from __future__ import absolute_import  
from celery import shared_task


@shared_task
def tasktest(param):  
    return 'The task executed with argument "%s" ' % param
```

If you have followed this project setup, your projct directory will look like this...  

```shell
/var/www/django_dev
|--config
|  | -- settings
|       | -- common.py
|       | -- local.py
|       | -- production.py
|       | -- dev.py
|  | -- wsgi.py
|  | -- celery.py
|  | -- urls.py
| -- manage.py
| -- taskapp
|    | -- __init__.py
|    | -- models.py
|    | -- tests.py
|    | -- views.py
|    | -- urls.py
|    | -- tasks.py
```

## 测试任务应用
In a real production environment, you would want your Celery task queue daemonized, however, for our demonstration, we will just call it from the command line and inspect the output.  

```shell
$ export PYTHONPATH=/var/www/django_dev/config.settings:$PYTHONPATH
$ /var/www/django_dev/bin/celery --app=config.settings.celery:app worker --loglevel=INFO
-------------------------- celery@django v3.1.20

...

-- * - ********** [config]
-- * - **********.> app:         taskapp
-- * - **********.> transport:   redis://localhost:6379/0
-- * - **********.> results:     disabled
-- * - **********.> concurrency: 2 (prefork)
...
---- ************ [queues]
-----------------.> celery     exchange=celery(direct) key=celery


[tasks]
. taskapp.tasks.test


[2016-03-01 08:45:30, 849: INFO/MainProcess] Connected to redis://localhost:6379/0
[2016-03-01 08:45:30, 852: INFO/MainProcess] mingle:  searching forneighbors
[2016-03-01 08:45:31, 236: INFO/MainProcess] mingle:  all alone
[2016-03-01 08:45:31, 428: WARNING/MainProcess] celery@django ready
```

As long as the configuration settings are all OK, you will be greeted by a welcome screen like above. The [tasks] section will list all tasks discovered in all apps within your Django project directory.  

## Queuing Up Tasks for Execution
In a separate terminal instance, start your project's virtual environment and launch a new task.  

```shell
/var/www/django_dev~$ source .env/bin/activate
(.env)/var/www/django_dev~$ cd taskapp
(.env)/var/www/django_dev/taskapp~$ python manage.py shell
Python 2.7.6 (default June 22 2015 17:58:13)  
[GCC 4.8.2] on linux2
Type "help", "copyright", "credits" or "license" formore information  
(InteractiveConsole)
>>> from taskapp.tasks import tasktest
>>> tasktest.delay('This is only a test.')
<AsyncResult:  67a43cf9-0b3f-4792-e746-2f3aa35e5b17
```

Now if you look at the Celery task terminal, you will notice task workers have started:  

```shell
[2016-03-01 10:30:24, 624: [INFO/MainProcess] Received task taskapp.tasks.tasktest[67a43cf9-0b3f-4792-e746-2f3aa35e5b17]
[2016-03-01 10:30:24, 673: [INFO/MainProcess] Task taskapp.tasks.tasktest[67a43cf9-0b3f-4792-e746-2f3aa35e5b17]
```

## Celery Daemons & Supervisord
Most Django devs run Celery as a daemon using a nifty little tool called supervisord that allows workers to be restarted in the event of a system reboot. To install supervisor, enter this command the terminal prompt.  

```shell
sudo apt-get install supervisor  
```

As soon as supervisor is installed, you can add programs to it's configuration file so it knows to watch those processes for changes and update the terminal accordingly.  

Supervisors config file is located at /etc/supervisor/conf.d. For our example, we will create a supervisor config file to watch our Celery processes.  

```
[program:  taskapp-celery
command=/var/www/django_dev/bin/celery --app=taskapp.celery:app worker -- loglevel=INFO  
directory=/var/www/django_dev/taskapp  
user=tech-walker  
stdout_logfile=/var/www/django_dev/logs/celery-worker.log  
stderr_logfile=/var/www/django_dev/logs/celery-worker.log  
autostart=true  
autorestart=true  
startsecs=10

; need to wait for any tasks still running
stopwaitsecs=900

; send SIGKILL to destroy all processes as a group
killasgroup=true

; if rabbitmq is supervised, set it to a higher priority
priority=998
```

This is a simple sample config file provided by the Celery team.  

Now, create a file to store the application's log messages.  

```shell
/var/www/django_dev:~$ mkdir logs
/var/www/django_dev:~$ touch logs/celery-worker.log
```

Send a command to supervisord to re-read the config file and update the process.  

```shell
$ sudo supervisorctl reread
taskapp-celery:  available  
$ sudo supervisorctl update
taskapp-celery:  added process group  
```

Monitor the output of the Celery task process by examining the log file.  

```shell
$ tail -f /var/www/django_dev/logs/celery-worker.log
```

For more information on configuring Celery and options for monitoring the task queue status, check out the Celery User Guide.  

## Wrap Up
Using Redis with Celery running in the application background is an easy way to automate many of the processes required to keep your application humming along with very little overhead. Use supervisord to monitor the task queue. In some of my recent projects, I use Celery tasks to make RESTful calls against a Django REST API, which in turn triggers both GET and POST methods to specified API endpoints to distribute data to multiple servers.  
