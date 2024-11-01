---
layout:  post
title:   "postgreSQL的锁"
date:   2024-11-01 9:22:00
author:  'Xiangxiao'
header-img: 'img/post-bg-2015.jpg'
catalog:   false
tags:
- c
- 锁
- 常规锁
- postgres内核
---

# 1 并发控制
数据库中的对象是共享的，如果同一时间不同的用户对同一个对象进行修改，就会出现数据不一致的情况，违反事务的隔离性原则，所以如果需要实现并发访问，要对这个过程加以控制。

postgreSQL针对于并发访问控制，一是采用的基于锁的机制，也就是两阶段锁机制（Two phase lock，2PL），二是基于时间戳的机制，也就是MVCC机制。

在pg中，使用快照隔离Sanpshot Isolation来实现多版本并发控制，同时以两阶段锁机制为辅，在执行DDL的时候使用2PL，在执行DML的时候使用SI。

简单的来说就是把锁操作分为两个阶段：加锁阶段和解锁阶段，并且强制要求在加锁阶段不能释放锁，在解锁的阶段不能申请锁。

## 1.1 锁的分类
在postgreSQL中锁可以分为以下三类：
- 自旋锁（SpinLock）：是一种结合硬件提供的原子操作来对一些共享变量进行封锁，适用于临界区比较小的情况。
- 轻量锁（Lightweight Lock）：在pg中使用了大量的共享内存，不同的进程需要对这些共享内存进行频繁的访问和修改，轻量锁负责保护这些共享内存中的数据结构，
- 常规锁（Regular Lock）：pg的两阶段所就是借助常规锁来实现的，根据加锁的对象不同，它又分为了多种粒度的锁，包括表、页面、元组、事务ID等。

# 2 各种锁的详解
## 2.1 自旋锁 Spin Lock
顾名思义，自旋锁就是不断旋转的锁，如果一个进程想要访问临界区，就必须先获取锁资源，如果不能获取锁资源，就一直在原地`忙等`，直到获取锁资源。

显然这种自旋非常的浪费CPU资源，所以其适用于临界资源非常“小”，持有锁的进程很快就会释放锁，这种时候在自旋对CPU资源的浪费小于释放CPU资源带来的上下文切换的消耗小，那就适用于使用自旋锁。故其有以下两个特点：
- 不想释放资源
- 保护的临界区“小”

但是需要注意的是这里的自旋锁的实现必须依赖原子操作，也就是说这个操作可能在c语言是一条语句，但是转换为汇编指令则可能是多条指令才能达到的效果，当进程在访问的时候可能出现同时获取资源的情况。

比如`*lock != 0`这条语句转换为汇编指令为`movq -0x8(%rbq), %rax`和`cmpl $0x0, (%rax)`这就不是原子操作。

pg借助TAS（TEST-AND-SET）来实现自旋锁，它的流程是向内存写入1，并返回内存变量的原值。
```c
int TAS(int *lock)
{
    int temp = *lock;
    *lock = 1;
    return *lock;
}
```
除了TAS还有另一种模型，CAS（COMPARE-AND-SWAP），它通过比较锁的值和期望值，如果锁的值和期望值相同，则设置为新值返回true，否则不设置新值，返回false。
```c
int CAS(int *lock, int expect, int new)
{
    if(*lock == expect)
    {
        *lock = new;
        return true;
    }
    return false;
}
```

在spin.h文件中定义了如下接口：
```c
#define SpinLockInit(lock)	S_INIT_LOCK(lock)

#define SpinLockAcquire(lock) S_LOCK(lock)

#define SpinLockRelease(lock) S_UNLOCK(lock)

#define SpinLockFree(lock)	S_LOCK_FREE(lock)


extern int	SpinlockSemas(void);
extern Size SpinlockSemaSize(void);

#ifndef HAVE_SPINLOCKS
extern void SpinlockSemaInit(void);
extern PGDLLIMPORT PGSemaphore *SpinlockSemaArray;
#endif
```
这里可以看出来pg采用了两种方式去实现自旋锁，一种是硬件提供的原子操作去实现，另一种是如果机器没有TAS指令集，则通过自己定义的信号量PGSemaphore来仿真SpinLock，但是后者的效率不如前者，这里通过宏定义`HAVE_SPINLOCKS`去标识机器是否有TAS指令集。

