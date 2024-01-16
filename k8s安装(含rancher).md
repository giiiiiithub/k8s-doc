# 宿主机上的服务

宿主机上最少仅需要部署两个**服务**即可将k8s拉起来：`kubelet`和`containerd `。  

`kubelet`是k8s的node agent组件，`containerd`是容器运行时组件，类似的组件有docker、CRI-O等。可以参考这篇文章：`https://zhuanlan.zhihu.com/p/490585683`

以下依赖关系只需要满足一个即可，也就是说`docker`不是必须的。
- kubelet---->containerd
- kubelet---->docker---->containerd

为某些方便起见，我们还是会安装`docker`。但是注意：本教程并不依赖`docker`作为作为容器运行时，而是直接使用`containerd`作为容器运行时。

# 宿主机上的工具

宿主机上还需要两个额外的**工具**，用来安装k8s集群。

- kubeadm: 部署k8s节点使用
- kubectl: k8s命令行工具，用来操作k8s集群，比如部署应用、查看日志等等。哪个节点需要就在哪个节点上安装

# 准备

- 最小硬件需求
  - 2核cpu
  - 2GB内存
  - 30GB硬盘
  - 3台宿主机(本教程使用虚拟机)
- 操作系统
  - `CentOS-7-x86_64-DVD-2009.iso`  
  - 下载地址: `https://mirrors.aliyun.com/centos/7/isos/x86_64/CentOS-7-x86_64-DVD-2009.iso`   
  - 为方便起见，安装时，在**安装信息摘要**步骤的**软件选择**栏，选择**带GUI的服务器**
- 节点规划:
  - master: 192.168.0.151
  - node:   192.168.0.152
  - node:   192.168.0.153
- vmware使用桥接网络
  - 如果是用vmware创建虚拟机，则需要指定虚拟机网络为`网络适配器`为`桥接模式`，以保持虚拟机和宿主机在同一网段
- 克隆虚拟机配置
  - 假设你使用vmware创建了一个centos7系统的虚拟机，为了快速方便，其他虚拟机是克隆来的，那么克隆的虚拟机必须修改网络接口uuid: `sed -i "s/UUID=.*/UUID=$(uuidgen)/" /etc/sysconfig/network-scripts/ifcfg-ens33`

# 开始安装
以下除`master节点安装`和`工作节点安装`两个章节区分`master`和`node`节点之外，其他安装步骤需要在**所有节点**上执行。

## centos7 基本设置
- 关闭防火墙
  - 临时 `systemctl stop firewalld`
  - 永久 `systemctl disable firewalld`
- 关闭selinux防火墙
  - `sudo setenforce 0`
  - `sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config`
- turn off swap
  - 临时 `swapoff -a`
  - 永久
    1. `yes | cp /etc/fstab /etc/fstab_bak`
    2. `cat /etc/fstab_bak | grep -v swap > /etc/fstab`
  - 检查是否关闭: `free -h`

- 配置hostname
   - 在`192.168.0.151`上执行命令：`hostnamectl set-hostname kb1`
   - 在`192.168.0.152`上执行命令：`hostnamectl set-hostname kb2`
   - 在`192.168.0.153`上执行命令：`hostnamectl set-hostname kb3`
   - 在所有节点执行命令：
      ```shell
      cat <<eof  >> /etc/hosts
      192.168.0.151  kb1
      192.168.0.152  kb2
      192.168.0.153  kb3
      eof
      ```

## 安装docker和containerd
docker依赖了containerd，k8s可直接使用containerd，不需要docker。但是containerd的deb/rpm是由docker发行的，所以这里为了方便使用docker安装。 

1. `yum install -y yum-utils`
2. `yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo`
3. `sed -i 's+https://download.docker.com+https://mirrors.tuna.tsinghua.edu.cn/docker-ce+' /etc/yum.repos.d/docker-ce.repo`
4. `yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`
5. containerd配置
    1. `containerd config default > /etc/containerd/config.toml`
    2. `sed -i 's/sandbox_image.*/sandbox_image = "registry.aliyuncs.com\/google_containers\/pause:3.9"/g' /etc/containerd/config.toml`  
    **注意**：配置文件中如果version = 1 则不会生效，version必须为2才能生效。
    3. `containerd config dump | grep pause`
    4. `systemctl daemon-reload`
    5. `systemctl enable --now containerd`
    6. `systemctl status containerd`
