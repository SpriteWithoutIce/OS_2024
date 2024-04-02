💡 这里定义了三种数据结构：list, tail queue, circle queue

## List

- LIST_HEAD(name, type)

```c
#define LIST_HEAD(name, type)                  \\
	struct name                                  \\
	{                                            \\
		struct type *lh_first; /* first element */ \\
	}
```

定义了一个结构体，list的名字叫name，第一个元素的类型是type

- LIST_ENTRY(type)

```c
#define LIST_ENTRY(type)                                          \\
	struct                                                          \\
	{                                                               \\
		struct type *le_next;	 /* next element */                     \\
		struct type **le_prev; /* address of previous next element */ \\
	}
```

这里定义的是list块里面用来连接的结构体部分

这个le_prev很有意思，它本身是指向上一个块的le_next指针的指针，解引用之后指着的就是上一个块的le_next，这个东西存在的意义是当要删除这个块的时候，使用*le_prev就可以改变前一个块的le_next，而不用遍历链表找到上一个块来改（*le_prev=le_next）

- LIST_ENPTY(head) 判空的

```c
#define LIST_EMPTY(head) ((head)->lh_first == NULL)
```

- LIST_FIRST(head) 返回head里面的第一个元素

```c
#define LIST_FIRST(head) ((head)->lh_first)
```

- LIST_FOREACH(var, head, field)

```c
#define LIST_FOREACH(var, head, field) \\
	for ((var) = LIST_FIRST((head)); (var); (var) = LIST_NEXT((var), field))
// 运用示例
struct node{
	int data;
	LIST_ENTRY(node) entry;
};
int main(){
	LIST_HEAD(list,node);
	struct node *node1=malloc(sizeof(struct node));
	node1->data=1;
	struct node *current_node;
	LIST_FOREACH(current_node,&head,entry){
		printf("%d\\n",current_node->data);
	}
}
```

- LIST_INIT(head) 清空列表

```c
#define LIST_INIT(head)        \\
	do                           \\
	{                            \\
		LIST_FIRST((head)) = NULL; \\
	} while (0)
```

- LIST_INSERT_AFTER(listelm, elm, field)

```c
#define LIST_INSERT_AFTER(listelm, elm, field)                               \\
	do                                                                         \\
	{                                                                          \\
		LIST_NEXT((elm), field) = LIST_NEXT((listelm), field);                   \\
		if (LIST_NEXT((listelm), field) != NULL)     \\
			LIST_NEXT((listelm), field)->field.le_prev = &LIST_NEXT((elm), field); \\
		LIST_NEXT((listelm), field) = (elm);                                     \\
		(elm)->field.le_prev = &LIST_NEXT((listelm), field);                     \\
	} while (0)
```

按照Hint的步骤写，LIST_NEXT就是取elm的field.le_next，先设置elm的le_next指针，然后如果不为空的话（前面还有块），把前面那个块的le_prev指针改一下，改成现在要插入的elm块。之后就是listelm的le_next改成elm，elm的le_prev需要指向listelm的le_next

- LIST_INSERT_BEFORE(listelm, elm, field)

```c
#define LIST_INSERT_BEFORE(listelm, elm, field)          \\
	do                                                     \\
	{                                                      \\
		(elm)->field.le_prev = (listelm)->field.le_prev;     \\
		LIST_NEXT((elm), field) = (listelm);                 \\
		*(listelm)->field.le_prev = (elm);                   \\
		(listelm)->field.le_prev = &LIST_NEXT((elm), field); \\
	} while (0)
```

和上面那个差不多，这个是吧elm插入到listelm的前面

- LIST_INSERT_HEAD(head, elm, field)，插到头节点前面

```c
#define LIST_INSERT_HEAD(head, elm, field)                          \\
	do                                                                \\
	{                                                                 \\
		if ((LIST_NEXT((elm), field) = LIST_FIRST((head))) != NULL)     \\
		/*前面拿到的是elm->field.le_next，后面拿到的是lh_first，事实上是一个指向第一个块的指针，
		现在是让这两个指针相等，也就是le_next指向现有的lh_first指向的那个块*/
			LIST_FIRST((head))->field.le_prev = &LIST_NEXT((elm), field); \\
		/*LIST_FIRST((head))取出未变的lh_first指针，它的le_prev应该变成插入的新块elm的le_next*/
		LIST_FIRST((head)) = (elm);                                     \\
		/*变更lh_first，仍然是指针，指向插入的这个块的指针*/
		(elm)->field.le_prev = &LIST_FIRST((head));                     \\
		/*第一个块的le_prev是要指向自己的*/
	} while (0)
```

- LIST_NEXT(elm, field)，拿到这个节点的le_next

```c
#define LIST_NEXT(elm, field) ((elm)->field.le_next)
```

- LIST_REMOVE(elm, field)，删掉elm

```c
#define LIST_REMOVE(elm, field)                                      \\
	do                                                                 \\
	{                                                                  \\
		if (LIST_NEXT((elm), field) != NULL)                             \\
			LIST_NEXT((elm), field)->field.le_prev = (elm)->field.le_prev; \\
		*(elm)->field.le_prev = LIST_NEXT((elm), field);                 \\
	} while (0)
```