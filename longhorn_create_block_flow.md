## csi总体流程:

-> createVolume (结束后，pvc变成Bound状态)

-> ControllerPublishVolume (先创建volumeAttachment，再call。 结束后，Node .status.VolumesAttached会显示对应的VolumeAttachement)

-> NodeStageVolume (将卷格式化，并mount到staging目录。之所以多这步，是因为k8s允许一个卷被mount到多个pod里面。staging目录是一个节点上的全局目录，先将卷mount到此目录，再通过NodePublishVolume mount到pod的目录里。)

(但是longhorn这边，针对block，stage的时候，几乎没做事情，mount的逻辑都放在nodePublishVolume里面)

-> NodePublishVolume(将staging目录挂载到pod 目录里面)


版本： longhorn v1.2.2

## csi createVolume：

csi-plugin createVolume -> longhornManager daemon的http api的volumeCreate(longhorn-manager router.go)

longhornManager daemon 各种controller在listAndWatch：

### volume_controller sync

-> v.ownerID = handyops-1

-> v.status.CurrentImage = v.spec.engineImage

-> v.status.CurrentNodeID = v.spec.nodeID (由于v.spec.nodeID 为"",因此这边依然为空)

-> 创建engine的crd (desireState stoped, e.spec.engineImage = v.status.currentImage)

-> 创建replicas的crd(desireState stoped, r.spec.engineImage = v.status.currentImage, r.spec.active = true)

-> 调度replicas到对应的disk (r.spec.nodeID = disk.NodeID, r.spec.DataDirectoryName赋值, r.spec.DiskID/DiskPath 赋值)

-> v.status.condition.schduled = true

-> v.status.state=creating

-> v.Status.Robustness=unknown

-------> volume_controller:1299 (等待engine stop)

### engine_controller sync

-> e.status.ownerID = handyops-1

-> e.status.started = false

-> e.status.currentState = stopped, e.status.currentImage = "", e.status.ip = "", e.status.port = 0

--------> engine_controller, instance_handler:206 (等待e.spec.nodeID 不为"")

### replicas_controller sync

-> r.status.ownerID = handyops-1

-> r.status.started = false

-> r.status.currentState = stopped, r.status.currentImage = "", r.status.ip = "", r.status.port = 0

### volume_controller sync

-> v.status.state = detaching (engines, replicas 如果status.currentState不是stop状态，则返回，保留此状态，等待下次sync)

-> v.status.state = detached

总结：此时v.status.state = detached, e.status.currentState = stopped, r.status.currentState = stopped, v.spec.nodeID = "", v.status.currentNodeId = "", e.spec.nodeID = "", r.spec.nodeID = "handyops-1"


## controllerPublishVolume:

manager/volume.go Attach

v.spec.nodeID = handyops-1, v.spec.DisableFrontend 赋值, v.spec.LastAttachedBy赋值(实际为""，因为csi那边没有传)

### volume_controller sync:

-> v.Status.CurrentNodeID = v.Spec.NodeID

-> v.Status.State = attaching

-> e.Spec.UpgradedReplicaAddressMap 初始化空map

-> r.Spec.DesireState = running

-------> volume_controller:1393 (等待replica running状态)

### replica_controller sync:

-> createInstance (创建replica的process)

下一次sync(等待process状态, 正常这次应该是starting或者running状态，取决于此次sync，process是否已经running，并且被instance-manager同步)

-> r.status.InstanceManagerName = im.name(instanceManagerName)

-> r.status.CurrentState = starting, r.status.CurrentImage = "", r.status.IP = "", r.status.Port = 0

下一次sync(process running状态)

-> r.status.started = true

-> r.status.CurrentState = running, r.status.CurrentImage = r.spec.EngineImage, r.status.IP = im.status.IP, r.status.Port = int(instance.Status.PortStart)

### volume_controller sync:

-> e.Spec.NodeID = v.Status.CurrentNodeID, e.Spec.ReplicaAddressMap = replicaAddressMap(replicas的地址map), e.Spec.DesireState = types.InstanceStateRunning, e.Spec.DisableFrontend = v.Status.FrontendDisabled, e.Spec.Frontend = v.Spec.Frontend

-------->volume_controller:1434 (等待engine running)

### engine_controller sync:

e.Status.CurrentReplicaAddressMap = e.Spec.ReplicaAddressMap

-> createInstance (创建engine的process)

下一次sync(等待process状态, 正常这次应该是starting或者running状态，取决于此次sync，process是否已经running，并且被instance-manager同步)

-> e.status.InstanceManagerName = im.name(instanceManagerName)

-> e.status.CurrentState = starting, e.status.CurrentImage = "", e.status.IP = "", e.status.Port = 0

下一次sync(process running状态)

-> e.status.started = true

-> e.status.CurrentState = running, e.status.CurrentImage = e.spec.EngineImage, e.status.IP = im.status.IP, e.status.Port = int(instance.Status.PortStart)

-> (start engine monitor, 同步backup/restore/snapshot/clone/expand相关信息) ec.engineMonitorMap[e.name] = stopCh

-> e.Status.Endpoint = endpoint

-> e.Status.CurrentSize, e.Status.IsExpanding, e.Status.LastExpansionError, e.Status.LastExpansionFailedAt 赋值

-> e.Status.RebuildStatus = rebuildStatus(空{})

-> e.Status.BackupStatus = backupStatusList(空{})

-> e.Status.PurgeStatus = purgeStatus

-> e.Status.ReplicaModeMap = currentReplicaModeMap

### volume_controller sync:

-> v.Status.State = attached

总结： 此时一切正常，v.Status.State = attached, e.status.CurrentState = running, r.status.CurrentState = running, 各种相关的nodeID也有值。


## nodeStageVolume:
对于block设备，几乎不做事情.

## nodePublishVolume:
将/dev/longhorn/pvc-xxxx 格式化并 mount到 /var/lib/kubelet/plugins/kubernetes.io/csi/volumeDevice/pvc-xxxx 




instance_manager sync
每一秒查询一次，instance-manager pod里面的processList，更新到status里面
