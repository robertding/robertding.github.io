---
layout: news_item
title: bottle 框架 reload 的机制与实现
categories: work
---

### reload 机制干什么用的
如果没有 reload 机制,那么保存完代码的时候需要:

1. `kill`掉还在跑测试的应用
2. 重新写参数启动应用

reload 就是在保存代码的时候干了上面两件事.

### reload 原理
reload 机制由2个进程实现:`应用进程`和`守护进程`. `守护进程`用于监视`应用进程`的状态. 
当`应用进程`由于文件改动而退出的时候重启`应用进程`. 

监控文件变动由`应用进程`中的一个`监控线程`完成. 
`监控线程`读取每一个应用需要加载的模块并记录模块文件的修改时间,并每隔一段时间重新检查
一次. 若发现文件最后修改时间不一致则通知主进程退出. 然后由`守护进程`重启`应用进程`. 

### bottle 框架中 reload 的实现 

在启动应用的时候默认的`reloader = False`, 所以指定启动参数:

` run(reloader=True) `

在应用启动的时候`守护进程` fork 出`应用进程`并且在`应用进程`中设置环境变量`BOTTLE_CHILD`
来区分`守护进程`与`应用进程`.

`守护进程`创建一个临时文件,并不断改变文件的最后修改时间来表明自己还在运行. 
同时还每隔一段时间检查`应用进程`的状态,若`应用进程`由于文件改动而退出则重新启动
`应用进程`, 否则直接退出整个应用.

{% highlight python linenos %}
if reloader and not os.environ.get('BOTTLE_CHILD'):
    import subprocess
    lockfile = None
    try:
        fd, lockfile = tempfile.mkstemp(prefix='bottle.', suffix='.lock')
        os.close(fd)  # We only need this file to exist. We never write to it
        while os.path.exists(lockfile):
            args = [sys.executable] + sys.argv
            environ = os.environ.copy()
            environ['BOTTLE_CHILD'] = 'true'
            environ['BOTTLE_LOCKFILE'] = lockfile
            p = subprocess.Popen(args, env=environ)
            while p.poll() is None:  # Busy wait...
                os.utime(lockfile, None)  # I am alive!
                time.sleep(interval)
            if p.poll() != 3:
                if os.path.exists(lockfile): os.unlink(lockfile)
                sys.exit(p.poll())
    except KeyboardInterrupt:
        pass
{% endhighlight %}
