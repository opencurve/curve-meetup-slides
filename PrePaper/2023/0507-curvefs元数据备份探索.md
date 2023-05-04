# curvefs元数据备份探索

<!-- vscode-markdown-toc -->
* 1. [背景](#)
* 2. [curvefs元数据有哪些？以及其特点](#curvefs)
	* 2.1. [mds的元数据](#mds)
		* 2.1.1. [存储方式](#-1)
		* 2.1.2. [元数据内容](#-1)
		* 2.1.3. [数据量估算](#-1)
	* 2.2. [metaserver的元数据](#metaserver)
		* 2.2.1. [存储方式](#-1)
		* 2.2.2. [元数据内容](#-1)
		* 2.2.3. [数据量估算](#-1)
* 3. [元数据备份](#-1)
	* 3.1. [mds元数据备份](#mds-1)
	* 3.2. [metaserver元数据备份](#metaserver-1)
		* 3.2.1. [存储目录结构](#-1)
		* 3.2.2. [方案1：human readable的元数据备份](#1humanreadable)
		* 3.2.3. [方案2：raw元数据备份](#2raw)
* 4. [总结](#-1)

<!-- vscode-markdown-toc-config
	numbering=true
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->

##  1. <a name=''></a>背景

对于一个大型存储系统，不存在可靠的组件这一说，硬盘、服务器都是消耗品，都有出故障的时候。因此为了保障数据的可靠性，可用性，出现了各种各种冗余、副本的设计，对于核心数据进行备份也是一种常见的想法。我们将探索一下curvefs元数据备份的思路和方案。

##  2. <a name='curvefs'></a>curvefs元数据有哪些？以及其特点

curvefs的管控面组件分为2种，一种是mds，另一种是metaserver，分别存储了各自负责的元数据。下面介绍一下各自的存储方式、数据内容以及数据量的大致估算。

###  2.1. <a name='mds'></a>mds的元数据

####  2.1.1. <a name='-1'></a>存储方式

mds的元数据持久化存储在etcd中

####  2.1.2. <a name='-1'></a>元数据内容

1. topology信息: fsid, pool, zone, server, copyset信息，具体proto定义可以查看https://github.com/opencurve/curve/blob/master/curvefs/proto/topology.proto
2. fsinfo信息: 包括s3和block存储后端的信息，具体proto定义可以查看https://github.com/opencurve/curve/blob/master/curvefs/proto/mds.proto
3. ChunkId: 目前已经分配的最大的chunk id, 单条kv记录

####  2.1.3. <a name='-1'></a>数据量估算

可以比较容易地发现，存储的信息大多是服务器、fs、copyset这类数量级一般在百/千级别的数据，并且单条记录的大小是固定的、较小的，所以数据量非常少，可能只有几兆几十兆。

###  2.2. <a name='metaserver'></a>metaserver的元数据

####  2.2.1. <a name='-1'></a>存储方式

metaserver的元数据持久化在本地硬盘的rocksdb上，数据的可靠性可用性通过三副本raft协议（使用的实现是braft）来保证，元数据请求->raft->rocksdb。

####  2.2.2. <a name='-1'></a>元数据内容

partition信息, inode信息, dentry信息等其余的所有元数据，具体proto定义可以查看 https://github.com/opencurve/curve/blob/master/curvefs/proto/metaserver.proto

####  2.2.3. <a name='-1'></a>数据量估算

考虑到fs的inode和dentry信息往往会达到上亿的量级，并且inode信息中需要记录文件数据位置到s3对象或bs后端的映射，整个数据量会比较庞大。metaserver元数据量达到上百G是非常正常的事情。

##  3. <a name='-1'></a>元数据备份

###  3.1. <a name='mds-1'></a>mds元数据备份

由于后端数据实际存储使用的是etcd，所以可以用etcd可以用的所有备份手段。
可以参考https://etcd.io/docs/v3.5/op-guide/recovery/ ,这里不做过多的介绍

###  3.2. <a name='metaserver-1'></a>metaserver元数据备份

####  3.2.1. <a name='-1'></a>存储目录结构

这里只用显示了一个copyset，一般copysets下会有多个copyset，元数据会被拆分为多个copyset。
``` bash
root@curvefs-metaserver-689d7df09643:/curvefs/metaserver/data# tree -a
.
|-- copysets
|   `-- 4294967308
|       |-- raft_log
|       |   `-- log_meta
|       |-- raft_meta
|       |   `-- raft_meta
|       |-- raft_snapshot
|       |   `-- snapshot_00000000000000003221
|       |       |-- __raft_snapshot_meta
|       |       |-- conf.epoch
|       |       |-- metadata
|       |       `-- rocksdb_checkpoint
|       |           |-- 000029.blob
|       |           |-- 000030.blob
|       |           |-- 000031.sst
|       |           |-- 000032.sst
|       |           |-- 000033.sst
|       |           |-- 000034.sst
|       |           |-- 000035.blob
|       |           |-- 000036.blob
|       |           |-- CURRENT
|       |           |-- MANIFEST-000018
|       |           `-- OPTIONS-000025
|       `-- storage_data
|           |-- 000019.log
|           |-- 000029.blob
|           |-- 000030.blob
|           |-- 000035.blob
|           |-- 000036.blob
|           |-- 000037.sst
|           |-- 000038.sst
|           |-- CURRENT
|           |-- IDENTITY
|           |-- LOCK
|           |-- LOG
|           |-- MANIFEST-000018
|           |-- OPTIONS-000023
|           `-- OPTIONS-000025
|-- metaserver.dat
`-- storage
```
可以发现raft_snapshot下的rocksb_checkpoint和storage_data中的数据有一部分是一样的，那是rocksdb的checkpoint机制使用hardlink导致的。raft默认为半个小时触发一次snapshot，从而触发生成一次rocksdb的checkpoint。

####  3.2.2. <a name='1humanreadable'></a>方案1：human readable的元数据备份

元数据是通过编码后存储在rocksdb中的，所以是难以直接阅读的。
通过将某个时间点的元数据解码并导出为human readable的格式（比如json）是非常容易想到的办法。
目前curvefs并不支持这样的元数据导出、导入的接口，有待后续开发，但是可以设想到的优缺点如下。
优点：
- 导出内容容易阅读和确认
- 方便控制备份的范围，可以指定某个文件夹、文件的导出
- 可以做到备份以外的事情，比如单独调整某个文件的元数据，导出元数据后修改再导入
缺点：
- 占用集群资源较多，在线的编解码需要metaserver介入，开销大
- 不适合备份大规模的文件夹或文件系统，耗时会非常久

####  3.2.3. <a name='2raw'></a>方案2：raw元数据备份

如果不导出fs的数据，而是直接备份编码后的raw数据，会有以下的优缺点。
优点：
- 不需要解码
缺点：
- 难以阅读和确认，需要开发额外的离线解码工具（暂无）
- 无法控制备份粒度
- 在线备份某个时间点的数据需要一个整体的snapshot动作，目前没有
目前后端为自己实现的raft+rocksdb的方案，数据可以分为两部分，raft snapshot的数据和不断在更新的数据。
简单介绍一下目前使用的braft的快照机制，每一个copyset（raft node）在创建的时候会初始化一个snapshot timer，在index有变化的情况下，接受主动的snapshot请求或是定时（半小时）快照。 
raw元数据备份的策略可以分为两种：
定期备份：
- 通过主动触发所有raft node的snapshot，然后备份所有的snapshot数据
实时备份：
- 通过lsyncd这样的工具，去跟踪所有的文件变更，相当于一个热的standby
- 需要注意lsyncd无法很好的处理metaserver中rocksdb checkpoint硬链接的场景。实际只是新增减了几个文件，但是如果使用lsyncd会把所有的内容重新删除复制一遍，需要自己开发简单的同步工具去处理一下这部分数据。

##  4. <a name='-1'></a>总结

本文介绍了curvefs中的元数据的存储方式，以及在此基础上的一些元数据备份的思考。现有的metaserver元数据备份方案只有raw元数据的定期和实时备份是可行的。后续可以考虑在以下几个方面进行跟进：
- 实现fs、inode元数据的在线导出和导入功能，以便实现细粒度的元数据备份和查看需求
- 实现离线的raw元数据编解码功能，便于备份数据的查看和处理
- 实现全局的snapshot功能，目前只能主动触发每一个raft node的snapshot来简单模拟，可能会有坑
