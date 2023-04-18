# Curve Operator上手初体验

目前curve Operator已经支持CurveBS集群部署和删除功能已经实现，这里我们对curve-operator项目的使用以及如何配合curve-csi，使用curve为后端Pod提供存储系统的能力做一个简单介绍。

## 部署curve cluster

### 环境要求

* 三个节点（目前不支持单机部署，该功能正在开发中）
* Kubernetes 1.19+

### 安装curve-operator

第一步是部署Curve Operator，只需要克隆下Operator仓库，然后安装即可：

```shell
git clone https://github.com/opencurve/curve-operator.git
cd curve-operator/config
kubectl apply -f deploy/manifests.yaml
```

执行完上述命令之后，你可以执行以下命令验证 `curve-operator` 处于running状态：

```shell
kubectl get pod -n curvebs

NAME                              READY   STATUS    RESTARTS   AGE
curve-operator-69bc69c75d-jfsjg   1/1     Running   0          7s
```

### 部署Curve cluster

经过上述第一步的操作，Curve Operator处于running状态，下一步我们就需要部署一个CurveBS后端集群，在部署之前，需要有几项内容需要配置。

完整的配置文件在`config/samples/cluster.yaml`

```yaml
apiVersion: operator.curve.io/v1
kind: CurveCluster
metadata:
  name: my-cluster
  namespace: curvebs
spec:
  curveVersion:
    image: opencurvedocker/curvebs:v1.2
  nodes:
  - curve-operator-node1
  - curve-operator-node2
  - curve-operator-node3
  hostDataDir: /curvebs
  etcd:
    port: 23880
    listenPort: 23890
  mds:
    port: 23970
    dummyPort: 23960
  storage:
    useSelectedNodes: false
    nodes:
    - curve-operator-node1
    - curve-operator-node2
    - curve-operator-node3
    port: 8200
    copysets: 100
    devices:
    - name: /dev/vdc
      mountPath: /data/chunkserver0
      percentage: 80
  snapShotClone:
    enable: false
    port: 5555
    dummyPort: 8083
    proxyPort: 8084
    s3Config:
      ak: minioadmin
      sk: minioadmin
      nosAddress: http://10.219.196.145:9000
      bucketName: curvebs
```

需要注意的以下几个字段：

* curveVersion：CurveBS集群版本；
* nodes:这里需要配置集群中的三个节点，后续在这三个节点上会部署etcd，mds和snapshotclone(可选)
* 各个服务的port端口：这些端口需要保证不能与节点上的其他服务产生冲突，否则会部署失败；
* storage：配置存储节点，即chunkserver。这里storage.nodes是需要部署chunkserver的所有节点，devices是配置存储设备，要求上述配置的所有节点都要存在该磁盘，否则会部署失败。
* snapShotClone：快照克隆服务，该服务是可选部署，如果不想要部署该服务，将snapShotClone.enable设置为false即可。

在修改完成上述内容之后，使用如下命令部署Curve cluster：

```shell
kubectl apply -f config/samples/cluster.yaml
```

随后，可以使用`kubectl`去查看curvebs命名空间下的所有的pod，应该能到类似下面的几个pod处于running 状态。chunkserver服务需要等待prepare-chunkfile job执行结束后才会创建运行，格式化的时间根据盘符大小确定，所以可能需要等待一段时间才能看到配置的若干curve-chunkserver pod处于running状态。

```shell
root@curve-operator-node1:~/curve-operator# kubectl get pod -n curvebs
NAME                                                          READY   STATUS      RESTARTS   AGE
curve-chunkserver-curve-operator-node1-vdc-556fc99467-5nx9q   1/1     Running     0          5m45s
curve-chunkserver-curve-operator-node2-vdc-7cf89768f9-hmcrs   1/1     Running     0          5m45s
curve-chunkserver-curve-operator-node3-vdc-f77dd85dc-z5bws    1/1     Running     0          5m45s
curve-etcd-a-d5bbfb755-lzgrm                                  1/1     Running     0          41m
curve-etcd-b-66c5b54f75-6nnnt                                 1/1     Running     0          41m
curve-etcd-c-86b7964f87-cj8zk                                 1/1     Running     0          41m
curve-mds-a-7b5989bddd-ln2sm                                  1/1     Running     0          40m
curve-mds-b-56d8f58645-gv6pd                                  1/1     Running     0          40m
curve-mds-c-997c7fd-vt5hw                                     1/1     Running     0          40m
gen-logical-pool-rzhlz                                        0/1     Completed   0          5m15s
gen-physical-pool-chnw8                                       0/1     Completed   0          5m45s
prepare-chunkfile-curve-operator-node1-vdc-znb66              0/1     Completed   0          40m
prepare-chunkfile-curve-operator-node2-vdc-6gf2z              0/1     Completed   0          40m
prepare-chunkfile-curve-operator-node3-vdc-2bkxm              0/1     Completed   0          40m
read-config-k272k                                             0/1     Completed   0          41m
```

至此整个集群已经部署完成。

## 部署curve-csi

在完成Curve后端集群部署之后，需要部署curve-csi进一步验证集群的正确性。

1. curve-csi的部署流程参考:[https://ask.opencurve.io/t/topic/89](https://ask.opencurve.io/t/topic/89)

在部署完成之后，可以在csi-system命名空间中看到如下pod处于running状态：

```shell
root@curve-operator-node1:~/code/curve-csi/deploy# kubectl get pod -n csi-system
NAME                                            READY   STATUS    RESTARTS   AGE
csi-curve-plugin-9lx6f                          2/2     Running   0          4s
csi-curve-plugin-pjd42                          2/2     Running   0          4s
csi-curve-plugin-provisioner-744d477fc7-blgvd   5/5     Running   0          5s
csi-curve-plugin-provisioner-744d477fc7-gqm2l   5/5     Running   0          5s
csi-curve-plugin-vqpbz                          2/2     Running   0          4s
```

2. 创建StorageClass以及PVC看是否可以创建Curve volume

```shell
kubectl apply -f example/storageclass.yaml
kubectl apply -f example/pvc-block.yaml
```

可以使用`kubectl`工具查看pvc/pv是否处于bound状态：

```shell
kubectl get pvc

NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
curve-test-pvc   Bound    pvc-119ca0e2-259f-4c29-8399-8899e1f7b96e   20Gi       RWO            curve          3s
```

```shell
kubectl get pv

NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   REASON   AGE
pvc-119ca0e2-259f-4c29-8399-8899e1f7b96e   20Gi       RWO            Delete           Bound    default/curve-test-pvc   curve                   78s
```

可以看到pv和pvc创建成功，并已经成功绑定。

3. 创建使用该pvc的pod

```shell
kubectl apply -f example/pod-block.yaml

NAME               READY   STATUS    RESTARTS   AGE
csi-curve-test     1/1     Running   0          20s
```

## 小结

Curve Operator项目以Curve为核心，通过Operator模式实现Curve在Kubernetes环境下的管理和运维。Operator本质是一个简单的容器应用可以用于构建以及管理Curve集群，使得Curve存储系统集成于云原生环境，该项目的最终目标是构建一个自动化部署和运维工具，提升Curve集群的易用性与稳定性。
