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

namespace 是k8s集群基本的资源，可以给不同的用户、租户、环境、项目创建对应的命名空间。例如可以给development、qa、productioin应用环境分别创建自己的命名空间。

其他集群级别的资源node、pod、pv。他们不属于任何命名空间，因此同类资源名称必须唯一。

k8s默认提供了几个namespace 用于特定的目的。例如 kube-system主要用于运行系统级资源，而default 是为那些没有制定namespace的资源操作而提供的一个默认空间。

## 查看命名空间

```bash
$ kubectl get namespaces  或 kubectl get ns
NAME                   STATUS   AGE
default                Active   2d1h
kube-node-lease        Active   2d1h
kube-public            Active   2d1h
kube-system            Active   2d1h
kubernetes-dashboard   Active   2d1h
```

前四个是默认的命名空间。

查看ns 描述

```bash
$ kubectl describe ns default
Name:         default
Labels:       kubernetes.io/metadata.name=default
Annotations:  <none>
Status:       Active

No resource quota.

No LimitRange resource.
```

## 管理ns资源

创建ns

```bash
# cerberus @ cerberusdeMacBook-Pro in ~ [13:35:19]
$ kubectl create ns shaw
namespace/shaw created
(base)
```

准备实验数据

```bash
$ cat pod_ns_shaw.yaml
apiVersion: v1
kind: Pod
metadata:
  name: tomcat-pod
  namespace: shaw
  labels:
    tomcat:  tomcat-pod
spec:
  containers:
  - name:  tomcat-pod-java
    ports:
    - containerPort: 8080
    image: tomcat:8.5-jre8-alpine
    imagePullPolicy: IfNotPresent
  - name:  nginx
    ports:
    - containerPort: 80
    image: nginx
    imagePullPolicy: IfNotPresent  #本地有就用本地，如果没有就拉取官方镜像
    
$ kubectl apply -f pod_ns_shaw.yaml
pod/tomcat-pod unchanged
(base)
$ kubectl get pods  --show-labels -A
NAMESPACE              NAME                                         READY   STATUS    RESTARTS      AGE    LABELS
default                my-nginx-85b7d5dfb5-6dghq                    1/1     Running   0             16h    pod-template-hash=85b7d5dfb5
default                my-nginx-85b7d5dfb5-9jx22                    1/1     Running   0             15h    env=dev,pod-template-hash=85b7d5dfb5,run=my-nginx
default                my-nginx-85b7d5dfb5-pmhfg                    1/1     Running   0             16h    pod-template-hash=85b7d5dfb5
default                my-nginx-85b7d5dfb5-xkgkt                    1/1     Running   0             15h    env=dev,pod-template-hash=85b7d5dfb5,run=my-nginx
kube-system            calico-kube-controllers-67bb5696f5-dwfmf     1/1     Running   3 (43h ago)   2d2h   k8s-app=calico-kube-controllers,pod-template-hash=67bb5696f5
kube-system            calico-node-4fd8c                            1/1     Running   1 (43h ago)   2d2h   controller-revision-hash=8556fb6c94,k8s-app=calico-node,pod-template-generation=1
kube-system            calico-node-8xp57                            1/1     Running   1 (43h ago)   2d2h   controller-revision-hash=8556fb6c94,k8s-app=calico-node,pod-template-generation=1
kube-system            calico-node-xs9hf                            1/1     Running   1 (43h ago)   2d2h   controller-revision-hash=8556fb6c94,k8s-app=calico-node,pod-template-generation=1
kube-system            coredns-7d89d9b6b8-766ml                     1/1     Running   1 (43h ago)   2d2h   k8s-app=kube-dns,pod-template-hash=7d89d9b6b8
kube-system            coredns-7d89d9b6b8-qh4d8                     1/1     Running   1 (43h ago)   2d2h   k8s-app=kube-dns,pod-template-hash=7d89d9b6b8
kube-system            etcd-master                                  1/1     Running   2 (43h ago)   2d2h   component=etcd,tier=control-plane
kube-system            kube-apiserver-master                        1/1     Running   2 (43h ago)   2d2h   component=kube-apiserver,tier=control-plane
kube-system            kube-controller-manager-master               1/1     Running   3 (43h ago)   2d2h   component=kube-controller-manager,tier=control-plane
kube-system            kube-proxy-24np4                             1/1     Running   1 (43h ago)   2d2h   controller-revision-hash=754f9cdf79,k8s-app=kube-proxy,pod-template-generation=1
kube-system            kube-proxy-hvvzd                             1/1     Running   1 (43h ago)   2d2h   controller-revision-hash=754f9cdf79,k8s-app=kube-proxy,pod-template-generation=1
kube-system            kube-proxy-z6vc8                             1/1     Running   1 (43h ago)   2d2h   controller-revision-hash=754f9cdf79,k8s-app=kube-proxy,pod-template-generation=1
kube-system            kube-scheduler-master                        1/1     Running   3 (43h ago)   2d2h   component=kube-scheduler,tier=control-plane
kubernetes-dashboard   dashboard-metrics-scraper-778b77d469-x2w4j   1/1     Running   1 (43h ago)   2d1h   k8s-app=dashboard-metrics-scraper,pod-template-hash=778b77d469
kubernetes-dashboard   kubernetes-dashboard-86899d4bc7-5xftp        1/1     Running   2 (43h ago)   2d1h   k8s-app=kubernetes-dashboard,pod-template-hash=86899d4bc7
kubernetes-dashboard   kubernetes-dashboard-86899d4bc7-stcgg        1/1     Running   1 (43h ago)   2d1h   k8s-app=kubernetes-dashboard,pod-template-hash=86899d4bc7
shaw                   tomcat-pod                                   2/2     Running   0             88s    tomcat=tomcat-pod
```



