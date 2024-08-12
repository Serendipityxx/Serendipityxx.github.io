---
layout:  post
title:   "postgres共享哈希表"
date:   2024-8-12 09:20:00
author:  'Xiangxiao'
header-img: 'img/post-bg-2015.jpg'
catalog:   false
tags:
- c
- 共享哈希表
- postgres

---

# postgres共享内存

## 目录

- [1 一、创建内置函数](#1-一创建内置函数)
- [2 二、创建共享内存的声明shmstring.h文件](#2-二创建共享内存的声明shmstringh文件)
- [3 三、创建两个函数的实现](#3-三创建两个函数的实现)
- [4 四、在ipci文件中加载共享内存](#4-四在ipci文件中加载共享内存)
- [5 五、对哈希表进行操作](#5-五对哈希表进行操作)
  - [5.1 哈希表的新增](#51-哈希表的新增)
  - [5.2 哈希表的查找](#52-哈希表的查找)
- [6 六、hash\_search函数](#6-六hash_search函数)

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

## 5 五、对哈希表进行操作

### 5.1 哈希表的新增

首先是对哈希表的增加操作，我们要添加一个哈希键值对进去，首先得申请一个实体类型，然后将他的key值和value值赋给它，然后通过hash\_search的方法将这个key值传进去查找，如果查找到了就返回这个key值对应的bucket地址，当然我们这里是做新增操作，所以肯定是查找不到的，这个时候他会返回一个新的bucket地址，我们将这个类型强转为HashTableEntry也就是我们自己定义的实体类型，然后将从函数参数获取的values值赋给这个返回的实体类型的val值。具体通过以下代码实现：

```c
Datum
set_hashstring(PG_FUNCTION_ARGS)
{
  HashTableEntry * hentry;
  bool   found;
  char  *key;
  int32  val;

  key = text_to_cstring(PG_GETARG_TEXT_PP(0));
  val = PG_GETARG_INT32(1);
  hentry = (HashTableEntry *) hash_search(shared_hashtable, key,
                    HASH_ENTER, &found);
  hentry->val = val;
  // strlcpy(hentry->key, key, sizeof(hentry->key));
  PG_RETURN_TEXT_P(cstring_to_text("set success"));
}

```

### 5.2 哈希表的查找

然后就是哈希表的查找操作，同样的这里也需要声明一个哈希表的实体类型，用于接收哈希表查找返回的bucket强转之后的哈希实体类型，然后也是使用了hash\_search函数，将需要查找的key值传入，获取查找到的元素，然后再获取该实体的val值。

```c
Datum
get_hashstring(PG_FUNCTION_ARGS)
{
  HashTableEntry * hentry;
  bool   found;
  char  *key;
  int32  val;
  char  res[512];

  key = text_to_cstring(PG_GETARG_TEXT_PP(0));
  hentry = (HashTableEntry *) hash_search(shared_hashtable, key,
                    HASH_FIND, &found);
  if(!found)
  {
    elog(WARNING,"can not find it");
  }                  
  val = hentry->val;
  sprintf(res,"您要查询的key:%s对应的值为:%d",key,val);
  PG_RETURN_TEXT_P(cstring_to_text((const char *)res));
}
```

## 6 六、hash\_search函数

hash\_search函数是一个可用作哈希表的新增，修改，查找的函数，他的函数声明如下：

```c
void *hash_search(HTAB *hashp, const void *keyPtr, HASHACTION action, bool *foundPtr);
```

- hashp是一个指向哈希表的指针 ，类型是HTAB。
- keyPtr是一个指向key值的void类型的常量指针。
- action是一个控制hash\_search行为的flag。是一个枚举类型，有四个元素。
  - HASH\_FIND：用于通过key查找哈希表的值，返回的是一个bucket的地址。HASH\_FIND: look up key in table
  - HASH\_ENTER：用于哈希表的新增，也是先进行查找，然后如果不存在该key值则新create一个bucket返回回来。look up key in table, creating entry if not present。
  - HASH\_ENTER\_NULL：和上面的HASH\_ENTER一样，用于哈希表的新增，但是如果超过了哈希表所能存储的内存的大小，则返回NULL值。same, but return NULL if out of memory
  - HASH\_REMOVE：用于移除哈希表中的键值对，也是查找该key对应的哈希键值对，并删除。look up key in table, remove entry if present
- foundPtr是一个指向bool类型的指针，用于接收hash\_search函数的执行结果。

内核中还有一个跟这个函数很类似的函数，叫做hash\_search\_with\_hash\_value，仔细观察这两个函数的声明，可以发现后者多了一个参数，那就是hashvalue，是一个无符号的整型，用于接收传入key值对应的哈希值。再观察hash\_search函数可以发现，其实hash\_search函数在函数内部就是在调用hash\_search\_with\_hash\_value这个函数，只是说hash\_search函数在内部自行计算了hash值，并将这个hash值传入后者函数进行调用。hash\_search函数的代码实现如下：

```c
void *
hash_search(HTAB *hashp,
      const void *keyPtr,
      HASHACTION action,
      bool *foundPtr)
{
  return hash_search_with_hash_value(hashp,
                     keyPtr,
                     hashp->hash(keyPtr, hashp->keysize),
                     action,
                     foundPtr);
}
```

```c
extern void *hash_search_with_hash_value(HTAB *hashp, const void *keyPtr,
                     uint32 hashvalue, HASHACTION action,
                     bool *foundPtr);
```
