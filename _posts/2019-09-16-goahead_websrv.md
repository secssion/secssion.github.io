---
layout:     post
title:      goahead 2019
subtitle:  	http
date:       2019-09-16
author:     Secssion
header-img:	img/bridge_back.jpg
catalog: true
tags:
    - network
---


### 前言 
最近在想一个`https`强制重定向的问题，阅读了`goaehad`相关代码，顺便记录。

### 基本I/O循环
`websServiceEvents`实现对`socket`的循环事件检测处理。`socketSelect`检测出事件类型。`socketProcess`对应调用事件对应处理回调函数处理。`websRunEvents`实现对定时器函数检测是否到期执行。值得一提的是，`select`所用的`dela`y时间是所有的注册定时器中所用时间最小的值，如此动态处理`select`时间能较好的满足定时处理的定时精确要求。
```
	PUBLIC void websServiceEvents(int *finished)
	{
  	……
        while (!finished || !*finished) {
            if (socketSelect(-1, delay)) {
                gTimeNow = system_get_uptime();//time(0);	
                socketProcess();
            }
            nextEvent = websRunEvents();
            delay = min(delay, nextEvent);
    ……
        }
	}
```
### socketSelect 函数
`socketSelect`只完成对注册感兴趣的事件进行检测。如其他地方需要做写处理的话，注册其写事件与写回调函数  `socketCreateHandler(wp->sid, sp->handlerMask | SOCKET_WRITABLE, socketEvent, wp)`，`socoketSelect`函数`socket`加入写集合检测，可写的话，将`currentEvents`置位，由`socketProcess`做处理。对于读事件每次循环都是做检测的;对于写事件，由于`select`是水平触发，写回调完成之后是要及时移动写检测，不然如果系统写缓存充裕的话，会一直触发写事件，不过目前没找着去掉写事件的地方。
````
	……
    for (; sid < socketMax; sid++) {
        if ((sp = socketList[sid]) == NULL) {
            continue;
        }
        if (sp->handlerMask & SOCKET_READABLE || sp->flags & SOCKET_BUFFERED_READ) {
            FD_SET(sp->sock, &readFds);
            nEvents++;
        }
        if (sp->handlerMask & SOCKET_WRITABLE || sp->flags & SOCKET_BUFFERED_READ) {
            FD_SET(sp->sock, &writeFds);
            nEvents++;
        }
        if (sp->handlerMask & SOCKET_EXCEPTION) {
            FD_SET(sp->sock, &exceptFds);
            nEvents++;
        }
        if (! all) {
            break;
        }
    }

    nEvents = select(socketHighestFd + 1, &readFds, &writeFds, &exceptFds, &tv);
    for (; sid < socketMax; sid++) {
        if ((sp = socketList[sid]) == NULL) {
            continue;
        }
        if (sp->flags & SOCKET_RESERVICE) {
            if (sp->handlerMask & SOCKET_READABLE) {
                sp->currentEvents |= SOCKET_READABLE;
            }
            if (sp->handlerMask & SOCKET_WRITABLE) {
                sp->currentEvents |= SOCKET_WRITABLE;
            }
            sp->flags &= ~SOCKET_RESERVICE;
        }
        if (FD_ISSET(sp->sock, &readFds)) {
            sp->currentEvents |= SOCKET_READABLE;
        }
        if (FD_ISSET(sp->sock, &writeFds)) {
            sp->currentEvents |= SOCKET_WRITABLE;
        }
        if (FD_ISSET(sp->sock, &exceptFds)) {
            sp->currentEvents |= SOCKET_EXCEPTION;
        }
    }
    ……
````

### socketProcess
`socketProcess`检测所有的`socket`，对于已经触发过事件的`socket`执行执行`handler`。
```
PUBLIC void socketProcess()
{
    WebsSocket    *sp;
    int         sid;

    for (sid = 0; sid < socketMax; sid++) {
        if ((sp = socketList[sid]) != NULL) {
            if (sp->currentEvents & sp->handlerMask) {
                socketDoEvent(sp);
            }
        }
    }
}
```

