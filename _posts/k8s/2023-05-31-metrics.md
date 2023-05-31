kubectl top node:

kubectl->metricServer->kubelet: /metrics/resource->cadvisor

kubectl top 从cadvisor里面获取
cAdvisor会创建一个名字为"/"的容器作为根容器，其他容器隶属于这个根容器，kubectl top node，会从这个根容器（"/"）获取节点信息
cAdvisor数据获取来源：cgroup。 （有层级结构。）



top/free 命令：
可以从 man free查看各个数值是怎么计算的，其中关心的一个数值是cache
cache=cache+slab. (centos)
