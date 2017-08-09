---
layout: post
title: domain socket与generic netlink的一点总结
categories: C
description: 做完了ECHO作业,趁还有点印象总结一波
keywords: C语言
---

　　上上周花了一个周末的时间赶了一个echo作业,上周一总算提交了,至于echo是什么你们就不用知道了,这里趁还有一点印象,总结一下domain socket还有generic netlink的一些要注意的点吧

## domain socket

#### SIGPIPE信号导致客户端崩溃

　　在连接建立以后,如果服务端意外断开,比如被`Ctrl C`了,这时候客户端再进行send就会引起崩溃,搜索之后是因为SIGPIPE信号的缘故,引起了崩溃,解决方法也很简单,将信号屏蔽了就好

```
signal(SIGPIPE, SIG_IGN);
```

#### 带超时的接收与发送

　　在发送与接收的时候会不希望其完全阻塞,以便可以在任意时候退出,这时可以设置发送与接收的超时时间,接收的设置函数如下

```
struct timeval timeout;
timeout.tv_sec = 1;
timeout.tv_usec = 0;

setsockopt(fd, SOL_SOCKET, SO_RCVTIMEO, (char *)&timeout, sizeof(struct timeval));
```

　　这里面的SO_RCVTIMEO就是指接收,当然发送也可以设置,有了这个思路以后自行google就好了


## generic netlink

　　这货说来其实挺恶心的,接口一直在变化,感觉换一个内核版本接口就变一次

　　网上找的那些例程基本用不了啦,除非你的内核版本和他们一模一样,但是我们还是要跟上时代嘛,装个新版本也好看不是:grinning:

　　所以怎么办呢

　　直接去找内核,我搜索时候找到哪种可以搜索各个版本内核里源码的网站,在里面搜索netlink相关的函数,一些基本的名字是不会变化啦,**family**什么的

　　找到以后基本随便找个短点的代码看看,参照一下网上的那些例程,基本内核部分的就能搞了

　　至于用户空间,目前来看好像接口变化倒是不太大,参照一下例程,内核里也能搜索到,我找到过的:sunglasses::sunglasses:

　　到这里基本就算可以通信啦

#### 客户端必须以root权限运行

　　RT,客户端必须以root权限运行,没有权限的话可以获取family id,但是就是无法通信,所以切记啊,我折腾了好久,真是醉了:sob::sob::sob:

#### 服务端的doit是单线程的

　　客户端发送消息后,服务端会在注册的operation里的doit函数中响应,这个doit其实是单线程的,也就是说,如果收到消息后,你想要对消息处理后再回传,而这个处理的时间又比较长,那么就会把这个线程卡住,客户端再想要发送消息是无法成功哒,会被阻塞住

　　所以如果有大量的内容要处理的话,还是要开辟线程,丢到线程中去处理然后回复

## others

#### KERNEL宏定义

　　其他一些小技巧吧算是,如果有的文件想要同时在用户空间和内核空间编译,可以使用**__KERNEL__**宏定义

```
#ifdef __KERNEL__

#else

#endif
```

#### 检测运行权限

　　上面说到netlink客户端需要root权限才能通信,可以通过判断uid来确认

```
if (geteuid() != 0) {
    printf("This program must run as root\n");
    return -1;
}
```

#### 单例程序

　　有时希望程序只能运行一份,可以通过文件锁的形式来保护

```
#define LOCKFILE "lock.pid"
#define LOCKMODE 0666

int fd;
/* 单一实例检测 */
fd = open(LOCKFILE, O_RDWR | O_CREAT, LOCKMODE);
if (fd < 0) {
    printf("Failed to open %s", LOCKFILE);
    return -1;
}
if (flock(fd, LOCK_EX | LOCK_NB) < 0) {
    printf("Server is running");
    close(fd);
    return -1;
}
```
