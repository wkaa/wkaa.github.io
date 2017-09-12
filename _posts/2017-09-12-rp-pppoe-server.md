---
layout: post
title: PPPoE学习（二）---rp-pppoe分析
categories: C PPPoE
description: 分析rp-pppoe
keywords: C语言 PPPoE
---

　　对Linux下开源PPPoE服务器rp-pppoe进行分析，主要针对pppoe-server

　　上一篇已经分析了PPPoE会话的建立流程，pppoe-server就是实现了这个流程，然后调用pppd开启会话

### 初始化

　　具体的rp-pppoe实现暂时没有细看，大致找到了一些关键的入口，在pppoe-server.c的main函数中，对每个interface注册了事件处理函数InterfaceHandler

```
for (i = 0; i < NumInterfaces; i++) {
    interfaces[i].eh = Event_AddHandler(event_selector,
                                        interfaces[i].sock,
                                        EVENT_FLAG_READABLE,
                                        InterfaceHandler,
                                        &interfaces[i]);
#ifdef HAVE_L2TP
    interfaces[i].session_sock = -1;
#endif
    if (!interfaces[i].eh) {
        rp_fatal("Event_AddHandler failed");
    }
}
```

　　在InterfaceHandler中对于各种类型的PPPoE报文进行处理，调用不同的处理函数

```
switch (packet.code) {
case CODE_PADI:
    processPADI(i, &packet, len);
    break;
case CODE_PADR:
    processPADR(i, &packet, len);
    break;
case CODE_PADT:
    /* Kill the child */
    processPADT(i, &packet, len);
    break;
case CODE_SESS:
    /* Ignore SESS -- children will handle them */
    break;
case CODE_PADO:
case CODE_PADS:
    /* Ignore PADO and PADS totally */
    break;
default:
    /* Syslog an error */
    break;
}
```

### processPADI

　　在processPADI中得到报文中的ServiceName，与本地的ServiceNames进行比对，判断能否提供对方所需要的服务

　　使用对方地址与本机地址还有一个随机数生成一个cookie

　　组装PADO报文，包含以上信息与其他relayId、hostUniq等信息。

### processPADR

　　在processPADR中对比在PADO中发出的cookie是否一致，再次对比ServiceName，以上都检验正确后，创建session，fork出子进程，在子进程中发送PADS，启动pppd

### pppd

　　在processPADR中启动pppd，使用pty参数将链路指定为pppoe，pppoe的实现在pppoe.c文件中，主要做两件事，将ppp报文加上pppoe头部发送到网卡，将网卡报文去掉pppoe头部发给ppp

