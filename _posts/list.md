
[toc]


Linux内核中使用了大量的链表结构来组织数据，包括设备列表以及各种功能模块中的数据组织。代码实现在include/linux/list.h中。


## list_head

```c
struct list_head {
	struct list_head *next, *prev;
};
```
list_head本身没有任何数据内容，只有两个方向指针prev和next，而自定义的数据结点结构体只要包含list_head指针，就可以使用 list提供的所有接口函数。

```c
struct list_node {
    struct list_node *next;
    int    data;
};
```
```c
/*声明一个链表头，它的next、prev指针都初始化为指向自己*/
#define LIST_HEAD_INIT(name) { &(name), &(name) }

#define LIST_HEAD(name) struct list_head name = LIST_HEAD_INIT(name)

/*初始化已经定义过的链表*/
static inline void INIT_LIST_HEAD(struct list_head *list)
{
	list->next = list;
	list->prev = list;
}
```

## list_add
```c
static inline void __list_add(struct list_head *new_entry,
       struct list_head *prev,
       struct list_head *next)
{
	next->prev = new_entry;
	new_entry->next = next;
	new_entry->prev = prev;
	prev->next = new_entry;
}

static inline void list_add(struct list_head *new_entry, struct list_head *head)
{
	__list_add(new_entry, head, head->next);
}

static inline void list_add_tail(struct list_head *new_entry, struct list_head *head)
{
	__list_add(new_entry, head->prev, head);
}
```


## list_del
```c
static inline void __list_del(struct list_head * prev, struct list_head * next)
{
	next->prev = prev;
	prev->next = next;
}

#define LIST_POISON1  ((void *) 0x00100100)
#define LIST_POISON2  ((void *) 0x00200200)

static inline void list_del(struct list_head *entry)
{
	__list_del(entry->prev, entry->next);
	entry->next = (struct list_head *)LIST_POISON1;
	entry->prev = (struct list_head *)LIST_POISON2;
}
```


## list_replace

新节点替换老节点

```c
static inline void list_replace(struct list_head *old,
				struct list_head *new_entry)
{
	new_entry->next = old->next;
	new_entry->next->prev = new_entry;
	new_entry->prev = old->prev;
	new_entry->prev->next = new_entry;
}
```

## list_move

移动节点到另一个链表

```c
static inline void list_move(struct list_head *list, struct list_head *head)
{
	__list_del_entry(list);
	list_add(list, head);
}


static inline void list_move_tail(struct list_head *list,
				  struct list_head *head)
{
	__list_del_entry(list);
	list_add_tail(list, head);
}
```


##list_empty
判断这个链表的头指针 head的next是否指向它自己，如果是，则说明为空。

```c
static inline int list_empty(const struct list_head *head)
{
	return head->next == head;
}
```

## list_entry

获取链表中某个节点的地址
```c
#define list_entry(ptr, type, member) \
	container_of(ptr, type, member)

#define offset_of(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)

#define container_of(ptr, type, member) ({\
	const __typeof__( ((type *)0)->member ) *__mptr = (ptr); \
	(type *)( (char *)__mptr - offset_of(type,member) ); })
```

获取链表中第一个节点的地址
```c

#define list_first_entry(ptr, type, member) \
	list_entry((ptr)->next, type, member)

#define list_last_entry(ptr, type, member) \
	list_entry((ptr)->prev, type, member)
```



## list_for_each

遍历整个链表，pos只是一个临时的变量。
```c
#define list_for_each(pos, head) \
	for (pos = (head)->next; pos != (head); pos = pos->next)

#define list_for_each_prev(pos, head) \
	for (pos = (head)->prev; pos != (head); pos = pos->prev)
```

## list_for_each_safe

遍历链表，如果在遍历的过程中涉及到节点的删除操作，则需要使用这个函数，这个函数中有个中间节点 n 作为临时存储区，更加安全。
```c
#define list_for_each_safe(pos, n, head) \
	for (pos = (head)->next, n = pos->next; pos != (head); \
		pos = n, n = pos->next)

#define list_for_each_prev_safe(pos, n, head) \
	for (pos = (head)->prev, n = pos->prev; \
	     pos != (head); \
	     pos = n, n = pos->prev)
```


## list_for_each_entry

节点结构遍历

```c
#define list_for_each_entry(pos, head, member) \
	for (pos = list_first_entry(head, __typeof__(*pos), member); \
	     &pos->member != (head);					\
	     pos = list_next_entry(pos, member))

#define list_for_each_entry_reverse(pos, head, member) \
	for (pos = list_last_entry(head, __typeof__(*pos), member); \
	     &pos->member != (head); \
	     pos = list_prev_entry(pos, member))

#define list_for_each_entry_safe(pos, n, head, member) \
	for (pos = list_first_entry(head, __typeof__(*pos), member), \
			n = list_next_entry(pos, member); \
	     &pos->member != (head); \
	     pos = n, n = list_next_entry(n, member))
```