同时在此基础上，pg在原地自旋的模式下还做了一些优化，比如说在自旋的过程中使用空指令，让其在自旋的过程中让CPU歇一会。

其次当多次尝试TAS之后还是无法获取自旋锁，则进入sleep，还有如果旋转了很长时间还是无法获取自旋锁的话则进入自杀模式。

## 2.2 轻量级锁 Lightweight Lock

自旋锁是一个互斥锁，对于临界区比较小的情况，能保证多个进程对临界区的访问是互斥的，但是在pg数据库中，多个进程回去读写共享内存，尤其是大量的读操作，这些读操作并不冲突，因为他们不会修改数据，所以就需要轻量级锁来解决这个问题。

轻量级锁有两种模式，共享和排他，在读操作的时候就加上共享锁，这样保证在其他进程读的时候不冲突，而又杜绝了其他写操作。在写操作的时候就加上排他锁，这样保证在写操作执行的过程中，其他进程既不可以读也不可以写。

轻量级锁的相容性矩阵如下：
| 相容性矩阵 | LW_SHARED | LW_EXCLUSIVE |
| --------- | --------- | ------------ |
| LW_SHARED |    √      |      ×       |
| LW_EXCLUSIVE |    ×   |      ×       |


轻量级锁类型定义在lwlocknames.h中，这个文件是在编译的时候由lwlocknames.txt生成的，在这里面定义了pg的轻量级锁的类型。

```c
#define ShmemIndexLock (&MainLWLockArray[1].lock)
#define OidGenLock (&MainLWLockArray[2].lock)
#define XidGenLock (&MainLWLockArray[3].lock)
...
```
从这个宏定义可以看出，在这个所有的轻量级锁的类型都存储在MainLWLockArray里面。在上面这种为individual lwlock，每个individual lwlock都有自己固定要保护的对象。individual lwlock的使用方法如下：
```c
LWLockAcquire(ShmemIndexLock, LW_EXCLUSIVE);
//这里对需要保护的变量进行写操作
LWLockRelease(ShmemIndexLock);
```
在individual lwlock中，每个lwlock都对应一个tranche id，它具有全局唯一性，在MainLWLockArray中前`NUM_INDIVIDUAL_LWLOCKS - 1`都是individual lwlock.

除了individual lwlock之外，还有Builtin tranche，每一个builtin tranche可能对应多个lwlock，他代表的是一组lwlocks，这组lwlocks虽然各自封锁各自的内容，但是他们的功能相同。builtin tranche包含如下类型，类型定义在lwlock.h文件的BuiltinTrancheIds枚举类型中。
```c
typedef enum BuiltinTrancheIds
{
	LWTRANCHE_XACT_BUFFER = NUM_INDIVIDUAL_LWLOCKS,
	LWTRANCHE_COMMITTS_BUFFER,
	LWTRANCHE_SUBTRANS_BUFFER,
	LWTRANCHE_MULTIXACTOFFSET_BUFFER,
	LWTRANCHE_MULTIXACTMEMBER_BUFFER,
	LWTRANCHE_NOTIFY_BUFFER,
	LWTRANCHE_SERIAL_BUFFER,
	LWTRANCHE_WAL_INSERT,
	LWTRANCHE_BUFFER_CONTENT,
	LWTRANCHE_REPLICATION_ORIGIN_STATE,
	LWTRANCHE_REPLICATION_SLOT_IO,
	LWTRANCHE_LOCK_FASTPATH,
	LWTRANCHE_BUFFER_MAPPING,
	LWTRANCHE_LOCK_MANAGER,
	LWTRANCHE_PREDICATE_LOCK_MANAGER,
	LWTRANCHE_PARALLEL_HASH_JOIN,
	LWTRANCHE_PARALLEL_QUERY_DSA,
	LWTRANCHE_PER_SESSION_DSA,
	LWTRANCHE_PER_SESSION_RECORD_TYPE,
	LWTRANCHE_PER_SESSION_RECORD_TYPMOD,
	LWTRANCHE_SHARED_TUPLESTORE,
	LWTRANCHE_SHARED_TIDBITMAP,
	LWTRANCHE_PARALLEL_APPEND,
	LWTRANCHE_PER_XACT_PREDICATE_LIST,
	LWTRANCHE_PGSTATS_DSA,
	LWTRANCHE_PGSTATS_HASH,
	LWTRANCHE_PGSTATS_DATA,
	LWTRANCHE_FIRST_USER_DEFINED
}			BuiltinTrancheIds;
```
这些Builtin Tranche对应的锁一部分保存在MainLWLockArray中，另一半部分被保存在使用他们的结构体中，如下所示：
```c
//存储在MainLWLockArray中
/* Register named extension LWLock tranches in the current process. */
	for (int i = 0; i < NamedLWLockTrancheRequests; i++)
		LWLockRegisterTranche(NamedLWLockTrancheArray[i].trancheId,
					NamedLWLockTrancheArray[i].trancheName);

//存储在自己的结构体中 xlog.c:4465 (pg15.8)
//这里初始化了NUM_XLOGINSERT_LOCKS个轻量锁
//这些轻量锁的tranche id都是lwtranche_wal_insert
//这些轻量锁没有保存在MainLWLockArray里面
	for (i = 0; i < NUM_XLOGINSERT_LOCKS; i++)
	{
		LWLockInitialize(&WALInsertLocks[i].l.lock, LWTRANCHE_WAL_INSERT);
		WALInsertLocks[i].l.insertingAt = InvalidXLogRecPtr;
		WALInsertLocks[i].l.lastImportantAt = InvalidXLogRecPtr;
	}
```
但是无论是individual lwlocks还是builtin tranches，他们都被保存在共享内存中，只是保存的方式不相同而已。