## 删除资源

```bash
$ kubectl delete all -n shaw
# 删除ns为shaw下的所有资源
```





# pod 资源清单详细解读

 ```yaml
apiVersion: v1  #版本号
kind: Pod  # 资源类型
metadata:  #元数据
  annotations: #注释
    cni.projectcalico.org/podIP: 192.168.166.142/32
    cni.projectcalico.org/podIPs: 192.168.166.142/32
  creationTimestamp: "2021-12-08T13:42:03Z"
  generateName: my-nginx-85b7d5dfb5-  #名字
  labels:  # 标签
    pod-template-hash: 85b7d5dfb5  #标签1
  name: my-nginx-85b7d5dfb5-6dghq  #标签2
  namespace: default  # 命名空间
  resourceVersion: "177224" # 资源版本号
  uid: da61f2f5-f833-4636-aad2-b1be5b39fb8e
spec: #pod容器总的详细定义
  containers: # 容器列表
  - image: nginx  # 容器镜像
    imagePullPolicy: Always  # 容器拉取策略 [Always | Never | IfNotPresent] Alawys 表示下载镜像 IfnotPresent 表示优先使用本地镜像，否则下载镜像，Nerver 表示仅使用本地镜像
    name: my-nginx  # 容器名字 
    ports:  # 暴露的端口
    - containerPort: 80  
      protocol: TCP  #协议，tcp/udp都支持默认tcp
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount  # 挂载到容器内部的存储卷配置
      name: kube-api-access-8kxxh   # 引用pod定义的共享存储卷的名称,要和volumes定义的一致
      readOnly: true  # 是否为只读模式
  dnsPolicy: ClusterFirst  # dns 策略
  enableServiceLinks: true  #是否允许服务链接
  nodeName: node1   #运行容器的节点
  preemptionPolicy: PreemptLowerPriority 
  priority: 0
  restartPolicy: Always  # 重启策略
  schedulerName: default-scheduler 
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: kube-api-access-8kxxh
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2021-12-08T13:42:03Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2021-12-08T13:42:20Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2021-12-08T13:42:20Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2021-12-08T13:42:03Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: docker://6095d9710be9ad00aa9a481a770da3ae1d43236bd5ebd8a1a5085c814e1e1a2c
    image: nginx:latest
    imageID: docker-pullable://nginx@sha256:9522864dd661dcadfd9958f9e0de192a1fdda2c162a35668ab6ac42b465f0603
    lastState: {}
    name: my-nginx
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2021-12-08T13:42:20Z"
  hostIP: 192.168.247.34
  phase: Running
  podIP: 192.168.166.142
  podIPs:
  - ip: 192.168.166.142
  qosClass: BestEffort
  startTime: "2021-12-08T13:42:03Z"
 ```



# pod 高级用法： node节点选择器

## 指定node

