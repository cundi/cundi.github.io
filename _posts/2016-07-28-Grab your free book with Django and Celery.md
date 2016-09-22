http://blog.idego.pl/2016/02/11/how-to-grab-your-free-ebook-automatically-with-django-and-celery/  

Grab your free book with Django and Celery
What is Celery and how to use it?

    Sometimes one may face a problem with running code periodically, e.g. once per hour or once a day. For example Packt Publishing offers Free Learning program where you can grab free e-book everyday. But what if one day you can't be online or just simply forget to log in and check? It would be awesome to have this done automatically.
    Celery offers mechanisms to lessen difficulties while creating distributed systems. This framework works with the concept of distribution of work units (tasks) by exchanging messages among the machines that are interconnected as a network, or local workers. A task is the key concept in Celery; any sort of job we must distribute has to be encapsulated in a task beforehand [1].
     Celery can distribute tasks in a transparent way among workers that are spread over the Internet. It supports synchronous, asynchronous, periodic, and scheduled tasks. These tasks can be re-executed in he case of errors. Architecture of Celery is presented on figure 1.

Celery-architecture

Fig. 1. The Celery architecture [1]

     The client components create and dispatch tasks to the brokers. When a task call is performed, it returns an instance of type AsyncResult. The AsyncResult object is an object that allows the task status to be checked, its ending, and obviously, its return when it exists. To dispatch a task, one should make use of some of the methods of the task, e.g. apply_async().
     However, to make use of this mechanism, another component, the result backend, has to be active. This component has the role of storing the status and result of the task to return to the client application. From the result backend supported by Celery, RabbitMQ, Redis, MongoDB, Memcached, SQLAlchemy/Django ORM can be highlighted or you can define your own. Each result backend listed previously has strong and weak points.
     Workers are responsible for executing the tasks they have received. Celery posesses a series of mechanisms so that one can find the best way to control how workers will behave. Most popular of them are concurrency mode, remote control, revoking tasks and few others.
    A broker is definitely a key component in Celery. Through it, one get to send and receive messages and communicate with workers. Celery supports a large number of brokers. However, to some of these, not all Celery mechanisms are implemented. The most complete in terms of functionality are RabbitMQ and Redis. There are few interesting tutorials showing usage of Celery + Redis or Celery + RabbitMQ to queue tasks, but sometimes we have only option to use Django ORM as a result backend, for example on "plain" server with no VPS option.
    The website of django-celery warns us, that the package is no longer required but redirects us to Celery's tutorial where django-celery is still mentioned. Hmmm ... So, it means that we can do it both ways (with or without django-celery)?

I. Configuration of the project

    I.1. Our project is called mysite. Create an app (in this case called freelearning) and add it to INSTALLED_APPS in settings.py. Create file celery.py inside project's directory and tasks.py inside the application. The structure of our project is presented below.

├── freelearning
│   ├── __init__.py
│   ├── admin.py
│   ├── models.py
│   ├── tasks.py
│   ├── tests.py
│   └── views.py
├── mysite
│   ├── __init__.py
│   ├── celery.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── manage.py
└── requirements.txt
In the celery.py so called Celery application (the instance of Celery) is created.

from __future__ import absolute_import  
import os  
from django.conf import settings  
from celery import Celery

# set the default Django settings module for the 'celery' program.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'mysite.settings')

app = Celery('mysite')

# Using a string here means the worker will not have to
# pickle the object when using Windows.
app.config_from_object('django.conf:settings')  
app.autodiscover_tasks(lambda: settings.INSTALLED_APPS)

# testing task to check if Celery works
@app.task(bind=True)
def debug_task(self):  
    print "Celery works"
    I.2. To be sure that Celery app is loaded every time Django starts, following code should be added to mysite/__init.py__.

from __future__ import absolute_import  
# This will make sure the app is always imported when Django starts
from .celery import app as celery_app  
    I.3. In settings.py set configuration required for working with Django as a database backend.

# CELERY SETTINGS
BROKER_URL = 'django://'  
CELERY_ACCEPT_CONTENT = ['json']  
CELERY_TASK_SERIALIZER = 'json'  
CELERY_RESULT_SERIALIZER = 'json'

# Database backend settings
# http://docs.celeryproject.org/en/latest/configuration.html#conf-database-result-backend
CELERY_RESULT_BACKEND = 'db+scheme://user:password@host:port/dbname'  
Django database has some limitations, but for simple purposes should be sufficient.

    I.4. Add 'kombu.transport.django', to INSTALLED_APPS and synchronize your database. After this step two tables related to kombu should be created - djkombu_message and djkombu_queue. Kombu is a messaging library for Python and it becomes a successor of django-celery in the area of working with different kinds of tasks.

II. Working with tasks

    II.1. Before moving further we should check if the Celery worker is ready for receiving tasks. If you already have Celery installed (also SQLAlchemy package may be needed), after command:
