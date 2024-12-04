---
layout:  post
title:   "postgreSQL的syscache"
date:   2024-11-19 14:22:00
author:  'Xiangxiao'
header-img: 'img/post-bg-2015.jpg'
catalog:   false
tags:
- c
- 系统缓存
- syscache
- postgres内核
---

# postgreSQL的虚拟列系统列

## 1 系统列
pg每张表都有几个系统列，系统列的特征是由系统隐式定义的，系统列在只有显示指定查询系统列，系统列才会被查询出来，其他情况都是隐藏的。

### 1.1 系统列的定义

系统列的定义在sysattr.h中。
```c
/*
 * Attribute numbers for the system-defined attributes
 */
#define SelfItemPointerAttributeNumber			(-1)
#define MinTransactionIdAttributeNumber			(-2)
#define MinCommandIdAttributeNumber				(-3)
#define MaxTransactionIdAttributeNumber			(-4)
#define MaxCommandIdAttributeNumber				(-5)
#define TableOidAttributeNumber					(-6)
#define FirstLowInvalidHeapAttributeNumber		(-7)
```

如果我们想要新增一个系统列的话就需要在这里新增，但是需要注意的是**在最后一个宏定义之前新增**。

随后在heap.c文件中添加系统列的定义，并将系统列添加到数组SysAtt[]中去。