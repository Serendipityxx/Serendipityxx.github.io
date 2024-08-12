---
layout:  post
title:   "共享内存"
date:   2024-8-12 09:45:00
author:  'Xiangxiao'
header-img: 'img/post-bg-2015.jpg'
catalog:   false
tags:
- c
- 共享内存
- postgres内核

---
# postgres共享内存

## 目录

- [1 一、创建内置函数](#1-一创建内置函数)
- [2 二、创建共享内存的声明shmstring.h文件](#2-二创建共享内存的声明shmstringh文件)
- [3 三、创建两个函数的实现](#3-三创建两个函数的实现)
- [4 四、在ipci文件中加载共享内存](#4-四在ipci文件中加载共享内存)

## 1 一、创建内置函数

共享内存的使用，这里是开两个psql连接，一个连接调用内置函数set\_string设置一个字符串到共享内存中，另一个连接调用内置函数get\_string从共享内存中获取字符串并返回。

在include/catalog/pg\_proc.dat中增加内置函数的声明

```perl
{ oid => '111', descr => 'set a string in shame', prorettype => 'text',
  proargtypes => 'text', prosrc => 'set_string'},
{ oid => '226', descr => 'get a string in shame', prorettype => 'text',
  proargtypes => '', prosrc => 'set_string'},
```

## 2 二、创建共享内存的声明shmstring.h文件

然后在include/utils下面创建共享内存的声明

```c
#ifndef SHMSTRING_H
#define SHMSTRING_H

#include "postgres.h"
#include "storage/lwlock.h"


#define SHARED_MEM_NAME "my_shared_string"
#define MAX_SHARED_STRING_SIZE 1024

typedef struct {
    char data[MAX_SHARED_STRING_SIZE];
    Size len;
    LWLock mutex; // spinlock for synchronization
} SharedString;


extern bool string_init_shmem(void);
extern Size StringshareShmemSize(void);

extern SharedString *shared_string;
#endif
```

该声明需要定义一个结构体，是为共享内存的结构，还需定义两个函数，一个是StringshareShmemSize函数，负责计算返回该共享内存的大小，用于在postgres启动的时候提前预留大小。

另一个是string\_init\_shmem函数，用于在pg初始化共享内存的时候进行初始化。

这两个都是自定义函数。

## 3 三、创建两个函数的实现

在backend/uitls/adt下面创建shmstring.c文件，主要实现上面的两个自定义函数。

```c
#include "utils/shmstring.h"
#include "storage/spin.h"
#include "postgres.h"
#include "storage/lwlock.h"
#include "c.h"
#include "storage/shmem.h"

SharedString *shared_string = NULL;

Size
StringshareShmemSize(void)
{
  Size    size = 0;

  size = add_size(size, sizeof(SharedString));

  return size;
}


bool
string_init_shmem(void) {
    bool found;
    Size sz;
    sz = StringshareShmemSize();
    shared_string = (SharedString *) ShmemInitStruct(SHARED_MEM_NAME,sz,&found);
    memset(shared_string, 0, sizeof(SharedString));
    LWLockInitialize(&shared_string->mutex,LWLockNewTrancheId());
    return true;
}
```

## 4 四、在ipci文件中加载共享内存

在ipci.c文件中首先将shmstring所需要的大小加入到pg要申请的共享内存总大小，然后再将共享内存初始化的函数加到整个共享内存加载和初始化的地方。

```c#
size = add_size(size, StringshareShmemSize());

string_init_shmem();

```