$ celery -A mysite worker -l info information about our "testing" task from celery.py should be seen:

[tasks]
  . mysite.celery.debug_task
[2015-12-22 10:23:45,154: INFO/MainProcess] Connected to django://localhost//

    II.2. It's time to create some other tasks. Let's try with standard example available at Celery's repo or official documentation. Our tasks.py file is given below.

from __future__ import absolute_import  
from celery import shared_task

@shared_task
def add(x, y):  
    return x + y

@shared_task
def mul(x, y):  
    return x * y
    a) the simplest way to check if Celery sees our task is to stop and run again command from point II.1. In the shell with Celery we should see existence of our two new tasks:

[tasks]
  . freelearning.tasks.add
  . freelearning.tasks.mul
  . mysite.celery.debug_task

    b) if we want to submit tasks to the queue for execution, in another terminal we could type for example:

(InteractiveConsole)
>>> from freelearning.tasks import add, mul
>>> add.delay(1, 3)
<AsyncResult: 196a450b-c100-4d8b-b58e-f1e4cd0c2cf0>  
>>> mul.delay(2, 4)
<AsyncResult: 089a8879-369d-45be-bfd8-b858655e4423>  
Method delay() allows to send a task message, but does not support execution options. Information about our task should be seen in terminal with Celery:

[2015-12-22 11:26:00,630: INFO/MainProcess] Received task: freelearning.tasks.add[196a450b-c100-4d8b-b58e-f1e4cd0c2cf0]
[2015-12-22 11:26:02,190: INFO/MainProcess] Task freelearning.tasks.add[196a450b-c100-4d8b-b58e-f1e4cd0c2cf0] succeeded in 1.557360202s: 4
[2015-12-22 11:26:11,929: INFO/MainProcess] Received task: freelearning.tasks.mul[089a8879-369d-45be-bfd8-b858655e4423]
[2015-12-22 11:26:13,055: INFO/MainProcess] Task freelearning.tasks.mul[089a8879-369d-45be-bfd8-b858655e4423] succeeded in 1.12491046s: 8
    II.3. Periodic tasks are sent at regular intervals and they are executed by available workers. To try periodic tasks let's add some new code to file tasks.py:

from random import randint  
from celery.task import periodic_task  
from celery.schedules import crontab  
...
@periodic_task(run_every=(crontab(minute='*/1')),
               name='task_generate_random_number',
               ignore_result=True
               )
def task_generate_random_number():  
    print randint(1, 100)
    logger.info("Random number generated")
The job of this task is simple - every minute generate random number from range (1, 100). Periodic tasks are started with celery beat.

$ celery -A mysite beat -l info 
celery beat v3.1.19 (Cipater) is starting.  
.....
[2015-12-23 18:18:00,854: INFO/MainProcess] beat: Starting...
[2015-12-23 18:19:00,027: INFO/MainProcess] Scheduler: Sending due task task_generate_random_number (task_generate_random_number)
[2015-12-23 18:20:00,664: INFO/MainProcess] Scheduler: Sending due task task_generate_random_number (task_generate_random_number)
And in the second terminal we can see what workers do with sent tasks:

$ celery -A mysite worker -l info
[tasks]
  . freelearning.tasks.add
  . freelearning.tasks.mul
  . mysite.celery.debug_task
  . task_generate_random_number

[2015-12-23 18:17:51,094: INFO/MainProcess] Connected to django://localhost//
[2015-12-23 18:19:02,010: INFO/MainProcess] Received task: task_generate_random_number[eaaaf4c2-0657-47cb-9efa-6cd1994ee2c2]
[2015-12-23 18:19:02,011: WARNING/Worker-1] 50
[2015-12-23 18:19:02,011: INFO/Worker-1] task_generate_random_number[eaaaf4c2-0657-47cb-9efa-6cd1994ee2c2]: Random number generated
[2015-12-23 18:19:02,011: INFO/MainProcess] Task task_generate_random_number[eaaaf4c2-0657-47cb-9efa-6cd1994ee2c2] succeeded in 0.000651497000945s: None
[2015-12-23 18:20:02,263: INFO/MainProcess] Received task: task_generate_random_number[2c59ed3a-24f1-4822-8f7c-2fe6a47030cd]
[2015-12-23 18:20:02,266: WARNING/Worker-2] 72
[2015-12-23 18:20:02,266: INFO/Worker-2] task_generate_random_number[2c59ed3a-24f1-4822-8f7c-2fe6a47030cd]: Random number generated
[2015-12-23 18:20:02,267: INFO/MainProcess] Task task_generate_random_number[2c59ed3a-24f1-4822-8f7c-2fe6a47030cd] succeeded in 0.00181740499829s: None
...
Tasks are sent and received every minute so everything works fine. It's time to create tasks that will be used in production.
III. Grabbing the free e-book

    III.1. There could be two tasks in our application - first (obligatory) to grab free e-book every day and second (optional) to get notifications about aforementioned e-book via e-mail. By using Requests and lxml libraries our tasks could be accomplished as follows:

