# label 标签使用技巧

在k8s上每一种资源都可以有标签。

现实使用中pod数量越来越多，我们期望能够进行分类管理，最简单和直接的效果就是把pod分成很多不同的小组。

无论对开发还是运维都能提高管理效率，况且我们的控制器、service资源需要标签来识别他们所管理域或 关联的资源。

**当我们给pod或者任何资源设定标签的时候，都可以使用标签查看，删除等对其执行相应的管理操作。**

简单来说标签就是附加在对应资源上的**键值对**，一个资源可以存在多个标签。

标签：

​	key:value (key value最多63个字符，key只能字符、数字、下划线组成。key只能以字母开头结尾，不能为空；value也是只能以字母数字开头和结尾，可以为空)

## *查看所有命名空间pod 的标签*

```bash
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s_gitlab/yml on git:main x [21:42:06]
$ kubectl get pods --show-labels -A
NAMESPACE              NAME                                         READY   STATUS              RESTARTS      AGE   LABELS
default                my-nginx-5b56ccd65f-6g82m                    1/1     Running             0             96s   pod-template-hash=5b56ccd65f,run=my-nginx
default                my-nginx-5b56ccd65f-rj2w7                    1/1     Running             0             96s   pod-template-hash=5b56ccd65f,run=my-nginx
default                my-nginx-85b7d5dfb5-6dghq                    0/1     ContainerCreating   0             14s   env=dev,pod-template-hash=85b7d5dfb5,run=my-nginx
kube-system            calico-kube-controllers-67bb5696f5-dwfmf     1/1     Running             3 (27h ago)   34h   k8s-app=calico-kube-controllers,pod-template-hash=67bb5696f5
kube-system            calico-node-4fd8c                            1/1     Running             1 (27h ago)   34h   controller-revision-hash=8556fb6c94,k8s-app=calico-node,pod-template-generation=1
kube-system            calico-node-8xp57                            1/1     Running             1 (27h ago)   34h   controller-revision-hash=8556fb6c94,k8s-app=calico-node,pod-template-generation=1
kube-system            calico-node-xs9hf                            1/1     Running             1 (27h ago)   34h   controller-revision-hash=8556fb6c94,k8s-app=calico-node,pod-template-generation=1
kube-system            coredns-7d89d9b6b8-766ml                     1/1     Running             1 (27h ago)   34h   k8s-app=kube-dns,pod-template-hash=7d89d9b6b8
kube-system            coredns-7d89d9b6b8-qh4d8                     1/1     Running             1 (27h ago)   34h   k8s-app=kube-dns,pod-template-hash=7d89d9b6b8
kube-system            etcd-master                                  1/1     Running             2 (27h ago)   34h   component=etcd,tier=control-plane
kube-system            kube-apiserver-master                        1/1     Running             2 (27h ago)   34h   component=kube-apiserver,tier=control-plane
kube-system            kube-controller-manager-master               1/1     Running             3 (27h ago)   34h   component=kube-controller-manager,tier=control-plane
kube-system            kube-proxy-24np4                             1/1     Running             1 (27h ago)   34h   controller-revision-hash=754f9cdf79,k8s-app=kube-proxy,pod-template-generation=1
kube-system            kube-proxy-hvvzd                             1/1     Running             1 (27h ago)   34h   controller-revision-hash=754f9cdf79,k8s-app=kube-proxy,pod-template-generation=1
kube-system            kube-proxy-z6vc8                             1/1     Running             1 (27h ago)   34h   controller-revision-hash=754f9cdf79,k8s-app=kube-proxy,pod-template-generation=1
kube-system            kube-scheduler-master                        1/1     Running             3 (27h ago)   34h   component=kube-scheduler,tier=control-plane
kubernetes-dashboard   dashboard-metrics-scraper-778b77d469-x2w4j   1/1     Running             1 (27h ago)   33h   k8s-app=dashboard-metrics-scraper,pod-template-hash=778b77d469
kubernetes-dashboard   kubernetes-dashboard-86899d4bc7-5xftp        1/1     Running             2 (27h ago)   33h   k8s-app=kubernetes-dashboard,pod-template-hash=86899d4bc7
kubernetes-dashboard   kubernetes-dashboard-86899d4bc7-stcgg        1/1     Running             1 (27h ago)   33h   k8s-app=kubernetes-dashboard,pod-template-hash=86899d4bc7
```

