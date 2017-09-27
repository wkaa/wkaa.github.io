---
layout: post
title: P4入门（四）-- meter功能
categories: P4
description: 分析一下meter的功能
keywords: P4 
---

　　介绍一下P4中meter的功能，meter在P4语言中用于对报文进行限速，首先看看meter的定义

```
meter_declaration ::=
meter meter_name {
    type : meter_type ;
    [ result : field_ref ; ]
    [ direct_or_static ; ]
    [ instance_count : const_expr ; ]
}
meter_type ::= bytes | packets
```

　　meter_type就是限速模式，有两种，对字节限速和对报文限速，以下先将对报文限速，对字节限速的我还没试过

　　meter的模式有两种，direct和static，direct是对一个table进行限速，包括table的所有entry，需要指定result，输出到result。static模式可以有多个meter空间，用来对table中的每个entry进行限速，需要指定instance_count，即为meter空间大小，一般与table大小一致。

　　meter输出有三种，绿、黄、红，对应0、1、2，设置速率需要两个速率，first rate指的是的绿灯的速率，second rate指的是绿和黄总共的速率，后者必须大于前者。 

　　目前看来每个meter表只能设置一个速率，该meter的所有空间都是同样的速率。

　　如果将一组相同速率的报文共用一个meter的空间，meter限制的是该组所有报文的总速率，因此只能为每个entry分配独立的meter

　　另外对于direct模式和byte模式，后续有测试了再行补充