这里先给出轻量级锁的结构体定义：
```c
typedef struct LWLock
{
	uint16		tranche;		/* tranche ID */
    //这是一个原子变量，用于保存轻量级锁的状态
	pg_atomic_uint32 state;		/* state of exclusive/nonexclusive lockers */
    //轻量级锁的等待队列
	proclist_head waiters;		/* list of waiting PGPROCs */
#ifdef LOCK_DEBUG
	pg_atomic_uint32 nwaiters;	/* number of waiters 等待者的数量 */
    //最后一次获取排他锁的进程
	struct PGPROC *owner;		/* last exclusive owner of the lock */
#endif
} LWLock;
```
在早期轻量级锁是通过自旋锁实现的，但是随着性能的提升，轻量级锁的实现开始借助原子操作来实现，`pg_atomic_uint32 state;`关注这个状态的变量，这是一个32位的无符号整型，其中他的低24位作为存储共享锁的计数器，因此一个轻量级锁最多拥有2^24个共享锁持有者。有1位作为排他锁的标记，因为同一时刻最多只能拥有一个持锁者。
![alt text](image.png)

- 当申请一个共享锁的时候：
    - 如果里面是共享锁或者无锁 ---->  直接获取
    - 如果里面排他锁 ---->  进入等待队列
- 当申请一个排他锁的时候：
    - 如果里面是共享锁或者排他锁 ----> 进入等待队列
    - 如果里面没有锁 ----> 直接获取

同样的，当一个锁释放的时候发现锁已经没有持有者，则需要唤醒等待队列的进程：
- 如果等待队列第一个是排他锁 ----> 只唤醒一个排他锁
- 如果等待队列第一个是共享锁 ----> 唤醒整个等待队列所有的共享锁

