# Service访问地址---->实际服务提供者的访问地址 的映射

1. `Endpoints`、`EndpointSlice`是两种类似但有些许不同的资源，共同点是都代表了实际服务提供者的访问地址的集合(ip:port)。它们可以是跑在一些pod里，应用的访问地址的集合(pod.ip+应用在容器中的port)，也可以是跟k8s无关的、可访问的、外部服务地址集合。比如你的应用端口是`8080`，跑在了两个pod(ip: `1.1.1.1、1.1.1.2`)里，那么endpints/endpointslice表示的就是:`1.1.1.1:8080`和`1.1.1.2:8080`

2. `kube-proxy`服务会近实时地监控`Service`、`Endpointslice`、`Node`资源的变化，往节点的`iptables`中写入`Service`访问地址(ip:port)到实际服务提供者的访问地址(ip:port)的映射规则(`DNAT`等)。早先版本也监控了`Endpoints`资源的变化，但因为`EndpointSlice`的出现，后续版本剔除了。

3. `kube-proxy`是以`DaemonSet`的形式部署的，也就是说在k8s集群中，每个节点上都会跑一个`kube-proxy`实例，并且使用宿主机的`network namespace`来操作宿主机的`iptables`。

# Endpoints和EndpointSlice的关系和区别

1. `Endpoints`是老的实现，`EndpointSlice`是新的实现。

2. 对于`EndpointSlice`，用户可以将实际服务提供者的访问地址集合自由地分割成多个子集，每个子集使用一个`EndpointSlice`来定义，然后绑定到同一个`Service`上，这样每个子集(`EndpointSlice`)都可以独立更新自己的访问地址集合。比如服务A，跑在了10个pod实例上，那么可以定义2个`EndpointSlice`来表示这10个pod+端口访问地址，每个`EndpointSlice`表示5个。但`Endpoints`不可以，每个`Service`只能有一个`Endpoints`。

3. 因为每个节点上的`kube-proxy`实例会近实时地监控同步`Service`、`EndpointSlice`、`Endpoints`的变化，并将`Service->Endpoints/EndpointSlice`地址映射关系写入每个`节点(node)`的`iptables`，所以当**规模(实际服务访问地址数量 * 节点数量) * 频率(EndpointSlice/Endpoints集合变化)** 越大，使用`EndpointSlice`越能带来明显收益，因为每个`EndpointSlice`不是全量的访问地址，并且可以独立更新。

4. 为什么多个`EndpointSlice`可以绑定到同一个`Service`上，但是只能有一个`Endpoints`可以绑定到同一个`Service`上？
   - 简单来说是因为`Endpoints`和`Service`关联时使用的是各自的`name`属性，即:`Endpoints.metadata.name == Service.metadata.name`，但是在同一个`namespace`以及同一种资源类型下，`name`是唯一的，这样就限制了一个`Service`只能关联到一个`Endpoints`上
   - 而定义`EndpointSlice`资源时，是通过`kubernetes.io/service-name`这个label和service关联上的，这样就不会有name唯一的限制，从而可以让多个`EndpointSlice`绑定到同一个`Service`上

# 新老版本和Endpoints和EndpointSlice
1. 在1.16版本之前，`kube-proxy`仅监控`Endpoints`资源的变化，生成对应的`iptables`规则
2. 从1.26版本开始，`kube-proxy`不再监控`Endpints`资源的变化，仅监控`EndpointSlice`资源的变化，并生成对应的`iptables`规则。但是从1.19版本开始，`kube-controller-manager`服务引入了`endpointslicemirroring Controller`来监控`Endpoints`资源的变化并为其生成对应的`EndpintSlice`资源。

综上，对于`Service`，有两种方式为其定义实际服务提供者访问地址，即：`Endpoints`和`EndpoitSlice`。在新版中它们映射到iptables的路径分别是：
1. Endpoints---->EndpointSlice---->iptables
2. EndpointSlice---->iptables

也就是说新版实际上是根据`EndpointSlice`来生成`iptables`，为了兼容`Endpints`，使用`endpointslicemirroring Controller`来将`Endpints`转换为`EndpointSlice`


# Service中的选择算符

