# k8s-ops
[k8s集群搭建](docker_k8s.md)

[yaml文件编写](yaml_concept.md)

[pod高级实战](pod_adv.md)

[kubectl管理k8s](./kubectl.md)
[k8s控制器 Replicaset 和 Deployment](./k8s_controllers_Replicaset_Deployment.md)
[存储类](./storages.md)
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
```

