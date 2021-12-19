# jenkins 配置k8s互通以及pod 模版

![image-20211219203856678](./pic/image-20211219203856678.png)

![image-20211219204820465](./pic/image-20211219204820465.png)

![image-20211219204910336](./pic/image-20211219204910336.png)

![image-20211219205056671](./pic/image-20211219205056671.png)

- jenkins slave 需要调用宿主机的docker 打镜像 所以需要共享宿主机的docker 命令等

- slave需要和k8s集群通信，kubectl apply yaml 文件，所以需要k8s集群的认证信息

![image-20211219205623058](/Users/luca/Documents/文稿/k8s-ops/pic/image-20211219205623058.png)

# 添加docker hub 凭据

[系统管理]---->[管理凭据]

![image-20211219210310386](/Users/luca/Documents/文稿/k8s-ops/pic/image-20211219210310386.png)

# 添加harbor