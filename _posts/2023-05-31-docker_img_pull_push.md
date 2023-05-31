---
layout: post
author: joy717
---

1. 外网拉取镜像并打成压缩包
2. 上传压缩包到内网环境，之后解压并push到内网镜像仓库

## 拉取镜像并保存成一个压缩包

有外网权限的地方：

执行以下脚本，需要在同目录创建一个a.txt文件，里面包含各种镜像名称。
```
#!/bin/bash
while IFS= read -r line; do
  if [ ! -z ${line} ]
  then
    filename=${line##*/}
  docker pull ${line}
  docker save ${line} -o ${filename}.tgz
  fi
done < a.txt
tar -cvzf target_imgs.tar  *.tgz
```

a.txt example
```
longhornio/longhorn-engine:v1.1.2
longhornio/longhorn-manager:v1.1.2
longhornio/longhorn-ui:v1.1.2
longhornio/longhorn-instance-manager:v1_20210621
longhornio/longhorn-share-manager:v1_20210416
longhornio/backing-image-manager:v1_20210422
longhornio/csi-attacher:v2.2.1-lh2
longhornio/csi-provisioner:v1.6.0-lh2
longhornio/csi-node-driver-registrar:v1.2.0-lh1
longhornio/csi-resizer:v0.5.1-lh2
longhornio/csi-snapshotter:v2.1.1-lh2
 
quay.io/kubevirt/virt-operator:v0.41.0
quay.io/kubevirt/virt-controller:v0.41.0
quay.io/kubevirt/virt-api:v0.41.0
quay.io/kubevirt/virt-handler:v0.41.0
quay.io/kubevirt/virt-launcher:v0.41.0
quay.io/kubevirt/virtio-container-disk:v0.41.0
 
quay.io/kubevirt/cdi-operator:v1.34.0
quay.io/kubevirt/cdi-controller:v1.34.0
quay.io/kubevirt/cdi-importer:v1.34.0
quay.io/kubevirt/cdi-apiserver:v1.34.0
quay.io/kubevirt/cdi-uploadproxy:v1.34.0
quay.io/kubevirt/cdi-cloner:v1.34.0
quay.io/kubevirt/cdi-uploadserver:v1.34.0
 
bitnami/kubectl:1.18.6
rancher/harvester:v0.2.0
rancher/harvester-network-controller:v0.1.6
nfvpe/multus:v3.6
k8s.gcr.io/sig-storage/snapshot-controller:v4.0.0
```

## Load镜像并Push到相应仓库

上传上一步的镜像压缩包到内网环境

执行以下脚本，也需要在脚本同一目录包含一个镜像列表文件 a.txt（同上面示例）

以下脚本会修改镜像名称，把k8s.gcr.io 以及 quay.io 开头的字符串去掉。

可按自己需求调整脚本。

```
#!/bin/bash
tar -zxvf target_imgs.tar
cd target_imgs
if [ ! -d $1]
then
  cd $1
fi
tars=($(ls))
for i in "${tars[@]}"; do
  if [ "${i: -4}" == ".tgz" ]
  then
    docker load -i $i
  fi
done
 
while IFS= read -r line; do
  imgTrimd=${line#quay.io/}
  if [ ! -z "${imgTrimd}" ]
  then
    imgTrimd=${imgTrimd#k8s.gcr.io/}
    #修改成自己需要的地址
    newImg="1.2.3.4:10043/vin/${imgTrimd}"
    echo "processing for....${newImg}"
    docker tag "${line}" "${newImg}" && docker push "${newImg}"
  fi
done < ../a.txt
```
