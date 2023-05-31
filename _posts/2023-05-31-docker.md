### GraphDriver

管理docker镜像分层。将各个镜像层组织合并成一个视图。(概念上，非实际实现。其实就是定义了接口)

具体的drivers：Vfs、Aufs、Overlay、Overlay2、Btrfs、ZFS、Devicemapper和Windows

> 镜像分层：一个镜像分为N个层，每一层都是独立的，也是公共的。不同的镜像，可以直接复用镜像层。
> 一般为只读。容器运行时候，会多一层可读写层，在这层上，一般通过CopyOnWrite的方式，将只读层的内容复制到读写层，再做修改。

镜像分层，构建文件系统的时候，（docker run），将所有相关的镜像层，一般通过union filesystem，合并成一个视图。

> union filesystem(联合文件系统)，即联合挂载，在操作系统中，
> 联合挂载（union mounting）是一种将多个目录结合成一个目录的方式，这个目录看起来就像包含了他们结合的内容一样。
> 联合文件系统，并不是一个真正的文件系统（比如xfs,ext4等），他是在现有的文件系统的基础上提供的功能。
> 因此，对于文件系统是有要求的。docker会检查底层文件系统（xfs/ext4等）与联合文件系统（overlay2/aufs等）是否兼容。

