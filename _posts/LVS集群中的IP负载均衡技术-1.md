﻿layout: '[post]'
title: LVS集群中的IP负载均衡技术
date: 2016-11-03 10:55:24
toc: true
categories:
- Network
tags: 
- 负载均衡

---

# [转]LVS集群中的IP负载均衡技术


> 本文在分析服务器集群实现虚拟网络服务的相关技术上，详细描述了LVS集群中实现的三种IP负载均衡技术（VS/NAT、VS/TUN和VS/DR）的工作原理，以及它们的优缺点。

-------------------


## 1 前言

在前面文章中，讲述了可伸缩网络服务的几种结构，它们都需要一个前端的负载调度器（或者多个进行主从备份）。我们先分析实现虚拟网络服务的主要技术，指出 IP负载均衡技术是在负载调度器的实现技术中效率最高的。在已有的IP负载均衡技术中，主要有通过网络地址转换（Network Address Translation）将一组服务器构成一个高性能的、高可用的虚拟服务器，我们称之为VS/NAT技术（Virtual Server via Network Address Translation）。在分析VS/NAT的缺点和网络服务的非对称性的基础上，我们提出了通过IP隧道实现虚拟服务器的方法VS/TUN （Virtual Server via IP Tunneling），和通过直接路由实现虚拟服务器的方法VS/DR（Virtual Server via Direct Routing），它们可以极大地提高系统的伸缩性。VS/NAT、VS/TUN和VS/DR技术是LVS集群中实现的三种IP负载均衡技术，我们将在文 章中详细描述它们的工作原理和各自的优缺点。

在以下描述中，我们称客户的socket和服务器的socket之间的数据通讯为连接，无论它们是使用TCP还是UDP协议。下面简述当前用服务器集群实现高可伸缩、高可用网络服务的几种负载调度方法，并列举几个在这方面有代表性的研究项目。

<!--more-->

## 2 实现虚拟服务的相关方法
在网络服务中，一端是客户程序，另一端是服务程序，在中间可能有代理程序。由此看来，可以在不同的层次上实现多台服务器的负载均衡。用集群解决网络服务性能问题的现有方法主要分为以下四类。

### 2.1 基于RR-DNS的解决方法
NCSA的可伸缩的WEB服务器系统就是最早基于RR-DNS（Round-Robin Domain Name System）的原型系统[1,2]。它的结构和工作流程如下图所示：
!["图1：基于RR-DNS的可伸缩WEB服务器"|center](/images/1478135814491.png)

有一组WEB服务器，他们通过分布式文件系统AFS(Andrew File System)来共享所有的HTML文档。这组服务器拥有相同的域名（如www.ncsa.uiuc.edu），当用户按照这个域名访问时, RR-DNS服务器会把域名轮流解析到这组服务器的不同IP地址，从而将访问负载分到各台服务器上。

这种方法带来几个问题。第一，域名服务器是一个分布式系统，是按照一定的层次结构组织的。当用户就域名解析请求提交给本地的域名服务器，它会因不能直接解析而向上一级域名服务器提交，上一级域 名服务器再依次向上提交，直到RR-DNS域名服器把这个域名解析到其中一台服务器的IP地址。可见，从用户到RR-DNS间存在多台域名服器，而它们都 会缓冲已解析的名字到IP地址的映射,这会导致该域名服器组下所有用户都会访问同一WEB服务器，出现不同WEB服务器间严重的负载不平衡。为了保证在域 名服务器中域名到IP地址的映射不被长久缓冲，RR-DNS在域名到IP地址的映射上设置一个TTL(Time To Live)值，过了这一段时间，域名服务器将这个映射从缓冲中淘汰。当用户请求，它会再向上一级域名服器提交请求并进行重新影射。这就涉及到如何设置这个 TTL值，若这个值太大，在这个TTL期间，很多请求会被映射到同一台WEB服务器上，同样会导致严重的负载不平衡。若这个值太小，例如是0，会导致本地域名服务器频繁地向RR-DNS提交请求，增加了域名解析的网络流量，同样会使RR-DNS服务器成为系统中一个新的瓶颈。

