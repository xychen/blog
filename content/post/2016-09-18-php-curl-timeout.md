---
title: "排查php中curl请求超时问题"
date: 2016-09-18 09:00:00
categories: ["php"]
tags: ["strace", "php"]
---

### 问题描述  

一个项目需要抓取一些网页数据，使用的是php中的curl扩展，模拟浏览器行为，通过http协议请求web数据。

今天收到了短信报警，爬虫任务队列（redis队列）堆积，于是登录到线上服务器紧急排查问题，发现有大量的进程“僵死”，导致任务处理缓慢。

我们的服务使用的是php多进程方式，master进程用于管理控制，worker进程用于实际任务处理，为避免内存泄漏，worker进程每处理1000条数据或者运行超过5分钟，就会主动退出，然后master会fork出新的worker进程填补，保证总worker进程数等于预设值。

通过ps命令查看，发现好多进程已经运行了好几个小时，肯定是不正常的。当大量进程“僵死”,这些进程不工作也不退出，master也就无法创建新的worker进程，因此整体处理能力下降，导致队列堆积。

### 问题排查  

首先，使用strace命令查看“僵死”进程当前的系统调用：  

```
strace -p pid
```

看到的信息如下：  

```
……
……
poll([{fd=38, events=POLLIN|POLLPRI|POLLRDNORM|POLLRDBAND}], 1, 0) = 0 (Timeout)
poll([{fd=38, events=POLLIN|POLLPRI|POLLRDNORM|POLLRDBAND}], 1, 0) = 1000 (Timeout)
……
……
```

由此可以初步判断，是在进行网络资源请求(poll)的时候超时了，资源描述符为38，进程阻塞在这里，不能继续执行。
使用lsof命令查看一下38号描述符具体是请求哪个资源：  

```
lsof -p pid
```

得到的信息如下：

```
……(其它的省略)
php     1000 root   38u  IPv4         3735516592      0t0        TCP ip-xxxx.compute.internal:21564->blob.db5prdstr04a.store.core.windows.net:http (ESTABLISHED)
……(其它的省略)
```

第4列表示的是描述符，找到38号描述符，可以清晰的看到，这是一个http请求，在请求数据的时候产生了超时。而我们的http请求都是通过php的curl扩展实现的，因此可以定位到问题发生在curl相关的参数设置上。

### 解决问题

#### 一、先以最快的方式解决  
第一想到的是重启相关进程。代码逻辑中设置了信号处理函数，先让正常的进程平滑退出，而这些异常进程，由于阻塞在这里，因此并不会执行到信号处理函数的逻辑，因此只能强制杀死：  

```
ps -ef | grep xxxx.daemon.php | grep -v grep | awk '{print $2}' | xargs kill -9
```
强杀之后，重新启动服务，队列里中的数据逐渐被消耗掉。

#### 二、设置curl超时时间
通过google搜索以及查询文档，知道php的curl扩展中有2个参数可以设置超时时间：  

```
//设置连接超时时间
curl_setopt($handle, CURLOPT_CONNECTTIMEOUT, 20);
//设置请求超时时间
curl_setopt($handle, CURLOPT_TIMEOUT, 20);
```

修复bug后上线。然而，事情并没有这样简单的结束，过了一段时间，又报警了。汗……

#### 三、借助工具，使问题复现
进一步排查代码，超时时间都已经设置了，为什么还会出现这种情况？于是又通过google搜索问题，大多数人遇到此类问题时都是通过设置超时时间解决的，排查代码也觉得没什么问题。

焦头烂额之际，看到了这篇文章：<http://www.jianshu.com/p/8a247cae629a>。

解决方案也是设置超时时间，但是里边介绍了一种复现问题的方式:  


```
//1.服务器端使用nc命令模拟web serve
nc -l 20000
//2.客户端模拟http请求,假设服务器端ip地址为： 10.0.0.1，此处以curl命令做说明
curl http://10.0.0.1:20000
```

此时，服务器端会显示：  

```
GET / HTTP/1.1
User-Agent: curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.19.1 Basic ECC zlib/1.2.3 libidn/1.18 libssh2/1.4.2
Host: 10.0.0.1:20000
Accept: */*
```

由于服务器端仅仅是监听了端口，并不会输出任何数据，客户端进程进行http请求时没有设置超时时间，因此客户端进程会阻塞。使用strace -p命令可以清楚的看到客户端进程http请求的时候出现Timeout的情况，和最开始描述的线上问题是一样的，如果不停止它，它会一直阻塞在那里：  

```
Process 4368 attached
restart_syscall(<... resuming interrupted call ...>) = 0
poll([{fd=3, events=POLLIN|POLLPRI|POLLRDNORM|POLLRDBAND}], 1, 0) = 0 (Timeout)
poll([{fd=3, events=POLLIN|POLLPRI|POLLRDNORM|POLLRDBAND}], 1, 1000) = 0 (Timeout)
poll([{fd=3, events=POLLIN|POLLPRI|POLLRDNORM|POLLRDBAND}], 1, 0) = 0 (Timeout)
poll([{fd=3, events=POLLIN|POLLPRI|POLLRDNORM|POLLRDBAND}], 1, 1000^CProcess 4368 detached
 <detached ...>
```

我们只需要把curl命令模拟的http请求，替换成我们使用php的curl扩展开发的代码，就可以用于问题复现。通过调试，发现另外一处的代码没有设置超时时间(之前排查不够仔细)，修改之，至此，问题才真正解决。

这里用到的原理是：服务器端使用nc命令监听端口，并不输出内容，客户端模拟http请求，由于不设置超时时间，因此拿不到服务器端返回的数据会一直阻塞。如果我们设置服务器端输出数据，则客户端不会阻塞，比如：  

```
nc -l 20000 < 1.txt
```

当客户端请求时，会得到1.txt中的内容，并结束进程，不会阻塞。

### 总结

首先，先进行自我反思：

- 其一，排查代码不够仔细，有遗漏的地方 

- 其二，阅读文章比较草率，其实第一轮排查的时候，看到过这篇文章，只不过当时读的不够仔细，没有看到这个复现方法  

- 其三，要多读书，网络相关的知识还比较薄弱

然后，解决问题要讲究方法，深入其中，发现问题本质，借助有效工具。本文虽然轻描淡写，其实解决问题的过程可谓一波三折。

最后，不得不赞叹nc工具的强大，短短的一段时间内，我已经2次用到了这个工具（前一篇：[使用netcat与服务器间传输文件](/archivers/netcat_usage)  ），强！

