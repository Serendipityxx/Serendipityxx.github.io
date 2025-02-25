---
layout:  post
title:   "postgreSQL的内存上下文"
date:   2025-02-24 13:39:00
author:  'Xiangxiao'
header-img: 'img/post-bg-2015.jpg'
catalog:   true
tags:
- 内存上下文
- postgres内核
---

-------------------------------
# 数据库内存上下文

当操作系统发生OOM的时候，对于数据库来说可能出现操作系统杀掉数据库的会话连接，或者更坏的情况导致杀掉数据库的postmaster进程，导致数据库宕机重启。

在PostgreSQL数据库7.0以前，采用大量的指针传递来处理大量的查询，如果这个时候只依赖使用malloc和free进行内存的申请和释放的话，很难做到及时释放，存在大量内存泄漏的风险。

所以PostgreSQL数据库在后期采用了内存上下文机制对内存进行管理，内存上下文机制管理数据库的内存分配。在申请和释放内存的时候变成了palloc和pfree。

## 1 内存上下文中涉及的数据结构

### 1.1 AllocSetContext

```c
typedef struct AllocSetContext
{
	MemoryContextData header;	/* Standard memory-context fields */
	/* Info about storage allocated in this context: */
	AllocBlock	blocks;			/* head of list of blocks in this set */
	AllocChunk	freelist[ALLOCSET_NUM_FREELISTS];	/* free chunk lists */
	/* Allocation parameters for this context: */
	Size		initBlockSize;	/* initial block size */
	Size		maxBlockSize;	/* maximum block size */
	Size		nextBlockSize;	/* next block size to allocate */
	Size		allocChunkLimit;	/* effective chunk size limit */
	AllocBlock	keeper;			/* keep this block over resets */
	/* freelist this context could be put in, or -1 if not a candidate: */
	int			freeListIndex;	/* index in context_freelists[], or -1 */
} AllocSetContext;

```
- header 内存上下文头部信息
- blocks 内存块链表，方便在回收的时候遍历所有内存块
- freelist 空闲链表，内存片回收之后保存在这里，分配内存的时候优先从freelist中获取空闲的chunk
- initBlockSize 初始化的时候需要申请的block大小
- maxBlockSize 申请的最大的block大小
- nextBlockSize 下一次申请的block大小
- allocChunkLimit 申请内存片大小的阈值，超过这个值，单独进行分配
- keeper 内存上下文重置的时候不释放的内存块，比如说在事务中，对于上下文频繁重置，保留一个常用块可提升性能
- freeListIndex 标识当前上下文是否可以被全局空闲列表缓存，-1标识不缓存，非负数表示对于空闲链表的索引


### 1.2 MemoryContextData

```c
typedef struct MemoryContextData
{
	NodeTag		type;			/* identifies exact kind of context */
	/* these two fields are placed here to minimize alignment wastage: */
	bool		isReset;		/* T = no space alloced since last reset */
	bool		allowInCritSection; /* allow palloc in critical section */
	Size		mem_allocated;	/* track memory allocated for this context */
	const MemoryContextMethods *methods;	/* virtual function table */
	MemoryContext parent;		/* NULL if no parent (toplevel context) */
	MemoryContext firstchild;	/* head of linked list of children */
	MemoryContext prevchild;	/* previous child of same parent */
	MemoryContext nextchild;	/* next child of same parent */
	const char *name;			/* context name (just for debugging) */
	const char *ident;			/* context ID if any (just for debugging) */
	MemoryContextCallback *reset_cbs;	/* list of reset/delete callbacks */
} MemoryContextData;
```

- type 标识上下文的具体类型
- isReset 标记当前上下文是否已重置，若为true表示重置后为重新分配，可以跳过某些请扫操作
- allowInCritSection  是否允许在关键区申请内存
- mem_allocated 记录当前上下文及其所有子上下文分配的内存总量
- methods 指向一个函数指针，定义上下文的具体操作
- parent 指向父上下文
- firstchild 指向该上下文的第一个子上下文
- prevchild 指向同一级上下文的前一个
- nextchild 指向同一级上下文的后一个
- name 上下文的名称
- ident 上下文的标识，为调试提供信息
- reset_cbs 上下文被重置和删除的时候的回调函数

