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

### 安装



您将在下方找到有关我们旨在支持并与本网站上的文档兼容的各种场景的详细信息。此外，下面针对每种情况列出了最适用的安装方法。

#### 默认静态安装

> 您不需要对 cert-manager 安装参数进行任何调整。

默认静态配置可以安装如下：

```bash
$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.yaml
```

[可以在此处找到](https://cert-manager.io/docs/installation/kubectl/)有关此安装方法的更多信息。

### 入门

> 您很快就想了解如何使用 cert-manager 以及它的用途。

我们建议使用[cmctl x install](https://cert-manager.io/docs/installation/cmctl/)来快速安装 cert-manager 并从命令行[与 cert-manager 资源进行交互](https://cert-manager.io/docs/usage/cmctl/)。

或者，如果您更喜欢 Helm 或者不想安装`cmctl`，则可以[使用 helm 安装 cert-manager](https://cert-manager.io/docs/installation/helm/)。

如果您在 OpenShift 集群上运行，请考虑通过[OperatorHub.io](https://cert-manager.io/docs/installation/operator-lifecycle-manager/)上的[cert-manager 进行](https://cert-manager.io/docs/installation/operator-lifecycle-manager/)安装。

### 持续部署

> 您知道如何配置您的证书管理器设置并希望自动执行此操作。

您可以使用`helm template`或`cmctl x install --dry-run`来生成自定义的 cert-manager 安装清单。有关更多详细信息，请参阅[使用 cmctl x install](https://cert-manager.io/docs/installation/cmctl/#output-yaml)[输出 YAML](https://cert-manager.io/docs/installation/helm/#output-yaml)和[使用 helm 模板输出 YAML](https://cert-manager.io/docs/installation/helm/#output-yaml)。这个模板化的证书管理器清单可以通过管道传输到您首选的部署工具中。

如果您使用 Helm 进行自动化，cert-manager[支持使用 Helm 进行安装](https://cert-manager.io/docs/installation/helm/)。

### 安装cmctl

您需要`cmctl.tar.gz`所用平台的文件，可以在我们的[GitHub 发布页面 上](https://github.com/jetstack/cert-manager/releases)找到这些文件 。为了使用`cmctl`你需要它的二进制文件可以`cmctl`在你的`$PATH`. 运行以下命令来设置 CLI。用您的系统等效物替换 OS 和 ARCH：

```console
OS=$(go env GOOS); ARCH=$(go env GOARCH); curl -L -o cmctl.tar.gz https://github.com/jetstack/cert-manager/releases/latest/download/cmctl-$OS-$ARCH.tar.gz
tar xzf cmctl.tar.gz
sudo mv cmctl /usr/local/bin
```

您可以运行`cmctl help`以测试 CLI 是否设置正确：

```console
$ cmctl help

cmctl is a CLI tool manage and configure cert-manager resources for Kubernetes

Usage: cmctl [command]

Available Commands:
  approve      Approve a CertificateRequest
  check        Check cert-manager components
  completion   Generate completion scripts for the cert-manager CLI
  convert      Convert cert-manager config files between different API versions
  create       Create cert-manager resources
  deny         Deny a CertificateRequest
  experimental Interact with experimental features
  help         Help about any command
  inspect      Get details on certificate related resources
  renew        Mark a Certificate for manual renewal
  status       Get details on current status of cert-manager resources
  version      Print the cert-manager CLI version and the deployed cert-manager version

Flags:
  -h, --help                           help for cmctl
      --log-flush-frequency duration   Maximum number of seconds between log flushes (default 5s)

Use "cmctl [command] --help" for more information about a command.
```

### 命令

### 批准和拒绝证书请求

可以 使用各自的 cmctl 命令[批准或拒绝](https://cert-manager.io/docs/concepts/certificaterequest/#approval)CertificateRequests：

> **注意**：除非使用 cert-manager-controller 上的标志禁用，否则内部 cert-manager 批准者可能会自动批准所有 CertificateRequests `--controllers=*,-certificaterequests-approver`

```bash
$ cmctl approve -n istio-system mesh-ca --reason "pki-team" --message "this certificate is valid"
Approved CertificateRequest 'istio-system/mesh-ca'
$ cmctl deny -n my-app my-app --reason "example.com" --message "violates policy"
Denied CertificateRequest 'my-app/my-app'
```

### 兑换

`cmctl convert`可用于在不同 API 版本之间转换 cert-manager 清单文件。接受 YAML 和 JSON 格式。该命令将文件名、目录路径或 URL 作为输入。内容转换为cert-manager已知的最新API版本格式，或者`--output-version`flag指定的格式。

默认输出将以 YAML 格式打印到标准输出。可以使用该选项`-o`来更改输出目的地。

例如，这将输出`cert.yaml`最新的 API 版本：

```console
cmctl convert -f cert.yaml
```

### 创建

`cmctl create`可用于手动创建证书管理器资源。子命令可用于创建不同的资源：

#### 证书请求

要创建证书管理器 CertificateRequest，请使用`cmctl create certificaterequest`. 该命令采用要创建的 CertificateRequest 的名称，并根据`--from-certificate-file` flag指定的证书资源的 YAML 清单，通过在本地生成私钥并创建要提交的“证书签名请求”来创建新的 CertificateRequest 资源到证书管理器颁发者。私钥将写入本地文件，默认为`<name_of_cr>.key`，或者可以使用`--output-key-file`标志指定。

如果您希望等待 CertificateRequest 签名并将 X.509 证书存储在文件中，您可以设置该`--fetch-certificate`标志。等待颁发证书时的默认超时时间为 5 分钟，但可以使用`--timeout`标志指定。存放X.509证书的文件的默认名称是`<name_of_cr>.crt`，您可以使用该`--output-certificate-file`标志来指定。

请注意，私钥和 X.509 证书都写入文件，**并未**存储在 Kubernetes 中。

例如，这将创建一个名为“my-cr”的 CertificateRequest 资源，该资源基于 中描述的 cert-manager 证书，`my-certificate.yaml`同时分别将私钥和 X.509 证书存储在`my-cr.key`和 中`my-cr.crt` 。

```console
cmctl create certificaterequest my-cr --from-certificate-file my-certificate.yaml --fetch-certificate --timeout 20m
```

### 更新

`cmctl`允许您手动触发特定证书的续订。这可以使用标签选择器 ( `-l app=example`) 或使用`--all`标志一次完成一个证书：

例如，您可以更新证书`example-com-tls`：

```console
$ kubectl get certificate
NAME                       READY   SECRET               AGE
example-com-tls            True    example-com-tls      1d

$ cmctl renew example-com-tls
Manually triggered issuance of Certificate default/example-com-tls

$ kubectl get certificaterequest
NAME                              READY   AGE
example-com-tls-tls-8rbv2         False    10s
```

您还可以更新给定命名空间中的所有证书：

```console
$ cmctl renew --namespace=app --all
```

更新命令允许指定几个选项：

- `--all` 更新给定命名空间中的所有证书，或与 `--all-namespaces`
- `-A`或 `--all-namespaces`跨命名空间标记证书以进行续订
- `-l` `--selector`允许设置标签查询以过滤以及`kubectl`像`--context`和这样的全局标志`--namespace`。

### 身份证明

`cmctl status certificate`如果是 ACME 证书，则输出证书资源和相关资源（如 CertificateRequest、Secret、Issuer 以及 Order 和 Challenges）的当前状态的详细信息。该命令输出有关资源的信息，包括条件、事件和资源特定字段，例如密钥用法和密钥的扩展密钥用法或订单的授权。这将有助于对证书进行故障排除。

该命令接受一个参数，指定证书资源的名称，并且可以像往常一样使用`-n`或 `--namespace`标志指定命名空间。

本示例查询命名`my-certificate`空间中命名的证书的状态`my-namespace`。

```console
cmctl status certificate my-certificate -n my-namespace
```

### 完成

`cmctl` 支持两个子命令的自动完成以及运行时对象的建议。

```console
$ cmctl approve -n <TAB> <TAB>
default             kube-node-lease     kube-public         kube-system         local-path-storage
```

可以按照您正在使用的 shell 的说明为您的环境安装 Completion。它目前支持 bash、fish、zsh 和 powershell。

```console
$ cmctl completion help
```

------

## jenkins 

1. 初始化 插件连接超时

   1. 在启动以后,输入密码,然后界面在缓冲 / 提示实例离线的时候, 打开`/pluginManager/advanced`

      在最下面设置`https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json`,然后提交,然后chenk now一下
      

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