## *查看所有标签下拥有run这个标签的标签值* **不常用**

```bash
$   kubectl get pods -L run -A
NAMESPACE              NAME                                         READY   STATUS    RESTARTS      AGE     RUN
default                my-nginx-85b7d5dfb5-6dghq                    1/1     Running   0             9m20s   my-nginx
default                my-nginx-85b7d5dfb5-pmhfg                    1/1     Running   0             9m3s    my-nginx
kube-system            calico-kube-controllers-67bb5696f5-dwfmf     1/1     Running   3 (27h ago)   34h
kube-system            calico-node-4fd8c                            1/1     Running   1 (27h ago)   34h
kube-system            calico-node-8xp57                            1/1     Running   1 (27h ago)   34h
kube-system            calico-node-xs9hf                            1/1     Running   1 (27h ago)   34h
kube-system            coredns-7d89d9b6b8-766ml                     1/1     Running   1 (27h ago)   34h
kube-system            coredns-7d89d9b6b8-qh4d8                     1/1     Running   1 (27h ago)   34h
kube-system            etcd-master                                  1/1     Running   2 (27h ago)   34h
kube-system            kube-apiserver-master                        1/1     Running   2 (27h ago)   34h
kube-system            kube-controller-manager-master               1/1     Running   3 (27h ago)   34h
kube-system            kube-proxy-24np4                             1/1     Running   1 (27h ago)   34h
kube-system            kube-proxy-hvvzd                             1/1     Running   1 (27h ago)   34h
kube-system            kube-proxy-z6vc8                             1/1     Running   1 (27h ago)   34h
kube-system            kube-scheduler-master                        1/1     Running   3 (27h ago)   34h
kubernetes-dashboard   dashboard-metrics-scraper-778b77d469-x2w4j   1/1     Running   1 (27h ago)   33h
kubernetes-dashboard   kubernetes-dashboard-86899d4bc7-5xftp        1/1     Running   2 (27h ago)   33h
kubernetes-dashboard   kubernetes-dashboard-86899d4bc7-stcgg        1/1     Running   1 (27h ago)   33h
```

## *查看具有run标签的资源对象*

```bash
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s_gitlab/yml on git:main x [21:45:29] C:1
$ kubectl get pods -l run -A
NAMESPACE   NAME                        READY   STATUS    RESTARTS   AGE
default     my-nginx-85b7d5dfb5-6dghq   1/1     Running   0          3m29s
default     my-nginx-85b7d5dfb5-pmhfg   1/1     Running   0          3m12s
```

## *查看所有具有k8s标签的资源并且打印相关标签*

```bash
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s_gitlab/yml on git:main x [21:56:49]
$ kubectl get pods -l k8s-app -A  --show-labels
NAMESPACE              NAME                                         READY   STATUS    RESTARTS      AGE   LABELS
kube-system            calico-kube-controllers-67bb5696f5-dwfmf     1/1     Running   3 (27h ago)   34h   k8s-app=calico-kube-controllers,pod-template-hash=67bb5696f5
kube-system            calico-node-4fd8c                            1/1     Running   1 (27h ago)   34h   controller-revision-hash=8556fb6c94,k8s-app=calico-node,pod-template-generation=1
kube-system            calico-node-8xp57                            1/1     Running   1 (27h ago)   34h   controller-revision-hash=8556fb6c94,k8s-app=calico-node,pod-template-generation=1
kube-system            calico-node-xs9hf                            1/1     Running   1 (27h ago)   34h   controller-revision-hash=8556fb6c94,k8s-app=calico-node,pod-template-generation=1
kube-system            coredns-7d89d9b6b8-766ml                     1/1     Running   1 (27h ago)   34h   k8s-app=kube-dns,pod-template-hash=7d89d9b6b8
kube-system            coredns-7d89d9b6b8-qh4d8                     1/1     Running   1 (27h ago)   34h   k8s-app=kube-dns,pod-template-hash=7d89d9b6b8
kube-system            kube-proxy-24np4                             1/1     Running   1 (27h ago)   34h   controller-revision-hash=754f9cdf79,k8s-app=kube-proxy,pod-template-generation=1
kube-system            kube-proxy-hvvzd                             1/1     Running   1 (27h ago)   34h   controller-revision-hash=754f9cdf79,k8s-app=kube-proxy,pod-template-generation=1
kube-system            kube-proxy-z6vc8                             1/1     Running   1 (27h ago)   34h   controller-revision-hash=754f9cdf79,k8s-app=kube-proxy,pod-template-generation=1
kubernetes-dashboard   dashboard-metrics-scraper-778b77d469-x2w4j   1/1     Running   1 (27h ago)   33h   k8s-app=dashboard-metrics-scraper,pod-template-hash=778b77d469
kubernetes-dashboard   kubernetes-dashboard-86899d4bc7-5xftp        1/1     Running   2 (27h ago)   33h   k8s-app=kubernetes-dashboard,pod-template-hash=86899d4bc7
kubernetes-dashboard   kubernetes-dashboard-86899d4bc7-stcgg        1/1     Running   1 (27h ago)   33h   k8s-app=kubernetes-dashboard,pod-template-hash=86899d4bc7
```



