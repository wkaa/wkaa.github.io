---
layout: post
title: PPPoE学习（三）---PPP会话建立流程
categories: PPPoE PPP
description: 介绍PPP会话建立流程
keywords: PPPoE PPP
---

　　介绍一下PPP会话建立流程，来源于RFC1661

## PPP会话建立流程

　　简单概括一下PPP会话的建立流程，主要分为了三个阶段

```
1. LCP协商阶段，Host与Server互相协商MRU、认证方式等信息，最终以双方都发送ACK为结束
2. 认证阶段，根据LCP阶段协商的认证方式进行认证，方式有PAP或CHAP，认证用户名密码
3. NCP协商阶段，一般为IPCP（IP协议），Host与Server互相协商IP地址、DNS等信息，最终以双方都发送ACK为结束
```

## LCP协商阶段

　　LCP类型为0xc021，分组包含如下信息：

```
编码：长1字节，指出了LCP分组的类型。
标识符：长1字节，用于匹配请求和应答。
长度：长2字节，指出了LCP分组的总长（包括所有字段）。
数据：长度由"长度"字段指出，可能为0或多个字节。"编码"字段决定了该字段的格式。数据中包含若干个TLV
```

　　LCP阶段双方互相发送request，LCP协商阶段完成最大传输单元（MTU），是否进行认证和采用何种认证方式（Authentication Type）的协商。

　　LCP协商的过程如下：协商双方互相发送一个LCP Config-Request报文，确认收到的Config-Request报文中的协商选项，根据这些选项的支持与接受情况，做出适当的回应。若两端都回应了Config-ACK，则标志LCP链路建立成功，不接受则发送reject或者NAK，分别代表拒绝和不接受，拒绝指不提供该功能，需带上要拒绝的选项，不接受指支持该功能但参数需要调整，并附带上期望的参数，如收到MTU的request大小为1500，期望的MTU为1480，则要发送MTU为1480的NAK。任一方收到NAK后，根据报文调整参数后再次发送Request报文，直到对端回应了ACK报文为止。

　　在协商的过程中一般会带上Magic Number，即魔术数（随机数），目的是为了检测环回链路，若存在环回链路，就可能收到两份含有相同魔术数的报文。

　　一份简单的LCP报文交互如下：

　　Server端发送Request，请求认证方式为PAP

```
Frame 5: 72 bytes on wire (576 bits), 72 bytes captured (576 bits)
Ethernet II, Src: Hangzhou_e3:7b:b5 (74:25:8a:e3:7b:b5), Dst: Performa_00:00:02 (00:10:94:00:00:02)
    Destination: Performa_00:00:02 (00:10:94:00:00:02)
    Source: Hangzhou_e3:7b:b5 (74:25:8a:e3:7b:b5)
    Type: PPPoE Session (0x8864)
PPP-over-Ethernet Session
    0001 .... = Version: 1
    .... 0001 = Type: 1
    Code: Session Data (0x00)
    Session ID: 0x0001
    Payload Length: 16
Point-to-Point Protocol
    Protocol: Link Control Protocol (0xc021)
PPP Link Control Protocol
    Code: Configuration Request (1)
    Identifier: 0 (0x00)
    Length: 14
    Options: (10 bytes), Authentication Protocol, Magic Number
        Authentication Protocol: Password Authentication Protocol (0xc023)
            Type: Authentication Protocol (3)
            Length: 4
            Authentication Protocol: Password Authentication Protocol (0xc023)
        Magic Number: 0x9baf0940
            Type: Magic Number (5)
            Length: 6
            Magic Number: 0x9baf0940
```

　　Host发送Request，请求MRU为1492

```
Frame 6: 64 bytes on wire (512 bits), 64 bytes captured (512 bits)
Ethernet II, Src: Performa_00:00:02 (00:10:94:00:00:02), Dst: Hangzhou_e3:7b:b5 (74:25:8a:e3:7b:b5)
    Destination: Hangzhou_e3:7b:b5 (74:25:8a:e3:7b:b5)
    Source: Performa_00:00:02 (00:10:94:00:00:02)
    Type: PPPoE Session (0x8864)
PPP-over-Ethernet Session
    0001 .... = Version: 1
    .... 0001 = Type: 1
    Code: Session Data (0x00)
    Session ID: 0x0001
    Payload Length: 16
Point-to-Point Protocol
    Protocol: Link Control Protocol (0xc021)
PPP Link Control Protocol
    Code: Configuration Request (1)
    Identifier: 1 (0x01)
    Length: 14
    Options: (10 bytes), Maximum Receive Unit, Magic Number
        Maximum Receive Unit: 1492
            Type: Maximum Receive Unit (1)
            Length: 4
            Maximum Receive Unit: 1492
        Magic Number: 0xca1f708c
            Type: Magic Number (5)
            Length: 6
            Magic Number: 0xca1f708c
```

