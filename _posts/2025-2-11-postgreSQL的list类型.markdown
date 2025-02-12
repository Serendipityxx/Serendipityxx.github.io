---
layout:  post
title:   "postgreSQL的list类型"
date:   2025-2-11 16:28:00
author:  'Xiangxiao'
header-img: 'img/post-bg-2015.jpg'
catalog:   true
tags:
- postgreSQL
- 链表
---


在postgreSQL中有封装好一个list类型，也就是单链表类型，本文就是对这个类型的总结。

## list的定义

list的定义是在pg_list.h文件中声明的，包括一个结构体和一个联合体。首先是list这个主要的结构体的定义，如下所示：

```c
typedef struct List
{
	NodeTag	    type;			/* T_List, T_IntList, or T_OidList */
	int         length;			/* number of elements currently present */
	int         max_length;		/* allocated length of elements[] */
	ListCell    *elements;		/* re-allocatable array of cells */
	/* We may allocate some cells along with the List header: */
	ListCell    initial_elements[FLEXIBLE_ARRAY_MEMBER];
	/* If elements == initial_elements, it's not a separate allocation */
} List;
```
- type 是这个链表的类型，大致分为三类，T_List、T_IntList、T_OidList
- length 是这个链表当前的长度
- max_length 允许链表的最大长度
- elements 是链表的元素
- initial_elements 

在上面的结构体里面提到了一个ListCell联合体：

```c
typedef union ListCell
{
	void	   *ptr_value;//指针
	int	    int_value;//int值
	Oid	    id_value;//oid值
} ListCell;
```
在list里面可以存储不同的类型的数据，但是同一个链表建议存储相同类型的数据。

