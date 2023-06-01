---
title: kubeadm部署k8s
---
# 修改hosts文件，每个节点都要
```
cat >> /etc/hosts << EOF
172.20.17.133 k8s-1
172.20.20.56 k8s-2
172.20.17.243 k8s-3
EOF
```

# 修改hostname，每个节点修改成自己的名字
```
hostnamectl set-hostname k8s-1
```

# ssh重新登录
exit

# ssh证书信任

## 节点k8s-1上操作
```
ssh-keygen -t rsa

ssh-copy-id -i /root/.ssh/id_rsa.pub root@k8s-1

ssh-copy-id -i /root/.ssh/id_rsa.pub root@k8s-2

ssh-copy-id -i /root/.ssh/id_rsa.pub root@k8s-3
```

# 修改yum仓库为阿里，每个节点都要
```
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo
```

```
cat >> /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
``` 
```
yum clean all
yum makecache fast
```

# 准备工作，每个节点都要
## 修改内核参数
```
systemctl stop firewalld && systemctl disable firewalld
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables=1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-ip6tables=1" >> /etc/sysctl.conf
sysctl -p
```

## 关闭swap
```
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## 关闭selinux
```
sed -i 's/^ *SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```

# 安装docker，每个节点都要
```
yum -y install docker-ce
systemctl enable docker --now
```


# 下载安装cri-dockerd，每个节点都要
https://github.com/Mirantis/cri-dockerd/releases
```
yum -y install cri-dockerd-0.2.5-3.el7.x86_64.rpm
```

## 修改/usr/lib/systemd/system/cri-docker.service的启动参数，添加
--pod-infra-container-image registry.aliyuncs.com/google_containers/pause:3.9

```
systemctl enable cri-docker --now
```

# 安装k8s相关的二进制程序，每个节点都要
```
yum -y install kubeadm kubectl kubelet
```

# 安装kubectl命令自动补充，每个节点都要（可选）
```
yum -y install bash-completion
echo 'source <(kubectl completion bash)' >>~/.bashrc
```

# 初始化第一个节点
```
kubeadm init \
    --apiserver-advertise-address=172.20.17.133 \
    --image-repository registry.aliyuncs.com/google_containers \
    --kubernetes-version v1.27.2 \
    --pod-network-cidr=10.244.0.0/16 \
    --cri-socket unix:///var/run/cri-dockerd.sock
```    
    
# 其他节点join到集群内，到需要加入到集群的节点上执行。如果需要作为control-plane，则加上参数 --control-plane。token跟discovery-token-ca-cert-hash的内容根据kubeadm init输出结果来修改。
```
kubeadm join 172.20.17.133:6443 --token 8i3cwl.zm3rqluggkanqoa8 \
	--discovery-token-ca-cert-hash sha256:ec2dd9a2aa690ee6a9d8055ecee0f17f3495b2ce2cf3415c9e78fd2e6f0e98a0  --cri-socket unix:///var/run/cri-dockerd.sock
```

# copy kube config，节点1上执行即可。
```
cd && mkdir .kube && cp /etc/kubernetes/admin.conf ~/.kube/config
```

# 安装flannel，https://github.com/flannel-io/flannel#deploying-flannel-manually。节点1执行一次即可。
```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
