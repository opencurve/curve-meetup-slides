# 云服务器单机部署 curveBS 集群踩坑指南（新人向）

## 前言

作为一名练习时长两年半的编程新手，我在搭建测试环境的过程中踩了不少奇怪的坑，但是在我自己和 Curve 社区的共同努力下最终部署成功！
在此之后，我在 Curve 上半年的开发者活动成为 curve 的贡献者。
回首整个活动，我想记录一下部署过程中的心路历程，向其他小伙伴的分享我的经验和教训，帮助大家更好地部署 curve 集群。

在一开始，我尝试了用 WSL 部署 curveBS 集群，在部署前咨询 Curve 社区的小伙伴，他们说没有经过测试，不建议使用，但是我还是想试试看。
在一番研究和与开发者活动的小伙伴交流后，发现这条路比较困难，就没有做下去。
之后我考虑使用云服务器来部署集群，对个人开发者来说，租用云服务器是一个比较省时省力的方式。
事实上，在交流群中我也发现不少人在使用云服务器来搭建 curve 的调试开发试用环境。

现在正好最近 curve 社区在举办 code camp，相信很多小伙伴已经跃跃欲试了，所以我打算分享一下在云服务上单机部署 curveBS 集群的个人经验，希望对大家能有所帮助。

## 云服务器单机部署 curveBS 集群

总体上各步骤和 [Curveadm WiKi](https://github.com/opencurve/curveadm/wiki/curvebs-cluster-deployment) 一致，因此对新人来说，好好读 wiki 是非常重要的。
仔细阅读 wiki 可以解决很多部署过程中遇到的问题。
curve 项目中也有很多文档值得细读，对于理解 curve 也是很有帮助的。
首先，我租用了一个2核2G的云服务器，系统盘容量为40G。
curve 貌似没有提供最低配置，但我使用更低配置时出现过内存不足等异常情况，因此建议使用此配置以上的云服务器。
如果你需要编译 curve 内核，可以选择更高配置的服务器或者直接在本地编译、构建镜像上传，再在云服务器中使用镜像部署。
根据环境要求，我使用了 Debian11 的镜像，并安装了 docker 和 curveAdm，然后准备主机列表。

```yaml
global:
  user: root
  ssh_port: 22
  private_key_file: /root/.ssh/id_rsa

hosts:
  - host: server-host
    hostname: {内网ip}
```

这里 wiki 上提供了多机部署的模板，我做了一些改动。
值得注意的是，云服务器厂商会提供公网 ip 和内网 ip。
而在配置中应使用内网 ip，不然在访问端口时会出现问题，同时不要忘记在云服务器控制台中配置安全组来解决防火墙问题。
此外，需要配置 rsa 密钥并做一下本机免密登陆。

```bash
cd /root/.ssh
ssh-keygen -t rsa
cat id_rsa.pub >> authorized_keys
```

之后格式化磁盘这一步是可以跳过的。
此外在云服务器中一般使用 vda 虚拟磁盘供用户使用，这一部分社区没有做过相关的测试，对于试用人群（非性能测试）来说，可以直接跳过，不影响正常功能使用。

在配置拓扑文件时，对于跳过快照的用户需要将s3相关字段删去。
而对于跳过格式化的用户则需要在拓扑文件中做如下修改。
如果需要配置快照，需要搭建一个s3对象存储，这里我选用了开源的 minio，网上的教程也是比较多的。

```yaml

chunkserver_services:
  config:
    chunkfilepool.enable_get_chunk_from_pool: false  
```

此外，因为某些原因，docker.io 的镜像下载不太稳定，可以使用自己构建的镜像，也可以通过 quay.io 来获取镜像。
Curve 社区考虑到 docker.io 下载的不稳定性，把 quay.io 作为 docker.io 之外的备份仓库，目前部分镜像已经上传，如有其他需要，大家可以联系 curve 社区，会有小伙伴帮大家上传。
对于开发调试来说，可能需要自己编译并构建镜像，可以参考[这篇文章](https://github.com/opencurve/curve/blob/master/docs/cn/build_and_run.md)。

完整的topology.yaml配置如下：

```yaml
kind: curvebs
global:
  container_image: quay.io/opencurve/curve/curvebs:v1.2.7-beta2_872d38c
  log_dir: ${home}/logs/${service_role}${service_host_sequence}
  data_dir: ${home}/data/${service_role}${service_host_sequence}
  s3.nos_address: **.*.*.**:9000
  s3.snapshot_bucket_name: curve
  s3.ak: ********
  s3.sk: ********
  variable:
    home: /tmp
    target: server-host

etcd_services:
  config:
    listen.ip: ${service_host}
    listen.port: 2380${service_host_sequence}
    listen.client_port: 2379${service_host_sequence}
  deploy:
    - host: ${target}
    - host: ${target}
    - host: ${target}

mds_services:
  config:
    listen.ip: ${service_host}
    listen.port: 670${service_host_sequence}
    listen.dummy_port: 770${service_host_sequence}
  deploy:
    - host: ${target}
    - host: ${target}
    - host: ${target}

chunkserver_services:
  config:
    listen.ip: ${service_host}
    listen.port: 820${service_host_sequence}  # 8200,8201,8202
    data_dir: /data/chunkserver${service_host_sequence}  # /data/chunkserver0, /data/chunksever1
    copysets: 100
    chunkfilepool.enable_get_chunk_from_pool: false
  deploy:
    - host: ${target}
    - host: ${target}
    - host: ${target}

snapshotclone_services:
  config:
    listen.ip: ${service_host}
    listen.port: 555${service_host_sequence}
    listen.dummy_port: 810${service_host_sequence}
    listen.proxy_port: 800${service_host_sequence}
  deploy:
    - host: ${target}
    - host: ${target}
    - host: ${target}
```

以上是我总结的一些注意点，但更可能出现一些预料不到的错误，这时候可以依据错误码在 wiki 上寻找解决方案，也可以观察 curveadm 和 docker 的日志来发现问题。

## 总结

通过本文的介绍，相信大家已经掌握了如何在云服务器上部署 curveBS 集群的技巧。
未来，我们还可以进一步探索 curveBS 的更多功能和应用场景，让我们一起期待吧！
curve 作为一个年轻的开源项目，还是比较欠缺一些非官方的教程和经验贴。
在遇到困难时，大家可以通过提 issue 和微信群等方式直接向社区寻求帮助。
在我为搭不起环境而辗转难眠的几个夜晚， @Cyber-SiKu 和微信群中的热心开发者们都为我提供了大量帮助，也希望大家多多分享相关经验，本文忝作抛砖引玉之用。
