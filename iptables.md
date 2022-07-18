iptables set-xmark 解释：
参考： https://blog.csdn.net/qq_31977125/article/details/124208384


执行顺序：
参考： https://www.jianshu.com/p/15bd803e3bb8


iptables链路表：
![iptables_workflow](https://user-images.githubusercontent.com/310284/179182094-5eb25e50-c7fc-4aab-8790-067be53a37f9.png)



iptables 扩展组件：
https://ipset.netfilter.org/iptables-extensions.man.html


iptables查看某个module的帮助
```
iptables -m addrtype --help
```

ipvs与netfiler交互
http://www.austintek.com/LVS/LVS-HOWTO/HOWTO/LVS-HOWTO.filter_rules.html

![交互](http://www.austintek.com/LVS/LVS-HOWTO/HOWTO/images/nf-lvs.png)

ipvs与iptables优先级
> Modules register with a priority, the lowest priority getting to look at the packets first. LVS registers itself with a higher priority than iptables rules, and thus iptables will get the packet first and then LVS

所以，ipvs转发时：PREROUTING->INPUT(iptables)->INPUT(lvs)->POSTROUTING


## iptables例子解释
```
-A KUBE-SERVICES -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS
```
解释：
如果目标地址是LOCAL的，则jump到KUBE-NODEPORTS。
什么是LOCAL？本地所有网卡上的IP地址，都属于LOCAL。换言之，如果是发往本机器（ip与网卡上任一匹配），则jump
