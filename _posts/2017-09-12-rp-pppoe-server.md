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

### 实验

#### 环境

　　主机：win7 64位

　　VMWare运行Ubuntu16.04 32位

　　rp-pppoe 3.12

　　测试环境是公司固定IP

#### 配置

　　VMWare使用桥接模式，给Ubuntu手动设置IP地址，确认可以上网

　　主机删除本地连接中的IP地址，此时无法上网

#### 配置pppoe

　　修改/etc/ppp/options

```
ms-dns x.x.x.x
ms-dns x.x.x.x
-pap
+chap
```

　　这里必须用chap，windows不支持pap好像

　　在/etc/ppp/chap-secrets中添加用户

```
# Secrets for authentication using CHAP
# client        server  secret                  IP addresses

"haha"  *       "123"   *
```

　　这里两个星号都要写

　　编辑 /etc/ppp/pppoe-server-options 文件加入require-chap

　　启用IP转发功能

```
echo "1" > /proc/sys/net/ipv4/ip_forward
```

　　这里必须用root用户来执行

　　设置iptables的IP策略

```
iptables -A POSTROUTING -t nat -s 10.10.10.0/24 -j MASQUERADE
```

　　注：-s 参数后面的网络地址是一会儿将要开启的pppoe-server设置的网络地址，这个地址可以根据需要自己设定，只要iptables和pppoe-server匹配就好。

#### 启动pppoe

```
sudo pppoe-server -I wlan0 -L 10.10.10.1 -R 10.10.10.100 -N 100
```

　　注：

　　-I 参数用于指定监听哪个网络端口。可以使用ifconfig命令查看当前工作的端口名称。由于本人的笔记本电脑使用无线网络，所以是wlan0端口。

　　-L 参数用于指定在一个PPP连接中，PPPoE服务器的IP地址。随便设置，不是真实的ip地址。

　　-R 参数用于指定当有客户连接到服务器上时，从哪个IP地址开始分配给客户，也是随便设置。

　　-N 参数用于指定至多可以有多少个客户同时连接到本服务器上。

#### 连接pppoe

　　在windows下新建宽带pppoe连接，输入刚刚设置的账号密码，连接即可连上

#### 问题

　　不能使用google了，会提示不是私密连接