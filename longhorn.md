## csi相关
1. 创建出pvc
2. csi-provisioner监控到新pvc(provisioner虽然是多例，但会选举出一个实例进行服务)
3. 通过csi driver的socket调用本机csi-plugin
4. 调用longhorn-manager的svc创建lh volume
5. volume_controller监控到新的lh volume
6. volume_controller创建对应的engine/replica instance




## 设备关联：
1. instance-manager-e pod中，/var/run/目录下，创建一个pvc的socket.
2. 将这个socket作为iscsi的新建的lun对应的backstore/block下的设备.（即用tgtadm创建lun的时候， --backing-store参数）(即iscsi服务端创建一个lun)
3. 用nsenter命令，在宿主机上用iscsiadm login到对应的lun，此时宿主机上创建出/dev/vdxx
4. 用nsenter命令，在宿主机上创建出设备/dev/longhorn/pvc-xxxx, (mknod命令，将/dev/longhorn/pvc-xxxx,的major/minor number指定成跟/dev/vdxx的一样)

### instance manager:
instance-manager pod里面包含：
1. grpc服务（负责pod内process的管理）
2. cli工具，数据封装/解封，调用步骤1的grpc服务，创建process
process是用longhorn-engine（lonhorn的engineimage对应的二进制）这个cli启动的.
longhorn-manager监听各种crd状态，通过调用instance-manager里的grpc服务，创建process
