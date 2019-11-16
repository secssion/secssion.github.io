---
layout:     post
title:      一致性hash 2019
subtitle:   算法
date:       2019-11-16
author:     Secssion
header-img:	img/bridge_back.jpg
catalog: true
tags:
    - 算法
---


### 传统hash缺陷

分布式应用的缓存放在N台机器中，通过hash(key)%N找到机器请求缓存服务。但是某一个机器宕机，N的大小发生改变，下一次的请求机器为hash(key)%(N-1),请求机器改变，缓存失效。所以我们希望在删除某台机器节点的时候，其他机器节点缓存仍然有效。

### 一致性hash基础

一致性hash不在采用取模的方式，而是把整个 [0, 2^32-1] 按照服务的hash值分块，每个服务器负责一块区域，某个落在该区域，则去访问该服务器。

#### 机器映射

将机器节点按照IP或其他ID信息通过hash映射到环上。

![hash_ring](/img/post-in/hash_ring.png)



##### key映射

​	因为key值和服务器hash值都在一个环上，我们定义请求key属于它顺时针方向的第一个Server.

![hash_ring3](/img/post-in/hash_ring3.png)

#### 机器删除

对于请求hash(key)在(Server2, Server3)都将访问Server4，该部分缓存失效，其他区间则不受影响。机器添加同理。

​	![hash_ring5](/img/post-in/hash_ring5.png)



####  虚拟节点

上面解决了机器节点的添加删除问题，但是存在服务器请求平衡性问题。如果在某个区间存在热点值，则该服务器压力会较大。

通过增加虚拟节点使得整个hash环节点的分布能更加紧密，请求能相对更好分散到多个机器，但是均衡程度取决于虚拟节点的添加算法。

如：之前是 hash(server1"192.168.10.123")  =>  Server1.1  , 增加Server1的虚拟节点 hash(Server“192.168.10.123 ##”) =》Server1.2。



![hash_ring6](C:\Users\xiedongsheng\Desktop\hash_ring6.PNG)



### 参考

- [consistent-hashing]( https://www.toptal.com/big-data/consistent-hashing )