`socketDoEvent`先检测是否是新连接过来，如果是则`accept`,然后调用`websaccept`处理新连接；否则，调用预先注册的`handler`处理触发事件。
```
static void socketDoEvent(WebsSocket *sp)
{
……
	  sid = sp->sid;
	 if (sp->currentEvents & SOCKET_READABLE) {
		if (sp->flags & SOCKET_LISTENING) { 
			TRACE;	
			socketAccept(sp);
			sp->currentEvents = 0;
			return;
		} 
	}
	if (sp->handler && (sp->handlerMask & sp->currentEvents)) {
		(sp->handler)(sid, sp->handlerMask & sp->currentEvents, sp->handler_data);
		if (socketList && sid < socketMax && socketList[sid] == sp) {
			sp->currentEvents = 0;
		}
	}
……
}
```


### websRunEvents 
`websRunEvents`检测注册的定时函数是否到期，到期执行，并给出最小定时时间。
对于每一连接过来在  `websAccept` 中都会给连接一个超时检测 ，代码用`wp->timeout = websStartEvent(PARSE_TIMEOUT, checkTimeout, (void*) wp)`注册一个定时`PARSE_TIMEOUT`检测函数`checkTimeout`。

```
WebsTime websRunEvents()
{
	……
    for (i = 0; i < callbackMax; i++) {
        if ((s = callbacks[i]) != NULL) {
            if ((delay = s->at - now) <= 0) {
                callEvent(i);
                delay = MAXINT / 1000;
                i = -1;
            }
            nextEvent = min(delay, nextEvent);
        }
    }
    return nextEvent * 1000;
	……
}
```

`checkTimeout`检测超时，对于超时部分返回`timeout`
```
static void checkTimeout(void *arg, int id)
{
	……
    wp = (Webs*) arg;
    assert(websValid(wp));
    elapsed = getTimeSinceMark(wp) * 1000;
    if (elapsed >= WEBS_TIMEOUT) {
        if (!(wp->flags & WEBS_HEADERS_CREATED)) {
            if (wp->state > WEBS_BEGIN) {
                websError(wp, HTTP_CODE_REQUEST_TIMEOUT, "Request exceeded timeout");
            } else {
                websError(wp, HTTP_CODE_REQUEST_TIMEOUT, "Idle connection closed");
            }
        }
        complete(wp, 0);
        websFree(wp);
        /* WARNING: wp not valid here */
        return;
    }
    delay = WEBS_TIMEOUT - elapsed;
    assert(delay > 0);
    websRestartEvent(id, (int) delay);
	……
}
```
### webslisten
`webslisten`调用`socketListen`创建监听端口的套接字，并注册`accept`之后的调用函数-`websaccept`。

```
PUBLIC int websListen(char *endpoint)
{
    ……
    WebsSocket  *sp;
    char        *ip, *ipaddr;
    int         port, secure, sid;
    if ((sid = socketListen(ip, port, websAccept, 0)) < 0) {
        error("Unable to open socket on port %d.", port);
        return -1;
    }
	……
    return sid;
}

PUBLIC int socketListen(char *ip, int port, SocketAccept accept, int flags)
{
    WebsSocket              *sp;
    struct sockaddr_storage addr;
    Socklen                 addrlen;
    char                    *sip;
    int                     family, protocol, sid, rc, only;
    if ((sid = socketAlloc(ip, port, accept, flags)) < 0) {
        return -1;
    }
    sp = socketList[sid];
    assert(sp);
    sip = ((ip == 0 || *ip == '\0') && socketHasDualNetworkStack()) ? "::" : ip;
    if (socketInfo(sip, port, &family, &protocol, &addr, &addrlen) < 0) {
        return -1;
    }
    if ((sp->sock = socket(family, SOCK_STREAM, protocol)) == SOCKET_ERROR) {
        socketFree(sid);
        return -1;
    }
    fcntl(sp->sock, F_SETFD, FD_CLOEXEC);
    if (bind(sp->sock, (struct sockaddr*) &addr, addrlen) == SOCKET_ERROR) {
        error("Can't bind to address %s:%d, errno %d", ip ? ip : "*", port, errno);
        socketFree(sid);
        return -1;
    }
    if (listen(sp->sock, SOMAXCONN) < 0) {
        socketFree(sid);
        return -1;
    }
    sp->flags |= SOCKET_LISTENING | SOCKET_NODELAY;
    sp->handlerMask |= SOCKET_READABLE;
    socketSetBlock(sid, (flags & SOCKET_BLOCK));
    if (sp->flags & SOCKET_NODELAY) {
        socketSetNoDelay(sid, 1);
    }
    return sid;
}
```
### socketEvent
一个新的连接过来后`websaccept`调用 `socketEvent(sid,SOCKET_READABLE, wp)`来读取数据。
`socketEvent`调用`socketEvent`读取数据，调用`websPump(wp)`按阶段解析请求行、请求头数据。如果注册过`route`,用`route->handler`处理数据;如果请求文件，则读取文件数据；存在返回数据给对方，则注册写事件。