```bash
[root@node1 ~]# docker load -i /opt/tomcat.tar.gz  
Loaded image: tomcat:8.5-jre8-alpine
[root@node1 ~]# docker images
REPOSITORY                                                       TAG               IMAGE ID       CREATED        SIZE
nginx                                                            latest            f652ca386ed1   6 days ago     141MB
registry.aliyuncs.com/google_containers/kube-proxy               v1.22.4           edeff87e4802   3 weeks ago    104MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy   v1.22.4           edeff87e4802   3 weeks ago    104MB
registry.aliyuncs.com/google_containers/coredns                  v1.8.4            8d147537fb7d   6 months ago   47.6MB
registry.cn-hangzhou.aliyuncs.com/google_containers/coredns      v1.8.4            8d147537fb7d   6 months ago   47.6MB
registry.aliyuncs.com/google_containers/pause                    3.5               ed210e3e4a5b   8 months ago   683kB
registry.cn-hangzhou.aliyuncs.com/google_containers/pause        3.5               ed210e3e4a5b   8 months ago   683kB
calico/pod2daemon-flexvol                                        v3.18.0           2a22066e9588   9 months ago   21.7MB
calico/node                                                      v3.18.0           5a7c4970fbc2   9 months ago   172MB
calico/cni                                                       v3.18.0           727de170e4ce   9 months ago   131MB
calico/kube-controllers                                          v3.18.0           9a154323fbf7   9 months ago   53.4MB
kubernetesui/dashboard                                           v2.0.0-beta8      eb51a3597525   2 years ago    90.8MB
kubernetesui/metrics-scraper                                     v1.0.1            709901356c11   2 years ago    40.1MB
tomcat                                                           8.5-jre8-alpine   8b8b1eb786b5   2 years ago    106MB
busybox                                                          1.28              8c811b4aec35   3 years ago    1.15MB
nginx                                                            1.9.1             94ec7e53edfc   6 years ago    133MB
```

重写一个pod的yaml,并且应用 vi node_pods.yaml

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
  namespace: default
  labels:
    app: myapp
    env: dev
spec:
  nodeName: node2  #指定运行的pod
  containers:
  - name:  tomcat-pod-java
    ports:
    - containerPort: 8080
    image: tomcat:8.5-jre8-alpine
    imagePullPolicy: IfNotPresent
  - name: busybox
    image: busybox:latest
    command:
    - "/bin/sh"
    - "-c"
    - "sleep 3600"


```

```bash
$ kubectl apply -f node_pods.yaml
pod/demo-pod created
(base)
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s-ops/yml on git:main x [15:15:33]
$ kubectl get pods  --show-labels -A -o wide

```

![image-20211209151710992](./pic/image-20211209151710992.png)

## node selector 

将指定的pod 调度到具有指定标签的节点上

首先给节点打标签

```bash
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s-ops/yml on git:main x [15:16:32]
$ kubectl label nodes node1 disk=ceph
node/node1 labeled
$ kubectl get nodes  --show-labels  # 查看是否打上了标签
NAME     STATUS   ROLES                  AGE    VERSION   LABELS
master   Ready    control-plane,master   2d3h   v1.22.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=,node.kubernetes.io/exclude-from-external-load-balancers=
node1    Ready    worker                 2d3h   v1.22.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disk=ceph,kubernetes.io/arch=amd64,kubernetes.io/hostname=node1,kubernetes.io/os=linux,node-role.kubernetes.io/worker=worker
node2    Ready    worker                 2d3h   v1.22.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node2,kubernetes.io/os=linux,node-role.kubernetes.io/worker=worker
(base)
```

新建yaml文件cat nodeselect_pods.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
  namespace: default
  labels:
    app: myapp
    env: dev
spec:
  nodeSelector: #节点选择器
   disk: ceph
  containers:
  - name:  tomcat-pod-java
    ports:
    - containerPort: 8080
    image: tomcat:8.5-jre8-alpine
    imagePullPolicy: IfNotPresent
  - name: busybox
    image: busybox:latest
    command:
    - "/bin/sh"
    - "-c"
    - "sleep 3600"
```

应用 查看结果

