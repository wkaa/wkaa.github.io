---
layout: post
title: P4入门（二）--flowlet switching（simple router）
categories: P4
description: P4教程第二弹！这次自信一点了！
keywords: P4 SDN Switch Route
---

　　P4教程第二弹！感觉入门些了，通过看文档还有查资料，大部分内容能够理解了，这里就介绍一下tutorials的第二个练习，Let's start！

　　首先，flowlet switching是干什么的，大致就是做一个ecmp流控的，ecmp是什么？google一下，介绍和例程戳这里[tutorials](https://github.com/p4lang/tutorials/tree/master/SIGCOMM_2015#obtaining-required-software)

## 源码

　　源码太多了，不好一口气贴，我讲到哪里贴到哪里吧，源码连接在[这里](https://github.com/p4lang/tutorials/tree/master/SIGCOMM_2015/flowlet_switching)

## 分析

　　这次需要结合p4源码与commands.txt一起来看，commands.txt定义了表与初始表项，需要在查表时用到，接下来从头部定义开始。

### 头部定义

　　headers.p4中定义了用到的头部，然后在主文件中`#include`，其中定义了以太网头部，ipv4头部，tcp头部。

　　intrinsic.p4中定义了一个元数据类型的头部，这种按我理解就算变量了吧。

　　其他其他还有一些定义就先不一一说了。

### 解析

　　依旧从start开始，一开始就先跳到以太网帧头部解析，提取了头部，判定是ipv4后跳到ip解析，在ip解析这里提取了ip头部后，判断协议类型，判定是tcp后跳到tcp解析，这里只提取一下tcp头部，然后直接转到ingress。

### 表匹配

　　在ingress中，一共有4个或5个表要匹配，一个一个来吧，先介绍一下这里定义了一个元数据，作用是储存一些临时参数

```
header_type ingress_metadata_t {
    fields {
        flow_ipg : 48; // inter-packet gap
        flowlet_map_index : FLOWLET_MAP_BITS; // flowlet map index
        flowlet_id : 16; // flowlet id
        flowlet_lasttime : 48; // flowlet's last reference time

        ecmp_offset : 14; // offset into the ecmp table

        nhop_ipv4 : 32;
    }
}

metadata ingress_metadata_t ingress_metadata;
```

　　具体每个字段的用处，后面一点点涉及，还有两个寄存器变量，这种变量能够全局一直保存

```
register flowlet_lasttime {
    width : 48;
    instance_count : 8192;
}

register flowlet_id {
    width : 16;
    instance_count : 8192;
}
```

　　width代表位宽，instance_count代表申请多少个单元的空间，这个空间大小8192就对应了map_size的大小，size是这个表的大小，可以储存多少表项。

#### flowlet表

　　这个表可谓是很重要了，它计算了hash值作为ecmp的基础，它没有表项，只有一个默认动作

```
action lookup_flowlet_map() {
    modify_field_with_hash_based_offset(ingress_metadata.flowlet_map_index, 0,
                                        flowlet_map_hash, FLOWLET_MAP_SIZE);

    register_read(ingress_metadata.flowlet_id,
                  flowlet_id, ingress_metadata.flowlet_map_index);

    modify_field(ingress_metadata.flow_ipg,
                 intrinsic_metadata.ingress_global_timestamp);
    register_read(ingress_metadata.flowlet_lasttime,
    flowlet_lasttime, ingress_metadata.flowlet_map_index);
    subtract_from_field(ingress_metadata.flow_ipg,
                        ingress_metadata.flowlet_lasttime);

    register_write(flowlet_lasttime, ingress_metadata.flowlet_map_index,
                   intrinsic_metadata.ingress_global_timestamp);
}

table flowlet {
    actions { lookup_flowlet_map; }
}
```

　　这里把表的内容和动作的action一起贴出来了，一点点解释一下

　　第一个`modify_field_with_hash_based_offset`函数，这个的操作是计算hash值填入第一个参数中，hash值得范围在第二个参数到最后一个参数之间，第三个参数是参与hash计算的表项。参与计算的表项就是tcp的五元组（源IP地址，源端口，目的IP地址，目的端口，和传输层协议），计算的算法为crc16。

```
field_list l3_hash_fields {
    ipv4.srcAddr;
    ipv4.dstAddr;
    ipv4.protocol;
    tcp.srcPort;
    tcp.dstPort;
}

field_list_calculation flowlet_map_hash {
    input {
        l3_hash_fields;
    }
    algorithm : crc16;
    output_width : FLOWLET_MAP_BITS;
}
```

　　将计算好的hash值存入了`ingress_metadata.flowlet_map_index`中，下一步读取寄存器（先这么称呼吧），将flowlet_id中偏移为刚刚算出的flowlet_map_index的位置的值提取出来存入ingress_metadata.flowlet_id中

　　接下来修改ingress_metadata.flow_ipg为当前的全局时间戳intrinsic_metadata.ingress_global_timestamp

　　再读取寄存器变量，将flowlet_lasttime中偏移为刚刚flowlet_map_index的位置的值提取出来存入ingress_metadata.flowlet_lasttime中

　　再将ingress_metadata.flow_ipg的值减去ingress_metadata.flowlet_lasttime。

　　最后写寄存器，将全局时间戳写入flowlet_lasttime的flowlet_map_index偏移位置。

　　***这里面到底是做了什么呢，就是计算了一下hash值，然后提取同hash值的id值，还有与上次同hash值报文的时间差。***

#### new_flowlet表

　　这个表的进入是有条件的，上一步算出了与上一个同hash值报文的时间差，那么这里判断若时间差大于一个设定值，才会进入这个表，这个表的目的就是计算一个新的flowlet_id值，实现ecmp。

　　这个表也是只有一个默认action，操作就是将id值加1，然后同时存入寄存器中。

　　以上两个表都是在commands.txt中指定了魔表操作

```
table_set_default flowlet lookup_flowlet_map
table_set_default new_flowlet update_flowlet_id
```

#### ecmp_group表

　　这个表时有表项匹配的，如下：

```
table ecmp_group {
    reads {
        ipv4.dstAddr : lpm;
    }
    actions {
        _drop;
        set_ecmp_select;
    }
    size : ECMP_GROUP_TABLE_SIZE;
}
```

　　匹配目的IP地址，使用lpm最长匹配，匹配表项的来源就是在commands.txt中设置的表项

```
table_set_default ecmp_group _drop
table_add ecmp_group set_ecmp_select 10.0.0.1/32 => 0 2
```

　　匹配成功后，得到参数0和2，进入action

```
action set_ecmp_select(ecmp_base, ecmp_count) {
    modify_field_with_hash_based_offset(ingress_metadata.ecmp_offset, ecmp_base,
                                        flowlet_ecmp_hash, ecmp_count);
}
```

　　action中还是计算hash值，将其存入ingress_metadata.ecmp_offset，这次的结果范围就是刚刚表项命中的结果，0到2，结果为0或者1（resul % 2 + 0），参与计算的除了五元组外增加了刚刚得到的id值。

#### ecmp_nhop表

　　这个表匹配上一步得到hash结果，0或者1，是exact精确匹配，表项同样定义在commands.txt中

```
table_set_default ecmp_nhop _drop
table_add ecmp_nhop set_nhop 0 => 10.0.1.1 1
table_add ecmp_nhop set_nhop 1 => 10.0.2.1 2
```

　　表项中对0或者1的下一跳IP和port做了定义，以此为参数执行set_nhop这个action，action中设置了ingress_metadata.nhop_ipv4的值，设置了报文出口端口，将ttl值减1.

#### forward表

　　这个表匹配上一步得到的下一跳IP，也是精确匹配

```
table forward {
    reads {
        ingress_metadata.nhop_ipv4 : exact;
    }
    actions {
        set_dmac;
        _drop;
    }
    size: 512;
}

action set_dmac(dmac) {
    modify_field(ethernet.dstAddr, dmac);
}

//commands.txt表项
table_set_default forward _drop
table_add forward set_dmac 10.0.1.1 => 00:04:00:00:00:00
table_add forward set_dmac 10.0.2.1 => 00:04:00:00:00:01
```

　　根据获得的下一跳IP，匹配中表项后得到MAC地址，其实也算是一个ARP表了，在action中设置了之前解析得到的报文头部的目的MAC地址。

### 出口

　　在出口处也做了一个表匹配，匹配send_frame表，在表中精确匹配之前设置的出口端口，表项设置如下

```
table_set_default send_frame _drop
table_add send_frame rewrite_mac 1 => 00:aa:bb:00:00:00
table_add send_frame rewrite_mac 2 => 00:aa:bb:00:00:01
```

　　查表后得到源MAC地址，在action中修改报文头部的目的MAC地址，然后输出。

## 补充

　　在IP头部中，定义了`calculated_field ipv4.hdrChecksum`，按我理解应该是自动计算IP头部校验和的，这个留着下回分析。。