### 1.3 AllocBlockData 双向链表

```c
typedef struct AllocBlockData
{
	AllocSet	aset;			/* aset that owns this block */
	AllocBlock	prev;			/* prev block in aset's blocks list, if any */
	AllocBlock	next;			/* next block in aset's blocks list, if any */
	char	   *freeptr;		/* start of free space in this block */
	char	   *endptr;			/* end of space in this block */
} AllocBlockData;
```
- aset 这个block所属的内存分配器
- prev 这个block的前置节点
- next 这个block的后置节点
- freeptr 这个block可用空间的起始地址
- endptr 这个block结束的地址，endptr - freeptr获取当前块的剩余空间

### 1.4 AllocChunkData

```c
typedef struct AllocChunkData
{
	/* size is always the size of the usable space in the chunk */
	Size	size;
#ifdef MEMORY_CONTEXT_CHECKING
	/* when debugging memory usage, also store actual requested size */
	/* this is zero in a free chunk */
	Size	requested_size;

#define ALLOCCHUNK_RAWSIZE  (SIZEOF_SIZE_T * 2 + SIZEOF_VOID_P)
#else
#define ALLOCCHUNK_RAWSIZE  (SIZEOF_SIZE_T + SIZEOF_VOID_P)
#endif	/* MEMORY_CONTEXT_CHECKING */

	/* ensure proper alignment by adding padding if needed */
#if (ALLOCCHUNK_RAWSIZE % MAXIMUM_ALIGNOF) != 0
	char	padding[MAXIMUM_ALIGNOF - ALLOCCHUNK_RAWSIZE % MAXIMUM_ALIGNOF];
#endif

	/* aset is the owning aset if allocated, or the freelist link if free */
	void	*aset;
	/* there must not be any padding to reach a MAXALIGN boundary here! */
}	AllocChunkData;
```

- size chunk的可用空间大小，内存分配器通过这个判断剩余空间是否满足
- requested_size 调试模式下，用户实际申请的大小
- aset 在已分配的时候指向所属的AllocSet，在未分配的时候指向下一个chunk
- padding 通过MAXINUM_ALIGNOF确保结构体自身满足最大对其要求，宏ALLOCCHUNK_RAWSIZE计算元数据总大小，若未对齐自动添加padding

### 1.5 MemoryContextMethods

```c
typedef struct MemoryContextMethods
{
	void	 *(*alloc) (MemoryContext context, Size size);
	/* call this free_p in case someone #define's free() */
	void	(*free_p) (MemoryContext context, void *pointer);
	void	*(*realloc) (MemoryContext context, void *pointer, Size size);
	void	(*reset) (MemoryContext context);
	void	(*delete_context) (MemoryContext context);
	Size	(*get_chunk_space) (MemoryContext context, void *pointer);
	bool	(*is_empty) (MemoryContext context);
	void	(*stats) (MemoryContext context,
			 MemoryStatsPrintFunc printfunc, void *passthru,
			 MemoryContextCounters *totals,
			 bool print_to_stderr);
#ifdef MEMORY_CONTEXT_CHECKING
	void	(*check) (MemoryContext context);
#endif
} MemoryContextMethods;
```

- alloc 内存分配函数，对应malloc
- free_p 内存释放函数，对应free
- realloc 内存重分配函数，对应relloc
- reset 快速重置整个内存上下文，释放所有块，但是保存上下文结构
- delete_context 彻底销毁整个内存上下文，释放所有资源
- get_chunk_space 计算指定内存片的实际占用空间
- is_empty 检测上下文是否为空
- stats 统计内存使用情况，可以通过回调函数输出详细信息
- check 内存完整性检查，用于调式场景，检测野指针、内存越界等

