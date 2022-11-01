## 背景
k8s LB类型的service，在多网卡环境下，通过externalIP访问，有时候无法正常访问。

## 排查
经tcpdump抓包后发现，集群外访问不通的时候，此ip所在的节点有收到sync的请求包，但没有返回sync的响应包。

进一步发现，请求包的目的mac地址为"fa:c4:55:76:3e:01"，即ip地址为172.32.4.114的eth1的网卡。与预期的不符合，应该为br_eth0的mac地址才对。

查看metallb的代码，发现metallb会将所有running的，带ip地址的网卡过滤出来，返回arp响应。这意味着收到的arp响应的客户端，`有可能`会获取到错误的mac地址。（即前面遇到的情况）

实际问题为：metallb无法判断哪张网卡才是他需要做响应的网卡。因此需要告知metallb相应的网卡信息。

## 解决
上metallb的github社区查询issue，发现有此问题的issue。https://github.com/metallb/metallb/issues/277

此问题在v0.13.6版本修复。

因此解决方法

1、可以将metallb升级到v0.13.6之后的版本（2022.11.1的最新版本为v0.13.7）

2、在旧版本的metallb代码上，做补丁，进行修改。增加一个网卡列表的参数，传给metallb，metallb在获取网卡mac地址时候，只针对该参数里的网卡做响应。
