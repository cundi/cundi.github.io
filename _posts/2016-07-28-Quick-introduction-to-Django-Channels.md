---
layout: post
title: "Django Channels简介"
categories: Django
tags: gevent Django 
---

* content
{toc}

原文链接：http://www.machinalis.com/blog/introduction-to-django-channels/  

Quick introduction to Django Channels
A brief description of Django Channels and its application on a real-life Django site

Posted by Agustín Bartó 4 months, 3 weeks ago Comments
Channels is an exciting upcoming feature of Django that will allow Django sites to support use cases that usually required the use of external tools and libraries (even non-Python ones) and even has the potential to change the way we work with the framework entirely.

According to its documentation, Django Channels is...

...a project to make Django able to handle more than just plain HTTP requests, including WebSockets and HTTP2, as well as the ability to run code after a response has been sent for things like thumbnailing or background calculation.
If you’ve worked with Django before, you already know how important this project is. Supporting the features mentioned with Django’s current implementation requires libraries like Celery (to handle complex tasks outside the real of a request), or Node.js, django-websocket-redis, or gevent-socketio to support WebSockets. With the exception of Celery (which is a defacto standard) all the other implementations are non-standard way of working around the limitations of Django with problems of their own. We’ve covered implementation of some of these features in older blogposts with varying degrees of success.

Having a standard implementation provides ease of maintenance, better security and faster turn-around times as most developers will be familiar with the concepts involved.

On this blogpost we’ll provide a quick introduction to the concepts involved in developing a Channels enabled Django site as well as presenting a working example covering the use case of pushing notifications to the client using WebSockets.

