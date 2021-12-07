# [pod概念](https://kubernetes.io/zh/docs/concepts/workloads/pods/)

​		![pod概念](./pic/pod.svg)

## 查看帮助文档

```bash
# 查看 pods
[root@master1 ~]# kubectl explain pods
KIND:     Pod
VERSION:  v1

DESCRIPTION:
     Pod is a collection of containers that can run on a host. This resource is
     created by clients and scheduled onto hosts.

FIELDS:
   apiVersion	<string>
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources

   kind	<string>
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds

   metadata	<Object>
     Standard object's metadata. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata

   spec	<Object>
     Specification of the desired behavior of the pod. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status

   status	<Object>
     Most recently observed status of the pod. This data may not be up to date.
     Populated by the system. Read-only. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
# 查看 deploy 控制器     
[root@master1 ~]# kubectl  explain deploy
KIND:     Deployment
VERSION:  apps/v1

DESCRIPTION:
     Deployment enables declarative updates for Pods and ReplicaSets.

FIELDS:
   apiVersion	<string>
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources

   kind	<string>
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds

   metadata	<Object>
     Standard object's metadata. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata

   spec	<Object>
     Specification of the desired behavior of the Deployment.

   status	<Object>
     Most recently observed status of the Deployment.

```



* yaml以键值对的形式出现 属性名:[空格]属性值

  * apiVersion: v1

  * 可以通过kubectl apiversion 查看支持的版本

    ```bash
    [root@master1 ~]# kubectl  api-versions
    admissionregistration.k8s.io/v1
    apiextensions.k8s.io/v1
    apiregistration.k8s.io/v1
    apps/v1   #组 以及组下面的版本
    authentication.k8s.io/v1
    authorization.k8s.io/v1
    autoscaling/v1
    autoscaling/v2beta1
    autoscaling/v2beta2
    batch/v1   #稳定版 核心版
    batch/v1beta1  # beta是公测版  alpha 是公测版可能被删除
    certificates.k8s.io/v1
    coordination.k8s.io/v1
    crd.projectcalico.org/v1
    discovery.k8s.io/v1
    discovery.k8s.io/v1beta1
    events.k8s.io/v1
    events.k8s.io/v1beta1
    flowcontrol.apiserver.k8s.io/v1beta1
    networking.k8s.io/v1
    node.k8s.io/v1
    node.k8s.io/v1beta1
    policy/v1
    policy/v1beta1
    rbac.authorization.k8s.io/v1
    scheduling.k8s.io/v1
    storage.k8s.io/v1
    storage.k8s.io/v1beta1
    v1
    ```

  * Kind: Pod  定义类型为pod

  * metadata: 这是一个对象通过以下命令查看帮助，可以定义资源的名字所属的命名空间标签等

    ```bash
    [root@master1 ~]# kubectl explain pods.metadata
    KIND:     Pod
    VERSION:  v1
    
    RESOURCE: metadata <Object>
    
    DESCRIPTION:
         Standard object's metadata. More info:
         https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
    
         ObjectMeta is metadata that all persisted resources must have, which
         includes all objects users must create.
    
    FIELDS:
       annotations	<map[string]string> #资源的注释
         Annotations is an unstructured key value map stored with a resource that
         may be set by external tools to store and retrieve arbitrary metadata. They
         are not queryable and should be preserved when modifying objects. More
         info: http://kubernetes.io/docs/user-guide/annotations
    
       clusterName	<string>
         The name of the cluster which the object belongs to. This is used to
         distinguish resources with same name and namespace in different clusters.
         This field is not set anywhere right now and apiserver is going to ignore
         it if set in create or update request.
    
       creationTimestamp	<string>
         CreationTimestamp is a timestamp representing the server time when this
         object was created. It is not guaranteed to be set in happens-before order
         across separate operations. Clients may not set this value. It is
         represented in RFC3339 form and is in UTC.
    
         Populated by the system. Read-only. Null for lists. More info:
         https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
    
       deletionGracePeriodSeconds	<integer>
         Number of seconds allowed for this object to gracefully terminate before it
         will be removed from the system. Only set when deletionTimestamp is also
         set. May only be shortened. Read-only.
    
       deletionTimestamp	<string>
         DeletionTimestamp is RFC 3339 date and time at which this resource will be
         deleted. This field is set by the server when a graceful deletion is
         requested by the user, and is not directly settable by a client. The
         resource is expected to be deleted (no longer visible from resource lists,
         and not reachable by name) after the time in this field, once the
         finalizers list is empty. As long as the finalizers list contains items,
         deletion is blocked. Once the deletionTimestamp is set, this value may not
         be unset or be set further into the future, although it may be shortened or
         the resource may be deleted prior to this time. For example, a user may
         request that a pod is deleted in 30 seconds. The Kubelet will react by
         sending a graceful termination signal to the containers in the pod. After
         that 30 seconds, the Kubelet will send a hard termination signal (SIGKILL)
         to the container and after cleanup, remove the pod from the API. In the
         presence of network partitions, this object may still exist after this
         timestamp, until an administrator or automated process can determine the
         resource is fully terminated. If not set, graceful deletion of the object
         has not been requested.
    
         Populated by the system when a graceful deletion is requested. Read-only.
         More info:
         https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
    
       finalizers	<[]string>
         Must be empty before the object is deleted from the registry. Each entry is
         an identifier for the responsible component that will remove the entry from
         the list. If the deletionTimestamp of the object is non-nil, entries in
         this list can only be removed. Finalizers may be processed and removed in
         any order. Order is NOT enforced because it introduces significant risk of
         stuck finalizers. finalizers is a shared field, any actor with permission
         can reorder it. If the finalizer list is processed in order, then this can
         lead to a situation in which the component responsible for the first
         finalizer in the list is waiting for a signal (field value, external
         system, or other) produced by a component responsible for a finalizer later
         in the list, resulting in a deadlock. Without enforced ordering finalizers
         are free to order amongst themselves and are not vulnerable to ordering
         changes in the list.
    
       generateName	<string>
         GenerateName is an optional prefix, used by the server, to generate a
         unique name ONLY IF the Name field has not been provided. If this field is
         used, the name returned to the client will be different than the name
         passed. This value will also be combined with a unique suffix. The provided
         value has the same validation rules as the Name field, and may be truncated
         by the length of the suffix required to make the value unique on the
         server.
    
         If this field is specified and the generated name exists, the server will
         NOT return a 409 - instead, it will either return 201 Created or 500 with
         Reason ServerTimeout indicating a unique name could not be found in the
         time allotted, and the client should retry (optionally after the time
         indicated in the Retry-After header).
    
         Applied only if Name is not specified. More info:
         https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#idempotency
    
       generation	<integer>
         A sequence number representing a specific generation of the desired state.
         Populated by the system. Read-only.
    
       labels	<map[string]string>   # 资源标签
         Map of string keys and values that can be used to organize and categorize
         (scope and select) objects. May match selectors of replication controllers
         and services. More info: http://kubernetes.io/docs/user-guide/labels
    
       managedFields	<[]Object>
         ManagedFields maps workflow-id and version to the set of fields that are
         managed by that workflow. This is mostly for internal housekeeping, and
         users typically shouldn't need to set or understand this field. A workflow
         can be the user's name, a controller's name, or the name of a specific
         apply path like "ci-cd". The set of fields is always in the version that
         the workflow used when modifying the object.
    
       name	<string>    #定义的资源的名字
         Name must be unique within a namespace. Is required when creating
         resources, although some resources may allow a client to request the
         generation of an appropriate name automatically. Name is primarily intended
         for creation idempotence and configuration definition. Cannot be updated.
         More info: http://kubernetes.io/docs/user-guide/identifiers#names
    
       namespace	<string>   # 分配的名称空间
         Namespace defines the space within which each name must be unique. An empty
         namespace is equivalent to the "default" namespace, but "default" is the
         canonical representation. Not all objects are required to be scoped to a
         namespace - the value of this field for those objects will be empty.
    
         Must be a DNS_LABEL. Cannot be updated. More info:
         http://kubernetes.io/docs/user-guide/namespaces
    
       ownerReferences	<[]Object>
         List of objects depended by this object. If ALL objects in the list have
         been deleted, this object will be garbage collected. If this object is
         managed by a controller, then an entry in this list will point to this
         controller, with the controller field set to true. There cannot be more
         than one managing controller.
    
       resourceVersion	<string>
         An opaque value that represents the internal version of this object that
         can be used by clients to determine when objects have changed. May be used
         for optimistic concurrency, change detection, and the watch operation on a
         resource or set of resources. Clients must treat these values as opaque and
         passed unmodified back to the server. They may only be valid for a
         particular resource or set of resources.
    
         Populated by the system. Read-only. Value must be treated as opaque by
         clients and . More info:
         https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#concurrency-control-and-consistency
    
       selfLink	<string>
         SelfLink is a URL representing this object. Populated by the system.
         Read-only.
    
         DEPRECATED Kubernetes will stop propagating this field in 1.20 release and
         the field is planned to be removed in 1.21 release.
    
       uid	<string>
         UID is the unique in time and space value for this object. It is typically
         generated by the server on successful creation of a resource and is not
         allowed to change on PUT operations.
    
         Populated by the system. Read-only. More info:
         http://kubernetes.io/docs/user-guide/identifiers#uids
    
    ```

  * spec  非常重要，定义容器.嵌套了很多二级三级字段，不通的资源类型spec需要嵌套的字段各不相同，如果某个字段的标题属性是require（必选字段），剩下的都是可选字段，系统会赋予默认值。不通的资源类型spec的值不相同，它是用户定义的期望状态

    ```bash
    [root@master1 ~]# kubectl explain pods.spec
    KIND:     Pod
    VERSION:  v1
    
    RESOURCE: spec <Object>
    
    DESCRIPTION:
         Specification of the desired behavior of the pod. More info:
         https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
    
         PodSpec is a description of a pod.
    
    FIELDS:
       activeDeadlineSeconds	<integer>
         Optional duration in seconds the pod may be active on the node relative to
         StartTime before the system will actively try to mark it failed and kill
         associated containers. Value must be a positive integer.
    
       affinity	<Object>
         If specified, the pod's scheduling constraints
    
       automountServiceAccountToken	<boolean>
         AutomountServiceAccountToken indicates whether a service account token
         should be automatically mounted.
    
       containers	<[]Object> -required-  # 必填字段
         List of containers belonging to the pod. Containers cannot currently be
         added or removed. There must be at least one container in a Pod. Cannot be
         updated.
    
       dnsConfig	<Object>
         Specifies the DNS parameters of a pod. Parameters specified here will be
         merged to the generated DNS configuration based on DNSPolicy.
    
       dnsPolicy	<string>
         Set DNS policy for the pod. Defaults to "ClusterFirst". Valid values are
         'ClusterFirstWithHostNet', 'ClusterFirst', 'Default' or 'None'. DNS
         parameters given in DNSConfig will be merged with the policy selected with
         DNSPolicy. To have DNS options set along with hostNetwork, you have to
         specify DNS policy explicitly to 'ClusterFirstWithHostNet'.
    
       enableServiceLinks	<boolean>
         EnableServiceLinks indicates whether information about services should be
         injected into pod's environment variables, matching the syntax of Docker
         links. Optional: Defaults to true.
    
       ephemeralContainers	<[]Object>
         List of ephemeral containers run in this pod. Ephemeral containers may be
         run in an existing pod to perform user-initiated actions such as debugging.
         This list cannot be specified when creating a pod, and it cannot be
         modified by updating the pod spec. In order to add an ephemeral container
         to an existing pod, use the pod's ephemeralcontainers subresource. This
         field is alpha-level and is only honored by servers that enable the
         EphemeralContainers feature.
    
       hostAliases	<[]Object>
         HostAliases is an optional list of hosts and IPs that will be injected into
         the pod's hosts file if specified. This is only valid for non-hostNetwork
         pods.
    
       hostIPC	<boolean>
         Use the host's ipc namespace. Optional: Default to false.
    
       hostNetwork	<boolean>
         Host networking requested for this pod. Use the host's network namespace.
         If this option is set, the ports that will be used must be specified.
         Default to false.
    
       hostPID	<boolean>
         Use the host's pid namespace. Optional: Default to false.
    
       hostname	<string>
         Specifies the hostname of the Pod If not specified, the pod's hostname will
         be set to a system-defined value.
    
       imagePullSecrets	<[]Object>  # harbo 镜像会使用到
         ImagePullSecrets is an optional list of references to secrets in the same
         namespace to use for pulling any of the images used by this PodSpec. If
         specified, these secrets will be passed to individual puller
         implementations for them to use. For example, in the case of docker, only
         DockerConfig type secrets are honored. More info:
         https://kubernetes.io/docs/concepts/containers/images#specifying-imagepullsecrets-on-a-pod
    
       initContainers	<[]Object>
         List of initialization containers belonging to the pod. Init containers are
         executed in order prior to containers being started. If any init container
         fails, the pod is considered to have failed and is handled according to its
         restartPolicy. The name for an init container or normal container must be
         unique among all containers. Init containers may not have Lifecycle
         actions, Readiness probes, Liveness probes, or Startup probes. The
         resourceRequirements of an init container are taken into account during
         scheduling by finding the highest request/limit for each resource type, and
         then using the max of of that value or the sum of the normal containers.
         Limits are applied to init containers in a similar fashion. Init containers
         cannot currently be added or removed. Cannot be updated. More info:
         https://kubernetes.io/docs/concepts/workloads/pods/init-containers/
    
       nodeName	<string>
         NodeName is a request to schedule this pod onto a specific node. If it is
         non-empty, the scheduler simply schedules this pod onto that node, assuming
         that it fits resource requirements.
    
       nodeSelector	<map[string]string>
         NodeSelector is a selector which must be true for the pod to fit on a node.
         Selector which must match a node's labels for the pod to be scheduled on
         that node. More info:
         https://kubernetes.io/docs/concepts/configuration/assign-pod-node/
    
       overhead	<map[string]string>
         Overhead represents the resource overhead associated with running a pod for
         a given RuntimeClass. This field will be autopopulated at admission time by
         the RuntimeClass admission controller. If the RuntimeClass admission
         controller is enabled, overhead must not be set in Pod create requests. The
         RuntimeClass admission controller will reject Pod create requests which
         have the overhead already set. If RuntimeClass is configured and selected
         in the PodSpec, Overhead will be set to the value defined in the
         corresponding RuntimeClass, otherwise it will remain unset and treated as
         zero. More info:
         https://git.k8s.io/enhancements/keps/sig-node/688-pod-overhead/README.md
         This field is beta-level as of Kubernetes v1.18, and is only honored by
         servers that enable the PodOverhead feature.
    
       preemptionPolicy	<string>
         PreemptionPolicy is the Policy for preempting pods with lower priority. One
         of Never, PreemptLowerPriority. Defaults to PreemptLowerPriority if unset.
         This field is beta-level, gated by the NonPreemptingPriority feature-gate.
    
       priority	<integer>
         The priority value. Various system components use this field to find the
         priority of the pod. When Priority Admission Controller is enabled, it
         prevents users from setting this field. The admission controller populates
         this field from PriorityClassName. The higher the value, the higher the
         priority.
    
       priorityClassName	<string>
         If specified, indicates the pod's priority. "system-node-critical" and
         "system-cluster-critical" are two special keywords which indicate the
         highest priorities with the former being the highest priority. Any other
         name must be defined by creating a PriorityClass object with that name. If
         not specified, the pod priority will be default or zero if there is no
         default.
    
       readinessGates	<[]Object>
         If specified, all readiness gates will be evaluated for pod readiness. A
         pod is ready when all its containers are ready AND all conditions specified
         in the readiness gates have status equal to "True" More info:
         https://git.k8s.io/enhancements/keps/sig-network/580-pod-readiness-gates
    
       restartPolicy	<string>
         Restart policy for all containers within the pod. One of Always, OnFailure,
         Never. Default to Always. More info:
         https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy
    
       runtimeClassName	<string>
         RuntimeClassName refers to a RuntimeClass object in the node.k8s.io group,
         which should be used to run this pod. If no RuntimeClass resource matches
         the named class, the pod will not be run. If unset or empty, the "legacy"
         RuntimeClass will be used, which is an implicit class with an empty
         definition that uses the default runtime handler. More info:
         https://git.k8s.io/enhancements/keps/sig-node/585-runtime-class This is a
         beta feature as of Kubernetes v1.14.
    
       schedulerName	<string>
         If specified, the pod will be dispatched by specified scheduler. If not
         specified, the pod will be dispatched by default scheduler.
    
       securityContext	<Object>
         SecurityContext holds pod-level security attributes and common container
         settings. Optional: Defaults to empty. See type description for default
         values of each field.
    
       serviceAccount	<string>
         DeprecatedServiceAccount is a depreciated alias for ServiceAccountName.
         Deprecated: Use serviceAccountName instead.
    
       serviceAccountName	<string>
         ServiceAccountName is the name of the ServiceAccount to use to run this
         pod. More info:
         https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/
    
       setHostnameAsFQDN	<boolean>
         If true the pod's hostname will be configured as the pod's FQDN, rather
         than the leaf name (the default). In Linux containers, this means setting
         the FQDN in the hostname field of the kernel (the nodename field of struct
         utsname). In Windows containers, this means setting the registry value of
         hostname for the registry key
         HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters to
         FQDN. If a pod does not have FQDN, this has no effect. Default to false.
    
       shareProcessNamespace	<boolean>
         Share a single process namespace between all of the containers in a pod.
         When this is set containers will be able to view and signal processes from
         other containers in the same pod, and the first process in each container
         will not be assigned PID 1. HostPID and ShareProcessNamespace cannot both
         be set. Optional: Default to false.
    
       subdomain	<string>
         If specified, the fully qualified Pod hostname will be
         "<hostname>.<subdomain>.<pod namespace>.svc.<cluster domain>". If not
         specified, the pod will not have a domainname at all.
    
       terminationGracePeriodSeconds	<integer>
         Optional duration in seconds the pod needs to terminate gracefully. May be
         decreased in delete request. Value must be non-negative integer. The value
         zero indicates stop immediately via the kill signal (no opportunity to shut
         down). If this value is nil, the default grace period will be used instead.
         The grace period is the duration in seconds after the processes running in
         the pod are sent a termination signal and the time when the processes are
         forcibly halted with a kill signal. Set this value longer than the expected
         cleanup time for your process. Defaults to 30 seconds.
    
       tolerations	<[]Object>
         If specified, the pod's tolerations.
    
       topologySpreadConstraints	<[]Object>
         TopologySpreadConstraints describes how a group of pods ought to spread
         across topology domains. Scheduler will schedule pods in a way which abides
         by the constraints. All topologySpreadConstraints are ANDed.
    
       volumes	<[]Object>
         List of volumes that can be mounted by containers belonging to the pod.
         More info: https://kubernetes.io/docs/concepts/storage/volumes
    ```
    
    * 查看ports 定义
    
      ```bash
      [root@master1 ~]# kubectl explain pods.spec.containers.ports
      KIND:     Pod
      VERSION:  v1
      
      RESOURCE: ports <[]Object>
      
      DESCRIPTION:
           List of ports to expose from the container. Exposing a port here gives the
           system additional information about the network connections a container
           uses, but is primarily informational. Not specifying a port here DOES NOT
           prevent that port from being exposed. Any port which is listening on the
           default "0.0.0.0" address inside a container will be accessible from the
           network. Cannot be updated.
      
           ContainerPort represents a network port in a single container.
      
      FIELDS:
         containerPort	<integer> -required-
           Number of port to expose on the pod's IP address. This must be a valid port
           number, 0 < x < 65536.
      
         hostIP	<string>
           What host IP to bind the external port to.
      
         hostPort	<integer>
           Number of port to expose on the host. If specified, this must be a valid
           port number, 0 < x < 65536. If HostNetwork is specified, this must match
           ContainerPort. Most containers do not need this.
      
         name	<string>
           If specified, this must be an IANA_SVC_NAME and unique within the pod. Each
           named port in a pod must have a unique name. Name for the port that can be
           referred to by services.
      
         protocol	<string>
           Protocol for port. Must be UDP, TCP, or SCTP. Defaults to "TCP".
      ```
    
      

