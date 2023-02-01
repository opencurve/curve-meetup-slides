# Curve文件存储回收站功能介绍

Curve作为一个分布式存储系统，数据的安全性是非常重要的。为了保证数据的安全性，Curve提供了回收站功能，回收站功能可以将删除的文件放入回收站，用户可以在回收站中恢复删除的文件。回收站功能可以保证数据的安全性，同时也可以减少用户误删文件的损失。

Curve 当前的分布式块存储 CurveBS 和分布式文件存储 CurveFS 都实现了回收站功能。这篇文章为大家介绍一下 CurveFS 的回收站功能。如果大家有兴趣，下次再为大家介绍以下 CurveBS 的回收站功能。

Curve 文件系统的回收站功能，分为几个功能点进行进行开发。

* 最基础的回收站功能
* 提供修改回收站过期时间修改功能（待实现）
* 提供回收站过滤功能（待实现）
* 提供目录批量删除加速功能（待实现）
* 提供目录的原路径恢复（待实现）

## CurveFS回收站功能

当前 Curve的主分支已经合入了文件系统回收站功能的最基础功能。包含以下能力。

### 1、开启回收站功能

能够在创建文件系统的时候，通过配置指定是否开启回收站功能。

通过一个recycleTimeHour配置项，这个值为一个非负整数。如果这个配置项如果设置为0，表示这个fs不开启fs功能；如果这个配置项不为0，表示删除的文件进入回收站之后，在回收站保留的时间，超过这个时间之后，这个文件会从回收站进行清理。这个配置项默认为0，在`curve/curvefs/conf/tools.conf`。

```
# fs recycle, if set 0, disable fs recycle, delete files directly,
# if set not 0, enable fs recycle, delete files after a period of time
recycleTimeHour=0
```

如果通过 curveadm 工具挂载 Curve 文件系统，可以修改 client.yaml 文件，增加一项 recycleTimeHour。比如，修改回收站时间为删除文件后，文件在回收站保留1个小时，1个小时后自动被回收站清理，可以在 curveadm 挂载 fs 使用的 client.yaml 配置文件中，增加一行

```
recycleTimeHour: 1
```

### 2、查看回收站功能是否开启

可通过curvefs_tool工具查看文件系统的recycleTimeHour值。

```
curvefs_tool list-fs
{
 "fsInfo": [
  {
   "fsId": 2,
   "fsName": "fs2",
   "status": "INITED",
   "rootInodeId": "1",
   ......
   "recycleTimeHour": 1
  },
  {
   "fsId": 1,
   "fsName": "fs",
   "status": "INITED",
   "rootInodeId": "1",
   ......
   "recycleTimeHour": 0
  }
 ]
}

```

也可通过观察文件系统根目录下是否有一个.recycle目录。这里有两个挂挂载点，cwdir1挂载了一个curvefs，未开启回收站功能；cwdir2挂载了一个curvefs，开启了回收站功能。

```
未开启回收站
$ ls cwdir1/ -alh
total 0
开启了回收站
$ ls cwdir2/ -alh
total 0
drwxrwxrwt 2 root root 0 Jan 28 16:18 .recycle
```

### 3、删除的文件进入回收站

CurveFS文件系统开启回收站之后，会在根目录下生成一个.recyle的隐藏目录。这个隐藏目录平时是空的，当有文件进行删除的时候，会在 `.recycle`下生成文件删除时间点的目录，精确到小时(24小时制)，格式为 `年-月-日-小时` 。删除的文件，会进入对应时间的目录，为了方便将来对回收站的文件进行恢复，删除的目录也会进入回收站，而且进入回收站的文件/目录的名字也发生了变化。原名字 `name` 会变成 `parentInodeId_InodeId_name` 的形式。

直接`ls -a`可查看回收站下的文件。如果想