第二，用户机器会缓冲从名字到IP地址的映射，而不受TTL值的影响，用户的访问请求会被送到同一台WEB服务器上。由于用户访问请求的突发性和访问方式不同，例如有的 人访问一下就离开了，而有的人访问可长达几个小时，所以各台服务器间的负载仍存在倾斜（Skew）而不能控制。假设用户在每个会话中平均请求数为20，负 载最大的服务器获得的请求数额高于各服务器平均请求数的平均比率超过百分之三十。也就是说，当TTL值为0时，因为用户访问的突发性也会存在着较严重的负 载不平衡。

第三，系统的可靠性和可维护性差。若一台服务器失效，会导致将域名解析到该服务器的用户看到服务中断，即使用户按 “Reload”按钮，也无济于事。系统管理员也不能随时地将一台服务器切出服务进行系统维护，如进行操作系统和应用软件升级，这需要修改RR-DNS服 务器中的IP地址列表，把该服务器的IP地址从中划掉，然后等上几天或者更长的时间，等所有域名服器将该域名到这台服务器的映射淘汰，和所有映射到这台服 务器的客户机不再使用该站点为止。

### 2.2 基于客户端的解决方法

基于客户端的解决方法需要每个客户程序都有一定的服务器集群的知识，进而把以负载均衡的方式将请求发到不同的服务器。例如，Netscape Navigator浏览器访问Netscape的主页时，它会随机地从一百多台服务器中挑选第N台，最后将请求送往wwwN.netscape.com。 然而，这不是很好的解决方法，Netscape只是利用它的Navigator避免了RR-DNS解析的麻烦，当使用IE等其他浏览器不可避免的要进行 RR-DNS解析。

Smart Client[3]是Berkeley做的另一种基于客户端的解决方法。服务提供一个Java Applet在客户方浏览器中运行，Applet向各个服务器发请求来收集服务器的负载等信息，再根据这些信息将客户的请求发到相应的服务器。高可用性也 在Applet中实现，当服务器没有响应时，Applet向另一个服务器转发请求。这种方法的透明性不好，Applet向各服务器查询来收集信息会增加额 外的网络流量，不具有普遍的适用性。

### 2.3 基于应用层负载均衡调度的解决方法
多台服务器通过高速的互联网络连接成一个集群系统，在前端有一个基于应用层的负载调度器。当用户访问请求到达调度器时，请求会提交给作负载均衡调度的应用程 序，分析请求，根据各个服务器的负载情况，选出一台服务器，重写请求并向选出的服务器访问，取得结果后，再返回给用户。

应用层负载均衡调度 的典型代表有Zeus负载调度器[4]、pWeb[5]、Reverse-Proxy[6]和SWEB[7]等。Zeus负载调度器是Zeus公司的商业 产品，它是在Zeus Web服务器程序改写而成的，采用单进程事件驱动的服务器结构。pWeb就是一个基于Apache 1.1服务器程序改写而成的并行WEB调度程序，当一个HTTP请求到达时，pWeb会选出一个服务器，重写请求并向这个服务器发出改写后的请求，等结果 返回后，再将结果转发给客户。Reverse-Proxy利用Apache 1.3.1中的Proxy模块和Rewrite模块实现一个可伸缩WEB服务器，它与pWeb的不同之处在于它要先从Proxy的cache中查找后，若 没有这个副本，再选一台服务器，向服务器发送请求，再将服务器返回的结果转发给客户。SWEB是利用HTTP中的redirect错误代码，将客户请求到 达一台WEB服务器后，这个WEB服务器根据自己的负载情况，自己处理请求，或者通过redirect错误代码将客户引到另一台WEB服务器，以实现一个 可伸缩的WEB服务器。

基于应用层负载均衡调度的多服务器解决方法也存在一些问题。第一，系统处理开销特别大，致使系统的伸缩性有限。当请 求到达负载均衡调度器至处理结束时，调度器需要进行四次从核心到用户空间或从用户空间到核心空间的上下文切换和内存复制；需要进行二次TCP连接，一次是 从用户到调度器，另一次是从调度器到真实服务器；需要对请求进行分析和重写。这些处理都需要不小的ＣＰＵ、内存和网络等资源开销，且处理时间长。所构成系 统的性能不能接近线性增加的，一般服务器组增至3或4台时，调度器本身可能会成为新的瓶颈。所以，这种基于应用层负载均衡调度的方法的伸缩性极其有限。第 二，基于应用层的负载均衡调度器对于不同的应用，需要写不同的调度器。以上几个系统都是基于HTTP协议，若对于FTP、Mail、POP3等应用，都需 要重写调度器。

