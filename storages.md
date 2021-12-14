# 卷

Container 中的文件在磁盘上是临时存放的，这给 Container 中运行的较重要的应用 程序带来一些问题。问题之一是当容器崩溃时文件丢失。kubelet 会重新启动容器， 但容器会以干净的状态重启。 第二个问题会在同一 `Pod` 中运行多个容器并共享文件时出现。 Kubernetes [卷（Volume）](https://kubernetes.io/zh/docs/concepts/storage/volumes/) 这一抽象概念能够解决这两个问题。

阅读本文前建议你熟悉一下 [Pods](https://kubernetes.io/zh/docs/concepts/workloads/pods)。

## 背景 

Docker 也有 [卷（Volume）](https://docs.docker.com/storage/) 的概念，但对它只有少量且松散的管理。 Docker 卷是磁盘上或者另外一个容器内的一个目录。 Docker 提供卷驱动程序，但是其功能非常有限。

Kubernetes 支持很多类型的卷。 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 可以同时使用任意数目的卷类型。 临时卷类型的生命周期与 Pod 相同，但持久卷可以比 Pod 的存活期长。 当 Pod 不再存在时，Kubernetes 也会销毁临时卷；不过 Kubernetes 不会销毁 持久卷。对于给定 Pod 中任何类型的卷，在容器重启期间数据都不会丢失。

卷的核心是一个目录，其中可能存有数据，Pod 中的容器可以访问该目录中的数据。 所采用的特定的卷类型将决定该目录如何形成的、使用何种介质保存数据以及目录中存放 的内容。

使用卷时, 在 `.spec.volumes` 字段中设置为 Pod 提供的卷，并在 `.spec.containers[*].volumeMounts` 字段中声明卷在容器中的挂载位置。 容器中的进程看到的是由它们的 Docker 镜像和卷组成的文件系统视图。 [Docker 镜像](https://docs.docker.com/userguide/dockerimages/) 位于文件系统层次结构的根部。各个卷则挂载在镜像内的指定路径上。 卷不能挂载到其他卷之上，也不能与其他卷有硬链接。 Pod 配置中的每个容器必须独立指定各个卷的挂载位置。

## 卷类型 

Kubernetes 支持下列类型的卷：

### awsElasticBlockStore

`awsElasticBlockStore` 卷将 Amazon Web服务（AWS）[EBS 卷](https://aws.amazon.com/ebs/) 挂载到你的 Pod 中。与 `emptyDir` 在 Pod 被删除时也被删除不同，EBS 卷的内容在删除 Pod 时 会被保留，卷只是被卸载掉了。 这意味着 EBS 卷可以预先填充数据，并且该数据可以在 Pod 之间共享。

**说明：** 你在使用 EBS 卷之前必须使用 `aws ec2 create-volume` 命令或者 AWS API 创建该卷。

使用 `awsElasticBlockStore` 卷时有一些限制：

- Pod 运行所在的节点必须是 AWS EC2 实例。
- 这些实例需要与 EBS 卷在相同的地域（Region）和可用区（Availability-Zone）。
- EBS 卷只支持被挂载到单个 EC2 实例上。

#### 创建 EBS 卷

在将 EBS 卷用到 Pod 上之前，你首先要创建它。

```shell
aws ec2 create-volume --availability-zone=eu-west-1a --size=10 --volume-type=gp2
```

确保该区域与你的群集所在的区域相匹配。还要检查卷的大小和 EBS 卷类型都适合你的用途。

#### AWS EBS 配置示例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-ebs
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-ebs
      name: test-volume
  volumes:
  - name: test-volume
    # 此 AWS EBS 卷必须已经存在
    awsElasticBlockStore:
      volumeID: "<volume-id>"
      fsType: ext4
```

如果 EBS 卷是分区的，你可以提供可选的字段 `partition: "<partition number>"` 来指定要挂载到哪个分区上。

#### AWS EBS CSI 卷迁移

**FEATURE STATE:** `Kubernetes v1.17 [beta]`

如果启用了对 `awsElasticBlockStore` 的 `CSIMigration` 特性支持，所有插件操作都 不再指向树内插件（In-Tree Plugin），转而指向 `ebs.csi.aws.com` 容器存储接口 （Container Storage Interface，CSI）驱动。为了使用此特性，必须在集群中安装 [AWS EBS CSI 驱动](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)， 并确保 `CSIMigration` 和 `CSIMigrationAWS` Beta 功能特性被启用。

#### AWS EBS CSI 迁移结束

**FEATURE STATE:** `Kubernetes v1.17 [alpha]`

要禁止控制器管理器和 kubelet 加载 `awsElasticBlockStore` 存储插件， 请将 `InTreePluginAWSUnregister` 标志设置为 `true`。

### azureDisk

`azureDisk` 卷类型用来在 Pod 上挂载 Microsoft Azure [数据盘（Data Disk）](https://azure.microsoft.com/en-us/documentation/articles/virtual-machines-linux-about-disks-vhds/) 。 若需了解更多详情，请参考 [`azureDisk` 卷插件](https://github.com/kubernetes/examples/tree/master/staging/volumes/azure_disk/README.md)。

#### azureDisk 的 CSI 迁移 

**FEATURE STATE:** `Kubernetes v1.19 [beta]`

启用 `azureDisk` 的 `CSIMigration` 功能后，所有插件操作从现有的树内插件重定向到 `disk.csi.azure.com` 容器存储接口（CSI）驱动程序。 为了使用此功能，必须在集群中安装 [Azure 磁盘 CSI 驱动程序](https://github.com/kubernetes-sigs/azuredisk-csi-driver)， 并且 `CSIMigration` 和 `CSIMigrationAzureDisk` 功能必须被启用。

### azureFile

`azureFile` 卷类型用来在 Pod 上挂载 Microsoft Azure 文件卷（File Volume）（SMB 2.1 和 3.0）。 更多详情请参考 [`azureFile` 卷插件](https://github.com/kubernetes/examples/tree/master/staging/volumes/azure_file/README.md)。

#### azureFile CSI 迁移 

**FEATURE STATE:** `Kubernetes v1.21 [beta]`

启用 `azureFile` 的 `CSIMigration` 功能后，所有插件操作将从现有的树内插件重定向到 `file.csi.azure.com` 容器存储接口（CSI）驱动程序。要使用此功能，必须在集群中安装 [Azure 文件 CSI 驱动程序](https://github.com/kubernetes-sigs/azurefile-csi-driver)， 并且 `CSIMigration` 和 `CSIMigrationAzureFile` [特性门控](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/feature-gates/) 必须被启用。

Azure 文件 CSI 驱动尚不支持为同一卷设置不同的 fsgroup。 如果 AzureFile CSI 迁移被启用，用不同的 fsgroup 来使用同一卷也是不被支持的。

### cephfs

`cephfs` 卷允许你将现存的 CephFS 卷挂载到 Pod 中。 不像 `emptyDir` 那样会在 Pod 被删除的同时也会被删除，`cephfs` 卷的内容在 Pod 被删除 时会被保留，只是卷被卸载了。这意味着 `cephfs` 卷可以被预先填充数据，且这些数据可以在 Pod 之间共享。同一 `cephfs` 卷可同时被多个写者挂载。

**说明：** 在使用 Ceph 卷之前，你的 Ceph 服务器必须已经运行并将要使用的 share 导出（exported）。

更多信息请参考 [CephFS 示例](https://github.com/kubernetes/examples/tree/master/volumes/cephfs/)。

### cinder

**说明：** Kubernetes 必须配置了 OpenStack Cloud Provider。

`cinder` 卷类型用于将 OpenStack Cinder 卷挂载到 Pod 中。

#### Cinder 卷示例配置

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-cinder
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-cinder-container
    volumeMounts:
    - mountPath: /test-cinder
      name: test-volume
  volumes:
  - name: test-volume
    # 此 OpenStack 卷必须已经存在
    cinder:
      volumeID: "<volume-id>"
      fsType: ext4
```

#### OpenStack CSI 迁移

**FEATURE STATE:** `Kubernetes v1.21 [beta]`

Cinder 的 `CSIMigration` 功能在 Kubernetes 1.21 版本中是默认被启用的。 此特性会将插件的所有操作从现有的树内插件重定向到 `cinder.csi.openstack.org` 容器存储接口（CSI）驱动程序。 为了使用此功能，必须在集群中安装 [OpenStack Cinder CSI 驱动程序](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/cinder-csi-plugin/using-cinder-csi-plugin.md)， 你可以通过设置 `CSIMigrationOpenStack` [特性门控](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/feature-gates/) 为 `false` 来禁止 Cinder CSI 迁移。 如果你禁用了 `CSIMigrationOpenStack` 功能特性，则树内的 Cinder 卷插件 会负责 Cinder 卷存储管理的方方面面。

### configMap

[`configMap`](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-pod-configmap/) 卷 提供了向 Pod 注入配置数据的方法。 ConfigMap 对象中存储的数据可以被 `configMap` 类型的卷引用，然后被 Pod 中运行的 容器化应用使用。

引用 configMap 对象时，你可以在 volume 中通过它的名称来引用。 你可以自定义 ConfigMap 中特定条目所要使用的路径。 下面的配置显示了如何将名为 `log-config` 的 ConfigMap 挂载到名为 `configmap-pod` 的 Pod 中：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
    - name: test
      image: busybox
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: log-config
        items:
          - key: log_level
            path: log_level
```

`log-config` ConfigMap 以卷的形式挂载，并且存储在 `log_level` 条目中的所有内容 都被挂载到 Pod 的 `/etc/config/log_level` 路径下。 请注意，这个路径来源于卷的 `mountPath` 和 `log_level` 键对应的 `path`。

**说明：**

- 在使用 [ConfigMap](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-pod-configmap/) 之前你首先要创建它。
- 容器以 [subPath](https://kubernetes.io/zh/docs/concepts/storage/volumes/#using-subpath) 卷挂载方式使用 ConfigMap 时，将无法接收 ConfigMap 的更新。
- 文本数据挂载成文件时采用 UTF-8 字符编码。如果使用其他字符编码形式，可使用 `binaryData` 字段。

### downwardAPI

`downwardAPI` 卷用于使 downward API 数据对应用程序可用。 这种卷类型挂载一个目录并在纯文本文件中写入所请求的数据。

**说明：** 容器以 [subPath](https://kubernetes.io/zh/docs/concepts/storage/volumes/#using-subpath) 卷挂载方式使用 downwardAPI 时，将不能接收到它的更新。

更多详细信息请参考 [`downwardAPI` 卷示例](https://kubernetes.io/zh/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/)。

### emptyDir

当 Pod 分派到某个 Node 上时，`emptyDir` 卷会被创建，并且在 Pod 在该节点上运行期间，卷一直存在。 就像其名称表示的那样，卷最初是空的。 尽管 Pod 中的容器挂载 `emptyDir` 卷的路径可能相同也可能不同，这些容器都可以读写 `emptyDir` 卷中相同的文件。 当 Pod 因为某些原因被从节点上删除时，`emptyDir` 卷中的数据也会被永久删除。

**说明：** 容器崩溃并**不**会导致 Pod 被从节点上移除，因此容器崩溃期间 `emptyDir` 卷中的数据是安全的。

`emptyDir` 的一些用途：

- 缓存空间，例如基于磁盘的归并排序。
- 为耗时较长的计算任务提供检查点，以便任务能方便地从崩溃前状态恢复执行。
- 在 Web 服务器容器服务数据时，保存内容管理器容器获取的文件。

取决于你的环境，`emptyDir` 卷存储在该节点所使用的介质上；这里的介质可以是磁盘或 SSD 或网络存储。但是，你可以将 `emptyDir.medium` 字段设置为 `"Memory"`，以告诉 Kubernetes 为你挂载 tmpfs（基于 RAM 的文件系统）。 虽然 tmpfs 速度非常快，但是要注意它与磁盘不同。 tmpfs 在节点重启时会被清除，并且你所写入的所有文件都会计入容器的内存消耗，受容器内存限制约束。

**说明：** 当启用 `SizeMemoryBackedVolumes` [特性门控](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/feature-gates/)时， 你可以为基于内存提供的卷指定大小。 如果未指定大小，则基于内存的卷的大小为 Linux 主机上内存的 50％。

#### emptyDir 配置示例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```

### fc (光纤通道)

`fc` 卷类型允许将现有的光纤通道块存储卷挂载到 Pod 中。 可以使用卷配置中的参数 `targetWWNs` 来指定单个或多个目标 WWN（World Wide Names）。 如果指定了多个 WWN，targetWWNs 期望这些 WWN 来自多路径连接。

**说明：** 你必须配置 FC SAN Zoning，以便预先向目标 WWN 分配和屏蔽这些 LUN（卷）， 这样 Kubernetes 主机才可以访问它们。

更多详情请参考 [FC 示例](https://github.com/kubernetes/examples/tree/master/staging/volumes/fibre_channel)。

### flocker （已弃用）

[Flocker](https://github.com/ClusterHQ/flocker) 是一个开源的、集群化的容器数据卷管理器。 Flocker 提供了由各种存储后端所支持的数据卷的管理和编排。

使用 `flocker` 卷可以将一个 Flocker 数据集挂载到 Pod 中。 如果数据集在 Flocker 中不存在，则需要首先使用 Flocker CLI 或 Flocker API 创建数据集。 如果数据集已经存在，那么 Flocker 将把它重新附加到 Pod 被调度的节点。 这意味着数据可以根据需要在 Pod 之间共享。

**说明：** 在使用 Flocker 之前你必须先安装运行自己的 Flocker。

更多详情请参考 [Flocker 示例](https://github.com/kubernetes/examples/tree/master/staging/volumes/flocker)。

### gcePersistentDisk

`gcePersistentDisk` 卷能将谷歌计算引擎 (GCE) [持久盘（PD）](http://cloud.google.com/compute/docs/disks) 挂载到你的 Pod 中。 不像 `emptyDir` 那样会在 Pod 被删除的同时也会被删除，持久盘卷的内容在删除 Pod 时会被保留，卷只是被卸载了。 这意味着持久盘卷可以被预先填充数据，并且这些数据可以在 Pod 之间共享。

**注意：** 在使用 PD 前，你必须使用 `gcloud` 或者 GCE API 或 UI 创建它。

使用 `gcePersistentDisk` 时有一些限制：

- 运行 Pod 的节点必须是 GCE VM
- 这些 VM 必须和持久盘位于相同的 GCE 项目和区域（zone）

GCE PD 的一个特点是它们可以同时被多个消费者以只读方式挂载。 这意味着你可以用数据集预先填充 PD，然后根据需要并行地在尽可能多的 Pod 中提供该数据集。 不幸的是，PD 只能由单个使用者以读写模式挂载 —— 即不允许同时写入。

在由 ReplicationController 所管理的 Pod 上使用 GCE PD 将会失败，除非 PD 是只读模式或者副本的数量是 0 或 1。

#### 创建 GCE 持久盘（PD） 

在 Pod 中使用 GCE 持久盘之前，你首先要创建它。

```shell
gcloud compute disks create --size=500GB --zone=us-central1-a my-data-disk
```

#### GCE 持久盘配置示例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    # 此 GCE PD 必须已经存在
    gcePersistentDisk:
      pdName: my-data-disk
      fsType: ext4
```

#### 区域持久盘 

[区域持久盘](https://cloud.google.com/compute/docs/disks/#repds) 功能允许你创建能在 同一区域的两个可用区中使用的持久盘。 要使用这个功能，必须以持久卷（PersistentVolume）的方式提供卷；直接从 Pod 引用这种卷 是不可以的。

#### 手动供应基于区域 PD 的 PersistentVolume

使用[为 GCE PD 定义的存储类](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#gce) 可以 实现动态供应。在创建 PersistentVolume 之前，你首先要创建 PD。

```shell
gcloud beta compute disks create --size=500GB my-data-disk
    --region us-central1
    --replica-zones us-central1-a,us-central1-b
```

PersistentVolume 示例：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: test-volume
  labels:
    failure-domain.beta.kubernetes.io/zone: us-central1-a__us-central1-b
spec:
  capacity:
    storage: 400Gi
  accessModes:
  - ReadWriteOnce
  gcePersistentDisk:
    pdName: my-data-disk
    fsType: ext4
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        # failure-domain.beta.kubernetes.io/zone 应在 1.21 之前使用
        - key: topology.kubernetes.io/zone
          operator: In
          values:
          - us-central1-a
          - us-central1-b
```

#### GCE CSI 迁移 

**FEATURE STATE:** `Kubernetes v1.17 [beta]`

启用 GCE PD 的 `CSIMigration` 功能后，所有插件操作将从现有的树内插件重定向到 `pd.csi.storage.gke.io` 容器存储接口（ CSI ）驱动程序。 为了使用此功能，必须在集群中上安装 [GCE PD CSI驱动程序](https://github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver)， 并且 `CSIMigration` 和 `CSIMigrationGCE` Beta 功能必须被启用。

#### GCE CSI 迁移完成

**FEATURE STATE:** `Kubernetes v1.21 [alpha]`

要禁止控制器管理器和 kubelet 加载 `gcePersistentDisk` 存储插件， 请将 `InTreePluginGCEUnregister` 标志设置为 `true`。

### gitRepo (已弃用) 

**警告：** `gitRepo` 卷类型已经被废弃。如果需要在容器中提供 git 仓库，请将一个 [EmptyDir](https://kubernetes.io/zh/docs/concepts/storage/volumes/#emptydir) 卷挂载到 InitContainer 中，使用 git 命令完成仓库的克隆操作， 然后将 [EmptyDir](https://kubernetes.io/zh/docs/concepts/storage/volumes/#emptydir) 卷挂载到 Pod 的容器中。

`gitRepo` 卷是一个卷插件的例子。 该查卷挂载一个空目录，并将一个 Git 代码仓库克隆到这个目录中供 Pod 使用。

下面给出一个 `gitRepo` 卷的示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: server
spec:
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
    - mountPath: /mypath
      name: git-volume
  volumes:
  - name: git-volume
    gitRepo:
      repository: "git@somewhere:me/my-git-repository.git"
      revision: "22f1d8406d464b0c0874075539c1f2e96c253775"
```

### glusterfs

`glusterfs` 卷能将 [Glusterfs](https://www.gluster.org/) (一个开源的网络文件系统) 挂载到你的 Pod 中。不像 `emptyDir` 那样会在删除 Pod 的同时也会被删除，`glusterfs` 卷的内容在删除 Pod 时会被保存，卷只是被卸载。 这意味着 `glusterfs` 卷可以被预先填充数据，并且这些数据可以在 Pod 之间共享。 GlusterFS 可以被多个写者同时挂载。

**说明：** 在使用前你必须先安装运行自己的 GlusterFS。

更多详情请参考 [GlusterFS 示例](https://github.com/kubernetes/examples/tree/master/volumes/glusterfs)。

### hostPath

**警告：**

HostPath 卷存在许多安全风险，最佳做法是尽可能避免使用 HostPath。 当必须使用 HostPath 卷时，它的范围应仅限于所需的文件或目录，并以只读方式挂载。

如果通过 AdmissionPolicy 限制 HostPath 对特定目录的访问， 则必须要求 `volumeMounts` 使用 `readOnly` 挂载以使策略生效。

`hostPath` 卷能将主机节点文件系统上的文件或目录挂载到你的 Pod 中。 虽然这不是大多数 Pod 需要的，但是它为一些应用程序提供了强大的逃生舱。

例如，`hostPath` 的一些用法有：

- 运行一个需要访问 Docker 内部机制的容器；可使用 `hostPath` 挂载 `/var/lib/docker` 路径。
- 在容器中运行 cAdvisor 时，以 `hostPath` 方式挂载 `/sys`。
- 允许 Pod 指定给定的 `hostPath` 在运行 Pod 之前是否应该存在，是否应该创建以及应该以什么方式存在。

除了必需的 `path` 属性之外，用户可以选择性地为 `hostPath` 卷指定 `type`。

支持的 `type` 值如下：

| 取值                | 行为                                                         |
| :------------------ | :----------------------------------------------------------- |
|                     | 空字符串（默认）用于向后兼容，这意味着在安装 hostPath 卷之前不会执行任何检查。 |
| `DirectoryOrCreate` | 如果在给定路径上什么都不存在，那么将根据需要创建空目录，权限设置为 0755，具有与 kubelet 相同的组和属主信息。 |
| `Directory`         | 在给定路径上必须存在的目录。                                 |
| `FileOrCreate`      | 如果在给定路径上什么都不存在，那么将在那里根据需要创建空文件，权限设置为 0644，具有与 kubelet 相同的组和所有权。 |
| `File`              | 在给定路径上必须存在的文件。                                 |
| `Socket`            | 在给定路径上必须存在的 UNIX 套接字。                         |
| `CharDevice`        | 在给定路径上必须存在的字符设备。                             |
| `BlockDevice`       | 在给定路径上必须存在的块设备。                               |

当使用这种类型的卷时要小心，因为：

- HostPath 卷可能会暴露特权系统凭据（例如 Kubelet）或特权 API（例如容器运行时套接字）， 可用于容器逃逸或攻击集群的其他部分。
- 具有相同配置（例如基于同一 PodTemplate 创建）的多个 Pod 会由于节点上文件的不同 而在不同节点上有不同的行为。
- 下层主机上创建的文件或目录只能由 root 用户写入。你需要在 [特权容器](https://kubernetes.io/zh/docs/tasks/configure-pod-container/security-context/) 中以 root 身份运行进程，或者修改主机上的文件权限以便容器能够写入 `hostPath` 卷。

#### hostPath 配置示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      # 宿主上目录位置
      path: /data
      # 此字段为可选
      type: Directory
```

**注意：** `FileOrCreate` 模式不会负责创建文件的父目录。 如果欲挂载的文件的父目录不存在，Pod 启动会失败。 为了确保这种模式能够工作，可以尝试把文件和它对应的目录分开挂载，如 [`FileOrCreate` 配置](https://kubernetes.io/zh/docs/concepts/storage/volumes/#hostpath-fileorcreate-example) 所示。

#### hostPath FileOrCreate 配置示例 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-webserver
spec:
  containers:
  - name: test-webserver
    image: k8s.gcr.io/test-webserver:latest
    volumeMounts:
    - mountPath: /var/local/aaa
      name: mydir
    - mountPath: /var/local/aaa/1.txt
      name: myfile
  volumes:
  - name: mydir
    hostPath:
      # 确保文件所在目录成功创建。
      path: /var/local/aaa
      type: DirectoryOrCreate
  - name: myfile
    hostPath:
      path: /var/local/aaa/1.txt
      type: FileOrCreate
```

### iscsi

`iscsi` 卷能将 iSCSI (基于 IP 的 SCSI) 卷挂载到你的 Pod 中。 不像 `emptyDir` 那样会在删除 Pod 的同时也会被删除，`iscsi` 卷的内容在删除 Pod 时 会被保留，卷只是被卸载。 这意味着 `iscsi` 卷可以被预先填充数据，并且这些数据可以在 Pod 之间共享。

**注意：** 在使用 iSCSI 卷之前，你必须拥有自己的 iSCSI 服务器，并在上面创建卷。

iSCSI 的一个特点是它可以同时被多个用户以只读方式挂载。 这意味着你可以用数据集预先填充卷，然后根据需要在尽可能多的 Pod 上使用它。 不幸的是，iSCSI 卷只能由单个使用者以读写模式挂载。不允许同时写入。

更多详情请参考 [iSCSI 示例](https://github.com/kubernetes/examples/tree/master/volumes/iscsi)。

### local

`local` 卷所代表的是某个被挂载的本地存储设备，例如磁盘、分区或者目录。

`local` 卷只能用作静态创建的持久卷。尚不支持动态配置。

与 `hostPath` 卷相比，`local` 卷能够以持久和可移植的方式使用，而无需手动将 Pod 调度到节点。系统通过查看 PersistentVolume 的节点亲和性配置，就能了解卷的节点约束。

然而，`local` 卷仍然取决于底层节点的可用性，并不适合所有应用程序。 如果节点变得不健康，那么`local` 卷也将变得不可被 Pod 访问。使用它的 Pod 将不能运行。 使用 `local` 卷的应用程序必须能够容忍这种可用性的降低，以及因底层磁盘的耐用性特征 而带来的潜在的数据丢失风险。

下面是一个使用 `local` 卷和 `nodeAffinity` 的持久卷示例：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - example-node
```

使用 `local` 卷时，你需要设置 PersistentVolume 对象的 `nodeAffinity` 字段。 Kubernetes 调度器使用 PersistentVolume 的 `nodeAffinity` 信息来将使用 `local` 卷的 Pod 调度到正确的节点。

PersistentVolume 对象的 `volumeMode` 字段可被设置为 "Block" （而不是默认值 "Filesystem"），以将 `local` 卷作为原始块设备暴露出来。

使用 `local` 卷时，建议创建一个 StorageClass 并将其 `volumeBindingMode` 设置为 `WaitForFirstConsumer`。要了解更多详细信息，请参考 [local StorageClass 示例](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#local)。 延迟卷绑定的操作可以确保 Kubernetes 在为 PersistentVolumeClaim 作出绑定决策时， 会评估 Pod 可能具有的其他节点约束，例如：如节点资源需求、节点选择器、Pod 亲和性和 Pod 反亲和性。

你可以在 Kubernetes 之外单独运行静态驱动以改进对 local 卷的生命周期管理。 请注意，此驱动尚不支持动态配置。 有关如何运行外部 `local` 卷驱动，请参考 [local 卷驱动用户指南](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner)。

**说明：** 如果不使用外部静态驱动来管理卷的生命周期，用户需要手动清理和删除 local 类型的持久卷。

### nfs

`nfs` 卷能将 NFS (网络文件系统) 挂载到你的 Pod 中。 不像 `emptyDir` 那样会在删除 Pod 的同时也会被删除，`nfs` 卷的内容在删除 Pod 时会被保存，卷只是被卸载。 这意味着 `nfs` 卷可以被预先填充数据，并且这些数据可以在 Pod 之间共享。

**注意：** 在使用 NFS 卷之前，你必须运行自己的 NFS 服务器并将目标 share 导出备用。

要了解更多详情请参考 [NFS 示例](https://github.com/kubernetes/examples/tree/master/staging/volumes/nfs)。

### persistentVolumeClaim

`persistentVolumeClaim` 卷用来将[持久卷](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/)（PersistentVolume） 挂载到 Pod 中。 持久卷申领（PersistentVolumeClaim）是用户在不知道特定云环境细节的情况下"申领"持久存储 （例如 GCE PersistentDisk 或者 iSCSI 卷）的一种方法。

更多详情请参考[持久卷示例](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/)。

### portworxVolume

`portworxVolume` 是一个可伸缩的块存储层，能够以超融合（hyperconverged）的方式与 Kubernetes 一起运行。 [Portworx](https://portworx.com/use-case/kubernetes-storage/) 支持对服务器上存储的指纹处理、 基于存储能力进行分层以及跨多个服务器整合存储容量。 Portworx 可以以 in-guest 方式在虚拟机中运行，也可以在裸金属 Linux 节点上运行。

`portworxVolume` 类型的卷可以通过 Kubernetes 动态创建，也可以预先配备并在 Kubernetes Pod 内引用。 下面是一个引用预先配备的 PortworxVolume 的示例 Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-portworx-volume-pod
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /mnt
      name: pxvol
  volumes:
  - name: pxvol
    # 此 Portworx 卷必须已经存在
    portworxVolume:
      volumeID: "pxvol"
      fsType: "<fs-type>"
```

**说明：**

在 Pod 中使用 portworxVolume 之前，你要确保有一个名为 `pxvol` 的 PortworxVolume 存在。

更多详情可以参考 [Portworx 卷](https://github.com/kubernetes/examples/tree/master/staging/volumes/portworx/README.md)。

### projected

`projected` 卷类型能将若干现有的卷来源映射到同一目录上。

目前，可以映射的卷来源类型如下：

- [`secret`](https://kubernetes.io/zh/docs/concepts/storage/volumes/#secret)
- [`downwardAPI`](https://kubernetes.io/zh/docs/concepts/storage/volumes/#downwardapi)
- [`configMap`](https://kubernetes.io/zh/docs/concepts/storage/volumes/#configmap)
- `serviceAccountToken`

所有的卷来源需要和 Pod 处于相同的命名空间。 更多详情请参考[一体化卷设计文档](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/all-in-one-volume.md)。

#### 包含 Secret、downwardAPI 和 configMap 的 Pod 示例 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
spec:
  containers:
  - name: container-test
    image: busybox
    volumeMounts:
    - name: all-in-one
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: mysecret
          items:
            - key: username
              path: my-group/my-username
      - downwardAPI:
          items:
            - path: "labels"
              fieldRef:
                fieldPath: metadata.labels
            - path: "cpu_limit"
              resourceFieldRef:
                containerName: container-test
                resource: limits.cpu
      - configMap:
          name: myconfigmap
          items:
            - key: config
              path: my-group/my-config
```

下面是一个带有非默认访问权限设置的多个 secret 的 Pod 示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
spec:
  containers:
  - name: container-test
    image: busybox
    volumeMounts:
    - name: all-in-one
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: mysecret
          items:
            - key: username
              path: my-group/my-username
      - secret:
          name: mysecret2
          items:
            - key: password
              path: my-group/my-password
              mode: 511
```

每个被投射的卷来源都在规约中的 `sources` 内列出。参数几乎相同，除了两处例外：

- 对于 `secret`，`secretName` 字段已被变更为 `name` 以便与 ConfigMap 命名一致。
- `defaultMode` 只能在整个投射卷级别指定，而无法针对每个卷来源指定。 不过，如上所述，你可以显式地为每个投射项设置 `mode` 值。

当开启 `TokenRequestProjection` 功能时，可以将当前 [服务帐号](https://kubernetes.io/zh/docs/reference/access-authn-authz/authentication/#service-account-tokens) 的令牌注入 Pod 中的指定路径。 下面是一个例子：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sa-token-test
spec:
  containers:
  - name: container-test
    image: busybox
    volumeMounts:
    - name: token-vol
      mountPath: "/service-account"
      readOnly: true
  volumes:
  - name: token-vol
    projected:
      sources:
      - serviceAccountToken:
          audience: api
          expirationSeconds: 3600
          path: token
```

示例 Pod 具有包含注入服务帐户令牌的映射卷。 该令牌可以被 Pod 中的容器用来访问 Kubernetes API 服务器。 `audience` 字段包含令牌的预期受众。 令牌的接收者必须使用令牌的受众中指定的标识符来标识自己，否则应拒绝令牌。 此字段是可选的，默认值是 API 服务器的标识符。

`expirationSeconds` 是服务帐户令牌的有效期时长。 默认值为 1 小时，必须至少 10 分钟（600 秒）。 管理员还可以通过设置 API 服务器的 `--service-account-max-token-expiration` 选项来 限制其最大值。 `path` 字段指定相对于映射卷的挂载点的相对路径。

**说明：**

使用投射卷源作为 [subPath](https://kubernetes.io/zh/docs/concepts/storage/volumes/#using-subpath) 卷挂载的容器将不会接收这些卷源的更新。

### quobyte (已弃用)

`quobyte` 卷允许将现有的 [Quobyte](https://www.quobyte.com/) 卷挂载到你的 Pod 中。

**说明：** 在使用 Quobyte 卷之前，你首先要进行安装 Quobyte 并创建好卷。

Quobyte 支持[容器存储接口（CSI）](https://kubernetes.io/zh/docs/concepts/storage/volumes/#csi)。 推荐使用 CSI 插件以在 Kubernetes 中使用 Quobyte 卷。 Quobyte 的 GitHub 项目包含以 CSI 形式部署 Quobyte 的 [说明](https://github.com/quobyte/quobyte-csi#quobyte-csi) 及使用示例。

### rbd

`rbd` 卷允许将 [Rados 块设备](https://docs.ceph.com/en/latest/rbd/) 卷挂载到你的 Pod 中. 不像 `emptyDir` 那样会在删除 Pod 的同时也会被删除，`rbd` 卷的内容在删除 Pod 时 会被保存，卷只是被卸载。 这意味着 `rbd` 卷可以被预先填充数据，并且这些数据可以在 Pod 之间共享。

**注意：** 在使用 RBD 之前，你必须安装运行 Ceph。

RBD 的一个特性是它可以同时被多个用户以只读方式挂载。 这意味着你可以用数据集预先填充卷，然后根据需要在尽可能多的 Pod 中并行地使用卷。 不幸的是，RBD 卷只能由单个使用者以读写模式安装。不允许同时写入。

更多详情请参考 [RBD 示例](https://github.com/kubernetes/examples/tree/master/volumes/rbd)。

### secret

`secret` 卷用来给 Pod 传递敏感信息，例如密码。你可以将 Secret 存储在 Kubernetes API 服务器上，然后以文件的形式挂在到 Pod 中，无需直接与 Kubernetes 耦合。 `secret` 卷由 tmpfs（基于 RAM 的文件系统）提供存储，因此它们永远不会被写入非易失性 （持久化的）存储器。

**说明：** 使用前你必须在 Kubernetes API 中创建 secret。

**说明：** 容器以 [subPath](https://kubernetes.io/zh/docs/concepts/storage/volumes/#using-subpath) 卷挂载方式挂载 Secret 时，将感知不到 Secret 的更新。

更多详情请参考[配置 Secrets](https://kubernetes.io/zh/docs/concepts/configuration/secret/)。

### storageOS (已弃用)

`storageos` 卷允许将现有的 [StorageOS](https://www.storageos.com/) 卷挂载到你的 Pod 中。

StorageOS 在 Kubernetes 环境中以容器的形式运行，这使得应用能够从 Kubernetes 集群中的任何节点访问本地的或挂接的存储。为应对节点失效状况，可以复制数据。 若需提高利用率和降低成本，可以考虑瘦配置（Thin Provisioning）和数据压缩。

作为其核心能力之一，StorageOS 为容器提供了可以通过文件系统访问的块存储。

StorageOS 容器需要 64 位的 Linux，并且没有其他的依赖关系。 StorageOS 提供免费的开发者授权许可。

**注意：** 你必须在每个希望访问 StorageOS 卷的或者将向存储资源池贡献存储容量的节点上运行 StorageOS 容器。有关安装说明，请参阅 [StorageOS 文档](https://docs.storageos.com/)。

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: redis
    role: master
  name: test-storageos-redis
spec:
  containers:
    - name: master
      image: kubernetes/redis:v1
      env:
        - name: MASTER
          value: "true"
      ports:
        - containerPort: 6379
      volumeMounts:
        - mountPath: /redis-master-data
          name: redis-data
  volumes:
    - name: redis-data
      storageos:
        # `redis-vol01` 卷必须在 StorageOS 中存在，并位于 `default` 名字空间内
        volumeName: redis-vol01
        fsType: ext4
```

关于 StorageOS 的进一步信息、动态供应和持久卷申领等等，请参考 [StorageOS 示例](https://github.com/kubernetes/examples/blob/master/volumes/storageos)。

### vsphereVolume

**说明：** 你必须配置 Kubernetes 的 vSphere 云驱动。云驱动的配置方法请参考 [vSphere 使用指南](https://vmware.github.io/vsphere-storage-for-kubernetes/documentation/)。

`vsphereVolume` 用来将 vSphere VMDK 卷挂载到你的 Pod 中。 在卸载卷时，卷的内容会被保留。 vSphereVolume 卷类型支持 VMFS 和 VSAN 数据仓库。

**注意：** 在挂载到 Pod 之前，你必须用下列方式之一创建 VMDK。

#### 创建 VMDK 卷 

选择下列方式之一创建 VMDK。

- [使用 vmkfstools 创建](https://kubernetes.io/zh/docs/concepts/storage/volumes/#tabs-volumes-0)
- [使用 vmware-vdiskmanager 创建](https://kubernetes.io/zh/docs/concepts/storage/volumes/#tabs-volumes-1)



首先 ssh 到 ESX，然后使用下面的命令来创建 VMDK：

```shell
vmkfstools -c 2G /vmfs/volumes/DatastoreName/volumes/myDisk.vmdk
```

#### vSphere VMDK 配置示例 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-vmdk
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-vmdk
      name: test-volume
  volumes:
  - name: test-volume
    # 此 VMDK 卷必须已经存在
    vsphereVolume:
      volumePath: "[DatastoreName] volumes/myDisk"
      fsType: ext4
```

进一步信息可参考 [vSphere 卷](https://github.com/kubernetes/examples/tree/master/staging/volumes/vsphere)。

#### vSphere CSI 迁移 

**FEATURE STATE:** `Kubernetes v1.19 [beta]`

当 `vsphereVolume` 的 `CSIMigration` 特性被启用时，所有插件操作都被从树内插件重定向到 `csi.vsphere.vmware.com` [CSI](https://kubernetes.io/zh/docs/concepts/storage/volumes/#csi) 驱动。 为了使用此功能特性，必须在集群中安装 [vSphere CSI 驱动](https://github.com/kubernetes-sigs/vsphere-csi-driver)， 并启用 `CSIMigration` 和 `CSIMigrationvSphere` [特性门控](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/feature-gates/)。

此特性还要求 vSphere vCenter/ESXi 的版本至少为 7.0u1，且 HW 版本至少为 VM version 15。

**说明：**

vSphere CSI 驱动不支持内置 `vsphereVolume` 的以下 StorageClass 参数：

- `diskformat`
- `hostfailurestotolerate`
- `forceprovisioning`
- `cachereservation`
- `diskstripes`
- `objectspacereservation`
- `iopslimit`

使用这些参数创建的现有卷将被迁移到 vSphere CSI 驱动，不过使用 vSphere CSI 驱动所创建的新卷都不会理会这些参数。

#### vSphere CSI 迁移完成 

**FEATURE STATE:** `Kubernetes v1.19 [beta]`

为了避免控制器管理器和 kubelet 加载 `vsphereVolume` 插件，你需要将 `InTreePluginvSphereUnregister` 特性设置为 `true`。你还必须在所有工作节点上安装 `csi.vsphere.vmware.com` [CSI](https://kubernetes.io/zh/docs/concepts/storage/volumes/#csi) 驱动。

## 使用 subPath 

有时，在单个 Pod 中共享卷以供多方使用是很有用的。 `volumeMounts.subPath` 属性可用于指定所引用的卷内的子路径，而不是其根路径。

下面例子展示了如何配置某包含 LAMP 堆栈（Linux Apache MySQL PHP）的 Pod 使用同一共享卷。 此示例中的 `subPath` 配置不建议在生产环境中使用。 PHP 应用的代码和相关数据映射到卷的 `html` 文件夹，MySQL 数据库存储在卷的 `mysql` 文件夹中：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-lamp-site
spec:
    containers:
    - name: mysql
      image: mysql
      env:
      - name: MYSQL_ROOT_PASSWORD
        value: "rootpasswd"
      volumeMounts:
      - mountPath: /var/lib/mysql
        name: site-data
        subPath: mysql
    - name: php
      image: php:7.0-apache
      volumeMounts:
      - mountPath: /var/www/html
        name: site-data
        subPath: html
    volumes:
    - name: site-data
      persistentVolumeClaim:
        claimName: my-lamp-site-data
```

### 使用带有扩展环境变量的 subPath 

**FEATURE STATE:** `Kubernetes v1.17 [stable]`

使用 `subPathExpr` 字段可以基于 Downward API 环境变量来构造 `subPath` 目录名。 `subPath` 和 `subPathExpr` 属性是互斥的。

在这个示例中，Pod 使用 `subPathExpr` 来 hostPath 卷 `/var/log/pods` 中创建目录 `pod1`。 `hostPath` 卷采用来自 `downwardAPI` 的 Pod 名称生成目录名。 宿主目录 `/var/log/pods/pod1` 被挂载到容器的 `/logs` 中。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: container1
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.name
    image: busybox
    command: [ "sh", "-c", "while [ true ]; do echo 'Hello'; sleep 10; done | tee -a /logs/hello.txt" ]
    volumeMounts:
    - name: workdir1
      mountPath: /logs
      subPathExpr: $(POD_NAME)
  restartPolicy: Never
  volumes:
  - name: workdir1
    hostPath:
      path: /var/log/pods
```

## 资源 

`emptyDir` 卷的存储介质（磁盘、SSD 等）是由保存 kubelet 数据的根目录 （通常是 `/var/lib/kubelet`）的文件系统的介质确定。 Kubernetes 对 `emptyDir` 卷或者 `hostPath` 卷可以消耗的空间没有限制， 容器之间或 Pod 之间也没有隔离。

要了解如何使用资源规约来请求空间，可参考 [如何管理资源](https://kubernetes.io/zh/docs/concepts/configuration/manage-resources-containers/)。

## 树外（Out-of-Tree）卷插件 

Out-of-Tree 卷插件包括 [容器存储接口（CSI）](https://kubernetes.io/zh/docs/concepts/storage/volumes/#csi) (CSI) 和 FlexVolume。 它们使存储供应商能够创建自定义存储插件，而无需将它们添加到 Kubernetes 代码仓库。

以前，所有卷插件（如上面列出的卷类型）都是“树内（In-Tree）”的。 “树内”插件是与 Kubernetes 的核心组件一同构建、链接、编译和交付的。 这意味着向 Kubernetes 添加新的存储系统（卷插件）需要将代码合并到 Kubernetes 核心代码库中。

CSI 和 FlexVolume 都允许独立于 Kubernetes 代码库开发卷插件，并作为扩展部署 （安装）在 Kubernetes 集群上。

对于希望创建树外（Out-Of-Tree）卷插件的存储供应商，请参考 [卷插件常见问题](https://github.com/kubernetes/community/blob/master/sig-storage/volume-plugin-faq.md)。

### CSI

[容器存储接口](https://github.com/container-storage-interface/spec/blob/master/spec.md) (CSI) 为容器编排系统（如 Kubernetes）定义标准接口，以将任意存储系统暴露给它们的容器工作负载。

更多详情请阅读 [CSI 设计方案](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md)。

**说明：** Kubernetes v1.13 废弃了对 CSI 规范版本 0.2 和 0.3 的支持，并将在以后的版本中删除。

**说明：** CSI 驱动可能并非兼容所有的 Kubernetes 版本。 请查看特定 CSI 驱动的文档，以了解各个 Kubernetes 版本所支持的部署步骤以及兼容性列表。

一旦在 Kubernetes 集群上部署了 CSI 兼容卷驱动程序，用户就可以使用 `csi` 卷类型来 挂接、挂载 CSI 驱动所提供的卷。

`csi` 卷可以在 Pod 中以三种方式使用：

- 通过 PersistentVolumeClaim(#persistentvolumeclaim) 对象引用
- 使用[一般性的临时卷](https://kubernetes.io/zh/docs/concepts/storage/ephemeral-volumes/#generic-ephemeral-volume) （Alpha 特性）
- 使用 [CSI 临时卷](https://kubernetes.io/zh/docs/concepts/storage/ephemeral-volumes/#csi-ephemeral-volume)， 前提是驱动支持这种用法（Beta 特性）

存储管理员可以使用以下字段来配置 CSI 持久卷：

- `driver`：指定要使用的卷驱动名称的字符串值。 这个值必须与 CSI 驱动程序在 `GetPluginInfoResponse` 中返回的值相对应； 该接口定义在 [CSI 规范](https://github.com/container-storage-interface/spec/blob/master/spec.md#getplugininfo)中。 Kubernetes 使用所给的值来标识要调用的 CSI 驱动程序；CSI 驱动程序也使用该值来辨识 哪些 PV 对象属于该 CSI 驱动程序。

- `volumeHandle`：唯一标识卷的字符串值。 该值必须与 CSI 驱动在 `CreateVolumeResponse` 的 `volume_id` 字段中返回的值相对应； 接口定义在 [CSI spec](https://github.com/container-storage-interface/spec/blob/master/spec.md#createvolume) 中。 在所有对 CSI 卷驱动程序的调用中，引用该 CSI 卷时都使用此值作为 `volume_id` 参数。

- `readOnly`：一个可选的布尔值，指示通过 `ControllerPublished` 关联该卷时是否设置 该卷为只读。默认值是 false。 该值通过 `ControllerPublishVolumeRequest` 中的 `readonly` 字段传递给 CSI 驱动。

- `fsType`：如果 PV 的 `VolumeMode` 为 `Filesystem`，那么此字段指定挂载卷时应该使用的文件系统。 如果卷尚未格式化，并且支持格式化，此值将用于格式化卷。 此值可以通过 `ControllerPublishVolumeRequest`、`NodeStageVolumeRequest` 和 `NodePublishVolumeRequest` 的 `VolumeCapability` 字段传递给 CSI 驱动。

- `volumeAttributes`：一个字符串到字符串的映射表，用来设置卷的静态属性。 该映射必须与 CSI 驱动程序返回的 `CreateVolumeResponse` 中的 `volume.attributes` 字段的映射相对应； [CSI 规范](https://github.com/container-storage-interface/spec/blob/master/spec.md#createvolume) 中有相应的定义。 该映射通过`ControllerPublishVolumeRequest`、`NodeStageVolumeRequest`、和 `NodePublishVolumeRequest` 中的 `volume_attributes` 字段传递给 CSI 驱动。

- `controllerPublishSecretRef`：对包含敏感信息的 Secret 对象的引用； 该敏感信息会被传递给 CSI 驱动来完成 CSI `ControllerPublishVolume` 和 `ControllerUnpublishVolume` 调用。 此字段是可选的；在不需要 Secret 时可以是空的。 如果 Secret 对象包含多个 Secret 条目，则所有的 Secret 条目都会被传递。

- `nodeStageSecretRef`：对包含敏感信息的 Secret 对象的引用。 该信息会传递给 CSI 驱动来完成 CSI `NodeStageVolume` 调用。 此字段是可选的，如果不需要 Secret，则可能是空的。 如果 Secret 对象包含多个 Secret 条目，则传递所有 Secret 条目。

- `nodePublishSecretRef`：对包含敏感信息的 Secret 对象的引用。 该信息传递给 CSI 驱动来完成 CSI `NodePublishVolume` 调用。 此字段是可选的，如果不需要 Secret，则可能是空的。 如果 Secret 对象包含多个 Secret 条目，则传递所有 Secret 条目。

#### CSI 原始块卷支持 

**FEATURE STATE:** `Kubernetes v1.18 [stable]`

具有外部 CSI 驱动程序的供应商能够在 Kubernetes 工作负载中实现原始块卷支持。

你可以和以前一样，安装自己的 [带有原始块卷支持的 PV/PVC](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#raw-block-volume-support)， 采用 CSI 对此过程没有影响。

#### CSI 临时卷 

**FEATURE STATE:** `Kubernetes v1.16 [beta]`

你可以直接在 Pod 规约中配置 CSI 卷。采用这种方式配置的卷都是临时卷， 无法在 Pod 重新启动后继续存在。 进一步的信息可参阅[临时卷](https://kubernetes.io/zh/docs/concepts/storage/ephemeral-volumes/#csi-ephemeral-volume)。

有关如何开发 CSI 驱动的更多信息，请参考 [kubernetes-csi 文档](https://kubernetes-csi.github.io/docs/)。

#### 从树内插件迁移到 CSI 驱动程序 

**FEATURE STATE:** `Kubernetes v1.17 [beta]`

启用 `CSIMigration` 功能后，针对现有树内插件的操作会被重定向到相应的 CSI 插件 （应已安装和配置）。 因此，操作员在过渡到取代树内插件的 CSI 驱动时，无需对现有存储类、PV 或 PVC （指树内插件）进行任何配置更改。

所支持的操作和功能包括：配备（Provisioning）/删除、挂接（Attach）/解挂（Detach）、 挂载（Mount）/卸载（Unmount）和调整卷大小。

上面的[卷类型](https://kubernetes.io/zh/docs/concepts/storage/volumes/#volume-types)节列出了支持 `CSIMigration` 并已实现相应 CSI 驱动程序的树内插件。

### flexVolume

FlexVolume 是一个自 1.2 版本（在 CSI 之前）以来在 Kubernetes 中一直存在的树外插件接口。 它使用基于 exec 的模型来与驱动程序对接。 用户必须在每个节点（在某些情况下是主控节点）上的预定义卷插件路径中安装 FlexVolume 驱动程序可执行文件。

Pod 通过 `flexvolume` 树内插件与 Flexvolume 驱动程序交互。 更多详情请参考 [FlexVolume](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-storage/flexvolume.md) 示例。

## 挂载卷的传播 

挂载卷的传播能力允许将容器安装的卷共享到同一 Pod 中的其他容器， 甚至共享到同一节点上的其他 Pod。

卷的挂载传播特性由 `Container.volumeMounts` 中的 `mountPropagation` 字段控制。 它的值包括：

- `None` - 此卷挂载将不会感知到主机后续在此卷或其任何子目录上执行的挂载变化。 类似的，容器所创建的卷挂载在主机上是不可见的。这是默认模式。

  该模式等同于 [Linux 内核文档](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt) 中描述的 `private` 挂载传播选项。

- `HostToContainer` - 此卷挂载将会感知到主机后续针对此卷或其任何子目录的挂载操作。

  换句话说，如果主机在此挂载卷中挂载任何内容，容器将能看到它被挂载在那里。

  类似的，配置了 `Bidirectional` 挂载传播选项的 Pod 如果在同一卷上挂载了内容， 挂载传播设置为 `HostToContainer` 的容器都将能看到这一变化。

  该模式等同于 [Linux 内核文档](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt) 中描述的 `rslave` 挂载传播选项。

- `Bidirectional` - 这种卷挂载和 `HostToContainer` 挂载表现相同。 另外，容器创建的卷挂载将被传播回至主机和使用同一卷的所有 Pod 的所有容器。

  该模式等同于 [Linux 内核文档](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt) 中描述的 `rshared` 挂载传播选项。

  **警告：** `Bidirectional` 形式的挂载传播可能比较危险。 它可以破坏主机操作系统，因此它只被允许在特权容器中使用。 强烈建议你熟悉 Linux 内核行为。 此外，由 Pod 中的容器创建的任何卷挂载必须在终止时由容器销毁（卸载）。

### 配置 

在某些部署环境中，挂载传播正常工作前，必须在 Docker 中正确配置挂载共享（mount share）， 如下所示。

编辑你的 Docker `systemd` 服务文件，按下面的方法设置 `MountFlags`：

```shell
MountFlags=shared
```

或者，如果存在 `MountFlags=slave` 就删除掉。然后重启 Docker 守护进程：

```shell
sudo systemctl daemon-reload
sudo systemctl restart docker
```

# 持久卷

本文描述 Kubernetes 中 *持久卷（Persistent Volume）* 的当前状态。 建议先熟悉[卷（Volume）](https://kubernetes.io/zh/docs/concepts/storage/volumes/)的概念。

## 介绍 

存储的管理是一个与计算实例的管理完全不同的问题。PersistentVolume 子系统为用户 和管理员提供了一组 API，将存储如何供应的细节从其如何被使用中抽象出来。 为了实现这点，我们引入了两个新的 API 资源：PersistentVolume 和 PersistentVolumeClaim。

持久卷（PersistentVolume，PV）是集群中的一块存储，可以由管理员事先供应，或者 使用[存储类（Storage Class）](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/)来动态供应。 持久卷是集群资源，就像节点也是集群资源一样。PV 持久卷和普通的 Volume 一样，也是使用 卷插件来实现的，只是它们拥有独立于任何使用 PV 的 Pod 的生命周期。 此 API 对象中记述了存储的实现细节，无论其背后是 NFS、iSCSI 还是特定于云平台的存储系统。

持久卷申领（PersistentVolumeClaim，PVC）表达的是用户对存储的请求。概念上与 Pod 类似。 Pod 会耗用节点资源，而 PVC 申领会耗用 PV 资源。Pod 可以请求特定数量的资源（CPU 和内存）；同样 PVC 申领也可以请求特定的大小和访问模式 （例如，可以要求 PV 卷能够以 ReadWriteOnce、ReadOnlyMany 或 ReadWriteMany 模式之一来挂载，参见[访问模式](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#access-modes)）。

尽管 PersistentVolumeClaim 允许用户消耗抽象的存储资源，常见的情况是针对不同的 问题用户需要的是具有不同属性（如，性能）的 PersistentVolume 卷。 集群管理员需要能够提供不同性质的 PersistentVolume，并且这些 PV 卷之间的差别不 仅限于卷大小和访问模式，同时又不能将卷是如何实现的这些细节暴露给用户。 为了满足这类需求，就有了 *存储类（StorageClass）* 资源。

参见[基于运行示例的详细演练](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-persistent-volume-storage/)。

## 卷和申领的生命周期 

PV 卷是集群中的资源。PVC 申领是对这些资源的请求，也被用来执行对资源的申领检查。 PV 卷和 PVC 申领之间的互动遵循如下生命周期：

### 供应 

PV 卷的供应有两种方式：静态供应或动态供应。

#### 静态供应 

集群管理员创建若干 PV 卷。这些卷对象带有真实存储的细节信息，并且对集群 用户可用（可见）。PV 卷对象存在于 Kubernetes API 中，可供用户消费（使用）。

#### 动态供应 

如果管理员所创建的所有静态 PV 卷都无法与用户的 PersistentVolumeClaim 匹配， 集群可以尝试为该 PVC 申领动态供应一个存储卷。 这一供应操作是基于 StorageClass 来实现的：PVC 申领必须请求某个 [存储类](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/)，同时集群管理员必须 已经创建并配置了该类，这样动态供应卷的动作才会发生。 如果 PVC 申领指定存储类为 `""`，则相当于为自身禁止使用动态供应的卷。

为了基于存储类完成动态的存储供应，集群管理员需要在 API 服务器上启用 `DefaultStorageClass` [准入控制器](https://kubernetes.io/zh/docs/reference/access-authn-authz/admission-controllers/#defaultstorageclass)。 举例而言，可以通过保证 `DefaultStorageClass` 出现在 API 服务器组件的 `--enable-admission-plugins` 标志值中实现这点；该标志的值可以是逗号 分隔的有序列表。关于 API 服务器标志的更多信息，可以参考 [kube-apiserver](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-apiserver/) 文档。

### 绑定 

用户创建一个带有特定存储容量和特定访问模式需求的 PersistentVolumeClaim 对象； 在动态供应场景下，这个 PVC 对象可能已经创建完毕。 主控节点中的控制回路监测新的 PVC 对象，寻找与之匹配的 PV 卷（如果可能的话）， 并将二者绑定到一起。 如果为了新的 PVC 申领动态供应了 PV 卷，则控制回路总是将该 PV 卷绑定到这一 PVC 申领。 否则，用户总是能够获得他们所请求的资源，只是所获得的 PV 卷可能会超出所请求的配置。 一旦绑定关系建立，则 PersistentVolumeClaim 绑定就是排他性的，无论该 PVC 申领是 如何与 PV 卷建立的绑定关系。 PVC 申领与 PV 卷之间的绑定是一种一对一的映射，实现上使用 ClaimRef 来记述 PV 卷 与 PVC 申领间的双向绑定关系。

如果找不到匹配的 PV 卷，PVC 申领会无限期地处于未绑定状态。 当与之匹配的 PV 卷可用时，PVC 申领会被绑定。 例如，即使某集群上供应了很多 50 Gi 大小的 PV 卷，也无法与请求 100 Gi 大小的存储的 PVC 匹配。当新的 100 Gi PV 卷被加入到集群时，该 PVC 才有可能被绑定。

### 使用 

Pod 将 PVC 申领当做存储卷来使用。集群会检视 PVC 申领，找到所绑定的卷，并 为 Pod 挂载该卷。对于支持多种访问模式的卷，用户要在 Pod 中以卷的形式使用申领 时指定期望的访问模式。

一旦用户有了申领对象并且该申领已经被绑定，则所绑定的 PV 卷在用户仍然需要它期间 一直属于该用户。用户通过在 Pod 的 `volumes` 块中包含 `persistentVolumeClaim` 节区来调度 Pod，访问所申领的 PV 卷。 相关细节可参阅[使用申领作为卷](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#claims-as-volumes)。

### 保护使用中的存储对象 

保护使用中的存储对象（Storage Object in Use Protection）这一功能特性的目的 是确保仍被 Pod 使用的 PersistentVolumeClaim（PVC）对象及其所绑定的 PersistentVolume（PV）对象在系统中不会被删除，因为这样做可能会引起数据丢失。

**说明：** 当使用某 PVC 的 Pod 对象仍然存在时，认为该 PVC 仍被此 Pod 使用。

如果用户删除被某 Pod 使用的 PVC 对象，该 PVC 申领不会被立即移除。 PVC 对象的移除会被推迟，直至其不再被任何 Pod 使用。 此外，如果管理员删除已绑定到某 PVC 申领的 PV 卷，该 PV 卷也不会被立即移除。 PV 对象的移除也要推迟到该 PV 不再绑定到 PVC。

你可以看到当 PVC 的状态为 `Terminating` 且其 `Finalizers` 列表中包含 `kubernetes.io/pvc-protection` 时，PVC 对象是处于被保护状态的。

```shell
kubectl describe pvc hostpath
Name:          hostpath
Namespace:     default
StorageClass:  example-hostpath
Status:        Terminating
Volume:
Labels:        <none>
Annotations:   volume.beta.kubernetes.io/storage-class=example-hostpath
               volume.beta.kubernetes.io/storage-provisioner=example.com/hostpath
Finalizers:    [kubernetes.io/pvc-protection]
...
```

你也可以看到当 PV 对象的状态为 `Terminating` 且其 `Finalizers` 列表中包含 `kubernetes.io/pv-protection` 时，PV 对象是处于被保护状态的。

```shell
kubectl describe pv task-pv-volume
Name:            task-pv-volume
Labels:          type=local
Annotations:     <none>
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    standard
Status:          Terminating
Claim:
Reclaim Policy:  Delete
Access Modes:    RWO
Capacity:        1Gi
Message:
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /tmp/data
    HostPathType:
Events:            <none>
```

### 回收 

当用户不再使用其存储卷时，他们可以从 API 中将 PVC 对象删除，从而允许 该资源被回收再利用。PersistentVolume 对象的回收策略告诉集群，当其被 从申领中释放时如何处理该数据卷。 目前，数据卷可以被 Retained（保留）、Recycled（回收）或 Deleted（删除）。

#### 保留（Retain） 

回收策略 `Retain` 使得用户可以手动回收资源。当 PersistentVolumeClaim 对象 被删除时，PersistentVolume 卷仍然存在，对应的数据卷被视为"已释放（released）"。 由于卷上仍然存在这前一申领人的数据，该卷还不能用于其他申领。 管理员可以通过下面的步骤来手动回收该卷：

1. 删除 PersistentVolume 对象。与之相关的、位于外部基础设施中的存储资产 （例如 AWS EBS、GCE PD、Azure Disk 或 Cinder 卷）在 PV 删除之后仍然存在。
2. 根据情况，手动清除所关联的存储资产上的数据。
3. 手动删除所关联的存储资产。

如果你希望重用该存储资产，可以基于存储资产的定义创建新的 PersistentVolume 卷对象。

#### 删除（Delete） 

对于支持 `Delete` 回收策略的卷插件，删除动作会将 PersistentVolume 对象从 Kubernetes 中移除，同时也会从外部基础设施（如 AWS EBS、GCE PD、Azure Disk 或 Cinder 卷）中移除所关联的存储资产。 动态供应的卷会继承[其 StorageClass 中设置的回收策略](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#reclaim-policy)，该策略默认 为 `Delete`。 管理员需要根据用户的期望来配置 StorageClass；否则 PV 卷被创建之后必须要被 编辑或者修补。参阅[更改 PV 卷的回收策略](https://kubernetes.io/zh/docs/tasks/administer-cluster/change-pv-reclaim-policy/).

#### 回收（Recycle） 

**警告：** 回收策略 `Recycle` 已被废弃。取而代之的建议方案是使用动态供应。

如果下层的卷插件支持，回收策略 `Recycle` 会在卷上执行一些基本的 擦除（`rm -rf /thevolume/*`）操作，之后允许该卷用于新的 PVC 申领。

不过，管理员可以按 [参考资料](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-controller-manager/) 中所述，使用 Kubernetes 控制器管理器命令行参数来配置一个定制的回收器（Recycler） Pod 模板。此定制的回收器 Pod 模板必须包含一个 `volumes` 规约，如下例所示：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pv-recycler
  namespace: default
spec:
  restartPolicy: Never
  volumes:
  - name: vol
    hostPath:
      path: /any/path/it/will/be/replaced
  containers:
  - name: pv-recycler
    image: "k8s.gcr.io/busybox"
    command: ["/bin/sh", "-c", "test -e /scrub && rm -rf /scrub/..?* /scrub/.[!.]* /scrub/*  && test -z \"$(ls -A /scrub)\" || exit 1"]
    volumeMounts:
    - name: vol
      mountPath: /scrub
```

定制回收器 Pod 模板中在 `volumes` 部分所指定的特定路径要替换为 正被回收的卷的路径。

### 预留 PersistentVolume 

通过在 PersistentVolumeClaim 中指定 PersistentVolume，你可以声明该特定 PV 与 PVC 之间的绑定关系。如果该 PersistentVolume 存在且未被通过其 `claimRef` 字段预留给 PersistentVolumeClaim，则该 PersistentVolume 会和该 PersistentVolumeClaim 绑定到一起。

绑定操作不会考虑某些卷匹配条件是否满足，包括节点亲和性等等。 控制面仍然会检查 [存储类](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/)、访问模式和所请求的 存储尺寸都是合法的。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: foo-pvc
  namespace: foo
spec:
  storageClassName: "" # 此处须显式设置空字符串，否则会被设置为默认的 StorageClass
  volumeName: foo-pv
  ...
```

此方法无法对 PersistentVolume 的绑定特权做出任何形式的保证。 如果有其他 PersistentVolumeClaim 可以使用你所指定的 PV，则你应该首先预留 该存储卷。你可以将 PV 的 `claimRef` 字段设置为相关的 PersistentVolumeClaim 以确保其他 PVC 不会绑定到该 PV 卷。

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: foo-pv
spec:
  storageClassName: ""
  claimRef:
    name: foo-pvc
    namespace: foo
  ...
```

如果你想要使用 `claimPolicy` 属性设置为 `Retain` 的 PersistentVolume 卷 时，包括你希望复用现有的 PV 卷时，这点是很有用的

### 扩充 PVC 申领 

**FEATURE STATE:** `Kubernetes v1.11 [beta]`

现在，对扩充 PVC 申领的支持默认处于被启用状态。你可以扩充以下类型的卷：

- gcePersistentDisk
- awsElasticBlockStore
- Cinder
- glusterfs
- rbd
- Azure File
- Azure Disk
- Portworx
- FlexVolumes [CSI](https://kubernetes.io/zh/docs/concepts/storage/volumes/#csi)

只有当 PVC 的存储类中将 `allowVolumeExpansion` 设置为 true 时，你才可以扩充该 PVC 申领。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gluster-vol-default
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://192.168.10.100:8080"
  restuser: ""
  secretNamespace: ""
  secretName: ""
allowVolumeExpansion: true
```

如果要为某 PVC 请求较大的存储卷，可以编辑 PVC 对象，设置一个更大的尺寸值。 这一编辑操作会触发为下层 PersistentVolume 提供存储的卷的扩充。 Kubernetes 不会创建新的 PV 卷来满足此申领的请求。 与之相反，现有的卷会被调整大小。

#### CSI 卷的扩充 

**FEATURE STATE:** `Kubernetes v1.16 [beta]`

对 CSI 卷的扩充能力默认是被启用的，不过扩充 CSI 卷要求 CSI 驱动支持 卷扩充操作。可参阅特定 CSI 驱动的文档了解更多信息。

#### 重设包含文件系统的卷的大小

只有卷中包含的文件系统是 XFS、Ext3 或者 Ext4 时，你才可以重设卷的大小。

当卷中包含文件系统时，只有在 Pod 使用 `ReadWrite` 模式来使用 PVC 申领的 情况下才能重设其文件系统的大小。 文件系统扩充的操作或者是在 Pod 启动期间完成，或者在下层文件系统支持在线 扩充的前提下在 Pod 运行期间完成。

如果 FlexVolumes 的驱动将 `RequiresFSResize` 能力设置为 `true`，则该 FlexVolume 卷可以在 Pod 重启期间调整大小。

#### 重设使用中 PVC 申领的大小 

**FEATURE STATE:** `Kubernetes v1.15 [beta]`

**说明：** Kubernetes 从 1.15 版本开始将调整使用中 PVC 申领大小这一能力作为 Beta 特性支持；该特性在 1.11 版本以来处于 Alpha 阶段。 `ExpandInUsePersistentVolumes` 特性必须被启用；在很多集群上，与此类似的 Beta 阶段的特性是自动启用的。 可参考[特性门控](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/feature-gates/) 文档了解更多信息。

在这种情况下，你不需要删除和重建正在使用某现有 PVC 的 Pod 或 Deployment。 所有使用中的 PVC 在其文件系统被扩充之后，立即可供其 Pod 使用。 此功能特性对于没有被 Pod 或 Deployment 使用的 PVC 而言没有效果。 你必须在执行扩展操作之前创建一个使用该 PVC 的 Pod。

与其他卷类型类似，FlexVolume 卷也可以在被 Pod 使用期间执行扩充操作。

**说明：** FlexVolume 卷的重设大小只能在下层驱动支持重设大小的时候才可进行。

**说明：** 扩充 EBS 卷的操作非常耗时。同时还存在另一个配额限制： 每 6 小时只能执行一次（尺寸）修改操作。

#### 处理扩充卷过程中的失败 

如果扩充下层存储的操作失败，集群管理员可以手动地恢复 PVC 申领的状态并 取消重设大小的请求。否则，在没有管理员干预的情况下，控制器会反复重试 重设大小的操作。

1. 将绑定到 PVC 申领的 PV 卷标记为 `Retain` 回收策略；
2. 删除 PVC 对象。由于 PV 的回收策略为 `Retain`，我们不会在重建 PVC 时丢失数据。
3. 删除 PV 规约中的 `claimRef` 项，这样新的 PVC 可以绑定到该卷。 这一操作会使得 PV 卷变为 "可用（Available）"。
4. 使用小于 PV 卷大小的尺寸重建 PVC，设置 PVC 的 `volumeName` 字段为 PV 卷的名称。 这一操作将把新的 PVC 对象绑定到现有的 PV 卷。
5. 不要忘记恢复 PV 卷上设置的回收策略。

## 持久卷的类型 

PV 持久卷是用插件的形式来实现的。Kubernetes 目前支持以下插件：

- [`awsElasticBlockStore`](https://kubernetes.io/zh/docs/concepts/storage/volumes/#awselasticblockstore) - AWS 弹性块存储（EBS）
- [`azureDisk`](https://kubernetes.io/zh/docs/concepts/storage/volumes/#azuredisk) - Azure Disk
- [`azureFile`](https://kubernetes.io/zh/docs/concepts/storage/volumes/#azurefile) - Azure File
- [`cephfs`](https://kubernetes.io/zh/docs/concepts/storage/volumes/#cephfs) - CephFS volume
- [`csi`](https://kubernetes.io/zh/docs/concepts/storage/volumes/#csi) - 容器存储接口 (CSI)
- [`fc`](https://kubernetes.io/zh/docs/concepts/storage/volumes/#fc) - Fibre Channel (FC) 存储
- [`flexVolume`](https://kubernetes.io/zh/docs/concepts/storage/volumes/#flexVolume) - FlexVolume
- [`gcePersistentDisk`](https://kubernetes.io/zh/docs/concepts/storage/volumes/#gcepersistentdisk) - GCE 持久化盘
- [`glusterfs`](https://kubernetes.io/zh/docs/concepts/storage/volumes/#glusterfs) - Glusterfs 卷
- [`hostPath`](https://kubernetes.io/zh/docs/concepts/storage/volumes/#hostpath) - HostPath 卷 （仅供单节点测试使用；不适用于多节点集群； 请尝试使用 `local` 卷作为替代）
- [`iscsi`](https://kubernetes.io/zh/docs/concepts/storage/volumes/#iscsi) - iSCSI (SCSI over IP) 存储
- [`local`](https://kubernetes.io/zh/docs/concepts/storage/volumes/#local) - 节点上挂载的本地存储设备
- [`nfs`](https://kubernetes.io/zh/docs/concepts/storage/volumes/#nfs) - 网络文件系统 (NFS) 存储
- [`portworxVolume`](https://kubernetes.io/zh/docs/concepts/storage/volumes/#portworxvolume) - Portworx 卷
- [`rbd`](https://kubernetes.io/zh/docs/concepts/storage/volumes/#rbd) - Rados 块设备 (RBD) 卷
- [`vsphereVolume`](https://kubernetes.io/zh/docs/concepts/storage/volumes/#vspherevolume) - vSphere VMDK 卷

以下的持久卷已被弃用。这意味着当前仍是支持的，但是 Kubernetes 将来的发行版会将其移除。

- [`cinder`](https://kubernetes.io/docs/concepts/storage/volumes/#cinder) - Cinder（OpenStack 块存储）（于 v1.18 **弃用**）
- [`flocker`](https://kubernetes.io/docs/concepts/storage/volumes/#flocker) - Flocker 存储（于 v1.22 **弃用**）
- [`quobyte`](https://kubernetes.io/docs/concepts/storage/volumes/#quobyte) - Quobyte 卷 （于 v1.22 **弃用**）
- [`storageos`](https://kubernetes.io/docs/concepts/storage/volumes/#storageos) - StorageOS 卷（于 v1.22 **弃用**）

旧版本的 Kubernetes 仍支持这些“树内（In-Tree）”持久卷类型：

- `photonPersistentDisk` - Photon 控制器持久化盘。（v1.15 之后 **不可用**）
- [`scaleIO`](https://kubernetes.io/docs/concepts/storage/volumes/#scaleio) - ScaleIO 卷（v1.21 之后 **不可用**）

## 持久卷 

每个 PV 对象都包含 `spec` 部分和 `status` 部分，分别对应卷的规约和状态。 PersistentVolume 对象的名称必须是合法的 [DNS 子域名](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/names#dns-subdomain-names).

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
```

**说明：** 在集群中使用持久卷存储通常需要一些特定于具体卷类型的辅助程序。 在这个例子中，PersistentVolume 是 NFS 类型的，因此需要辅助程序 `/sbin/mount.nfs` 来支持挂载 NFS 文件系统。

### 容量 

一般而言，每个 PV 卷都有确定的存储容量。 容量属性是使用 PV 对象的 `capacity` 属性来设置的。 参考 Kubernetes [资源模型（Resource Model）](https://git.k8s.io/community/contributors/design-proposals/scheduling/resources.md) 设计提案，了解 `capacity` 字段可以接受的单位。

目前，存储大小是可以设置和请求的唯一资源。 未来可能会包含 IOPS、吞吐量等属性。

### 卷模式 

**FEATURE STATE:** `Kubernetes v1.18 [stable]`

针对 PV 持久卷，Kubernetes 支持两种卷模式（`volumeModes`）：`Filesystem（文件系统）` 和 `Block（块）`。 `volumeMode` 是一个可选的 API 参数。 如果该参数被省略，默认的卷模式是 `Filesystem`。

`volumeMode` 属性设置为 `Filesystem` 的卷会被 Pod *挂载（Mount）* 到某个目录。 如果卷的存储来自某块设备而该设备目前为空，Kuberneretes 会在第一次挂载卷之前 在设备上创建文件系统。

你可以将 `volumeMode` 设置为 `Block`，以便将卷作为原始块设备来使用。 这类卷以块设备的方式交给 Pod 使用，其上没有任何文件系统。 这种模式对于为 Pod 提供一种使用最快可能方式来访问卷而言很有帮助，Pod 和 卷之间不存在文件系统层。另外，Pod 中运行的应用必须知道如何处理原始块设备。 关于如何在 Pod 中使用 `volumeMode: Block` 的卷，可参阅 [原始块卷支持](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#raw-block-volume-support)。

### 访问模式 

PersistentVolume 卷可以用资源提供者所支持的任何方式挂载到宿主系统上。 如下表所示，提供者（驱动）的能力不同，每个 PV 卷的访问模式都会设置为 对应卷所支持的模式值。 例如，NFS 可以支持多个读写客户，但是某个特定的 NFS PV 卷可能在服务器 上以只读的方式导出。每个 PV 卷都会获得自身的访问模式集合，描述的是 特定 PV 卷的能力。

访问模式有：

- `ReadWriteOnce`

  卷可以被一个节点以读写方式挂载。 ReadWriteOnce 访问模式也允许运行在同一节点上的多个 Pod 访问卷。

- `ReadOnlyMany`

  卷可以被多个节点以只读方式挂载。

- `ReadWriteMany`

  卷可以被多个节点以读写方式挂载。

- `ReadWriteOncePod`

  卷可以被单个 Pod 以读写方式挂载。 如果你想确保整个集群中只有一个 Pod 可以读取或写入该 PVC， 请使用ReadWriteOncePod 访问模式。这只支持 CSI 卷以及需要 Kubernetes 1.22 以上版本。

这篇博客文章 [Introducing Single Pod Access Mode for PersistentVolumes](https://kubernetes.io/blog/2021/09/13/read-write-once-pod-access-mode-alpha/) 描述了更详细的内容。

在命令行接口（CLI）中，访问模式也使用以下缩写形式：

- RWO - ReadWriteOnce
- ROX - ReadOnlyMany
- RWX - ReadWriteMany
- RWOP - ReadWriteOncePod

> **重要提醒！** 每个卷同一时刻只能以一种访问模式挂载，即使该卷能够支持 多种访问模式。例如，一个 GCEPersistentDisk 卷可以被某节点以 ReadWriteOnce 模式挂载，或者被多个节点以 ReadOnlyMany 模式挂载，但不可以同时以两种模式 挂载。

| 卷插件               | ReadWriteOnce | ReadOnlyMany |          ReadWriteMany           | ReadWriteOncePod |
| :------------------- | :-----------: | :----------: | :------------------------------: | ---------------- |
| AWSElasticBlockStore |       ✓       |      -       |                -                 | -                |
| AzureFile            |       ✓       |      ✓       |                ✓                 | -                |
| AzureDisk            |       ✓       |      -       |                -                 | -                |
| CephFS               |       ✓       |      ✓       |                ✓                 | -                |
| Cinder               |       ✓       |      -       |                -                 | -                |
| CSI                  |  取决于驱动   |  取决于驱动  |            取决于驱动            | 取决于驱动       |
| FC                   |       ✓       |      ✓       |                -                 | -                |
| FlexVolume           |       ✓       |      ✓       |            取决于驱动            | -                |
| Flocker              |       ✓       |      -       |                -                 | -                |
| GCEPersistentDisk    |       ✓       |      ✓       |                -                 | -                |
| Glusterfs            |       ✓       |      ✓       |                ✓                 | -                |
| HostPath             |       ✓       |      -       |                -                 | -                |
| iSCSI                |       ✓       |      ✓       |                -                 | -                |
| Quobyte              |       ✓       |      ✓       |                ✓                 | -                |
| NFS                  |       ✓       |      ✓       |                ✓                 | -                |
| RBD                  |       ✓       |      ✓       |                -                 | -                |
| VsphereVolume        |       ✓       |      -       | - （Pod 运行于同一节点上时可行） | -                |
| PortworxVolume       |       ✓       |      -       |                ✓                 | -                |
| StorageOS            |       ✓       |      -       |                -                 | -                |

### 类 

每个 PV 可以属于某个类（Class），通过将其 `storageClassName` 属性设置为某个 [StorageClass](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/) 的名称来指定。 特定类的 PV 卷只能绑定到请求该类存储卷的 PVC 申领。 未设置 `storageClassName` 的 PV 卷没有类设定，只能绑定到那些没有指定特定 存储类的 PVC 申领。

早前，Kubernetes 使用注解 `volume.beta.kubernetes.io/storage-class` 而不是 `storageClassName` 属性。这一注解目前仍然起作用，不过在将来的 Kubernetes 发布版本中该注解会被彻底废弃。

### 回收策略 

目前的回收策略有：

- Retain -- 手动回收
- Recycle -- 基本擦除 (`rm -rf /thevolume/*`)
- Delete -- 诸如 AWS EBS、GCE PD、Azure Disk 或 OpenStack Cinder 卷这类关联存储资产也被删除

目前，仅 NFS 和 HostPath 支持回收（Recycle）。 AWS EBS、GCE PD、Azure Disk 和 Cinder 卷都支持删除（Delete）。

### 挂载选项 

Kubernetes 管理员可以指定持久卷被挂载到节点上时使用的附加挂载选项。

**说明：** 并非所有持久卷类型都支持挂载选项。

以下卷类型支持挂载选项：

- AWSElasticBlockStore
- AzureDisk
- AzureFile
- CephFS
- Cinder （OpenStack 块存储）
- GCEPersistentDisk
- Glusterfs
- NFS
- Quobyte 卷
- RBD （Ceph 块设备）
- StorageOS
- VsphereVolume
- iSCSI

Kubernetes 不对挂载选项执行合法性检查。如果挂载选项是非法的，挂载就会失败。

早前，Kubernetes 使用注解 `volume.beta.kubernetes.io/mount-options` 而不是 `mountOptions` 属性。这一注解目前仍然起作用，不过在将来的 Kubernetes 发布版本中该注解会被彻底废弃。

### 节点亲和性 

每个 PV 卷可以通过设置节点亲和性来定义一些约束，进而限制从哪些节点上可以访问此卷。 使用这些卷的 Pod 只会被调度到节点亲和性规则所选择的节点上执行。 要设置节点亲和性，配置 PV 卷 `.spec` 中的 `nodeAffinity`。 [持久卷](https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/persistent-volume-v1/#PersistentVolumeSpec) API 参考关于该字段的更多细节。

**说明：** 对大多数类型的卷而言，你不需要设置节点亲和性字段。 [AWS EBS](https://kubernetes.io/zh/docs/concepts/storage/volumes/#awselasticblockstore)、 [GCE PD](https://kubernetes.io/zh/docs/concepts/storage/volumes/#gcepersistentdisk) 和 [Azure Disk](https://kubernetes.io/zh/docs/concepts/storage/volumes/#azuredisk) 卷类型都能 自动设置相关字段。 你需要为 [local](https://kubernetes.io/zh/docs/concepts/storage/volumes/#local) 卷显式地设置 此属性。

### 阶段 

每个卷会处于以下阶段（Phase）之一：

- Available（可用）-- 卷是一个空闲资源，尚未绑定到任何申领；
- Bound（已绑定）-- 该卷已经绑定到某申领；
- Released（已释放）-- 所绑定的申领已被删除，但是资源尚未被集群回收；
- Failed（失败）-- 卷的自动回收操作失败。

命令行接口能够显示绑定到某 PV 卷的 PVC 对象。

## PersistentVolumeClaims

每个 PVC 对象都有 `spec` 和 `status` 部分，分别对应申领的规约和状态。 PersistentVolumeClaim 对象的名称必须是合法的 [DNS 子域名](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/names#dns-subdomain-names).

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```

### 访问模式

申领在请求具有特定访问模式的存储时，使用与卷相同的[访问模式约定](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#access-modes)。

### 卷模式

申领使用[与卷相同的约定](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#access-modes)来表明是将卷作为文件系统还是块设备来使用。

### 资源 

申领和 Pod 一样，也可以请求特定数量的资源。在这个上下文中，请求的资源是存储。 卷和申领都使用相同的 [资源模型](https://git.k8s.io/community/contributors/design-proposals/scheduling/resources.md)。

### 选择算符 

申领可以设置[标签选择算符](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/#label-selectors) 来进一步过滤卷集合。只有标签与选择算符相匹配的卷能够绑定到申领上。 选择算符包含两个字段：

- `matchLabels` - 卷必须包含带有此值的标签
- `matchExpressions` - 通过设定键（key）、值列表和操作符（operator） 来构造的需求。合法的操作符有 In、NotIn、Exists 和 DoesNotExist。

来自 `matchLabels` 和 `matchExpressions` 的所有需求都按逻辑与的方式组合在一起。 这些需求都必须被满足才被视为匹配。

### 类 

申领可以通过为 `storageClassName` 属性设置 [StorageClass](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/) 的名称来请求特定的存储类。 只有所请求的类的 PV 卷，即 `storageClassName` 值与 PVC 设置相同的 PV 卷， 才能绑定到 PVC 申领。

PVC 申领不必一定要请求某个类。如果 PVC 的 `storageClassName` 属性值设置为 `""`， 则被视为要请求的是没有设置存储类的 PV 卷，因此这一 PVC 申领只能绑定到未设置 存储类的 PV 卷（未设置注解或者注解值为 `""` 的 PersistentVolume（PV）对象在系统中不会被删除，因为这样做可能会引起数据丢失。 未设置 `storageClassName` 的 PVC 与此大不相同，也会被集群作不同处理。 具体筛查方式取决于 [`DefaultStorageClass` 准入控制器插件](https://kubernetes.io/zh/docs/reference/access-authn-authz/admission-controllers/#defaultstorageclass) 是否被启用。

- 如果准入控制器插件被启用，则管理员可以设置一个默认的 StorageClass。 所有未设置 `storageClassName` 的 PVC 都只能绑定到隶属于默认存储类的 PV 卷。 设置默认 StorageClass 的工作是通过将对应 StorageClass 对象的注解 `storageclass.kubernetes.io/is-default-class` 赋值为 `true` 来完成的。 如果管理员未设置默认存储类，集群对 PVC 创建的处理方式与未启用准入控制器插件 时相同。如果设定的默认存储类不止一个，准入控制插件会禁止所有创建 PVC 操作。
- 如果准入控制器插件被关闭，则不存在默认 StorageClass 的说法。 所有未设置 `storageClassName` 的 PVC 都只能绑定到未设置存储类的 PV 卷。 在这种情况下，未设置 `storageClassName` 的 PVC 与 `storageClassName` 设置未 `""` 的 PVC 的处理方式相同。

取决于安装方法，默认的 StorageClass 可能在集群安装期间由插件管理器（Addon Manager）部署到集群中。

当某 PVC 除了请求 StorageClass 之外还设置了 `selector`，则这两种需求会按 逻辑与关系处理：只有隶属于所请求类且带有所请求标签的 PV 才能绑定到 PVC。

**说明：** 目前，设置了非空 `selector` 的 PVC 对象无法让集群为其动态供应 PV 卷。

早前，Kubernetes 使用注解 `volume.beta.kubernetes.io/storage-class` 而不是 `storageClassName` 属性。这一注解目前仍然起作用，不过在将来的 Kubernetes 发布版本中该注解会被彻底废弃。

## 使用申领作为卷 

Pod 将申领作为卷来使用，并藉此访问存储资源。 申领必须位于使用它的 Pod 所在的同一名字空间内。 集群在 Pod 的名字空间中查找申领，并使用它来获得申领所使用的 PV 卷。 之后，卷会被挂载到宿主上并挂载到 Pod 中。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```

### 关于名字空间的说明 

PersistentVolume 卷的绑定是排他性的。 由于 PersistentVolumeClaim 是名字空间作用域的对象，使用 "Many" 模式（`ROX`、`RWX`）来挂载申领的操作只能在同一名字空间内进行。

### 类型为 `hostpath` 的 PersistentVolume 

`hostPath` PersistentVolume 使用节点上的文件或目录来模拟网络附加（network-attached）存储。 相关细节可参阅[`hostPath` 卷示例](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-persistent-volume-storage/#create-a-persistentvolume)。

## 原始块卷支持 

**FEATURE STATE:** `Kubernetes v1.18 [stable]`

以下卷插件支持原始块卷，包括其动态供应（如果支持的话）的卷：

- AWSElasticBlockStore
- AzureDisk
- CSI
- FC （光纤通道）
- GCEPersistentDisk
- iSCSI
- Local 卷
- OpenStack Cinder
- RBD （Ceph 块设备）
- VsphereVolume

### 使用原始块卷的持久卷 

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: block-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  persistentVolumeReclaimPolicy: Retain
  fc:
    targetWWNs: ["50060e801049cfd1"]
    lun: 0
    readOnly: false
```

### 申请原始块卷的 PVC 申领 

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 10Gi
```

## 卷填充器（Populator）与数据源 

**FEATURE STATE:** `Kubernetes v1.22 [alpha]`

**说明：**

Kubernetes 支持自定义的卷填充器；Kubernetes 1.18 版本引入了这个 alpha 特性。 Kubernetes 1.22 使用重新设计的 API 重新实现了该机制。 确认你正在阅读与你的集群版本一致的 Kubernetes 文档。 要获知版本信息，请输入 `kubectl version`.

要使用自定义的卷填充器，你必须为 kube-apiserver 和 kube-controller-manager 启用 `AnyVolumeDataSource` [特性门控](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/feature-gates/)。

卷填充器利用了 PVC 规约字段 `dataSourceRef`。 不像 `dataSource` 字段只能包含对另一个持久卷申领或卷快照的引用， `dataSourceRef` 字段可以包含对同一命名空间中任何对象的引用（不包含除 PVC 以外的核心资源）。 对于启用了特性门控的集群，使用 `dataSourceRef` 比 `dataSource` 更好。

## 数据源引用 

`dataSourceRef` 字段的行为与 `dataSource` 字段几乎相同。 如果其中一个字段被指定而另一个字段没有被指定，API 服务器将给两个字段相同的值。 这两个字段都不能在创建后改变，如果试图为这两个字段指定不同的值，将导致验证错误。 因此，这两个字段将总是有相同的内容。

在 `dataSourceRef` 字段和 `dataSource` 字段之间有两个用户应该注意的区别：

- `dataSource` 字段会忽略无效的值（如同是空值）， 而 `dataSourceRef` 字段永远不会忽略值，并且若填入一个无效的值，会导致错误。 无效值指的是 PVC 之外的核心对象（没有 apiGroup 的对象）。
- `dataSourceRef` 字段可以包含不同类型的对象，而 `dataSource` 字段只允许 PVC 和卷快照。

用户应该始终在启用了特性门控的集群上使用 `dataSourceRef`，而在没有启用特性门控的集群上使用 `dataSource`。 在任何情况下都没有必要查看这两个字段。 这两个字段的值看似相同但是语义稍微不一样，是为了向后兼容。 特别是混用旧版本和新版本的控制器时，它们能够互通。

## 使用卷填充器 

卷填充器是能创建非空卷的[控制器](https://kubernetes.io/zh/docs/concepts/architecture/controller/)， 其卷的内容通过一个自定义资源决定。 用户通过使用 `dataSourceRef` 字段引用自定义资源来创建一个被填充的卷：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: populated-pvc
spec:
  dataSourceRef:
    name: example-name
    kind: ExampleDataSource
    apiGroup: example.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

因为卷填充器是外部组件，如果没有安装所有正确的组件，试图创建一个使用卷填充器的 PVC 就会失败。 外部控制器应该在 PVC 上产生事件，以提供创建状态的反馈，包括在由于缺少某些组件而无法创建 PVC 的情况下发出警告。

你可以把 alpha 版本的[卷数据源验证器](https://github.com/kubernetes-csi/volume-data-source-validator) 控制器安装到你的集群中。 如果没有填充器处理该数据源的情况下，该控制器会在 PVC 上产生警告事件。 当一个合适的填充器被安装到 PVC 上时，该控制器的职责是上报与卷创建有关的事件，以及在该过程中发生的问题。

### 在容器中添加原始块设备路径的 Pod 规约

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-block-volume
spec:
  containers:
    - name: fc-container
      image: fedora:26
      command: ["/bin/sh", "-c"]
      args: [ "tail -f /dev/null" ]
      volumeDevices:
        - name: data
          devicePath: /dev/xvda
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: block-pvc
```

**说明：** 向 Pod 中添加原始块设备时，你要在容器内设置设备路径而不是挂载路径。

### 绑定块卷 

如果用户通过 PersistentVolumeClaim 规约的 `volumeMode` 字段来表明对原始 块设备的请求，绑定规则与之前版本中未在规约中考虑此模式的实现略有不同。 下面列举的表格是用户和管理员可以为请求原始块设备所作设置的组合。 此表格表明在不同的组合下卷是否会被绑定。

静态供应卷的卷绑定矩阵：

| PV volumeMode | PVC volumeMode | Result |
| ------------- | :------------: | -----: |
| 未指定        |     未指定     |   绑定 |
| 未指定        |     Block      | 不绑定 |
| 未指定        |   Filesystem   |   绑定 |
| Block         |     未指定     | 不绑定 |
| Block         |     Block      |   绑定 |
| Block         |   Filesystem   | 不绑定 |
| Filesystem    |   Filesystem   |   绑定 |
| Filesystem    |     Block      | 不绑定 |
| Filesystem    |     未指定     |   绑定 |

**说明：** Alpha 发行版本中仅支持静态供应的卷。 管理员需要在处理原始块设备时小心处理这些值。

## 对卷快照及从卷快照中恢复卷的支持

**FEATURE STATE:** `Kubernetes v1.17 [beta]`

卷快照（Volume Snapshot）功能的添加仅是为了支持 CSI 卷插件。 有关细节可参阅[卷快照](https://kubernetes.io/zh/docs/concepts/storage/volume-snapshots/)文档。

要启用从卷快照数据源恢复数据卷的支持，可在 API 服务器和控制器管理器上启用 `VolumeSnapshotDataSource` 特性门控。

### 基于卷快照创建 PVC 申领 

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restore-pvc
spec:
  storageClassName: csi-hostpath-sc
  dataSource:
    name: new-snapshot-test
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## 卷克隆 

[卷克隆](https://kubernetes.io/zh/docs/concepts/storage/volume-pvc-datasource/)功能特性仅适用于 CSI 卷插件。

### 基于现有 PVC 创建新的 PVC 申领 

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cloned-pvc
spec:
  storageClassName: my-csi-plugin
  dataSource:
    name: existing-src-pvc-name
    kind: PersistentVolumeClaim
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## 编写可移植的配置 

如果你要编写配置模板和示例用来在很多集群上运行并且需要持久性存储，建议你使用以下模式：

- 将 PersistentVolumeClaim 对象包含到你的配置包（Bundle）中，和 Deployment 以及 ConfigMap 等放在一起。
- 不要在配置中包含 PersistentVolume 对象，因为对配置进行实例化的用户很可能 没有创建 PersistentVolume 的权限。

- 为用户提供在实例化模板时指定存储类名称的能力。
  - 仍按用户提供存储类名称，将该名称放到 `persistentVolumeClaim.storageClassName` 字段中。 这样会使得 PVC 在集群被管理员启用了存储类支持时能够匹配到正确的存储类，
  - 如果用户未指定存储类名称，将 `persistentVolumeClaim.storageClassName` 留空（nil）。 这样，集群会使用默认 `StorageClass` 为用户自动供应一个存储卷。 很多集群环境都配置了默认的 `StorageClass`，或者管理员也可以自行创建默认的 `StorageClass`。

- 在你的工具链中，监测经过一段时间后仍未被绑定的 PVC 对象，要让用户知道这些对象， 因为这可能意味着集群不支持动态存储（因而用户必须先创建一个匹配的 PV），或者 集群没有配置存储系统（因而用户无法配置需要 PVC 的工作负载配置）。

# 存储类

本文描述了 Kubernetes 中 StorageClass 的概念。建议先熟悉 [卷](https://kubernetes.io/zh/docs/concepts/storage/volumes/)和 [持久卷](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes)的概念。

## 介绍

StorageClass 为管理员提供了描述存储 "类" 的方法。 不同的类型可能会映射到不同的服务质量等级或备份策略，或是由集群管理员制定的任意策略。 Kubernetes 本身并不清楚各种类代表的什么。这个类的概念在其他存储系统中有时被称为 "配置文件"。

## StorageClass 资源

每个 StorageClass 都包含 `provisioner`、`parameters` 和 `reclaimPolicy` 字段， 这些字段会在 StorageClass 需要动态分配 PersistentVolume 时会使用到。

StorageClass 对象的命名很重要，用户使用这个命名来请求生成一个特定的类。 当创建 StorageClass 对象时，管理员设置 StorageClass 对象的命名和其他参数，一旦创建了对象就不能再对其更新。

管理员可以为没有申请绑定到特定 StorageClass 的 PVC 指定一个默认的存储类： 更多详情请参阅 [PersistentVolumeClaim 章节](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
  - debug
volumeBindingMode: Immediate
```

### 存储制备器 

每个 StorageClass 都有一个制备器（Provisioner），用来决定使用哪个卷插件制备 PV。 该字段必须指定。

| 卷插件               | 内置制备器 |                           配置例子                           |
| :------------------- | :--------: | :----------------------------------------------------------: |
| AWSElasticBlockStore |     ✓      | [AWS EBS](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#aws-ebs) |
| AzureFile            |     ✓      | [Azure File](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#azure-文件) |
| AzureDisk            |     ✓      | [Azure Disk](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#azure-磁盘) |
| CephFS               |     -      |                              -                               |
| Cinder               |     ✓      | [OpenStack Cinder](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#openstack-cinder) |
| FC                   |     -      |                              -                               |
| FlexVolume           |     -      |                              -                               |
| Flocker              |     ✓      |                              -                               |
| GCEPersistentDisk    |     ✓      | [GCE PD](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#gce-pd) |
| Glusterfs            |     ✓      | [Glusterfs](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#glusterfs) |
| iSCSI                |     -      |                              -                               |
| Quobyte              |     ✓      | [Quobyte](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#quobyte) |
| NFS                  |     -      | [NFS](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#nfs) |
| RBD                  |     ✓      | [Ceph RBD](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#ceph-rbd) |
| VsphereVolume        |     ✓      | [vSphere](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#vsphere) |
| PortworxVolume       |     ✓      | [Portworx Volume](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#portworx-卷) |
| ScaleIO              |     ✓      | [ScaleIO](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#scaleio) |
| StorageOS            |     ✓      | [StorageOS](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#storageos) |
| Local                |     -      | [Local](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#本地) |

你不限于指定此处列出的 "内置" 制备器（其名称前缀为 "kubernetes.io" 并打包在 Kubernetes 中）。 你还可以运行和指定外部制备器，这些独立的程序遵循由 Kubernetes 定义的 [规范](https://git.k8s.io/community/contributors/design-proposals/storage/volume-provisioning.md)。 外部供应商的作者完全可以自由决定他们的代码保存于何处、打包方式、运行方式、使用的插件（包括 Flex）等。 代码仓库 [kubernetes-sigs/sig-storage-lib-external-provisioner](https://github.com/kubernetes-sigs/sig-storage-lib-external-provisioner) 包含一个用于为外部制备器编写功能实现的类库。你可以访问代码仓库 [kubernetes-sigs/sig-storage-lib-external-provisioner](https://github.com/kubernetes-sigs/sig-storage-lib-external-provisioner) 了解外部驱动列表。

例如，NFS 没有内部制备器，但可以使用外部制备器。 也有第三方存储供应商提供自己的外部制备器。

### 回收策略

由 StorageClass 动态创建的 PersistentVolume 会在类的 `reclaimPolicy` 字段中指定回收策略，可以是 `Delete` 或者 `Retain`。如果 StorageClass 对象被创建时没有指定 `reclaimPolicy`，它将默认为 `Delete`。

通过 StorageClass 手动创建并管理的 PersistentVolume 会使用它们被创建时指定的回收政策。

### 允许卷扩展

**FEATURE STATE:** `Kubernetes v1.11 [beta]`

PersistentVolume 可以配置为可扩展。将此功能设置为 `true` 时，允许用户通过编辑相应的 PVC 对象来调整卷大小。

当下层 StorageClass 的 `allowVolumeExpansion` 字段设置为 true 时，以下类型的卷支持卷扩展。

| 卷类型               | Kubernetes 版本要求       |
| :------------------- | :------------------------ |
| gcePersistentDisk    | 1.11                      |
| awsElasticBlockStore | 1.11                      |
| Cinder               | 1.11                      |
| glusterfs            | 1.11                      |
| rbd                  | 1.11                      |
| Azure File           | 1.11                      |
| Azure Disk           | 1.11                      |
| Portworx             | 1.11                      |
| FlexVolume           | 1.13                      |
| CSI                  | 1.14 (alpha), 1.16 (beta) |

**说明：** 此功能仅可用于扩容卷，不能用于缩小卷。

### 挂载选项

由 StorageClass 动态创建的 PersistentVolume 将使用类中 `mountOptions` 字段指定的挂载选项。

如果卷插件不支持挂载选项，却指定了挂载选项，则制备操作会失败。 挂载选项在 StorageClass 和 PV 上都不会做验证，如果其中一个挂载选项无效，那么这个 PV 挂载操作就会失败。

### 卷绑定模式

`volumeBindingMode` 字段控制了[卷绑定和动态制备](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#provisioning) 应该发生在什么时候。

默认情况下，`Immediate` 模式表示一旦创建了 PersistentVolumeClaim 也就完成了卷绑定和动态制备。 对于由于拓扑限制而非集群所有节点可达的存储后端，PersistentVolume 会在不知道 Pod 调度要求的情况下绑定或者制备。

集群管理员可以通过指定 `WaitForFirstConsumer` 模式来解决此问题。 该模式将延迟 PersistentVolume 的绑定和制备，直到使用该 PersistentVolumeClaim 的 Pod 被创建。 PersistentVolume 会根据 Pod 调度约束指定的拓扑来选择或制备。这些包括但不限于 [资源需求](https://kubernetes.io/zh/docs/concepts/configuration/manage-resources-containers/)、 [节点筛选器](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector)、 [pod 亲和性和互斥性](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity/)、 以及[污点和容忍度](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/taint-and-toleration)。

以下插件支持动态供应的 `WaitForFirstConsumer` 模式:

- [AWSElasticBlockStore](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#aws-ebs)
- [GCEPersistentDisk](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#gce-pd)
- [AzureDisk](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#azure-disk)

以下插件支持预创建绑定 PersistentVolume 的 `WaitForFirstConsumer` 模式：

- 上述全部
- [Local](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#local)

**FEATURE STATE:** `Kubernetes v1.17 [stable]`

动态配置和预先创建的 PV 也支持 [CSI卷](https://kubernetes.io/zh/docs/concepts/storage/volumes/#csi)， 但是你需要查看特定 CSI 驱动程序的文档以查看其支持的拓扑键名和例子。

**说明：**

如果你选择使用 `WaitForFirstConsumer`，请不要在 Pod 规约中使用 `nodeName` 来指定节点亲和性。 如果在这种情况下使用 `nodeName`，Pod 将会绕过调度程序，PVC 将停留在 `pending` 状态。

相反，在这种情况下，你可以使用节点选择器作为主机名，如下所示

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  nodeSelector:
    kubernetes.io/hostname: kube-01
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: task-pv-claim
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```

### 允许的拓扑结构 

**FEATURE STATE:** `Kubernetes v1.12 [beta]`

当集群操作人员使用了 `WaitForFirstConsumer` 的卷绑定模式， 在大部分情况下就没有必要将制备限制为特定的拓扑结构。 然而，如果还有需要的话，可以使用 `allowedTopologies`。

这个例子描述了如何将供应卷的拓扑限制在特定的区域，在使用时应该根据插件 支持情况替换 `zone` 和 `zones` 参数。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
- matchLabelExpressions:
  - key: failure-domain.beta.kubernetes.io/zone
    values:
    - us-central1-a
    - us-central1-b
```

## 参数

Storage Classes 的参数描述了存储类的卷。取决于制备器，可以接受不同的参数。 例如，参数 type 的值 io1 和参数 iopsPerGB 特定于 EBS PV。 当参数被省略时，会使用默认值。

一个 StorageClass 最多可以定义 512 个参数。这些参数对象的总长度不能 超过 256 KiB, 包括参数的键和值。

### AWS EBS

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1
  iopsPerGB: "10"
  fsType: ext4
```

- `type`：`io1`，`gp2`，`sc1`，`st1`。详细信息参见 [AWS 文档](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSVolumeTypes.html)。默认值：`gp2`。
- `zone`(弃用)：AWS 区域。如果没有指定 `zone` 和 `zones`， 通常卷会在 Kubernetes 集群节点所在的活动区域中轮询调度分配。 `zone` 和 `zones` 参数不能同时使用。
- `zones`(弃用)：以逗号分隔的 AWS 区域列表。 如果没有指定 `zone` 和 `zones`，通常卷会在 Kubernetes 集群节点所在的 活动区域中轮询调度分配。`zone`和`zones`参数不能同时使用。
- `iopsPerGB`：只适用于 `io1` 卷。每 GiB 每秒 I/O 操作。 AWS 卷插件将其与请求卷的大小相乘以计算 IOPS 的容量， 并将其限制在 20000 IOPS（AWS 支持的最高值，请参阅 [AWS 文档](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSVolumeTypes.html)。 这里需要输入一个字符串，即 `"10"`，而不是 `10`。
- `fsType`：受 Kubernetes 支持的文件类型。默认值：`"ext4"`。
- `encrypted`：指定 EBS 卷是否应该被加密。合法值为 `"true"` 或者 `"false"`。 这里需要输入字符串，即 `"true"`, 而非 `true`。
- `kmsKeyId`：可选。加密卷时使用密钥的完整 Amazon 资源名称。 如果没有提供，但 `encrypted` 值为 true，AWS 生成一个密钥。关于有效的 ARN 值，请参阅 AWS 文档。

**说明：**

`zone` 和 `zones` 已被弃用并被 [允许的拓扑结构](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#allowed-topologies) 取代。

### GCE PD

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
   fstype: ext4
  replication-type: none
```

- `type`：`pd-standard` 或者 `pd-ssd`。默认：`pd-standard`
- `zone`(弃用)：GCE 区域。如果没有指定 `zone` 和 `zones`，通常 卷会在 Kubernetes 集群节点所在的活动区域中轮询调度分配。 `zone` 和 `zones` 参数不能同时使用。
- `zones`(弃用)：逗号分隔的 GCE 区域列表。如果没有指定 `zone` 和 `zones`， 通常卷会在 Kubernetes 集群节点所在的活动区域中轮询调度（round-robin）分配。 `zone` 和 `zones` 参数不能同时使用。
- `fstype`: `ext4` 或 `xfs`。 默认: `ext4`。宿主机操作系统必须支持所定义的文件系统类型。
- `replication-type`：`none` 或者 `regional-pd`。默认值：`none`。

如果 `replication-type` 设置为 `none`，会制备一个常规（当前区域内的）持久化磁盘。

如果 `replication-type` 设置为 `regional-pd`，会制备一个 [区域性持久化磁盘（Regional Persistent Disk）](https://cloud.google.com/compute/docs/disks/#repds)。

强烈建议设置 `volumeBindingMode: WaitForFirstConsumer`，这样设置后， 当你创建一个 Pod，它使用的 PersistentVolumeClaim 使用了这个 StorageClass， 区域性持久化磁盘会在两个区域里制备。 其中一个区域是 Pod 所在区域。 另一个区域是会在集群管理的区域中任意选择。磁盘区域可以通过 `allowedTopologies` 加以限制。

**说明：** `zone` 和 `zones` 已被弃用并被 [allowedTopologies](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#allowed-topologies) 取代。

### Glusterfs

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://127.0.0.1:8081"
  clusterid: "630372ccdc720a92c681fb928f27b53f"
  restauthenabled: "true"
  restuser: "admin"
  secretNamespace: "default"
  secretName: "heketi-secret"
  gidMin: "40000"
  gidMax: "50000"
  volumetype: "replicate:3"
```

- `resturl`：制备 gluster 卷的需求的 Gluster REST 服务/Heketi 服务 url。 通用格式应该是 `IPaddress:Port`，这是 GlusterFS 动态制备器的必需参数。 如果 Heketi 服务在 OpenShift/kubernetes 中安装并暴露为可路由服务，则可以使用类似于 `http://heketi-storage-project.cloudapps.mystorage.com` 的格式，其中 fqdn 是可解析的 heketi 服务网址。
- `restauthenabled`：Gluster REST 服务身份验证布尔值，用于启用对 REST 服务器的身份验证。 如果此值为 'true'，则必须填写 `restuser` 和 `restuserkey` 或 `secretNamespace` + `secretName`。 此选项已弃用，当在指定 `restuser`、`restuserkey`、`secretName` 或 `secretNamespace` 时，身份验证被启用。
- `restuser`：在 Gluster 可信池中有权创建卷的 Gluster REST服务/Heketi 用户。
- `restuserkey`：Gluster REST 服务/Heketi 用户的密码将被用于对 REST 服务器进行身份验证。 此参数已弃用，取而代之的是 `secretNamespace` + `secretName`。

- `secretNamespace`，`secretName`：Secret 实例的标识，包含与 Gluster REST 服务交互时使用的用户密码。 这些参数是可选的，`secretNamespace` 和 `secretName` 都省略时使用空密码。 所提供的 Secret 必须将类型设置为 "kubernetes.io/glusterfs"，例如以这种方式创建：

  ```
  kubectl create secret generic heketi-secret \
    --type="kubernetes.io/glusterfs" --from-literal=key='opensesame' \
    --namespace=default
  ```

  Secret 的例子可以在 [glusterfs-provisioning-secret.yaml](https://github.com/kubernetes/examples/tree/master/staging/persistent-volume-provisioning/glusterfs/glusterfs-secret.yaml) 中找到。

- `clusterid`：`630372ccdc720a92c681fb928f27b53f` 是集群的 ID，当制备卷时， Heketi 将会使用这个文件。它也可以是一个 clusterid 列表，例如： `"8452344e2becec931ece4e33c4674e4e,42982310de6c63381718ccfa6d8cf397"`。这个是可选参数。
- `gidMin`，`gidMax`：StorageClass GID 范围的最小值和最大值。 在此范围（gidMin-gidMax）内的唯一值（GID）将用于动态制备卷。这些是可选的值。 如果不指定，所制备的卷为一个 2000-2147483647 之间的值，这是 gidMin 和 gidMax 的默认值。

- `volumetype`：卷的类型及其参数可以用这个可选值进行配置。如果未声明卷类型，则 由制备器决定卷的类型。 例如：

  - 'Replica volume': `volumetype: replicate:3` 其中 '3' 是 replica 数量.
  - 'Disperse/EC volume': `volumetype: disperse:4:2` 其中 '4' 是数据，'2' 是冗余数量.
  - 'Distribute volume': `volumetype: none`

  有关可用的卷类型和管理选项，请参阅 [管理指南](https://access.redhat.com/documentation/en-US/Red_Hat_Storage/3.1/html/Administration_Guide/part-Overview.html)。

  更多相关的参考信息，请参阅 [如何配置 Heketi](https://github.com/heketi/heketi/wiki/Setting-up-the-topology)。

  当动态制备持久卷时，Gluster 插件自动创建名为 `gluster-dynamic-<claimname>` 的端点和无头服务。在 PVC 被删除时动态端点和无头服务会自动被删除。

### NFS

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: example-nfs
provisioner: example.com/external-nfs
parameters:
  server: nfs-server.example.com
  path: /share
  readOnly: false
```

- `server`：NFS 服务器的主机名或 IP 地址。
- `path`：NFS 服务器导出的路径。
- `readOnly`：是否将存储挂载为只读的标志（默认为 false）。

Kubernetes 不包含内部 NFS 驱动。你需要使用外部驱动为 NFS 创建 StorageClass。 这里有些例子：

- [NFS Ganesha 服务器和外部驱动](https://github.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner)
- [NFS subdir 外部驱动](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)

### OpenStack Cinder

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gold
provisioner: kubernetes.io/cinder
parameters:
  availability: nova
```

- `availability`：可用区域。如果没有指定，通常卷会在 Kubernetes 集群节点 所在的活动区域中轮转调度。

**说明：**

**FEATURE STATE:** `Kubernetes 1.11 [deprecated]`

OpenStack 的内部驱动已经被弃用。请使用 [OpenStack 的外部云驱动](https://github.com/kubernetes/cloud-provider-openstack)。

### vSphere

vSphere 存储类有两种制备器

- [CSI 制备器](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#vsphere-provisioner-csi): `csi.vsphere.vmware.com`
- [vCP 制备器](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#vcp-provisioner): `kubernetes.io/vsphere-volume`

树内制备器已经被 [弃用](https://kubernetes.io/blog/2019/12/09/kubernetes-1-17-feature-csi-migration-beta/#why-are-we-migrating-in-tree-plugins-to-csi)。 更多关于 CSI 制备器的详情，请参阅 [Kubernetes vSphere CSI 驱动](https://vsphere-csi-driver.sigs.k8s.io/) 和 [vSphereVolume CSI 迁移](https://kubernetes.io/zh/docs/concepts/storage/volumes/#csi-migration-5)。

#### CSI 制备器

vSphere CSI StorageClass 制备器在 Tanzu Kubernetes 集群下运行。示例请参 [vSphere CSI 仓库](https://github.com/kubernetes-sigs/vsphere-csi-driver/blob/master/example/vanilla-k8s-RWM-filesystem-volumes/example-sc.yaml)。

#### vCP 制备器

以下示例使用 VMware Cloud Provider (vCP) StorageClass 调度器

1. 使用用户指定的磁盘格式创建一个 StorageClass。

   ```yaml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: fast
   provisioner: kubernetes.io/vsphere-volume
   parameters:
     diskformat: zeroedthick
   ```

   `diskformat`: `thin`, `zeroedthick` 和 `eagerzeroedthick`。默认值: `"thin"`。

1. 在用户指定的数据存储上创建磁盘格式的 StorageClass。

   ```yaml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: fast
   provisioner: kubernetes.io/vsphere-volume
   parameters:
       diskformat: zeroedthick
       datastore: VSANDatastore
   ```

   `datastore`：用户也可以在 StorageClass 中指定数据存储。 卷将在 storage class 中指定的数据存储上创建，在这种情况下是 `VSANDatastore`。 该字段是可选的。 如果未指定数据存储，则将在用于初始化 vSphere Cloud Provider 的 vSphere 配置文件中指定的数据存储上创建该卷。

1. Kubernetes 中的存储策略管理

   - 使用现有的 vCenter SPBM 策略

     vSphere 用于存储管理的最重要特性之一是基于策略的管理。 基于存储策略的管理（SPBM）是一个存储策略框架，提供单一的统一控制平面的 跨越广泛的数据服务和存储解决方案。 SPBM 使能 vSphere 管理员克服先期的存储配置挑战，如容量规划，差异化服务等级和管理容量空间。

     SPBM 策略可以在 StorageClass 中使用 `storagePolicyName` 参数声明。

   - Kubernetes 内的 Virtual SAN 策略支持

     Vsphere Infrastructure（VI）管理员将能够在动态卷配置期间指定自定义 Virtual SAN 存储功能。你现在可以在动态制备卷期间以存储能力的形式定义存储需求，例如性能和可用性。 存储能力需求会转换为 Virtual SAN 策略，之后当持久卷（虚拟磁盘）被创建时， 会将其推送到 Virtual SAN 层。虚拟磁盘分布在 Virtual SAN 数据存储中以满足要求。

     你可以参考[基于存储策略的动态制备卷管理](https://vmware.github.io/vsphere-storage-for-kubernetes/documentation/policy-based-mgmt.html)， 进一步了解有关持久卷管理的存储策略的详细信息。

有几个 [vSphere 例子](https://github.com/kubernetes/examples/tree/master/staging/volumes/vsphere) 供你在 Kubernetes for vSphere 中尝试进行持久卷管理。

### Ceph RBD

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/rbd
parameters:
  monitors: 10.16.153.105:6789
  adminId: kube
  adminSecretName: ceph-secret
  adminSecretNamespace: kube-system
  pool: kube
  userId: kube
  userSecretName: ceph-secret-user
  userSecretNamespace: default
  fsType: ext4
  imageFormat: "2"
  imageFeatures: "layering"
```

- `monitors`：Ceph monitor，逗号分隔。该参数是必需的。
- `adminId`：Ceph 客户端 ID，用于在池 ceph 池中创建映像。默认是 "admin"。
- `adminSecret`：`adminId` 的 Secret 名称。该参数是必需的。 提供的 secret 必须有值为 "kubernetes.io/rbd" 的 type 参数。
- `adminSecretNamespace`：`adminSecret` 的命名空间。默认是 "default"。
- `pool`: Ceph RBD 池. 默认是 "rbd"。
- `userId`：Ceph 客户端 ID，用于映射 RBD 镜像。默认与 `adminId` 相同。

- `userSecretName`：用于映射 RBD 镜像的 `userId` 的 Ceph Secret 的名字。 它必须与 PVC 存在于相同的 namespace 中。该参数是必需的。 提供的 secret 必须具有值为 "kubernetes.io/rbd" 的 type 参数，例如以这样的方式创建：

  ```shell
  kubectl create secret generic ceph-secret --type="kubernetes.io/rbd" \
    --from-literal=key='QVFEQ1pMdFhPUnQrSmhBQUFYaERWNHJsZ3BsMmNjcDR6RFZST0E9PQ==' \
    --namespace=kube-system
  ```

- `userSecretNamespace`：`userSecretName` 的命名空间。
- `fsType`：Kubernetes 支持的 fsType。默认：`"ext4"`。
- `imageFormat`：Ceph RBD 镜像格式，"1" 或者 "2"。默认值是 "1"。
- `imageFeatures`：这个参数是可选的，只能在你将 `imageFormat` 设置为 "2" 才使用。 目前支持的功能只是 `layering`。默认是 ""，没有功能打开。

### Quobyte

**FEATURE STATE:** `Kubernetes v1.22 [deprecated]`

Quobyte 树内（in-tree）存储插件已弃用， 你可以在 Quobyte CSI 仓库中找到用于树外（out-of-tree）Quobyte 插件的 `StorageClass` [示例](https://github.com/quobyte/quobyte-csi/blob/master/example/StorageClass.yaml)。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: slow
provisioner: kubernetes.io/quobyte
parameters:
    quobyteAPIServer: "http://138.68.74.142:7860"
    registry: "138.68.74.142:7861"
    adminSecretName: "quobyte-admin-secret"
    adminSecretNamespace: "kube-system"
    user: "root"
    group: "root"
    quobyteConfig: "BASE"
    quobyteTenant: "DEFAULT"
```

- `quobyteAPIServer`：Quobyte API 服务器的格式是 `"http(s)://api-server:7860"`
- `registry`：用于挂载卷的 Quobyte 仓库。你可以指定仓库为 `<host>:<port>` 或者如果你想指定多个 registry，在它们之间添加逗号，例如 `<host1>:<port>,<host2>:<port>,<host3>:<port>`。 主机可以是一个 IP 地址，或者如果你有正在运行的 DNS，你也可以提供 DNS 名称。
- `adminSecretNamespace`：`adminSecretName` 的名字空间。 默认值是 "default"。

- `adminSecretName`：保存关于 Quobyte 用户和密码的 Secret，用于对 API 服务器进行身份验证。 提供的 secret 必须有值为 "kubernetes.io/quobyte" 的 type 参数和 `user` 与 `password` 的键值， 例如以这种方式创建：

  ```shell
  kubectl create secret generic quobyte-admin-secret \
    --type="kubernetes.io/quobyte" --from-literal=key='opensesame' \
    --namespace=kube-system
  ```

- `user`：对这个用户映射的所有访问权限。默认是 "root"。
- `group`：对这个组映射的所有访问权限。默认是 "nfsnobody"。
- `quobyteConfig`：使用指定的配置来创建卷。你可以创建一个新的配置， 或者，可以修改 Web 控制台或 quobyte CLI 中现有的配置。默认是 "BASE"。
- `quobyteTenant`：使用指定的租户 ID 创建/删除卷。这个 Quobyte 租户必须 已经于 Quobyte 中存在。默认是 "DEFAULT"。

### Azure 磁盘

#### Azure Unmanaged Disk Storage Class（非托管磁盘存储类）

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: slow
provisioner: kubernetes.io/azure-disk
parameters:
  skuName: Standard_LRS
  location: eastus
  storageAccount: azure_storage_account_name
```

- `skuName`：Azure 存储帐户 Sku 层。默认为空。
- `location`：Azure 存储帐户位置。默认为空。
- `storageAccount`：Azure 存储帐户名称。 如果提供存储帐户，它必须位于与集群相同的资源组中，并且 `location` 是被忽略的。如果未提供存储帐户，则会在与群集相同的资源组中创建新的存储帐户。

#### Azure 磁盘 Storage Class（从 v1.7.2 开始）

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: slow
provisioner: kubernetes.io/azure-disk
parameters:
  storageaccounttype: Standard_LRS
  kind: managed
```

- `storageaccounttype`：Azure 存储帐户 Sku 层。默认为空。
- `kind`：可能的值是 `shared`、`dedicated` 和 `managed`（默认）。 当 `kind` 的值是 `shared` 时，所有非托管磁盘都在集群的同一个资源组中的几个共享存储帐户中创建。 当 `kind` 的值是 `dedicated` 时，将为在集群的同一个资源组中新的非托管磁盘创建新的专用存储帐户。
- `resourceGroup`: 指定要创建 Azure 磁盘所属的资源组。必须是已存在的资源组名称。 若未指定资源组，磁盘会默认放入与当前 Kubernetes 集群相同的资源组中。

- Premium VM 可以同时添加 Standard_LRS 和 Premium_LRS 磁盘，而 Standard 虚拟机只能添加 Standard_LRS 磁盘。
- 托管虚拟机只能连接托管磁盘，非托管虚拟机只能连接非托管磁盘。

### Azure 文件

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: azurefile
provisioner: kubernetes.io/azure-file
parameters:
  skuName: Standard_LRS
  location: eastus
  storageAccount: azure_storage_account_name
```

- `skuName`：Azure 存储帐户 Sku 层。默认为空。
- `location`：Azure 存储帐户位置。默认为空。
- `storageAccount`：Azure 存储帐户名称。默认为空。 如果不提供存储帐户，会搜索所有与资源相关的存储帐户，以找到一个匹配 `skuName` 和 `location` 的账号。 如果提供存储帐户，它必须存在于与集群相同的资源组中，`skuName` 和 `location` 会被忽略。
- `secretNamespace`：包含 Azure 存储帐户名称和密钥的密钥的名称空间。 默认值与 Pod 相同。
- `secretName`：包含 Azure 存储帐户名称和密钥的密钥的名称。 默认值为 `azure-storage-account-<accountName>-secret`
- `readOnly`：指示是否将存储安装为只读的标志。默认为 false，表示"读/写"挂载。 该设置也会影响VolumeMounts中的 `ReadOnly` 设置。

在存储制备期间，为挂载凭证创建一个名为 `secretName` 的 Secret。如果集群同时启用了 [RBAC](https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/) 和 [控制器角色](https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/#controller-roles)， 为 `system:controller:persistent-volume-binder` 的 clusterrole 添加 `Secret` 资源的 `create` 权限。

在多租户上下文中，强烈建议显式设置 `secretNamespace` 的值，否则 其他用户可能会读取存储帐户凭据。

### Portworx 卷

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: portworx-io-priority-high
provisioner: kubernetes.io/portworx-volume
parameters:
  repl: "1"
  snap_interval:   "70"
  priority_io:  "high"
```

- `fs`：选择的文件系统：`none/xfs/ext4`（默认：`ext4`）。
- `block_size`：以 Kbytes 为单位的块大小（默认值：`32`）。
- `repl`：同步副本数量，以复制因子 `1..3`（默认值：`1`）的形式提供。 这里需要填写字符串，即，`"1"` 而不是 `1`。
- `io_priority`：决定是否从更高性能或者较低优先级存储创建卷 `high/medium/low`（默认值：`low`）。
- `snap_interval`：触发快照的时钟/时间间隔（分钟）。 快照是基于与先前快照的增量变化，0 是禁用快照（默认：`0`）。 这里需要填写字符串，即，是 `"70"` 而不是 `70`。
- `aggregation_level`：指定卷分配到的块数量，0 表示一个非聚合卷（默认：`0`）。 这里需要填写字符串，即，是 `"0"` 而不是 `0`。
- `ephemeral`：指定卷在卸载后进行清理还是持久化。 `emptyDir` 的使用场景可以将这个值设置为 true ， `persistent volumes` 的使用场景可以将这个值设置为 false （例如 Cassandra 这样的数据库） `true/false`（默认为 `false`）。这里需要填写字符串，即， 是 `"true"` 而不是 `true`。

### ScaleIO

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: slow
provisioner: kubernetes.io/scaleio
parameters:
  gateway: https://192.168.99.200:443/api
  system: scaleio
  protectionDomain: pd0
  storagePool: sp1
  storageMode: ThinProvisioned
  secretRef: sio-secret
  readOnly: false
  fsType: xfs
```

- `provisioner`：属性设置为 `kubernetes.io/scaleio`
- `gateway` 到 ScaleIO API 网关的地址（必需）
- `system`：ScaleIO 系统的名称（必需）
- `protectionDomain`：ScaleIO 保护域的名称（必需）
- `storagePool`：卷存储池的名称（必需）
- `storageMode`：存储提供模式：`ThinProvisioned`（默认）或 `ThickProvisioned`
- `secretRef`：对已配置的 Secret 对象的引用（必需）
- `readOnly`：指定挂载卷的访问模式（默认为 false）
- `fsType`：卷的文件系统（默认是 ext4）

ScaleIO Kubernetes 卷插件需要配置一个 Secret 对象。 Secret 必须用 `kubernetes.io/scaleio` 类型创建，并与引用它的 PVC 所属的名称空间使用相同的值。如下面的命令所示：

```shell
kubectl create secret generic sio-secret --type="kubernetes.io/scaleio" \
  --from-literal=username=sioadmin --from-literal=password=d2NABDNjMA== \
  --namespace=default
```

### StorageOS

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/storageos
parameters:
  pool: default
  description: Kubernetes volume
  fsType: ext4
  adminSecretNamespace: default
  adminSecretName: storageos-secret
```

- `pool`：制备卷的 StorageOS 分布式容量池的名称。如果未指定，则使用 通常存在的 `default` 池。
- `description`：指定给动态创建的卷的描述。所有卷描述对于存储类而言都是相同的， 但不同的 storage class 可以使用不同的描述，以区分不同的使用场景。 默认为 `Kubernetes volume`。
- `fsType`：请求的默认文件系统类型。 请注意，在 StorageOS 中用户定义的规则可以覆盖此值。默认为 `ext4`
- `adminSecretNamespace`：API 配置 secret 所在的命名空间。 如果设置了 adminSecretName，则是必需的。
- `adminSecretName`：用于获取 StorageOS API 凭证的 secret 名称。 如果未指定，则将尝试默认值。

StorageOS Kubernetes 卷插件可以使 Secret 对象来指定用于访问 StorageOS API 的端点和凭据。 只有当默认值已被更改时，这才是必须的。 Secret 必须使用 `kubernetes.io/storageos` 类型创建，如以下命令：

```shell
kubectl create secret generic storageos-secret \
--type="kubernetes.io/storageos" \
--from-literal=apiAddress=tcp://localhost:5705 \
--from-literal=apiUsername=storageos \
--from-literal=apiPassword=storageos \
--namespace=default
```

用于动态制备卷的 Secret 可以在任何名称空间中创建，并通过 `adminSecretNamespace` 参数引用。 预先配置的卷使用的 Secret 必须在与引用它的 PVC 在相同的名称空间中。

### 本地

**FEATURE STATE:** `Kubernetes v1.14 [stable]`

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

本地卷还不支持动态制备，然而还是需要创建 StorageClass 以延迟卷绑定， 直到完成 Pod 的调度。这是由 `WaitForFirstConsumer` 卷绑定模式指定的。

延迟卷绑定使得调度器在为 PersistentVolumeClaim 选择一个合适的 PersistentVolume 时能考虑到所有 Pod 的调度限制。