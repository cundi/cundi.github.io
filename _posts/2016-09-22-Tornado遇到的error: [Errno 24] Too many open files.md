---
layout: post
title: "2016-09-22-Tornado遇到的error: [Errno 24] Too many open files"
categories: Tornado
tags: tonado nginx 
---

* content
{toc}

部署到服务器上Web应用－微信遥控器，在运行一段时间，访问出现如下情况：

```
504 Gateway Time-out

nginx/1.2.9
```

StackOverFlow上的找到的解答是, 重新设置ulimit ,查看本机：




```
[root@AY1404272234249537e7Z logs]# ulimit -n
1024
[root@AY1404272234249537e7Z logs]# ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 14875
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 10240
cpu time               (seconds, -t) unlimited
max user processes              (-u) 14875
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```

不过有人设置的非常大的进程打开文件上限，还是会出现问题。

Tornado的网页连接过多：

```python
Tornado “error: [Errno 24] Too many open files” error 
```

解决是在HTTPServer实例化时添加关键字参数no_keep_alive=True。

```python
class Application(tornado.web.Application):
    def __init__(self):
        ...

http_server = tornado.httpserver.HTTPServer(Application(),no_keep_alive=True)
http_server.listen(port)
tornado.ioloop.IOLoop.instance().start()
```

