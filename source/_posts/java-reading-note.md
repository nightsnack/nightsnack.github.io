---
title: 深入理解java虚拟机阅读笔记
date: 2018-03-19 23:12:09
tags:
    - java源码
    - java
---



### 第二章java内存区域与内存溢出异常

1. java虚拟机规范中对于java虚拟机栈规定了两种异常状况，如果线程请求的栈的深度大于虚拟机所允许的深度，将抛出**Stack Overflow**的异常，虚拟机动态扩展时都无法申请到足够的内存时，抛出**OutofMemory**的异常。

2. java虚拟机的栈内存（Stack）存放的是局部变量表的部分，包括各种基本数据类型（Boolean，byte，cjhar short float long int double），对象引用和returnAddress类型（指向了一条字节码指令的地址）。

   堆内存是虚拟机管理的内存中最大的一块。**被所有线程共享**，存放对象实例。垃圾收集器的主要管理区域。

3. ​