　　Server回复ACK

```
Frame 8: 72 bytes on wire (576 bits), 72 bytes captured (576 bits)
Ethernet II, Src: Hangzhou_e3:7b:b5 (74:25:8a:e3:7b:b5), Dst: Performa_00:00:02 (00:10:94:00:00:02)
    Destination: Performa_00:00:02 (00:10:94:00:00:02)
    Source: Hangzhou_e3:7b:b5 (74:25:8a:e3:7b:b5)
    Type: PPPoE Session (0x8864)
PPP-over-Ethernet Session
    0001 .... = Version: 1
    .... 0001 = Type: 1
    Code: Session Data (0x00)
    Session ID: 0x0001
    Payload Length: 16
Point-to-Point Protocol
    Protocol: Link Control Protocol (0xc021)
PPP Link Control Protocol
    Code: Configuration Ack (2)
    Identifier: 1 (0x01)
    Length: 14
    Options: (10 bytes), Maximum Receive Unit, Magic Number
        Maximum Receive Unit: 1492
            Type: Maximum Receive Unit (1)
            Length: 4
            Maximum Receive Unit: 1492
        Magic Number: 0xca1f708c
            Type: Magic Number (5)
            Length: 6
            Magic Number: 0xca1f708c
```

　　Host回复ACK

```
Frame 7: 64 bytes on wire (512 bits), 64 bytes captured (512 bits)
Ethernet II, Src: Performa_00:00:02 (00:10:94:00:00:02), Dst: Hangzhou_e3:7b:b5 (74:25:8a:e3:7b:b5)
    Destination: Hangzhou_e3:7b:b5 (74:25:8a:e3:7b:b5)
    Source: Performa_00:00:02 (00:10:94:00:00:02)
    Type: PPPoE Session (0x8864)
PPP-over-Ethernet Session
    0001 .... = Version: 1
    .... 0001 = Type: 1
    Code: Session Data (0x00)
    Session ID: 0x0001
    Payload Length: 16
Point-to-Point Protocol
    Protocol: Link Control Protocol (0xc021)
PPP Link Control Protocol
    Code: Configuration Ack (2)
    Identifier: 0 (0x00)
    Length: 14
    Options: (10 bytes), Authentication Protocol, Magic Number
        Authentication Protocol: Password Authentication Protocol (0xc023)
            Type: Authentication Protocol (3)
            Length: 4
            Authentication Protocol: Password Authentication Protocol (0xc023)
        Magic Number: 0x9baf0940
            Type: Magic Number (5)
            Length: 6
            Magic Number: 0x9baf0940
```

## 认证阶段

　　认证方式有两种，PAP和CHAP，PAP将账号与密码明文传输，容易被截取泄漏，CHAP采用挑战的方式，通过携带随机数来保护密码

### PAP

　　PAP的PPP类型为0xc023，在LCP协商完成后由Host主动发起，将用户名与密码发送给Server，Server验证后回复是否通过

　　Host发出验证请求

```
Frame 9: 64 bytes on wire (512 bits), 64 bytes captured (512 bits)
Ethernet II, Src: Performa_00:00:02 (00:10:94:00:00:02), Dst: Hangzhou_e3:7b:b5 (74:25:8a:e3:7b:b5)
    Destination: Hangzhou_e3:7b:b5 (74:25:8a:e3:7b:b5)
    Source: Performa_00:00:02 (00:10:94:00:00:02)
    Type: PPPoE Session (0x8864)
PPP-over-Ethernet Session
    0001 .... = Version: 1
    .... 0001 = Type: 1
    Code: Session Data (0x00)
    Session ID: 0x0001
    Payload Length: 18
Point-to-Point Protocol
    Protocol: Password Authentication Protocol (0xc023)
PPP Password Authentication Protocol
    Code: Authenticate-Request (1)
    Identifier: 3
    Length: 16
    Data
        Peer-ID-Length: 4
        Peer-ID: test
        Password-Length: 6
        Password: 123456
```

