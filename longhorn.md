## csi相关
创建出pvc
csi-provisioner监控到新pvc(provisioner虽然是多例，但会选举出一个实例进行服务)
->通过csi driver的socket调用本机csi-plugin
->调用longhorn-manager的svc创建lh volume
->volume_controller监控到新的lh volume
->volume_controller创建对应的engine/replica instance




## 设备关联：
instance-manager-e pod中，/var/run/目录下，创建一个pvc的socket.
->将这个socket作为iscsi的新建的lun对应的backstore/block下的设备.（即用tgtadm创建lun的时候， --backing-store参数）(即iscsi服务端创建一个lun)
->用nsenter命令，在宿主机上用iscsiadm login到对应的lun，此时宿主机上创建出/dev/vdxx
->用nsenter命令，在宿主机上创建出设备/dev/longhorn/pvc-xxxx, (mknod命令，将/dev/longhorn/pvc-xxxx,的major/minor number指定成跟/dev/vdxx的一样)