# freelearning/tasks.py

import requests  
from lxml import html  
from django.core.mail import EmailMessage  
...

# grab e-book everyday at 8.00 AM
@periodic_task(run_every=(crontab(minute=0, hour=8)),
               name='task_grab_free_ebook',
               ignore_result=True
               )
def task_grab_free_ebook():  
    # parameters required to log into PacktPub account
    params = {'email': 'youremail',
          'password': 'yourpassword', 
          'op': 'Login', 
          'form_id': 'packt_user_login_form'}
    FREE_LEARNING_URL = 'https://www.packtpub.com/packt/offers/free-learning'
    PACKT_URL = 'https://www.packtpub.com'
    page = requests.get(FREE_LEARNING_URL)
    webpage = html.fromstring(page.content)
    book_number = webpage.xpath("//a[@class='twelve-days-claim']/@href")
    # Use 'with' to ensure the session context is closed after use.
    with requests.Session() as s:
        p = s.post(FREE_LEARNING_URL, data=params)
        # An authorised request.
        r = s.get(PACKT_URL + book_number[0])
        # Log out
        l = s.get(PACKT_URL + '/logout')


# send e-mail at 8.01 AM
@periodic_task(run_every=(crontab(minute=1, hour=8)),
               name='task_send_email_about_ebook',
               ignore_result=True
               )
def task_send_email_about_ebook():  
    FREE_LEARNING_URL = 'https://www.packtpub.com/packt/offers/free-learning'
    page = requests.get(FREE_LEARNING_URL)
    webpage = html.fromstring(page.content)
    book_title = webpage.xpath("//div[@class='dotd-title']/h2/text()")
    subject = 'Your free e-book from PacktPub has just arrived'
    message = "Your new e-book is - " + book_title[0].strip()
    email = EmailMessage(subject,
                         message,
                         'from@example.com',
                         ['to@example.com'])
    email.send(fail_silently=True)
Remember about SMTP settings for Django to send e-mails properly.

IV. Time for production

    IV.1. It is recommended that the Celery worker and Celery scheduler should be run in a background as a daemon with Supervisor. Supervisor needs to know about Celery and configuration files are necessary to do it. Two files will are needed - one for worker and the second for the scheduler. Great explanation of using Supervisor and creating configuration files is given at this and this tutorial concerning asynchronous tasks with Django. Examples of Celery's configuration files for Supervisor are also available at GitHub. Very promising option is django-supervisor, but at the moment of writing this article not too much examples are available.
    In aforementioned tutorials access to directories other than 'home' folder of our remote server is required. But what to do at plain hosting account where you don't have root access? 

    IV.2. Fortunately, Celery worker can be daemonized. Celery can't do it itself, but proper tools can be used. Unprivileged users can use the celery multi utility or celery worker --detach. In our project second option was used:

celery -A mysite worker -l info --detach

    For unprivileged users interesting way to handle with Celery scheduler can be Linux Screen, which allows i.a. to start a long running process without maintaining an active shell session:

screen -A -m -d -S celery_beat celery -A mysite beat

    Connection of -m and -d flags has special meaning. It allows for starting screen in "detached" mode. This command creates a new session but doesn't attach to it. This is useful for system startup scripts.

V. What about "old, good" django-celery?

    V.1. Although working with asynchronous or periodic tasks can be done without django-celery, it is still very popular tool. The package is mature, well-tried and number of downloads shows continuous interest in using this software. Useful feature of django-celery is integration with admin interface. Figure 2 presents our periodic tasks created for purposes of this article. Third task is a built-in periodic task described in documentation. Also workers, schedulers and usual tasks can be inspected and managed through admin interface.

Admin-panel-django-celery

Fig. 2. Periodic tasks in admin panel with django-celery

    V.2. Interesting tutorials concerning usage of django-celery combined with RabbitMQ can be found here and here. Mainly due to fact, that this package is no longer necessary to integrate Django and Celery, in my opinion other tools like kombu and Flower will be developed with the idea of replacing django-celery.

Conclusion

    Combination of Django and Celery can be very handy in everyday tasks. We can use different tools for distributing our tasks, both for privileged and unprivileged users, what makes that this idea has a versatile usage.
    Ask Solem - creator of django-celery and one of the main contributors to Celery still plans new interesting features of his tools. For example in Celery 4.0 one will be able to execute task according to sunrises, sunsets etc. It shows that the idea of distributing different kinds of tasks could be all-purpose and still worth research.
    Most snippets of code from this article can be found at author's repo.

