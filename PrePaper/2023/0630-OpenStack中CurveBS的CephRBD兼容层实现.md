# OpenStack中CurveBS的CephRBD兼容层实现
## 背景与目标

OpenStack是目前广泛使用的资源管理平台，向用户提供包括镜像、云主机、云硬盘等在内的资源，而Ceph RBD是最为流行的OpenStack块存储后端之一，提供了可伸缩的存储能力。

CurveBS（后称CBD）是Curve社区推出的高性能块存储软件。OpenStack对于存储的访问是通过各个存储的驱动实现。CBD如果想要对接OpenStack, 那么需要维护对应的CBD存储驱动，改动包括glance, nova, cinder, libvirt和qemu。目前改动暂时并没有推到对应上游的计划，如果用户需要使用，需要自行修改代码并且编译上述的所有软件。

除了这一种做法，我们可以通过使用CBD的接口来模拟OpenStack中使用的部分RBD接口，来直接使用RBD驱动来调用CBD，这也就是Ceph RBD的兼容层，用户只需要替换RBD对应的librados和librbd动态库以及python3-rbd和python3-rados的实现就可以进行体验。如果后续有长期使用的需求，再将这种使用方式替换为CBD专属驱动。

## RBD和CBD的概念映射

在配置使用Ceph RBD时，我们往往需要指定几个配置，包括集群配置，逻辑池等。在CBD中，我们需要找到一种映射关系来正确合理地处理他们。

以下是概念的映射表：
```
rbd   -> cbd
pool  -> dir
user  -> owner
vol   -> vol
```

其中需要注意的是，CBD中目前没有类似ceph中pool的概念可以配置，但是可以创建dir，用以模拟RBD的pool；CBD在访问卷的时候，需要指定volume的owner，我们这里不传入passwd（sdk不支持），用以对应RBD的user概念。

## 兼容层实现

这里主要分为两块，一块是基于python开发的OpenStack的管控服务，一块是机器虚拟化的基石服务。

### 镜像/虚拟主机/虚拟硬盘的管理 Glance/Nova/Cinder

OpenStack组件使用python开发，对接了python3-rbd和python3-rados，来连接后端ceph集群。

涉及到rbd驱动的OpenStack主要的核心操作如下：

- 镜像（Glance）：
  - 上传镜像并打快照并设置保护flag
  - 下载镜像
- 虚拟硬盘（Cinder）：
  - 创建（新卷或克隆卷）
  - 删除
  - 调整大小
  - 打快照/克隆
- 虚拟主机（Nova）：
  - 创建（使用cinder卷或是自行创建系统卷，一般来源是镜像快照或硬盘快照）
  - 删除
  - 挂载卷

#### 兼容层工作

我们需要做的就是提供一个新的python3 module（rados、rbd）实现来替换现有的。

我们将用到CBD的python sdk库 https://github.com/opencurve/curve/tree/master/curvefs_python 来进行模拟，文档在 https://github.com/opencurve/curve/blob/master/docs/cn/curve-client-python-api.md

这里将开发中遇到的问题进行了记录，给后来人避坑。

1. 在python多进程（涉及到fork）的环境中，如果在父进程已经进行了CBDclient的初始化，由于fork只会拷贝调用线程，而CBDclient初始化时会创建rpc工作线程池，导致子进程异常，无法正常发送rpc和后端通信。这个问题主要出现在Nova中，nova会fork出N个worker，可以通过延迟初始化来绕过。
2. CBD的python模块的read/write相关接口使用的是str来接受bytes数据。在python2中byte和str的区分还不是很严格不会出问题，但是python3中str的使用是必须需要经过编解码的，接口使用str会导致数据的错误。
3. CBD现在的快照机制是将源卷的数据转储到S3兼容的对象存储，如果写入的数据很多，将耗时非常久。快照完成后，源卷和快照解耦。快照克隆时，如果不开启COW，将会需要将对象存储的数据再拷贝一次到CBD中，如果启用COW，那么后续IO有可能会需要走对象存储的IO栈，可能会很慢。Glance在镜像上传完成后会自动打一次快照，后续虚拟机的创建也受到这一问题的影响。
4. 由于CBD的python sdk依赖CBD的共享库，在各个平台需要单独编译和打包，目前并没有完善的打包，可能需要自己编译处理
5. CBD逻辑卷大小最小为10G，如果创建一个1G的卷，实际在CBD调用size()获得的仍是10G
6. 除了直接调用python库，可能会需要调用"ceph df", "ceph mon dump", "rbd status", "rbd import" 等命令，需要hook实现
7. 目前不支持上传COW镜像