　　Server对验证请求进行回复

```
Frame 10: 72 bytes on wire (576 bits), 72 bytes captured (576 bits)
Ethernet II, Src: Hangzhou_e3:7b:b5 (74:25:8a:e3:7b:b5), Dst: Performa_00:00:02 (00:10:94:00:00:02)
    Destination: Performa_00:00:02 (00:10:94:00:00:02)
    Source: Hangzhou_e3:7b:b5 (74:25:8a:e3:7b:b5)
    Type: PPPoE Session (0x8864)
PPP-over-Ethernet Session
    0001 .... = Version: 1
    .... 0001 = Type: 1
    Code: Session Data (0x00)
    Session ID: 0x0001
    Payload Length: 34 [incorrect, should be 52]
Point-to-Point Protocol
    Protocol: Password Authentication Protocol (0xc023)
PPP Password Authentication Protocol
    Code: Authenticate-Ack (2)
    Identifier: 3
    Length: 32
    Data
        Message-Length: 27
        Message: Welcome to use this device.
```

### CHAP

　　CHAP验证的PPP类型为0xx223，CHAP验证相比PAP更为安全，采用的是一种叫挑战（challenge）的机制，由Server主动发起，Server发出的请求携带了一个随机数，Host收到之后使用这个随机数与自己的密码一起进行加密运算（MD5+HASH），将这的运算后的值与自己的用户名一起回复给Server，Server根据用户名查询自己的数据库得到密码，将密码与自己刚刚发出的随机数一起进行一样的加密运算，将得到的结果与收到的结果进行比较，一致则为通过验证，发出验证通过的报文

　　Server发出的CHAP请求报文

```
Frame 151: 53 bytes on wire (424 bits), 53 bytes captured (424 bits) on interface 0
Ethernet II, Src: Vmware_7c:22:7c (00:0c:29:7c:22:7c), Dst: 50:9a:4c:4c:d5:a2 (50:9a:4c:4c:d5:a2)
    Destination: 50:9a:4c:4c:d5:a2 (50:9a:4c:4c:d5:a2)
    Source: Vmware_7c:22:7c (00:0c:29:7c:22:7c)
    Type: PPPoE Session (0x8864)
PPP-over-Ethernet Session
    0001 .... = Version: 1
    .... 0001 = Type: 1
    Code: Session Data (0x00)
    Session ID: 0x0002
    Payload Length: 33
Point-to-Point Protocol
    Protocol: Challenge Handshake Authentication Protocol (0xc223)
PPP Challenge Handshake Authentication Protocol
    Code: Challenge (1)
    Identifier: 198
    Length: 31
    Data
        Value Size: 20
        Value: 9da9d8a3184d57ec37151c5cd59c8fe77d6ec4b2
        Name: ubuntu
```

　　Host回复的CHAP验证

```
Frame 155: 47 bytes on wire (376 bits), 47 bytes captured (376 bits) on interface 0
Ethernet II, Src: 50:9a:4c:4c:d5:a2 (50:9a:4c:4c:d5:a2), Dst: Vmware_7c:22:7c (00:0c:29:7c:22:7c)
    Destination: Vmware_7c:22:7c (00:0c:29:7c:22:7c)
    Source: 50:9a:4c:4c:d5:a2 (50:9a:4c:4c:d5:a2)
    Type: PPPoE Session (0x8864)
PPP-over-Ethernet Session
    0001 .... = Version: 1
    .... 0001 = Type: 1
    Code: Session Data (0x00)
    Session ID: 0x0002
    Payload Length: 27
Point-to-Point Protocol
    Protocol: Challenge Handshake Authentication Protocol (0xc223)
PPP Challenge Handshake Authentication Protocol
    Code: Response (2)
    Identifier: 198
    Length: 25
    Data
        Value Size: 16
        Value: 8648dca1e0f417eb04e563808a5f8bff
        Name: haha
```

　　Server回复验证通过