## *给运行的pod打标签:* 

给所有run 标签的pod 打上k8sapp=nginx的标签 强制更改可以使用--overwrite

```bash
$ kubectl get pods -l run -A  --show-labels
NAMESPACE   NAME                        READY   STATUS    RESTARTS   AGE   LABELS
default     my-nginx-85b7d5dfb5-6dghq   1/1     Running   0          19m   env=dev,pod-template-hash=85b7d5dfb5,run=my-nginx
default     my-nginx-85b7d5dfb5-pmhfg   1/1     Running   0          18m   env=dev,pod-template-hash=85b7d5dfb5,run=my-nginx
(base)
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s_gitlab/yml on git:main x [22:01:06]
$ kubectl label pod  my-nginx-85b7d5dfb5-6dghq  k8s-app=nginx 
pod/my-nginx-85b7d5dfb5-6dghq labeled
(base)
$ kubectl label pod  my-nginx-85b7d5dfb5-pmhfg  k8s-app=abc

# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s_gitlab/yml on git:main x [22:09:31]
$ kubectl label pod  my-nginx-85b7d5dfb5-pmhfg  k8s-app=nginx
error: 'k8s-app' already has a value (abc), and --overwrite is false
(base)
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s_gitlab/yml on git:main x [22:09:41] C:1
$ kubectl label pod  my-nginx-85b7d5dfb5-pmhfg  k8s-app=nginx --overwrite
pod/my-nginx-85b7d5dfb5-pmhfg labeled
(base)
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s_gitlab/yml on git:main x [22:02:56]
$ kubectl get pods -l k8s-app -A  --show-labels
NAMESPACE              NAME                                         READY   STATUS    RESTARTS      AGE   LABELS
default                my-nginx-85b7d5dfb5-6dghq                    1/1     Running   0             20m   env=dev,k8s-app=nginx,pod-template-hash=85b7d5dfb5,run=my-nginx
default                my-nginx-85b7d5dfb5-pmhfg                    1/1     Running   0             20m   env=dev,k8s-app=nginx,pod-template-hash=85b7d5dfb5,run=my-nginx
kube-system            calico-kube-controllers-67bb5696f5-dwfmf     1/1     Running   3 (27h ago)   34h   k8s-app=calico-kube-controllers,pod-template-hash=67bb5696f5
kube-system            calico-node-4fd8c                            1/1     Running   1 (27h ago)   34h   controller-revision-hash=8556fb6c94,k8s-app=calico-node,pod-template-generation=1
kube-system            calico-node-8xp57                            1/1     Running   1 (27h ago)   34h   controller-revision-hash=8556fb6c94,k8s-app=calico-node,pod-template-generation=1
kube-system            calico-node-xs9hf                            1/1     Running   1 (27h ago)   34h   controller-revision-hash=8556fb6c94,k8s-app=calico-node,pod-template-generation=1
kube-system            coredns-7d89d9b6b8-766ml                     1/1     Running   1 (27h ago)   34h   k8s-app=kube-dns,pod-template-hash=7d89d9b6b8
kube-system            coredns-7d89d9b6b8-qh4d8                     1/1     Running   1 (27h ago)   34h   k8s-app=kube-dns,pod-template-hash=7d89d9b6b8
kube-system            kube-proxy-24np4                             1/1     Running   1 (27h ago)   34h   controller-revision-hash=754f9cdf79,k8s-app=kube-proxy,pod-template-generation=1
kube-system            kube-proxy-hvvzd                             1/1     Running   1 (27h ago)   34h   controller-revision-hash=754f9cdf79,k8s-app=kube-proxy,pod-template-generation=1
kube-system            kube-proxy-z6vc8                             1/1     Running   1 (27h ago)   34h   controller-revision-hash=754f9cdf79,k8s-app=kube-proxy,pod-template-generation=1
kubernetes-dashboard   dashboard-metrics-scraper-778b77d469-x2w4j   1/1     Running   1 (27h ago)   33h   k8s-app=dashboard-metrics-scraper,pod-template-hash=778b77d469
kubernetes-dashboard   kubernetes-dashboard-86899d4bc7-5xftp        1/1     Running   2 (27h ago)   33h   k8s-app=kubernetes-dashboard,pod-template-hash=86899d4bc7
kubernetes-dashboard   kubernetes-dashboard-86899d4bc7-stcgg        1/1     Running   1 (27h ago)   33h   k8s-app=kubernetes-dashboard,pod-template-hash=86899d4bc7
```

