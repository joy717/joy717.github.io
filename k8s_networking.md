# 图解Kubernetes网络（一）
【编者的话】本文阐述了Kubernetes网络模型，并详细描述了Kubernetes Pods在节点内和节点间的通信方式，帮助读者在碰到Kubernetes网络问题时从容应对。

你一直在Kubernetes集群中运行一系列服务并已从中获益，或者你正打算这么做。尽管有一系列工具能帮助你建立并管理集群，你仍困惑于集群底层是如何工作的，以及出现问题该如何处理。我曾经就是这样的。

![1](/blog/images/1.png "1")
诚然Kubernetes对初学者来说已足够易用，但我们仍然不得不承认，它的底层实现异常复杂。Kubernetes由许多部件组成，如果你想对失败场景做好应对准备，那么你必须知道各部件是如何协调工作的。其中一个最复杂，甚至可以说是最关键的部件就是网络。

因此我着手精确理解Kubernetes网络是如何工作的。我阅读了许多文章，看了很多演讲，甚至浏览了代码库。以下就是我的所得。
# Kubernetes网络模型
核心点是，Kubernetes网络有一个重要的基本设计原则：

**每个Pod拥有唯一的IP。**

这个Pod IP被该Pod内的所有容器共享，并且其它所有Pod都可以路由到该Pod。你可曾注意到，你的Kubernetes节点上运行着一些"pause"容器？它们被称作“沙盒容器（sandbox containers）"，其唯一任务是保留并持有一个网络命名空间（netns），该命名空间被Pod内所有容器共享。通过这种方式，即使一个容器死掉，新的容器创建出来代替这个容器，Pod IP也不会改变。这种IP-per-pod模型的巨大优势是，Pod和底层主机不会有IP或者端口冲突。我们不用担心应用使用了什么端口。

这点满足后，Kubernetes唯一的要求是，这些Pod IP可被其它所有Pod访问，不管那些Pod在哪个节点。
## 节点内通信

第一步是确保同一节点上的Pod可以相互通信，然后可以扩展到跨节点通信、internet上的通信，等等。
![Kubernetes Node（root network namespace）](/blog/images/2.png "Kubernetes Node（root network namespace）")
*Kubernetes Node（root network namespace）*

在每个Kubernetes节点（本场景指的是Linux机器）上，都有一个根（root）命名空间（root是作为基准，而不是超级用户）--root netns。

最主要的网络接口 <font color="red"><span style='background: pink'>eth0</span></font> 就是在这个root netns下。

![3](/blog/images/3.png "3")
*Kubernetes Node（pod network namespace）*

类似的，每个Pod都有其自身的netns，通过一个虚拟的以太网对连接到root netns。这基本上就是一个管道对，一端在root netns内，另一端在Pod的nens内。

我们把Pod端的网络接口叫 <font color="red"><span style='background: pink'>eth0</span></font>，这样Pod就不需要知道底层主机，它认为它拥有自己的根网络设备。另一端命名成比如 <font color="red"><span style='background: pink'>vethxxx</span></font>。你可以用<font color="red"><span style='background: pink'>ifconfig</span></font> 或者 <font color="red"><span style='background: pink'>ip a</span></font>命令列出你的节点上的所有这些接口。

![Kubernetes Node（linux network bridge）](/blog/images/4.png "Kubernetes Node（linux network bridge）")
*Kubernetes Node（linux network bridge）*

节点上的所有Pod都会完成这个过程。这些Pod要相互通信，就要用到linux的以太网桥 <font color="red"><span style='background: pink'>cbr0</span></font> 了。Docker使用了类似的网桥，称为<font color="red"><span style='background: pink'>docker0</span></font>。

你可以用 <font color="red"><span style='background: pink'>brctl show</span></font> 命令列出所有网桥。

![Kubernetes Node（same node pod-to-pod communication）](/blog/images/5.gif "Kubernetes Node（same node pod-to-pod communication）")
*Kubernetes Node（same node pod-to-pod communication）*

假设一个网络数据包要由<font color="red"><span style='background: pink'>pod1</span></font>到<font color="red"><span style='background: pink'>pod2</span></font>。
1. 它由pod1中netns的<font color="red"><span style='background: pink'>eth0</span></font>网口离开，通过<font color="red"><span style='background: pink'>vethxxx</span></font>进入root netns。
2. 然后被传到<font color="red"><span style='background: pink'>cbr0</span></font>，<font color="red"><span style='background: pink'>cbr0</span></font>使用ARP请求，说“谁拥有这个IP”，从而发现目标地址。
3. <font color="red"><span style='background: pink'>vethyyy</span></font>说它有这个IP，因此网桥就知道了往哪里转发这个包。
4. 数据包到达<font color="red"><span style='background: pink'>vethyyy</span></font>，跨过管道对，到达<font color="red"><span style='background: pink'>pod2</span></font>的netns。

这就是同一节点内容器间通信的流程。当然也可以用其它方式，但是无疑这是最简单的方式，同时也是Docker采用的方式。