```
Frame 156: 40 bytes on wire (320 bits), 40 bytes captured (320 bits) on interface 0
Ethernet II, Src: Vmware_7c:22:7c (00:0c:29:7c:22:7c), Dst: 50:9a:4c:4c:d5:a2 (50:9a:4c:4c:d5:a2)
    Destination: 50:9a:4c:4c:d5:a2 (50:9a:4c:4c:d5:a2)
    Source: Vmware_7c:22:7c (00:0c:29:7c:22:7c)
    Type: PPPoE Session (0x8864)
PPP-over-Ethernet Session
    0001 .... = Version: 1
    .... 0001 = Type: 1
    Code: Session Data (0x00)
    Session ID: 0x0002
    Payload Length: 20
Point-to-Point Protocol
    Protocol: Challenge Handshake Authentication Protocol (0xc223)
PPP Challenge Handshake Authentication Protocol
    Code: Success (3)
    Identifier: 198
    Length: 18
    Message: Access granted
```

## NCP协商阶段

　　NCP有很多种，如IPCP、BCP、IPv6CP，最为常用的是IPCP（Internet Protocol Control Protocol）协议。NCP的主要功能是协商PPP报文的网络层参数，如IP地址，DNS Server IP地址，WINS Server IP地址等。PPPoE用户主要通过IPCP来获取访问网络的IP地址或IP地址段。

　　NCP流程与LCP流程类似，用户与ME设备之间互相发送NCP Config-Request报文并且互相回应NCP Config-Ack报文后，标志NCP己协商完，用户上线成功，可以正常访问网络了。

　　IPCP的PPP类型为0x8021，以下为一段简单的IPCP协商过程

```
1 Hangzhou_e3:7b:b5   Performa_00:00:02   PPP IPCP    72  Configuration Request
2 Performa_00:00:02   Hangzhou_e3:7b:b5   PPP IPCP    64  Configuration Request
3 Performa_00:00:02   Hangzhou_e3:7b:b5   PPP IPCP    64  Configuration Ack
4 Hangzhou_e3:7b:b5   Performa_00:00:02   PPP IPCP    72  Configuration Nak
5 Performa_00:00:02   Hangzhou_e3:7b:b5   PPP IPCP    64  Configuration Request
6 Hangzhou_e3:7b:b5   Performa_00:00:02   PPP IPCP    72  Configuration Ack
```

　　以上的具体过程如下：

1. Server发送Request，内容为主机IP
2. Host发送Request，请求一个IP，IP为全0
3. Host确认第一条Request，得到主机IP
4. Server对第二条Request回复Nak，附上给Host分配的IP
5. Host收到分配的IP地址后，以这条IP地址发送Request
6. Server对上一条Request回复确认

　　至此就完成IPCP协商过程。

　　当然也有较为复杂的协商过程，这里的Host只请求IP，实际上还可能请求DNS，Server也会根据请求的内容作出判断，如Server没有设置DNS，若收到DNS请求则会对这个请求回复Reject，Host收到后会再次组织一次不包含DNS请求的Request，双方互相协商直到达成一致。

## PPP会话开始

　　当NCP协商完成后，就代表着PPP会话已成功建立，此时可以正常发送三层报文（IP等），如下为一个ICMP报文的例子，此时的PPP类型为0x0021。

```
Frame 19: 110 bytes on wire (880 bits), 110 bytes captured (880 bits)
Ethernet II, Src: Performa_00:00:02 (00:10:94:00:00:02), Dst: Hangzhou_e3:7b:b5 (74:25:8a:e3:7b:b5)
    Destination: Hangzhou_e3:7b:b5 (74:25:8a:e3:7b:b5)
    Source: Performa_00:00:02 (00:10:94:00:00:02)
    Type: PPPoE Session (0x8864)
PPP-over-Ethernet Session
    0001 .... = Version: 1
    .... 0001 = Type: 1
    Code: Session Data (0x00)
    Session ID: 0x0001
    Payload Length: 86
Point-to-Point Protocol
    Protocol: Internet Protocol version 4 (0x0021)
Internet Protocol Version 4, Src: 100.1.24.97, Dst: 192.168.1.121
    0100 .... = Version: 4
    .... 0101 = Header Length: 20 bytes
    Differentiated Services Field: 0x00 (DSCP: CS0, ECN: Not-ECT)
    Total Length: 84
    Identification: 0x8e08 (36360)
    Flags: 0x00
    Fragment offset: 0
    Time to live: 64
    Protocol: ICMP (1)
    Header checksum: 0xae1d [validation disabled]
    Source: 100.1.24.97
    Destination: 192.168.1.121
    [Source GeoIP: Unknown]
    [Destination GeoIP: Unknown]
Internet Control Message Protocol
```