## 2 内存上下文的组织结构

在前面提到的MemoryContextData结构体其实可以看做是一个抽象类，它包含了内存上下文之间的联系，以及内存上下文进行操作的函数，可以有多种实现，但是目前只有AllocSetContext这一种实现。在C语言中实现多态和继承，AllocSetContext的起始位置必须是MemoryContextData。

![MemoryContextData](/img/postimg/image-1.png)

通过这个类型提供的上下文联系，可以画出内存上下文的结构。

![内存上下文的结构](/img/postimg/image-2.png)

AllocSetContext整体结构

![AllocSetContext整体结构](/img/postimg/image-3.png)


## 3 内存上下文的细节

PostgreSQL将内存分为内存块和内存片，其中内存块是通过malloc函数向操作系统申请获得的，而一个内存块中将会有一个或者多个内存片，**内存片是PostgreSQL中的最小存储内存单元**。

这样做的话一是可以**减少系统调用**，只需要使用malloc申请一次内存，并将这个内存分割为一个或者多个内存片返回使用，二是**提高内存使用率**，每一次malloc申请返回的内存，会有一个Header记录内存总大小，否则free无法正常工作，每一次malloc都会有一段Header，而以上实现则只需要一次malloc，只有一个块的Header。

### 3.1 内存分配alloc函数的细节

alloc函数接收一个内存上下文context参数和一个size参数，当进行申请分配的时候首先会判断size的大小，将size的大小和context->allocChunkLimit进行比较，如果需要申请的大小大于所允许的最大chunk大小，则单独申请一块block只分割为一个chunk，大小即为size大小然后返回。

```c
	if (size > set->allocChunkLimit)
	{
		chunk_size = MAXALIGN(size);
		blksize = chunk_size + ALLOC_BLOCKHDRSZ + ALLOC_CHUNKHDRSZ;
		block = (AllocBlock) malloc(blksize);
		if (block == NULL)
			return NULL;

		context->mem_allocated += blksize;

		block->aset = set;
		block->freeptr = block->endptr = ((char *) block) + blksize;

		chunk = (AllocChunk) (((char *) block) + ALLOC_BLOCKHDRSZ);
		chunk->aset = set;
		chunk->size = chunk_size;
#ifdef MEMORY_CONTEXT_CHECKING
		chunk->requested_size = size;
		/* set mark to catch clobber of "unused" space */
		if (size < chunk_size)
			set_sentinel(AllocChunkGetPointer(chunk), size);
#endif
#ifdef RANDOMIZE_ALLOCATED_MEMORY
		/* fill the allocated space with junk */
		randomize_mem((char *) AllocChunkGetPointer(chunk), size);
#endif
…………
```

如果size大小合适的话，首先去freelist中进行分配。

```c
fidx = AllocSetFreeIndex(size);//根据size大小获得在freelist中的位置
	chunk = set->freelist[fidx];
	if (chunk != NULL)
	{
		Assert(chunk->size >= size);

		set->freelist[fidx] = (AllocChunk) chunk->aset;

		chunk->aset = (void *) set;//将chunk的aset指向所属的AllocSet

#ifdef MEMORY_CONTEXT_CHECKING
		chunk->requested_size = size;
		/* set mark to catch clobber of "unused" space */
		if (size < chunk->size)
			set_sentinel(AllocChunkGetPointer(chunk), size);
#endif
#ifdef RANDOMIZE_ALLOCATED_MEMORY
		/* fill the allocated space with junk */
		randomize_mem((char *) AllocChunkGetPointer(chunk), size);
#endif

		/* Ensure any padding bytes are marked NOACCESS. */
		VALGRIND_MAKE_MEM_NOACCESS((char *) AllocChunkGetPointer(chunk) + size,
								   chunk->size - size);

		/* Disallow external access to private part of chunk header. */
		VALGRIND_MAKE_MEM_NOACCESS(chunk, ALLOCCHUNK_PRIVATE_LEN);

		return AllocChunkGetPointer(chunk);
	}
```