### 2.4 基于IP层负载均衡调度的解决方法
用户通过虚拟IP地址（Virtual IP Address）访问服务时，访问请求的报文会到达负载调度器，由它进行负载均衡调度，从一组真实服务器选出一个，将报文的目标地址Virtual IP Address改写成选定服务器的地址，报文的目标端口改写成选定服务器的相应端口，最后将报文发送给选定的服务器。真实服务器的回应报文经过负载调度器 时，将报文的源地址和源端口改为Virtual IP Address和相应的端口，再把报文发给用户。Berkeley的MagicRouter[8]、Cisco的LocalDirector、 Alteon的ACEDirector和F5的Big/IP等都是使用网络地址转换方法。MagicRouter是在Linux 1.3版本上应用快速报文插入技术，使得进行负载均衡调度的用户进程访问网络设备接近核心空间的速度，降低了上下文切换的处理开销，但并不彻底，它只是研 究的原型系统，没有成为有用的系统存活下来。Cisco的LocalDirector、Alteon的ACEDirector和F5的Big/IP是非常 昂贵的商品化系统，它们支持部分TCP/UDP协议，有些在ICMP处理上存在问题。

IBM的TCP Router[9]使用修改过的网络地址转换方法在SP/2系统实现可伸缩的WEB服务器。TCP Router修改请求报文的目标地址并把它转发给选出的服务器，服务器能把响应报文的源地址置为TCP Router地址而非自己的地址。这种方法的好处是响应报文可以直接返回给客户，坏处是每台服务器的操作系统内核都需要修改。IBM的 NetDispatcher[10]是TCP Router的后继者，它将报文转发给服务器，而服务器在non-ARP的设备配置路由器的地址。这种方法与LVS集群中的VS/DR类似，它具有很高的 可伸缩性，但一套在IBM SP/2和NetDispatcher需要上百万美金。总的来说，IBM的技术还挺不错的。

在贝尔实验室的 ONE-IP[11]中，每台服务器都独立的IP地址，但都用IP Alias配置上同一VIP地址，采用路由和广播两种方法分发请求，服务器收到请求后按VIP地址处理请求，并以VIP为源地址返回结果。这种方法也是为 了避免回应报文的重写，但是每台服务器用IP Alias配置上同一VIP地址，会导致地址冲突，有些操作系统会出现网络失效。通过广播分发请求，同样需要修改服务器操作系统的源码来过滤报文，使得只 有一台服务器处理广播来的请求。

微软的Windows NT负载均衡服务（Windows NT Load Balancing Service，WLBS）[12]是1998年底收购Valence Research公司获得的，它与ONE-IP中的基于本地过滤方法一样。WLBS作为过滤器运行在网卡驱动程序和TCP/IP协议栈之间，获得目标地址 为VIP的报文，它的过滤算法检查报文的源IP地址和端口号，保证只有一台服务器将报文交给上一层处理。但是，当有新结点加入和有结点失效时，所有服务器 需要协商一个新的过滤算法，这会导致所有有Session的连接中断。同时，WLBS需要所有的服务器有相同的配置，如网卡速度和处理能力。

## 3 通过NAT实现虚拟服务器（VS/NAT）
由于IPv4中IP地址空间的日益紧张和安全方面的原因，很多网络使用保留IP地址（10.0.0.0/255.0.0.0、 172.16.0.0/255.128.0.0和192.168.0.0/255.255.0.0）[64, 65, 66]。这些地址不在Internet上使用，而是专门为内部网络预留的。当内部网络中的主机要访问Internet或被Internet访问时，就需要 采用网络地址转换（Network Address Translation, 以下简称NAT），将内部地址转化为Internets上可用的外部地址。NAT的工作原理是报文头（目标地址、源地址和端口等）被正确改写后，客户相信 它们连接一个IP地址，而不同IP地址的服务器组也认为它们是与客户直接相连的。由此，可以用NAT方法将不同IP地址的并行网络服务变成在一个IP地址 上的一个虚拟服务。