6. docker配置（可选）
    k8s不使用docker的话可以不配置不启动
    1. `echo '{ "exec-opts": ["native.cgroupdriver=systemd"] }' > /etc/docker/daemon.json`
    2. `systemctl daemon-reload`
    3. `systemctl enable --now docker`
    4. `docker run hello-world`

## containerd 镜像加速

- `mkdir -p /etc/containerd/certs.d/docker.io`
- `sed -i 's/config_path = ""/config_path = "\/etc\/containerd\/certs.d"/g' /etc/containerd/config.toml`
- ```shell
  cat <<EOF | tee /etc/containerd/certs.d/docker.io/hosts.toml
  server = "https://docker.io"
  
  [host."https://dockerproxy.com"]
    capabilities = ["pull", "resolve"]
  ```
- `systemctl daemon-reload`
- `systemctl status containerd`

containerd参考：`https://github.com/containerd/containerd/blob/main/docs/hosts.md`    
仓库代理参考：`https://dockerproxy.com`

## 转发 IPv4 并让 iptables 看到桥接流量 
```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# 加载内核模块overlay和br_netfilter
sudo modprobe overlay
sudo modprobe br_netfilter

# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 应用 sysctl 参数而不重新启动
sudo sysctl --system
```

## kubeadm 安装(含kubelet、kubectl)

1. 配置源
```shell
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=kubernetes
baseurl=https://mirrors.tuna.tsinghua.edu.cn/kubernetes/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```
2. 安装kubelet、kubectl、kubeadm：`yum install -y kubelet-1.27.6 kubectl-1.27.6 kubeadm-1.27.6 --disableexcludes=kubernetes`
3. 启用kubelet服务：`systemctl enable --now kubelet`
4. 查看kubelet服务是否正常启动：`systemctl status kubelet`

## master节点安装

1. 找一个节点作为master，其他为工作节点。在master节点上执行以下：
2. 清空脏数据，安装如果失败了，重新安装要执行：`kubeadm reset -f`
3.  执行初始化命令。 --apiserver-advertise-address参数值是master节点ip，其他几个ip参数可以不变
```shell
  kubeadm init \
   --image-repository registry.aliyuncs.com/google_containers \
   --kubernetes-version v1.27.6 \
   --apiserver-advertise-address 192.168.0.151 \
   --service-cidr=10.1.0.0/16 \
   --pod-network-cidr=10.244.0.0/16
```

**安装完成后会有提示，需要拷贝提示中的kubeadm join ...命令**，类似如下：
```
kubeadm join 192.168.0.151:6443 --token sakdfb.whea5g8ebwhjgk4v \
  --discovery-token-ca-cert-hash sha256:e02263c547a19edd88a9f6f779d759b095b6c4cd5d64b3f0a9844f0439c76def
```

4. 生成kubectl命令行工具配置文件
```shell
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```
5. 安装CNI，这里选flannel，命令：`kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml`
6. 查看节点：`kubectl get node -o wide`，或查看节点详情：`kubectl describe nodes`

## 杂项
- 移除污点，假设`master`节点的`hostname`是`kb1`，则执行命令：`kubectl taint nodes kb1 node-role.kubernetes.io/control-plane=:NoSchedule-`
- 命令自动补全 `echo 'source <(kubectl completion bash)' >> ~/.bashrc`


## 工作节点安装

1. 安装并注册node到master。粘贴执行安装master节点完成后提示的kubeadmin join...命令。类似如下：
    ```shell
    kubeadm join --v=9 --token sakdfb.whea5g8ebwhjgk4v 192.168.0.153:6443 \
      --discovery-token-ca-cert-hash sha256:e02263c547a19edd88a9f6f779d759b095b6c4cd5d64b3f0a9844f0439c76def
    ```  
    如果不记得可以按照以下步骤，找出token及其hash，然后执行`kube join`命令。
   
        - 查看token: 在master上执行`kubeadm token list`
        - 创建token: 如果看不到token，就在master上执行`kubeadm token create --print-join-command`
        - 生成discovery-token-ca-cert-hash: 在master上执行`openssl x509 -in /etc/kubernetes/pki/ca.crt -noout -pubkey | openssl rsa -pubin -outform DER 2>/dev/null | sha256sum | cut -d' ' -f1`