## The application
The example that we’re presenting is a modification of an application that we built for a blogpost on real-time notifications with gevent-socketio. The goal is to give you a chance to see how much simpler it is to implement the same site using Django Channels. The code for the example is available on [GitHub](https://github.com/abarto/tracker_project/tree/use-django-channels).

## Event-oriented Django
A default Django site follows the request-response model: A request comes in, it is routed to the proper view, the view generates a response, the response is sent to the client, and everything is done in a single process.

This works just fine for most applications, but it has its limitations. If the request is complex, it might hold the worker process for a long time, making subsequent requests wait for their turn. That’s why we use Celery to do things like generating thumbnails: the image upload comes in, we enqueue the thumbnail generation task and respond to the client right away, while the Celery worker takes care of the image on its own process.

The same happens if we want real-time two-way communication with clients. In a request-response scenario, we would need to keep a process dedicated to each client so we can receive and send information until the connection is no longer needed.

Django channels provides a different model: An event-oriented model. In this model, instead of requests and response we have just events. An event with a request is received, then it is given to the proper event handler which generates a new event with the response that is going to be sent to the client.

But the event model can be applied to other scenarios and not just mimicking the request-response model. For instance, suppose that a hardware sensor is triggered due to external condition, this generates an event which is given to the event handler, which in turn generates another event to notify whomever is interested in the occurrence of the original event.

But how does this process work? We need to discuss channels first before we get our hands on a working example.

What is a channel?
According to Django channel’s documentation, a channel is...

...an ordered, first-in first-out queue with message expiry and at-most-once delivery to only one listener at a time.
Several producers writes messages into a channel (which is identified by a name), and when a consumer that was subscribed to that channel becomes available, it picks the first message that came into the queue. Simple as that.

Channels changes the way Django works, making it act as a worker. Each worker listens on all channels with consumers assigned (or routed). When a message comes in, the appropriate consumer is invoked. In order for this to work, we need three layers:

Interface servers: It connects the site with the clients, through a WSGI adapter as well as a separate WebSocket server.
Channel back-end: It transports the messages between the interface and the workers. It is a datastore (memory for single-server situations, a database or Redis) and Python code to tie it all in.
The workers: They listen on all the channels and and runs the consumers (functions) when a message is ready.
The interface servers transform connections (HTTP, WebSockets, etc.) into messages on channels, and workers are in charge of handling those messages. The trick here is that message need not come from the interface servers. Messages can be created anywhere, in views, forms, signals, you name it.

Time to get our hands dirty.

Our first consumer
We’ll start with an fresh Django 1.8 (works with 1.9, too) installation. First, we need to install the “channels” projects which is available through PyPi. If you want to install the latest development version of channels, follow the instructions on the documentation.

Next we need to put “channels” into our INSTALLED_APPS setting:

INSTALLED_APPS = (
    ...
    'channels_test',  # Our test app
    'channels',
)
That’s it. Channels default configuration uses an in-memory back-end which works just fine for sites working on a single server.

We’re going to write a simple consumer that listens to messages on the default “http.message” channel and writes a new message onto the reply channel. Let’s create a module called “consumers.py” within our test Django app called “channels_test”:

# consumers.py

from json import dumps
from django.http import HttpResponse
from django.utils.timezone import now

def http_consumer(message):
    response = HttpResponse(
        "It is now {} and you've requested {} with {} as request parameters.".format(
            now(),
            message.content['path'],
            dumps(message.content['get'])
        )
    )

    message.reply_channel.send(response.channel_encode())
A message comes through the standard “request.http” channel and our consumer writes a new message with a response through the response channel. Something worth mentioning is that there are two types of channel: the regular ones used to get messages to consumers, and response channels. Only the interface server is listening on these response channels, and it knows which channel is connected to which connection (client) so it know who to send the responses to.

Before we can proceed, we need a way to tell Django to send messages on the “request.http” channel to our brand new consumer. Create a module called “routing.py” next to your settings module:

channel_routing = {
    "http.request": "channels_test.consumers.http_consumer"
}
Now we run the server (with the development server o a WSGI server, doesn’t matter at this point), and make a request to our site:

$ curl http://localhost:8000/some/path?foo=bar
It is now 2016-02-01 11:49:25.166799+00:00 and you've requested /some/path with {"foo": ["bar"]} as request parameters.
We got the response we were expecting. Channels took care of most of the problems for us. Now let’s do something a little more interesting.

Real-time notifications (yet again)
We’ve covered the topic of real-time notifications on several occasions before, and this gives us a great opportunity to cover one of Channels uses cases and compare two solutions side by side.

We’re going to adapt an existing project to track real life events geographically on specific areas of interesting notifying users of the site of the occurrence of such events in (close to) real-time. In that project we used gevent-socketio, SocketIO, and RabbitMQ (and Node.js second attempt). We’re going to do the same now using Channels, regular WebSockets and Redis

As we said, we’re going to use WebSockets to push notifications to the client. Channels has full support of WebSockets. All we need to do is hook-up a few channels to consumers on our “tracker” app:

# routing.py

channel_routing = {
    "websocket.connect": "tracker.consumers.websocket_connect",
    "websocket.keepalive": "tracker.consumers.websocket_keepalive",
    "websocket.disconnect": "tracker.consumers.websocket_disconnect"
}
We’re not interested in the “websocket.message” channel as we’re not going to be receiving messages from the client. Our goal is to send notifications to all connected clients whenever an events occur in an area of interest. This is quite easy to do using a Group. Let’s take a look at out consumers:

# tracker/consumers.py

import logging

from channels import Group
from channels.sessions import channel_session
from channels.auth import channel_session_user_from_http


logger = logging.getLogger(__name__)


# Connected to websocket.connect and websocket.keepalive
@channel_session_user_from_http
def websocket_connect(message):
    logger.info('websocket_connect. message = %s', message)
    # transfer_user(message.http_session, message.channel_session)
    Group("notifications").add(message.reply_channel)

# Connected to websocket.keepalive
@channel_session
def websocket_keepalive(message):
    logger.info('websocket_keepalive. message = %s', message)
    Group("notifications").add(message.reply_channel)


# Connected to websocket.disconnect
@channel_session
def websocket_disconnect(message):
    logger.info('websocket_disconnect. message = %s', message)
    Group("notifications").discard(message.reply_channel)
Whenever a client connects, a message is sent through the “websocket.connect” channel. All we do is then add the reply channel (which comes with the original message), and add it to the “notifications” Groups. Groups allow us to send the same message to all the channels in the group at the same time. All we need to do is keep the set of group channels updated. So whenever a client connects, we add the reply channel to the group and when it disconnects, we remove it. Simple as that. Groups also drop channels after a while, so we use the “websocket.keepalive” channel to add the reply channel to the “notifications” group whenever a keepalive message is received. If the channel was already in the group it won’t be added twice.

Notice that we haven’t sent anything to the Group yet. We need to notify the users whenever an Incident within an AreaOfInterest is reported or updated. We can do that easily using the post_save signals:

# signals.py

import logging

from json import dumps

from django.db.models.signals import post_save, post_delete
from django.dispatch import receiver

from channels import Group

from .models import Incident, AreaOfInterest


logger = logging.getLogger(__name__)


def send_notification(notification):
    logger.info('send_notification. notification = %s', notification)
    Group("notifications").send({'text': dumps(notification)})


@receiver(post_save, sender=Incident)
def incident_post_save(sender, **kwargs):
    send_notification({
        'type': 'post_save',
        'created': kwargs['created'],
        'feature': kwargs['instance'].geojson_feature
    })

    if not kwargs['instance'].closed:
        areas_of_interest = [
            area_of_interest.geojson_feature for area_of_interest in AreaOfInterest.objects.filter(
                polygon__contains=kwargs['instance'].location,
                severity__in=kwargs['instance'].alert_severities,
            )
        ]

        if areas_of_interest:
            send_notification(dict(
                type='alert',
                feature=kwargs['instance'].geojson_feature,
                areas_of_interest=[
                    {
                        'id': area_of_interest['id'],
                        'name': area_of_interest['properties']['name'],
                        'severity': area_of_interest['properties']['severity'],
                        'url': area_of_interest['properties']['url'],
                    }
                    for area_of_interest in areas_of_interest
                ]
            ))


@receiver(post_save, sender=AreaOfInterest)
def area_of_interest_post_save(sender, **kwargs):
    send_notification({
        'type': 'post_save',
        'created': kwargs['created'],
        'feature': kwargs['instance'].geojson_feature
    })


@receiver(post_delete, sender=Incident)
@receiver(post_delete, sender=AreaOfInterest)
def post_delete(sender, **kwargs):
    send_notification({
        'type': 'post_delete',
        'feature': kwargs['instance'].geojson_feature
    })
All the notification magic occurs in send_notification which, as you can see, it’s just a freakin’ line of code! (go ahead, check the older implementations). The rest of the code is the same as before.

So far, we’ve only used the in-memory channels backend, In order for our notification system to work in a multi-server environment, we need to use the database backend (slow, shouldn’t be used for anything other than development) or the Redis back-end. Let’s use the latter. We have to add the following snippet to our settings.py module:

CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "asgi_redis.RedisChannelLayer",
        "CONFIG": {
            "hosts": [("localhost", 6379)],
        },
        "ROUTING": "tracker_project.routing.channel_routing",
    },
}
In order for this backend to work we also need to install the asgi_redis package (in-memory and database layers are included in the default channels package). The last thing we need to change on the Django side is creating an asgi.py module that servers a similar role as wsgi.py does for WSGI servers.

