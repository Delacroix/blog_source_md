---
title: OSPF的基本原理与实现
date: 2016-11-04 11:14:33
categories:
- Network
tags:
- OSPF
---

# OSPF的基本原理与实现
OSPF（Open Shortest Path First，开放最短路径优先）是由IETF开发路径选择协议。
OSPF是开放的链路状态路由协议。
OSPFv2应用于IPv4中，由RFC2328规范。
## OSPF的特征：
1. 收敛快速；
2. 支持支持大规模的网络；
3. 使用Area的概念；
4. Classless（无类）路由协议。支持VLSM、CIDR、不连续子网；
5. 支持无大小限制、任意的度量值；
6. 支持多条等价路径的负载匀衡；
7. 使用保留的组播地址（224.0.0.5和224.0.0.6）；
8. 支持更安全的路由选择认证；
9. 使用可以跟踪外部路由的路由标记；

<!--more-->

## OSPF基本的工作过程：
1. Neighbor（邻居）
宣告OSPF的路由器从所有启动0SPF协议的接口上发出Hello数据包。如果两台路由器共享一条公共数据链路,并且能够相互成功协商它们各自Hello数据包中所指定的某些参数,那么它们就成为了邻居(Neighbor)。
2. Adjacency（邻接）
可以想象成为一条点到点的虚链路,它是在一些邻居路由器之间构成的。OSPF协议定义了一些网络类型和一些路由器类型的邻接关系。邻接关系的建立是由交换Hello信息的路由器类型和交换Hello信息的网络类型决定的。
3. LSA（Link State Advertisement，链路状态通告）
每一台路由器都会在所有形成邻接关系的邻居之间发送链路状态通告(Link State Advertisement，链路状态通告)。LSA描述了路由器所有的链路、接口、路由器的邻居以及链路状态信息。这些链路可以是到一个末梢网络(stub network,是指没有和其他路由器相连的网络)的链路、到其他OSPF路由器的链路、到其他区域网络的链路,或是到外部网络(从其他的路由选择进程学习到的网络)的链路。由于这些链路状态信息的多样性,OSPF协议定义了许多LSA类型。
4. LSDB（Link State Database，链路状态数据库）
每一台收到从邻居路由器发出的LSA的路由器都会把这些LSA记录在它的LSDB（Link State Database，链路状态数据库）当中,并且发送一份LSA的拷贝给该路由器的其他所有邻居。
5. Full（完全邻接）
通过LSA泛洪扩散到整个区域,所有的路由器都会形成同样的链路状态数据库。
6. SPF（Short Path First，最短路径优先）
当这些邻接的OSPF路由器的数据库完全相同时,每—台路由器都将以其自身为根,使用SPF算法来计算一个无环路的拓扑图,以描述它所知道的到达每一个目的地的最短路径(最小的路径代价)。这个拓扑图就是SPF算法树。
7. IP Routing Table（IP路由表）
每一台路由器都将从SPF算法树中构建出自己的路由表。

<div id="disqus_thread"></div>
<script>

/**
*  RECOMMENDED CONFIGURATION VARIABLES: EDIT AND UNCOMMENT THE SECTION BELOW TO INSERT DYNAMIC VALUES FROM YOUR PLATFORM OR CMS.
*  LEARN WHY DEFINING THESE VARIABLES IS IMPORTANT: https://disqus.com/admin/universalcode/#configuration-variables*/
/*
var disqus_config = function () {
this.page.url = PAGE_URL;  // Replace PAGE_URL with your page's canonical URL variable
this.page.identifier = PAGE_IDENTIFIER; // Replace PAGE_IDENTIFIER with your page's unique identifier variable
};
*/
(function() { // DON'T EDIT BELOW THIS LINE
var d = document, s = d.createElement('script');
s.src = '//delacroixsblog.disqus.com/embed.js';
s.setAttribute('data-timestamp', +new Date());
(d.head || d.body).appendChild(s);
})();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
                                