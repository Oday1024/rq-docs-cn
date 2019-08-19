

# 简介

RQ (*Redis Queue*)是一个简单的Python库，用于队列任务并在后台与工人（worker）一起处理它们。它由Redis提供支持，旨在降低入门门槛。它可以轻松集成到您的Web堆栈中。

RQ 要求 Redis >= 3.0.0.

## 开始

首先，运行Redis服务。你可以使用现有的。将任务放在队列中，你不必执行任何特殊操作，只需定义一般冗长或阻塞的函数：

```python
import requests

def count_words_at_url(url):
    resp = requests.get(url)
    return len(resp.text.split())
```

然后，创建一个RQ队列：

```python
from redis import Redis
from rq import Queue

q = Queue(connection=Redis())
```

再将函数调用放入队列中：

```python
from my_module import count_words_at_url
result = q.enqueue(
             count_words_at_url, 'http://nvie.com')
```

## Worker

在后台开始执行入队函数，从项目的目录中启动一个worker：

```sh
$ rq worker
*** Listening for work on default
Got count_words_at_url('http://nvie.com') from default
Job result = 818
*** Listening for work on default
```

就是这样。

## 安装

只需使用以下命令安装最新发布的版本：

```sh
pip install rq
```

如果你想要最新的版本（可能会破坏），请使用：

```sh
pip install -e git+git@github.com:nvie/rq.git@master#egg=rq
```

## 项目历史

本项目的灵感来自[Celery](http://www.celeryproject.org/)，[Resque](https://github.com/defunkt/resque)和[这个代码段](http://flask.pocoo.org/snippets/73/)的优点，作为现有队列框架的轻量级替代品，它具有较低的入门门槛。

------

RQ由[Vincent Driessen](http://nvie.com/about)撰写

在[BSD](https://raw.github.com/nvie/rq/master/LICENSE)许可条款下开源