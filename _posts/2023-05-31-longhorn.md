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
3. 用nsenter命令，在宿主机上用iscsiadm login到对应的lun，此时宿主机上自动创建出/dev/vdxx
4. 用nsenter命令，在宿主机上创建出设备/dev/longhorn/pvc-xxxx, (mknod命令，将/dev/longhorn/pvc-xxxx,的major/minor number指定成跟/dev/vdxx的一样，即指向同一块设备)
5. csi-plugin, 将/dev/longhorn/pvc-xxxx挂载到/var/lib/kubelet/plugins/kubernetes.io/csi/volumeDevices/publish/pvc-xxxx
6. instance-manager-r pod所在节点，/var/lib/longhorn/replicas/pvc-xxx  最终落盘数据，存在此处

### instance manager:
instance-manager pod里面包含：
1. grpc服务（负责pod内process的管理）
2. cli工具，数据封装/解封，调用步骤1的grpc服务，创建process

process是用longhorn-engine（lonhorn的engineimage对应的二进制）这个cli启动的.

longhorn-manager监听各种crd状态，通过调用instance-manager里的grpc服务，创建process. process再走iscsi相关流程.

### pod内服务

代码项目longhorn-engine controller(engine) 提供grpc服务，控制volume、replicas、backup等。

代码项目longhonr-engine replica：
1. 提供grpc服务，控制replicas以及disk。 
2. 提供一个rpc服务，数据传输。

### 相关目录
/dev/xxxx  iscisadm登录后，产生的本地设备

/dev/longhorn/pvc-xxx longhorn设备. 通过mknod与iscisadm 登录后的设备同min/max number. csi-plugin会将此设备挂到/var/lib/kubelet/plugins/kubernetes.io/csi/volumeDevices/publish/pvc-xxxx.

/var/lib/longhorn/replicas/pvc-xxx  最终落盘数据，存在此处

instance-manager-e pod里面，/var/run/ 底下，有个对应pvc的socket. 作为iscsi的target lun.


## 数据流：
1. pod往pvc挂载的目录里面读写，以下以写为例
2. 对应pod目录为：/var/lib/kubelet/pods/xxxxx/volumeDevices/kubernetes.io~csi/pvc-xxxx
3. 上面的目录实际上，是一个软链接，实际为 /var/lib/kubelet/plugins/kubernetes.io/csi/volumeDevices/publish/pvc-xxxx
4. 步骤3的目录，实际为/dev/longhorn/pvc-xxxx的挂载点
5. /dev/longhorn/pvc-xxxx实际指向/dev/xxxx，即iscsiadm登录之后，创建出来的设备.(因为major/minor number一致)
6. 往/dev/xxxx设备里面写，会通过iscsi，写入到服务端 target的lun里面。而这个实际指向instance-manager-e里面的/var/run/xxxx.socket
7. instance-manager-e pod里面对应的controller进程，会监听这个socket，之后，通过dataconn.Server（server1），将读到的内容，转化成longhorn自己的一套协议，写入后端3个replicas. 这边也是通过dataConn，
只不过dataConn.client在controller进程里，对应的dataConn.Server（server2），在replicas进程.
8. instance-manager-r pod里面对应的replicas进程，监听一个独立的tcp端口，dataconn.Server（server2），将读到的内容，转义一下，写入server2的后端,真正的磁盘目录/var/lib/longhorn/replicas/pvc-xxxx