```bash
$ kubectl apply -f nodeselect_pods.yaml
pod/demo-pod created
(base)
$ kubectl get pods  --show-labels  -o wide
NAME                        READY   STATUS    RESTARTS   AGE    IP                NODE    NOMINATED NODE   READINESS GATES   LABELS
demo-pod                    2/2     Running   0          105s   192.168.166.145   node1   <none>           <none>            app=myapp,env=dev
my-nginx-85b7d5dfb5-6dghq   1/1     Running   0          17h    192.168.166.142   node1   <none>           <none>            pod-template-hash=85b7d5dfb5
my-nginx-85b7d5dfb5-9jx22   1/1     Running   0          16h    192.168.166.143   node1   <none>           <none>            env=dev,pod-template-hash=85b7d5dfb5,run=my-nginx
my-nginx-85b7d5dfb5-pmhfg   1/1     Running   0          17h    192.168.104.15    node2   <none>           <none>            pod-template-hash=85b7d5dfb5
my-nginx-85b7d5dfb5-xkgkt   1/1     Running   0          16h    192.168.104.16    node2   <none>           <none>            env=dev,pod-template-hash=85b7d5dfb5,run=my-nginx
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s-ops/yml on git:main x [15:27:57]
$ kubectl get pods  --show-labels  -o wide
NAME                        READY   STATUS    RESTARTS   AGE    IP                NODE    NOMINATED NODE   READINESS GATES   LABELS
demo-pod                    2/2     Running   0          105s   192.168.166.145   node1   <none>           <none>            app=myapp,env=dev
my-nginx-85b7d5dfb5-6dghq   1/1     Running   0          17h    192.168.166.142   node1   <none>           <none>            pod-template-hash=85b7d5dfb5
my-nginx-85b7d5dfb5-9jx22   1/1     Running   0          16h    192.168.166.143   node1   <none>           <none>            env=dev,pod-template-hash=85b7d5dfb5,run=my-nginx
my-nginx-85b7d5dfb5-pmhfg   1/1     Running   0          17h    192.168.104.15    node2   <none>           <none>            pod-template-hash=85b7d5dfb5
my-nginx-85b7d5dfb5-xkgkt   1/1     Running   0          16h    192.168.104.16    node2   <none>           <none>            env=dev,pod-template-hash=85b7d5dfb5,run=my-nginx
(base)
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s-ops/yml on git:main x [15:28:01]
$ kubectl get nodes  --show-labels  -o wide
NAME     STATUS   ROLES                  AGE    VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION               CONTAINER-RUNTIME   LABELS
master   Ready    control-plane,master   2d3h   v1.22.4   122.114.50.242   <none>        CentOS Linux 7 (Core)   3.10.0-693.21.1.el7.x86_64   docker://20.10.11   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=,node.kubernetes.io/exclude-from-external-load-balancers=
node1    Ready    worker                 2d3h   v1.22.4   192.168.247.34   <none>        CentOS Linux 7 (Core)   3.10.0-693.21.1.el7.x86_64   docker://20.10.11   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disk=ceph,kubernetes.io/arch=amd64,kubernetes.io/hostname=node1,kubernetes.io/os=linux,node-role.kubernetes.io/worker=worker
node2    Ready    worker                 2d3h   v1.22.4   192.168.0.110    <none>        CentOS Linux 7 (Core)   3.10.0-693.21.1.el7.x86_64   docker://20.10.11   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node2,kubernetes.io/os=linux,node-role.kubernetes.io/worker=worker
(base)
```



![image-20211209153006818](./pic/image-20211209153006818.png)

## 恢复初始状态

```bash
$ kubectl label nodes node1 disk-
node/node1 labeled
(base)
# cerberus @ cerberusdeMacBook-Pro in ~/Documents/k8s-ops/yml on git:main x [15:36:03]
$ kubectl get nodes  --show-labels  -o wide
NAME     STATUS   ROLES                  AGE    VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION               CONTAINER-RUNTIME   LABELS
master   Ready    control-plane,master   2d3h   v1.22.4   122.114.50.242   <none>        CentOS Linux 7 (Core)   3.10.0-693.21.1.el7.x86_64   docker://20.10.11   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=,node.kubernetes.io/exclude-from-external-load-balancers=
node1    Ready    worker                 2d3h   v1.22.4   192.168.247.34   <none>        CentOS Linux 7 (Core)   3.10.0-693.21.1.el7.x86_64   docker://20.10.11   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node1,kubernetes.io/os=linux,node-role.kubernetes.io/worker=worker
node2    Ready    worker                 2d3h   v1.22.4   192.168.0.110    <none>        CentOS Linux 7 (Core)   3.10.0-693.21.1.el7.x86_64   docker://20.10.11   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node2,kubernetes.io/os=linux,node-role.kubernetes.io/worker=worker
```



#  pod高级用法：污点和容忍度

# pod高级用法：pod状态和重启策略