在比较极端的情况下，如果申请一个排他锁，然后目前持有的是共享锁，则进入等待队列，在这个过程中不断的申请共享锁，那么这个排他锁将持续等待，出现排他锁饥饿的情况。
### 2.2.1 获取锁
获取锁的调用`LWLockAcquire`方法，释放调用`LWLockRelease`方法。
获取的步骤大致为：
1. 判断锁的持有数量是否超出能记录的持有列表大小。
```c
if (num_held_lwlocks >= MAX_SIMUL_LWLOCKS)
	elog(ERROR, "too many LWLocks taken");
```
2. 尝试获取，成功则返回
```c
mustwait = LWLockAttemptLock(lock, mode);
if (!mustwait)
{
	LOG_LWDEBUG("LWLockAcquire", lock, "immediately acquired lock");
	break;				/* got the lock 获取成功则返回 */
}
```
3. 失败则进入等待队列
```c
/* add to the queue */
LWLockQueueSelf(lock, mode);
```
4. 再次尝试获取，成功则出队列，并返回
```c
mustwait = LWLockAttemptLock(lock, mode);
/* ok, grabbed the lock the second time round, need to undoqueueing */
if (!mustwait)
{
	LOG_LWDEBUG("LWLockAcquire", lock, "acquired, undoing queue");
	LWLockDequeueSelf(lock);//从队列中出来，获取成功，直接返回
	break;
}
```
5. 失败则报告等待事件开始，并持续等待被唤醒，然后报告等待事件结束
```c
LWLockReportWaitStart(lock);
if (TRACE_POSTGRESQL_LWLOCK_WAIT_START_ENABLED())
	TRACE_POSTGRESQL_LWLOCK_WAIT_START(T_NAME(lock), mode);
for (;;)
{
	PGSemaphoreLock(proc->sem);
	if (proc->lwWaiting == LW_WS_NOT_WAITING)
		break;
	extraWaits++;
}
/* Retrying, allow LWLockRelease to release waiters again. 使用原子操作获取更新锁的状态 */
pg_atomic_fetch_or_u32(&lock->state, LW_FLAG_RELEASE_OK);
if (TRACE_POSTGRESQL_LWLOCK_WAIT_DONE_ENABLED())
			TRACE_POSTGRESQL_LWLOCK_WAIT_DONE(T_NAME(lock), mode);
LWLockReportWaitEnd();
```
6. 然后持续再次尝试获取锁，回到1，直到获取成功
7. 获取成功之后将这个锁增加到held_lwlocks中去
```c
held_lwlocks[num_held_lwlocks].lock = lock;
held_lwlocks[num_held_lwlocks++].mode = mode;
```
### 2.2.2 释放锁
释放轻量级锁的步骤大致如下：
1. 将这个锁在held_lwlocks中定位，并将数组该位置之后的锁前移
```c
for (i = num_held_lwlocks; --i >= 0;)
		if (lock == held_lwlocks[i].lock)
			break;

	if (i < 0)
		elog(ERROR, "lock %s is not held", T_NAME(lock));

	mode = held_lwlocks[i].mode;

	num_held_lwlocks--;
	//释放之后数组前移
	for (; i < num_held_lwlocks; i++)
		held_lwlocks[i] = held_lwlocks[i + 1];

	PRINT_LWDEBUG("LWLockRelease", lock, mode);
```
2. 释放锁
```c
if (mode == LW_EXCLUSIVE)
    oldstate = pg_atomic_sub_fetch_u32(&lock->state, LW_VAL_EXCLUSIVE);
else
    oldstate = pg_atomic_sub_fetch_u32(&lock->state, LW_VAL_SHARED);
```

