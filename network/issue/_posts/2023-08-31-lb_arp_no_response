# 现象
lb svc对应的ip，丢包严重，大部分时候，在集群外网络不可访问。
网络拓扑为 一个bond底下，有多个vlan的子网卡，在这些子网卡上，分别建一个对应的网桥。lb绑定的网卡，在这些网桥上。

具体的网络情况为：
lb的ip在节点1（metallb响应在该节点），对应的后端pod在节点2。
lb对应的网卡为br_bond0_45

# 原因

访问lb:port，
通过tcpdump抓包，发现pod所在的节点2通过vxlan.calico有正常返回响应包。
节点1有接收到vxlan.calico返回的响应包，
此时应该通过lb进来的网卡（br_bond0_45），将响应包（syn+ack）返回给集群外的client端。
但抓到的包里面，并没有这个以及之后的网络包。

尝试ping lb的ip。
通过tcpdump抓包，只有icmp请求，没有响应。


tcpdump中发现
发现在发响应包（syn+ack）给client端时候，会发arp请求网关的mac地址，arp的sourceIP是br_bond0_44上的ip（另外一个网段，由于arp_announce内核参数为2，因此这个ip是不对的）。
因此无法收到arp响应，进而无法知道网关的地址，而无法通过br_bond0_45与网关交互，发回响应包

> 172.31.13.0/24的为br_bond0_45的网卡， 172.31.12.6 为br_bond0_44上的ip

> sourceIP的选择与内核参数有关： `net.ipv4.conf.all.arp_announce`  相关文档：https://docs.kernel.org/networking/ip-sysctl.html?highlight=arp_announce

> 这边与arp相关的内核参数还有 `arp_filter`，`arp_ignore`。kube-proxy的strictARP参数会影响到这几个内核参数的value

# 解决
给br_bond0_45上配置对应网段的ip，请求网关的arp的sourceIP会直接使用该网卡上的ip，进而能正常进行arp响应。后续网络包也都正常。


# 负面影响
由于在节点上添加了ip，所以在默认路由表里面，会有添加一个ip段的路由，导致网络使用受限制。
比如默认网关是13网段，
此时加了一个14网段的ip，内核会自动添加14网段的路由规则。
当节点外14网段的client端，要访问该节点13网段的ip，此时网络会不通。（从13网卡进来，但出去的时候会走14的网卡出去）
