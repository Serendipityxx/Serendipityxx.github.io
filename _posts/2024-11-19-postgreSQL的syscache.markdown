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

# PostgreSQL的系统缓存
在pg的内核中，针对于如何增加一个系统缓存，有详细的介绍，具体在src/backend/utils/cache/syscache.c中：

1、在include/utils/syscache.h中新增一个新的cache放到SysCacheIdentifier中，但是要注意的是按照字符大小顺序进行排列。

2、然后在syscache.c中的cacheinfo[]数组中增加你的实体，也是按照字符大小顺序进行排列，新增的信息就是一个cachedesc结构体，他的成员如下所示：

```c
struct cachedesc
{
	Oid	reloid;			/* OID of the relation being cached */
	Oid	indoid;			/* OID of index relation for this cache */
	int	nkeys;			/* # of keys needed for cache lookup */
	int	key[4];			/* attribute numbers of key attrs */
	int	nbuckets;		/* number of hash buckets for this cache */
};
```
包括relation oid、index oid、number of keys、key attribute numbers和initial number of hash buckets

示例如下：
```c
{AggregateRelationId,		/* AGGFNOID */
		AggregateFnoidIndexId,
		1,
		{
			Anum_pg_aggregate_aggfnoid,
			0,
			0,
			0
		},
		16
	},
```

散列桶的数量必须是2的幂次。

3、每个syscache必须有一个唯一的索引，如果没有则需要在catalog/pg_*.h中使用DECLARE_UNIQUE_INDEX增加一个唯一索引，新增索引需要更新catversion.h，如果只是简单的新增syscache的话只需要重新编译。

4、最后，在关系调用heap_insert（）或heap_update（）的任何地方，请使用CatalogTupleInsert（）或CatalogTupleUpdate()，它们也会更新索引。heap_*调用不会这样做。


## 详细介绍
### 1 SysCacheIdentifier

SysCacheIdentifier是一个枚举类型，这是一个syscache的标识，他的顺序是按照字母的大小顺序进行组织的，如果需要增加，也需要按照字母顺序进行增加，如果增加位置为最后一个，还需要修改一下SysCacheSize宏定义。

### 2 cachedesc cacheinfo[]
首先cachedesc是一个结构体，定义在前面已经给出来了，主要就是存储syscache的信息，包括被缓存表的oid，这个缓存扫描的索引oid，这个缓存查找的key的数量，这几个key的属性，这个缓存的哈希槽的数量。

cacheinfo是这个结构体的数组，存储了SysCacheIdentifier所给出来的缓存定义。

### 3 CatCache

```c
typedef struct catcache
{
	int			id;				/* cache identifier --- see syscache.h */
	int			cc_nbuckets;	/* # of hash buckets in this cache */
	TupleDesc	cc_tupdesc;		/* tuple descriptor (copied from reldesc) */
	dlist_head *cc_bucket;		/* hash buckets */
	CCHashFN	cc_hashfunc[CATCACHE_MAXKEYS];	/* hash function for each key */
	CCFastEqualFN cc_fastequal[CATCACHE_MAXKEYS];	/* fast equal function for
													 * each key */
	int			cc_keyno[CATCACHE_MAXKEYS]; /* AttrNumber of each key */
	dlist_head	cc_lists;		/* list of CatCList structs */
	int			cc_ntup;		/* # of tuples currently in this cache */
	int			cc_nkeys;		/* # of keys (1..CATCACHE_MAXKEYS) */
	const char *cc_relname;		/* name of relation the tuples come from */
	Oid			cc_reloid;		/* OID of relation the tuples come from */
	Oid			cc_indexoid;	/* OID of index matching cache keys */
	bool		cc_relisshared; /* is relation shared across databases? */
	slist_node	cc_next;		/* list link */
	ScanKeyData cc_skey[CATCACHE_MAXKEYS];	/* precomputed key info for heap
											 * scans */

	/*
	 * Keep these at the end, so that compiling catcache.c with CATCACHE_STATS
	 * doesn't break ABI for other modules
	 */
#ifdef CATCACHE_STATS
	long		cc_searches;	/* total # searches against this cache */
	long		cc_hits;		/* # of matches against existing entry */
	long		cc_neg_hits;	/* # of matches against negative entry */
	long		cc_newloads;	/* # of successful loads of new entry */

	/*
	 * cc_searches - (cc_hits + cc_neg_hits + cc_newloads) is number of failed
	 * searches, each of which will result in loading a negative entry
	 */
	long		cc_invals;		/* # of entries invalidated from cache */
	long		cc_lsearches;	/* total # list-searches */
	long		cc_lhits;		/* # of matches against existing lists */
#endif
} CatCache;
```




