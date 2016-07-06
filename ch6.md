# 第六章 内存管理

> 作者：[Allen B. Downey](http://greenteapress.com/wp/)

> 原文：[Chapter 6  Memory management](http://greenteapress.com/thinkos/html/thinkos007.html)

> 译者：[飞龙](https://github.com/)

> 协议：[CC BY-NC-SA 4.0](http://creativecommons.org/licenses/by-nc-sa/4.0/)

C提供了4中用于动态内存分配的函数：

+ `malloc`，它接受表示字节单位的大小的整数，返回指向新分配的、（至少）为指定大小的内存块的指针。如果不能满足要求，它会返回特殊的值为`NULL`的指针。
+ `calloc`，它和`malloc`一样，除了它会清空新分配的空间。也就是说，它会设置块中所有字节为0。
+ `free`，它接受指向之前分配的内存块的指针，并会释放它。也就是说，使这块空间可用于未来的分配。
+ `realloc`，它接受指向之前分配的内存块的指针，和一个新的大小。它使用新的大小