选择算符是用来匹配相应的pod，结合端口号，生成`Endpoint`和`EndpointSlice`资源，作为实际服务提供者的访问地址。

## 有选择算符

在定义`Service`时指定`spec.selector`，被用来查询具有指定相同标签的Pods。   

`kube-controller-manager`中有两个服务：`endpoint Controller`和`endpointslice Controller`，它们会分别监控`Service`的变化（新增&删除），比如当前新增一个`Service`，会根据`Service`中的`spec.selector`找到相应的Pod的ip地址，结合定义的端口，来生成对应的`Endpoint`和`EndpointSlice`资源。`kube-proxy`也会监控到`EndpointSlice`的变化，写入对应的`iptables`规则。

例子：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: kubernetes-bootcamp-v1-8080
  name: kubernetes-bootcamp-v1-8080
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: kubernetes-bootcamp-v1-8080
  template:
    metadata:
      labels:
        app.kubernetes.io/name: kubernetes-bootcamp-v1-8080
    spec:
      containers:
      - image: docker.io/hhitzhl/kubernetes-bootcamp:v1
        name: kubernetes-bootcamp-v1-8080
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: svc-8091
  namespace: default
spec:
  ports:
    - name: svc-p8091
      port: 8091
      protocol: TCP
      targetPort: 8080
  selector:
    app.kubernetes.io/name: kubernetes-bootcamp-v1-8080 # pod标签
  type: ClusterIP
```

创建完之后，可以观察到，多了一个和service.name相同的`Endpoints`(name=bootcamp-svc-8060)，和一个以service.name为前缀的`EndpointSlice`资源(name=bootcamp-svc-8060-$***)。它们的port.name都是相同的

## 无选择算符

无选择算符，即实际提供服务的地址跟k8s的pod没有强关联，可以是外部的一个地址，只要在k8s集群中可以访问。需要在`Endpoints`或者`EndpointSlice`中显式地指定服务提供者访问地址，然后将`Endpoint/EndpointSlice`绑定到`Service`上来提供服务。

### Endpoints例子

显示定义Endpoint必须遵守以下几个规则：
-  Endpoint的name和Service的name属性必须一致
-  Endpoint中的Port的name属性必须和Service中的Port name一致

一个隐性的规则：
Service中的spec.ports[i].targetPort可以不指定，即便指定了也不起作用，因为它俩使用：Service.spec.ports[i].name==Endpoints.subsets[m].ports[n].name来关联，，关联后使用Endpoints的port，而不是Service中的targetPort

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-8092
spec:
  ports:
    - protocol: TCP
      name: svc-p8092
      port: 8092
---
apiVersion: v1
kind: Endpoints
metadata:
  name: svc-8092 # 必须和service的name一致
  namespace: default
subsets:
  - addresses:
      - ip: 10.244.2.5 # 实际服务地址，这里指定了一个pod地址
    ports:
      - name: svc-p8092 # 必须和service中的某个Port的name一致
        port: 8080
        protocol: TCP
```

可以观察到，k8s会生成一个`name`为`bootcamp-svc-8090-***`的`EndpointSlice`资源

### EndpointSlice例子

显示定义EndpointSlice必须遵守以下几个规则：
-  `EndpointSlice`中的`metadata.labels.kubernetes.io/service-name`属性值必须和`Service`的`metadata.name`属性必须一致
-  `EndpointSlice`中的`ports[i].name`属性必须和`Service`中的`spec.ports[m].ports[n].name`一致

一个隐性的规则：
Service中的spec.ports[i].targetPort可以不指定，即便指定了也不起作用，因为它俩使用：Service.spec.ports[i].name==Endpoints.subsets[m].ports[n].name来关联，，关联后使用Endpoints的port，而不是Service中的targetPort

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-8093
spec:
  ports:
    - protocol: TCP
      name: svc-p8093
      port: 8093
---

apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
addressType: IPv4
endpoints:
  - addresses:
      - 10.244.1.25 # 实际服务地址，这里指定了一个pod的地址
metadata:
  labels:
    kubernetes.io/service-name: svc-8093
  name: 'svc-8093-endpointslice'
  namespace: default
ports:
  - name: svc-p8093 # 必须和service中的某个Port的name一致
    port: 8080
    protocol: TCP
```