当然在pg12版本以下这个list定义有所不同，这里给出定义，不单独分析，详情可以看[pg12代码](https://github.com/postgres/postgres/blob/REL_12_STABLE)

```c
typedef struct List
{
	NodeTag     type;			/* T_List, T_IntList, or T_OidList */
	int         length;
	ListCell    *head;
	ListCell    *tail;
} List;

struct ListCell
{
	union
	{
            void    *ptr_value;
            int     int_value;
            Oid     oid_value;
	}data;
	ListCell   *next;
};
```

## list的一些常用接口
### 初始化链表
可以使用这些接口初始化list，还有其他初始化方法：
- list_make1
- list_make1_int
- list_make1_oid

```c
/*
 * Convenience macros for building fixed-length lists用于构建定长列表的便利宏
 */
#define list_make_ptr_cell(v)	((ListCell) {.ptr_value = (v)})
#define list_make_int_cell(v)	((ListCell) {.int_value = (v)})
#define list_make_oid_cell(v)	((ListCell) {.oid_value = (v)})

#define list_make1(x1) \
	list_make1_impl(T_List, list_make_ptr_cell(x1))
#define list_make2(x1,x2) \
	list_make2_impl(T_List, list_make_ptr_cell(x1), list_make_ptr_cell(x2))
#define list_make3(x1,x2,x3) \
	list_make3_impl(T_List, list_make_ptr_cell(x1), list_make_ptr_cell(x2), \
					list_make_ptr_cell(x3))
#define list_make4(x1,x2,x3,x4) \
	list_make4_impl(T_List, list_make_ptr_cell(x1), list_make_ptr_cell(x2), \
					list_make_ptr_cell(x3), list_make_ptr_cell(x4))
#define list_make5(x1,x2,x3,x4,x5) \
	list_make5_impl(T_List, list_make_ptr_cell(x1), list_make_ptr_cell(x2), \
					list_make_ptr_cell(x3), list_make_ptr_cell(x4), \
					list_make_ptr_cell(x5))
```
### 获取链表的元素
- list_nth_cell(const List *list, int n) 获取链表的第n个元素，从0开始数
- list_last_cell(const List *list) 获取链表的最后一个元素
- list_cell_number(const List *l, const ListCell *c)
当然还有一些获取元素的值的快捷接口，根据需要可以任意组合
```c
#define lfirst(lc)		((lc)->ptr_value)
#define lfirst_int(lc)		((lc)->int_value)
#define lfirst_oid(lc)		((lc)->oid_value)
#define lfirst_node(type,lc) castNode(type, lfirst(lc))
#define linitial(l)		lfirst(list_nth_cell(l, 0))
#define linitial_int(l)		lfirst_int(list_nth_cell(l, 0))
#define linitial_oid(l)		lfirst_oid(list_nth_cell(l, 0))
#define linitial_node(type,l) castNode(type, linitial(l))
#define lsecond(l)		lfirst(list_nth_cell(l, 1))
#define lsecond_int(l)		lfirst_int(list_nth_cell(l, 1))
#define lsecond_oid(l)		lfirst_oid(list_nth_cell(l, 1))
#define lsecond_node(type,l) castNode(type, lsecond(l))
#define lthird(l)		lfirst(list_nth_cell(l, 2))
#define lthird_int(l)		lfirst_int(list_nth_cell(l, 2))
#define lthird_oid(l)		lfirst_oid(list_nth_cell(l, 2))
#define lthird_node(type,l)	castNode(type, lthird(l))
#define lfourth(l)		lfirst(list_nth_cell(l, 3))
#define lfourth_int(l)		lfirst_int(list_nth_cell(l, 3))
#define lfourth_oid(l)		lfirst_oid(list_nth_cell(l, 3))
#define lfourth_node(type,l) castNode(type, lfourth(l))
#define llast(l)		lfirst(list_last_cell(l))
#define llast_int(l)		lfirst_int(list_last_cell(l))
#define llast_oid(l)		lfirst_oid(list_last_cell(l))
#define llast_node(type,l)	castNode(type, llast(l))
```
**castNode** 将listcell转换成对应的数据类型，用于void指针转化
```c
#define castNode(_type_, nodeptr) ((_type_ *) (nodeptr))
```
### 循环遍历链表

#### foreach(cell, lst)
从0开始遍历链表，但是需要注意的是在遍历过程中不能随便删除元素，只能通过foreach_delete_current(lst, cell)去删除函数。

```c
#define foreach(cell, lst)	\
	for (ForEachState cell##__state = {(lst), 0}; \
		 (cell##__state.l != NIL && \
		  cell##__state.i < cell##__state.l->length) ? \
		 (cell = &cell##__state.l->elements[cell##__state.i], true) : \
		 (cell = NULL, false); \
		 cell##__state.i++)
```

```c
#define foreach_delete_current(lst, cell)	\
	(cell##__state.i--, \
	 (List *) (cell##__state.l = list_delete_cell(lst, cell)))
```

还得注意的是ForEachState结构体
```c
typedef struct ForEachState
{
	const List *l;				/* list we're looping through */
	int			i;				/* current element index */
} ForEachState; 
```
还有其他类似的结构体ForBothState、ForBothCellState、ForThreeState、ForFourState、ForFiveState。根据需要可以同时遍历多个链表，最大能同时遍历5个list。与其对应的遍历函数是forboth(cell1, list1, cell2, list2)、for_both_cell(cell1, list1, initcell1, cell2, list2, initcell2)，etc.

#### for_each_from(cell, lst, N)
除了从头开始遍历之外，也支持从某个位置开始遍历，使用的是for_each_from(cell, lst, N)

```c
#define for_each_from(cell, lst, N)	\
	for (ForEachState cell##__state = for_each_from_setup(lst, N); \
		 (cell##__state.l != NIL && \
		  cell##__state.i < cell##__state.l->length) ? \
		 (cell = &cell##__state.l->elements[cell##__state.i], true) : \
		 (cell = NULL, false); \
		 cell##__state.i++)
```

### 链表的插入

#### 尾插法
- lappend(List *list, void *datum) 从尾部插入一个元素，他的值是一个void指针
- lappend_int(List *list, int datum) 值是int
- lappend_oid(List *list, Oid datum) 值是oid

这里只给出lappend的定义，其他两个类似。
逻辑是在插入之前先检查list是否是null，是的话则新建链表，否则的话就调用new_tail_cell在最后新增一个元素，并给这个元素设置值。


```c
List *
lappend(List *list, void *datum)
{
	Assert(IsPointerList(list));

	if (list == NIL)
		list = new_list(T_List, 1);
	else
		new_tail_cell(list);

	llast(list) = datum;
	check_list_invariants(list);
	return list;
}
```
如果在新增的时候当前的length大于max_length，则调用enlarge_list拓展list

```c
static void
new_tail_cell(List *list)
{
	/* Enlarge array if necessary */
	if (list->length >= list->max_length)
		enlarge_list(list, list->length + 1);
	list->length++;
}
```

#### 唯一性插入
- list_append_unique(List *list, void *datum) 当插入的值已经存在，则直接返回，不进行操作， 否则插入新元素。
- list_append_unique_ptr(List *list, void *datum)
- list_append_unique_int(List *list, int datum)
- list_append_unique_oid(List *list, Oid datum)

```c
List *
list_append_unique(List *list, void *datum)
{
	if (list_member(list, datum))
		return list;
	else
		return lappend(list, datum);
}
```
主要是通过list_member进行判断，判断当前插入的值是否存在。

#### 头插法
- lcons(void *datum, List *list)
- lcons_int(int datum, List *list)
- lcons_oid(Oid datum, List *list)

这里同尾插法一样只给出一个示例，lcons.
同样在插入之前先检查list是否为null，然后再调用new_head_cell在头部创建元素，在对元素进行设置。

```c
List *
lcons(void *datum, List *list)
{
	Assert(IsPointerList(list));

	if (list == NIL)
		list = new_list(T_List, 1);
	else
		new_head_cell(list);

	linitial(list) = datum;
	check_list_invariants(list);
	return list;
}
```
new_head_cell也是在list存满的情况调用enlarge_list去在尾部新建一个元素，然后再调用memmove函数进行移动，这是一个c库函数，第一个参数是要存储的位置，第二个参数是要复制的位置，第三个参数是复制的大小。
```c
static void
new_head_cell(List *list)
{
	/* Enlarge array if necessary */
	if (list->length >= list->max_length)
		enlarge_list(list, list->length + 1);
	/* Now shove the existing data over */
	memmove(&list->elements[1], &list->elements[0],
			list->length * sizeof(ListCell));
	list->length++;
}
```

### 删除某个元素
- list_delete_nth_cell(List *list, int n) 删除某个位置的元素
- list_delete_cell(List *list, ListCell *cell) 删除某个元素，将元素转换为索引进行删除。
- list_delete(List *list, void *datum)  删除某个值，只删除第一个查到的值
    - list_delete_ptr(List *list, void *datum)
    - list_delete_int(List *list, int datum)
    - list_delete_oid(List *list, Oid datum)
- list_delete_first(List *list) 删除第一个元素
- list_delete_last(List *list) 删除最后一个元素，当长度大于1时，只需要将length减1即可，不需要数据的移动，速度很快。
- list_delete_first_n(List *list, int n) 删除开头N个元素，将数据元素往前移动N个位置，这样就相当于删除了。

### 链表的拼接
- list_concat(List *list1, const List *list2) 拼接两个list，将list2拼接在list1后面，并返回list1
- list_concat_copy(const List *list1, const List *list2) 重新新建一个list，将list1和list2放进去后返回
- list_truncate(List *list, int new_size) 截断list到指定长度，对于截断长度大于当前长度，不进行扩容。
- list_union(const List *list1, const List *list2) 两个list取并集，也是新建了一个list作为返回
    - list_union_ptr(const List *list1, const List *list2)
    - list_union_int(const List *list1, const List *list2)
    - list_union_oid(const List *list1, const List *list2)
- list_intersection(const List *list1, const List *list2)取两个list的交集
    - list_intersection_int(const List *list1, const List *list2)
- list_difference(const List *list1, const List *list2) 取两个list的差集
    - list_difference_ptr(const List *list1, const List *list2)
    - list_difference_int(const List *list1, const List *list2)
    - list_difference_oid(const List *list1, const List *list2)

### 链表的拷贝
- list_copy(const List *oldlist) 浅拷贝，对于ptr_value类型的值，只是拷贝了指针值，新旧list都指向相同的对象， 修改会互相影响。

```c
List *
list_copy(const List *oldlist)
{
	List	   *newlist;

	if (oldlist == NIL)
		return NIL;

	newlist = new_list(oldlist->type, oldlist->length);
	memcpy(newlist->elements, oldlist->elements,
		   newlist->length * sizeof(ListCell));

	check_list_invariants(newlist);
	return newlist;
}
```
- list_copy_tail(const List *oldlist, int nskip) 跳过开头N个元素再拷贝
- list_copy_deep(const List *oldlist) 深拷贝，将元素的值也拷贝一份，这样两个list将完全没有关系，修改互不影响。

```c
List *
list_copy_deep(const List *oldlist)
{
	List	   *newlist;

	if (oldlist == NIL)
		return NIL;

	/* This is only sensible for pointer Lists */
	Assert(IsA(oldlist, List));

	newlist = new_list(oldlist->type, oldlist->length);
	for (int i = 0; i < newlist->length; i++)
		lfirst(&newlist->elements[i]) =
			copyObjectImpl(lfirst(&oldlist->elements[i]));

	check_list_invariants(newlist);
	return newlist;
}
```

### 链表的排序
- list_sort(List *list, list_sort_comparator cmp) 使用标准的qsort函数，传递不同的cmp函数进行排序。

```c
void
list_sort(List *list, list_sort_comparator cmp)
{
	typedef int (*qsort_comparator) (const void *a, const void *b);
	int			len;

	check_list_invariants(list);

	/* Nothing to do if there's less than two elements */
	len = list_length(list);
	if (len > 1)
		qsort(list->elements, len, sizeof(ListCell), (qsort_comparator) cmp);
}
```

官方有给出两个比较函数
```c
int
list_int_cmp(const ListCell *p1, const ListCell *p2)
{
	int			v1 = lfirst_int(p1);
	int			v2 = lfirst_int(p2);

	if (v1 < v2)
		return -1;
	if (v1 > v2)
		return 1;
	return 0;
}

/*
 * list_sort comparator for sorting a list into ascending OID order.
 */
int
list_oid_cmp(const ListCell *p1, const ListCell *p2)
{
	Oid			v1 = lfirst_oid(p1);
	Oid			v2 = lfirst_oid(p2);

	if (v1 < v2)
		return -1;
	if (v1 > v2)
		return 1;
	return 0;
}
```

### 链表的释放
- list_free_private(List *list, bool deep) 释放列表中的所有存储空间，以及可选的所指向的元素

```c
static void
list_free_private(List *list, bool deep)
{
	if (list == NIL)
		return;					/* nothing to do */

	check_list_invariants(list);

	if (deep)
	{
		for (int i = 0; i < list->length; i++)
			pfree(lfirst(&list->elements[i]));
	}
	if (list->elements != list->initial_elements)
		pfree(list->elements);
	pfree(list);
}
```
根据deep这个bool值的不同又给出了两个api，其实最终也是调用了list_free_private函数
```c
void
list_free(List *list)
{
	list_free_private(list, false);
}

void
list_free_deep(List *list)
{
	/*
	 * A "deep" free operation only makes sense on a list of pointers.
	 */
	Assert(IsPointerList(list));
	list_free_private(list, true);
}
```

## 参考

[postgresql之List](https://blog.csdn.net/happytree001/article/details/125382194)
[postgreSQL v15.8](https://www.postgresql.org/docs/15/release-15-8.html)