## PPP会话维持

　　设备主动发送Echo Request进行PPPoE心跳保活，若3次未得到服务器的响应，则设备主动释放地址。发LCP Echo Request 的时候，魔术字字段要和之前通信的Configure_Request使用的魔术字字段保持一致。有些设备或终端不支持主动发送 Echo-Request 报文, 只能支持回应Echo-Reply报文。

　　Echo Request

```
Frame 39: 72 bytes on wire (576 bits), 72 bytes captured (576 bits)
Ethernet II, Src: Hangzhou_e3:7b:b5 (74:25:8a:e3:7b:b5), Dst: Performa_00:00:02 (00:10:94:00:00:02)
    Destination: Performa_00:00:02 (00:10:94:00:00:02)
    Source: Hangzhou_e3:7b:b5 (74:25:8a:e3:7b:b5)
    Type: PPPoE Session (0x8864)
PPP-over-Ethernet Session
    0001 .... = Version: 1
    .... 0001 = Type: 1
    Code: Session Data (0x00)
    Session ID: 0x0001
    Payload Length: 10
Point-to-Point Protocol
    Protocol: Link Control Protocol (0xc021)
PPP Link Control Protocol
    Code: Echo Request (9)
    Identifier: 4 (0x04)
    Length: 8
    Magic Number: 0x9baf0940
```

　　Echo Reply

```
Frame 40: 64 bytes on wire (512 bits), 64 bytes captured (512 bits)
Ethernet II, Src: Performa_00:00:02 (00:10:94:00:00:02), Dst: Hangzhou_e3:7b:b5 (74:25:8a:e3:7b:b5)
    Destination: Hangzhou_e3:7b:b5 (74:25:8a:e3:7b:b5)
    Source: Performa_00:00:02 (00:10:94:00:00:02)
    Type: PPPoE Session (0x8864)
PPP-over-Ethernet Session
    0001 .... = Version: 1
    .... 0001 = Type: 1
    Code: Session Data (0x00)
    Session ID: 0x0001
    Payload Length: 10
Point-to-Point Protocol
    Protocol: Link Control Protocol (0xc021)
PPP Link Control Protocol
    Code: Echo Reply (10)
    Identifier: 4 (0x04)
    Length: 8
    Magic Number: 0xca1f708c
```

## PPP会话终结

　　PPP会话可由任一方发起结束，发送LCP控制报文，类型为Termination Request，收到的一方回复Termination Ack后会话终结。

　　Host发送Request报文

```
Frame 41: 64 bytes on wire (512 bits), 64 bytes captured (512 bits)
Ethernet II, Src: Performa_00:00:02 (00:10:94:00:00:02), Dst: Hangzhou_e3:7b:b5 (74:25:8a:e3:7b:b5)
    Destination: Hangzhou_e3:7b:b5 (74:25:8a:e3:7b:b5)
    Source: Performa_00:00:02 (00:10:94:00:00:02)
    Type: PPPoE Session (0x8864)
PPP-over-Ethernet Session
    0001 .... = Version: 1
    .... 0001 = Type: 1
    Code: Session Data (0x00)
    Session ID: 0x0001
    Payload Length: 18
Point-to-Point Protocol
    Protocol: Link Control Protocol (0xc021)
PPP Link Control Protocol
    Code: Termination Request (5)
    Identifier: 2 (0x02)
    Length: 16
    Data: 557365722072657175657374
```

　　Server回复ACK

```
Frame 42: 72 bytes on wire (576 bits), 72 bytes captured (576 bits)
Ethernet II, Src: Hangzhou_e3:7b:b5 (74:25:8a:e3:7b:b5), Dst: Performa_00:00:02 (00:10:94:00:00:02)
    Destination: Performa_00:00:02 (00:10:94:00:00:02)
    Source: Hangzhou_e3:7b:b5 (74:25:8a:e3:7b:b5)
    Type: PPPoE Session (0x8864)
PPP-over-Ethernet Session
    0001 .... = Version: 1
    .... 0001 = Type: 1
    Code: Session Data (0x00)
    Session ID: 0x0001
    Payload Length: 6
Point-to-Point Protocol
    Protocol: Link Control Protocol (0xc021)
PPP Link Control Protocol
    Code: Termination Ack (6)
    Identifier: 2 (0x02)
    Length: 4
```