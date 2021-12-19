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

## 常用命令

```bash
 kubectl  get pod --all-namespaces  # 获取所有的pod
 kubectl describe pod -n kube-system  coredns-7d89d9b6b8-5r6hn  #查看pod 描述
 kubectl get nodes   #获得nodes 
 kubectl get secret admin-token-4vgg2 -o jsonpath={.data.token} -n kube-system |base64 -  #查看密码
 kubectl  logs  -n kube-system coredns-7d89d9b6b8-p5hxq   #查看日志
 kubeadm token create --print-join-command  #查看加入集群的命令
 kubectl get deploy -A  # 获取所有的deploy
 kubectl  delete deploy kubernetes-dashboard -n kube-system # 删除特定的deploy
 kubectl delete pod  pod名字   --force --grace-period=0  # 强制删除pod
 kubectl  rollout history deploy  #查看历史发布版本
 kubectl rollout undo deployment appv1 --to-revision=1 # 回滚到版本1
 kubectl create sa monitor -n monitor-sa  # 创建sa账号
 kubectl create clusterrolebinding monitor-clusterrolebinding -n monitor-sa --clusterrole=cluster-admin --serviceaccount=monitor-sa:monitor # 绑定sa账号到cluster-admin这个集群角色
 kubectl create clusterrolebinding monitor-user-clusterrolebinding  --clusterrole=cluster-admin --user=system:serviceaccount:monitor-sa # 对用户进行授权
```

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

## jenkins 

1. 初始化 插件连接超时

   1. 在启动以后,输入密码,然后界面在缓冲 / 提示实例离线的时候, 打开`/pluginManager/advanced`

      在最下面设置`https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json`,然后提交,然后chenk now一下
      