## 节点间通信
正如我前面提到，Pod也需要跨节点可达。Kubernetes不关心如何实现。我们可以使用L2（ARP跨节点），L3（IP路由跨节点，就像云提供商的路由表），Overlay网络，或者甚至信鸽。无所谓，只要流量能到达另一个节点的期望Pod就好。每个节点都为Pod IPs分配了唯一的CIDR块（一段IP地址范围），因此每个Pod都拥有唯一的IP，不会和其它节点上的Pod冲突。

大多数情况下，特别是在云环境上，云提供商的路由表就能确保数据包到达正确的目的地。我们在每个节点上建立正确的路由也能达到同样的目的。许多其它的网络插件通过自己的方式达到这个目的。

这里我们有两个节点，与之前看到的类似。每个节点有不同的网络命名空间、网络接口以及网桥。

![Kubernetes Nodes with route table（cross node pod-to-pod communication）](/blog/images/6.gif "Kubernetes Nodes with route table（cross node pod-to-pod communication）")
*Kubernetes Nodes with route table（cross node pod-to-pod communication）*

假设一个数据包要从<font color="red"><span style='background: pink'>pod1</span></font>到达<font color="red"><span style='background: pink'>pod4</span></font>（在不同的节点上）。
1. 它由<font color="red"><span style='background: pink'>pod1</span></font>中netns的<font color="red"><span style='background: pink'>eth0</span></font>网口离开，通过<font color="red"><span style='background: pink'>vethxxx</span></font>进入root netns。
2. 然后被传到<font color="red"><span style='background: pink'>cbr0</span></font>，<font color="red"><span style='background: pink'>cbr0</span></font>通过发送ARP请求来找到目标地址。
3. 本节点上没有Pod拥有<font color="red"><span style='background: pink'>pod4</span></font>的IP地址，因此数据包由<font color="red"><span style='background: pink'>cbr0</span></font> 传到 主网络接口 <font color="red"><span style='background: pink'>eth0</span></font>.
4. 数据包的源地址为<font color="red"><span style='background: pink'>pod1</span></font>，目标地址为<font color="red"><span style='background: pink'>pod4</span></font>，它以这种方式离开<font color="red"><span style='background: pink'>node1</span></font>进入电缆。
5. 路由表有每个节点的CIDR块的路由设定，它把数据包路由到CIDR块包含<font color="red"><span style='background: pink'>pod4</span></font>的IP的节点。
6. 因此数据包到达了<font color="red"><span style='background: pink'>node2</span></font>的主网络接口<font color="red"><span style='background: pink'>eth0</span></font>。现在即使<font color="red"><span style='background: pink'>pod4</span></font>不是<font color="red"><span style='background: pink'>eth0</span></font>的IP，数据包也仍然能转发到<font color="red"><span style='background: pink'>cbr0</span></font>，因为节点配置了IP forwarding enabled。节点的路由表寻找任意能匹配<font color="red"><span style='background: pink'>pod4</span></font> IP的路由。它发现了 <font color="red"><span style='background: pink'>cbr0</span></font> 是这个节点的CIDR块的目标地址。你可以用<font color="red"><span style='background: pink'>route -n</span></font>命令列出该节点的路由表，它会显示<font color="red"><span style='background: pink'>cbr0</span></font>的路由，类型如下：
![7](/blog/images/7.png "7")
7. 网桥接收了数据包，发送ARP请求，发现目标IP属于<font color="red"><span style='background: pink'>vethyyy</span></font>。
8. 数据包跨过管道对到达<font color="red"><span style='background: pink'>pod4</span></font>。

这就是Kubernetes网络的基础。下次你碰到问题，务必先检查这些网桥和路由表。

#图解Kubernetes网络（二）
【编者的话】本文是Kubernetes网络系列的第二部分，详细阐述了Overlay网络在Kubernetes中的工作模式，并指出Overlay不是必须的，只有当我们知道为什么要使用Overlay模式时才使用它。

![1](/blog/images/8.png "1")

这篇文章的前一部分，我们漫谈了Kubernetes的网络模型。我们观察了数据包是如何在同一节点上的pod 间和跨节点的 pod 间流动的。我们也注意到了Linux网桥和路由表在这个过程中所扮演的角色。

今天我们将进一步阐述这些概念，并阐述Overlay网络是如何工作的。我们也将理解Kubernetes千变万化的Pod是如何从运行的应用中抽象出来，并在幕后处理的。

## Overlay 网络
Overlay网络不是默认必须的，但是它们在特定场景下非常有用。比如当我们没有足够的IP空间，或者网络无法处理额外路由，抑或当我们需要Overlay提供的某些额外管理特性。一个常见的场景是当云提供商的路由表能处理的路由数是有限制时。例如，AWS路由表最多支持50条路由才不至于影响网络性能。因此如果我们有超过50个Kubernetes节点，AWS路由表将不够。这种情况下，使用Overlay网络将帮到我们。

本质上来说，Overlay就是在跨节点的本地网络上的包中再封装一层包。你可能不想使用Overlay网络，因为它会带来由封装和解封所有报文引起的时延和复杂度开销。通常这是不必要的，因此我们应当在知道为什么我们需要它时才使用它。

