---
layout: post
title:  "a web crawler use asyncio"
date:   2016-12-15 18:21:05
categories: web-programming
tags: Python socket 500lines
---

* content
{:toc}

https://github.com/aosabook/500lines




## 前言

这个项目来自于github上“500 lines or less”系列，是一个小型开源项目的合集，在同步出版的书《The Architecture of Open Source Applications》上可以看到对这些项目详细的分析，在线阅读该书的地址在[这里](http://aosabook.org/en/index.html)。

这个名叫crawler的项目实现了一个简单的网络爬虫，传统的计算机程序总是专注于优化算法复杂度，但是网络编程时，消耗时间的往往不是计算，而是保持多个缓慢、而频率又不高的连接……在这类程序中我们要面对的挑战是：**高效的等待并处理大量的网络请求**。如果对每个请求都开一个线程去处理，则资源很快会被耗尽，这里crawler用了异步的I/O来解决。

## 传统方法
```
def fetch(url):
    sock = socket.socket()
    sock.connect(('xkcd.com', 80))
    request = 'GET {} HTTP/1.0\r\nHost: xkcd.com\r\n\r\n'.format(url)
    sock.send(request.encode('ascii'))
    response = b
    chunk = sock.recv(4096)
    while chunk:
        response += chunk
        chunk = sock.recv(4096)
    # Page is now downloaded.
    links = parse_links(response)
    q.add(links)
```
这是一个传统的线程池实现，threads are expensive，一个python的thread约占50k内存，同时启用上万个线程就会导致失败，**这一点会发生在socket（启用过多而失败）之前**，因此这里每个thread的overhead和系统限定的thread数量上限成为瓶颈！

## 异步

异步的I/O框架要求非阻塞的socket：
```
sock = socket.socket()
sock.setblocking(False)
try:
    sock.connect(('xkcd.com', 80))
except BlockingIOError:
    pass
```
接下来我们可以通过发http请求来确认连接已建立：
```
request = 'GET {} HTTP/1.0\r\nHost: xkcd.com\r\n\r\n'.format(url)
encoded = request.encode('ascii')

while True:
    try:
        sock.send(encoded)
        break  # Done.
    except OSError as e:
        pass

print('sent')
```

这种方法本身很直观，但是在应对多个socket通信时效率很低，BSD Unix中使用一个叫select的C函数来解决这个问题，目前也不知道是怎么实现的……不过可以先用着。Python的DefaultSelector库中对此有支持，其中connected函数是回调函数，传入的EVENT_WRITE函数表示：我们想知道**这个socket什么时候能够写**，connected定义了这个时刻出现时的行为。

```
from selectors import DefaultSelector, EVENT_WRITE

selector = DefaultSelector()

sock = socket.socket()
sock.setblocking(False)
try:
    sock.connect(('xkcd.com', 80))
except BlockingIOError:
    pass

def connected():
    selector.unregister(sock.fileno())
    print('connected!')

selector.register(sock.fileno(), EVENT_WRITE, connected)
```
**我们在一个循环里处理I/O notifications**，其实这里我有点不理解，先跟着走吧：
```
def loop():
    while True:
        events = selector.select()
        for event_key, event_mask in events:
            callback = event_key.data
            callback()
```

好了，到这里可以稍微做个总结（总算来了-_-|||），

>以上这些主要说明了什么呢——一个异步框架就建立在**非阻塞的socket**和**event循环**里——这样子就实现了在单个thread里并行的处理事件。这里指的并不是严格意义上的同步计算，而是做I/O层面的overlapping。

以上这段来自我个人的拙劣翻译，目前我对此的理解很有限，姑且先粗略的认知为**流水线作业**，在上一个任务进入下一阶段时，紧接着启动下一个任务。好吧总之这样做的目的在于优化之前提到的thread瓶颈，因为自己对多线程编程其实很不熟悉，对于优化的方向和瓶颈的理解都很模糊，所以也希望能在学习这个项目的过程中能有所成长吧╮(￣▽￣")╭。
