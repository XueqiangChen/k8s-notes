---
title: "使用kubeadm 安装k8s集群"
date: 2021-02-20T09:12:03+08:00
draft: false
tags: ["kubernetes","kubeadm"]
categories: ["云计算"]
---

## 1. 准备工作

### 机器配置

1. 8核CPU、8GB内存；
2. 40GB磁盘
3. centos 7.9
4. 内网互同
5. 外网访问不受限制

### 组件信息

| 组件       | 版本       |
| ---------- | :--------- |
| 系统       | Centos 7.9 |
| Docker版本 | 18.09.9    |
| k8s 版本   | 1.20.0     |
| Pod 网段   | 10.32.0.0/ |

### 实践目标

1. 在所有节点上安装 Docker 和 kubeadm；
2. 部署 Kubernetes Master；
3. 部署容器网络插件；
4. 部署 Kubernetes Worker；
5. 部署 Dashboard 可视化插件；
6. 部署容器存储插件

### 基本配置

开始安装之前，我们还需要对系统做一些基本的配置。

**所有节点配置 hosts**

```bash
# cat /etc/hosts

10.186.4.100 master
10.186.4.167 node1
10.186.4.168 node2
10.186.4.169 node3
```

**所有节点关闭防火墙、selinux、dnsmasq**

```bash
systemctl disable --now firewalld
#关闭dnsmasq(否则可能导致docker容器无法解析域名)
systemctl disable --now dnsmasq
systemctl disable --now NetworkManager

# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

**关闭swap**

```bash
swapoff -a && sysctl -w vm.swappiness=0
sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab
```

**允许iptales查看网桥流量**

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
```

**安装ntpdate**

```bash
rpm -ivh http://mirrors.wlnmp.com/centos/wlnmp-release-centos.noarch.rpm
yum install ntpdate -y
```

同步时间

```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
echo 'Asia/Shanghai' >/etc/timezone
ntpdate time2.aliyun.com
```

加入到 crontab

```bash
*/5 * * * * ntpdate time2.aliyun.com
```

**节点之间免密登录**

```bash
ssh-keygen -t rsa
for i in master node1 node2 node3;do ssh-copy-id -i .ssh/id_rsa.pub $i;done
```

**开放端口**

