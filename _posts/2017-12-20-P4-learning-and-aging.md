---
layout: post
title: P4入门（五）-- Learning and Aging
categories: P4
description: 简单介绍一下表项的学习和老化
keywords: P4 
---

　　久违的P4系列学习教程，前几天看了一下学习与老化相关的内容，这里分享一下，本文基于P4_14的spec

　　看P4的spec中没有明确说明学习和老化到底是怎么做的，这里可以结合github上的Switch项目来看

　　具体switch项目如何工作这些的这里就不（lan）细（de）说了，以后有机会可以分享一下，switch在bmv2中模拟了传统交换机的功能，其中有一个就是mac地址的学习和老化，就从这个点来看P4是如何做到学习和老化的。

## Learning

### P4实现

　　在`l2.p4`里是关于二层的功能，其中实现了mac地址学习，源码如下：

```
field_list mac_learn_digest {
    ingress_metadata.bd;
    l2_metadata.lkp_mac_sa;
    ingress_metadata.ifindex;
}

action generate_learn_notify() {
    generate_digest(MAC_LEARN_RECEIVER, mac_learn_digest);
}

table learn_notify {
    reads {
        l2_metadata.l2_src_miss : ternary;
        l2_metadata.l2_src_move : ternary;
        l2_metadata.stp_state : ternary;
    }
    actions {
        nop;
        generate_learn_notify;
    }
    size : LEARN_NOTIFY_TABLE_SIZE;
}
#endif /* L2_DISABLE */

control process_mac_learning {
#ifndef L2_DISABLE
    if (l2_metadata.learning_enabled == TRUE) {
        apply(learn_notify);
    }
#endif /* L2_DISABLE */
}
```

　　从这个control看起，首先判断元数据是否开启了学习功能，然后进入学习（通告）表的查找，匹配的条件自然是通告的相关信息，源mac地址没匹配到（新地址）、源mac地址移动了、stp状态。匹配之后执行的action就是通告了

　　最上面的field_list定义了需要通告的信息，这里面包含了二层mac的相关信息，vlan、mac、ifx。然后在通告action中执行了generate_digest，这是P4的标准函数，spec有定义，用于学习功能，它会过滤重复的流，同样的流只会通告一份给CPU，减少了带宽消耗还有提高效率。

### PD_API

　　完成了P4编码后，与用户相关的就是其自动生成的PD_API了，可以在`switch\p4-build\bmv2\switch\pd`路径下找到，编写了上面的学习代码后，生成了相应的API，具体如下：

```
p4_pd_status_t p4_pd_dc_set_learning_timeout
(
 p4_pd_sess_hdl_t sess_hdl,
 uint8_t device_id,
 uint64_t usecs
);

typedef struct p4_pd_dc_mac_learn_digest_digest_entry {
  uint16_t ingress_metadata_bd;
  uint8_t l2_metadata_lkp_mac_sa[6];
  uint16_t ingress_metadata_ifindex;
} p4_pd_dc_mac_learn_digest_digest_entry_t;

typedef struct p4_pd_dc_mac_learn_digest_digest_msg {
  p4_pd_dev_target_t dev_tgt;
  uint16_t num_entries;
  p4_pd_dc_mac_learn_digest_digest_entry_t *entries;
  uint64_t buffer_id; // added by me, needed for BMv2 compatibility
} p4_pd_dc_mac_learn_digest_digest_msg_t;

typedef p4_pd_status_t
(*p4_pd_dc_mac_learn_digest_digest_notify_cb)(p4_pd_sess_hdl_t,
                                            p4_pd_dc_mac_learn_digest_digest_msg_t *,
                                            void *cookie);

p4_pd_status_t p4_pd_dc_mac_learn_digest_register
(
 p4_pd_sess_hdl_t sess_hdl,
 uint8_t device_id,
 p4_pd_dc_mac_learn_digest_digest_notify_cb cb_fn,
 void *cb_fn_cookie
);

p4_pd_status_t p4_pd_dc_mac_learn_digest_deregister
(
 p4_pd_sess_hdl_t sess_hdl,
 uint8_t device_id
);

p4_pd_status_t p4_pd_dc_mac_learn_digest_notify_ack
(
 p4_pd_sess_hdl_t sess_hdl,
 p4_pd_dc_mac_learn_digest_digest_msg_t *msg
);
```

　　生成的API中针对P4中定义的digest生成了相应的结构体，用来传递信息。同时还可以设置timeout时间，通告的时机是攒到一定数量后通告，或者一定时间后不管攒满了没都通过一次，这个timeout时间就是这个时间，时间的具体单位还没验证过。

　　使用上就是用register函数来注册一个回调函数，回调函数的参数中就传入了digest的信息，具体有点c语言基础一个就能看懂了，当然，如果对一些参数如何赋值、构造不明确的，可以去switchapi的源码里面找找，搜索这个函数就可以了。

　　另外要说明的是，收到通告后要调用ack函数进行确认，不确认理论上会导致通告阻塞而无法再接收到通告吧。

## Aging

　　老化的实现是通过给表加上timeout属性来实现的，表默认是关闭了timeout功能，开启之后，可以根据每个表项设置TTL（time to live），TTL的初值是自己设置的，表项命中会更新TTL时间，TTL计到0会发出通告

### P4实现

　　具体的P4实现也看switch中的mac地址表老化，源码如下：

```
table dmac {
    reads {
        ingress_metadata.bd : exact;
        l2_metadata.lkp_mac_da : exact;
    }
    actions {
#ifdef OPENFLOW_ENABLE
        openflow_apply;
        openflow_miss;
#endif /* OPENFLOW_ENABLE */
        nop;
        dmac_hit;
        dmac_multicast_hit;
        dmac_miss;
        dmac_redirect_nexthop;
        dmac_redirect_ecmp;
        dmac_drop;
    }
    size : MAC_TABLE_SIZE;
    support_timeout: true;
}
```

　　这个就没啥可以解释的了，为什么是针对dmac老化，上面又是针对smac进行学习呢？

### PD_API

　　生成的PD_API感觉也不完整，估计是没有实现吧，暂时不用在意这些细节，看看就好：

```
p4_pd_status_t
p4_pd_dc_dmac_table_add_with_dmac_hit
(
 p4_pd_sess_hdl_t sess_hdl,
 p4_pd_dev_target_t dev_tgt,
 p4_pd_dc_dmac_match_spec_t *match_spec,
 p4_pd_dc_dmac_hit_action_spec_t *action_spec,
 uint32_t ttl,
 p4_pd_entry_hdl_t *entry_hdl
);
```

```
p4_pd_status_t
p4_pd_dc_dmac_set_entry_ttl(p4_pd_sess_hdl_t sess_hdl, p4_pd_entry_hdl_t entry_hdl, uint32_t ttl);

p4_pd_status_t
p4_pd_dc_dmac_enable_entry_timeout(p4_pd_sess_hdl_t sess_hdl,
                  p4_pd_notify_timeout_cb cb_fn,
                  uint32_t max_ttl,
                  void *client_data);
```

　　加入了timeout的表生成的API会带有ttl参数，在表项下发的时候就可以同时设置老化的时间，单位应该是ms

　　使用时需要使能，同时绑定回调函数，回调函数中应该会通告ttl到时的表项entry，很尴尬的是没有找到回调函数的定义，暂时不要在意这些细节，另外可以对每一个表项设置ttl时间

## 尾巴

　　以上就是P4如何实现学习和老化，敬请期待下一篇啦，如果有什么错误和欢迎指出哈