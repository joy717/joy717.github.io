---
 categories:
   - volume
---
云主机上扩容系统盘成功，但在系统内，只在磁盘上扩容了，没有在系统分区上进行扩容。

# 现象

通过lsblk命令查看，可以看到/dev/vda大小变了，但/dev/vda1大小没变。

通过 df -Th 命令，查看 / 的容量没有改变。

# 解决

## 安装工具growpart（非必选）
```
# 添加阿里的yum repo
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo

yum clean all
yum makecache fast

#安装
yum -y install cloud-utils-growpart
```

## 常规（直接基于物理硬盘的块设备）

lsblk 查看mountPoint `/` type是`part`

1. 给分区扩容

```
# growpart <DeviceName> <PartionNumber>
growpart /dev/vda 1

# 如果遇到unexpected output in sfdisk --version [sfdisk，来自 util-linux 2.23.2] 错误，先执行以下命令，再次执行growpart命令
LANG=en_US.UTF-8

# 验证
lsblk
```

2. 给文件系统扩容

```
# 由df -Th 来确认系统盘的文件系统类型，这边以xfs为例. ext系列用resize2fs代替xfs_growfs
# xfs_growfs <mountpoint>
xfs_growfs /

# 验证
df -h
```

## 基于lvm的块设备

lsblk 查看mountPoint `/` type是`lvm`

### 没有growpart工具

1. 创建新分区

```
fdisk /dev/vda
# 按顺序输入命令
## n 创建新分区
## p 创建主分区
## 一路enter
## w 保存修改

###假设创建的新分区为 /dev/vda3

## 将新分区同步给系统
partprobe
## 验证
lsblk
```

2. 调整lvm相关

```
# 创建pv
pvcreate /dev/vda3
# 查询当前vg name （一般为centos）
vgdisplay
# 扩容vg
vgextend centos /dev/vda3
# 查询可用的空间 （这边假设系统盘从100G扩容到200G，即可用空间为100G）
vgdisplay
# 查询系统盘，lv当前的path。（一般为 /dev/centos/root）
lvdisplay
# 扩容lv
lvresize -L +100G /dev/centos/root
## 或者
lvresize -l +100%FREE /dev/centos/root

#验证
lvdisplay
lsblk
```

3. 扩容文件系统

```
# 由df -Th 来确认系统盘的文件系统类型，这边以xfs为例. ext系列用resize2fs代替xfs_growfs
# xfs_growfs <mountpoint>
xfs_growfs /

# 验证
df -hT
```

### 使用growpart工具(推荐)

1. 扩容分区

```
# growpart <DeviceName> <PartionNumber>
growpart /dev/vda 2

# 如果遇到unexpected output in sfdisk --version [sfdisk，来自 util-linux 2.23.2] 错误，先执行以下命令，再次执行growpart命令
LANG=en_US.UTF-8

# 验证
lsblk
```

2. 调整lvm相关

```
# 将vda2对应的pv重新适配下大小
pvresize /dev/vda2

# 验证pv的大小
# pvdisplay

# vgdisplay查看可用空间，以及vg名称
# vgdisplay

# 查看系统盘lv（一般为/dev/centos/root）
# lvdisplay

# 扩容lv
lvresize -l +100%FREE /dev/centos/root
# 验证
lvdisplay
lsblk
```

3. 扩容文件系统

```
# 由df -Th 来确认系统盘的文件系统类型，这边以xfs为例. ext系列用resize2fs代替xfs_growfs
# xfs_growfs <mountpoint>
xfs_growfs /

# 验证
df -hT
```
