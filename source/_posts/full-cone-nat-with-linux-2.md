---
title: Linux 内核态实现 Full Cone NAT（2）
date: 2018-03-31 15:44:29
tags:
- network
- linux
categories:
- 开发笔记
- 网络
---
> 接上篇 [从DNAT到netfilter内核子系统，浅谈Linux的Full Cone NAT实现](/2018/02/09/full-cone-nat-with-linux/) 。

Padavan固件的Cone NAT实现
-------------------
[Padavan](https://bitbucket.org/padavan/rt-n56u) 是基于华硕路由器固件的第三方智能路由器固件，这个固件通过给内核 netfilter 打 patch 的方式实现了 Cone NAT。关于该固件实现 Cone NAT 的原理及问题，在 [netfilter-full-cone-nat #1](https://github.com/Chion82/netfilter-full-cone-nat/issues/1) 中有简短的讨论。

先来看代码。Padavan 修改了 netfilter 下的 nf_conntrack_core.c 中的 `resolve_normal_ct()` 函数的实现：
```c
static inline struct nf_conn *
resolve_normal_ct(/* ... */)
{
  //...
  /* look for tuple match */
  hash = hash_conntrack_raw(&tuple);
#if defined(CONFIG_NAT_CONE)
  if (protonum == IPPROTO_UDP && nf_conntrack_nat_mode > 0 && skb->dev != NULL &&
  #if IS_ENABLED(CONFIG_PPP)
      (skb->dev->ifindex == cone_man_ifindex || skb->dev->ifindex == cone_ppp_ifindex)) {
  #else
      (skb->dev->ifindex == cone_man_ifindex)) {
  #endif
    /* CASE III To Cone NAT */
    h = __nf_cone_conntrack_find_get(net, &tuple, hash);
  } else
#endif
  {
    /* CASE I.II.IV To Linux NAT */
    h = __nf_conntrack_find_get(net, &tuple, hash);
  }

    // ...
}
```
可见，在配置项 `CONFIG_NAT_CONE` 打开时，调用了 `__nf_cone_conntrack_find_get()` 来取代原本的 `__nf_conntrack_find_get()`。来看看这个 padavan 新增的 `__nf_cone_conntrack_find_get()` 函数是如何实现的：
```c
static struct nf_conntrack_tuple_hash *
__nf_cone_conntrack_find_get(struct net *net,
           const struct nf_conntrack_tuple *tuple, u32 hash)
{
  // ...
begin:
  h = __nf_cone_conntrack_find(net, tuple, hash);
  if (h) {
    // ...
      if (unlikely(!nf_ct_cone_tuple_equal(tuple, &h->tuple))) {
          nf_ct_put(ct);
          goto begin;
      }
    // ...
  }
  // ...
  return h;
}
```
其实这个函数的实现和 `__nf_conntrack_find_get()` 逻辑上完全一致，所以我省略了大部分的展示代码。其实特别的地方在于，这个函数通过调用新增的 `__nf_cone_conntrack_find()` 来根据 tuple 查找对应的 conntrack ，通过调用 `nf_ct_cone_tuple_equal()` 来判断两个 tuple 是否相等。这两个 padavan 添加的函数就暗藏了玄机。事实上，`__nf_cone_conntrack_find()`的实现与 `____nf_conntrack_find()` 逻辑上也没有多大不同，而区别就在于：在根据 hash 遍历索引到的 bucket 时，`____nf_conntrack_find()` 使用 `nf_ct_key_equal()` 来判断两个 tuple 是否完全相等，而 `__nf_cone_conntrack_find()` 则使用了 `nf_ct_cone_tuple_equal()` 来判断两个 tuple 是否“在锥形”上相等。

```c
static inline bool
nf_ct_cone_tuple_equal(const struct nf_conntrack_tuple *t1, const struct nf_conntrack_tuple *t2) {
  if (nf_conntrack_nat_mode == NAT_MODE_FCONE)
    return __nf_ct_tuple_dst_equal(t1, t2);
  else if (nf_conntrack_nat_mode == NAT_MODE_RCONE)
    return (__nf_ct_tuple_dst_equal(t1, t2) &&
      nf_inet_addr_cmp(&t1->src.u3, &t2->src.u3)  &&
      t1->src.l3num == t2->src.l3num);
  else
    return false;
}
```
上面的这段代码就是padavan对Cone NAT实现的最关键之处。对于 Full Cone，`nf_ct_cone_tuple_equal()` 只判断两个tuple的“目标三元组”（目标IP、目标端口、协议号）是否相同；对于 IP Restricted Cone，除了判断“目标三元组”是否相同，还需要同时判断源IP是否相同。注意，此处的tuple是相对于远程主机端，即 conntrack.tuple_hash 中的 `TUPLE_REPLY` 而言的，“源”指的是远程主机（如UDP服务器），“目标”指的是本机（Linux NAT网关）的 WAN 出口。

相对应的，由于 `resolve_normal_ct()` 对 conntrack 的查找是基于对 tuple 进行 hash 的，padavan 对 `hash_conntrack_raw()` 函数也做了少量的调整，来确保在同一锥形中的两个tuple的hash相同：
```c
static u32 hash_conntrack_raw(const struct nf_conntrack_tuple *tuple) {
  unsigned int n;
#if defined(CONFIG_NAT_CONE)
  u32 a, b;
  if (nf_conntrack_nat_mode == NAT_MODE_FCONE) {
    return jhash2(tuple->dst.u3.all, sizeof(tuple->dst.u3.all) / sizeof(u32),
            nf_conntrack_hash_rnd ^ (((__force __u16)tuple->dst.u.all << 16) | tuple->dst.protonum)); // dst ip & dst port & dst proto
  }
  else if (nf_conntrack_nat_mode == NAT_MODE_RCONE) {
    a = jhash2(tuple->src.u3.all, sizeof(tuple->src.u3.all) / sizeof(u32), tuple->src.l3num); //src ip & l3 proto
    b = jhash2(tuple->dst.u3.all, sizeof(tuple->dst.u3.all) / sizeof(u32), ((__force __u16)tuple->dst.u.all << 16) | tuple->dst.protonum); // dst ip & dst port & dst proto
    return jhash_2words(a, b, nf_conntrack_hash_rnd);
  }
  else
#endif
  // ...
}
```
简而言之，padavan对Cone NAT的实现是这样的：（为简化过程，我们这里只讲Full Cone）
当入站包到达本机（Linux NAT网关）时，`resolve_normal_ct()` 被调用以用于查找该包所对应的conntrack，原版netfilter的逻辑是根据tuple的“全匹配”来查找对应conntrack，而padavan在启用Cone NAT功能后，只根据tuple的“半匹配”来找到对应的conntrack（“半匹配”即：只匹配单边的地址和端口信息，当Full Cone时，这个单边是tuple的dst）。

举个例子来说明这样实际运行的效果：
* 内网主机 192.168.1.3:3000  --> Linux网关 NAT后地址：223.0.0.1:3000 （标记为“conntrack 1”）  --> 远程主机 123.0.0.1:5000
* 另一台远程主机 123.2.2.2:6000 --> 223.0.0.1:3000 Linux网关查找到“conntrack 1”并进行restore NAT --> 内网主机 192.168.1.3:3000

映射端口随机化及问题
--------------
这里我们讨论SNAT的 `--random` 或 `--random-fully` 参数。在未使用该参数时，NAT后的源端口号会尽量保持不变，如果遇到冲突，则从端口范围的最小值（未显式指定时是1024）开始逐一往上寻找可用源端口号；当使用了随机化参数后，NAT后的源端口号是随机的，每次NAT映射产生的端口号都无特别确定的规律可循。

### Padavan Cone NAT和端口随机化

从上文的源码分析可以看出，padavan对cone nat的实现是仅考虑了入站方向的，而出站方向则没有做改动。如果我们同时考虑出站方向，当 `--random` 或 `--random-fully` 都 **未启用** 时，出入站的流程是这样的：

* 内网主机A(192.168.1.3:3000) --> NAT(223.0.0.1:3000) 【建立 "conntrack 1" 】 --> 远端主机B(123.0.0.1:5000)
* 远端主机C(123.2.2.2:6000) --> NAT(223.0.0.1:3000)【重用 "conntrack 1" 】 --> 内网主机A(192.168.1.3:3000)
* 内网主机A(192.168.1.3:3000) --> NAT(223.0.0.1:3000) 【建立 "conntrack 2" 】 --> 远端主机C(123.2.2.2:6000)

注意，当 内网主机A 向另一台 远端主机C 发送出站UDP包时，会建立一个新的 "conntrack 2"，同时进行一次端口分配。由于未启用端口随机化，linux会尽量保持源端口不变，即再次映射成3000端口，这样就保证了这个锥形NAT是双向完整的。

但是如果我们使用了 `--random` 或 `--random-fully`，这个过程就变得有趣了：

* 内网主机A(192.168.1.3:3000) --> NAT(223.0.0.1:36000) 【建立 "conntrack 1" 】 --> 远端主机B(123.0.0.1:5000)
* 远端主机C(123.2.2.2:6000) --> NAT(223.0.0.1:36000)【重用 "conntrack 1" 】 --> 内网主机A(192.168.1.3:3000)
* 内网主机A(192.168.1.3:3000) --> NAT(223.0.0.1:47000) 【建立 "conntrack 2" 】 --> 远端主机C(123.2.2.2:6000)

在建立 "conntrack 2" 的时候，端口随机化导致NAT分配了一个与 "conntrack 1" 截然不同的端口号，如果 远端主机C 对此变动不知情（比如C在Symmetric NAT后面），仍然向 223.0.0.1:36000 发送UDP包，那么导致的结果是：只有A可以收到C发送的包，而C则收不到A发送的包。

这个过程可以简单地通过 `netcat` 命令模拟，通过显式输入对方的IP:PORT，并使用 `-p` 参数指定本地的监听端口。

### netfilter-full-cone-nat 对端口随机化的改进
以下称 [netfilter-full-cone-nat](https://github.com/Chion82/netfilter-full-cone-nat) 为 xt_FULLCONENAT ，即此项目的内核模块名称。

与padavan的netfilter patch实现不同，xt_FULLCONENAT是一个第三方内核模块，没有修改conntrack核心部分的代码。padavan的原理是将一个入站packet关联至一个先前建立的conntrack；而xt_FULLCONENAT的原理是通过主动DNAT来还原入站映射。

当未使用 `--random` 或 `--random-fully` 时，xt_FULLCONENAT的表现是这样的：

* 内网主机A(192.168.1.3:3000) --> NAT(223.0.0.1:3000) 【SNAT，建立 "conntrack 1" 】 --> 远端主机B(123.0.0.1:5000)
* 远端主机C(123.2.2.2:6000) --> NAT(223.0.0.1:3000)【DNAT，建立 "conntrack 2" 】 --> 内网主机A(192.168.1.3:3000)
* 内网主机A(192.168.1.3:3000) --> NAT(223.0.0.1:3000) 【复用 "conntrack 2" 】 --> 远端主机C(123.2.2.2:6000)

当使用了 `--random` 或 `--random-fully` 时，会变成这样：

* 内网主机A(192.168.1.3:3000) --> NAT(223.0.0.1:36000) 【SNAT，建立 "conntrack 1" 】 --> 远端主机B(123.0.0.1:5000)
* 远端主机C(123.2.2.2:6000) --> NAT(223.0.0.1:36000)【DNAT，建立 "conntrack 2" 】 --> 内网主机A(192.168.1.3:3000)
* 内网主机A(192.168.1.3:3000) --> NAT(223.0.0.1:36000) 【复用 "conntrack 2" 】 --> 远端主机C(123.2.2.2:6000)

这样看上去很正常。但是，如果我们交换一下第二步和第三步，就会变成这样的结果：

* 内网主机A(192.168.1.3:3000) --> NAT(223.0.0.1:36000) 【SNAT，建立 "conntrack 1" 】 --> 远端主机B(123.0.0.1:5000)
* 内网主机A(192.168.1.3:3000) --> NAT(223.0.0.1:47000) 【SNAT，建立 "conntrack 2" 】 --> 远端主机C(123.2.2.2:6000)
* 远端主机C(123.2.2.2:6000) --> NAT(223.0.0.1:36000)【DNAT，建立 "conntrack 3" 】 --> 内网主机A(192.168.1.3:3000)

可以见到，如果C在向A发包之前，A先向C发了包，那么NAT会映射成一个新的端口，这就破坏了Cone NAT，同样会造成单边不通的问题。

为此，xt_FULLCONENAT 在最近的commit中改造了NAT映射机制：
* 除了自动DNAT，SNAT的映射端口也在模块内加以限制，以保证出站端口号的一致性。
* 因为使用了Cone NAT后，WAN接口上的一个UDP端口号只能关联唯一的内网IP:PORT，默认的linux端口分配机制（即 `unique_tuple` ）不能保证这个关联的唯一性，因此我们要在模块内完成端口分配。

现在，xt_FULLCONENAT 对出站包的处理逻辑如下：
```c
/* 注：这里的“映射”指本模块内的 natmapping 结构 */
/* 根据内网IP:PORT搜索已经存在的映射 */
src_mapping = get_mapping_by_int_src(ip, original_port);
if (src_mapping != NULL && check_mapping(src_mapping, net, zone)) {
  /* 如果这个映射已经存在，就复用这个映射：将NAT端口号限制为该条映射所记录的外网端口号 */
  newrange.flags = NF_NAT_RANGE_MAP_IPS | NF_NAT_RANGE_PROTO_SPECIFIED;
  newrange.min_proto.udp.port = cpu_to_be16(src_mapping->port);
  newrange.max_proto = newrange.min_proto;
} else {
  /* 如果未找到已存在的映射，则在本模块内进行端口分配，并创建新的映射 */
  want_port = find_appropriate_port(net, zone, original_port, ifindex, range);
  newrange.flags = NF_NAT_RANGE_MAP_IPS | NF_NAT_RANGE_PROTO_SPECIFIED;
  newrange.min_proto.udp.port = cpu_to_be16(want_port);
  newrange.max_proto = newrange.min_proto;
  src_mapping = NULL;
}
```
通过本次改造，xt_FULLCONENAT已经完全兼容了端口随机化下的Cone NAT：当内网IP:PORT不变时，无论出入站顺序和远端地址，都将映射为相同的外网端口号.

* 内网主机A(192.168.1.3:3000) --> NAT(223.0.0.1:36000) 【SNAT，建立 "conntrack 1"，随机分配出站端口 】 --> 远端主机B(123.0.0.1:5000)
* 内网主机A(192.168.1.3:3000) --> NAT(223.0.0.1:36000) 【SNAT，建立 "conntrack 2"，并复用 "conntrack 1"的出站端口 】 --> 远端主机C(123.2.2.2:6000)
* 远端主机C(123.2.2.2:6000) --> NAT(223.0.0.1:36000)【复用 "conntrack 2" 】 --> 内网主机A(192.168.1.3:3000)

xt_FULLCONENAT更新日志：映射表老化问题
------------------------
下面我们来关注 xt_FULLCONENAT 遗留的问题和更新的解决方案。
注：*映射项* 即 xt_FULLCONENAT 中 `struct natmapping` 所对应的结构。

### 主动式映射项老化检测：CONNTRACK_EVENTS

在上篇博文中，我提到：xt_FULLCONENAT 目前暂未能主动判断映射项失效，只能在需要用到该映射项的时候进行一次 `check_mapping()`，如果失效了就用新的映射替换之，以此十分被动的方法来避免映射表无限增长。

现在，我在 conntrack 核心代码中发现了在其 conntrack 回收函数 `gc_worker()` 中，在杀死 conntrack 时会触发一次 conntrack event，这让我想到了可以通过注册conntrack event回调来实现主动的映射项老化。

conntrack 事件回调的注册非常简单，在 `struct nf_ct_event_notifier` 结构体中定义一个回调函数，再通过调用 `nf_conntrack_register_notifier()` 即可在当前网络命名空间（network namespace）上注册一个事件回调。
接下来我们要做的，就是在回调函数中过滤掉除 `IPCT_DESTROY` 外的事件，获得对应的 conntrack 和 tuple ，并根据tuple的地址信息找到对应的映射项（但这时并不能立即删除映射项，后文会提到）：

```c
static int ct_event_cb(unsigned int events, struct nf_ct_event *item) {
  // ...
  ct = item->ct;
  /* we handle only conntrack destroy events */
  if (ct == NULL || !(events & (1 << IPCT_DESTROY))) {
    return 0;
  }
  // ...
  /* 这里我们不知道这个是出站conntrack（SNAT），还是入站conntrack（DNAT） 
   * 因此需要做尝试：如果不是出站conntrack，则认为是入站conntrack */
  ip = (ct_tuple->src).u3.ip;
  port = be16_to_cpu((ct_tuple->src).u.udp.port);
  mapping = get_mapping_by_int_src(ip, port);
  if (mapping == NULL) {
    ct_tuple = &(ct->tuplehash[IP_CT_DIR_REPLY].tuple);
    ip = (ct_tuple->src).u3.ip;
    port = be16_to_cpu((ct_tuple->src).u.udp.port);
    mapping = get_mapping_by_int_src(ip, port);
  }
  /* 如果都还是找不到，那么认为该conntrack与本模块无关，不作处理 */
  /* 如果找到了，就做相应的回收处理 */
}
```

通过注册conntrack事件回调，我们可以在精准的时机老化对应的映射项了。

不过，CONNTRACK_EVENTS 并不是 netfilter 中常用的 feature。在内核树中，只有 nf_conntrack_netlink 模块使用了它。
可能正由于此，conntrack notifier被设计成 **在同一个network namespace中，同时只能有一个回调被注册** ，能够注册的唯一的事件回调会保存在 `net->ct.nf_conntrack_event_cb` 。因此这造成了 [issue #5](https://github.com/Chion82/netfilter-full-cone-nat/issues/5) 的BUG：该模块不能与其他占用 CONNTRACK_EVENTS 的模块同时共用。linux内核树中占用 CONNTRACK_EVENTS 的模块只有 nf_conntrack_netlink ，用户态的 `conntrack` 命令（在  conntrack-tools 包中 ）依赖这个内核模块，该命令通常用于在用户态查询和更新 conntrack table 。

万幸的是，就算不注册conntrack事件回调，我们仍可以通过老方法，被动地防止映射表无限增长。由于我对映射项建立了双边索引（LAN端和WAN端），过长的映射表虽然会浪费较多的内存，但对索引表项的性能影响很小。

### 映射项的引用计数及 GC 时机
这里有个地方要注意。上面提到的 CONNTRACK_EVENTS 是针对conntrack而言的，当 `IPCT_DESTROY` 触发时，认为该conntrack已超时失效。由于本模块中映射项是对Cone NAT而言的，一个映射项会对应多个conntracks。

在 xt_FULLCONENAT 原来的设计中，并没有考虑后来DNAT或SNAT所产生的对应同一个映射项的多个conntracks，对一个映射是否超时的判断，是基于第一个conntrack所对应的tuple `mapping->original_tuple` 的，因此，可能会造成这样的场景：

* 内网主机A -> SNAT -> 外网主机B 【保存 original_tuple 到 mapping1】
* 外网主机C -> DNAT -> 内网主机A 【通过 mapping1 进行DNAT】
* A与B的通信停止，A与C持续通信，持续5分钟
* `original_tuple` 被 conntrack_core 的 gc_worker() 回收
* 外网主机D -> DNAT -> 丢弃【对 mapping1 进行检查，`original_tuple`失效，删除 mapping1】

为了解决这个问题，我引入了 *映射项引用计数* 的概念：当有 N 个 conntrack 与一个映射项关联时，该映射项的引用计数为 N 。
现在一个映射项的结构如下：
```c
struct nat_mapping {
  uint16_t port;     /* external UDP port */
  int ifindex;       /* external interface index*/
  __be32 int_addr;   /* internal source ip address */
  uint16_t int_port; /* internal source port */
  int refer_count;   /* how many references linked to this mapping
                      * aka. length of original_tuple_list */
  struct list_head original_tuple_list;
  struct hlist_node node_by_ext_port;
  struct hlist_node node_by_int_src;
};
```
每当一次SNAT或DNAT时，我们将所对应conntrack的 `ct->tuplehash[IP_CT_DIR_ORIGINAL].tuple` 追加至 `original_tuple_list` 列表，并累加引用计数：
```c
static void add_original_tuple_to_mapping(struct nat_mapping *mapping, const struct nf_conntrack_tuple* original_tuple) {
  struct nat_mapping_original_tuple *item = kmalloc(sizeof(struct nat_mapping_original_tuple), GFP_ATOMIC);
  if (item == NULL) {
    pr_debug("xt_FULLCONENAT: ERROR: kmalloc() for nat_mapping_original_tuple failed.\n");
    return;
  }
  memcpy(&item->tuple, original_tuple, sizeof(struct nf_conntrack_tuple));
  list_add(&item->node, &mapping->original_tuple_list);
  (mapping->refer_count)++;
}
```
当 conntrack 的 `IPCT_DESTROY` 事件触发，或需要 `check_mapping()` 时，我们从该映射项的 `original_tuple_list` 列表中删除该失效的 tuple，并将引用计数递减；当且仅当一个映射项的引用计数为零时，我们才杀掉这个映射项：
```c
static int ct_event_cb(unsigned int events, struct nf_ct_event *item) {
  // ... 接上文的 ct_event_cb()，这里已从conntrack获取到对应的mapping
  /* 遍历original_tuple_list，查找并清理失效的tuple */
  list_for_each_safe(iter, tmp, &mapping->original_tuple_list) {
    original_tuple_item = list_entry(iter, struct nat_mapping_original_tuple, node);
    if (nf_ct_tuple_equal(&original_tuple_item->tuple, ct_tuple_origin)) {
      list_del(&original_tuple_item->node);
      kfree(original_tuple_item);
      (mapping->refer_count)--;
    }
  }
  /* 当引用计数减至零时，杀死这个映射项 */
  if (mapping->refer_count <= 0) {
    kill_mapping(mapping);
  }
}
```

至此，xt_FULLCONENAT 的更新进入了新的阶段。

这里感谢 xd5520026 提出了各种冲突case及修改建议。
And many thanks to 4t0m1k for hacking into the kernel source which helped me figure out the *CONNTRACK_EVENTS* bug.
