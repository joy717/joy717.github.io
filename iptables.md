**iptables set-xmark 解释** https://blog.csdn.net/qq_31977125/article/details/124208384


**各种JUMP的执行顺序** https://www.jianshu.com/p/15bd803e3bb8


**kube-proxy iptable/ipvs 详解**

https://zhuanlan.zhihu.com/p/94418251

https://www.digihunch.com/2020/11/ipvs-iptables-and-kube-proxy/


**iptables链路表**

![iptables_workflow](https://user-images.githubusercontent.com/310284/179182094-5eb25e50-c7fc-4aab-8790-067be53a37f9.png)



**iptables 扩展组件**

https://ipset.netfilter.org/iptables-extensions.man.html


iptables查看某个module的帮助

```
iptables -m addrtype --help
```


**路由表与netfilter交互**

![image](https://user-images.githubusercontent.com/310284/179714017-02bc98aa-7752-44e4-b4e9-a1469d7ecdab.png)

> Linux路由表其实有2个主要概念：
> * 路由策略（rule）
> * 路由表（table）
> 
> Linux可以配置很多很多策略，数据包将依次通过各个策略，一旦匹配某个策略则进一步应用策略对应的路由表，如果当前路由表无法匹配到路由则继续执行后续策略匹配。



**ipvs与netfilter交互**

http://www.austintek.com/LVS/LVS-HOWTO/HOWTO/LVS-HOWTO.filter_rules.html

![交互](https://user-images.githubusercontent.com/310284/179713789-221ca15a-d952-4bb2-ab9b-45e159bc2191.png)

**ipvs与iptables优先级**

> Modules register with a priority, the lowest priority getting to look at the packets first. LVS registers itself with a higher priority than iptables rules, and thus iptables will get the packet first and then LVS

所以，ipvs转发时：PREROUTING->路由表->INPUT(iptables)->INPUT(lvs)->POSTROUTING


## iptables例子解释
```
-A KUBE-SERVICES -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS
```
解释：

如果目标地址是LOCAL的，则jump到KUBE-NODEPORTS。
什么是LOCAL？本地所有网卡上的IP地址，都属于LOCAL。换言之，如果是发往本机器（ip与网卡上任一匹配），则jump
