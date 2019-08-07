---
layout:     post
title:      linux链表 2019
subtitle:  	链表
date:       2019-07-30
author:     Secssion
header-img:	img/bridge_back.jpg
catalog: true
tags:
    - linux
---


## 链表基本结构和一般链表基本结构比较

一般的数据链表结构定义如下，链表的基本增删改减功能都依赖`struct type`, 对于种类型的链表，都得实现一遍功能。

````
	struct type{
		int number;
		struct type* next;
	}
````

`Linux`下的链表结构只带指针域，基本的增删改减功能都通过`struct list_head`来完成，而不涉及链表结构数据域，所以重用性较高。

linux下链表结构：

````
    struct list_head {
        struct list_head *next, *prev;
    };
	struct type {
		int number;
		struct list_head *hnode;
	}
````

![img](/img/post-in/list.PNG)


### 链表初始化

`LIST_HEAD(name)` 定义声明 `name` 链表,  链表的 `next`,  `pre` 都指向自身。
`INIT_LIST_HEAD()` 初始一个链表， `next`, `pre`域都执行自身。
````
#define LIST_HEAD_INIT(name) { &(name), &(name) }
#define LIST_HEAD(name) \
	struct list_head name = LIST_HEAD_INIT(name)
	
static __inline void INIT_LIST_HEAD(struct list_head *list)
{
	list->next = list;
	list->prev = list;
}
````
###  链表获取 entry

`offsetof(TYPE, MEMER)` 求出MEMER在TYPE类型中的偏移量。 
`container_of(ptr, type, member)`求ptr所在结构体的首地址。

````
	#ifndef offsetof
    #define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)  
    #endif
    
    #ifndef container_of
    #define container_of(ptr, type, member) \
    ((type *)((char *)(ptr) - offsetof(type, member)))
    #endif
````

### 链表遍历

`pos = list_entry((head)->next, typeof(*pos), member)` 链表头部的第一个节点入口地址 ,  `pos = list_entry(pos->member.next, typeof(*pos), member)` 依次遍历入口地址。
`list_for_each_entry_safe(pos, n, head, member)` 参数 n用于 提前保存next指针，使删除节点安全。 

````
#define list_for_each_entry(pos, head, member)				\
	for (pos = list_entry((head)->next, typeof(*pos), member);	\
	     &pos->member != (head); 	\
	     pos = list_entry(pos->member.next, typeof(*pos), member))
	     
#define list_entry(ptr, type, member) \
	container_of(ptr, type, member)
	
#define list_for_each_entry_safe(pos, n, head, member)			\
	for (pos = list_entry((head)->next, typeof(*pos), member),	\
		n = list_entry(pos->member.next, typeof(*pos), member);	\
	     &pos->member != (head); 					\
	     pos = n, n = list_entry(n->member.next, typeof(*n), member))	
````

##  链表插入

`list_add()`将 `new` 节点加入`head`之后。

```
static __inline void list_add(struct list_head *node, struct list_head *head)
{
	__list_add(node, head, head->next);
}

static inline void __list_add(struct list_head *new,
			      struct list_head *prev,
			      struct list_head *next)
{
	next->prev = new;
	new->next = next;
	new->prev = prev;
	prev->next = new;
}
```
### 链表删除

`list_del(struct list_head *entry)` 删除 `entry`节点, 并将 `next`, `prev`指针置空。
````
static __inline void list_del(struct list_head *entry)
{
	__list_del(entry->prev, entry->next);
	entry->next = (struct list_head *)LIST_POISON1;
	entry->prev = (struct list_head *)LIST_POISON2;
}

static __inline void __list_del(struct list_head * prev, struct list_head * next)
{
	next->prev = prev;
	prev->next = next;
}
````

### 示例代码

````
#include <time.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h> 

#include "list.h"

#define HASH_MASK		((1 << 8) - 1)

struct list_head user_slots[HASH_MASK + 1];

struct user {
	struct list_head hnode;
	char mac[18]; 
	time_t active_time; 
	time_t magic_time; 
};

static void hash_init() {
	int i =0;
	for(; i<HASH_MASK; i++) {
		INIT_LIST_HEAD(&user_slots[i]); 
	}
}


static unsigned int hash_crc(const void *p, int len) {
	int i;
	unsigned int hash = 0;
	const unsigned char *addr = (const unsigned char*)p;

	for (i = 0; i < len; i++)
		hash += hash * 33 + addr[i];
	return hash;
}

static void set_user(int add, struct user *user , struct list_head *macslot,  char* mac,  time_t now) {
	if (!add) {
		list_del(&user->hnode);
		free(user);
		return;
	}
	if (strcmp(user->mac, mac)) {
		strcpy(user->mac, mac);
	}
	user->active_time = now;
	list_add(&user->hnode, macslot);
}


struct user *user_insert(char *mac) {

	struct user *user = NULL, *pos = NULL;
	struct list_head *user_slot =  &user_slots[hash_crc(mac, strlen(mac)) & HASH_MASK];
	time_t now = time(NULL);
	list_for_each_entry_safe(user, pos, user_slot, hnode) {
		if (!strcmp(mac, user->mac)) {
			user->active_time = now;
			return user;
		}
	}
	user = (struct user*)malloc(sizeof(struct user));
	if (!user) {
		return NULL;
	}
	memset(user, 0, sizeof(struct user));
	set_user(1, user, user_slot,  mac,  now);
	return user;
}

static void  user_delete(char *mac) {
	struct user *user = NULL, *pos = NULL;
	time_t now = time(NULL);
	struct list_head *user_slot =  &user_slots[hash_crc(mac, strlen(mac)) & HASH_MASK];
	list_for_each_entry_safe(user, pos, user_slot, hnode) {
		if (!strcmp(mac, user->mac)) {
			set_user(0, user, user_slot,  mac, now);
			return;
		}
	}
}

struct user *user_lookup(const char *mac) {
	struct user *user = NULL, *pos = NULL;
	struct list_head *user_slot =  &user_slots[hash_crc(mac, strlen(mac)) & HASH_MASK];
	list_for_each_entry_safe(user,pos, user_slot, hnode) {
		if ( !strcmp(mac, user->mac)) {
			return user;
		}
	}
	return NULL;
}

static void list_print() {
	int i = 0;
	struct user *user = NULL, *pos = NULL;
	struct list_head *user_slot = NULL;
	
	for(i=0; i<HASH_MASK; i++) {
		user_slot = &user_slots[i];
		list_for_each_entry_safe(user, pos, user_slot, hnode) {
			printf("user->mac = %s, user->actiev_time = %lu\n", user->mac, user->active_time);
		}
	}
	printf("\n");
}


int main(){
	hash_init();

	user_insert("EE:EE:FF:FF:EF");
	user_insert("12:34:56:78:98");
	user_insert("FF:FF:FF:FF:FF");
	user_insert("E3:E3:E3:E3:E4");
	
	list_print();
	
	user_delete("E3:E3:E3:E3:E4");
	user_delete("E3:E3:E3:E3:E4");
	
	user_insert("EF:88:88:88:88");
	
	list_print();
	
	return 0;
}
````

- [源码](https://github.com/secssion/LinuxLearnNote/tree/master/kernel_list_test)

