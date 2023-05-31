---
layout: post
author: joy717
---

libvirt: 一套虚拟的接口。（底层也会调用qemu）
qemu: 一套完整的虚拟化技术，可以独立工作，但性能比较差。也可以搭配kvm工作，性能比较好。
kvm: linux的一个内核模块，可以虚拟化出cpu、内存。当与qemu配合时，cpu、内存等硬件由kvm处理，起到加速作用。所以虚拟机的性能较好。
virsh: 一套命令行工具，方便查看/操作 libvirt接口下的一些资源

简单化的来看，可以约略看成：virsh(最上层)-->libvirt-->qemu-->kvm(最底层)
