---
layout: post
title: "Linux的fork()/vfork()/clone()"
author: "Inela"
---
​	Redis中通过fork子进程来进行RDB备份，为什么使用fork()呢.

​	Linux中除了fork函数，其实还有vfork()以及clone()，出于好奇心，往深处了解下.

# Fork()

​	如下图所示，调用fork()的进程会创建一个子进程，返回的pid是0即为子进程(子进程进程ID不是0)，子进程与父进程并发执行，子进程与父进程共享地址空间，内存空间——即子进程能访问到父进程的数据，最初的fork()实现，fork()后子进程会完全拷贝父进程的内存数据，由于很多数据不做修改导致拷贝浪费很多系统资源，后面fork()的实现底层是Copy on write，即子进程修改或者父进程修改数据后，底层操作系统会将数据拷贝一份，让子进程与父进程互不影响.

![fork](https://github.com/MingJunDuan/mingjunduan.github.io/raw/main/images/mjduan/2023-12-21/fork.png)

# Vfork()

​		vfork()是由于最初fork()底层未实现Copy on wirte前新增的函数，vfork()也是创建子进程，但是vfork()创建的子进程后父进程会被挂起等子进程执行完才会执行父进程，这是因为vfork()的子进程与父进程共享地址空间、栈、程序指令指针(fork()不与父进程内存地址、栈所以可并发执行)，如果子进程与父进程同时执行会相互影响.

![vfork](https://github.com/MingJunDuan/mingjunduan.github.io/raw/main/images/mjduan/2023-12-21/vfork.png)

# Clone()

​	clone()也能创建子进程，但是clone()函数有很多个参数，可让调用者选择创建的子进程与父进程共享哪些(如地址空间、栈、内存变量等)

​	底层上fork()、vfork()、clone()都调用do_fork()函数，只是参数不同：

```
The fork(),vfork() and clone() all call the do_fork() to do the real work, but with different parameters.
```

​	Redis中RDB备份，主要是备份内存中的数据，使用fork()创建子进程，子进程在可以遍历/访问父进程的内存变量数据，而父进程在做修改时，由于Copy on write，子进程拿到的是副本数据，从底层操作系统层面高效实现了期望的功能.


参考：

1. Redis Cluster Spec：https://redis.io/docs/reference/cluster-spec/#ask-redirection