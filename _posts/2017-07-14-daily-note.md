---
layout: post
title: "工作日志"
categories: Python Ansible
tags: Ansible
---

* content
{toc}

# 读取本地文件并赋值给变量

```yaml
- hosts: all
  vars:
     contents: "{{ lookup('file', '/etc/foo.txt') }}"
  tasks:
     - debug: msg="the value of foo.txt is {{ contents }}"
```
