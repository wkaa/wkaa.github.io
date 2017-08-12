---
layout: post
title: 对jhash的一点分析
categories: C jhash
description: 分析一下jhash吧
keywords: C jhash
---

　　之前在看内核里IP报文重组的源码时就看到了jhash，也没什么概念，就想着反正内核里都用，也拿来用就是了，现在来分析一下吧。

## hash的好处

　　hash的中文意思是“散列”，可解释为：分散排列。一个好的hash函数应该做到对所有元素平均分散排列，尽量避免或者降低他们之间的冲突（Collision）。有必要再次提醒大家的是，hash函数的选择必须慎重，如果不幸所有的元素之间都产生了冲突，那么hash表将退化为链表，其性能会大打折扣，时间复杂度迅速降为O(n)，绝对不要存在任何侥幸心理，因为那是相当危险的。历史上就出现过利用Linux内核hash函数的漏洞，成功构造出大量使hash表发生碰撞的元素，导致系统被DoS，所以目前内核的大部分hash函数都有一个随机数作为参数进行掺杂，以使其最后的值不能或者是不易被预测。这又对 hash函数提出了第二点安全方面的要求：hash函数最好是单向的，并且要用随机数进行掺杂。提到单向，你也许会想到单向散列函数md4和md5，很不幸地告诉你，他们是不适合的，因为hash函数需要有相当好的性能。

　　Linux内核里面用的jhash是一个久经考验，并被实践证明经得起考验的hash函数。

## jhash源码

　　这里就贴出部分源码，来自2.6内核

```
/* NOTE: Arguments are modified. */
#define __jhash_mix(a, b, c) \
{ \
  a -= b; a -= c; a ^= (c>>13); \
  b -= c; b -= a; b ^= (a<<8); \
  c -= a; c -= b; c ^= (b>>13); \
  a -= b; a -= c; a ^= (c>>12);  \
  b -= c; b -= a; b ^= (a<<16); \
  c -= a; c -= b; c ^= (b>>5); \
  a -= b; a -= c; a ^= (c>>3);  \
  b -= c; b -= a; b ^= (a<<10); \
  c -= a; c -= b; c ^= (b>>15); \
}

/* The golden ration: an arbitrary value */
#define JHASH_GOLDEN_RATIO	0x9e3779b9

static inline u32 jhash_3words(u32 a, u32 b, u32 c, u32 initval)
{
    a += JHASH_GOLDEN_RATIO;
    b += JHASH_GOLDEN_RATIO;
    c += initval;

    __jhash_mix(a, b, c);

    return c;
}
```

## 分析

　　这上面两个宏一个函数，我们倒过来介绍，`jhash_3words`函数就是一个计算hash的函数，可以对三个参数进行计算hash值，第四个参数需要传入一个随机数，这个随机数需要在第一次使用前生成好，然后要一直使用，通过这个随机数就能够保证每次运行程序使用jhash时的状态都不一样，不易被预测。

　　`JHASH_GOLDEN_RATIO`宏就是一个随机数，使用这个随机数对输入进行叠加，一定程度提高不可预测性。

　　`__jhash_mix`就是jhash计算的核心了，恕我愚钝，完全看不懂为什么这么做，总之通过了这个函数，三个输入参数都被充分地“混合”了，应该在设计上也有充分平均的作用，使得不同输入充分平均到整个hash表中，另外这里面只用了简单的减法、移位、异或运算，速度上可以得到保证。

　　新版内核中有些变化，但是大体上结构不变，主要变的还是这个mix计算，如下是4.12内核的mix函数，我同样看不懂，一起贴上来欣赏一下吧。。:joy::joy::joy:

```
/* __jhash_mix -- mix 3 32-bit values reversibly. */
#define __jhash_mix(a, b, c)			\
{						\
	a -= c;  a ^= rol32(c, 4);  c += b;	\
	b -= a;  b ^= rol32(a, 6);  a += c;	\
	c -= b;  c ^= rol32(b, 8);  b += a;	\
	a -= c;  a ^= rol32(c, 16); c += b;	\
	b -= a;  b ^= rol32(a, 19); a += c;	\
	c -= b;  c ^= rol32(b, 4);  b += a;	\
}
```