为了理解Overlay网络中流量的流向，我们拿Flannel做例子，它是CoreOS 的一个开源项目。

![Kubernetes Node with route table（cross node pod-to-pop Traffic flow with flannel overlay network）](/blog/images/9.gif "Kubernetes Node with route table（cross node pod-to-pop Traffic flow with flannel overlay network）")
*Kubernetes Node with route table（cross node pod-to-pop Traffic flow with flannel overlay network）*

这里我们注意到它和之前我们看到的设施是一样的，只是在root netns中新增了一个虚拟的以太网设备，称为flannel0。它是虚拟扩展网络Virtual Extensible LAN（VXLAN）的一种实现，但是在Linux上，它只是另一个网络接口。

从<font color="red"><span style='background: pink'>pod1</span></font>到<font color="red"><span style='background: pink'>pod4</span></font>（在不同节点）的数据包的流向类似如下：

  1、它由<font color="red"><span style='background: pink'>pod1</span></font>中netns的<font color="red"><span style='background: pink'>eth0</span></font>网口离开，通过<font color="red"><span style='background: pink'>vethxxx</span></font>进入root netns。

  2、然后被传到<font color="red"><span style='background: pink'>cbr0</span></font>，<font color="red"><span style='background: pink'>cbr0</span></font>通过发送ARP请求来找到目标地址。

  3a、由于本节点上没有Pod拥有<font color="red"><span style='background: pink'>pod4</span></font>的IP地址，因此网桥把数据包发送给了<font color="red"><span style='background: pink'>flannel0</span></font>，因为节点的路由表上<font color="red"><span style='background: pink'>flannel0</span></font>被配成了Pod网段的目标地址。

  3b、flanneld daemon和Kubernetes apiserver或者底层的etcd通信，它知道所有的Pod IP，并且知道它们在哪个节点上。因此Flannel创建了Pod IP和Node IP之间的映射（在用户空间）。

<font color="red"><span style='background: pink'>flannel0</span></font>取到这个包，并在其上再用一个UDP包封装起来，该UDP包头部的源和目的IP分别被改成了对应节点的IP，然后发送这个新包到特定的VXLAN端口（通常是8472）。

![Packet-in-packet encapsulation（notice the packet is encapsulated from 3c to 6b in previous diagram）](/blog/images/10.png "Packet-in-packet encapsulation（notice the packet is encapsulated from 3c to 6b in previous diagram）")

*Packet-in-packet encapsulation（notice the packet is encapsulated from 3c to 6b in previous diagram）*

尽管这个映射发生在用户空间，真实的封装以及数据的流动发生在内核空间，因此仍然是很快的。

  3c、封装后的包通过<font color="red"><span style='background: pink'>eth0</span></font>发送出去，因为它涉及了节点间的路由流量。

  4、包带着节点IP信息作为源和目的地址离开本节点。

  5、云提供商的路由表已经知道了如何在节点间发送报文，因此该报文被发送到目标地址<font color="red"><span style='background: pink'>node2</span></font>。

  6a、包到达<font color="red"><span style='background: pink'>node2</span></font>的<font color="red"><span style='background: pink'>eth0</span></font>网卡，由于目标端口是特定的VXLAN端口，内核将报文发送给了 <font color="red"><span style='background: pink'>flannel0</span></font>。

  6b、<font color="red"><span style='background: pink'>flannel0</span></font>解封报文，并将其发送到 root 命名空间下。

从这里开始，报文的路径和我们之前在Part 1 中看到的非Overlay网络就是一致的了。

  6c、由于IP forwarding开启着，内核按照路由表将报文转发给了<font color="red"><span style='background: pink'>cbr0</span></font>。

  7、网桥获取到了包，发送ARP请求，发现目标IP属于<font color="red"><span style='background: pink'>vethyyy</span></font>。

  8、包跨过管道对到达<font color="red"><span style='background: pink'>pod4</span></font>。

这就是Kubernetes中Overlay网络的工作方式，虽然不同的实现还是会有细微的差别。有个常见的误区是，当我们使用Kubernetes，我们就不得不使用Overlay网路。事实是，这完全依赖于特定场景。因此请确保在确实需要的场景下才使用。

本部分到此结束。在前一部分我们学习了Kubernetes网络的基础知识。现在我们知道了Overlay网络是如何工作的。在下一部分，我们将看到Pod创建和删除过程中网络变化是如何发生的，以及出站和进站流量是如何流动的。

## 第三部分，就不搬了。请参照底下链接。

### 原文链接：
[中文翻译链接1](http://dockone.io/article/3211)

[中文翻译链接2](http://dockone.io/article/3212)

[中文翻译链接3](http://dockone.io/article/8436)

[英文原文链接1](https://medium.com/@ApsOps/an-illustrated-guide-to-kubernetes-networking-part-1-d1ede3322727)

[英文原文链接2](https://medium.com/@ApsOps/an-illustrated-guide-to-kubernetes-networking-part-2-13fdc6c4e24c)

[英文原文链接3](https://itnext.io/an-illustrated-guide-to-kubernetes-networking-part-3-f35957784c8e)
