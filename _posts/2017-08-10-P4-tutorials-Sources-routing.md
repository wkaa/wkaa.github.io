---
layout: post
title: P4入门（一）--Source Routing
categories: P4
description: 对P4一脸懵逼，一点点总结分析吧
keywords: P4 SDN Switch Route
---

　　第一次听说P4这东西，目前还是一脸懵逼状态，但是一点点记录下来可能会好理解一些。

## P4是什么

　　P4(Programming protocol-independent packet processors)是由Pat Bosshart等人提出来的高级“协议独立数据包处理编程语言”，如OpenFlow一样是一种南向协议，但是其范围要比OpenFlow要大。不仅可以指导数据流进行转发，还可以对交换机等转发设备的数据处理流程进行软件编程定义，是真正意义上的完全SDN。

　　按我目前理解就是针对于交换机等转发设备的一种编程语言实现转发流程吧

## Source Routing

　　鉴于我现在什么都还不懂，直接先走一个实验亲身体会一下吧，在P4的github上有一个[tutorials](https://github.com/p4lang/tutorials/tree/master/SIGCOMM_2015#exercise-1-source-routing。)里面有Source Routing的详细介绍，设计了一个三台交换机与两台host的拓扑

　　介绍暂时先算啦，怎么配置运行工程也先不说了。另外，本文中文档基于P4 14，文档戳这里[P4-14-spec](https://p4lang.github.io/p4-spec/)

### 源码

　　先贴上Source Routing的示例代码如下

```
header_type easyroute_head_t {
    fields {
        preamble: 64;
        num_valid: 32;
    }
}

header easyroute_head_t easyroute_head;

header_type easyroute_port_t {
    fields {
        port: 8;
    }
}

header easyroute_port_t easyroute_port;

parser start {
    return select(current(0, 64)) {
        0: parse_head;
        default: ingress;
    }
}

parser parse_head {
    extract(easyroute_head);
    return select(latest.num_valid) {
        0: ingress;
        default: parse_port;
    }
}

parser parse_port {
    extract(easyroute_port);
    return ingress;
}

action _drop() {
    drop();
}

action route() {
    modify_field(standard_metadata.egress_spec, easyroute_port.port);
    add_to_field(easyroute_head.num_valid, -1);
    remove_header(easyroute_port);
}

table route_pkt {
    reads {
        easyroute_port: valid;
    }
    actions {
        _drop;
        route;
    }
    size: 1;
}

control ingress {
    apply(route_pkt);
}

control egress {
    // leave empty
}
```

### 分析

　　稍微解释一下，header的头部申明比较好理解，就类似c语言，fields里面的数字就类似struct的位段，表示这个变量占几位，这里面申明了两个头部`easyroute_head`和`easyroute_port` ， `easyroute_head`包括了前导码和计数值，`easyroute_port`就是下一跳的port口

#### 报文头部有效性判断

　　报文的解析从`parser start`开始，这里取了前64位判断是否为0，若为0就进行头部解析，跳转到`parser parse_head`，在这里面提取了其头部，然后判断其中的计数值字段，若为0则转到ingress（丢弃），否则跳转到port的解析`parser parse_port`，在这里面提取了port，然后转到ingress

#### 表匹配

　　在ingress中，进行表匹配，源码中定义了一个表`table route_pkt`，表中匹配`easyroute_port`的值，判定为valid，至于这个valid什么意思呢，文档介绍如下：

> Match types have the following meanings.
> 
> • exact: The field value is matched against the table entry and the values must be identical for the entry to be considered.
> 
> • ternary: A mask provided with each entry in the table. This mask is ANDed with the field value before a comparison is made. The field value and the table entry value need only agree on the bits set in the entry’s mask. Because of the possibilities of overlapping matches, a priority must be associated with each entry in a table using ternary matches.
> 
> • lpm: This is a special case of a ternary match. Each entry’s mask selects a prefix by having a divide between 1s in the high order bits and 0s in the low order bits. The 5611 TABLE DECLARATIONS number of 1 bits gives the length of the prefix which is used as the priority of the entry.
> 
> • range: Each entry specifies a low and high value for the entry and the field matches only if it is in this range. Range end points are inclusive. Signedness of the field is used in evaluating the order.
> 
> • valid: Only applicable to packet header fields or header instances (not metadata fields), the table entry must specify a value of true (the field is valid) or false (the field is not valid) as match criteria.

　　虽然摘取了文档，对于这个valid我也知道是有效的意思，但是这个有效从何而来，这个在文档的后面还有补充解释，

> The value of 1 will match when the header is valid; 0 will match when the header is not valid. Note that metadata fields are always valid.

　　所以其实这里判断的就是端口号大于0，在后面的脚本解析中说过了，启动交换机时，自动为每个端口按1、2、3顺序分配了端口号。

　　那么现在表项匹配成功，action操作是`_drop`和`route`，看commands.txt中的定义

```
table_set_default route_pkt route
table_add route_pkt _drop 0 =>
```

　　默认操作是route操作，如果是0则执行drop操作

#### route操作

　　来看`route`，里面进行了三项操作，这个我知道，是并行的，第一个修改了报文的出口，改为了之前提取到的port，那么这个`standard_metadata.egress_spec`是什么呢，有幸我又找到了文档，如下 

| Field | Notes |
| :----- | :----- | :----- |
| ingress_port | The port on which the packet arrived. Set prior to parsing. Always defined. Read only. |
| packet_length | The number of bytes in the packet. For Ethernet, does not include the CRC. Set prior to parsing. Cannot be used for matching or referenced in actions if the switch is in "cut-through" mode. Read only. |
| egress_spec | Specification of an egress. Undefined until set by match+action during ingress processing. This is the “intended” egress as opposed to the committed physical port(s) (see egress_port below). May be a physical port, a logical interface (such as a tunnel, a LAG, a route, or a VLAN flood group) or a multicast group. |
| egress_port | The physical port to which this packet instance is committed. Read only. This value is determined by the Buffering Mechanism and so is valid only for egress match+action stages. See Section 13 below. Read only. |
| egress_instance | An opaque identifier differentiating instances of a replicated packet. Read only. Like egress_port, this value is determined by the Buffering Mechanism and is valid only for egress match+action stages. See Section 13 below. |
| instance_type | Represents the type of instance of the packet: <br> • normal <br> • ingress clone <br> • egress clone <br> • recirculated <br> The representation of this data is target specific. |
| parser_status | Result of the parser. 0 means no error. Otherwise, the value indicates what error occurred during parsing. The representation of this data is target specific. |
| parser_error_location | If a parser error occurred, this is an indication of the location in the parser program where the error occurred. The representation of this data is target specific. |

　　这个egress_spec就是设置了出口，那么又有个疑问，这个元数据`standard_metadata`是什么鬼？

　　好在文档也有说明

> Metadata is state associated with each packet. It can be treated like a set of variables associated with each packet, read and written by actions executed by tables. 

　　总之就是对于每个报文自动生成的描述其属性的东西了。

　　回到刚刚的route，第二行就是修改计数值，将其减1，比较好理解，第三行，把这个port口移除，为什么这么做回去看tutorial里面有讲实验细节。

　　至此，完成了报文的所有处理，也设定了出口，然后就自动送到出口empress了，文档这么描述的：

> When the packet is dequeued, it is processed in the Egress Pipeline by the control function egress.

#### drop操作

　　刚刚那个`drop`操作，按照文档对于`drop`的解释：

> Indicate that the packet should not be transmitted. This primitive is intended for the egress pipeline where it is the only way to indicate that the packet should not be transmitted. On the ingress pipeline, this primitive is equivalent to setting the egress_spec metadata to a drop value (specific to the target).  
> 
> If executed on the ingress pipeline, the packet will continue through the end of the pipeline. A subsequent table may change the value of egress_spec which will override the drop action. The action cannot be overridden in the egress pipeline.  

　　在入口处执行的话会输出到管道末端，如果后续修改了egress_spec就会覆盖这个操作，也当然此处不存在这个条件，但是有个问题，valid是判断非0吗，那为何会有0这个可能呢，还要在看看

　　另外表里有个size，是接受报文数量大小限制的意思，文档中描述如下：

> • The min_size attribute indicates the minimum number of entries required for the table. If this cannot be supported, an error will be signaled when the declaration is processed.
> 
> • The max_size attribute is an indication that the table is not expected to grow larger than this size. If, at run time, the table has this many entries and another insert operation applied, it may be rejected.
> 
> • The size attribute is equivalent to specifying min_size and max_size with the same value.
> 
> • Although size and min_size are optional, failing to specify at least one of them may result in the table being eliminated as the compiler attempts to satisfy the other requirements of the program

　　这里姑且先认为是限制同时可处理的只有一条报文吧。

## 遗留问题汇总

　　1、 P4是怎么与拓扑进行连接的，怎么知道自己是怎样的拓扑，怎么知道交换机的端口号等信息

　　在后面那篇脚本解析中已经说明了，在启动时创建了拓扑，端口号是自动按照1、2、3顺序分配，固定了启动时拓扑中连接关系读取顺序也就唯一确定了端口号，p4源码将会被解释为json文件，写入交换机中，用来定义交换机。

　　2、 valid是如何判定的，来源是哪里，这个也跟上一个问题有关系

　　这个问题已经在原文中修改回答了，只要端口号大于0就是valid。这个还有疑问，再看看

　　3、 本文还没分析启动脚本和send、receive两个Python脚本，在启动脚本中建立了拓扑关系，需要分析一下

　　在后面已经分析了，顺便附上连接[Source Routing脚本分析](https://wkaa.github.io/2017/08/13/P4-tutorials-Sources-routing-python-ans/)

