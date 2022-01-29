# k8s-ops
[k8s集群搭建](docker_k8s.md)

[yaml文件编写](yaml_concept.md)

[pod高级实战](pod_adv.md)

[kubectl管理k8s](./kubectl.md)

[k8s控制器 Replicaset 和 Deployment](./k8s_controllers_Replicaset_Deployment.md)

[存储类](./storages.md)

[ingress](./ingress.md)

[elk](./elk.md)

[pod自动扩展](./pod_autoscale.md)

[devops/CICD](./devops.md)

[微服务部署](./ms.md)

# 常用连接以及问题处理

## 相关镜像

[jenkins镜像](https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json)

[docker源](https://2c2nqqhp.mirror.aliyuncs.com)

[阿里云镜像](https://developer.aliyun.com/mirror/)

## k8s调优的方向

1. 网络层面  网络插件
2. 存储 
3. iptables/ipvs

## 集群安装之后的操作

1. 所有节点安装metrics 
2. 安装监控
3. 配置基础的elk
4. 更新k8s证书时间

## 证书相关（let'sencrypt）

[kube-cert-manager](https://cert-manager.io/docs/installation/)

## 概述

随着 HTTPS 不断普及，大多数网站开始由 HTTP 升级到 HTTPS。使用 HTTPS 需要向权威机构申请证书，并且需要付出一定的成本，如果需求数量多，则开支也相对增加。cert-manager 是 Kubernetes 上的全能证书管理工具，支持利用 cert-manager 基于 [ACME](https://tools.ietf.org/html/rfc8555) 协议与 [Let's Encrypt](https://letsencrypt.org/) 签发免费证书并为证书自动续期，实现永久免费使用证书。

#### cert-manager 工作原理

cert-manager 部署到 Kubernetes 集群后会查阅其所支持的自定义资源 CRD，可通过创建 CRD 资源来指示 cert-manager 签发证书并为证书自动续期。如下图所示：
![img](./pic/f4e57b54c56515446c86ba05e7bc8f6c.svg)

- Issuer/ClusterIssuer

  ：用于指示 cert-manager 签发证书的方式，本文主要讲解签发免费证书的 ACME 方式。

  > 说明：
  >
  > Issuer 与 ClusterIssuer 之间的区别是：Issuer 只能用来签发自身所在 namespace 下的证书，ClusterIssuer 可以签发任意 namespace 下的证书。

- **Certificate**：用于向 cert-manager 传递域名证书的信息、签发证书所需要的配置，以及对 Issuer/ClusterIssuer 的引用。

### 免费证书签发原理

Let’s Encrypt 利用 ACME 协议校验域名的归属，校验成功后可以自动颁发免费证书。免费证书有效期只有90天，需在到期前再校验一次实现续期。使用 cert-manager 可以自动续期，即实现永久使用免费证书。校验域名归属的两种方式分别是 **HTTP-01** 和 **DNS-01**，校验原理详情可参见 [Let's Encrypt 的运作方式](https://letsencrypt.org/zh-cn/how-it-works/)。

- HTTP-01 校验原理
-  

- DNS-01 校验原理

HTTP-01 的校验原理是给域名指向的 HTTP 服务增加一个临时 location。此方法仅适用于给使用 Ingress 暴露流量的服务颁发证书，并且不支持泛域名证书。
例如，Let’s Encrypt 会发送 HTTP 请求到 `http://<YOUR_DOMAIN>/.well-known/acme-challenge/<TOKEN>`。`YOUR_DOMAIN` 是被校验的域名。`TOKEN` 是 ACME 协议客户端负责放置的文件，在此处 ACME 客户端即 cert-manager，通过修改或创建 Ingress 规则来增加临时校验路径并指向提供 `TOKEN` 的服务。Let’s Encrypt 会对比 `TOKEN` 是否符合预期，校验成功后就会颁发证书。

### 校验方式对比

- HTTP-01 校验方式的优点是配置简单通用，不同 DNS 提供商均可使用相同的配置方法。缺点是需要依赖 Ingress，若仅适用于服务支持 Ingress 暴露流量，不支持泛域名证书。
- DNS-01 校验方式的优点是不依赖 Ingress，并支持泛域名。缺点是不同 DNS 提供商的配置方式不同，DNS 提供商过多而 cert-manager 的 Issuer 不能全部支持。部分可以通过部署实现 cert-manager 的 [Webhook](https://cert-manager.io/docs/concepts/webhook/) 服务来扩展 Issuer 进行支持。例如 DNSPod 和 阿里 DNS，详情请参见 [Webhook 列表](https://cert-manager.io/docs/configuration/acme/dns01/#webhook)。

本文向您推荐 `DNS-01` 方式，其限制较少，功能较全。

### 操作步骤

### 安装 cert-manager

通常使用 yaml 方式一键安装 cert-manager 到集群，可参考官网文档 [Installing with regular manifests](https://cert-manager.io/docs/installation/kubernetes/#installing-with-regular-manifests)。
cert-manager 官方使用的镜像在 `quay.io` 进行拉取。也可以执行以下命令，使用同步到国内 CCR 的镜像一键安装：

> 注意：
>
> 使用这种方式要求集群版本不得低于1.16。



```
kubectl apply --validate=false -f https://raw.githubusercontent.com/TencentCloudContainerTeam/manifest/master/cert-manager/cert-manager.yaml
```



### 配置 DNS

登录 DNS 提供商后台，配置域名的 DNS A 记录，指向所需要证书的后端服务对外暴露的 IP 地址。以 cloudflare 为例，如下图所示：
![img](./pic/c133190ee796d15fb56b54e6b2417dc6.png)

#### HTTP-01 校验方式签发证书

若使用 HTTP-01 的校验方式，则需要用到 Ingress 来配合校验。cert-manager 会通过自动修改 Ingress 规则或自动新增 Ingress 来实现对外暴露校验所需的临时 HTTP 路径。为 Issuer 配置 HTTP-01 校验时，如果指定 Ingress 的 `name`，表示会自动修改指定 Ingress 的规则来暴露校验所需的临时 HTTP 路径，如果指定 `class`，则表示会自动新增 Ingress，可参考以下 [示例](https://cloud.tencent.com/document/product/457/49368#eg1)。

TKE 自带的 Ingress 中，每个 Ingress 资源都会对应一个负载均衡 CLB，如果使用 TKE 自带的 Ingress 暴露服务，并且使用 HTTP-01 方式校验，那么只能使用自动修改 Ingress 的方式，不能自动新增 Ingress。自动新增的 Ingress 会自动创建其他 CLB，使对外的 IP 地址与后端服务的 Ingress 不一致，Let's Encrypt 校验时将无法从服务的 Ingress 找到校验所需的临时路径，从而导致校验失败，无法签发证书。如果使用自建 Ingress，例如 [在 TKE 上部署 Nginx Ingress](https://cloud.tencent.com/document/product/457/47293)，同一个 Ingress class 的 Ingress 共享同一个 CLB，则支持使用自动新增 Ingress 的方式。

#### 示例

如果服务使用 TKE 自带的 Ingress 暴露服务，则不适合用 cert-manager 签发管理免费证书，证书从 [证书管理](https://console.cloud.tencent.com/ssl) 中被引用，不在 Kubernetes 中管理。
假设是 [在 TKE 上部署 Nginx Ingress](https://cloud.tencent.com/document/product/457/47293)，且后端服务的 Ingress 是 `prod/web`，可参考以下代码示例创建 Issuer：

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-http01
  namespace: prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-http01-account-key
    solvers:
    - http01:
       ingress:
         name: web # 指定被自动修改的 Ingress 名称
```

使用 Issuer 签发证书，cert-manager 会自动创建 Ingress 资源，并自动修改 Ingress 的资源 `prod/web`，以暴露校验所需的临时路径。参考以下代码示例，自动新增 Ingress：

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-http01
  namespace: prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-http01-account-key
    solvers:
    - http01:
       ingress:
         class: nginx # 指定自动创建的 Ingress 的 ingress class
```



成功创建 Issuer 后，参考以下代码示例，创建 Certificate 并引用 Issuer 进行签发：

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: test-mydomain-com
  namespace: prod
spec:
  dnsNames:
  - test.mydomain.com # 要签发证书的域名
  issuerRef:
    kind: Issuer
    name: letsencrypt-http01 # 引用 Issuer，指示采用 http01 方式进行校验
  secretName: test-mydomain-com-tls # 最终签发出来的证书会保存在这个 Secret 里面
```



#### DNS-01 校验方式签发证书

若使用 DNS-01 的校验方式，则需要选择 DNS 提供商。cert-manager 内置 DNS 提供商的支持，详细列表和用法请参见 [Supported DNS01 providers](https://cert-manager.io/docs/configuration/acme/dns01/#supported-dns01-providers)。若需要使用列表外的 DNS 提供商，可参考以下两种方案：

- 方案1：设置 Custom Nameserver
-  

- 方案2：使用 Webhook

在 DNS 提供商后台设置 custom nameserver，指向例如 cloudflare 此类可管理其它 DNS 提供商域名的 nameserver 地址，具体地址可登录 cloudflare 后台查看。如下图所示：
![img](./pic/9e07f843cae3ff5123442e7dc5b024d0.png)
namecheap 可以设置 custom nameserver，如下图所示：
![img](./pic/1ad9889154d2b4125cef8a41de26d413.png)
最后配置 Issuer 指定 DNS-01 验证时，添加 cloudflare 的信息即可。

### 获取和使用证书

[创建 Certificate](https://cloud.tencent.com/document/product/457/49368#Certificate) 后，即可通过 kubectl 查看证书是否签发成功。

```shell
$ kubectl get certificate -n prod
NAME                READY   SECRET                  AGE
test-mydomain-com   True    test-mydomain-com-tls   1m
```

- `READY` 为 `False`：则表示签发失败，可以通过 describe 命令查看 event 来排查失败原因

  ```shell
   kubectl describe certificate test-mydomain-com -n prod
  ```

- `READY` 为 `True`：则表示签发成功，证书将保存在所指定的 Secret 中。例如，`default/test-mydomain-com-tls`。可以通过 kubectl 查看，其中 `tls.crt` 是证书，`tls.key` 是密钥。

  ```shell
  $ kubectl get secret test-mydomain-com-tls -n default
  ...
  data:
    tls.crt: <cert>
    tls.key: <private key>
  ```

  

您可以将其挂载到需要证书的应用中，或者直接在自建的 Ingress 中引用 secret。可参考以下示例：

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    kubernetes.io/Ingress.class: nginx
spec:
  rules:
  - host: test.mydomain.com
    http:
      paths:
      - path: /web
        backend:
          service:
web
          servicePort: 80
  tls:
    hosts:
    - test.mydomain.com
    secretName: test-mydomain-com-tls
```

## jenkins 

1. 初始化 插件连接超时

   1. 在启动以后,输入密码,然后界面在缓冲 / 提示实例离线的时候, 打开`/pluginManager/advanced`

      在最下面设置`https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json`,然后提交,然后chenk now一下
      

## 使用rbd-provisioner提供rbd持久化存储

rbd-provisioner为kubernetes 1.5+版本提供了类似于`kubernetes.io/rbd`的ceph rbd持久化存储动态配置实现。

一些用户会使用kubeadm来部署集群，或者将kube-controller-manager以容器的方式运行。这种方式下，kubernetes在创建使用ceph rbd pv/pvc时没任何问题，但使用dynamic provisioning自动管理存储生命周期时会报错。提示`"rbd: create volume failed, err: failed to create rbd image: executable file not found in $PATH:"`。

问题来自gcr.io提供的kube-controller-manager容器镜像未打包ceph-common组件，缺少了rbd命令，因此无法通过rbd命令为pod创建rbd image，查了github的相关文章，目前kubernetes官方在kubernetes-incubator/external-storage项目通过External Provisioners的方式来解决此类问题。

本文主要针对该问题，通过rbd-provisioner的方式，解决ceph rbd的dynamic provisioning问题。

- 参考链接[RBD Volume Provisioner for Kubernetes 1.5+](https://github.com/kubernetes-incubator/external-storage/tree/master/ceph/rbd)

### 部署rbd-provisioner

首先得在kubernetes集群中安装rbd-provisioner，github仓库链接https://github.com/kubernetes-incubator/external-storage

```bash
[root@k8s01 ~]# git clone https://github.com/kubernetes-incubator/external-storage.git
[root@k8s01 ~]# cd external-storage/ceph/rbd/deploy
[root@k8s01 deploy]# NAMESPACE=kube-system
[root@k8s01 deploy]# sed -r -i "s/namespace: [^ ]+/namespace: $NAMESPACE/g" ./rbac/clusterrolebinding.yaml ./rbac/rolebinding.yaml
[root@k8s01 deploy]# kubectl -n $NAMESPACE apply -f ./rbac
```

- 根据自己需要，修改rbd-provisioner的namespace；

部署完成后检查rbd-provisioner deployment，确保已经正常部署；

```bash
[root@k8s01 ~]# kubectl describe deployments.apps -n kube-system rbd-provisioner
Name:               rbd-provisioner
Namespace:          kube-system
CreationTimestamp:  Sat, 13 Oct 2018 20:08:45 +0800
Labels:             app=rbd-provisioner
Annotations:        deployment.kubernetes.io/revision: 1
                    kubectl.kubernetes.io/last-applied-configuration:
                      {"apiVersion":"extensions/v1beta1","kind":"Deployment","metadata":{"annotations":{},"name":"rbd-provisioner","namespace":"kube-system"},"s...
Selector:           app=rbd-provisioner
Replicas:           1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:       Recreate
MinReadySeconds:    0
Pod Template:
  Labels:           app=rbd-provisioner
  Service Account:  rbd-provisioner
  Containers:
   rbd-provisioner:
    Image:      quay.io/external_storage/rbd-provisioner:latest
    Port:       <none>
    Host Port:  <none>
    Environment:
      PROVISIONER_NAME:  ceph.com/rbd
    Mounts:              <none>
  Volumes:               <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  <none>
NewReplicaSet:   rbd-provisioner-db574c5c (1/1 replicas created)
Events:          <none>
```

### 创建storageclass

部署完rbd-provisioner，还需要创建StorageClass。创建SC前，我们还需要创建相关用户的secret；

```bash
[root@k8s01 ~]# vi secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-admin-secret
  namespace: kube-system
type: "kubernetes.io/rbd"
data:
  # ceph auth get-key client.admin | base64
  key: QVFCdng4QmJKQkFsSFJBQWl1c1o0TGdOV250NlpKQ1BSMHFCa1E9PQ==
---
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
  namespace: kube-system
type: "kubernetes.io/rbd"
data:
  # ceph auth add client.kube mon 'allow r' osd 'allow rwx pool=kube'
  # ceph auth get-key client.kube | base64
  key: QVFCTHdNRmJueFZ4TUJBQTZjd1MybEJ2Q0JUcmZhRk4yL2tJQVE9PQ==

[root@k8s01 ~]#  kubectl create -f secrets.yaml

[root@k8s01 ~]# vi secrets-default.yaml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
type: "kubernetes.io/rbd"
data:
  # ceph auth add client.kube mon 'allow r' osd 'allow rwx pool=kube'
  # ceph auth get-key client.kube | base64
  key: QVFCTHdNRmJueFZ4TUJBQTZjd1MybEJ2Q0JUcmZhRk4yL2tJQVE9PQ==

[root@k8s01 ~]#  kubectl create -f secrets-default.yaml -n default
```

- 创建secret保存client.admin和client.kube用户的key，client.admin和client.kube用户的secret可以放在kube-system namespace，但如果其他namespace需要使用ceph rbd的dynamic provisioning功能的话，要在相应的namespace创建secret来保存client.kube用户key信息；

```bash
[root@k8s01 ~]# vi ceph-rbd-sc.yaml
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: ceph-rbd
  annotations:
     storageclass.beta.kubernetes.io/is-default-class: "true"
provisioner: ceph.com/rbd
parameters:
  monitors: 172.16.16.81,172.16.16.82,172.16.16.83
  adminId: admin
  adminSecretName: ceph-admin-secret
  adminSecretNamespace: kube-system
  pool: rbd
  userId: kube
  userSecretName: ceph-secret
  fsType: ext4
  imageFormat: "2"
  imageFeatures: "layering"

[root@k8s01 ~]#  kubectl create -f  ceph-rbd-sc.yaml
```

- 其他设置和普通的ceph rbd StorageClass一致，但provisioner需要设置为`ceph.com/rbd`，不是默认的`kubernetes.io/rbd`，这样rbd的请求将由rbd-provisioner来处理；
- 考虑到兼容性，建议尽量关闭rbd image feature，并且kubelet节点的ceph-common版本尽量和ceph服务器端保持一致，我的环境都使用的L版本；

### 测试ceph rbd自动分配

在kube-system和default namespace分别创建pod，通过启动一个busybox实例，将ceph rbd镜像挂载到`/usr/share/busybox`；

```bash
[root@k8s01 ~]# vi test-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: ceph-pod1
spec:
  containers:
  - name: ceph-busybox
    image: busybox
    command: ["sleep", "60000"]
    volumeMounts:
    - name: ceph-vol1
      mountPath: /usr/share/busybox
      readOnly: false
  volumes:
  - name: ceph-vol1
    persistentVolumeClaim:
      claimName: ceph-claim
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ceph-claim
spec:
  accessModes:  
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi

[root@k8s01 ~]# kubectl create -f test-pod.yaml -n kube-system
pod/ceph-pod1 created
persistentvolumeclaim/ceph-claim created
[root@k8s01 ~]# kubectl create -f test-pod.yaml -n default
pod/ceph-pod1 created
persistentvolumeclaim/ceph-claim created
```

检查pv和pvc的创建状态，是否都已经创建；

```bash
[root@k8s01 ~]# kubectl get pvc
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ceph-claim   Bound    pvc-ee0f1c35-cef7-11e8-8484-005056a33f16   2Gi        RWO            ceph-rbd       25s
[root@k8s01 ~]# kubectl get pvc -n kube-system
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ceph-claim   Bound    pvc-ea377cad-cef7-11e8-8484-005056a33f16   2Gi        RWO            ceph-rbd       36s
[root@k8s01 ~]# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   REASON   AGE
pvc-ea377cad-cef7-11e8-8484-005056a33f16   2Gi        RWO            Delete           Bound    kube-system/ceph-claim   ceph-rbd                40s
pvc-ee0f1c35-cef7-11e8-8484-005056a33f16   2Gi        RWO            Delete           Bound    default/ceph-claim       ceph-rbd                32s
```

在ceph服务器上，检查rbd镜像创建情况和镜像的信息；

```bash
[root@k8s01 ~]# rbd ls --pool rbd
kubernetes-dynamic-pvc-ea390cbf-cef7-11e8-aa22-0a580af40202
kubernetes-dynamic-pvc-eef5814f-cef7-11e8-aa22-0a580af40202

[root@k8s01 ~]# rbd info rbd/kubernetes-dynamic-pvc-ea390cbf-cef7-11e8-aa22-0a580af40202
rbd image 'kubernetes-dynamic-pvc-ea390cbf-cef7-11e8-aa22-0a580af40202':
    size 2048 MB in 512 objects
    order 22 (4096 kB objects)
    block_name_prefix: rbd_data.456876b8b4567
    format: 2
    features: layering
    flags:
    create_timestamp: Sat Oct 13 22:54:41 2018
[root@k8s01 ~]# rbd info rbd/kubernetes-dynamic-pvc-eef5814f-cef7-11e8-aa22-0a580af40202
rbd image 'kubernetes-dynamic-pvc-eef5814f-cef7-11e8-aa22-0a580af40202':
    size 2048 MB in 512 objects
    order 22 (4096 kB objects)
    block_name_prefix: rbd_data.ad6c6b8b4567
    format: 2
    features: layering
    flags:
    create_timestamp: Sat Oct 13 22:54:49 2018
```

检查busybox内的文件系统挂载和使用情况，确认能正常工作；

```bash
[root@k8s01 ~]# kubectl exec -it ceph-pod1 mount |grep rbd
/dev/rbd0 on /usr/share/busybox type ext4 (rw,seclabel,relatime,stripe=1024,data=ordered)
[root@k8s01 ~]# kubectl exec -it -n kube-system ceph-pod1 mount |grep rbd
/dev/rbd0 on /usr/share/busybox type ext4 (rw,seclabel,relatime,stripe=1024,data=ordered)

[root@k8s01 ~]# kubectl exec -it -n kube-system ceph-pod1 df |grep rbd
/dev/rbd0              1998672      6144   1976144   0% /usr/share/busybox
[root@k8s01 ~]# kubectl exec -it ceph-pod1 df |grep rbd
/dev/rbd0              1998672      6144   1976144   0% /usr/share/busybox
```

测试删除pod能否自动删除pv和pvc，生产环境中谨慎，设置好回收策略；

```bash
[root@k8s01 ~]# kubectl delete -f test-pod.yaml
pod "ceph-pod1" deleted
persistentvolumeclaim "ceph-claim" deleted

[root@k8s01 ~]# kubectl delete -f test-pod.yaml -n kube-system
pod "ceph-pod1" deleted
persistentvolumeclaim "ceph-claim" deleted

[root@k8s01 ~]# kubectl get pv
No resources found.
[root@k8s01 ~]# kubectl get pvc
No resources found.
[root@k8s01 ~]# kubectl get pvc -n kube-system
No resources found.
```

ceph服务器上的rbd image也已清除，自动回收成功；

```bash
[root@k8s01 ~]# rbd ls --pool rbd
```

- 确认之前创建的rbd images都已经删除；

### 总结

大部分情况下，我们无需使用rbd provisioner来提供ceph rbd的dynamic provisioning能力。经测试，在OpenShift、Rancher、SUSE CaaS以及本Handbook的二进制文件方式部署，在安装好ceph-common软件包的情况下，定义StorageClass时使用`kubernetes.io/rbd`即可正常使用ceph rbd provisioning功能。

### 参考

- [RBD Volume Provisioner for Kubernetes 1.5+](https://github.com/kubernetes-incubator/external-storage/tree/master/ceph/rbd)



# kubectl 常用命令

Kubctl 命令是操作 kubernetes 集群的最直接和最 skillful 的途径，这个60多MB大小的二进制文件，到底有啥能耐呢？请看下文：

## Kubectl 自动补全

```bash
$ source <(kubectl completion bash) # setup autocomplete in bash, bash-completion package should be installed first.
$ source <(kubectl completion zsh)  # setup autocomplete in zsh
```

## Kubectl 上下文和配置

设置 `kubectl` 命令交互的 kubernetes 集群并修改配置信息。参阅 [使用 kubeconfig 文件进行跨集群验证](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters) 获取关于配置文件的详细信息。

```bash
$ kubectl config view # 显示合并后的 kubeconfig 配置

# 同时使用多个 kubeconfig 文件并查看合并后的配置
$ KUBECONFIG=~/.kube/config:~/.kube/kubconfig2 kubectl config view

# 获取 e2e 用户的密码
$ kubectl config view -o jsonpath='{.users[?(@.name == "e2e")].user.password}'

$ kubectl config current-context              # 显示当前的上下文
$ kubectl config use-context my-cluster-name  # 设置默认上下文为 my-cluster-name

# 向 kubeconf 中增加支持基本认证的新集群
$ kubectl config set-credentials kubeuser/foo.kubernetes.com --username=kubeuser --password=kubepassword

# 使用指定的用户名和 namespace 设置上下文
$ kubectl config set-context gce --user=cluster-admin --namespace=foo \
  && kubectl config use-context gce
```

## 创建对象

Kubernetes 的清单文件可以使用 json 或 yaml 格式定义。可以以 `.yaml`、`.yml`、或者 `.json` 为扩展名。

```yaml
$ kubectl create -f ./my-manifest.yaml           # 创建资源
$ kubectl create -f ./my1.yaml -f ./my2.yaml     # 使用多个文件创建资源
$ kubectl create -f ./dir                        # 使用目录下的所有清单文件来创建资源
$ kubectl create -f https://git.io/vPieo         # 使用 url 来创建资源
$ kubectl run nginx --image=nginx                # 启动一个 nginx 实例
$ kubectl explain pods,svc                       # 获取 pod 和 svc 的文档

# 从 stdin 输入中创建多个 YAML 对象
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: busybox-sleep
spec:
  containers:
  - name: busybox
    image: busybox
    args:
    - sleep
    - "1000000"
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox-sleep-less
spec:
  containers:
  - name: busybox
    image: busybox
    args:
    - sleep
    - "1000"
EOF

# 创建包含几个 key 的 Secret
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  password: $(echo "s33msi4" | base64)
  username: $(echo "jane" | base64)
EOF
```

## 显示和查找资源

```bash
# Get commands with basic output
$ kubectl get services                          # 列出所有 namespace 中的所有 service
$ kubectl get pods --all-namespaces             # 列出所有 namespace 中的所有 pod
$ kubectl get pods -o wide                      # 列出所有 pod 并显示详细信息
$ kubectl get deployment my-dep                 # 列出指定 deployment
$ kubectl get pods --include-uninitialized      # 列出该 namespace 中的所有 pod 包括未初始化的

# 使用详细输出来描述命令
$ kubectl describe nodes my-node
$ kubectl describe pods my-pod

$ kubectl get services --sort-by=.metadata.name # List Services Sorted by Name

# 根据重启次数排序列出 pod
$ kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'

# 获取所有具有 app=cassandra 的 pod 中的 version 标签
$ kubectl get pods --selector=app=cassandra rc -o \
  jsonpath='{.items[*].metadata.labels.version}'

# 获取所有节点的 ExternalIP
$ kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'

# 列出属于某个 PC 的 Pod 的名字
# “jq”命令用于转换复杂的 jsonpath，参考 https://stedolan.github.io/jq/
$ sel=${$(kubectl get rc my-rc --output=json | jq -j '.spec.selector | to_entries | .[] | "\(.key)=\(.value),"')%?}
$ echo $(kubectl get pods --selector=$sel --output=jsonpath={.items..metadata.name})

# 查看哪些节点已就绪
$ JSONPATH='{range .items[*]}{@.metadata.name}:{range @.status.conditions[*]}{@.type}={@.status};{end}{end}' \
 && kubectl get nodes -o jsonpath="$JSONPATH" | grep "Ready=True"

# 列出当前 Pod 中使用的 Secret
$ kubectl get pods -o json | jq '.items[].spec.containers[].env[]?.valueFrom.secretKeyRef.name' | grep -v null | sort | uniq
```

## 更新资源

```bash
$ kubectl rolling-update frontend-v1 -f frontend-v2.json           # 滚动更新 pod frontend-v1
$ kubectl rolling-update frontend-v1 frontend-v2 --image=image:v2  # 更新资源名称并更新镜像
$ kubectl rolling-update frontend --image=image:v2                 # 更新 frontend pod 中的镜像
$ kubectl rolling-update frontend-v1 frontend-v2 --rollback        # 退出已存在的进行中的滚动更新
$ cat pod.json | kubectl replace -f -                              # 基于 stdin 输入的 JSON 替换 pod

# 强制替换，删除后重新创建资源。会导致服务中断。
$ kubectl replace --force -f ./pod.json

# 为 nginx RC 创建服务，启用本地 80 端口连接到容器上的 8000 端口
$ kubectl expose rc nginx --port=80 --target-port=8000

# 更新单容器 pod 的镜像版本（tag）到 v4
$ kubectl get pod mypod -o yaml | sed 's/\(image: myimage\):.*$/\1:v4/' | kubectl replace -f -

$ kubectl label pods my-pod new-label=awesome                      # 添加标签
$ kubectl annotate pods my-pod icon-url=http://goo.gl/XXBTWq       # 添加注解
$ kubectl autoscale deployment foo --min=2 --max=10                # 自动扩展 deployment “foo”
```

## 修补资源

使用策略合并补丁并修补资源。

```bash
$ kubectl patch node k8s-node-1 -p '{"spec":{"unschedulable":true}}' # 部分更新节点

# 更新容器镜像； spec.containers[*].name 是必须的，因为这是合并的关键字
$ kubectl patch pod valid-pod -p '{"spec":{"containers":[{"name":"kubernetes-serve-hostname","image":"new image"}]}}'

# 使用具有位置数组的 json 补丁更新容器镜像
$ kubectl patch pod valid-pod --type='json' -p='[{"op": "replace", "path": "/spec/containers/0/image", "value":"new image"}]'

# 使用具有位置数组的 json 补丁禁用 deployment 的 livenessProbe
$ kubectl patch deployment valid-deployment  --type json   -p='[{"op": "remove", "path": "/spec/template/spec/containers/0/livenessProbe"}]'
```

## 编辑资源

在编辑器中编辑任何 API 资源。

```bash
$ kubectl edit svc/docker-registry                      # 编辑名为 docker-registry 的 service
$ KUBE_EDITOR="nano" kubectl edit svc/docker-registry   # 使用其它编辑器
```

## Scale 资源

```bash
$ kubectl scale --replicas=3 rs/foo                                 # Scale a replicaset named 'foo' to 3
$ kubectl scale --replicas=3 -f foo.yaml                            # Scale a resource specified in "foo.yaml" to 3
$ kubectl scale --current-replicas=2 --replicas=3 deployment/mysql  # If the deployment named mysql's current size is 2, scale mysql to 3
$ kubectl scale --replicas=5 rc/foo rc/bar rc/baz                   # Scale multiple replication controllers
```

## 删除资源

```bash
$ kubectl delete -f ./pod.json                                              # 删除 pod.json 文件中定义的类型和名称的 pod
$ kubectl delete pod,service baz foo                                        # 删除名为“baz”的 pod 和名为“foo”的 service
$ kubectl delete pods,services -l name=myLabel                              # 删除具有 name=myLabel 标签的 pod 和 serivce
$ kubectl delete pods,services -l name=myLabel --include-uninitialized      # 删除具有 name=myLabel 标签的 pod 和 service，包括尚未初始化的
$ kubectl -n my-ns delete po,svc --all                                      # 删除 my-ns namespace 下的所有 pod 和 serivce，包括尚未初始化的
```

## 与运行中的 Pod 交互

```bash
$ kubectl logs my-pod                                 # dump 输出 pod 的日志（stdout）
$ kubectl logs my-pod -c my-container                 # dump 输出 pod 中容器的日志（stdout，pod 中有多个容器的情况下使用）
$ kubectl logs -f my-pod                              # 流式输出 pod 的日志（stdout）
$ kubectl logs -f my-pod -c my-container              # 流式输出 pod 中容器的日志（stdout，pod 中有多个容器的情况下使用）
$ kubectl run -i --tty busybox --image=busybox -- sh  # 交互式 shell 的方式运行 pod
$ kubectl attach my-pod -i                            # 连接到运行中的容器
$ kubectl port-forward my-pod 5000:6000               # 转发 pod 中的 6000 端口到本地的 5000 端口
$ kubectl exec my-pod -- ls /                         # 在已存在的容器中执行命令（只有一个容器的情况下）
$ kubectl exec my-pod -c my-container -- ls /         # 在已存在的容器中执行命令（pod 中有多个容器的情况下）
$ kubectl top pod POD_NAME --containers               # 显示指定 pod 和容器的指标度量
```

## 与节点和集群交互

```bash
$ kubectl cordon my-node                                                # 标记 my-node 不可调度
$ kubectl drain my-node                                                 # 清空 my-node 以待维护
$ kubectl uncordon my-node                                              # 标记 my-node 可调度
$ kubectl top node my-node                                              # 显示 my-node 的指标度量
$ kubectl cluster-info                                                  # 显示 master 和服务的地址
$ kubectl cluster-info dump                                             # 将当前集群状态输出到 stdout                                    
$ kubectl cluster-info dump --output-directory=/path/to/cluster-state   # 将当前集群状态输出到 /path/to/cluster-state

# 如果该键和影响的污点（taint）已存在，则使用指定的值替换
$ kubectl taint nodes foo dedicated=special-user:NoSchedule
```

## 资源类型

下表列出的是 kubernetes 中所有支持的类型和缩写的别名。

| 资源类型                   | 缩写别名 |
| -------------------------- | -------- |
| `clusters`                 |          |
| `componentstatuses`        | `cs`     |
| `configmaps`               | `cm`     |
| `daemonsets`               | `ds`     |
| `deployments`              | `deploy` |
| `endpoints`                | `ep`     |
| `event`                    | `ev`     |
| `horizontalpodautoscalers` | `hpa`    |
| `ingresses`                | `ing`    |
| `jobs`                     |          |
| `limitranges`              | `limits` |
| `namespaces`               | `ns`     |
| `networkpolicies`          |          |
| `nodes`                    | `no`     |
| `statefulsets`             |          |
| `persistentvolumeclaims`   | `pvc`    |
| `persistentvolumes`        | `pv`     |
| `pods`                     | `po`     |
| `podsecuritypolicies`      | `psp`    |
| `podtemplates`             |          |
| `replicasets`              | `rs`     |
| `replicationcontrollers`   | `rc`     |
| `resourcequotas`           | `quota`  |
| `cronjob`                  |          |
| `secrets`                  |          |
| `serviceaccount`           | `sa`     |
| `services`                 | `svc`    |
| `storageclasses`           |          |
| `thirdpartyresources`      |          |

### 格式化输出

要以特定的格式向终端窗口输出详细信息，可以在 `kubectl` 命令中添加 `-o` 或者 `-output` 标志。

| 输出格式                            | 描述                                                         |
| ----------------------------------- | ------------------------------------------------------------ |
| `-o=custom-columns=<spec>`          | 使用逗号分隔的自定义列列表打印表格                           |
| `-o=custom-columns-file=<filename>` | 使用 文件中的自定义列模板打印表格                            |
| `-o=json`                           | 输出 JSON 格式的 API 对象                                    |
| `-o=jsonpath=<template>`            | 打印 [jsonpath](https://kubernetes.io/docs/user-guide/jsonpath) 表达式中定义的字段 |
| `-o=jsonpath-file=<filename>`       | 打印由 文件中的 [jsonpath](https://kubernetes.io/docs/user-guide/jsonpath) 表达式定义的字段 |
| `-o=name`                           | 仅打印资源名称                                               |
| `-o=wide`                           | 以纯文本格式输出任何附加信息，对于 Pod ，包含节点名称        |
| `-o=yaml`                           | 输出 YAML 格式的 API 对象                                    |

### Kubectl 详细输出和调试

使用 `-v` 或 `--v` 标志跟着一个整数来指定日志级别。

| 详细等级 | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| `--v=0`  | 总是对操作人员可见。                                         |
| `--v=1`  | 合理的默认日志级别，如果您不需要详细输出。                   |
| `--v=2`  | 可能与系统的重大变化相关的，有关稳定状态的信息和重要的日志信息。这是对大多数系统推荐的日志级别。 |
| `--v=3`  | 有关更改的扩展信息。                                         |
| `--v=4`  | 调试级别详细输出。                                           |
| `--v=6`  | 显示请求的资源。                                             |
| `--v=7`  | 显示HTTP请求的header。                                       |
| `--v=8`  | 显示HTTP请求的内容。                                         |

# k8s 初始化

## 使用 kubeadm 创建一个高可用 etcd 集群

**说明：**

在本指南中，当 kubeadm 用作为外部 etcd 节点管理工具，请注意 kubeadm 不计划支持此类节点的证书更换或升级。对于长期规划是使用 [etcdadm](https://github.com/kubernetes-sigs/etcdadm) 增强工具来管理这方面。

默认情况下，kubeadm 运行单成员的 etcd 集群，该集群由控制面节点上的 kubelet 以静态 Pod 的方式进行管理。由于 etcd 集群只包含一个成员且不能在任一成员不可用时保持运行，所以这不是一种高可用设置。本任务，将告诉你如何在使用 kubeadm 创建一个 kubernetes 集群时创建一个外部 etcd：有三个成员的高可用 etcd 集群。

### 准备开始

- 三个可以通过 2379 和 2380 端口相互通信的主机。本文档使用这些作为默认端口。不过，它们可以通过 kubeadm 的配置文件进行自定义。

- 每个主机必须 [安装有 docker、kubelet 和 kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)。

- 一些可以用来在主机间复制文件的基础设施。例如 `ssh` 和 `scp` 就可以满足需求。

### 建立集群

一般来说，是在一个节点上生成所有证书并且只分发这些*必要*的文件到其它节点上。

**说明：**

kubeadm 包含生成下述证书所需的所有必要的密码学工具；在这个例子中，不需要其他加密工具。

1. 将 kubelet 配置为 etcd 的服务管理器。

   

   **说明：** 你必须在要运行 etcd 的所有主机上执行此操作。

   由于 etcd 是首先创建的，因此你必须通过创建具有更高优先级的新文件来覆盖 kubeadm 提供的 kubelet 单元文件。

   

   ```sh
   cat << EOF > /etc/systemd/system/kubelet.service.d/20-etcd-service-manager.conf
   [Service]
   ExecStart=
   # 将下面的 "systemd" 替换为你的容器运行时所使用的 cgroup 驱动。
   # kubelet 的默认值为 "cgroupfs"。
   ExecStart=/usr/bin/kubelet --address=127.0.0.1 --pod-manifest-path=/etc/kubernetes/manifests --cgroup-driver=systemd
   Restart=always
   EOF
   
   systemctl daemon-reload
   systemctl restart kubelet
   ```

   检查 kubelet 的状态以确保其处于运行状态：

   ```shell
   systemctl status kubelet
   ```

1. 为 kubeadm 创建配置文件。

   使用以下脚本为每个将要运行 etcd 成员的主机生成一个 kubeadm 配置文件。

   ```sh
   # 使用 IP 或可解析的主机名替换 HOST0、HOST1 和 HOST2
   export HOST0=10.0.0.6
   export HOST1=10.0.0.7
   export HOST2=10.0.0.8
   
   # 创建临时目录来存储将被分发到其它主机上的文件
   mkdir -p /tmp/${HOST0}/ /tmp/${HOST1}/ /tmp/${HOST2}/
   
   ETCDHOSTS=(${HOST0} ${HOST1} ${HOST2})
   NAMES=("infra0" "infra1" "infra2")
   
   for i in "${!ETCDHOSTS[@]}"; do
   HOST=${ETCDHOSTS[$i]}
   NAME=${NAMES[$i]}
   cat << EOF > /tmp/${HOST}/kubeadmcfg.yaml
   apiVersion: "kubeadm.k8s.io/v1beta3"
   kind: ClusterConfiguration
   etcd:
       local:
           serverCertSANs:
           - "${HOST}"
           peerCertSANs:
           - "${HOST}"
           extraArgs:
               initial-cluster: infra0=https://${ETCDHOSTS[0]}:2380,infra1=https://${ETCDHOSTS[1]}:2380,infra2=https://${ETCDHOSTS[2]}:2380
               initial-cluster-state: new
               name: ${NAME}
               listen-peer-urls: https://${HOST}:2380
               listen-client-urls: https://${HOST}:2379
               advertise-client-urls: https://${HOST}:2379
               initial-advertise-peer-urls: https://${HOST}:2380
   EOF
   done
   ```

1. 生成证书颁发机构

   如果你已经拥有 CA，那么唯一的操作是复制 CA 的 `crt` 和 `key` 文件到 `etc/kubernetes/pki/etcd/ca.crt` 和 `/etc/kubernetes/pki/etcd/ca.key`。 复制完这些文件后继续下一步，“为每个成员创建证书”。

   如果你还没有 CA，则在 `$HOST0`（你为 kubeadm 生成配置文件的位置）上运行此命令。

   ```
   kubeadm init phase certs etcd-ca
   ```

   这一操作创建如下两个文件

   - `/etc/kubernetes/pki/etcd/ca.crt`
   - `/etc/kubernetes/pki/etcd/ca.key`

1. 为每个成员创建证书

   ```shell
   kubeadm init phase certs etcd-server --config=/tmp/${HOST2}/kubeadmcfg.yaml
   kubeadm init phase certs etcd-peer --config=/tmp/${HOST2}/kubeadmcfg.yaml
   kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
   kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
   cp -R /etc/kubernetes/pki /tmp/${HOST2}/
   # 清理不可重复使用的证书
   find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete
   
   kubeadm init phase certs etcd-server --config=/tmp/${HOST1}/kubeadmcfg.yaml
   kubeadm init phase certs etcd-peer --config=/tmp/${HOST1}/kubeadmcfg.yaml
   kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
   kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
   cp -R /etc/kubernetes/pki /tmp/${HOST1}/
   find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete
   
   kubeadm init phase certs etcd-server --config=/tmp/${HOST0}/kubeadmcfg.yaml
   kubeadm init phase certs etcd-peer --config=/tmp/${HOST0}/kubeadmcfg.yaml
   kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
   kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
   # 不需要移动 certs 因为它们是给 HOST0 使用的
   
   # 清理不应从此主机复制的证书
   find /tmp/${HOST2} -name ca.key -type f -delete
   find /tmp/${HOST1} -name ca.key -type f -delete
   ```

1. 复制证书和 kubeadm 配置

   证书已生成，现在必须将它们移动到对应的主机。

   ```shell
   USER=ubuntu
   HOST=${HOST1}
   scp -r /tmp/${HOST}/* ${USER}@${HOST}:
   ssh ${USER}@${HOST}
   USER@HOST $ sudo -Es
   root@HOST $ chown -R root:root pki
   root@HOST $ mv pki /etc/kubernetes/
   ```

1. 确保已经所有预期的文件都存在

   `$HOST0` 所需文件的完整列表如下：

   ```none
   /tmp/${HOST0}
   └── kubeadmcfg.yaml
   ---
   /etc/kubernetes/pki
   ├── apiserver-etcd-client.crt
   ├── apiserver-etcd-client.key
   └── etcd
       ├── ca.crt
       ├── ca.key
       ├── healthcheck-client.crt
       ├── healthcheck-client.key
       ├── peer.crt
       ├── peer.key
       ├── server.crt
       └── server.key
   ```

   在 `$HOST1` 上：

   ```
   $HOME
   └── kubeadmcfg.yaml
   ---
   /etc/kubernetes/pki
   ├── apiserver-etcd-client.crt
   ├── apiserver-etcd-client.key
   └── etcd
       ├── ca.crt
       ├── healthcheck-client.crt
       ├── healthcheck-client.key
       ├── peer.crt
       ├── peer.key
       ├── server.crt
       └── server.key
   ```

   在 `$HOST2` 上：

   ```
   $HOME
   └── kubeadmcfg.yaml
   ---
   /etc/kubernetes/pki
   ├── apiserver-etcd-client.crt
   ├── apiserver-etcd-client.key
   └── etcd
       ├── ca.crt
       ├── healthcheck-client.crt
       ├── healthcheck-client.key
       ├── peer.crt
       ├── peer.key
       ├── server.crt
       └── server.key
   ```

1. 创建静态 Pod 清单

   既然证书和配置已经就绪，是时候去创建清单了。 在每台主机上运行 `kubeadm` 命令来生成 etcd 使用的静态清单。

   ```shell
   root@HOST0 $ kubeadm init phase etcd local --config=/tmp/${HOST0}/kubeadmcfg.yaml
   root@HOST1 $ kubeadm init phase etcd local --config=/tmp/${HOST1}/kubeadmcfg.yaml
   root@HOST2 $ kubeadm init phase etcd local --config=/tmp/${HOST2}/kubeadmcfg.yaml
   ```

1. 可选：检查群集运行状况

   ```shell
   docker run --rm -it \
   --net host \
   -v /etc/kubernetes:/etc/kubernetes k8s.gcr.io/etcd:${ETCD_TAG} etcdctl \
   --cert /etc/kubernetes/pki/etcd/peer.crt \
   --key /etc/kubernetes/pki/etcd/peer.key \
   --cacert /etc/kubernetes/pki/etcd/ca.crt \
   --endpoints https://${HOST0}:2379 endpoint health --cluster
   ...
   https://[HOST0 IP]:2379 is healthy: successfully committed proposal: took = 16.283339ms
   https://[HOST1 IP]:2379 is healthy: successfully committed proposal: took = 19.44402ms
   https://[HOST2 IP]:2379 is healthy: successfully committed proposal: took = 35.926451ms
   ```

   - 将 `${ETCD_TAG}` 设置为你的 etcd 镜像的版本标签，例如 `3.4.3-0`。 要查看 kubeadm 使用的 etcd 镜像和标签，请执行 `kubeadm config images list --kubernetes-version ${K8S_VERSION}`， 例如，其中的 `${K8S_VERSION}` 可以是 `v1.17.0`。
   - 将 `${HOST0}` 设置为要测试的主机的 IP 地址。

## 利用 kubeadm 创建高可用集群

本文讲述了使用 kubeadm 设置一个高可用的 Kubernetes 集群的两种不同方式：

- 使用具有堆叠的控制平面节点。这种方法所需基础设施较少。etcd 成员和控制平面节点位于同一位置。
- 使用外部集群。这种方法所需基础设施较多。控制平面的节点和 etcd 成员是分开的。

在下一步之前，你应该仔细考虑哪种方法更好的满足你的应用程序和环境的需求。 [这是对比文档](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/ha-topology/) 讲述了每种方法的优缺点。

如果你在安装 HA 集群时遇到问题，请在 kubeadm [问题跟踪](https://github.com/kubernetes/kubeadm/issues/new)里向我们提供反馈。

你也可以阅读 [升级文件](https://kubernetes.io/zh/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

**注意：** 这篇文档没有讲述在云提供商上运行集群的问题。在云环境中，此处记录的方法不适用于类型为 LoadBalancer 的服务对象，或者具有动态的 PersistentVolumes。

## 准备开始

对于这两种方法，你都需要以下基础设施：

- 配置满足 [kubeadm 的最低要求](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin) 的三台机器作为控制面节点
- 配置满足 [kubeadm 的最低要求](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin) 的三台机器作为工作节点
- 在集群中，确保所有计算机之间存在全网络连接（公网或私网）
- 在所有机器上具有 sudo 权限
- 从某台设备通过 SSH 访问系统中所有节点的能力
- 所有机器上已经安装 `kubeadm` 和 `kubelet`，`kubectl` 是可选的。

仅对于外部 etcd 集群来说，你还需要：

- 给 etcd 成员使用的另外三台机器

## 这两种方法的第一步

### 为 kube-apiserver 创建负载均衡器

**说明：**

使用负载均衡器需要许多配置。你的集群搭建可能需要不同的配置。 下面的例子只是其中的一方面配置。

1. 创建一个名为 kube-apiserver 的负载均衡器解析 DNS。
   - 在云环境中，应该将控制平面节点放置在 TCP 后面转发负载平衡。 该负载均衡器将流量分配给目标列表中所有运行状况良好的控制平面节点。 API 服务器的健康检查是在 kube-apiserver 的监听端口（默认值 `:6443`） 上进行的一个 TCP 检查。
   - 不建议在云环境中直接使用 IP 地址。
   - 负载均衡器必须能够在 API 服务器端口上与所有控制平面节点通信。 它还必须允许其监听端口的入站流量。
   - 确保负载均衡器的地址始终匹配 kubeadm 的 `ControlPlaneEndpoint` 地址。
   - 阅读[软件负载平衡选项指南](https://git.k8s.io/kubeadm/docs/ha-considerations.md#options-for-software-load-balancing)以获取更多详细信息。

1. 添加第一个控制平面节点到负载均衡器并测试连接：

   ```shell
   nc -v LOAD_BALANCER_IP PORT
   ```

   - 由于 apiserver 尚未运行，预期会出现一个连接拒绝错误。 然而超时意味着负载均衡器不能和控制平面节点通信。 如果发生超时，请重新配置负载均衡器与控制平面节点进行通信。

2. 将其余控制平面节点添加到负载均衡器目标组。

## 使用堆控制平面和 etcd 节点

### 控制平面节点的第一步

1. 初始化控制平面：

   ```shell
   sudo kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" --upload-certs
   ```

   - 你可以使用 `--kubernetes-version` 标志来设置要使用的 Kubernetes 版本。 建议将 kubeadm、kebelet、kubectl 和 Kubernetes 的版本匹配。
   - 这个 `--control-plane-endpoint` 标志应该被设置成负载均衡器的地址或 DNS 和端口。
   - 这个 `--upload-certs` 标志用来将在所有控制平面实例之间的共享证书上传到集群。 如果正好相反，你更喜欢手动地通过控制平面节点或者使用自动化 工具复制证书，请删除此标志并参考如下部分[证书分配手册](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/high-availability/#manual-certs)。

   **说明：**

   ```
   标志 `kubeadm init`、`--config` 和 `--certificate-key` 不能混合使用，
   因此如果你要使用
   [kubeadm 配置](/docs/reference/config-api/kubeadm-config.v1beta3/)，你必须在相应的配置文件
   （位于 `InitConfiguration` 和 `JoinConfiguration: controlPlane`）添加 `certificateKey` 字段。
   ```

   **说明：**

   ```
   一些 CNI 网络插件如 Calico 需要 CIDR 例如 `192.168.0.0/16` 和一些像 Weave 没有。参考
   [CNI 网络文档](/zh/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network)。
   通过传递 `--pod-network-cidr` 标志添加 pod CIDR，或者你可以使用 kubeadm
   配置文件，在 `ClusterConfiguration` 的 `networking` 对象下设置 `podSubnet` 字段。
   ```

   - 输出类似于：

     ```sh
     ...
     You can now join any number of control-plane node by running the following command on each as a root:
     kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 --control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07
     
     Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
     As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use kubeadm init phase upload-certs to reload certs afterward.
     
     Then you can join any number of worker nodes by running the following on each as root:
       kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866
     ```

   - 将此输出复制到文本文件。 稍后你将需要它来将控制平面节点和工作节点加入集群。

   - 当 `--upload-certs` 与 `kubeadm init` 一起使用时，主控制平面的证书 被加密并上传到 `kubeadm-certs` Secret 中。

   - 要重新上传证书并生成新的解密密钥，请在已加入集群节点的控制平面上使用以下命令：

     ```shell
     sudo kubeadm init phase upload-certs --upload-certs
     ```

   - 你还可以在 `init` 期间指定自定义的 `--certificate-key`，以后可以由 `join` 使用。 要生成这样的密钥，可以使用以下命令：

     ```shell
     kubeadm certs certificate-key
     ```

   **说明：**

   ```
   `kubeadm-certs` 密钥和解密密钥会在两个小时后失效。
   ```

   **注意：**

   ```
   正如命令输出中所述，证书密钥可访问群集敏感数据。请妥善保管！
   ```

1. 应用你所选择的 CNI 插件： [请遵循以下指示](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network) 安装 CNI 提供程序。如果适用，请确保配置与 kubeadm 配置文件中指定的 Pod CIDR 相对应。

   在此示例中，我们使用 Weave Net：

   ```shell
   kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
   ```

1. 输入以下内容，并查看控制平面组件的 Pods 启动：

   ```shell
   kubectl get pod -n kube-system -w
   ```

### 其余控制平面节点的步骤

**说明：**

从 kubeadm 1.15 版本开始，你可以并行加入多个控制平面节点。 在此版本之前，你必须在第一个节点初始化后才能依序的增加新的控制平面节点。

对于每个其他控制平面节点，你应该：

1. 执行先前由第一个节点上的 `kubeadm init` 输出提供给你的 join 命令。 它看起来应该像这样：

   ```sh
   sudo kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 --control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07
   ```

   - 这个 `--control-plane` 命令通知 `kubeadm join` 创建一个新的控制平面。
   - `--certificate-key ...` 将导致从集群中的 `kubeadm-certs` Secret 下载 控制平面证书并使用给定的密钥进行解密。

## 外部 etcd 节点

使用外部 etcd 节点设置集群类似于用于堆叠 etcd 的过程， 不同之处在于你应该首先设置 etcd，并在 kubeadm 配置文件中传递 etcd 信息。

### 设置 ectd 集群

1. 按照 [这些指示](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/setup-ha-etcd-with-kubeadm/) 去设置 etcd 集群。

2. 根据[这里](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/high-availability/#manual-certs)的描述配置 SSH。

3. 将以下文件从集群中的任何 etcd 节点复制到第一个控制平面节点：

   ```shell
   export CONTROL_PLANE="ubuntu@10.0.0.7"
   scp /etc/kubernetes/pki/etcd/ca.crt "${CONTROL_PLANE}":
   scp /etc/kubernetes/pki/apiserver-etcd-client.crt "${CONTROL_PLANE}":
   scp /etc/kubernetes/pki/apiserver-etcd-client.key "${CONTROL_PLANE}":
   ```

   - 用第一台控制平面机的 `user@host` 替换 `CONTROL_PLANE` 的值。

### 设置第一个控制平面节点

1. 用以下内容创建一个名为 `kubeadm-config.yaml` 的文件：

   ```yaml
   apiVersion: kubeadm.k8s.io/v1beta2
   kind: ClusterConfiguration
   kubernetesVersion: stable
   controlPlaneEndpoint: "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT"
   etcd:
       external:
           endpoints:
           - https://ETCD_0_IP:2379
           - https://ETCD_1_IP:2379
           - https://ETCD_2_IP:2379
           caFile: /etc/kubernetes/pki/etcd/ca.crt
           certFile: /etc/kubernetes/pki/apiserver-etcd-client.crt
           keyFile: /etc/kubernetes/pki/apiserver-etcd-client.key
   ```

   **说明：**

   ```
   这里的内部（stacked） etcd 和外部 etcd 之前的区别在于设置外部 etcd
   需要一个 `etcd` 的 `external` 对象下带有 etcd 端点的配置文件。
   如果是内部 etcd，是自动管理的。
   ```

   - 在你的集群中，将配置模板中的以下变量替换为适当值：
     - `LOAD_BALANCER_DNS`
     - `LOAD_BALANCER_PORT`
     - `ETCD_0_IP`
     - `ETCD_1_IP`
     - `ETCD_2_IP`

以下的步骤与设置内置 etcd 的集群是相似的：

1. 在节点上运行 `sudo kubeadm init --config kubeadm-config.yaml --upload-certs` 命令。

2. 记下输出的 join 命令，这些命令将在以后使用。

3. 应用你选择的 CNI 插件。以下示例适用于 Weave Net：

   ```shell
   kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
   ```

### 其他控制平面节点的步骤

步骤与设置内置 etcd 相同：

- 确保第一个控制平面节点已完全初始化。
- 使用保存到文本文件的 join 命令将每个控制平面节点连接在一起。 建议一次加入一个控制平面节点。
- 不要忘记默认情况下，`--certificate-key` 中的解密秘钥会在两个小时后过期。

## 列举控制平面之后的常见任务

### 安装工作节点

你可以使用之前存储的 `kubeadm init` 命令的输出将工作节点加入集群中：

```sh
sudo kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866
```

## 手动证书分发

如果你选择不将 `kubeadm init` 与 `--upload-certs` 命令一起使用， 则意味着你将必须手动将证书从主控制平面节点复制到 将要加入的控制平面节点上。

有许多方法可以实现这种操作。在下面的例子中我们使用 `ssh` 和 `scp`：

如果要在单独的一台计算机控制所有节点，则需要 SSH。

1. 在你的主设备上启用 ssh-agent，要求该设备能访问系统中的所有其他节点：

   ```shell
   eval $(ssh-agent)
   ```

1. 将 SSH 身份添加到会话中：

   ```shell
   ssh-add ~/.ssh/path_to_private_key
   ```

1. 检查节点间的 SSH 以确保连接是正常运行的

   - SSH 到任何节点时，请确保添加 `-A` 标志：

     ```shell
     ssh -A 10.0.0.7
     ```

   - 当在任何节点上使用 sudo 时，请确保保持环境变量设置，以便 SSH 转发能够正常工作：

     ```shell
     sudo -E -s
     ```

1. 在所有节点上配置 SSH 之后，你应该在运行过 `kubeadm init` 命令的第一个 控制平面节点上运行以下脚本。 该脚本会将证书从第一个控制平面节点复制到另一个控制平面节点：

   在以下示例中，用其他控制平面节点的 IP 地址替换 `CONTROL_PLANE_IPS`。

   ```sh
   USER=ubuntu # 可定制
   CONTROL_PLANE_IPS="10.0.0.7 10.0.0.8"
   for host in ${CONTROL_PLANE_IPS}; do
       scp /etc/kubernetes/pki/ca.crt "${USER}"@$host:
       scp /etc/kubernetes/pki/ca.key "${USER}"@$host:
       scp /etc/kubernetes/pki/sa.key "${USER}"@$host:
       scp /etc/kubernetes/pki/sa.pub "${USER}"@$host:
       scp /etc/kubernetes/pki/front-proxy-ca.crt "${USER}"@$host:
       scp /etc/kubernetes/pki/front-proxy-ca.key "${USER}"@$host:
       scp /etc/kubernetes/pki/etcd/ca.crt "${USER}"@$host:etcd-ca.crt
       scp /etc/kubernetes/pki/etcd/ca.key "${USER}"@$host:etcd-ca.key
   done
   ```

   **注意：**

   只需要复制上面列表中的证书。kubeadm 将负责生成其余证书以及加入控制平面实例所需的 SAN。 如果你错误地复制了所有证书，由于缺少所需的 SAN，创建其他节点可能会失败。

1. 然后，在每个即将加入集群的控制平面节点上，你必须先运行以下脚本，然后 再运行 `kubeadm join`。 该脚本会将先前复制的证书从主目录移动到 `/etc/kubernetes/pki`：

   ```shell
   USER=ubuntu # 可定制
   mkdir -p /etc/kubernetes/pki/etcd
   mv /home/${USER}/ca.crt /etc/kubernetes/pki/
   mv /home/${USER}/ca.key /etc/kubernetes/pki/
   mv /home/${USER}/sa.pub /etc/kubernetes/pki/
   mv /home/${USER}/sa.key /etc/kubernetes/pki/
   mv /home/${USER}/front-proxy-ca.crt /etc/kubernetes/pki/
   mv /home/${USER}/front-proxy-ca.key /etc/kubernetes/pki/
   mv /home/${USER}/etcd-ca.crt /etc/kubernetes/pki/etcd/ca.crt
   mv /home/${USER}/etcd-ca.key /etc/kubernetes/pki/etcd/ca.key
   ```

# pod清理

```sh
 podman system prune --all --volumes
WARNING! This will remove:
        - all stopped containers
        - all networks not used by at least one container
        - all volumes not used by at least one container
        - all images without at least one container associated to them
        - all build cache
```

