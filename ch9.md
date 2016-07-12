# 第九章 线程

> 作者：[Allen B. Downey](http://greenteapress.com/wp/)

> 原文：[Chapter 9  Threads](http://greenteapress.com/thinkos/html/thinkos010.html)

> 译者：[飞龙](https://github.com/)

> 协议：[CC BY-NC-SA 4.0](http://creativecommons.org/licenses/by-nc-sa/4.0/)

当我在2.3节提到线程的时候，我说过线程就是一种进程。现在我会更仔细地解释它。

当你创建进程时，操作系统会创建一块新的地址空间，它包含`text`段、`static`段、和堆区。它也会创建新的“执行线程”，这包括程序计数器和其它硬件状态，以及运行时栈。

我们目前为止看到的进程都是“单线程”的，也就是说每个地址空间中只运行一个执行线程。在这一章中，你会了解“多线程”的进程，它在相同地址空间内拥有多个运行中的线程。

在单一进程中，所有线程都共享相同的`text`段，所以它们运行相同的代码。但是不同线程通常运行代码的不同部分。

而且，它们共享相同的`static`段，所以如果一个线程修改了某个全局变量，其它线程会看到改动。它们也共享堆区，所以线程可以共享动态分配的内存块。

但是每个线程都有它自己的栈。所以线程可以调用函数而不相互影响。通常，线程并不能访问其它线程的局部变量。

这一章的示例代码在本书的仓库中，在名为`counter`的目录中。有关代码下载的更多信息，请见第零章。

## 9.1 创建线程

C语言使用的所普遍的线程标准就是POSIX线程，简写为`pthread`。POSIX标准定义了线程模型和用于创建和控制线程的接口。多数UNIX的版本提供了POSIX的实现。

> 译者注：C11标准也提供了POSIX线程的实现。为了避免冲突，函数的前缀改为了`thrd`。

使用`pthread`就像使用大多数C标准库那样：

+ 你需要将头文件包含到程序开头。
+ 你需要编写调用`pthread`所定义函数的代码。
+ 当你编译程序时，需要链接`pthread`库。

例如，我包含了下列头文件：

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
```

前两个是标准库，第三个就是`pthread`。为了在`gcc`中和`pthread`一起编译，你可以在命令行中使用`-l`选项：

```
gcc -g -O2 -o array array.c -lpthread
```

这会编译名为`array.c`的源文件，带有调试信息和优化，并链接`pthread`库，之后生成名为`array`的可执行文件。

## 9.2 创建线程

用于创建线程的`pthread`函数叫做`pthread_create`。下面的函数展示了如何使用它：

```c
pthread_t make_thread(void *(*entry)(void *), Shared *shared)
{
  int n;
  pthread_t thread;

  n = pthread_create(&thread, NULL, entry, (void *)shared);
  if (n != 0) {
    perror("pthread_create failed");
    exit(-1);
  }
  return thread;
}
```

`make_thread`是一个包装，我编写它便于使`pthread_create`更加易用，并提供错误检查。

`pthread_create`的返回类型是`pthread_t`，你可以将其看做新线程的ID或者“句柄”。

如果`pthread_create`成功了，它会返回0，`make_pthread`也会返回新线程的句柄。如果出现了错误，`pthread_create`会返回错误代码，`make_thread`会打印错误消息并退出。

`pthread_create`的参数需要一些解释。从第二个开始，`Shared`是我定义的结构体，用于包含在两个线程之间共享的值。下面的`typedef`语句创建了这个新类型：

```c
typedef struct {
  int counter;
} Shared;
```

这里，唯一的共享变量是`counter`，`make_shared`为`Shared`结构体分配空间，并且初始化其内容：

```c
Shared *make_shared()
{
  int i;
  Shared *shared = check_malloc(sizeof (Shared));
  shared->counter = 0;
  return shared;
}
```

`entry`的参数声明为`void`指针，但在这个程序中我们知道它是一个指向`Shared`结构体的指针，所以我们可以对其做相应转换，之后将它传给执行实际工作的`child_code`。

作为一个简单的示例，`child_code`打印了共享计数器的值，并增加它。

```c
void child_code(Shared *shared)
{  
  printf("counter = %d\n", shared->counter);
  shared->counter++;
}
```

当`child_code`返回时，`entry`调用了`pthread_exit`，它可以用于将一个值传递给回收（join）当前线程的线程。这里，子线程没有什么要返回的，所以我们传递了`NULL`。

最后，下面是创建子线程的代码：

```c
int i;
pthread_t child[NUM_CHILDREN];

Shared *shared = make_shared(1000000);

for (i=0; i<NUM_CHILDREN; i++) {
    child[i] = make_thread(entry, shared);
}
```

`NUM_CHILDREN`是用于定义子线程数量的编译期常量。`child`是线程句柄的数组。

## 9.3 回收线程

当一个线程希望等待其它线程执行完毕，它需要调用`pthread_join`。下面是我对`pthread_join`的包装：

```
void join_thread(pthread_t thread)
{
  int ret = pthread_join(thread, NULL);
  if (ret == -1) {
    perror("pthread_join failed");
    exit(-1);
  }
}
```

参数是你想要等待的线程句柄。这个包装所做的事情就是调用`pthread_join`之后检查结果。

任何线程都可以回收其它线程，但是多数普遍的情况下，父线程创建并回收所有子线程。我们继续使用上一节的例子，下面是等待子线程的代码：

```c
for (i=0; i<NUM_CHILDREN; i++) {
    join_thread(child[i]);
}
```

这个循环一次等待一个子线程，以它们创建的顺序。没有办法来保证子线程按照顺序执行完毕，但是这个循环在它们不这样的时候也会正确执行。如果某个子线程迟于其它线程，这个循环会等待它，其它子线程也会在同时执行完毕。但是无论如何，所有子线程执行完毕后，循环才会退出。

如果你下载这本书的仓库，你可以在`counter/counter.c`中找到它。你可以像这样编译并运行它：

```sh
$ make counter
gcc -Wall counter.c -o counter -lpthread
$ ./counter
```

当我以5个子线程运行它时，我获得了如下输出：

```
counter = 0
counter = 0
counter = 1
counter = 0
counter = 3
```

当你运行它时，你可能得到了不同的结果。并且如果你再次运行它，你可能每次都得到不同的结果。到底发生了什么呢？

## 9.4 同步错误

上一个程序的问题就是，子线程访问了共享变量`counter`，不带任何同步机制，所以在任何线程增加`counter`之前，这些线程读取到了它的相同值。

下面是一个事件序列，这可以解释上一节的输出：

```
Child A reads 0
Child B reads 0
Child C reads 0
Child A prints   0
Child B prints   0
Child A sets counter=1
Child D reads 1
Child D prints   1
Child C prints   0
Child A sets counter=1
Child B sets counter=2
Child C sets counter=3
Child E reads 3
Child E prints   3
Child D sets counter=4
Child E sets counter=5
```

每次你运行这个程序的时候，线程都会在不同时间点上中断，或者调度器可能选择不同的线程来运行，所以时间序列和结果都是不同的。

假设我们需要强行规定一个顺序。例如，我们想让每个线程读到`counter`的不同值并增加它，让`counter`的值反映出执行`child_code`的线程数量。

为了达到这一要求，我们可以使用“互斥体”（mutex），它提供了互斥体对象，来保证一段代码是“互斥”的，也就是说，一次只有一个线程可以执行这段代码。

我编写了一个叫做`mutex.c`的小型模块，来提供互斥体对象。我会首先向你展示如何使用，之后再展示工作原理。

下面是`child_code`使用互斥体同步线程的版本：

```c
void child_code(Shared *shared)
{
  mutex_lock(shared->mutex);
  printf("counter = %d\n", shared->counter);
  shared->counter++;
  mutex_unlock(shared->mutex);
}
```

在任何线程访问`counter`之前，它们需要“锁住”互斥体，这样可以阻塞住所有其它线程。假设线程A锁住互斥体，并且执行到`child_code`的中间位置。如果线程B到达并执行了`mutex`，它会被阻塞。

当线程A执行完毕后，它执行了`mutex_unlock`，它允许线程B继续执行。实际上，一次只有一个排队中的线程会执行`child_code`，所以它们不会互相影响。当我以5个子线程运行这段代码时，我会得到：

```
counter = 0
counter = 1
counter = 2
counter = 3
counter = 4
```

这样就满足了要求。为了使这个方案能够工作，我向`Shared`结构体中添加了`Mutex`:

```c
typedef struct {
  int counter;
  Mutex *mutex;
} Shared;
```

之后在`make_shared`中初始化它：

```c
Shared *make_shared(int end)
{
  Shared *shared = check_malloc(sizeof(Shared));
  shared->counter = 0;
  shared->mutex = make_mutex();   //-- this line is new
  return shared;
}
```

这一节的代码在`counter_mutex.c`中，`Mutex`的定义在`mutex.c`中，我会在下一节解释它。

## 9.5 互斥体

我的`Mutex`的定义是`pthread_mutex_t`类型的包装，它定义在POSIX线程API中。

为了创建POSIX互斥体，你需要为`pthread_mutex_t`分配空间，之后调用`pthread_mutex_init`。

一个问题就是在这个API下，`pthread_mutex_t`表现为结构体，所以如果你将它作为参数传递，它会复制，这会使互斥体表现不正常。你需要传递`pthread_mutex_t`的地址来避免这种情况。

我的代码更加容易正确使用。它定义了一个类型，`Mutex`，它是`pthread_mutex_t`的更加可读的名称：

```c
#include <pthread.h>

typedef pthread_mutex_t Mutex;
```

之后它定义了`make_mutex`，它为`mutex`分配空间并初始化：

```c
Mutex *make_mutex()
{
  Mutex *mutex = check_malloc(sizeof(Mutex));
  int n = pthread_mutex_init(mutex, NULL);
  if (n != 0) perror_exit("make_lock failed"); 
  return mutex;
}
```

返回值是一个指针，你可以将其作为参数传递，而不会有非预期的复制。

对互斥体加锁和解锁的函数都是POSIX函数的简单包装：

```c
void mutex_lock(Mutex *mutex)
{
  int n = pthread_mutex_lock(mutex);
  if (n != 0) perror_exit("lock failed");
}

void mutex_unlock(Mutex *mutex)
{
  int n = pthread_mutex_unlock(mutex);
  if (n != 0) perror_exit("unlock failed");
}
```

代码在`mutex.c`和头文件`mutex.h`中。
