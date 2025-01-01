---
layout: post
title: MASQUERADE误用导致源IP选择出错
categories: [kernel, netfilter]
tags: [linux, netfilter, iptables]
date: 2024-04-04 16:00:00 +0800
---


## 背景
项目在使用MASQUERADE时遇到过这样一个问题，复现步骤如下：
1. 主机的eth0网卡上依次配置了公网ip：`123.1.1.10/24`和私网ip：`10.1.1.10/24`。
2. 主机配置了MASQUERADE规则：`iptables -t nat -A POSTROUTING -j MASQUERADE`。
3. 主机上运行一个服务器，监听端口8080。
4. 在该主机上使用telnet连接本机服务器：`telnet 10.1.1.10 8080`，发现服务器上看到的tcp连接的源IP是公网ip，而不是按预期使用私网ip作为源IP。

该问题原因是对所有网卡，包括lo网卡，设置了MASQUERADE；如果去除掉这个MASQUERADE规则，则不会有这个错误。本文以此问题为例，着重分析设置MASQUERADE对于源IP选择的影响。

## 路由分析

linux内核发送数据包的流程，可以参考：https://arthurchiao.art/blog/tuning-stack-tx-zh/ 。我们关注发送数据包过程中的路由匹配操作。首先看下到达目标地址`10.1.1.10`的匹配路由：
```shell
# ip r get 10.1.1.10
local 10.1.1.10 dev lo src 10.1.1.10 uid 1000
    cache <local>
```
这条路由定义了本机发送给`10.1.1.10`的数据包，从lo网卡发出，并且如果没有设置源IP时，会使用10.1.1.10作为源IP。设置源IP是在`udp_sendmsg()`或`ip_queue_xmit()`中查找到路由后进行设置的。
如果是udp协议，代码位置大致在：
```shell
udp_sendmsg() -> ip_make_skb() -> ip_copy_addrs(iph, fl4)
```

如果是tcp协议，代码位置大致在：
```shell
# 调用顺序（慢速查询）：ip_queue_xmit() -> ip_route_output_ports() -> ... -> ip_route_output_key_hash_rcu()
struct rtable *ip_route_output_key_hash_rcu(...) {
    ...
    err = fib_lookup(net, fl4, res, 0);
    ...
    if (res->type == RTN_LOCAL) {
		if (!fl4->saddr) {
			if (res->fi->fib_prefsrc)
				fl4->saddr = res->fi->fib_prefsrc;
			else
				fl4->saddr = fl4->daddr;
		}
        ...
    }
    ...
}
```

## MASQUERADE分析

基于上一节的路由分析，在网络层封装数据包时，已经基于路由设置好源IP和数据包发出的网卡。后续数据包会进入Netfilter框架，经过OUTPUT链进入到POSTROUTING后，如果设置了MASQUERADE规则，则会执行到如下代码：
```shell
unsigned int nf_nat_masquerade_ipv4(...) {
    ...
    newsrc = inet_select_addr(out, nh, RT_SCOPE_UNIVERSE);
    ...
    newrange.flags       = range->flags | NF_NAT_RANGE_MAP_IPS;
	newrange.min_addr.ip = newsrc;
	newrange.max_addr.ip = newsrc;
	newrange.min_proto   = range->min_proto;
	newrange.max_proto   = range->max_proto;

	/* Hand modified range to generic setup. */
	return nf_nat_setup_info(ct, &newrange, NF_NAT_MANIP_SRC);
}
```
后续调用nf_nat_setup_info()时会基于`newrange`对链接跟踪表项的源IP进行设置，具体流程如：nf_nat_setup_info() -> get_unique_tuple() -> find_best_ips_proto() 
```
static void find_best_ips_proto(...) {
    ...
    if (maniptype == NF_NAT_MANIP_SRC)
		var_ipp = &tuple->src.u3;
	else
		var_ipp = &tuple->dst.u3;

	/* Fast path: only one choice. */
	if (nf_inet_addr_cmp(&range->min_addr, &range->max_addr)) {
		*var_ipp = range->min_addr;
		return;
	}
    ...
}
```
这里设置好链接跟踪表项后，后续匹配到该表项的数据包就会按照链接跟踪表项的规则对源IP进行修改，这里不再展开。

