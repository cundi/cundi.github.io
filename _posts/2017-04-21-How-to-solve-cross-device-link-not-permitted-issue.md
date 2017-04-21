---
layout: post
title:  "解决cross-device link not permitted问题"
categories: Docker
tags: Docker Dockerfile npm
---

* content
{:toc}




这两天准备把nodejs项目装进docker。

遇到很多坑，先是apt install的node版本低，装不了gulp。

只有从官网源码安装。dockerfile如下：

```docker
FROM ubuntu:16.04 
COPY sources.list /etc/apt/sources.list
RUN mkdir -p /root/agent/ /root/Kandy.js/ /root/kandy-redux /var/run/sshd
COPY agent /root/agent
COPY Kandy.js /root/Kandy.js/
COPY kandy-redux /root/kandy-redux/


RUN apt-get update -y && apt-get install --no-install-recommends -y -q curl python build-essential git ca-certificates openssh-server
RUN mkdir /nodejs && curl http://nodejs.org/dist/v4.8.2/node-v4.8.2-linux-x64.tar.gz | tar xvzf - -C /nodejs --strip-components=1

RUN echo 'root:123456' | chpasswd

ENV PATH $PATH:/nodejs/bin
RUN npm install -g npm --prefix=/usr/local
RUN ln -s -f /usr/local/bin/npm /usr/bin/npm
RUN npm install gulp webpack webpack-dev-server -g

RUN cd /root/Kandy.js && npm link
RUN cd /root/kandy-redux && npm link kandy-js && npm link
RUN cd /root/agent/ && npm link kandy-redux


RUN ls -al $(npm root -g)

EXPOSE 4322:22 60800:60800
CMD ["/user/sbin/sshd", "-D"]
ENTRYPOINT ["/bin/sh", "/root/agent/start.sh"]
```

在Docker内执行npm link出现个不小的问题：

```
npm ERR! EXDEV: cross-device link not permitted,
```

I found two solutions on the web， here there are： 

1. 
```
RUN cd $(npm root -g)/npm 
&& npm install fs-extra 
&& sed -i -e s/graceful-fs/fs-extra/ -e s/fs.rename/fs.move/ ./lib/utils/rename.js
```

2. 
```
# to circumvent the cross-device link not permitted issue
RUN npm instaSolvell -g npm --prefix=/usr/local
RUN ln -s -f /usr/local/bin/npm /usr/bin/npm
```

目前使用方法2解决了这个问题，如果没有预期效果的话可以换方法1试一试。


