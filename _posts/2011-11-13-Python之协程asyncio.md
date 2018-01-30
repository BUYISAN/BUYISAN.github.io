---
layout: post
title:  "Python之协程asyncio"
date:   2017-11-13 00:00:00
categories: Python
excerpt: 
---

* content
{:toc}

### 背景知识
网络模型有很多中，为了实现高并发也有很多方案，多线程，多进程。无论多线程和多进程，IO的调度更多取决于系统，而协程的方式，调度来自用户，用户可以在函数中yield一个状态。使用协程可以实现高效的并发任务
- 多进程、多线程
- 同步、异步调用
- 阻塞、非阻塞
关于同步、异步、阻塞|非阻塞详见[Python中关于同步异步、阻塞非阻塞的理解](https://www.jianshu.com/p/47ee57646369)

### 同步 异步 回调机制
同步调用:即提交一个任务后就在原地等待任务结束,等到拿到任务结果后再执行下一段代码效率低下
#### 解决方案一
IO密集型任务,使用多线程或多进程,让每个任务都以一个独立的线程或进程,这样任何一个任务阻塞都不会影响其他任务
~~~
开启多进程或都线程的方式，我们是无法无限制地开启多进程或多线程的：在遇到要同时响应成百上千任务请求，则无论多线程还是多进程都会严重占据系统资源，降低系统对外界响应效率，而且线程与进程本身也更容易进入假死状态。
~~~

#### 解决方案二
线程池和进程池+异步调用:提交一个任务后并不会等待任务结束,而是继续下一行代码.
“线程池”旨在减少创建和销毁线程的频率，其维持一定合理数量的线程，并让空闲的线程重新承担新的执行任务。
~~~
“线程池”和“连接池”技术也只是在一定程度上缓解了频繁调用IO接口带来的资源占用。而且，所谓“池”始终有其上限，当请求大大超过上限时，“池”构成的系统对外界的响应并不比没有池的时候效果好多少。所以使用“池”必须考虑其面临的响应规模，并根据响应规模调整“池”的大小。
~~~
#### 解决方案三 --协程(asyncio)
上述方案都没有解决一个性能相关的问题:IO阻塞,无论是多进程还是多线程,在遇到IO阻塞是都会被操作系统强行夺走走CPU的执行权限，程序的执行效率因此就降低了下来。

 解决这一问题的关键在于，我们自己从应用程序级别检测IO阻塞然后切换到我们自己程序的其他任务执行，这样把我们程序的IO降到最低，我们的程序处于就绪态就会增多，以此来迷惑操作系统，操作系统便以为我们的程序是IO比较少的程序，从而会尽可能多的分配CPU给我们，这样也就达到了提升程序执行效率的目的.

#### asyncio
`asyncio`的编程模型就是一个消息循环。我们从`asyncio`模块中直接获取一个`EventLoop`的引用，然后把需要执行的协程扔到`EventLoop`中执行，就实现了异步IO。
先看一段代码吧.然后再补充一些概念
~~~
import asyncio

# @asyncio.coroutine 把一个generator标记为coroutine类型,我们可以把这个coroutine扔到EventLoop中执行
@asyncio.coroutine
def task(task_id, seconds):
    print('%s is start' % task_id)
    # 异步调用asyncio.sleep(...)
    yield from asyncio.sleep(seconds)
    print('%s is end' % task_id)

tasks = [task('任务1', seconds=3), task('任务2', seconds=2)]

# 获取EventLoop
# 事件循环,程序开启一个无限的循环,把一些函数注册到事件循环中,当满足事件发生时,调用相应的协程函数
loop = asyncio.get_event_loop()
# 执行coroutine
loop.run_until_complete(asyncio.wait(tasks))
loop.close()
~~~

`yield from`语法可以让我们方便地调用另一个`generato`r。由于asyncio.sleep()也是一个`coroutine`，所以线程不会等待asyncio.sleep()，而是直接中断并执行下一个消息循环。当asyncio.sleep()返回时，线程就可以从yield from拿到返回值（此处是None），然后接着执行下一行语句。
把asyncio.sleep()看成是一个耗时second秒的IO操作，在此期间，主线程并未等待，而是去执行`EventLoop`中其他可以执行的`coroutine`了，因此可以实现并发执行。

- event_loop 事件循环：
程序开启一个无限的循环，程序员会把一些函数注册到事件循环上。当满足事件发生的时候，调用相应的协程函数。
- coroutine 协程：
协程对象,协程对象需要注册到事件循环，由事件循环调用。
- task 任务：
一个协程对象就是一个原生可以挂起的函数，任务则是对协程进一步封装，其中包含任务的各种状态。
### 协程获取 网站请求
~~~
import asyncio
import requests
import uuid
user_agent='Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/49.0.2623.221 Safari/537.36 SE 2.X MetaSr 1.0'

def parse_page(host,res):
    print('%s 解析结果 %s' %(host,len(res)))
    with open('%s.html' %(uuid.uuid1()),'wb') as f:
        f.write(res)

@asyncio.coroutine
def get_page(host,port=80,url='/',callback=parse_page,ssl=False):
    print('下载 http://%s:%s%s' %(host,port,url))

    #步骤一（IO阻塞）：发起tcp链接，是阻塞操作，因此需要yield from
    if ssl:
        port=443
    recv,send=yield from asyncio.open_connection(host=host,port=443,ssl=ssl)

    # 步骤二：封装http协议的报头，因为asyncio模块只能封装并发送tcp包，因此这一步需要我们自己封装http协议的包
    request_headers="""GET %s HTTP/1.0\r\nHost: %s\r\nUser-agent: %s\r\n\r\n""" %(url,host,user_agent)
    # requset_headers="""POST %s HTTP/1.0\r\nHost: %s\r\n\r\nname=egon&password=123""" % (url, host,)
    request_headers=request_headers.encode('utf-8')

    # 步骤三（IO阻塞）：发送http请求包
    send.write(request_headers)
    yield from send.drain()

    # 步骤四（IO阻塞）：接收响应头
    while True:
        line=yield from recv.readline()
        if line == b'\r\n':
            break
        print('%s Response headers：%s' %(host,line))

    # 步骤五（IO阻塞）：接收响应体
    text=yield from recv.read()

    # 步骤六：执行回调函数
    callback(host,text)

    # 步骤七：关闭套接字
    send.close() #没有recv.close()方法，因为是四次挥手断链接，双向链接的两端，一端发完数据后执行send.close()另外一端就被动地断开
if __name__ == '__main__':
    tasks=[
        get_page('www.baidu.com',url='/s?wd=美女',ssl=True),
        get_page('www.cnblogs.com',url='/',ssl=True),
    ]

    loop=asyncio.get_event_loop()
    loop.run_until_complete(asyncio.wait(tasks))
    loop.close()
asyncio+自定义http协议报头
~~~
