---
layout:  post
title:   "共享哈希表"
date:   2024-8-12 09:44:00
author:  'Xiangxiao'
header-img: 'img/post-bg-2015.jpg'
catalog:   false
tags:
- c
- 共享哈希表
- postgres内核

---
# postgres共享哈希表

## 目录

- [1 😀哈希表实体的创建](#1-哈希表实体的创建)
- [2 哈希表的初始化](#2-哈希表的初始化)
- [3 对哈希表进行操作](#3-对哈希表进行操作)
  - [3.1 哈希表的新增](#31-哈希表的新增)
  - [3.2 哈希表的查找](#32-哈希表的查找)
- [4 hash\_search函数](#4-hash_search函数)

## 1 😀哈希表实体的创建

首先，写一个hashtable的实体

```c
#ifndef HASHTABLE_H
#define HASHTABLE_H

#include "postgres.h"
#include "storage/lwlock.h"
#include "utils/hsearch.h"

typedef struct
{
    /* data */
    char* key;
    int32 val;
}HashTableEntry;

extern Size HashTableShmemSize(void);
extern void Hashtable_init_shmem(void);

extern  HTAB * shared_hashtable;

#endif
```

这里需要有两个函数的声明和一个变量的声明。

其中HashTableShmemSize是计算hash表在共享内存中占多大的大小，Hashtable\_init\_shmem是实现hash表的初始化操作。shared\_hashtable是提供给操作的哈希表的名字。

然后是对hashtable头文件的实现

```c
#include "utils/hashtable.h"
#include "storage/spin.h"
#include "postgres.h"
#include "storage/lwlock.h"
#include "c.h"
#include "storage/shmem.h"

#define MAX_TABLE_SIZE 1024

HTAB * shared_hashtable = NULL;

Size
HashTableShmemSize(void)
{
    Size size = 0;
    size = add_size(size, sizeof(HashTableEntry)*1024);
    return size;
}

void 
Hashtable_init_shmem(void)
{
    HASHCTL hash_ctl;
    long    init_table_size,
            max_table_size;

    max_table_size = MAX_TABLE_SIZE;
    init_table_size = max_table_size / 2;
    hash_ctl.keysize = sizeof(char*);
    hash_ctl.entrysize = sizeof(HashTableEntry);

    shared_hashtable = ShmemInitHash("test hashtable",
                     init_table_size,
                     max_table_size,
                     &hash_ctl,
                     HASH_ELEM | HASH_BLOBS | HASH_PARTITION);
}
```

在这里设置这个hashtable最大能存储MAX\_TABLE\_SIZE 也就是1024个，然后通过HashTableShmemSize函数计算大小，大小就为1024个实体累加的总大小。

Hashtable\_init\_shmem对hashtable进行初始化，初始化创建的时候需要有个hashtable的控制信息结构体，如下所示：

```c
typedef struct HASHCTL
{
  /* Used if HASH_PARTITION flag is set: */
  long    num_partitions; /* # partitions (must be power of 2) */
  /* Used if HASH_SEGMENT flag is set: */
  long    ssize;      /* segment size */
  /* Used if HASH_DIRSIZE flag is set: */
  long    dsize;      /* (initial) directory size */
  long    max_dsize;    /* limit to dsize if dir size is limited */
  /* Used if HASH_ELEM flag is set (which is now required): */
  Size    keysize;    /* hash key length in bytes */
  Size    entrysize;    /* total user element size in bytes */
  /* Used if HASH_FUNCTION flag is set: */
  HashValueFunc hash;      /* hash function */
  /* Used if HASH_COMPARE flag is set: */
  HashCompareFunc match;    /* key comparison function */
  /* Used if HASH_KEYCOPY flag is set: */
  HashCopyFunc keycopy;    /* key copying function */
  /* Used if HASH_ALLOC flag is set: */
  HashAllocFunc alloc;    /* memory allocator */
  /* Used if HASH_CONTEXT flag is set: */
  MemoryContext hcxt;      /* memory context to use for allocations */
  /* Used if HASH_SHARED_MEM flag is set: */
  HASHHDR    *hctl;      /* location of header in shared mem */
} HASHCTL;
```

## 2 哈希表的初始化

然后创建hashtable的函数如下，其中name为hashtable的名字，init\_size为初始化哈希桶的数量，max\_size为最大哈希桶的数量，infoP就是上面提到的控制信息，hash\_flags是指创建哈希表的时候需要指定的一些配置信息，具体如下所示。

```c
HTAB *ShmemInitHash(const char *name, long init_size, long max_size,
               HASHCTL *infoP, int hash_flags);
```

```c
/* Flag bits f or hash_create; most indicate which parameters are supplied */
#define HASH_PARTITION  0x0001  /* Hashtable is used w/partitioned locking */
#define HASH_SEGMENT  0x0002  /* Set segment size */
#define HASH_DIRSIZE  0x0004  /* Set directory size (initial and max) */
#define HASH_ELEM    0x0008  /* Set keysize and entrysize (now required!) */
#define HASH_STRINGS  0x0010  /* Select support functions for string keys */
#define HASH_BLOBS    0x0020  /* Select support functions for binary keys */
#define HASH_FUNCTION  0x0040  /* Set user defined hash function */
#define HASH_COMPARE  0x0080  /* Set user defined comparison function */
#define HASH_KEYCOPY  0x0100  /* Set user defined key-copying function */
#define HASH_ALLOC    0x0200  /* Set memory allocator */
#define HASH_CONTEXT  0x0400  /* Set memory allocation context */
#define HASH_SHARED_MEM 0x0800  /* Hashtable is in shared memory */
#define HASH_ATTACH    0x1000  /* Do not initialize hctl */
#define HASH_FIXED_SIZE 0x2000  /* Initial size is a hard limit */
```

然后跟共享内存一样，需要在系统初始化共享内存的时候将需要的大小统计进去，并且一起初始化。

```c
size = add_size(size, HashTableShmemSize());

Hashtable_init_shmem();
```

## 3 对哈希表进行操作

### 3.1 哈希表的新增

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

### 3.2 哈希表的查找

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

## 4 hash\_search函数

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
