---
title: "常用的k8s命令"
date: 2021-02-25T07:37:43+08:00
draft: false
tags: ["kubernetes"]
categories: ["云计算"]
---

## 1. yaml 文件

### 1.1 创建

```bash
$ kubectl create -f 我的配置文件
```

### 1.2 修改

更新 yaml 文件：

```bash
$ kubectl replace -f nginx-deployment.yaml
```

声明式的表达方式，使用 apply 命令：

```bash
$ kubectl apply -f nginx-deployment.yaml
```

## 2. Pod

### 2.1 查询

**获取 pod 的信息**

```bash
kubectl get pods -n ${namespace}
```

**根据 pod 的标签过滤 **

```bash
kubectl get pods -l app=nginx
```

注意的是，在命令行中，所有 key-value 格式的参数，都使用“=”而非“:”表示。

**获取 pod 的描述信息**

```bash
$ kubectl describe pod <pod-name>
```

### 2.2 登录

进入到 Pod 中：

```bash
$ kubectl exec -it nginx-deployment-5c678cfb6d-lg9lw -- /bin/bash
```

### 2.3 删除

删除 Pod：

```bash
$ kubectl delete -f nginx-deployment.yaml
```

### 2.4. 更新

通过修改yaml文件更新pod

```yaml
...    
    spec:
      containers:
      - name: nginx
        image: nginx:1.8 #这里被从1.7.9修改为1.8
        ports:
      - containerPort: 80
```

执行 `kubectl replace` 命令

```bash
$ kubectl replace -f nginx-deployment.yaml
```

这里更推荐用 `kubectl apply` 命令，来统一进行 Kubernetes 对象的创建和更新操作

```bash

# 修改nginx-deployment.yaml的内容

$ kubectl apply -f nginx-deployment.yaml
```

### 2.5 创建一个临时的Pod

```bash
$ kubectl run -i --tty --image busybox:1.28.4 dns-test --restart=Never --rm /bin/sh 
```

通过这条命令，我们启动了一个一次性的 Pod，因为–rm 意味着 Pod 退出后就会被删除掉。

## 3. 日志

### 3.1. kube-apiserver 日志

```bash
PODNAME=$(kubectl -n kube-system get pod -l component=kube-apiserver -o jsonpath='{.items[0].metadata.name}')
kubectl -n kube-system logs $PODNAME --tail 100
```

以上命令操作假设控制平面以 Kubernetes 静态 Pod 的形式来运行。如果 kube-apiserver 是用 systemd 管理的，则需要登录到 master 节点上，然后使用 `journalctl -u kube-apiserver` 查看其日志。

### 3.2. kube-controller-manager 日志

```bash
PODNAME=$(kubectl -n kube-system get pod -l component=kube-controller-manager -o jsonpath='{.items[0].metadata.name}')
kubectl -n kube-system logs $PODNAME --tail 100
```

以上命令操作假设控制平面以 Kubernetes 静态 Pod 的形式来运行。如果 kube-controller-manager 是用 systemd 管理的，则需要登录到 master 节点上，然后使用 `journalctl -u kube-controller-manager` 查看其日志。

### 3.3. kube-scheduler 日志

```bash
PODNAME=$(kubectl -n kube-system get pod -l component=kube-scheduler -o jsonpath='{.items[0].metadata.name}')
kubectl -n kube-system logs $PODNAME --tail 100
```

以上命令操作假设控制平面以 Kubernetes 静态 Pod 的形式来运行。如果 kube-scheduler 是用 systemd 管理的，则需要登录到 master 节点上，然后使用 `journalctl -u kube-scheduler` 查看其日志。

### 3.4. kubelet 日志

```bash
journalctl -l -u kubelet
```

### 3.5. Pod 日志

查看指定Pod的日志

```bash
kubectl logs <pod-name> -n <namespace>

# 类似 tail -f 的日志
kubectl logs -f <pod-name> -n <namespace>
```

例子：

```bash
# Return snapshot logs from pod nginx with only one container
kubectl logs nginx

# Return snapshot logs from pod nginx with multi containers
kubectl logs nginx --all-containers=true

# Return snapshot logs from all containers in pods defined by label app=nginx
kubectl logs -lapp=nginx --all-containers=true

# Return snapshot of previous terminated ruby container logs from pod web-1
kubectl logs -p -c ruby web-1

# Begin streaming the logs of the ruby container in pod web-1
kubectl logs -f -c ruby web-1

# Begin streaming the logs from all containers in pods defined by label app=nginx
kubectl logs -f -lapp=nginx --all-containers=true

# Display only the most recent 20 lines of output in pod nginx
kubectl logs --tail=20 nginx

# Show all logs from pod nginx written in the last hour
kubectl logs --since=1h nginx

# Show logs from a kubelet with an expired serving certificate
kubectl logs --insecure-skip-tls-verify-backend nginx

# Return snapshot logs from first container of a job named hello
kubectl logs job/hello

# Return snapshot logs from container nginx-1 of a deployment named nginx
kubectl logs deployment/nginx -c nginx-1
```

## 4. 污点

一旦某个节点被加上了一个 Taint，即被“打上了污点”，那么所有 Pod 就都不能在这个节点上运行，因为 Kubernetes 的 Pod 都有“洁癖”。

除非，有个别的 Pod 声明自己能“容忍”这个“污点”，即声明了 Toleration，它才可以在这个节点上运行。

为节点打污点(Taint)的命令是：

```bash
$ kubectl taint nodes node1 foo=bar:NoSchedule
```

这时，该 node1 节点上就会增加一个键值对格式的 Taint，即：foo=bar:NoSchedule。其中值里面的 NoSchedule，意味着这个 Taint 只会在调度新 Pod 时产生作用，而不会影响已经在 node1 上运行的 Pod，哪怕它们没有 Toleration。

那么 Pod 又如何声明 Toleration 呢？

我们只要在 Pod 的.yaml 文件中的 spec 部分，加入 tolerations 字段即可：

```yaml
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

通常，master 节点上会自带一个污点：

```yaml
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

如上所示，我们在“node-role.kubernetes.io/master”这个键后面加上了一个短横线“-”，这个格式就意味着移除所有以“node-role.kubernetes.io/master”为键的 Taint。

## 5. ProjectedVolume

### 5.1 Secret

#### 5.1.1. 创建

```bash
kubectl create secret generic user --from-file=./username.txt
```

#### 5.1.2 查询

```bash
kubectl get secrets
```

### 5.2. ConfigMap

#### 5.2.1. 创建

```bash
$ kubectl create configmap <configmap-na,e> --from-file=<file-name>
```

#### 5.22. 查询 

```bash
kubectl get configmaps <configmap-name> -o yaml
```

