---
title: container_of
categories: 备忘录
tags:
  - container_of
  - linux
  - 链表
---
linux 内核链表的实现有段代码比较有意思，这里记录一下。
# 背景
当在阅读遍历链表这个宏定义时，可能会对container_of有些许疑惑。
{% codeblock lang:c %}
/**    
 * list_for_each_entry  -   iterate over list of given type
 * @pos:    the type * to use as a loop cursor.
 * @head:   the head for your list.
 * @member: the name of the list_head within the struct.
 */    
#define list_for_each_entry(pos, head, member)              \
    for (pos = list_first_entry(head, typeof(*pos), member);    \
         &pos->member != (head);                    \
         pos = list_next_entry(pos, member))

		 
/**
 * list_entry - get the struct for this entry
 * @ptr:    the &struct list_head pointer.
 * @type:   the type of the struct this is embedded in.
 * @member: the name of the list_head within the struct.
 */
#define list_entry(ptr, type, member) \
    container_of(ptr, type, member)

/**
 * list_first_entry - get the first element from a list
 * @ptr:    the list head to take the element from.
 * @type:   the type of the struct this is embedded in.
 * @member: the name of the list_head within the struct.
 *
 * Note, that list is expected to be not empty.
 */
#define list_first_entry(ptr, type, member) \
    list_entry((ptr)->next, type, member)


#ifndef container_of
#define container_of(ptr, type, member) \
    (type *)((char *)(ptr) - (char *) &((type *)0)->member)
#endif
{% endcodeblock %}
# 解析
{% codeblock lang:c %}
#define container_of(ptr, type, member) \
    (type *)((char *)(ptr) - (char *) &((type *)0)->member)
#endif 
{% endcodeblock %}

如上定义，为了更直观，定义如下数据结构：
{% codeblock lang:c %}
struct list_head 
{
	struct list_head *next;
	struct list_head *prev; 
};

struct type
{
	int a;
	int b;
	struct list_head list;
	int c;
};
{% endcodeblock %}
那么对于container_of宏定义，三个参数分别对应：
* ptr: list的内存地址
* type: list所在数据结构的类型
* member： 是指list_head定义变量的名字，那么在这里就叫list

这个宏定义的功能：通过member的地址和member的名字获取member所在数据结构的首地址。
首先： (char *) &((type *)0)->member获取member在type类型中的偏移量
然后： 用ptr减去member在type类型数据结构中的偏移量，那么就得到了member所在type变量的首地址。

最后举例：
{% codeblock lang:c %}
#include<stdio.h>

struct A
{
   int a;
   int b;
   int c;
};

int main(void)
{
    struct A x;
    struct A *y;
    x.a = 1024;
    printf("output1: %0x\n", &(((struct A*)0)->b)); 
    y = (struct A*) (((char*)(&x.b)) - ((char*)&(((struct A*)0)->b)));
    printf("output2: %d\n", y->a); 
    printf("output3:%0x\n", &x);
    printf("output4:%0x\n", y);
    return 0;
}

运行结果为：
output1: 4
output2: 1024
output3:5b22a1e0
output4:5b22a1e0
{% endcodeblock %}