```
创建一个目录dir1，和目录下文件file1
~/cwdir2$ mkdir dir1
~/cwdir2$ touch dir1/file1

删除之后，查看原路径，目录和文件不存在了
~/cwdir2$ rm -r dir1
~/cwdir2$ ls -alh
total 0
drwxrwxrwt 3 root root 0 Jan 28 16:38 .recycle

查看回收站，多了一个目录，格式为 年-月-日-小时
~/cwdir2$ ls -alh .recycle/
total 0
drwxr-xr-x 3 root root 4.0K Jan 28 16:38 2023-01-28-16

时间目录下，有删除的文件和目录，名字加上了自己和父母的inodeid作为前缀
~/cwdir2$ ls -alh .recycle/2023-01-28-16/
total 0
drwxr-xr-x 2 root root 4.0K Jan 28 16:38 1_5242880_dir1
-rw-r--r-- 1 root root    0 Jan 28 16:38 5242880_2097152_file1
~/cwdir2$
```

### 4、CurveFS 回收站的恢复

回收站功能是为误删兜底的，当发现文件被误删了，可以删除时间对应的回收站目录中，找到需要恢复的文件，直接通过mv的方式，把文件挪出回收站。通过这种简单的方式，即可完成文件的恢复。

```
~/cwdir2$ ls -alh .recycle/2023-01-28-17/
total 0
drwxr-xr-x 2 root root 4.0K Jan 28 17:06 1_8388608_dir1
-rw-r--r-- 1 root root    0 Jan 28 17:06 8388608_4194304_file1
~/cwdir2$ mv .recycle/2023-01-28-17/8388608_4194304_file1 ./file1_new
~/cwdir2$ ls
file1_new
~/cwdir2$ ls -alh .recycle/2023-01-28-17/
total 0
drwxr-xr-x 2 root root 4.0K Jan 28 17:06 1_8388608_dir1
```

### 5、CurveFS 回收站的清理

CurveFS 回收站会定期检查是否有文件已经过期，清理过期的文件，自动清理工作由 metaserver 负责。metaserver 会定期扫描回收站下的目录，根据删除的时间（前面提到的 年-月-日-小时 目录）和设置的回收站时间(前面提到的 recycleTimeHour)判断是否需要清理。每个fs有自己独立的扫描清理任务，各fs之间互不干扰。每个fs默认1小时扫描检查1次回收站。相关的配置在 curve/curvefs/conf/metaserver.conf。

```
# recycle options
# metaserver scan recycle period, default 1h
recycle.manager.scanPeriodSec=3600
# metaserver recycle cleaner scan list dentry limit, default 1000
recycle.cleaner.scanLimit=1000
```

## CurveFS 待开发功能

当前 CurveFS 的回收站功能只提供了上述基本功能，还有一些的回收站功能未开发，包括但不限于以下几点。

* 目前fs的回收站功能是否开启不支持修改，以及回收站时间也不支持修改，只能在创建fs的时候指定。后续需要提供修改回收站过期时间修改功能。
* 提供回收站过滤功能。如果 CurveFS 所有的文件删除后不加筛选都进入回收站，可能会在回收站放入太多的垃圾文件，占用fs的资源。可考虑增加回收站白名单或者黑名单，对进入回收站的文件进行过滤。
* 提供目录批量删除加速功能。CurveFS 是通过fuse的方式挂载到宿主机上使用，如果删除一个有1亿文件的目录，fuse会向CurveFS下发1亿条删除请求。如果提供一个批量删除加速的工具，CurveFS只接收1条删除请求，由CurveFS系统自己进行删除，会大大提高删除的效率。
* 提供目录的原路径恢复，当前恢复回收站中的文件，只能通过mv的方式一个一个恢复文件。如果删除一个目录，有时候是希望能够按照目录原来的层级结构，把目录下的文件进行恢复。回收站中的文件记录了删除的时候的父目录的inodeid和自己的inodeid，利用这些信息，可以拼凑出的原来的层级结构，进而把文件自动恢复的删除前的原路径。

以上功能都是CurveFS回收站希望开发，但是还未来得及开发的功能。社区感兴趣的小伙伴可以联系我们一起开发。

不知道Curve社区的小伙伴是否有其他想要的CurveFS回收站功能。可以通过Curve用户群和我们联系。

![curve_bot](./image/0109-curve_bot.jpg)