###     WebsBuf
`goahead`采用`websBuf`作为 数据输入输出的`buf`。`WebsBuf`是一种环形队列结构。采用环形队列读取数据都从`endp`开始读，每次取完数据调整`servp`，避免了每次追加数据要获取数据长度，每次读取完部分数据后手动数据下部分起始。当`servp==endp`时，为空；`servp==（endp+1)%size`时，缓存区已满。
基本数据结构：
````
    typedef struct WebsBuf {
        char    *buf;               /**< Holding buffer for data */
        char    *servp;             /**< Pointer to start of data */
        char    *endp;              /**< Pointer to end of data */
        char    *endbuf;            /**< Pointer to end of buffer */
        ssize   buflen;             /**< Length of ring queue */
        ssize   maxsize;            /**< Maximum size */
        int     increment;          /**< Growth increment */
    } WebsBuf;
 ````
 
创建一个环形队列，首尾指向同处。
```
    PUBLIC int bufCreate(WebsBuf *bp, int initSize, int maxsize)
    {
        int increment;
        if (initSize <= 0) {
            initSize = BIT_GOAHEAD_LIMIT_BUFFER;
        }
        if (maxsize <= 0) {
            maxsize = initSize;
        }
        assert(initSize >= 0);
        increment = getBinBlockSize(initSize);
        if ((bp->buf = walloc((increment))) == NULL) {
            return -1;
        }
        bp->maxsize = maxsize;
        bp->buflen = increment;
        bp->increment = increment;
        bp->endbuf = &bp->buf[bp->buflen];
        bp->servp = bp->buf;
        bp->endp = bp->buf;
        *bp->servp = '\0';
        return 0;
    }
```

往环形队列填充数据，endp指针往后移，调整缓存区剩余大小。

````
    PUBLIC ssize bufPutStr(WebsBuf *bp, char *str)
    {
        ssize   rc;
        rc = bufPutBlk(bp, str, strlen(str) * sizeof(char));
        *((char*) bp->endp) = (char) '\0';
        return rc;
    }
    PUBLIC ssize bufPutBlk(WebsBuf *bp, char *buf, ssize size)
    {
        ssize   this, added;
        added = 0;
        while (size > 0) {
            this = min(bufRoom(bp), size);
            if (this <= 0) {
                if (! bufGrow(bp, 0)) {
                    break;
                }
                this = min(bufRoom(bp), size);
            }
            memcpy(bp->endp, buf, this);
            buf += this;
            bp->endp += this;
            size -= this;
            added += this;
            if (bp->endp >= bp->endbuf) {
                bp->endp = bp->buf;
            }
        }
        return added;
    }
````	
	

从队列读取数据，servp往后移动。



```
	PUBLIC ssize bufGetBlk(WebsBuf *bp, char *buf, ssize size)
    {
        ssize   this, bytes_read;
        bytes_read = 0;
        while (size > 0) {
            this = bufGetBlkMax(bp);
            this = min(this, size);
            if (this <= 0) {
                break;
            }
            memcpy(buf, bp->servp, this);
            buf += this;
            bp->servp += this;
            size -= this;
            bytes_read += this;

            if (bp->servp >= bp->endbuf) {
                bp->servp = bp->buf;
            }
        }
        return bytes_read;
    }
```
