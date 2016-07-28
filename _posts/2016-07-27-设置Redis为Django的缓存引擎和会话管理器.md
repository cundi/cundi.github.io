---
layout: post
title: "设置Redis为Django的缓存引擎和会话管理器"
categories: Django Redis
tags: Django Redis Session 翻译
---

* content
{:toc}

## Configure Redis as Django's Cache Engine and Session Storage Manager
`原始链接：http://www.craigderington.me/configure-redis-as-djangos-cache-engine/`  

Redis is an open-source, persistent, in-memory data structure server that stores it's data in key-value pairs. Redis can be used as a database, cache or message broker. Redis data structures support a variety of formats including strings, lists, sets, hashes and sorted sets. More advanced data structures include bitmaps, hyper-logs, and geo-spatial index with radius queries. Redis has built in replication, Lua scripting and transactions. It offers different levels on on-disk persistence and provides high availability and automatic partitioning.  

Redis是一个开源的，持久化的，内存数据结构服务器，




Due to it's fast, in-memory data storage retrieval, Redis is an optimal solution to be used with Django; as both a cache engine and for session storage. The primary benefit is storing cache and session data separate from your application database, which prevents unnecessary data read and write operations and can provide a nice performance boost for your Django applications.  

## Installing Redis
Redis is written in C and works with Unix, BSD, Linux and OS X. For our test case, we will be installing Redis on Ubuntu 14.04. OK, let's get cooking...  

Redis使用C编写，可以运行于Unix，BSD，Linux和OS X。

```shell
sudo apt-get update  
sudo apt-get install redis-server  
```

Once Redis is finished downloading and installing, check the redis-server for it's version info to confirm it was installed correctly and is up and running.  

```shell
redis-server --version  
Redis server v=2.8.4 sha=00000000:0 malloc=jemalloc-3.4.1 bits=64 
build=a44a05d76f06a5d9  
```

## Configure Redis
I like to connect to Redis via a Unix socket instead of TCP. This simply alleviates Redis of the overhead of managing TCP connections.  

Configure Redis to communicate and accept direct socket connections, edit the /etc/redis/redis.conf file and comment out the BIND and PORT directives and uncomment the unixsocket directives.  

```
#comment out
#port 6379
#bind 127.0.0.1

#uncomment
unixsocket /var/run/redis/redis.sock  
unixsocketperm 700  
```

Save your changes and restart the Redis server.  

```shell
sudo service redis-server restart 
```

After restarting the redis-server, check to make sure the server is listening and accepting connections.  

```
redis-cli  
> ping
PONG
```

Success! Our Redis server is up and running.  

## Use Redis with Django
You will need to install the django-redis-cache module via pip in your Django project's virtual environment in order for Django to use the Redis server for caching and session storage.  

```shell
$ source .env/bin/activate
(.env)$ pip install django-redis-cache
```

Then add the following Redis config statement to settings.py.  

```python
CACHES = {  
    'default': {
        'BACKEND': 'redis_cache.RedisCache',
        'LOCATION': '/var/run/redis/redis.sock',
    },
}
```

You will also need to add the following lines to your Django Middleware classes section in settings.py:  

```python
MIDDLEWARE_CLASSES = (  
    'django.middleware.cache.UpdateCacheMiddleware',
     ...
    'django.middleware.cache.FetchFromCacheMiddleware',
)
```

Ensure the UpdateCacheMiddleware is listed first and the FetchFromCacheMiddleware is last in the Middleware classes tuple.  

Now, launch your Django development server to take advantage of your new cache engine.  

## Using the Django cache
Django's cache engine is extremely flexible and can be configured to cache the entire website or a single view. The behavior is controlled by using a decorator. As an example, we'll cache the following view for 15 minutes...  

```python
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)
def index(request):  
   ...
```

## Use Django Redis Cache Inside Functions
A nice feature of the cache engine is the ability to store data for fast and reliable retrieval...  

```python
from django.core.cache import cache

#store some data
cache.set('craig', 'derington')

#now retrieve the key's value
cache.get('craig')

#store a dict
cache.set_many({'a': 1, 'b': 2, 'c': 3})

#get the values
cache.get_many(['a', 'b', "c"])

#store more complex data structures
cache.set('my-key', {  
    'a string': 'this is a simple string',
    'an integer': 123,
    'one list': [9, 10, 11, 12, 13],
    'this tuple': (1, 2, 3, 4),
    'my dict': {'a': 26, 'b': 25, 'c': 24},
})

#get the values
cache.get('my-key')
```

### Use Redis for Django Session Storage
Since we are using Redis as our caching engine, it only makes sense to use it to store our user's session data. Again, keeping this out of the application database has many performance improvement benefits.  

In the settings.py, add this line:  

```python
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
```

If you'd prefer to write the session information to the database and only load it from the Redis cache, use this option:  

```python
SESSION_ENGINE = 'django.contrib.sessions.backends.cached_db'
```

This will make sure the session data is persistent and can survive a server restart or Redis server reload.  

## Alternative Session Data Manager without Cache
You can also use Redis to store just the user session data without the need to use the caching engine.  

Install django-redis-sessions:  

```shell
(.env)$ pip install django-redis-sessions
```

And append the following to your Django settings.py file.  

>Since we are using a unix socket to connect to Redis, you must include the path.

```python
SESSION_ENGINE = 'redis_sessions.session'  
SESSION_REDIS_UNIX_DOMAIN_SOCKET_PATH = '/var/run/redis/redis.sock'  
```

## Wrapping Up
I like to visualize the data in my Redis cache. There are a number of utilities that provide a nice UI to help you visualize your Redis data. For now, I just use the command line.  

```shell
$redis-cli
> info keyspace
# Keyspace
db0:keys=16, expires=0, avg_ttl=0  
db1:keys=4,expires=1,avg_ttle=30411590898  
```

To see the individual keys in the keyspace...  

```shell
$ select 1
[1] keys *
1) "redis"  
2) "cache"  
3) "session"  
4) "django1"  
OK  
```

Redis offers many benefits for Django developers. Use it as a caching engine to cache your website and as a session storage data store. The performance improvements will be quite noticeable as your site scales. You will also notice a decrease in the number of queries that are executed when using the debug_toolbar in your development environment.  

For more information about Redis or available Redis commands, [click here](http://redis.io/commands).  

Redis GUI's:   
1. redsmin 
2. redis-command 
3. redis desktop manager 
4. redis-browser

