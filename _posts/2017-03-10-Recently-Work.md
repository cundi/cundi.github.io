---
layout: post
title: ""
categories: 工作
tags: 工作 
---

* content
{toc}

最近一直在忙着公司的手机小号业务，总结下用到的东西。  

这个业务可以说是JS全栈。前端用AngularJS+MySQL(持久化) 后端 NodeJS+express+redis（临时存储）。日志文件用winston。底层的SYS模块使用mocha+chai+sinon做BDD测试。当然这个架构都是老师傅写好的，目前负责测试和部署顺便熟悉代码以便后期接手维护。  

期间遇到很多难点，主要是对ES6的新特性缺乏了解，比如，生成器，比如，Promise。  

测试是从底层的lua脚本开始: 

```shell
redis-cli --ldb --eval ./prepare.lua HuaWeiAXB 'cp_cmd' hw 13157204810 13157204812 13157204813 13157204811
```

使用--ldb 可以在沙箱环境下执行lua脚本，而不会真正写入Redis。

Node后端记录的时间戳精确到毫秒，而Angular模板默认解析的到秒，要想页面上显示毫秒得使用模板过滤器：

```html
<tr ng-repeat="item in searchResult">
    <td>{{item.startAt | date:'yyyy-MM-dd HH:mm:ss:sss'}}</td>
    <td>{{item.endAt | date:'yyyy-MM-dd HH:mm:ss:sss'}}</td>
    <!--<td>{{toFormatTime(item.startAt)}}</td>-->
    <!--<td>{{toFormatTime(item.endAt)}}</td>-->
    <td ng-bind="item.partya"></td>
    <td ng-bind="item.partyb"></td>
    <td ng-bind="item.partyx"></td>
    </tr>
```

后期为了性能和扩展性上docker集群，rabiitmq。  
