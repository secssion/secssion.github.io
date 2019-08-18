---
layout:     post
title:      linux内存屏障 2019
subtitle:  	内存屏障
date:       2019-08-18
author:     Secssion
header-img:	img/bridge_back.jpg
catalog: true
tags:
    - linux
---



## 指令重排
cpu执行程序期间，访问内存非常快， 为了提高cpu性能，只要程序**依赖**关系保持一致，cpu可能会以任意顺序执行内存操作。
编译期间，编译器也会以任意顺序编排编译指令，只要不影响程序的**表面**执行结果。
为了保证代码能够**有序**执行，需要在代码中加入内存屏障。

例如，考虑以下序列的执行事件。

`````
      CPU 1		CPU 2
	===============	===============
	{ A == 1; B == 2 }
	A = 3;		x = B;
	B = 4;		y = A;
    
`````

由于cpu1和cpu2各自的指令之间没有任何依赖，cpu并不保证执行的顺序，并且一个cpu上完成的store操作不会马上被一个所cpu感知到。结果可能的组合：
`````
	x == 2, y == 1
	x == 2, y == 3
	x == 4, y == 1
	x == 4, y == 3

```````

更进一步，考虑以下序列的执行事件：
``````
	CPU 1		CPU 2
	===============	===============
	{ A == 1, B == 2, C == 3, P == &A, Q == &C }
	B = 4;		Q = P;
	P = &B		D = *Q;
```````
只有 Q=p, D=*Q之间存在指令依赖关系，只能能保证Q=p，在 D=*Q之前执行， 结果可能为：
``````
    (Q == &A) and (D == 1)
    (Q == &B) and (D == 2)
    (Q == &B) and (D == 4)
 ``````

## cpu上执行顺序的最小保证  
1、给定一个cpu上，有依赖关系的内存访问将被顺序执行。
 ````
 Q = p；D=*Q；
 ````` 
 cpu执行顺序为：
 ````
 Q = LOAD P, D = LOAD *Q 
 ````
 2、在给定cpu上，出现  `load `和 `store` 重叠。
     给定指定序列事件： 
 ````
      a = READ_ONCE(*X); WRITE_ONCE(*X, b);
 ````
 cpu执行结果为：
`````
      a = LOAD *X, STORE *X = b
``````
或者给定序列事件：
 `````
    WRITE_ONCE(*X, c); d = READ_ONCE(*X);
 `````
 cpu制定结果为： 
  ``````
  STORE *X = c, d = LOAD *X
  ``````
    
## 一些能够假定和不能假定：
1、没有使用 READ_ONCE() 和WRITE_ONCE()一定不能假定编译器能和你设想一致的编译。没有它们，可能发生编译重排。
2、不能假定没有依赖关系的数据执行顺序和命令所给的执行顺序一样。
给定执行序列
`````
    X = *A; Y = *B; *D = Z;
`````
 可能得到的指令顺序结果为：
 `````
    X = LOAD *A,  Y = LOAD *B,  STORE *D = Z
	X = LOAD *A,  STORE *D = Z, Y = LOAD *B
	Y = LOAD *B,  X = LOAD *A,  STORE *D = Z
	Y = LOAD *B,  STORE *D = Z, X = LOAD *A
	STORE *D = Z, X = LOAD *A,  Y = LOAD *B
	STORE *D = Z, Y = LOAD *B,  X = LOAD *A
``````
3、能够假定发生内存重叠的操做要么被和并了，要么被忽略了。
给定执行序列
```
    X = *A; Y = *(A + 4);
```
得到结果组合可能为：
   
 ````
   X = LOAD *A; Y = LOAD *(A + 4);
	Y = LOAD *(A + 4); X = LOAD *A;
	{X, Y} = LOAD {*A, *(A + 4) };
````
        
不能保证的地方：
 * 上面保证并不适用位操作，因为编译器通常会对非原子操做的读、改、写序列产生代码新代码以达到修改目的。
 * 即使在被锁保护的位域中，所有的位一定要被一把锁所护。如果两个位分别被两把锁所保护，则编译器对非原子操做的读改写操做序列的去更新一个位进行可能造成另一个位损坏。
 
## 内存屏障
 从上可知，虽然独立的内存操作乱序执行效率非常高，但是多cpu之间的IO交互存在问题。
而内存屏障就是指导编译器和cpu严格顺序编译和顺序执行的产物。
使用内存屏障同步CPU数据过程：当一个cpu提交store操作，另一个的cpu load可能并不会感知到（cpu缓存没有更新过来）。而通过Write屏障使cpu 先提交Store操作并写入到系统内存中，另一个cpu通过Read屏障从系统内存加载数据执行到cpu缓存执行 load操作，保证数据一致

## 内存屏障分类
* Write (or store) 屏障  write屏障能够保证所有Wrtie屏障之前的 STORE操作比 Write屏障之后的STORE操作先被执行完。Write屏障应该和Read屏障或者Data dependency屏障配对。
* Data dependency 屏障：Read屏障的弱形式。第二个load会依赖第一个load形式，保证load顺序。
* Read 屏障：Read屏障能够保证所有Read屏障之前的 LOAD操作比 Read屏障之后的LOAD操作先被执行完。
* STORE操作：。

## volatile关键字
volatile关键字本质告诉编译器，对该voliate变量不做优化。volatile使用内存屏蔽，使cpu每次取值都从系统内存中取出。每次写完之后将变量所在cpu缓存的数据写回系统中。
volatile在c中，一般用于修饰外部I/O变量。
多线程编程中，可用于修饰多线程共享**标记**变量如下，并不保变量证原子性. 因为Thtread2只有把flag置1操作，即使在置1过程中，发生上下文切换，最终还是会置1， Thread1多等一会就行。

````
Thread1:
    while(flag)
    {
        dosomething();
    }
    
 Thtread2:
    if(condition)
    {
        flag = 1;
    }
```