我们这里重点关注下nf_nat_masquerade_ipv4() 中调用了 "inet_select_addr()" 如何对源IP进行选择，具体看代码注释：
```shell
__be32 inet_select_addr(const struct net_device *dev, __be32 dst, int scope)
{
	const struct in_ifaddr *ifa;
	__be32 addr = 0;
	unsigned char localnet_scope = RT_SCOPE_HOST;
	struct in_device *in_dev;
	struct net *net = dev_net(dev);
	int master_idx;

	rcu_read_lock();
	in_dev = __in_dev_get_rcu(dev);
	if (!in_dev)
		goto no_in_dev;

	if (unlikely(IN_DEV_ROUTE_LOCALNET(in_dev)))
		localnet_scope = RT_SCOPE_LINK;
    
    /* 遍历设备in_dev-> ifa_list地址列表中所有的primary地址，primary地址即同网段第一个配置到该网卡的地址。
        本例中，出口设备为lo，其上只有一个地址`127.0.0.1`，该地址的scope通过`ip addr`查到为host=254；
        目标地址dst为`10.1.1.10`，该地址的scope通过`ip addr`查到为global=0；
        因此该循环无法找到合适的地址。
    */
	in_dev_for_each_ifa_rcu(ifa, in_dev) {
		if (ifa->ifa_flags & IFA_F_SECONDARY)
			continue;
        /* 设备地址的scope需要小于或等于dst的scope；*/
		if (min(ifa->ifa_scope, localnet_scope) > scope)
			continue;
        /* 
        1、若目的地址为空，则直接选择该primary ip地址
        2、若该primary ip地址与dst地址在同一个子网内，则将ip地址赋值给addr，程序返回
        */
		if (!dst || inet_ifa_match(dst, ifa)) {
			addr = ifa->ifa_local;
			break;
		}
        /*将第一个primary ip地址作为默认选择地址*/
		if (!addr)
			addr = ifa->ifa_local;
	}

	if (addr)
		goto out_unlock;
no_in_dev:
	master_idx = l3mdev_master_ifindex_rcu(dev);

	/* For VRFs, the VRF device takes the place of the loopback device,
	 * with addresses on it being preferred.  Note in such cases the
	 * loopback device will be among the devices that fail the master_idx
	 * equality check in the loop below.
	 */
	if (master_idx &&
	    (dev = dev_get_by_index_rcu(net, master_idx)) &&
	    (in_dev = __in_dev_get_rcu(dev))) {
		addr = in_dev_select_addr(in_dev, scope);
		if (addr)
			goto out_unlock;
	}

    /* Not loopback addresses on loopback should be preferred
	   in this case. It is important that lo is the first interface
	   in dev_base list.
	 */
    /* 如果在指定的设备上没有找到符合要求的ip地址，则会放宽条件后，遍历所有的设备上查找符合要求的ip地址
        此处会依设备注册顺序尝试lo、eth0、eth1、...
        本例中，在尝试eth0设备时，发现可以匹配到第一个primary地址，即：公网ip为`123.1.1.10`。于是就使用该地址
    */
	for_each_netdev_rcu(net, dev) {
		if (l3mdev_master_ifindex_rcu(dev) != master_idx)
			continue;

		in_dev = __in_dev_get_rcu(dev);
		if (!in_dev)
			continue;

        /* 调用in_dev_select_addr()在该设备查找符合条件的地址。
		addr = in_dev_select_addr(in_dev, scope);
		if (addr)
			goto out_unlock;
	}
out_unlock:
	rcu_read_unlock();
	return addr;
}

static __be32 in_dev_select_addr(const struct in_device *in_dev,
				 int scope)
{
	const struct in_ifaddr *ifa;

    /* 遍历设备in_dev-> ifa_list地址列表中所有的primary地址 */
	in_dev_for_each_ifa_rcu(ifa, in_dev) {
		if (ifa->ifa_flags & IFA_F_SECONDARY)
			continue;
        /* 这里只需匹配scope不为link，且设备地址小于等于目标地址的scope即可 */
		if (ifa->ifa_scope != RT_SCOPE_LINK &&
		    ifa->ifa_scope <= scope)
			return ifa->ifa_local;
	}

	return 0;
}
```

## 优化方案

通过上文分析，我们可以发现整个IP选择出错的根本原因，是`in_dev_select_addr()`函数中对于设备IP的选择条件过于宽松，不过为了尽可能选到一个源IP，这样宽松的条件也有一定的道理。不过这样有点偏离MASQUERADE原本的含义，即选择本设备的可用的ip作为源IP，出现了选到别的设备上的不可用的ip的尴尬情况。这个问题后续或许可能会被修复，不过最新的内核版本是还没有改动的（截至2024-08-29）。
基于这样的背景，我们在应用层还是要采取必要的优化方案避免这种误用，这里列举几个方法：
* 方法一：限制对于MASQUERADE的使用，只对特定网卡生效，或不用MASQUERADE而改用SNAT明确所用的源IP。
* 方法二：将eth0、eth1等网卡的地址，也同时配置到lo网卡。