## *查看所有命名空间下既有k8s-app 又有run的 标签*

```bash
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s_gitlab/yml on git:main x [22:03:02]
$ kubectl get pods -l k8s-app,run -A  --show-labels
NAMESPACE   NAME                        READY   STATUS    RESTARTS   AGE   LABELS
default     my-nginx-85b7d5dfb5-6dghq   1/1     Running   0          25m   env=dev,k8s-app=nginx,pod-template-hash=85b7d5dfb5,run=my-nginx
default     my-nginx-85b7d5dfb5-pmhfg   1/1     Running   0          25m   env=dev,k8s-app=nginx,pod-template-hash=85b7d5dfb5,run=my-nginx
```

## *给node节点打标签*

查看node节点的标签

```bash
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s_gitlab/yml on git:main x [22:12:51]
$ kubectl get nodes --show-labels
NAME     STATUS   ROLES                  AGE   VERSION   LABELS
master   Ready    control-plane,master   34h   v1.22.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=,node.kubernetes.io/exclude-from-external-load-balancers=
node1    Ready    worker                 34h   v1.22.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node1,kubernetes.io/os=linux,node-role.kubernetes.io/worker=worker
node2    Ready    worker                 34h   v1.22.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node2,kubernetes.io/os=linux,node-role.kubernetes.io/worker=worker
(base)
```

打标签

```bash
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s_gitlab/yml on git:main x [22:12:51]
$ kubectl get nodes --show-labels
NAME     STATUS   ROLES                  AGE   VERSION   LABELS
master   Ready    control-plane,master   34h   v1.22.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=,node.kubernetes.io/exclude-from-external-load-balancers=
node1    Ready    worker                 34h   v1.22.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node1,kubernetes.io/os=linux,node-role.kubernetes.io/worker=worker
node2    Ready    worker                 34h   v1.22.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node2,kubernetes.io/os=linux,node-role.kubernetes.io/worker=worker
(base)
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s_gitlab/yml on git:main x [22:13:24]
$ kubectl label nodes node1 useful=1
node/node1 labeled
(base)
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s_gitlab/yml on git:main x [22:16:15]
$ kubectl get nodes --show-labels
NAME     STATUS   ROLES                  AGE   VERSION   LABELS
master   Ready    control-plane,master   34h   v1.22.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=,node.kubernetes.io/exclude-from-external-load-balancers=
node1    Ready    worker                 34h   v1.22.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node1,kubernetes.io/os=linux,node-role.kubernetes.io/worker=worker,useful=1
node2    Ready    worker                 34h   v1.22.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node2,kubernetes.io/os=linux,node-role.kubernetes.io/worker=worker
(base)
```

## 删除标签 

kubectl label node  [节点名]  [键值对中的键]+[-]

