---
layout:     post
title:      linux网桥 2019
subtitle:   协议栈
date:       2019-07-22
author:     Secssion
header-img:	img/bridge_back.jpg
catalog: true
tags:
    - linux
---


## 网桥概念 

网桥设备：网桥工作于数据链路层，用于端口于端口之间的数据转发，不对数据帧进行修改，数据帧一般是进过`L2`层被转发出去，不会入协议栈。
路由设备：路由设备工作于网络层，用于数据帧的路由转发，数据帧进入协议帧。
	
## linux下的桥接

`Bridge`是`linux`下的虚拟设备，工作于`L2`层，功能与网桥设备类似。被添加到`Bridge`的从设备同样工作在`L2`，处于混杂模式，接收所有数据包，
处于不具备`ip`地址，对路由系统不可见。被添加到`Bridge`的从设备会把任何接收的数据帧都转发给先转发Bridge,
由`Bridge`根据`MAC`映射表决定数据包去向：转发、丢弃、发往高层协议栈。
`linux`上桥接和路由功能是可以同时存在的，所以主机本身是需要配置ip。添加到Bridge的接口处于混杂模式，不具备设置`ip`力。
但是`Bridge`本身可以被添加`ip`, `Brige`被设置ip地址后，`Bridge`将参与`ip`层的路由选择(`route -n`最后的`Iface`)。
	
## 无线虚拟网卡桥接有线网卡设备
在无线网桥设备中，同时桥接无线虚拟设备和真实网卡设备，实现无线虚拟设备得以上网。通过以下命令，实现无线设备桥有线设备，数据走向如图。

### ex:
  
	brctl addbr br0  //创建网桥
	brctl addif br0 eth0  //真实网卡设备加入网桥
	brctl addif br0 ath0 //将无线网卡设备加入网桥
	brctl addif br0 ath1 //将无线网卡设备加入网桥


![img](/img/post-in/brige.PNG)	

## bridge与netfilter

`netfilter`是`linux`下的防火墙框架,主要用于包过滤、地址转换、包处理。
如上图中，对于无线中终端来说，数据从`ath0`进入,经`Bridge`转发，从`eth0`真实网卡出去。数据经过`L2`即被转发，并不进入`L3`，常规`iptable`规则无法对数据包进行处理。
为了实现在`bridge`上的包过滤、处理，`linux`引入了`bridge_nf`, `bridge_nf`在`Bridge`转发过程中插入了钩子函数，
`Bridge`在`NF_BR_LOCAL_IN`、`NF_BR_LOCAL_OUT`、`NF_BR_PRE_ROUTING`、`NF_BR_FORWARD`、`NF_BR_POST_ROUTING`分别执行对应钩子函数，执行过程见下。

## bridge上的包处理过程


![img](/img/post-in/bridge_frame_hane.PNG)	

`br_handle_frame` 函数在桥上处理进入的数据帧。该函数对帧对初步的合法检查，并且开始解析数据帧，因为发送本地的数据帧不需要转发。
同时，该函数决定转发模式，对于发往本地数据帧，在 `NF_BR_LOCAL_IN` 点，遍历钩子函数，并更新转发表。对于需要转发给其他接口的数据帧，
在`NF_BR_PRE_ROUTING`遍历钩子函数，并回调`br_handle_frame_finish`函数。

	struct sk_buff *br_handle_frame(struct net_bridge_port *p, struct sk_buff *skb)
	{
		const unsigned char *dest = eth_hdr(skb)->h_dest;
		int (*rhook)(struct sk_buff *skb);
	
		if (!is_valid_ether_addr(eth_hdr(skb)->h_source))
			goto drop;
	
		skb = skb_share_check(skb, GFP_ATOMIC);
		if (!skb)
			return NULL;
	
		if (unlikely(is_link_local(dest))) {
			/* Pause frames shouldn't be passed up by driver anyway */
			if (skb->protocol == htons(ETH_P_PAUSE))
				goto drop;
	
			/* If STP is turned off, then forward */
			if (p->br->stp_enabled == BR_NO_STP && dest[5] == 0)
				goto forward;
	
			if (NF_HOOK(PF_BRIDGE, NF_BR_LOCAL_IN, skb, skb->dev,
					NULL, br_handle_local_finish))
				return NULL;	/* frame consumed by filter */
			else
				return skb;	/* continue processing */
		}
	
	forward:
		switch (p->state) {
		case BR_STATE_FORWARDING:
			rhook = rcu_dereference(br_should_route_hook);
			if (rhook != NULL) {
				if (rhook(skb))
					return skb;
				dest = eth_hdr(skb)->h_dest;
			}
			/* fall through */
		case BR_STATE_LEARNING:
			if (!compare_ether_addr(p->br->dev->dev_addr, dest))
				skb->pkt_type = PACKET_HOST;
	
			NF_HOOK(PF_BRIDGE, NF_BR_PRE_ROUTING, skb, skb->dev, NULL,
				br_handle_frame_finish);
			break;
		default:
	drop:
			kfree_skb(skb);
		}
		return NULL;
	}
