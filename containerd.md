k8s使用containerd作为cri，容器镜像通过dockerfile+docker来build，基础镜像为alpine
问题：

1. 文件明明存在，但提示文件`no found`或者`no such file or directory`, 原因可能是丢失lib库。
   dockerfile里面增加此链接： `RUN mkdir /lib64 && ln -s /lib/libc.musl-x86_64.so.1 /lib64/ld-linux-x86-64.so.2`
   
2. 什么是snapshoter
   当镜像拉取下来后，会是一个压缩的文件。将此文件解压到系统的某个目录里（比如overlayfs的snapshoter，对应的目录/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots)。方便容器启动后，直接以这个目录作为root文件系统。