```bash
$ kubectl get nodes -l useful --show-labels
NAME    STATUS   ROLES    AGE   VERSION   LABELS
node1   Ready    worker   34h   v1.22.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node1,kubernetes.io/os=linux,node-role.kubernetes.io/worker=worker,useful=1
(base)
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s_gitlab/yml on git:main x [22:25:23]
$ kubectl label nodes node1 useful-
node/node1 labeled
(base)
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s_gitlab/yml on git:main x [22:25:43]
$ kubectl get nodes -l useful --show-labels
No resources found
```

pod 同理

```bash
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s_gitlab/yml on git:main x [22:27:01] C:1
$ kubectl get pods  --show-labels -A
NAMESPACE              NAME                                         READY   STATUS    RESTARTS      AGE   LABELS
default                my-nginx-85b7d5dfb5-6dghq                    1/1     Running   0             45m   env=dev,k8s-app=nginx,pod-template-hash=85b7d5dfb5,run=my-nginx
default                my-nginx-85b7d5dfb5-pmhfg                    1/1     Running   0             44m   env=dev,k8s-app=nginx,pod-template-hash=85b7d5dfb5,run=my-nginx
kube-system            calico-kube-controllers-67bb5696f5-dwfmf     1/1     Running   3 (27h ago)   34h   k8s-app=calico-kube-controllers,pod-template-hash=67bb5696f5
kube-system            calico-node-4fd8c                            1/1     Running   1 (27h ago)   34h   controller-revision-hash=8556fb6c94,k8s-app=calico-node,pod-template-generation=1
kube-system            calico-node-8xp57                            1/1     Running   1 (27h ago)   34h   controller-revision-hash=8556fb6c94,k8s-app=calico-node,pod-template-generation=1
kube-system            calico-node-xs9hf                            1/1     Running   1 (27h ago)   34h   controller-revision-hash=8556fb6c94,k8s-app=calico-node,pod-template-generation=1
kube-system            coredns-7d89d9b6b8-766ml                     1/1     Running   1 (27h ago)   34h   k8s-app=kube-dns,pod-template-hash=7d89d9b6b8
kube-system            coredns-7d89d9b6b8-qh4d8                     1/1     Running   1 (27h ago)   34h   k8s-app=kube-dns,pod-template-hash=7d89d9b6b8
kube-system            etcd-master                                  1/1     Running   2 (27h ago)   34h   component=etcd,tier=control-plane
kube-system            kube-apiserver-master                        1/1     Running   2 (27h ago)   34h   component=kube-apiserver,tier=control-plane
kube-system            kube-controller-manager-master               1/1     Running   3 (27h ago)   34h   component=kube-controller-manager,tier=control-plane
kube-system            kube-proxy-24np4                             1/1     Running   1 (27h ago)   34h   controller-revision-hash=754f9cdf79,k8s-app=kube-proxy,pod-template-generation=1
kube-system            kube-proxy-hvvzd                             1/1     Running   1 (27h ago)   34h   controller-revision-hash=754f9cdf79,k8s-app=kube-proxy,pod-template-generation=1
kube-system            kube-proxy-z6vc8                             1/1     Running   1 (27h ago)   34h   controller-revision-hash=754f9cdf79,k8s-app=kube-proxy,pod-template-generation=1
kube-system            kube-scheduler-master                        1/1     Running   3 (27h ago)   34h   component=kube-scheduler,tier=control-plane
kubernetes-dashboard   dashboard-metrics-scraper-778b77d469-x2w4j   1/1     Running   1 (27h ago)   34h   k8s-app=dashboard-metrics-scraper,pod-template-hash=778b77d469
kubernetes-dashboard   kubernetes-dashboard-86899d4bc7-5xftp        1/1     Running   2 (27h ago)   34h   k8s-app=kubernetes-dashboard,pod-template-hash=86899d4bc7
kubernetes-dashboard   kubernetes-dashboard-86899d4bc7-stcgg        1/1     Running   1 (27h ago)   34h   k8s-app=kubernetes-dashboard,pod-template-hash=86899d4bc7
(base)
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s_gitlab/yml on git:main x [22:27:04]
$ kubectl label pod  my-nginx-85b7d5dfb5-pmhfg  k8s-app-
pod/my-nginx-85b7d5dfb5-pmhfg labeled
(base)
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s_gitlab/yml on git:main x [22:27:38]
$ kubectl label pod  my-nginx-85b7d5dfb5-6dghq  k8s-app- env-
pod/my-nginx-85b7d5dfb5-6dghq labeled
(base)
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s_gitlab/yml on git:main x [22:28:09]
$ kubectl label pod  my-nginx-85b7d5dfb5-pmhfg  k8s-app- env-
label "k8s-app" not found.
pod/my-nginx-85b7d5dfb5-pmhfg labeled
(base)
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s_gitlab/yml on git:main x [22:28:28]
$ kubectl get pods  --show-labels -A
NAMESPACE              NAME                                         READY   STATUS    RESTARTS      AGE   LABELS
default                my-nginx-85b7d5dfb5-6dghq                    1/1     Running   0             46m   pod-template-hash=85b7d5dfb5,run=my-nginx
default                my-nginx-85b7d5dfb5-pmhfg                    1/1     Running   0             46m   pod-template-hash=85b7d5dfb5,run=my-nginx
kube-system            calico-kube-controllers-67bb5696f5-dwfmf     1/1     Running   3 (27h ago)   34h   k8s-app=calico-kube-controllers,pod-template-hash=67bb5696f5
kube-system            calico-node-4fd8c                            1/1     Running   1 (27h ago)   34h   controller-revision-hash=8556fb6c94,k8s-app=calico-node,pod-template-generation=1
kube-system            calico-node-8xp57                            1/1     Running   1 (27h ago)   34h   controller-revision-hash=8556fb6c94,k8s-app=calico-node,pod-template-generation=1
kube-system            calico-node-xs9hf                            1/1     Running   1 (27h ago)   34h   controller-revision-hash=8556fb6c94,k8s-app=calico-node,pod-template-generation=1
kube-system            coredns-7d89d9b6b8-766ml                     1/1     Running   1 (27h ago)   34h   k8s-app=kube-dns,pod-template-hash=7d89d9b6b8
kube-system            coredns-7d89d9b6b8-qh4d8                     1/1     Running   1 (27h ago)   34h   k8s-app=kube-dns,pod-template-hash=7d89d9b6b8
kube-system            etcd-master                                  1/1     Running   2 (27h ago)   34h   component=etcd,tier=control-plane
kube-system            kube-apiserver-master                        1/1     Running   2 (27h ago)   34h   component=kube-apiserver,tier=control-plane
kube-system            kube-controller-manager-master               1/1     Running   3 (27h ago)   34h   component=kube-controller-manager,tier=control-plane
kube-system            kube-proxy-24np4                             1/1     Running   1 (27h ago)   34h   controller-revision-hash=754f9cdf79,k8s-app=kube-proxy,pod-template-generation=1
kube-system            kube-proxy-hvvzd                             1/1     Running   1 (27h ago)   34h   controller-revision-hash=754f9cdf79,k8s-app=kube-proxy,pod-template-generation=1
kube-system            kube-proxy-z6vc8                             1/1     Running   1 (27h ago)   34h   controller-revision-hash=754f9cdf79,k8s-app=kube-proxy,pod-template-generation=1
kube-system            kube-scheduler-master                        1/1     Running   3 (27h ago)   34h   component=kube-scheduler,tier=control-plane
kubernetes-dashboard   dashboard-metrics-scraper-778b77d469-x2w4j   1/1     Running   1 (27h ago)   34h   k8s-app=dashboard-metrics-scraper,pod-template-hash=778b77d469
kubernetes-dashboard   kubernetes-dashboard-86899d4bc7-5xftp        1/1     Running   2 (27h ago)   34h   k8s-app=kubernetes-dashboard,pod-template-hash=86899d4bc7
kubernetes-dashboard   kubernetes-dashboard-86899d4bc7-stcgg        1/1     Running   1 (27h ago)   34h   k8s-app=kubernetes-dashboard,pod-template-hash=86899d4bc7
```



# 名称空间

# pod 资源清单详细解读

# pod 高级用法： node节点选择器

# pod高级用法：污点和容忍度

# pod高级用法：pod状态和重启策略