VS/NAT的体系结构如图2所示。在一组服务器前有一个调度器，它们是通过Switch/HUB相连接的。这些服务器 提供相同的网络服务、相同的内容，即不管请求被发送到哪一台服务器，执行结果是一样的。服务的内容可以复制到每台服务器的本地硬盘上，可以通过网络文件系 统（如NFS）共享，也可以通过一个分布式文件系统来提供。
!["图2：VS/NAT的体系结构"|center](/images/1478136573503.png)

客户通过Virtual IP Address（虚拟服务的IP地址）访问网络服务时，请求报文到达调度器，调度器根据连接调度算法从一组真实服务器中选出一台服务器，将报文的目标地址 Virtual IP Address改写成选定服务器的地址，报文的目标端口改写成选定服务器的相应端口，最后将修改后的报文发送给选出的服务器。同时，调度器在连接Hash 表中记录这个连接，当这个连接的下一个报文到达时，从连接Hash表中可以得到原选定服务器的地址和端口，进行同样的改写操作，并将报文传给原选定的服务 器。当来自真实服务器的响应报文经过调度器时，调度器将报文的源地址和源端口改为Virtual IP Address和相应的端口，再把报文发给用户。我们在连接上引入一个状态机，不同的报文会使得连接处于不同的状态，不同的状态有不同的超时值。在TCP 连接中，根据标准的TCP有限状态机进行状态迁移，这里我们不一一叙述，请参见W. Richard Stevens的《TCP/IP Illustrated Volume I》；在UDP中，我们只设置一个UDP状态。不同状态的超时值是可以设置的，在缺省情况下，SYN状态的超时为1分钟，ESTABLISHED状态的超 时为15分钟，FIN状态的超时为1分钟；UDP状态的超时为5分钟。当连接终止或超时，调度器将这个连接从连接Hash表中删除。

这样，客户所看到的只是在Virtual IP Address上提供的服务，而服务器集群的结构对用户是透明的。对改写后的报文，应用增量调整Checksum的算法调整TCP Checksum的值，避免了扫描整个报文来计算Checksum的开销。

在 一些网络服务中，它们将IP地址或者端口号在报文的数据中传送，若我们只对报文头的IP地址和端口号作转换，这样就会出现不一致性，服务会中断。所以，针 对这些服务，需要编写相应的应用模块来转换报文数据中的IP地址或者端口号。我们所知道有这个问题的网络服务有FTP、IRC、H.323、 CUSeeMe、Real Audio、Real Video、Vxtreme / Vosiac、VDOLive、VIVOActive、True Speech、RSTP、PPTP、StreamWorks、NTT AudioLink、NTT SoftwareVision、Yamaha MIDPlug、iChat Pager、Quake和Diablo。

下面，举个例子来进一步说明VS/NAT，如图3所示：
!["图3：VS/NAT的例子"|center](/images/1478136640306.png)

VS/NAT的配置如下表所示，所有到IP地址为202.103.106.5和端口为80的流量都被负载均衡地调度的真实服务器172.16.0.2:80和 172.16.0.3:8000上。目标地址为202.103.106.5:21的报文被转移到172.16.0.3:21上。而到其他端口的报文将被拒绝。

| Protocol  | Virtual IP Address | Port | Real IP Address | Port | Weight |
| :-------- | --------:| :--: |
| TCP	    | 202.103.106.5 | 80 | 172.16.0.2 | 80 | 1 |
|           |               |    | 172.16.0.3 | 80 | 2 |
| TCP       | 202.103.106.5 | 21 | 172.16.0.3 | 21 | 1 |

从以下的例子中，我们可以更详细地了解报文改写的流程。
访问Web服务的报文可能有以下的源地址和目标地址：

| SOURCE	| 202.100.1.2:3456	| DEST	| 202.103.106.5:80 |
| :-------- | --------:| :--: |

调度器从调度列表中选出一台服务器，例如是172.16.0.3:8000。该报文会被改写为如下地址，并将它发送给选出的服务器。

| SOURCE    |     	202.100.1.2:3456|   DEST | 	172.16.0.3:8000 |
| :-------- | --------:| :------: |

