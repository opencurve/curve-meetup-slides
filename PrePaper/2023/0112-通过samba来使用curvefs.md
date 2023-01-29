# 通过samba来使用curvefs

本文将会介绍curvefs与samba以及如何将他们结合使用, 并且介绍samba的第三方moudle开发知识.

## curvefs介绍
Curve共享文件存储（简称CurveFS）是Curve社区开发的新一代分布式共享文件存储系统，在网易内部已有PB级规模应用，其核心架构可以参考设计文档：https://github.com/opencurve/curve/blob/master/docs/cn/curvefs_architecture.md

相比传统的分布式文件系统，CurveFS具备如下几个特色：
1. 针对AI训练/机器学习场景、elasticsearch/clickhouse/starrocks等大数据场景做了专门优化，支持内存缓存、本地磁盘缓存、分布式内存缓存（基于memcached或其他kv存储）等多级缓存，可对文件读写进行缓存加速并保持数据写入可靠性，支持数据预热到缓存中加速读取过程，大数据场景下使用curvefs+对象存储相比本地副本存储方案具备更高的性价比优势以及易用性优势
2. 基于multi-raft支持元数据的分区存储，具备无限扩展性，性能可随节点数线性扩展，不存在性能热点问题
3. 支持将文件数据存储到对象存储和CurveBS块存储上，可以做到冷热数据分层存储和生命周期管理（roadmap），数据容量可无限扩展

## Samba介绍

homepage: https://www.samba.org/

Samba是目前最流行的文件和打印机共享软件之一. 得益于它良好的兼容性, 可以轻松地跨平台(Windows/Linux/MacOS), 以及灵活的可定制性和可配置性, 受到了各个系统的广泛支持.

Samba的特性非常多, 其中对于第三方开发者来说比较重要的就是Samba VFS Module特性. 其让开发者可以自由地实现一些功能, 接下来我们的话题会围绕着Samba的基础设定以及如何为Samba添加新的VFS Module.

本文使用的Samba版本为4.10.16, 系统为CentOS 7.

## 如何配置Samba使用curvefs

### 配置curvefs

#### 部署curvefs

通过curveadm进行curvefs后端服务的部署, 参考以下文档

https://github.com/opencurve/curveadm/wiki/curvefs-cluster-deployment

#### 创建一个fs并挂载

通过curveadm进行curvefs的挂载, 参考以下文档, 如果fs不存在的话, 会自动进行创建

https://github.com/opencurve/curveadm/wiki/curvefs-client-deployment

### 安装配置samba服务

```bash
sudo yum update -y
sudo yum install samba -y
```

修改/etc/samba/smb.conf, 以下是一个example, 更多的配置请参阅`man smb.conf`

```ini
[global]
workgroup      = SAMBA
log level      = 0
max log size   = 0
security       = user
passdb backend = tdbsam
create mask    = 0700
directory mask = 0700
writable       = yes
public         = no
guest ok       = no

[curvefs]
path = /mnt/curvefs # 需要确保samba用户有访问这个目录的权限
```

配置user的密码, user需要是本地已有的linux用户, 这里以root举例

```bash
sudo smbpasswd -a root
```

启动samba server服务

```
sudo systemctl enable --now smb
```

然后就可以通过samba的client(windows/linux/macos)来连接访问

##  Samba VFS Module介绍

参考文档: `https://wiki.samba.org/index.php/Writing_a_Samba_VFS_Module`

Samba VFS Module是Samba VFS提供的一种功能扩展机制, 就像Extention之于Chomre, 非常重要.

