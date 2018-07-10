---
title: 从DNAT到netfilter内核子系统，浅谈Linux的Full Cone NAT实现
date: 2018-02-09 20:51:54
tags:
- network
- linux
categories:
- 开发笔记
- 网络
---

前言
---

* 根据 [RFC 3489](https://www.ietf.org/rfc/rfc3489.txt) 中的定义，NAT类型被划分为以下四种： Full Cone, Restricted Cone, Port Restricted Cone, Symmetric. (RFC 5780 中对NAT类型定义有了更严格的拓展分类，但不是十分常用，本文暂不探讨。)
* STUN过程( [RFC 3489](https://www.ietf.org/rfc/rfc3489.txt) 和 [RFC 5389](https://tools.ietf.org/html/rfc5389) )可以实现内网穿透，从而实现点到点通信。STUN技术被广泛应用于 VoIP、WebRTC、网络游戏、VPN等涉及低延迟点到点通讯的领域。
* 基于STUN的内网穿透成功率取决于通信双方的NAT类型，其中 Full Cone NAT 的成功率最高。
* 关于NAT type和STUN协议、STUN过程的详细知识请参考其他文章。本文仅探讨 RFC 3489 下定义的 Full Cone NAT 和 UDP hole punching 技术。
* 本文中的NAT特指NAPT。ALG等周边技术在部分厂商的linux网络设备上已有相应实现，暂不讨论。
* 你可以直接到 https://github.com/Chion82/netfilter-full-cone-nat 使用本文的解决方案。

关于NAT类型在Linux上实现的现状
------------
TLDR: Linux内核树上未实现真正意义上的Full Cone NAT，Linux的 SNAT/MASQUERADE（以`iptables`的配置为例）均是Symmetric NAT。

2018.3 更新：  
一个比较少众的华硕路由器第三方固件 padavan 通过netfilter patch的方式实现了 Cone NAT，但是其实现存在较多问题，如不支持端口随机化、内核升级不友好等，具体在下一篇博客中讨论。

## 现有的在Linux上的Full Cone NAT(?)的实现
之所以“Linux上未真正实现Full Cone NAT”这个问题至今没有被重视，是因为总有那么个“看似能够代替的方案”。事实上，稍微谷歌一下不难找到这样来实现Full Cone NAT的方法（假设`eth0`是外网出口网卡，内网主机是`192.168.1.3`）：
```
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE #普通的SNAT
iptables -t nat -A PREROUTING -i eth0 -j DNAT --to-destination 192.168.1.3 #将入站流量DNAT转发到内网主机192.168.1.3
```
类似的实现还有`NETMAP`。这种设置都有一个共同点：入站流量其实是 **定向DNAT转发** 到某一台内网主机中，从而对该台内网主机而言，它的NAT类型总是Full Cone的。事实上这种设置更像是DMZ host，是不分条件地转发所有入站流量到内网DMZ主机。然而，这种设置只能对该台内网主机有效，对于同一内网下的其他主机，它们的NAT类型仍然是Symmetric。

### UPnP技术
为了解决这个问题，UPnP起到了很重要的作用。事实上，UPnP技术基本上解决了家庭用户的内网穿透需求。（鉴于现在大多数用户都使用基于Linux的智能路由器的光猫等）在Linux上，实现UPnP的应用程序通常是 [miniupnp](https://github.com/miniupnp/miniupnp)。
内网主机的应用程序在需要进行内网穿透前，对路由器进行一次UPnP请求，该请求映射一个外网端口（UDP或TCP）到内网主机的端口。miniupnp随后会执行若干条 `iptables` 命令，例如最重要的是这条：（为便于理解，该命令已简化，事实上miniupnpd会在 `nat` 表下另建一个链）
```
iptables -t nat -A PREROUTING -i eth0 -p udp --dport 5000 -j DNAT --to-destination 192.168.1.3:5000
```
这条命令映射了外网的 `udp:5000` 端口到内网主机 `192.168.1.3` 的 `udp:5000`。只要外网端口号不重复，内网内的任何主机都能够再次发起UPnP请求来申请新的映射。映射成功后，该内网主机使用申请的内网端口进行通信时是Full Cone的。

然而，UPnP有较大的局限性：
* UPnP涉及SSDP组播发现过程，miniupnpd的默认配置更是会向客户端返回全局映射表，相对而言不太安全，不适合应用于复杂的企业级大型网络及ISP入户网络。
* 一次UPnP请求只能进行一级NAT的映射，当然你也可以转发UPnP请求到上级NAT设备，但是似乎至今没有一款家用智能路由器会这么做。这意味着如果你在家使用两台以上的智能路由器级联上网，仅通过UPnP是无法实现内网穿透的。

综上所述，目前Linux下对于Full Cone NAT的实现均是基于主动的端口映射技术（添加DNAT规则），这种实现一定程度上能满足现今的大多数家用场景，但仍在很多场合下有较大的局限性：如网吧、ISP的NAT(LSN)网关等，这些组织对于Full Cone NAT的实现通常是使用非Linux内核的企业级路由设备，这些设备可能基于如思科的硬转发芯片、FPGA等。

## Mailing
对于Full Cone NAT这个feature，可以从内核（具体是netfilter子系统）的mailing list上找到相关的问题：
> [Configure to Full Cone](http://lists.netfilter.org/pipermail/netfilter/2005-February/058428.html) :
> How can I configure IPtables to be Full Cone?
> \- You cannot. iptable_nat only implements the most sophisticated version
of NAT: fully symmetric.

> [IPTables and different types of NAT](http://lists.netfilter.org/pipermail/netfilter/2007-February/067963.html) :
> "Full cone NAT" can be implemented with 1-to-1 bidirectional NAT using
SNAT+DNAT or NETMAP.

由此可见，他们的回答基本和我们的讨论基本无二：netfilter只实现最成熟的symmetric NAT，如果要实现Full Cone NAT，需要使用一对一的定向DNAT/NETMAP转发。

## 还有没有必要在Linux上实现真正意义上的Full Cone NAT？
经过对DMZ host和UPnP的分析，我们重新定义何为“真正意义上的Full Cone NAT”：对于内网主机中的任意一台主机，在不借助“主动端口映射”申请（如UPnP）和“手动端口转发配置”（如DMZ host或DNAT规则）下，在路由器端口用完之前，都是Full Cone的；路由器的“入站端口映射规则”是根据“内网主机出站端口映射记录”动态生成的，无论内网主机数量是否改变，内网主机的IP地址是否变更，在不依赖人工改变配置时，内网主机的NAT类型始终为Full Cone。
当然，这个定义只是为了方便理解本文的内容而定下的。

Linux的NAT类型的影响范围：涉及了所有日渐普及的基于Linux的嵌入式网络设备，如家用光猫、家用智能路由器等。事实上，部分小区级别的ISP由于节约成本使用的也是基于Linux的NAT路由器，出于安全考虑这些路由器当然不会开启UPnP，因此这部分ISP的终端用户将不可能得到完整的Full Cone NAT。对于电信、移动等一级运营商，通过光纤入户的普通用户通常经过企业级LSN设备的NAT，至此入户部分的PPPoE拨号终端通常是Full Cone的，但是，由于一部分用户使用光猫拨号再由光猫NAT后路由至用户最终的计算机（这也是运营商推荐的方式），而光猫没有开启UPnP或DMZ（用户通常没有权限配置），这部分用户的终端设备也不可能得到Full Cone NAT而通常是Symmetric NAT。

因此作者认为，还是有必要在Linux上实现Full Cone NAT。Symmetric NAT意味着更高的安全性，而Cone NAT能兼容更多的应用程序。我们当然不会通过硬编码来实现这个feature，而应该是为用户提供一个新的选择。

netfilter内核子系统和conntrack
--------------
经过作者的各种尝试，要实现一个高效的原生Full Cone NAT，在应用层是无法实现的了（就算通过程序抓包后实时添加iptables规则，或从用户态刷新conntrack，也非常低效），因此，我们只能深入linux内核来进一步研究。

**HACK THE KERNEL!**

上文已有提到，在linux中负责处理nat的内核子系统是netfilter，而netfilter对应的前端有iptables和nftables。

为对nat的状态进行跟踪，netfilter引入了conntrack。conntrack用来记录每一个连接（TCP/UDP/ICMP/DCCP等会话）的双向地址和端口（或id, code等会话标识符）信息。netfilter对每一个conntrack定义了一个 `struct nf_conn` 结构：
```c
// include/net/netfilter/nf_conntrack.h
struct nf_conn {
  struct nf_conntrack ct_general;
  struct nf_conntrack_tuple_hash tuplehash[IP_CT_DIR_MAX];
  unsigned long status;
  // ...
};
```
其中，`tuplehash` 数组是我们需要关心的。该数组通常有2个成员：`tuplehash[IP_CT_DIR_ORIGINAL]` 和 `tuplehash[IP_CT_DIR_REPLY]`。要了解它们分别代表什么，先来看一下 `struct nf_conntrack_tuple_hash` 里的 `struct nf_conntrack_tuple tuple` 成员。这个 `tuple` 结构定义如下（为便于理解，一些不在同一源文件定义的struct和union被整合进来）：
```c
// include/net/netfilter/nf_conntrack_tuple.h
struct nf_conntrack_tuple {
  struct nf_conntrack_man {   //tuple.src 指示了源地址和源会话标识符
    union nf_inet_addr u3;    //源IP地址（ipv4或ipv6）
    union nf_conntrack_man_proto {
      // 这个union指示了源会话标识符，可以是一个TCP/UDP端口，或ICMP id，dccp port等。
      // 此处省略除TCP、UDP外的其它协议
      __be16 all;
      struct {
        __be16 port;
      } tcp;
      struct {
        __be16 port;
      } udp;
      // ...
    } u;
    u_int16_t l3num;
  } src;
  struct {                  //tuple.dst 指示了目的地址和目的会话标识符
    union nf_inet_addr u3;  //目的IP地址
    union {                 //和tuple.src一样，这个union指示了目的会话标识符
      __be16 all;
      struct {
        __be16 port;
      } tcp;
      struct {
        __be16 port;
      } udp;
      // ...
    } u;
    u_int8_t protonum;      //协议号
    u_int8_t dir;
  } dst;
};
```
不难看出，`struct nf_conntrack_tuple` 结构是一个五元组，其包括了：源地址/源端口/目的地址/目的端口/协议号。而在刚才的 `tuplehash` 数组中，`tuplehash[IP_CT_DIR_ORIGINAL]` 和 `tuplehash[IP_CT_DIR_REPLY]` 分别代表 “源五元组” 和 “期望收到的应答五元组”。在这里，你可以分别把它们暂时简单的理解为出站和入站的五元组，一个conntrack由一对五元组组成。

在 `tuplehash[IP_CT_DIR_ORIGINAL]` 中，`src` 指的是内网主机（或本机）的源地址，`dst` 指的是TCP/UDP流量的远端目的地址；而在 `tuplehash[IP_CT_DIR_REPLY]` 中，`src` 是TCP/UDP的远端主机的地址，`dst` 是本机的地址。

在内网主机经过NAT网关一次普通的SNAT后，一个conntrack存放的一对五元组tuple应包含如下信息：
* `tuplehash[IP_CT_DIR_ORIGINAL].tuple` ： 相对于内网主机到远端主机的方向，如 `src`192.168.1.3:5000 -> `dst`114.114.114.114:53
* `tuplehash[IP_CT_DIR_REPLY].tuple` ：相对于远端主机到本机的方向，如 `src`114.114.114.114:53 -> `dst`120.239.65.166:38720 （其中120.239.65.166是本机即NAT网关的外网IP，38720是SNAT映射后得到的外网端口）

参考nf_nat，我们可以概括出一次SNAT由如下步骤完成：
1. 来自内网主机的数据包流经nat表的POSTROUTING链并触发SNAT/MASQUERADE hook，nf_nat 进行源IP转换和端口映射，并调用 `nf_nat_setup_info()` 对当前conntrack（我们假设这个conntrack命名为conn1）的tuplehash信息进行修改，将转换后的源外网IP和映射后的外网端口写到 `tuplehash[IP_CT_DIR_REPLY].tuple.dst` 中。
2. 当有新的数据包流入时，nf_conntrack_core 通过在 `resolve_normal_ct()` 中根据流入的数据包得到对应的一个tuple，因为这个tuple的信息与 conn1 的 `tuplehash[IP_CT_DIR_REPLY].tuple` 一致，调用 `nf_conntrack_find_get()` 即获取到先前的 conn1。
3. 再根据 conn1 的 `tuplehash[IP_CT_DIR_ORIGINAL].tuple` 信息，将流入的数据包的目的IP和端口还原成内网主机的IP的端口。

\* 以上过程仅从代码层面推测，未经过严格debug验证，如实际过程有误欢迎提出。

编写一个xt_FULLCONENAT内核模块
---------------------------
要实现一个原生的Full Cone NAT功能，至少需要编写一个netfilter内核模块，外加一个或多个前端模块（前端模块是在用户态的一个.so文件，不涉及内核态API）。

注意，我们现在只关注RFC3489，即只实现UDP的Full Cone。其它协议的内网穿透相对而言不太常用，我们暂且搁置。

在此列出以下两种实现方案：
* 实现应用于mangle表的hook，对每个流出流入的包进行地址和端口信息修改。相当于手动实现一遍DNAT和SNAT。这种方案可以绕过conntrack对五元组的严格验证，但实现复杂，而且性能较差。
* 参考现有的NAT模块（如NETMAP），实现应用于nat表的hook。这种方案采用netfilter现有的nat方法来实现地址转换。但是受限于conntrack的五元组约束，除了需要依赖conntrack模块内置的映射规则来进行标准的nat，还需要另行在我们的模块中维护一张映射表。

作者采用了第二种方案。大体的设想是：在nat表的POSTROUTING链和PREROUTING链各添加一个FULLCONENAT规则，对于POSTROUTING的操作，FULLCONENAT与MASQUERADE表现无异，但需要将“端口映射记录”暂存到本模块维护的映射表中；而在PREROUTING链，FULLCONENAT对于每一个未被conntrack记录的入站连接，根据本模块维护的端口映射表，**按需DNAT**至相应的内网主机。

在前端使用iptables时，应用本模块的规则应该是这样的（假设 `eth0` 是公网出口网卡）：
```
iptables -t nat -A POSTROUTING -o eth0 -j FULLCONENAT #same as MASQUERADE  
iptables -t nat -A PREROUTING -i eth0 -j FULLCONENAT  #automatically restore NAT for inbound packets
```

我们使用一个hashmap来维护这个端口映射表。作者定义了这样的一个hashmap节点：
```c
struct natmapping {
  uint16_t port;    /* external port */
  __be32 int_addr;  /* internal ip address */
  uint16_t int_port; /* internal port */
  struct nf_conntrack_tuple original_tuple;
  struct hlist_node node;
};
```
当FULLCONENAT target应用在POSTROUTING链时，我们实现与MASQUERADE基本相同的SNAT：
```c
__be32 new_ip = get_device_ip(skb->dev);  // 获取interface的本地IP
newrange.min_addr.ip = new_ip;            // 将nat ip限制为interface的本地IP
newrange.max_addr.ip = new_ip;
ret = nf_nat_setup_info(ct, &newrange, HOOK2MANIP(xt_hooknum(par)));  // 调用nf_nat_setup_info()进行SNAT
```
然后，我们将映射结果保存到端口映射表中：
```c
ct_tuple_origin = &(ct->tuplehash[IP_CT_DIR_ORIGINAL].tuple);
ct_tuple = &(ct->tuplehash[IP_CT_DIR_REPLY].tuple);
ip = (ct_tuple_origin->src).u3.ip;  //这里获取到内网主机的IP
original_port = be16_to_cpu((ct_tuple_origin->src).u.udp.port); //内网主机的端口号
port = be16_to_cpu((ct_tuple->dst).u.udp.port); //映射后的外网端口号

//把映射存进映射表中
mapping = get_mapping(port);  //根据外网端口号从映射表中获取对应的映射记录，可能创建了新的hashmap节点
mapping->int_addr = ip;   //内网主机的IP
mapping->int_port = original_port;  //内网主机的端口号
//拷贝整个tuplehash[IP_CT_DIR_ORIGINAL].tuple，至于为什么这么做稍后会解释
memcpy(&mapping->original_tuple, ct_tuple_origin, sizeof(struct nf_conntrack_tuple));
```
当FULLCONENAT target应用在PREROUTING链时，我们需要实时查询映射表，并实现相应的DNAT：
```c
ct_tuple = &(ct->tuplehash[IP_CT_DIR_ORIGINAL].tuple);
ip = (ct_tuple->src).u3.ip;   //外网IP
port = be16_to_cpu((ct_tuple->dst).u.udp.port); //外网端口号
mapping = get_mapping(port); //根据外网端口号从映射表中获取对应的映射记录
if (is_mapping_active(mapping, ct)) {
  //如果映射记录有效，则继续根据该映射记录执行DNAT：
  newrange.flags |= NF_NAT_RANGE_PROTO_SPECIFIED;
  newrange.min_addr.ip = mapping->int_addr;   //将nat ip限制为内网主机IP
  newrange.max_addr.ip = mapping->int_addr;
  newrange.min_proto.udp.port = cpu_to_be16(mapping->int_port);   //将nat port限制为内网主机端口号
  newrange.max_proto = newrange.min_proto;
  ret = nf_nat_setup_info(ct, &newrange, HOOK2MANIP(xt_hooknum(par)));  // 调用nf_nat_setup_info()进行DNAT
}
```
对于如何“在conntrack失效（超时）时，主动从映射表中删除对应的映射记录”，作者目前暂时未找到好的方法（尽量不改动nf_conntrack部分，是否能够注册ct失效回调？）。也就是说现在这个映射表“只能添加和替换记录，而不能删除记录”（映射表的长度不会超过映射端口的范围，暂时不会导致严重的内存泄漏问题）。

为了确保在DNAT之前，先前SNAT时注册的conntrack仍然有效，作者实现了一个 `is_mapping_active()` 函数，该函数调用 `nf_conntrack_find_get()` ，通过在映射记录中保存的 `original_tuple` 来查询对应的conntrack，如果查找结果为空，则认为该conntrack已失效，否则认为该conntrack仍然有效，并继续进行DNAT。当然，这并不是最好的方法，如果你有更好的建议，欢迎联系我。
```c
static int is_mapping_active(const struct natmapping* mapping, const struct nf_conn *ct)
{
  const struct nf_conntrack_zone *zone;
  struct net *net;
  struct nf_conntrack_tuple_hash *original_tuple_hash;
  if (mapping->port == 0 || mapping->int_addr == 0 || mapping->int_port == 0) {
    return 0;
  }
  /* get corresponding conntrack from the saved tuple */
  net = nf_ct_net(ct);
  zone = nf_ct_zone(ct);
  if (net == NULL || zone == NULL) {
    return 0;
  }
  original_tuple_hash = nf_conntrack_find_get(net, zone, &mapping->original_tuple);
  if (original_tuple_hash) {
    /* if the corresponding conntrack is found, consider the mapping is active */
    return 1;
  } else {
    return 0;
  }
}
```
至此，我们的xt_FULLCONENAT模块的核心部分已阐述完毕。除此之外，还需要编写对应的前端模块，作者参考了ipt_MASQUERADE，通过简单的关键字替换实现了ipt_FULLCONENAT，作为iptables extension可提供给iptables使用。

这个内核模块以及iptables extension的完整代码、编译和安装方法请参考GitHub仓库： https://github.com/Chion82/netfilter-full-cone-nat

FAQ
---
* 使用了Linux设备作为NAT网关，内网主机运行STUN测试的测试结果是Port Restricted Cone NAT，而不是Symmetric NAT。
这是因为Stun和netfilter对映射要素的理解存在差异造成的，具体可参考 [这篇文章](http://blog.csdn.net/u011245325/article/details/9294229)。但不能因此就断定linux的SNAT不是Symmetric。实际上，这是因为linux NAT网关上存在这条iptables规则导致的（OpenWRT发行版的firewall默认存在这条规则，虽然是在INPUT的子链中）：
```
iptables -t filter -A INPUT -i eth0 -j REJECT #或者INPUT的policy为REJECT
```
如果删除了这条规则（在OpenWRT上则是在Firewall设置中，将wan zone的Input策略从reject改为accept），内网主机运行STUN测试的结果会始终是Symmetric NAT。