从服务器返回到调度器的响应报文如下：

| SOURCE	|     172.16.0.3:8000|   DEST	|202.100.1.2:3456 |
| :-------- | --------:| :------: |

响应报文的源地址会被改写为虚拟服务的地址，再将报文发送给客户：

| SOURCE	|     202.103.106.5:80|   DEST	| 202.100.1.2:3456 |
| :-------- | --------:| :------: |

这样，客户认为是从202.103.106.5:80服务得到正确的响应，而不会知道该请求是服务器172.16.0.2还是服务器172.16.0.3处理的。

## 4 通过IP隧道实现虚拟服务器（VS/TUN）

在VS/NAT 的集群系统中，请求和响应的数据报文都需要通过负载调度器，当真实服务器的数目在10台和20台之间时，负载调度器将成为整个集群系统的新瓶颈。大多数 Internet服务都有这样的特点：请求报文较短而响应报文往往包含大量的数据。如果能将请求和响应分开处理，即在负载调度器中只负责调度请求而响应直 接返回给客户，将极大地提高整个集群系统的吞吐量。

IP隧道（IP tunneling）是将一个IP报文封装在另一个IP报文的技术，这可以使得目标为一个IP地址的数据报文能被封装和转发到另一个IP地址。IP隧道技 术亦称为IP封装技术（IP encapsulation）。IP隧道主要用于移动主机和虚拟私有网络（Virtual Private Network），在其中隧道都是静态建立的，隧道一端有一个IP地址，另一端也有唯一的IP地址。

我们利用IP隧道技术将请求报文封装转 发给后端服务器，响应报文能从后端服务器直接返回给客户。但在这里，后端服务器有一组而非一个，所以我们不可能静态地建立一一对应的隧道，而是动态地选择 一台服务器，将请求报文封装和转发给选出的服务器。这样，我们可以利用IP隧道的原理将一组服务器上的网络服务组成在一个IP地址上的虚拟网络服务。 VS/TUN的体系结构如图4所示，各个服务器将VIP地址配置在自己的IP隧道设备上。
![图4：VS/TUN的体系结构|center](/images/1478137499989.png)

VS/TUN 的工作流程如图5所示：它的连接调度和管理与VS/NAT中的一样，只是它的报文转发方法不同。调度器根据各个服务器的负载情况，动态地选择一台服务器， 将请求报文封装在另一个IP报文中，再将封装后的IP报文转发给选出的服务器；服务器收到报文后，先将报文解封获得原来目标地址为VIP的报文，服务器发 现VIP地址被配置在本地的IP隧道设备上，所以就处理这个请求，然后根据路由表将响应报文直接返回给客户。
![图5：VS/TUN的工作流程|center](/images/1478137535676.png)
在这里需要指出，根据缺省的TCP/IP协议栈处理，请求报文的目标地址为VIP，响应报文的源地址肯定也为VIP，所以响应报文不需要作任何修改，可以直接返回给客户，客户认为得到正常的服务，而不会知道究竟是哪一台服务器处理的。
![图6：半连接的TCP有限状态机|center](/images/1478137588931.png)

## 5 通过直接路由实现虚拟服务器（VS/DR）
跟VS/TUN 方法相同，VS/DR利用大多数Internet服务的非对称特点，负载调度器中只负责调度请求，而服务器直接将响应返回给客户，可以极大地提高整个集群 系统的吞吐量。该方法与IBM的NetDispatcher产品中使用的方法类似（其中服务器上的IP地址配置方法是相似的），但IBM的 NetDispatcher是非常昂贵的商品化产品，我们也不知道它内部所使用的机制，其中有些是IBM的专利。

