---
title: "supervisor在PHP项目中的使用"
date: 2017-12-16 20:00:00
categories: ["php"]
tags: ["php", "supervisor"]
---

## 使用背景  
很多场景下，我们需要使用PHP开发一些脚本，用于处理离线数据。常用的实现方式有以下几种:  

1.crontab：这种方式适用于定时任务，对于数据量较大时不太适合

2.循环+后台运行： 启动一个脚本，让其在后台运行，脚本中是一个循环，可以一直处理任务。如果需要多个进程，就多启动几次脚本，简单粗暴。   

```bash
nohup /usr/bin/php dojob.php > /dev/null 2>&1 &
```

3.PHP实现daemon+多进程： 类似nginx、php-fpm的多进程方式，通过fork实现多进程，master进程负责管理，worker进程负责处理具体的任务。

4.supervisor管理： 原理和多进程方式类似，supervisor充当一个master进程的角色，相对于PHP自己实现多进程，使用起来更简单一些。
本文将重点讨论supervisor的方式。

## supervisor配置

supervisor的安装我就不再多说，网上有很多教程，可以自行查找。如下，我直接贴过来一个配置：

```
[program:test]
directory = /data/app/
command = /usr/bin/php test.php
process_name = %(program_name)s_%(process_num)02d
numprocs = 1        //启动的进程数
autostart = true    //supervisord启动的时候，是否自动启动这个任务
autorestart = true  //进程退出后，是否自动重启该进程
startsecs = 1
startretries = 3
exitcodes = 0,2
stopsignal = QUIT   //退出时，supervisor给进程发送的信号，可以改成其他的
stopwaitsecs = 10
stopasgroup=true
user = test
redirect_stderr = false
stdout_logfile_backups = 90
stdout_logfile = /data/logs/qrcode.log
environment = USER="test",APP_ENV="test"
```

我会重点对我备注的这几个配置项如何在PHP项目中正确使用进行说明，并进行代码展示。
假设的场景，test.php脚本文件的作用是从redis队列中读取数据，并处理数据:  

```
$job = new Job();
$job->handle();

class Job
{
    private $redis;
    public function __(){
        $this->redis = new \Redis();
        $this->redis->connect('127.0.0.1', 6379);
    }

    //处理函数
    public function handle()
    {
        while(true) {
            $data = $this->redis->lPop('QUEUE:JOB');
            if(!$data) {
                sleep(5);
                continue;
            }
            //do data
            //比如：将数据存储到文件中……
            file_put_contents('/data/data.txt', $data, FILE_APPEND);
        }
    }
}
```



## 信号处理

```
stopsignal = QUIT   //退出时，supervisor给进程发送的信号
```

当执行supervisorctl stop test操作时,supervisor的管理进程会发出一个 QUIT 信号。test.php脚本中没有处理这个信号，那么test.php进程会直接退出，这样就会有一个问题，假设进程正在写文件，直接退出了这条数据就写失败了。正确的做法是，让进程处理完本次写入再退出：  

1.设置一个while循环的变量 (loop变量，初始值为true)  
2.绑定信号处理函数: (pcntl_signal)  
3.while循环中，每次循环进行一次信号分发 (pcntl_signal_dispatch)  
4.当接收到相应信号时，将loop变量设置为false，那么在下一次循环时，函数将退出执行  

```php
$job = new Job();
$job->handle();

class Job
{
    private $redis;
    private $loop = true;
    public function __()
    {
        //信号安装
        $this->installSignal();
        $this->redis = new \Redis();
        $this->redis->connect('127.0.0.1', 6379);
    }

    //处理函数
    public function handle()
    {
        while($this->loop) {
            //信号分发
            pcntl_signal_dispatch();
            $data = $this->redis->lPop('QUEUE:JOB');
            if(!$data) {
                sleep(5);       //spare sleep
                continue;
            }
            //do data
            //比如：将数据存储到文件中……
            file_put_contents('/data/data.txt', $data, FILE_APPEND);
        }
    }

    //信号安装
    function installSignal()
    {
        //绑定QUIT信号
        pcntl_signal(SIGQUIT, array($this, 'sigalHndler'));
        pcntl_signal(SIGTERM, array($this, 'sigalHndler'));
        pcntl_signal(SIGHUP, array($this, 'sigalHndler'));
        pcntl_signal(SIGINT, array($this, 'sigalHndler'));

    }

    //信号处理函数
    private function sigalHndler($signo)
    {
        switch ($signo) {
            case SIGTERM:
            case SIGHUP:
            case SIGINT:
            case SIGQUIT:
                $this->loop = false;
                break;
            default:
                // 处理所有其他信号
        }
    }
}
```


## 定时重启

nginx和php-fpm中有一个机制，worker进程在运行一段时间后，会主动退出，释放资源，master进程会再创建出新的worker进程继续运行，可以在一定程度上避免内存泄漏。supervisor中也可以实现这种功能。

1.首先在supervisor的配置文件中设置autorestart为true

```php
autorestart = true  //进程退出后，是否自动重启该进程
```


2.在脚本中添加时间检测的逻辑

至此，我们已经完成了supervisor管理php进程的功能。完整代码如下:

```
$job = new Job();
$job->handle();

class Job
{
    const MAX_RUN_TIME = 3600;  //最长运行1小时
    private $redis;
    private $loop = true;
    private $startTime;

    public function __()
    {
        //信号安装
        $this->installSignal();
        $this->redis = new \Redis();
        $this->redis->connect('127.0.0.1', 6379);
    }

    //处理函数
    public function handle()
    {
        $this->startTime = time();
        while($this->loop) {
            //检查运行时间
            $this->checkRunTime();
            //信号分发
            pcntl_signal_dispatch();
            $data = $this->redis->lPop('QUEUE:JOB');
            if(!$data) {
                sleep(5);       //spare sleep
                continue;
            }
            //do data
            //比如：将数据存储到文件中……
            file_put_contents('/data/data.txt', $data, FILE_APPEND);
        }
    }

    //检查运行时间
    private function checkRunTime()
    {
        if (time() - $this->startTime > self::MAX_RUN_TIME) {
            $this->loop = false;
        }
    }

    //信号安装
    function installSignal()
    {
        //绑定QUIT信号
        pcntl_signal(SIGQUIT, array($this, 'sigalHndler'));
        pcntl_signal(SIGTERM, array($this, 'sigalHndler'));
        pcntl_signal(SIGHUP, array($this, 'sigalHndler'));
        pcntl_signal(SIGINT, array($this, 'sigalHndler'));

    }

    //信号处理函数
    private function sigalHndler($signo)
    {
        switch ($signo) {
            case SIGTERM:
            case SIGHUP:
            case SIGINT:
            case SIGQUIT:
                $this->loop = false;
                break;
            default:
                // 处理所有其他信号
        }
    }
}
```

## 总结

使用supervisor管理php脚本,要做好以下几点：  

- 信号处理：是程序优雅的退出  
- 主动重启：php的运行进程工作一段时间后重新退出，supervisor启动新的进程继续工作  


## 参考列表

- [使用 supervisor 管理进程](http://liyangliang.me/posts/2015/06/using-supervisor/)  
- [supervisor官方文档](http://supervisord.org/)

