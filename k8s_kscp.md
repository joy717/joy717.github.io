# 前提
准备一个k8s集群，如果没有，参考[kubeadm部署k8s](deploy_k8s_kubeadm.md)

# 下载一致性e2e工具：sonobuoy

https://github.com/vmware-tanzu/sonobuoy/releases	

## sonobuoy用的镜像：(有镜像需要翻墙，如果该环境无法翻墙，可以到其他可翻墙的机器下载，用docker save/load的方式导入该环境)
sonobuoy images list --dry-run

# 建议可翻墙的环境，由于sonobuoy会去拉取一些k8s.io的镜像来跑测试。如果无法翻墙，会导致很多测试失败，同时测试时间也会啦的很长

# 开始跑测试
sonobuoy run --mode=certified-conformance

# 查询状态
sonobuoy status

# 查看日志
sonobuoy logs

# 获取结果，在./results目录

outfile=$(sonobuoy retrieve)
mkdir ./results; tar xzf $outfile -C ./results

# 清理sonobuoy
sonobuoy delete

# sonobuoy 使用：
https://github.com/cncf/k8s-conformance/blob/master/instructions.md	