3. 如果需要kubectl，则将master上的`/etc/kubernetes/admin.conf`文件拷贝到`$HOME/.kube/config`，然后执行:  
   `chown $(id -u):$(id -g) $HOME/.kube/config
`
4. 查看节点状态， 需要一点时间状态才会转换成ready
    - `kubectl get nodes`  `kubectl describe node $nodeName`

## 工作节点卸载
1. 查找：`kubectl get nodes`
2. 进入维护模式：`kubectl drain $nodeName --delete-local-data --force --ignore-daemonsets node/$nodeName`
3. 删除：`kubectl delete node $nodeName`
4. 在node节点上清理资源：`kubeadm reset -f`

## rancher安装

以下步骤在master节点上执行即可。

1. 安装helm：`https://helm.sh/docs/intro/install/#from-the-binary-releases`
2. 
    ```
    helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
    
    kubectl create namespace cattle-system
    
    kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/<VERSION>/cert-manager.crds.yaml
    
    helm repo add jetstack https://charts.jetstack.io
    
    helm repo update
    
    helm install cert-manager jetstack/cert-manager \
      --namespace cert-manager \
      --create-namespace \
      --version v1.13.3
    ```

3. 提前使用代理下载镜像:
    1. `ctr --namespace k8s.io image pull dockerproxy.com/rancher/rancher:v2.8.0`
    2. `ctr --namespace k8s.io images tag dockerproxy.com/rancher/rancher:v2.8.0 docker.io/rancher/rancher:v2.8.0`
    3. `ctr --namespace k8s.io images remove dockerproxy.com/rancher/rancher:v2.8.0`
5. 安装rancher服务。(以下hostname是给ingress用的，本教程是没有的，可以随意设置
    ```
    helm install rancher rancher-latest/rancher \
      --namespace cattle-system \
      --version v2.8.0
      --set hostname=<IP_OF_LINUX_NODE>.sslip.io \
      --set replicas=1 \
      --set bootstrapPassword=<PASSWORD_FOR_RANCHER_ADMIN>
    ```
6. 将`spec.type`改为`NodePort`：`kubectl edit svc -n cattle-system rancher`
7. 访问：

    a. 查看nodeport: `kubectl get svc -A`，有一行：  
        ```
        cattle-system  rancher  NodePort  10.1.24.156  <none>  80:31235/TCP,443:30699/TCP  16m
        ```
   
    b. 假如集群中中一台宿主机的ip是`192.168.0.151`，则访问https端口：`https://192.168.0.151:30699` 即可。使用集群中其他宿主机ip也可以。

## 忘记密码

重置: `kubectl -n cattle-system exec $(kubectl -n cattle-system get pods | grep ^rancher | head -n 1 | awk '{ print $1 }') reset-password`


# 日志设置

为方便学习用，可以把把日志级别设置更高一些使得日志更为详细。

## kubelet日志级别调整

1. kubelet 日志级别调整：`vim /var/lib/kubelet/kubeadm-flags.env`，在参数行追加：`--v=6`，
2. `systemctl restart kubelet`
3. kubelet日志查看：`journalctl -r -u kubelet.service`

## kube-controller-manager、kube-apiserver、kube-scheduler、etcd日志级别调整

1. 由于kube-controller-manager是静态pod，需要先找到pod描述文件：`more /var/lib/kubelet/config.yaml | grep -i staticPodPath`
2. 编辑对应组件的yaml文件，在`spec.containers.command`中加入参数：`--v=6`。例：`vim /etc/kubernetes/manifests/kube-controller-manager.yaml`
3. 重启kubelet，各个pod也会随之重启：`systemctl restart kubelet`

### 日志参考参考

- `https://docs.openshift.com/container-platform/4.8/rest_api/editing-kubelet-log-level-verbosity.html`
- `https://github.com/kubernetes/community/blob/master/contributors/devel/sig-instrumentation/logging.md`
- `https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/system-logs`