VS/DR的体系结构如图 7所示：调度器和服务器组都必须在物理上有一个网卡通过不分断的局域网相连，如通过高速的交换机或者HUB相连。VIP地址为调度器和服务器组共享，调度 器配置的VIP地址是对外可见的，用于接收虚拟服务的请求报文；所有的服务器把VIP地址配置在各自的Non-ARP网络设备上，它对外面是不可见的，只 是用于处理目标地址为VIP的网络请求。
![图7：VS/DR的体系结构|center](/images/1478137677651.png)
VS/DR 的工作流程如图8所示：它的连接调度和管理与VS/NAT和VS/TUN中的一样，它的报文转发方法又有不同，将报文直接路由给目标服务器。在VS/DR 中，调度器根据各个服务器的负载情况，动态地选择一台服务器，不修改也不封装IP报文，而是将数据帧的MAC地址改为选出服务器的MAC地址，再将修改后 的数据帧在与服务器组的局域网上发送。因为数据帧的MAC地址是选出的服务器，所以服务器肯定可以收到这个数据帧，从中可以获得该IP报文。当服务器发现 报文的目标地址VIP是在本地的网络设备上，服务器处理这个报文，然后根据路由表将响应报文直接返回给客户。
![图8：VS/DR的工作流程|center](/images/1478137860518.png)
在VS/DR中，根据缺省的TCP/IP协议栈处理，请求报文的目标地址为VIP，响应报文的源地址肯定也为VIP，所以响应报文不需要作任何修改，可以直接返回给客户，客户认为得到正常的服务，而不会知道是哪一台服务器处理的。
VS/DR负载调度器跟VS/TUN一样只处于从客户到服务器的半连接中，按照半连接的TCP有限状态机进行状态迁移。

## 6 三种方法的优缺点比较
三种IP负载均衡技术的优缺点归纳在下表中：

|   _  |     VS/NAT |   VS/TUN  | VS/DR |
| :-------- | --------:| :------: |
| Server|   any	|  Tunneling	| Non-arp device |
| server network	|   private	|  LAN/WAN| LAN |
| server number|   low (10~20)|  High (100)| High (100) |
| server gateway|   load balancer|  own router| Own router |

> **注：** 以上三种方法所能支持最大服务器数目的估计是假设调度器使用100M网卡，调度器的硬件配置与后端服务器的硬件配置相同，而且是对一般Web服务。使用更高的硬件配置（如千兆网卡和更快的处理器）作为调度器，调度器所能调度的服务器数量会相应增加。当应用不同时，服务器的数目也会相应地改变。所以，以上数 据估计主要是为三种方法的伸缩性进行量化比较。

### 6.1 Virtual Server via NAT
VS/NAT 的优点是服务器可以运行任何支持TCP/IP的操作系统，它只需要一个IP地址配置在调度器上，服务器组可以用私有的IP地址。缺点是它的伸缩能力有限， 当服务器结点数目升到20时，调度器本身有可能成为系统的新瓶颈，因为在VS/NAT中请求和响应报文都需要通过负载调度器。 我们在Pentium 166 处理器的主机上测得重写报文的平均延时为60us，性能更高的处理器上延时会短一些。假设TCP报文的平均长度为536 Bytes，则调度器的最大吞吐量为8.93 MBytes/s. 我们再假设每台服务器的吞吐量为800KBytes/s，这样一个调度器可以带动10台服务器。（注：这是很早以前测得的数据）

基于 VS/NAT的的集群系统可以适合许多服务器的性能要求。如果负载调度器成为系统新的瓶颈，可以有三种方法解决这个问题：混合方法、VS/TUN和 VS/DR。在DNS混合集群系统中，有若干个VS/NAT负载调度器，每个负载调度器带自己的服务器集群，同时这些负载调度器又通过RR-DNS组成简 单的域名。但VS/TUN和VS/DR是提高系统吞吐量的更好方法。

对于那些将IP地址或者端口号在报文数据中传送的网络服务，需要编写相应的应用模块来转换报文数据中的IP地址或者端口号。这会带来实现的工作量，同时应用模块检查报文的开销会降低系统的吞吐率。

### 6.2 Virtual Server via IP Tunneling
在VS/TUN 的集群系统中，负载调度器只将请求调度到不同的后端服务器，后端服务器将应答的数据直接返回给用户。这样，负载调度器就可以处理大量的请求，它甚至可以调 度百台以上的服务器（同等规模的服务器），而它不会成为系统的瓶颈。即使负载调度器只有100Mbps的全双工网卡，整个系统的最大吞吐量可超过 1Gbps。所以，VS/TUN可以极大地增加负载调度器调度的服务器数量。VS/TUN调度器可以调度上百台服务器，而它本身不会成为系统的瓶颈，可以 用来构建高性能的超级服务器。