如果freelist中没有适合的chunk，则在block分配一个chunk，如果block中的内存不足的话，则需要再申请一个block然后再分配一个chunk。

![alt text](/img/postimg/image-4.png)

### 3.2 内存释放AllocSetFree函数细节

该函数接受两个参数：

- MemoryContext：内存上下文
- pointer：需要释放的资源指针

释放的逻辑是，如果释放的这个是单片块的话，则将这个块从blocks链表里面去除，然后再free这个块，如果这是一个普通的chunk的话，则直接将这个chunk释放并放如对应的freelist中。

```c
    static void
    AllocSetFree(MemoryContext context, void *pointer)
    {
	AllocSet	set = (AllocSet) context;
	AllocChunk	chunk = AllocPointerGetChunk(pointer);//获取需要释放的chunk

	/* Allow access to private part of chunk header. */
	VALGRIND_MAKE_MEM_DEFINED(chunk, ALLOCCHUNK_PRIVATE_LEN);

#ifdef MEMORY_CONTEXT_CHECKING
	/* Test for someone scribbling on unused space in chunk */
	if (chunk->requested_size < chunk->size)
		if (!sentinel_ok(pointer, chunk->requested_size))
			elog(WARNING, "detected write past chunk end in %s %p",
				 set->header.name, chunk);
#endif

	if (chunk->size > set->allocChunkLimit)//如果是单片块的话
	{
		/*
		 * Big chunks are certain to have been allocated as single-chunk
		 * blocks.  Just unlink that block and return it to malloc().
		 */
		AllocBlock	block = (AllocBlock) (((char *) chunk) - ALLOC_BLOCKHDRSZ);//获取单片块

		/*
		 * Try to verify that we have a sane block pointer: it should
		 * reference the correct aset, and freeptr and endptr should point
		 * just past the chunk.
		 */
		if (block->aset != set ||
			block->freeptr != block->endptr ||
			block->freeptr != ((char *) block) +
			(chunk->size + ALLOC_BLOCKHDRSZ + ALLOC_CHUNKHDRSZ))
			elog(ERROR, "could not find block containing chunk %p", chunk);

		/* OK, remove block from aset's list and free it */
		if (block->prev)
			block->prev->next = block->next;//在链表中将这个block移除
		else
			set->blocks = block->next;
		if (block->next)
			block->next->prev = block->prev;

		context->mem_allocated -= block->endptr - ((char *) block);//减去申请的块

#ifdef CLOBBER_FREED_MEMORY
		wipe_mem(block, block->freeptr - ((char *) block));
#endif
		free(block);//释放这个块
	}
	else
	{
		/* Normal case, put the chunk into appropriate freelist */
		int			fidx = AllocSetFreeIndex(chunk->size);

		chunk->aset = (void *) set->freelist[fidx];//直接将这个chunk链接到freelist

#ifdef CLOBBER_FREED_MEMORY
		wipe_mem(pointer, chunk->size);
#endif

#ifdef MEMORY_CONTEXT_CHECKING
		/* Reset requested_size to 0 in chunks that are on freelist */
		chunk->requested_size = 0;
#endif
		set->freelist[fidx] = chunk;
	}
}
```

### 3.3 AllocSetRealloc函数细节

该函数接受三个参数，一是内存上下文，二是需要relloc的对象指针，三是需要relloc的大小。

也是分情况来看的，当是要relloc的对象是一个单片块的时候，则需要重新计算chunk和block的大小，并使用relloc去获取结果返回；如果不是的话，就需要考虑oldsize和新提供的size大小关系，如果oldsize较大则直接返回pointer，否则的话则重新AllocSet申请一个内存片返回。

### 3.4 AllocSetGetChunkSpace函数细节

该函数接受两个参数，一是内存上下文，二是对象指针