### 虚拟机的运行 LibVirt/Qemu

宿主机虚拟机的管理和虚拟化靠libvirt和qemu，libvirt接收来自nova生成的xml，生成对应的qemu命令运行虚拟机。rbd驱动下，libvirt和qemu使用了ceph提供的librados.so和librbd.so共享库来对接后端的IO。

#### 兼容层工作

我们需要做的是提供一个新的动态库来模拟librbd和librados的接口，CBD侧我们将会使用libnebd-client来和通过nebd和CBD后端进行IO交互。nebd是一个用于解决client端热升级的组件，server端和CBD后端进行交互，client端和server端通信，升级时只需要升级重启nebd-server即可，不用重启client（在这里就是qemu）

这里将开发中遇到的问题进行了记录，给后来人避坑。

1. libnebd-client库的回调不支持传args，无法和rbd的接口兼容。通过修改libnebd-client来适配。
2. 由于libnebd-client中链接了brpc，对pthread造成了影响，需要在libvirtd的env中设置preload，不然会导致symbol找不到的问题。https://github.com/apache/brpc/issues/1086
3. 虚拟机中lsblk看到的硬盘大小是调用的后端size接口，由于前面提到的CBD逻辑卷大小最小为10G，小硬盘的显示可能不正确
4. qemu的rbd driver没有限制io对齐，允许任意大小的io，但是CBD要求IO必须4096/4K字节对齐，这个需要CBD进行后端支持，或者修改qemu的IO限制，默认应该是512字节。

## 简易操作指南

操作系统：debian11, qemu 5.2, libvirt 7.0

下面所需的文件都可以在这里下载：
https://github.com/h0hmj/curve/releases/download/cbd2rbd-beta-v1/openstack-cbd-mod.tar.gz

0. 部署一个curvebs后端集群
1. 计算节点上安装qemu和libvirt以及rbd的支持(qemu-block-extra, libvirt-daemon-driver-storage-rbd), 修改qemu的执行用户为root,关闭apparmor
2. 计算节点上安装nebd软件包，和部署nebd, 用于支持连接后端的curve集群
3. 替换librados2和librbd1包中的对应```librados.so```和```librbd.so```为我们魔改的so（用cbd模拟了rbd的接口，代码https://github.com/h0hmj/curve/commit/d1df07c86c38b5d0d16a025cb560977d3bfc568f）还有 ```libnebdclientmod.so```
4. 修改/etc/default/libvirtd，添加preload参数LD_PRELOAD=/usr/lib/nebd/libnebdclientmod.so
5. 使用kolla部署openstack, base镜像选用debian, 为glance、nova和cinder启用rbd支持，为了简单，使用user为admin，pool为curve对应的文件夹名称，如果需要从glance卷clone来创建vm，打开glance的选项show_image_direct_url = True 。在curve后端创建对应的文件夹。
6. 修改openstack nova-compute配置使用本机的libvirt
7. 为glance-api/nova-compute/cinder-volume的容器
    - apt install libunwind8
    - 添加/etc/curve/client.conf
    - 安装curvefs的wheel包
    - 删除自带的python-rbd/rados的so实现，替换为我们的魔改版
    - 替换/usr/bin/ceph和/usr/bin/rbd为我们的魔改版本
8. 重启所有相关服务
9. 后续可以通过horizon的web界面进行试用，镜像的创建删除，云主机的创建删除，云硬盘的创建删除等

## 总结

我们探索了一种可能的rbd兼容方式，即提供hook了对应接口的共享库，以及适配当中遇到的问题。这会导致失去原有的ceph rbd的支持，如果有需求，建议仍然使用单独的cbd驱动。
