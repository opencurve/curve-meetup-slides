## 需求背景
AI训练业务场景下，业务希望通过S3接口上传训练所用到的数据集，实际训练过程中则想要使用CurveFS共享文件系统来实现多节点共享访问训练数据集，以及存储训练过程中产生的临时文件，并利用CurveFS为AI业务专门开发的文件/目录预热、内存缓存、本地盘缓存、分布式KV缓存特性，最终在低存储成本前提下的训练加速。

简单来说就是想要实现S3协议和Posix协议的统一命名空间能力，两种协议上传/写入的文件可以互相访问。本文简述相关部署过程，供有类似需求的社区小伙伴们参考。


## 部署前提
1. 已经部署了一套CurveFS共享文件存储系统
2. 已经挂载了一个CurveFS文件系统到服务器的`/mnt/minio-data`目录上
3. 已安装docker运行环境

部署CurveFS和挂载文件系统的操作步骤可以参考Curve官方部署工具CurveAdm的用户手册：
- https://github.com/opencurve/curveadm/wiki/curvefs-cluster-deployment
- https://github.com/opencurve/curveadm/wiki/curvefs-client-deployment

## 部署步骤
这里使用`minio-s3-gateway`服务作为S3网关，并用docker进行部署，相关参考资料：https://github.com/minio/minio/blob/RELEASE.2022-04-26T01-20-24Z/docs/gateway/nas.md

```
$ docker run --privileged -p 9000:9000 \
--name curvefs-minio-s3-gateway \
-v /mnt/minio-data:/data \
-e "MINIO_ROOT_USER=minio-access-key" \
-e "MINIO_ROOT_PASSWORD=minio-secret-key" \
-e "MINIO_REGION=us-east-1" \
--console-address ":9001" \
docker.io/minio/minio:RELEASE.2022-04-26T01-20-24Z \
gateway nas /data
```
执行上述命令就可以启动一个CurveFS的minio-s3-gateway服务，非常的简单方便。

如果部署成功，会在屏幕上打印如下内容：
```
API: http://10.88.0.18:9000  http://127.0.0.1:9000     

Console: http://10.88.0.18:9001 http://127.0.0.1:9001   

Documentation: https://docs.min.io
Finished loading IAM sub-system (took 0.0s of 0.0s to load data).
......   // 以下内容省略
```

Console地址就是web控制台地址，可以在浏览器里直接打开访问，用户名密码是上面docker命令行里配置的环境变量`MINIO_ROOT_USER`和`MINIO_ROOT_PASSWORD`对应的值，参考资料：https://min.io/docs/minio/linux/administration/minio-console.html


## 功能验证
接下来我们部署一个minio的客户端，来验证S3网关的可用性，以及S3网关与CurveFS文件系统挂载点是否可以做到统一命名空间下互相访问文件。

### 部署minio客户端
minio客户端我们也是用docker来部署：
```
$ docker run -it --entrypoint=/bin/sh minio/mc:RELEASE.2022-04-26T18-00-22Z
```
执行上述命令就可以启动一个`minio`命令行工具`mc`的运行环境了。

修改`mc`命令行的配置文件，默认是`/root/.mc/config.json`（如果容器内缺少编辑器不方便修改可以在本地修改好之后用`docker cp`命令复制进去）：
```
{
	"version": "10",
	"aliases": {
		"curvefs": {
			"url": "http://10.88.0.18:9000",
			"accessKey": "minio-access-key",
			"secretKey": "minio-secret-key",
			"api": "S3v4",
			"path": "auto"
		}
	}
}

```
其中url就是部署`minio-s3-gateway`服务时屏幕打印的API地址，`accessKey`就是启动`minio-s3-gateway`容器的`MINIO_ROOT_USER`，`secretKey`就是`MINIO_ROOT_PASSWORD`，其他两个保持默认即可。`curvefs`是minio集群的别名，下面会用到。

### 统一命名空间功能验证

#### 目标1：使用S3创建桶并上传文件，在CurveFS挂载点对应目录下访问
使用`mc`命令行工具分别进行创建桶、上传文件、列出桶内文件操作：
```
$ mc mb curvefs/bucket1
Bucket created successfully `curvefs/bucket1`.
$ mc cp anaconda-ks.cfg curvefs/bucket1/
/root/anaconda-ks.cfg:  7.53 KiB / 7.53 KiB ┃▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓┃ 777.03 KiB/s 0s
$ mc ls curvefs/bucket1/
[2023-04-25 09:16:37 UTC] 7.5KiB STANDARD anaconda-ks.cfg
```

在CurveFS的挂载点目录下列出目录和文件，并校验文件md5值（校验md5步骤省略，已确认一致）：
```
$ ls /mnt/minio-data/
bucket1
$ ls /mnt/minio-data/bucket1/
anaconda-ks.cfg
```

#### 目标2：使用S3上传包含子目录的文件到桶内，在CurveFS挂载点对应目录下访问
```
$ mc cp curvefs/bucket1/dir1/bigfile.500M
...t1/dir1/bigfile.500M:  500.00 MiB / 500.00 MiB ┃▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓┃ 628.02 MiB/s 0s
```

在CurveFS的挂载点目录下列出目录和文件，并校验文件md5值（校验md5步骤省略，已确认一致）：
```
$ ls /mnt/minio-data/bucket1/dir1
bigfile.500M
```

#### 目标3：在CurveFS挂载点创建根目录及目录下的文件，使用S3查看桶及桶内文件
首先使用dd命令在CurveFS挂载点的bucket1目录下创建一个10M的文件：
```
$ cd /mnt/minio-data/bucket1/dir1
$ dd if=/dev/zero of=./newbigfile.10M bs=1M count=10
```

之后用`mc`命令查看桶内文件并下载后校验md5值（校验md5步骤省略，已确认一致）：
```
$ mc ls curvefs/bucket1/dir1
[2023-04-25 09:29:33 UTC]  10MiB STANDARD newbigfile.10M
$ mc cp curvefs/bucket1/dir1/newbigfile.10M .
/root/newbigfile.10M:   10.00 MiB / 10.00 MiB ┃▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓┃ 76.14 MiB/s 0s
```


## 补充说明


### 高可用及单节点性能问题
这里的操作步骤均是单节点模式，如果要做到s3网关的高可用及防止性能瓶颈，可以在多台服务器上重复上述步骤，并给S3网关服务部署负载均衡服务（如Nginx或haproxy等），只需要CurveFS挂载的是同一个文件系统即可。

### CurveAdm部署工具集成问题
本次实践是一次功能验证，后续Curve社区将把相关功能集成到CurveAdm部署工具中，方便用户使用维护。

### minio gateway的废弃问题
minio的s3 gateway服务已经于2020年初开始逐步废弃了，但老版本的仍然可以继续使用，只需要指定版本号即可正常部署。如果有特殊需求，也可以fork minio的代码仓库自行定制修改。

参考资料：https://blog.min.io/deprecation-of-the-minio-gateway/