![](https://chenxqblog-1258795182.cos.ap-guangzhou.myqcloud.com/k8s-port.png)

网络插件weave的端口是6783。

## 2. 安装 kubeadm 和 docker

### docker 安装

> 安装过程参考[docker install](https://docs.docker.com/engine/install/centos/)

**yum源配置**

```bash
yum install -y yum-utils

# 官方源
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

# 阿里云源
yum-config-manager \
	--add-repo \
	https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

如果是安装最新版本的 docker，可以直接安装:

```bash
$ sudo yum install docker-ce docker-ce-cli containerd.io
```

指定版本安装的话，先查看源中可用的docker版本：

```bash
$ yum list docker-ce --showduplicates | sort -r

docker-ce.x86_64  3:18.09.1-3.el7                     docker-ce-stable
docker-ce.x86_64  3:18.09.0-3.el7                     docker-ce-stable
docker-ce.x86_64  18.06.1.ce-3.el7                    docker-ce-stable
docker-ce.x86_64  18.06.0.ce-3.el7                    docker-ce-stable
```

通过软件包名称安装特定版本，软件包名称是软件包名称（docker-ce）加上版本字符串（第二列），从第一个冒号（:)开始，直至第一个连字符，并用连字符（-）连接。例如docker-ce-18.09.9。

```bash
$ sudo yum install -y docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io
```

设置开机启动

```bash
$ sudo systemctl start docker 
$ systemctl daemon-reload && systemctl enable --now docker
```

### kubeadm

> 安装过程参考 [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

添加 yum 源，执行安装命令。

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# 阿里云源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```

在上述安装 kubeadm 的过程中，kubeadm 和 kubelet、kubectl、kubernetes-cni 这几个二进制文件都会被自动安装好。

安装的时候指定版本：

```bash
sudo yum install -y kubelet-1.20.0-0 kubeadm-1.20.0-0 kubectl-1.20.0-0 --disableexcludes=kubernetes
```

**`kubectl` 命令启用 `shell` 自动补齐功能**

```bash
# 1.安装bash-completion
yum install bash-completion
# 重新加载你的 Shell 并运行 type _init_completion
type _init_completion

# 2.启动 kubectl 自动补齐
echo 'source <(kubectl completion bash)' >>~/.bashrc
source ~/.bashrc
```

**开机自启动**

```bash
systemctl daemon-reload && systemctl enable --now kubelet
```

## 3. 部署kubernetes 的master节点

[kubeadm 初始化](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/)的时候可以通过命令行参数或者配置文件的方式进行。我们可以通过 [`kubeadm config`](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-config) 命令来查看 `kubeadm` 的配置。

初始化需要的镜像可以通过`kubeadm config images list` 来查看：

```bash
# kubeadm config images list

k8s.gcr.io/kube-apiserver:v1.20.4
k8s.gcr.io/kube-controller-manager:v1.20.4
k8s.gcr.io/kube-scheduler:v1.20.4
k8s.gcr.io/kube-proxy:v1.20.4
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.13-0
k8s.gcr.io/coredns:1.7.0
```

为了减少初始化的时间，可以用 `kubeadm config images pull` 命令提前拉取镜像，获取镜像的时候，默认使用的 `k8s.gcr.io` 这个镜像源，国内是下载不了的。有两个解决办法：

1. 使用国内的阿里云镜像源： 

   ```bash
   cat >/etc/sysconfig/kubelet<<EOF
   KUBELET_EXTRA_ARGS="--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.2"
   EOF
   
   # 或者通过命令行指定 image repo
   kubeadm config images pull --image-repository registry.aliyuncs.com/google_containers
   ```

2. 设置代理，`docker pull` 的代理配置参考[Docker的三种网络代理配置](https://note.qidong.name/2020/05/docker-proxy/)

`kubeadm` 的初始化这里采用了配置文件的方式，`kubeadm` 对于低版本的配置文件是不兼容的，我们通过 `kubeadm config migrate --new-config ${new-file} --old config ${old-file}` 命令来转换。

```yaml
# kubeadm.yml
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: bw4tlw.owb266appoos6afw
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.186.4.100
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: yxj-test
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  extraArgs:
    runtime-config: api/all=true
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager:
  extraArgs:
    horizontal-pod-autoscaler-sync-period: 10s
    horizontal-pod-autoscaler-use-rest-clients: "true"
    node-monitor-grace-period: 10s
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: v1.20.3
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
networking:
  dnsDomain: cluster.local
  podSubnet: 10.32.0.0/12
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

这个配置中，给kube-controller-manager 设置了

```bash
horizontal-pod-autoscaler-use-rest-clients: "true"
```

这意味着，将来部署的 kube-controller-manager 能够使用自定义资源（Custom Metrics）进行自动水平扩展。

由于我们这里使用的网络插件是wave，自定义Pod 的 cidr 网络为：

```bash
podSubnet: 10.32.0.0/12
```

然后，执行一句指令：

```bash
$ kubeadm init --config kubeadm.yaml
[init] Using Kubernetes version: v1.20.0
[preflight] Running pre-flight checks
        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local yxj-test] and IPs [10.96.0.1 10.186.4.100]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost yxj-test] and IPs [10.186.4.100 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost yxj-test] and IPs [10.186.4.100 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 15.004566 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.20" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node yxj-test as control-plane by adding the labels "node-role.kubernetes.io/master=''" and "node-role.kubernetes.io/control-plane='' (deprecated)"
[mark-control-plane] Marking the node yxj-test as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: bw4tlw.owb266appoos6afw
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.186.4.100:6443 --token bw4tlw.owb266appoos6afw \
    --discovery-token-ca-cert-hash sha256:67c5b1f31669be60c07280f52b9aab867af54d2d4bb679490216028f858c7fb6
```

就可以完成 Kubernetes Master 的部署了，这个过程只需要几分钟。部署完成后，kubeadm 会生成一行指令

```bash
kubeadm join 10.186.4.100:6443 --token bw4tlw.owb266appoos6afw \
    --discovery-token-ca-cert-hash sha256:67c5b1f31669be60c07280f52b9aab867af54d2d4bb679490216028f858c7fb6
```

这个 kubeadm join 命令，就是用来给这个 Master 节点添加更多工作节点（Worker）的命令。我们在后面部署 Worker 节点的时候马上会用到它，所以找一个地方把这条命令记录下来。

此外，kubeadm 还会提示我们第一次使用 Kubernetes 集群所需要的配置命令：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

而需要这些配置命令的原因是：Kubernetes 集群默认需要加密方式访问。所以，这几条命令，就是将刚刚部署生成的 Kubernetes 集群的安全配置文件，保存到当前用户的.kube 目录下，kubectl 默认会使用这个目录下的授权信息访问 Kubernetes 集群。如果不这么做的话，我们每次都需要通过 export KUBECONFIG 环境变量告诉 kubectl 这个安全配置文件的位置。现在，我们就可以使用 kubectl get 命令来查看当前唯一一个节点的状态了：

```bash
# kubectl get nodes

NAME       STATUS   ROLES                  AGE     VERSION
yxj-test   Ready    control-plane,master   3h14m   v1.20.0
```

默认情况下，出于安全原因，不会在master节点上调度Pod。如果希望能够在master节点上调度Pod，就可以运行如下的命令：

```bash
# kubectl taint nodes --all node-role.kubernetes.io/master-
node/yxj-test untainted
taint "node-role.kubernetes.io/master" not found
taint "node-role.kubernetes.io/master" not found
taint "node-role.kubernetes.io/master" not found
```

这样就删除主节点上的 `node-role.kubernetes.io/master ` 污点，调度器就能将Pod调度到 master 节点上。

**修改组件参数**

如果我们需要修改 k8s 组件的参数，那么可以在 `/etc/kubernetes/manifests/` 这个目录下编辑组件的yaml文件，保存后会重启对应的组件。

## 4. 部署网络插件

以weave为例子：

```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

部署完成后，我们可以通过 kubectl get 重新检查 Pod 的状态：

```bash
# kubectl get pods -n kube-system
NAME                               READY   STATUS    RESTARTS   AGE
coredns-74ff55c5b-f95sj            1/1     Running   0          131m
coredns-74ff55c5b-zvjdz            1/1     Running   0          131m
etcd-yxj-test                      1/1     Running   0          131m
kube-apiserver-yxj-test            1/1     Running   0          131m
kube-controller-manager-yxj-test   1/1     Running   0          131m
kube-proxy-nm4tt                   1/1     Running   0          131m
kube-scheduler-yxj-test            1/1     Running   0          131m
weave-net-pl4qp                    2/2     Running   1          126m
```

可以看到，所有的系统 Pod 都成功启动了，而刚刚部署的 Weave 网络插件则在 kube-system 下面新建了一个名叫 weave-net-pl4qp 的 Pod，一般来说，这些 Pod 就是容器网络插件在每个节点上的控制组件。

Kubernetes 支持容器网络插件，使用的是一个名叫 CNI 的通用接口，它也是当前容器网络的事实标准，市面上的所有容器网络开源项目都可以通过 CNI 接入 Kubernetes，比如 Flannel、Calico、Canal、Romana 等等，它们的部署方式也都是类似的“一键部署”。

> weave 插件部署报错：weave Inconsistent bridge state detected. Please do 'weave reset' and try again
>
> 解决：安装[weave](https://www.weave.works/docs/net/latest/install/installing-weave/)
>
> ```bash
> sudo curl -L git.io/weave -o /usr/local/bin/weave
> sudo chmod a+x /usr/local/bin/weave
> ```
>
> 

## 5. 部署 k8s 的 worker 节点

Kubernetes 的 Worker 节点跟 Master 节点几乎是相同的，它们运行着的都是一个 kubelet 组件。唯一的区别在于，在 kubeadm init 的过程中，kubelet 启动后，Master 节点上还会自动运行 kube-apiserver、kube-scheduler、kube-controller-manger 这三个系统 Pod。

所以，相比之下，部署 Worker 节点反而是最简单的，只需要两步即可完成。

* 第一步，在所有 Worker 节点上执行“安装 kubeadm 和 Docker”一节的所有步骤。

* 第二步，执行部署 Master 节点时生成的 kubeadm join 指令：

  ```bash
  kubeadm join 10.186.4.100:6443 --token bw4tlw.owb266appoos6afw \
      --discovery-token-ca-cert-hash sha256:36f6f60943015fadcc0fc9824611af4d594b7c4b43ab264c2113342fbd1e3994
  ```

**通过 Taint/Toleration 调整 Master 执行 Pod 的策略**

我在前面提到过，默认情况下 Master 节点是不允许运行用户 Pod 的。而 Kubernetes 做到这一点，依靠的是 Kubernetes 的 Taint/Toleration 机制。

它的原理非常简单：一旦某个节点被加上了一个 Taint，即被“打上了污点”，那么所有 Pod 就都不能在这个节点上运行，因为 Kubernetes 的 Pod 都有“洁癖”。除非，有个别的 Pod 声明自己能“容忍”这个“污点”，即声明了 Toleration，它才可以在这个节点上运行。

其中，为节点打上“污点”（Taint）的命令是：

```bash
$ kubectl taint nodes node1 foo=bar:NoSchedule
```

这时，该 node1 节点上就会增加一个键值对格式的 Taint，即：foo=bar:NoSchedule。其中值里面的 NoSchedule，意味着这个 Taint 只会在调度新 Pod 时产生作用，而不会影响已经在 node1 上运行的 Pod，哪怕它们没有 Toleration。

那么 Pod 又如何声明 Toleration 呢？

我们只要在 Pod 的.yaml 文件中的 spec 部分，加入 tolerations 字段即可：

```bash

apiVersion: v1
kind: Pod
...
spec:
  tolerations:
  - key: "foo"
    operator: "Equal"
    value: "bar"
    effect: "NoSchedule"
```

这个 Toleration 的含义是，这个 Pod 能“容忍”所有键值对为 foo=bar 的 Taint（ operator: “Equal”，“等于”操作）。

现在回到我们已经搭建的集群上来。这时，如果你通过 kubectl describe 检查一下 Master 节点的 Taint 字段，就会有所发现了：

```bash

$ kubectl describe node master

Name:               master
Roles:              master
Taints:             node-role.kubernetes.io/master:NoSchedule
```

可以看到，Master 节点默认被加上了node-role.kubernetes.io/master:NoSchedule这样一个“污点”，其中“键”是node-role.kubernetes.io/master，而没有提供“值”。

此时，你就需要像下面这样用“Exists”操作符（operator: “Exists”，“存在”即可）来说明，该 Pod 能够容忍所有以 foo 为键的 Taint，才能让这个 Pod 运行在该 Master 节点上：

```bash
apiVersion: v1
kind: Pod
...
spec:
  tolerations:
  - key: "foo"
    operator: "Exists"
    effect: "NoSchedule"
```

当然，如果你就是想要一个单节点的 Kubernetes，删除这个 Taint 才是正确的选择：

```bash
$ kubectl taint nodes --all node-role.kubernetes.io/master-
```

如上所示，我们在“node-role.kubernetes.io/master”这个键后面加上了一个短横线“-”，这个格式就意味着移除所有以“node-role.kubernetes.io/master”为键的 Taint。到了这一步，一个基本完整的 Kubernetes 集群就部署完毕了。是不是很简单呢？有了 kubeadm 这样的原生管理工具，Kubernetes 的部署已经被大大简化。更重要的是，像证书、授权、各个组件的配置等部署中最麻烦的操作，kubeadm 都已经帮你完成了。接下来，我们再在这个 Kubernetes 集群上安装一些其他的辅助插件，比如 Dashboard 和存储插件。

## 6. 部署 Dashboard 可视化插件

在 Kubernetes 社区中，有一个很受欢迎的 Dashboard 项目，它可以给用户提供一个可视化的 Web 界面来查看当前集群的各种信息。毫不意外，它的部署也相当简单：

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.2.0/aio/deploy/recommended.yaml
```

这里通过暴露NodePort的方式来提供服务，需要修改默认的yaml文件：

```yaml
...
kind: Service
 apiVersion: v1
 metadata:
   labels:
     k8s-app: kubernetes-dashboard
   name: kubernetes-dashboard
   namespace: kubernetes-dashboard
 spec:
   type: NodePort
   ports:
     - port: 443
       targetPort: 8443
       nodePort: 30000
   selector:
     k8s-app: kubernetes-dashboard
 ...
```

部署完成之后，我们就可以查看 Dashboard 对应的 Pod 的状态了：

```bash
# kubectl get pods -n kubernetes-dashboard
NAME                                         READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-79c5968bdc-wwhns   1/1     Running   0          12m
kubernetes-dashboard-9f9799597-pkc76         1/1     Running   0          12m
```

需要注意的是，由于 Dashboard 是一个 Web Server，很多人经常会在自己的公有云上无意地暴露 Dashboard 的端口，从而造成安全隐患。所以，1.7 版本之后的 Dashboard 项目部署完成后，默认只能通过 Proxy 的方式在本地访问。具体的操作，你可以查看 Dashboard 项目的[官方文档](https://github.com/kubernetes/dashboard)。

### 创建dashboard管理员

```bash
[root@fztelecom3-34 dashboard]# vim dashboard-admin.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: dashboard-admin
  namespace: kubernetes-dashboard


[root@fztelecom3-34 dashboard]# kubectl create -f ./dashboard-admin.yaml
```

### 6.2. 为用户分配权限

```bash
[root@fztelecom3-34 dashboard]# vim dashboard-admin-bind-cluster-role.yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-admin-bind-cluster-role
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: dashboard-admin
  namespace: kubernetes-dashboard

[root@fztelecom3-34 dashboard]# kubectl create -f ./dashboard-admin-bind-cluster-role.yaml
```

### 6.3. 查看登录的token

```ba
[root@fztelecom3-34 dashboard]# kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep dashboard-admin | awk '{print $1}')
```

打开nodeip:port访问

## 7. 部署容器存储插件

很多时候我们需要用数据卷（Volume）把外面宿主机上的目录或者文件挂载进容器的 Mount Namespace 中，从而达到容器和宿主机共享这些目录或者文件的目的。容器里的应用，也就可以在这些数据卷中新建和写入文件。

可是，如果你在某一台机器上启动的一个容器，显然无法看到其他机器上的容器在它们的数据卷里写入的文件。这是容器最典型的特征之一：无状态。

而容器的持久化存储，就是用来保存容器存储状态的重要手段：存储插件会在容器里挂载一个基于网络或者其他机制的远程数据卷，使得在容器里创建的文件，实际上是保存在远程存储服务器上，或者以分布式的方式保存在多个节点上，而与当前宿主机没有任何绑定关系。这样，无论你在其他哪个宿主机上启动新的容器，都可以请求挂载指定的持久化存储卷，从而访问到数据卷里保存的内容。这就是“持久化”的含义。

由于 Kubernetes 本身的松耦合设计，绝大多数存储项目，比如 Ceph、GlusterFS、NFS 等，都可以为 Kubernetes 提供持久化存储能力。在这次的部署实战中，我会选择部署一个很重要的 Kubernetes 存储插件项目：Rook。

Rook 项目是一个基于 Ceph 的 Kubernetes 存储插件（它后期也在加入对更多存储实现的支持）。不过，不同于对 Ceph 的简单封装，Rook 在自己的实现中加入了水平扩展、迁移、灾难备份、监控等大量的企业级功能，使得这个项目变成了一个完整的、生产级别可用的容器存储插件。

得益于容器化技术，用几条指令，Rook 就可以把复杂的 Ceph 存储后端部署起来：

https://rook.io/docs/rook/v1.5/ceph-quickstart.html#ceph-storage-quickstart

```bash
git clone --single-branch --branch v1.5.8 https://github.com/rook/rook.git
cd rook/cluster/examples/kubernetes/ceph
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
kubectl create -f cluster.yaml

# 所有的镜像，需要通过外网来拉取
ceph/ceph:v15.2.9
k8s.gcr.io/sig-storage/csi-node-driver-registrar:v2.0.1
k8s.gcr.io/sig-storage/csi-attacher:v3.0.0
k8s.gcr.io/sig-storage/csi-snapshotter:v3.0.0
k8s.gcr.io/sig-storage/csi-resizer:v1.0.0
k8s.gcr.io/sig-storage/csi-provisioner:v2.0.0
quay.io/cephcsi/cephcsi:v3.2.0
ceph/ceph:v15.2.9
```

在部署完成后，你就可以看到 Rook 项目会将自己的 Pod 放置在由它自己管理的两个 Namespace 当中：

```bash

$ kubectl get pods -n rook-ceph-system
NAME                                  READY     STATUS    RESTARTS   AGE
rook-ceph-agent-7cv62                 1/1       Running   0          15s
rook-ceph-operator-78d498c68c-7fj72   1/1       Running   0          44s
rook-discover-2ctcv                   1/1       Running   0          15s

$ kubectl get pods -n rook-ceph
NAME                   READY     STATUS    RESTARTS   AGE
rook-ceph-mon0-kxnzh   1/1       Running   0          13s
rook-ceph-mon1-7dn2t   1/1       Running   0          2s
```

这样，一个基于 Rook 持久化存储集群就以容器的方式运行起来了，而接下来在 Kubernetes 项目上创建的所有 Pod 就能够通过 Persistent Volume（PV）和 Persistent Volume Claim（PVC）的方式，在容器里挂载由 Ceph 提供的数据卷了。

这时候，你可能会有个疑问：为什么我要选择 Rook 项目呢？其实，是因为这个项目很有前途

如果你去研究一下 Rook 项目的实现，就会发现它巧妙地依赖了 Kubernetes 提供的编排能力，合理的使用了很多诸如 Operator、CRD 等重要的扩展特性（这些特性我都会在后面的文章中逐一讲解到）。这使得 Rook 项目，成为了目前社区中基于 Kubernetes API 构建的最完善也最成熟的容器存储插件。我相信，这样的发展路线，很快就会得到整个社区的推崇。

> 备注：其实，在很多时候，大家说的所谓“云原生”，就是“Kubernetes 原生”的意思。而像 Rook、Istio 这样的项目，正是贯彻这个思路的典范。在我们后面讲解了声明式 API 之后，相信你对这些项目的设计思想会有更深刻的体会。

## 8. 卸载集群

### 8.1 Master

kubeadm reset

停止docker

```bash
sudo systemctl stop docker kubelet
```

删除相关配置文件：

```bash
rm -rf ~/.kube/
```

Node

```bash
[root@test-2 ~]# kubeadm reset
[reset] WARNING: Changes made to this host by 'kubeadm init' or 'kubeadm join' will be reverted.
[reset] Are you sure you want to proceed? [y/N]: y
[preflight] Running pre-flight checks
W0305 11:09:18.394192   10467 removeetcdmember.go:79] [reset] No kubeadm config, using etcd pod spec to get data directory
[reset] No etcd config found. Assuming external etcd
[reset] Please, manually reset etcd to prevent further issues
[reset] Stopping the kubelet service
[reset] Unmounting mounted directories in "/var/lib/kubelet"
[reset] Deleting contents of config directories: [/etc/kubernetes/manifests /etc/kubernetes/pki]
[reset] Deleting files: [/etc/kubernetes/admin.conf /etc/kubernetes/kubelet.conf /etc/kubernetes/bootstrap-kubelet.conf /etc/kubernetes/controller-manager.conf /etc/kubernetes/scheduler.conf]
[reset] Deleting contents of stateful directories: [/var/lib/kubelet /var/lib/dockershim /var/run/kubernetes /var/lib/cni]

The reset process does not clean CNI configuration. To do so, you must remove /etc/cni/net.d

The reset process does not reset or clean up iptables rules or IPVS tables.
If you wish to reset iptables, you must do so manually by using the "iptables" command.

If your cluster was setup to utilize IPVS, run ipvsadm --clear (or similar)
to reset your system's IPVS tables.

The reset process does not clean your kubeconfig files and you must remove them manually.
Please, check the contents of the $HOME/.kube/config file.
```

停止相关的服务

```bash
sudo systemctl stop docker && yum remove docker-ce docker-ce-cli containerd.io -y
```

删除相关配置文件：

```bash
# 清除网络插件的配置
rm -rf /etc/cni/net.d/

# 清除iptables规则或者 IPVS 表
# 查看规则以number的方式，一条一条的出来，然后我们根据号码来删除哪一条规则
iptables -L INPUT --line-numbers
# 删除第七条规则
iptables -D INPUT 7
# 删除所有规则
sudo iptables -F && sudo iptables -X && sudo iptables -F -t nat && sudo iptables -X -t nat
# 如果开启了ipvs，通过下面的命令清除
ipvsadm --clear

# 清除网桥
ip link del weave
ip link del docker0
ip link del vethwe-bridge
ip link del vxlan-6784

# 删除kubeconfig文件
rm -rf ~/.kube/
```



## 参考文档：

[1] [kubeadm](https://feisky.gitbooks.io/kubernetes/content/deploy/kubeadm.html)

[2] [Kubernetes实战指南（三十四）： 高可用安装K8s集群1.20.x](https://www.cnblogs.com/dukuan/p/14124600.html)

[3] [kubeadm安装最新高可用K8S集群v1.20.2](https://zhuanlan.zhihu.com/p/349342658)

[4] [k8s 1.20.0 在centos7 使用 kubeadm 安装](https://blog.csdn.net/weixin_40814867/article/details/111031110)

[5] [Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

