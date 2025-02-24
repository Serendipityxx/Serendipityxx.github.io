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

![MemoryContextData](image-1.png)

通过这个类型提供的上下文联系，可以画出内存上下文的结构。

![内存上下文的结构](image-2.png)


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

如果freelist中没有适合的chunk，则再另外分配