示例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tomcat-pod
  namespace: default
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
```

应用ymal

```bash
[root@master1 opt]#  kubectl get pods -n default -o wide
NAME                        READY   STATUS    RESTARTS      AGE     IP                NODE    NOMINATED NODE   READINESS GATES
my-nginx-5b56ccd65f-6tqg9   1/1     Running   0             135m    192.168.166.139   node1   <none>           <none>
my-nginx-5b56ccd65f-vfrtl   1/1     Running   0             135m    192.168.104.12    node2   <none>           <none>
tomcat-pod                  2/2     Running   1 (59s ago)   2m14s   192.168.104.13    node2   <none>           <none>
[root@master1 opt]# kubectl  describe pods tomcat-pod
Name:         tomcat-pod
Namespace:    default
Priority:     0
Node:         node2/192.168.0.110
Start Time:   Tue, 07 Dec 2021 23:58:37 +0800
Labels:       tomcat=tomcat-pod
Annotations:  cni.projectcalico.org/podIP: 192.168.104.13/32
              cni.projectcalico.org/podIPs: 192.168.104.13/32
Status:       Running
IP:           192.168.104.13
IPs:
  IP:  192.168.104.13
Containers:
  tomcat-pod-java:
    Container ID:   docker://6d1363532d996353ba5e138edce9321c35d8fee3c31218e712b40b76c16c685e
    Image:          tomcat:8.5-jre8-alpine
    Image ID:       docker://sha256:8b8b1eb786b54145731e1cd36e1de208d10defdbb0b707617c3e7ddb9d6d99c8
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Tue, 07 Dec 2021 23:58:38 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-lgd85 (ro)
  nginx:
    Container ID:   docker://0c78f07cee1156a287d0110830776a9caede5a4b10f058602a4edc0ccede8361
    Image:          nginx
    Image ID:       docker-pullable://nginx@sha256:9522864dd661dcadfd9958f9e0de192a1fdda2c162a35668ab6ac42b465f0603
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Tue, 07 Dec 2021 23:59:52 +0800
    Last State:     Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Tue, 07 Dec 2021 23:59:43 +0800
      Finished:     Tue, 07 Dec 2021 23:59:52 +0800
    Ready:          True
    Restart Count:  1
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-lgd85 (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-lgd85:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason         Age                    From               Message
  ----     ------         ----                   ----               -------
  Normal   Scheduled      3m37s                  default-scheduler  Successfully assigned default/tomcat-pod to node2
  Normal   Pulled         3m36s                  kubelet            Container image "tomcat:8.5-jre8-alpine" already present on machine
  Normal   Created        3m36s                  kubelet            Created container tomcat-pod-java
  Normal   Started        3m36s                  kubelet            Started container tomcat-pod-java
  Warning  InspectFailed  2m57s (x6 over 3m36s)  kubelet            Failed to apply default image tag "nginx：alpine": couldn't parse image reference "nginx：alpine": invalid reference format
  Warning  Failed         2m57s (x6 over 3m36s)  kubelet            Error: InvalidImageName
  Normal   Pulling        2m51s                  kubelet            Pulling image "nginx:alpine"
  Normal   Pulled         2m31s                  kubelet            Successfully pulled image "nginx:alpine" in 19.997293352s
  Normal   Created        2m22s (x2 over 2m31s)  kubelet            Created container nginx
  Normal   Started        2m22s (x2 over 2m31s)  kubelet            Started container nginx
  Normal   Killing        2m22s                  kubelet            Container nginx definition changed, will be restarted
  Normal   Pulled         2m22s                  kubelet            Container image "nginx" already present on machine
```