# asgi.py

import os
from channels.asgi import get_channel_layer

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "tracker_project.settings")

channel_layer = get_channel_layer()
This module will be used later to run the interface server.

We also needed to change the client code to use WebSockets instead of of Socket.io. Since the code is pretty much the same (construct socket, hook-up the proper events), we won’t go into details here. You can see the implementation here.

All that is left is running the server. For development purposes, we can use the runserver management command. It has been modified to run the Daphne ASGI server and a worker.

(tracker_project_venv)$ ./manage.py runserver
Worker thread running, channels enabled
Performing system checks...

System check identified no issues (0 silenced).
February 01, 2016 - 13:19:43
Django version 1.8.8, using settings 'tracker_project.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
On a live environment, we would need to run Daphne (or any ASGI compatible server with support for WebSockets) and as many workers as we see fit. Daphne runs with:

(tracker_project_venv)$ daphne tracker_project.asgi:channel_layer
We’re telling Daphne to look for the channel layer on our asgi.py module. Each worker runs with:

(venv)$ python ./manage.py runworker
With everything up and running, you’ll be able to report incidents and receive notifications if the home view is open.

Note on authentication
Something we didn’t mention is that our WebSockets have authentication. The initial step of a WebSocket connection is a regular HTTP request, so we can take advantage of the existing Django session management through the usage of Channel’s @channel_session_user_from_http decorator. If the user is not authenticated (it as no valid session) the consumer won’t run and the reply channel won’t be hooked up to the “notifications” group.

We need to make sure that we pass the session key when constructing the WebSocket:

var socket = new WebSocket('ws://localhost:8000?session_key=' + sessionKey);
It is rudimentary, but it gets the job done. There are far better options for WebSocket authentication (I’m partial to token based systems like JWT), but covering them might take a blogpost of its own.

Conclusions
Django channels is going to change the way we work dramatically. It will make our lives easier, and it will allow us to tackle a wider range of problems with a Django site, but one of the neatest features of the project is that IT IS ENTIRELY OPTIONAL. In our example, we only routed the channels that we were interested in. We didn’t have to change our existing request-response based code. We get the best of both worlds.

Channels is not ready for a production environment, but once it is integrated into the next versions of Django, it won’t take long for it to mature into a stable framework. Another cool thing is that is going to be back-ported to Django 1.8 as an external app, so you’ll be able to integrate it into your existing sites.

Vagrant
A Vagrant configuration file is included if you want to test the solutions.

Feedback
As usual, I welcome comments, suggestions and pull requests.