```c
/*
 * AllocSetGetChunkSpace
 *		Given a currently-allocated chunk, determine the total space
 *		it occupies (including all memory-allocation overhead).
 */
static Size
AllocSetGetChunkSpace(MemoryContext context, void *pointer)
{
	AllocChunk	chunk = AllocPointerGetChunk(pointer);
	Size		result;

	VALGRIND_MAKE_MEM_DEFINED(chunk, ALLOCCHUNK_PRIVATE_LEN);//标记chunk的头部访问是合法的，防止valgrind误报
	result = chunk->size + ALLOC_CHUNKHDRSZ;//返回的大小需要包括chunk的头部大小
	VALGRIND_MAKE_MEM_NOACCESS(chunk, ALLOCCHUNK_PRIVATE_LEN);//恢复valgrind标记
	return result;
}
```

### 3.5 AllocSetIsEmpty函数

该函数接受一个内存上下文变量作为参数

```c
/*
 * AllocSetIsEmpty
 *		Is an allocset empty of any allocated space?
 */
static bool
AllocSetIsEmpty(MemoryContext context)
{
	/*
	 * For now, we say "empty" only if the context is new or just reset. We
	 * could examine the freelists to determine if all space has been freed,
	 * but it's not really worth the trouble for present uses of this
	 * functionality.
	 */
	if (context->isReset)
		return true;
	return false;
}

```

### 3.6 AllocSetDelete函数

该函数接受一个内存上下文作为参数

首先判断上下文是否可以被全局freelist缓存，进行缓存，然后对block全部free。

```c
if (set->freeListIndex >= 0) {
    // 如果上下文支持重用，则将其加入空闲列表而非直接销毁
    AllocSetFreeList *freelist = &context_freelists[set->freeListIndex];

    if (!context->isReset)
        MemoryContextResetOnly(context); // 重置上下文到初始状态

    // 若空闲列表已满，先释放旧条目
    if (freelist->num_free >= MAX_FREE_CONTEXTS) {
        while (freelist->first_free != NULL) {
            AllocSetContext *oldset = freelist->first_free;
            freelist->first_free = (AllocSetContext *) oldset->header.nextchild;
            freelist->num_free--;
            free(oldset); // 释放旧的空闲上下文
        }
    }

    // 将当前上下文加入空闲列表
    set->header.nextchild = (MemoryContext) freelist->first_free;
    freelist->first_free = set;
    freelist->num_free++;
    return; // 直接返回，不销毁
}
```

释放内存：

```c
// 遍历并释放所有内存块（跳过保留块 "keeper"）
while (block != NULL) {
    AllocBlock  next = block->next;
    if (block != set->keeper) { // 保留块最后释放
        context->mem_allocated -= block->endptr - ((char *) block); // 更新已分配内存统计

#ifdef CLOBBER_FREED_MEMORY
        wipe_mem(block, block->freeptr - ((char *) block)); // 调试时覆盖已释放内存（检测野指针）
#endif
        free(block); // 释放非保留块
    }
    block = next;
}

// 确保仅保留块未释放
Assert(context->mem_allocated == keepersize);

// 释放上下文头（包含保留块）
free(set);
```

当然还有check函数和stats函数，这个此次不做分析，可以看代码。


## 参考链接

[PostgreSQL内存上下文系统设计概述_toptransactioncontext-CSDN博客](https://blog.csdn.net/kmblack1/article/details/136296473)

[PostgreSQL内存上下文_postgresql的memorycontext-CSDN博客](https://blog.csdn.net/shiyuefei1004/article/details/108869476)

[PG源码分析系列：内存上下文-阿里云开发者社区](https://developer.aliyun.com/article/711925)

[深入理解 PostgreSQL 中的内存上下文（MemoryContext）](https://smartkeyerror.com/PostgreSQL-MemoryContext)

[Postgresql内存池源码分析_psql内存分配源码解读-CSDN博客](https://mingjie.blog.csdn.net/article/details/89432427?spm=1001.2014.3001.5502)