3. 获取修改前的状态，判断是否需要唤醒等待队列的进程
```c
//如果锁未被持有并且等待队列为空则不需要唤醒
if ((oldstate & (LW_FLAG_HAS_WAITERS | LW_FLAG_RELEASE_OK)) ==
	(LW_FLAG_HAS_WAITERS | LW_FLAG_RELEASE_OK) &&
	(oldstate & LW_LOCK_MASK) == 0)
	check_waiters = true;
else
	check_waiters = false;
```
4. 如果需要唤醒，则唤醒
```c
if (check_waiters)
{
	/* XXX: remove before commit? */
	LOG_LWDEBUG("LWLockRelease", lock, "releasing waiters");
	LWLockWakeup(lock);
}
```
### 2.2.3 Extension拓展轻量级锁
方法一是通过`RequestNamedLWLockTranche`函数和`GetNamedLWLockTranche`函数来实现。其中`RequestNamedLWLockTranche`函数负责注册Tranche的名字以及自己所需要的轻量级锁的数量，`GetNamedLWLockTranche`负责根据Tranche Name来获得对应的锁。每个tranche都有自己唯一的id，全局唯一，tranche id和tranche name一一对应。
```c
void
RequestNamedLWLockTranche(const char *tranche_name, int num_lwlocks)
{
	NamedLWLockTrancheRequest *request;
	//先判断申请是否合法
	if (!process_shmem_requests_in_progress)
		elog(FATAL, "cannot request additional LWLocks outside shmem_request_hook");
	//如果是第一次拓展，初始16
	if (NamedLWLockTrancheRequestArray == NULL)
	{
		NamedLWLockTrancheRequestsAllocated = 16;
		NamedLWLockTrancheRequestArray = (NamedLWLockTrancheRequest *)
			MemoryContextAlloc(TopMemoryContext,
							   NamedLWLockTrancheRequestsAllocated
							   * sizeof(NamedLWLockTrancheRequest));
	}
	//如果上一次的拓展数量再次超出，则增长到当前数量的下一个2的幂次
	if (NamedLWLockTrancheRequests >= NamedLWLockTrancheRequestsAllocated)
	{
		int			i = pg_nextpower2_32(NamedLWLockTrancheRequests + 1);
		// 再使用repalloc进行扩容
		NamedLWLockTrancheRequestArray = (NamedLWLockTrancheRequest *)
			repalloc(NamedLWLockTrancheRequestArray,
					 i * sizeof(NamedLWLockTrancheRequest));
		NamedLWLockTrancheRequestsAllocated = i;
	}
	//再将新申请的tranche组加入进去
	request = &NamedLWLockTrancheRequestArray[NamedLWLockTrancheRequests];
	Assert(strlen(tranche_name) + 1 <= NAMEDATALEN);
	strlcpy(request->tranche_name, tranche_name, NAMEDATALEN);
	request->num_lwlocks = num_lwlocks;
	NamedLWLockTrancheRequests++;
}


LWLockPadded *
GetNamedLWLockTranche(const char *tranche_name)
{
	int			lock_pos;
	int			i;

	/*
	 * Obtain the position of base address of LWLock belonging to requested
	 * tranche_name in MainLWLockArray.  LWLocks for named tranches are placed
	 * in MainLWLockArray after fixed locks.
	 */
	//偏移到NamedLWLockTrancheRequestArray开始在MainLWLockArray的位置
	lock_pos = NUM_FIXED_LWLOCKS;
	for (i = 0; i < NamedLWLockTrancheRequests; i++)
	{	
		if (strcmp(NamedLWLockTrancheRequestArray[i].tranche_name,
				   tranche_name) == 0)
			return &MainLWLockArray[lock_pos];//返回在MainLWLockArray中地址
		//不断跳过tranche_name不相符的lock组
		lock_pos += NamedLWLockTrancheRequestArray[i].num_lwlocks;
	}

	elog(ERROR, "requested tranche is not registered");

	/* just to keep compiler quiet */
	return NULL;
}
```

方法二是通过`LWLockNewTrancheId`函数获取新的Tranche ID，然后将Tranche ID和Tranche Name通过`LWLockRegisterTranche`函数建立联系，然后再由`LWLockInitialize`函数进行初始化。
```c
/*
 * Allocate a new tranche ID.
 */
int
LWLockNewTrancheId(void)
{
	int			result;
	int		   *LWLockCounter;

	LWLockCounter = (int *) ((char *) MainLWLockArray - sizeof(int));
	SpinLockAcquire(ShmemLock);
	result = (*LWLockCounter)++;//获取新的tranche ID
	SpinLockRelease(ShmemLock);

	return result;
}


void
LWLockRegisterTranche(int tranche_id, const char *tranche_name)
{
	//新申请的tranche id应该大于初始的id最大
	/* This should only be called for user-defined tranches. */
	if (tranche_id < LWTRANCHE_FIRST_USER_DEFINED)
		return;
	//将其转换为索引idx
	/* Convert to array index. */
	tranche_id -= LWTRANCHE_FIRST_USER_DEFINED;
	//如果目前的空间不足于存储一个新的tranche id的lock，则扩容
	/* If necessary, create or enlarge array. */
	if (tranche_id >= LWLockTrancheNamesAllocated)
	{
		int			newalloc;
		//扩容到新的大小
		newalloc = pg_nextpower2_32(Max(8, tranche_id + 1));
		
		if (LWLockTrancheNames == NULL)
			LWLockTrancheNames = (const char **)
				MemoryContextAllocZero(TopMemoryContext,
									   newalloc * sizeof(char *));
		else
		{
			LWLockTrancheNames = (const char **)
				repalloc(LWLockTrancheNames, newalloc * sizeof(char *));
			memset(LWLockTrancheNames + LWLockTrancheNamesAllocated,
				   0,
				   (newalloc - LWLockTrancheNamesAllocated) * sizeof(char *));
		}
		LWLockTrancheNamesAllocated = newalloc;
	}
	//存储tranche_name
	LWLockTrancheNames[tranche_id] = tranche_name;
}
```

## 2.3 常规锁 Regular Lock
