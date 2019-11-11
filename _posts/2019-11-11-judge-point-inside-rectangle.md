---
layout:     post
title:      判断点在矩形内 2019
subtitle:   算法
date:       2019-11-11
author:     Secssion
header-img:	img/bridge_back.jpg
catalog: true
tags:
    - 算法
---

### 题意

平面坐标系中，给定一矩形四点和一点的坐标，如何该点是否在矩形内。


### 叉乘
叉乘又称为外积，它的运算结果是一个向量而不是一个标量,它有以下性质。
	 a^b = -b^a
	 |c| =  |a^b| = |a|*|b|*sin<a,b>
	 
它有以下性质：
 
	
 
 1、a^b这个向量的系数为正，则a在b的顺时针方向。

 
  
  2、a^b这个向量的系数为负，则a在b的逆时针方向。
	
	
###  判断
第一想法就是构造四个三角形是否等于矩形面积来判断，计算比较多。叉乘判断更简洁。

于点在矩形内，可判断分别判断点是否在平行的两个线段之间。
下图中，判断P分别在线段AP和CD之间，在线段BC和AD之间, 那边向量BP和CP对于向量AB而已方向肯定是一个顺时针一个逆时针。
就有：(BA^BP )*(CD^CP) <= 0   &&  (BC^BP)*(AD^AP) <= 0
![img](/img/post-in/retangle.jpg)

### 参考

- [点是否在矩形内](https://www.cnblogs.com/fangsmile/p/9306510.html)

