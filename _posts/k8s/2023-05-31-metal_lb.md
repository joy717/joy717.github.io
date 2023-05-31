# MetalLB 如何接管IP池

MetalLB只是简单的声明了一个IP池，就可以直接拿来用，而不需要修改网卡信息，这个是怎么做到的呢？

答案是：`ARP`。当ARP请求某个IP池里的IP的时候，MetalLB将所在节点的Mac地址作为响应，发了出去。（有点类似ARP代理）

因此，此IP相关的请求包会发到当前节点上。此后，就交给kube-proxy将请求分流到对应的pod上去了。

如果外面的网络，希望通过IP池里面的IP访问集群的话，则需要节点网络之上的路由/交换机能将此IP路由到此网段中，然后由MetalLB响应对应IP的ARP。

如果集群内部的网络，希望通过IP池里面的IP访问的话，则直接通过kube-proxy进行访问。


# 架构
MetalLB有2个组件，Controller跟Speaker。

Controller负责k8s 的svc资源的修改，将IP池里的IP选择出来，设置进去。

Speaker主要负责对应IP地址的ARP响应，(通过[ARP开源库](https://github.com/mdlayher/arp)），以及维护一个[MemberList](https://github.com/hashicorp/memberlist)。

MemberList是一个开源组件，负责将节点都加入到一个List里面，同时剔除失去响应的节点。保证MemberList里面都是可用的节点。

# 流程：
1. 所有节点启动一个Pod Speaker(hostNetwork模式)，同时Speaker所在的节点作为一个member加入到MemberList里面。
2. 当有LB类型的svc创建/更新时（通过client-go去watch相关资源），Controller组件从IP池里面选一个IP设置给这个SVC。

   同时，作为leader的那个Speaker，将这个SVC跟IP地址记录到内存中。（即IP地址由作为leader的Speaker接管）其他的Speaker忽略此信息。
> 这边的选举算法为：是否为MemberList的第一个member，如果是，则为leader。
3. 这之后，如果有此IP的ARP请求，所有的Speaker都在自己的内存中查找是否有相应的记录（实际上只有leader节点有），来选择是否发送ARP响应（以自己节点的MAC地址）。

* 当作为leader的Speaker挂了之后，会触发强制同步的策略，将所有的svc再过一遍，执行步骤2。
