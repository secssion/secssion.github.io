---
layout:     post
title:      认证 2019
subtitle:   网络
date:       2019-12-10
author:     Secssion
header-img:	img/bridge_back.jpg
catalog: true
tags:
    - network
---


###  HTTP认证
对于http认证，使用http302重定向到用户认证页面。通常是在内核中检查到未通过认证的用户有http请求时，设备在内核中构造一个http302返回给用户。一般的，由于设备IP地址不确定,会在302中返回一个URL。浏览器根据URL查询DNS，设备上做DNS重定向，考虑浏览器上会做DNS缓存，该DNS不会直接返回设备的IP，该DNS重定向URL到一个“虚拟”IP地址上，由设备上对“虚拟”IP地址做dnat，经过上面一系列过程，用户就访问到了设备的认证页面。

### HTTP302
````
	HTTP/1.1 302 Moved Temporarily\r\nLocation: http:xxx
````
### HTTPS认证
对于https认证，因为存在加密，无法像http认证一样直接返回http 302，所以只能通过在内核中修改数据包的地址，将用户与外部服务器的连接偷偷改成用户和设备本身建立连接。具体操作：在PRE_ROUTING钩子点对数据做检查，对于一些特殊的数据包放行(如DNS)； 对于tcp数据帧检查MAC是否已通过认证，通过认证的MAC数据包进行放行操作，没有通过认证的数据包DNAT到设备的IP地址上去，这样用户访问外部https就被改成访问本地https(加密密钥也是和本地https交换的），由本地https进程对未认证的用户做http302返回URL，再经过如http一样URL解析过程请求过程，用户得到一个https的认证页面。

### 问题
* 上面https认证过程中，浏览器会提示证书安全问题。
* 对于三层设备https认证，可以在IP层中的NF_INET_PRE_ROUTING注册钩子函数，钩子函数里面完成把要访问的服务器IP DNAT到本身设备地址上，操作也比较容易实现(连接重定向)。
* 对于两层网桥设备https认证实现比较难弄。要把经brx桥转发的MAC未通过认证的https数据转到本地上IP层上，对网桥性能会有影响，并且怎么把经两层的数据做个类似DNAT操作转到IP层上也是个问题。


### 参考
- [iphoneportal分析](https://gmd20.github.io/blog/iPhone连接wifi热点跳转captive-portal页面原理以及页面跳转慢原因分析/)
- [手机自动弹portal分析](https://gmd20.github.io/blog/WiFi热点强制登录认证页面Captive-Portal相关资料/)
- [captibve portal技术](https://www.jianshu.com/p/b4da31480f2c)