VS/TUN技术对服务器有要求，即所有的服务器必须支持“IP Tunneling”或者“IP Encapsulation”协议。目前，VS/TUN的后端服务器主要运行Linux操作系统，我们没对其他操作系统进行测试。因为“IP Tunneling”正成为各个操作系统的标准协议，所以VS/TUN应该会适用运行其他操作系统的后端服务器。

### 6.3 Virtual Server via Direct Routing
跟VS/TUN方法一样，VS/DR调度器只处理客户到服务器端的连接，响应数据可以直接从独立的网络路由返回给客户。这可以极大地提高LVS集群系统的伸缩性。

跟VS/TUN相比，这种方法没有IP隧道的开销，但是要求负载调度器与实际服务器都有一块网卡连在同一物理网段上，服务器网络设备（或者设备别名）不作ARP响应，或者能将报文重定向（Redirect）到本地的Socket端口上。

## 7 小结
本文主要讲述了LVS集群中的三种IP负载均衡技术。在分析网络地址转换方法（VS/NAT）的缺点和网络服务的非对称性的基础上，我们给出了通过IP隧道实现虚拟服务器的方法VS/TUN，和通过直接路由实现虚拟服务器的方法VS/DR，极大地提高了系统的伸缩性。


参考文献

1. Eric Dean Katz, Michelle Butler, and Robert McGrath, "A Scalable HTTP Server: The NCSA Prototype", Computer Networks and ISDN Systems, pp155-163, 1994.
2. Thomas T. Kwan, Robert E. McGrath, and Daniel A. Reed, "NCSA's World Wide Web Server: Design and Performance", IEEE Computer, pp68-74, November 1995.
3. C. Yoshikawa, B. Chun, P. Eastham, A. Vahdat, T. Anderson, and D. Culler. Using Smart Clients to Build Scalable Services. In Proceedings of the 1997 USENIX Technical Conference, January 1997.
4. Zeus Technology, Inc. Zeus Load Balancer v1.1 User Guide. http://www.zeus.com/
5. Edward Walker, "pWEB - A Parallel Web Server Harness", http://www.ihpc.nus.edu.sg/STAFF/edward/pweb.html, April, 1997.
6. Ralf S.Engelschall. Load Balancing Your Web Site: Practical Approaches for Distributing HTTP Traffic. Web Techniques Magazine, Volume 3, Issue 5, http://www.webtechniques.com, May 1998.
7. Daniel Andresen, Tao Yang, Oscar H. Ibarra. Towards a Scalable Distributed WWW Server on Workstation Clusters. In Proceedings of 10th IEEE International Symposium Of Parallel Processing (IPPS'96), pp.850-856, http://www.cs.ucsb.edu-/Research/rapid_sweb/SWEB.html, April 1996.
8. Eric Anderson, Dave Patterson, and Eric Brewer, "The Magicrouter: an Application of Fast Packet Interposing", http://www.cs.berkeley.edu/~eanders-/magicrouter/, May, 1996.
9. D. Dias, W. Kish, R. Mukherjee, and R. Tewari. A Scalable and Highly Available Server. In Proceeding of COMPCON 1996, IEEE-CS Press, Santa Clara, CA, USA, Febuary 1996, pp. 85-92.
10. Guerney D.H. Hunt, German S. Goldszmidt, Richard P. King, and Rajat Mukherjee. Network Dispatcher: a connection router for scalable Internet services. In Proceedings of the 7th International WWW Conference, Brisbane, Australia, April 1998.
11. Om P. Damani, P. Emerald Chung, Yennun Huang, "ONE-IP: Techniques for Hosting a Service on a Cluster of Machines", http://www.cs.utexas.edu-/users/damani/, August 1997.
12. Microsoft Corporation. Microsoft Windows NT Load Balancing Service. http://www.micorsoft.com/ntserver/NTServerEnterprise/exec/feature/WLBS/.

>关于作者
章文嵩博士，开放源码及Linux内核的开发者，著名的Linux集群项目－－LVS(Linux Virtual Server)的创始人和主要开发人员。他目前工作于国家并行与分布式处理重点实验室，主要从事集群技术、操作系统、对象存储与数据库的研究。他一直在自 由软件的开发上花费大量时间，并以此为乐。