1. Samba VFS提供了一系列的接口定义, 具体可查看结构体[struct vfs_fn_pointers](https://github.com/samba-team/samba/blob/samba-4.10.16/source3/include/vfs.h#L618), 也是使用了c语言比较常见的函数指针的方式来实现调用module的实现.
2. 每一个module都是一个Shared Library (.so), 它包含了对上面定义的部分或全部接口的实现, 通过修改配置文件指定module来进行调用, Samba会在client连接的时候加载so.

3. 多个VFS module可以stackable使用, 也就是如果多个module的实现不冲突, 可以修改配置, 按照顺序依次调用多个module的实现.

### module常见类型

#### module用于支持新的后端fs

默认的samba vfs最终会调用vfs system call来对接文件系统.

比如说你想要对接cephfs, 那么可能是需要先把cephfs挂载到本机上, 然后再通过vfs system call来对接samba, 相当于多了一层中间system call的开销.

如果你想避免中间system call的调用, 只需要开发一个直接对接ceph sdk的module, 来当作stackable调用链的最末端, 替代default的实现即可. 就可以实现不挂载cephfs, 直接将cephfs通过samba协议分享出去的效果.

对于别的fs比如GlusterFS, 也可以使用这样的方式来对接.

https://www.samba.org/samba/docs/current/man-html/vfs_ceph.8.html

https://www.samba.org/samba/docs/current/man-html/vfs_glusterfs.8.html

#### module用于扩充已有的fs的功能集

在已存在一个fs后端实现的情况下, (默认的是调用vfs system call), 对部分接口进行功能的增强.

module作为stackable调用链的中间一环, 完成本身module的功能后可以继续往后调用.

比如说fruit模块就提供了对Apple SMB clients更好的兼容性支持.

https://www.samba.org/samba/docs/current/man-html/vfs_fruit.8.html

## 开始开发你的第一个Samba VFS Moudle

#### 一个module包含什么?

现有的module都位于此目录下

https://github.com/samba-team/samba/tree/samba-4.10.16/source3/modules

一个module由代码实现和对应的wscript构建配置组成, samba使用WAF来构建

我们将以一个名为demo的vfs module举例, 我们会有如下变动

- 代码实现: source3/wscript b/source3/wscript/vfs_demo.c

- 改动source3/wscript b/source3/wscript和source3/modules/wscript_build增加构建配置

#### module代码实现

##### 模块的初始化以及生命周期

包括几个部分

- 模块的注册: 名称
- connect接口: 响应启用了此module的client的连接请求, 模块的初始化应该在此处完成, 包括自定义的context的构造等
- disconnect接口: 响应启用了此module的client的disconnect请求, 模块的退出清理应该在此处完成
- 如果connect和disconnect有改动的需求, 那么实现自己自定义的方法即可.

从connect到disconnect为一个module的生命周期.

下面的代码是一个简单的范例:

```c
// vfs_demo.c

// initialzation
NTSTATUS vfs_demo_init(TALLOC_CTX *ctx) {
  return smb_register_vfs(SMB_VFS_INTERFACE_VERSION, "demo",
                          &vfs_demo_fns); // vfs_demo_fns是接口实现函数指针的struct
}

/*
static struct vfs_fn_pointers vfs_cloudnas_quota_fns = {
   .connect_fn = demo_connect,
   .disconnect_fn = demo_disconnect,
   ... 还有别的fn
}
*/

// context
struct demo_ctx {
  uint64_t whoami;
}

// connect
static int demo_connect(vfs_handle_struct *handle,
                        const char *service, const char *user) {
  int res = 0;
	struct demo_ctx *ctx = NULL;
  /*
   * Allow the next module to handle connections first
   * If we get an error, don't do any of our initialization.
   */
  res = SMB_VFS_NEXT_CONNECT(handle, service, user);
  if (res) {
    return res;
  }

  ctx = talloc_zero(handle, struct demo_ctx);
  if (!ctx) {
    DEBUG(1, ("talloc_zero() failed\n"));
		errno = ENOMEM;
		return -1;
  }
  ctx->whoami = lp_parm_ulonglong(SNUM(handle->conn),
					       "demo",
					       "whoami",
					       0);
  SMB_VFS_HANDLE_SET_DATA(handle, ctx, NULL,
													struct demo_ctx, return -1);
  return res;
}

// disconnect
static void demo_disconnect(vfs_handle_struct *handle) {
  struct demo_ctx *ctx;
  
  SMB_VFS_NEXT_DISCONNECT(handle);
  
  SMB_VFS_HANDLE_GET_DATA(handle, ctx, struct demo_ctx,
                          return -1);
  DEBUG(1, ("%llu disconnect...", ctx->whoami));
}

```

##### 编码需要了解的常用知识

1. 如果你的代码不是整条stackable调用链上的最末端, 记得使用SMB_VFS_NEXT_XXX系列函数来维持调用链, 比如说SMB_VFS_NEXT_CONNECT和SMB_VFS_NEXT_DISCONNECT
2. conf中的模块自定义参数可以通过lp_parm_xxx系列函数来进行解析
3. 自定义的ctx结构体可以通过SMB_VFS_HANDLE_SET_DATA和SMB_VFS_HANDLE_GET_DATA设定与解析
4. share的路径存储在handle->conn->connectpath中

#### 使用C ABI兼容的代码

C是一个历史久远的底层语言, 基于C的开发经常会遇到缺少一些数据结构、一些方便的软件库的情况. 

基于更现代的语言的开发往往更具有便利性, 比如go, rust等等. 

为了结合两者的优点, 我们可以使用在代码里include对应头文件和链接对应的so的方式进行开发.

##### 需要注意的问题

- samba的client connect默认采用的是fork一个新的smbd进程来处理后续的动作, 对于go语言来说这可能会造成问题, 参见https://github.com/golang/go/issues/14767

#### 编译与安装

参考https://wiki.samba.org/index.php/Build_Samba_from_Source

简而言之

```bash
./configure
make
sudo make install
```


