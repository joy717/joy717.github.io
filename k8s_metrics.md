kubectl top node:

kubectl->metricServer->kubelet: /metrics/resource->cadvisor

kubectl top 从cadvisor里面获取



top/free 命令：
可以从 man free查看各个数值是怎么计算的，其中关心的一个数值是cache
cache=cache+slab. (centos)