`br_handle_frame_finish`:函数首先调用`br_ fdb_update`函数，根据数据帧的源`MA`C地址学习转发表; 然后调用 ` __br_fdb_get`函数，
根据数据帧目的`MAC`地址，查找转发接口，如果查找到了转发接口，数据帧通过`br_forward`函数发送给指定接口，
否则，通过`br_flood_forward`发送个所有的接口设备， `br_forward`或`br_flood_forward`执行过程中依次遍历`NF_BR_FORWARD`、`NF_BR_POST_ROUTING`钩子，
最终调用`br_dev_queue_push_xmit`将数据帧发送。为了接收多播帧和广播帧， 桥被设置成了接收所有的流量，或则目的`MAC`匹配了一个本地接口，
`br_pass_frame_up`  函数通过调用`netif_receive_skb` 函数修该 `dev`，并把数据帧拷贝一份发到网络层.

	int br_handle_frame_finish(struct sk_buff *skb)
	{
		const unsigned char *dest = eth_hdr(skb)->h_dest;
		struct net_bridge_port *p = rcu_dereference(skb->dev->br_port);
		struct net_bridge *br;
		struct net_bridge_fdb_entry *dst;
		struct sk_buff *skb2;
	
		if (!p || p->state == BR_STATE_DISABLED)
			goto drop;
	
		/* insert into forwarding database after filtering to avoid spoofing */
		br = p->br;
		br_fdb_update(br, p, eth_hdr(skb)->h_source);
	
		if (p->state == BR_STATE_LEARNING)
			goto drop;
	
		/* The packet skb2 goes to the local host (NULL to skip). */
		skb2 = NULL;
	
		if (br->dev->flags & IFF_PROMISC)
			skb2 = skb;
	
		dst = NULL;
	
		if (is_multicast_ether_addr(dest)) {
			br->dev->stats.multicast++;
			skb2 = skb;
		} else if ((dst = __br_fdb_get(br, dest)) && dst->is_local) {
			skb2 = skb;
			/* Do not forward the packet since it's local. */
			skb = NULL;
		}
	
		if (skb2 == skb)
			skb2 = skb_clone(skb, GFP_ATOMIC);
	
		if (skb2)
			br_pass_frame_up(br, skb2);
	
		if (skb) {
			if (dst)
				br_forward(dst->dst, skb);
			else
				br_flood_forward(br, skb);
		}
	
	out:
		return 0;
	drop:
		kfree_skb(skb);
		goto out;
	}

​`br_add_if`函数中会中注册`br_handle_frame`为`dev->rx_handle`，所以通过从设备的数据帧就被桥决定了数据处理方法

	int br_add_if(struct net_bridge *br, struct net_device *dev)
	{
		...
		err = netdev_rx_handler_register(dev, br_handle_frame, p);
		/* Make entry in forwarding database*/
		if (br_fdb_insert(br, p, dev->dev_addr, 0))
			...
		...
	}


### 参考:

- [liunx网桥解剖](https://wiki.aalto.fi/download/attachments/70789083/linux_bridging_final.pdf)

- [skb网桥转发](https://blog.csdn.net/NW_NW_NW/article/details/76153027)

​		

​	

