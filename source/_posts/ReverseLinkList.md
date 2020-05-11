---
title: ReverseLinkList
date: 2019-07-16 16:02:55
tags: [LinkList]
categories:
- 算法与数据结构
- LeetCode
comments: true
---

# Reverse Link List

## 题目
单向链表反向，输入为 1->2->3->4->5->NULL，输出为5->4->3->2->1->NULL 

## 分析
遍历一次，单次循环中将遍历的元素next指向上一个元素

* 关键点：需要3个指针，一个前向指针prev用于指向当前遍历元素的上一个元素；当前遍历元素指针；由于修改了当前元素的next指针，因此需要一个后向指针next，提前保存下一个元素地址
* 特殊情况：当链表元素只有头或者2个元素时

## C语言实现

```c
typedef struct _list_node {
	int x;
	struct _list_node *next;
} list_node;

static inline bool list_node_empty(struct _list_node *head)
{
	return (!head->next);
}

static inline void list_node_init(struct _list_node *head, int x)
{
	head->x = x;
	head->next = NULL;
}

static inline struct _list_node *list_node_tail(struct _list_node *head)
{
	struct _list_node *p = head;

	while (p->next != NULL) {
		p = p->next;
	}

	return p;
}

static inline void list_node_append(struct _list_node *new, struct _list_node *head)
{
	struct _list_node *tail;
	
	tail = list_node_tail(head);
	tail->next = new;
	new->next = NULL;
}

#define list_node_foreach(pos, head)		\
	for (pos = (head);				        \
		 pos != NULL;						\
		 pos = pos->next)					\

static inline struct _list_node *list_node_revert(struct _list_node *head)
{
	struct _list_node *p_prev=head, *p, *p_next;

	if (!head || !head->next)
		return head;
	
	p = head->next;
	p_prev->next = NULL;
	
	while (p != NULL) {
		p_next = p->next;
		p->next = p_prev;
		p_prev = p;
		p = p_next;
	}

	return p_prev;
}

int main(int argc, char *argv)
{
    list_node *p, *revert;
    list_node head, node1, node2, node3, node4;	
    
    list_node_init(&head, 1);
    list_node_init(&node1, 2);
	list_node_init(&node2, 3);
	list_node_init(&node3, 4);
	list_node_init(&node4, 5);
	
    list_node_append(&node1, &head);
	list_node_append(&node2, &head);
	list_node_append(&node3, &head);
	list_node_append(&node4, &head);

	list_node_foreach(p, &head) {
		log_debug("data:%d", p->x);
	}

    revert = list_node_revert(&head);

    list_node_foreach(p, revert) {
		log_debug("data:%d", p->x);
	}	
}
```

## Python实现

```python

class ListNode(object):
    def __init__(self, x):
        self.val = x
        self.next = None

def reverseList(head):
    cur, pre = head, None
    while cur:
        cur.next, pre, cur = pre, cur, cur.next
    return pre    

def getListTail(head):
    curr = head
    while (curr.next):
        curr = curr.next
    return curr

def appendList(item, head):
    tail = getListTail(head)
    tail.next, item.next = item, None

list_head = ListNode(1)
data1 = ListNode(2)
data2 = ListNode(3)
data3 = ListNode(4)
data4 = ListNode(5)
appendList(data1, list_head)
appendList(data2, list_head)
appendList(data3, list_head)
appendList(data4, list_head)

curr = list_head
while (curr):
    print("data:{}".format(curr.val))
    curr = curr.next

reverse = reverseList(list_head)
curr = reverse
while (curr):
    print("data:{}".format(curr.val))
    curr = curr.next
```

python代码可利用一个赋值语句对多个对象赋值特性，省略后向指针