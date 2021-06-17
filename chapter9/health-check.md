---
title: "容器健康检查"
date: 2021-02-27T17:40:49+08:00
draft: false
tags: ["kubernetes","health check"]
categories: ["云计算"]
---

<!--more-->

## 一. 背景

发布容器的时候，需要输入Health Check的检查方式，以便K8S集群能够管理容器的健康状态，做容器的停止和重建的操作。公司发布的容器应用类型包括多种，有Nginx+PHP，OSP，Nginx+Tomcat等，而每个应用的Health Check的方式都各异，因此需要统一各应用的Health Check检查方式。

## 二. Health Check方式

先归纳下各种应用Health Check的使用方式

- Nginx+PHP 和 Nginx+Tomcat，以及 ai-agent 类型应用容器，是**通过调用`{domain-name}:port/_health_check`的http接口做Health Check**，各应用必须提供这个url
- OSP服务，Saturn服务以及ai-model类型应用容器，是**通过telnet连接指定的端口做Health Check**
- 其他类型的容器，是**通过容器自定义脚本做Health Check，如果没有自定义脚本，则直接touch /tmp/health，默认容器健康检查成功**

## 三. 容器化的Health Check机制

K8S的Health Check目的是检查业务的Pod是否可用

容器统一使用ExecProbe调用脚本`/apps/sh/k8s_node/health_check.sh`

容器健康检查的开关机制：

- 检查/apps/sh/no_health_check文件，如果文件存在且其中内容为true，则跳过健康检查，直接返回0。这个功能适用于禁用单个容器的健康检查，可以配合vjdump等命令使用。
- 检查/docker/logs/<domain_name>/no_health_check文件，如果文件存在且其中内容为true，则跳过健康检查，直接返回0。由于/docker/logs/<domain_name>目录对应到宿主机的/apps/logs/log_receiver/<domain_name>目录，运维可以下发文件来禁用整个域所有容器的健康检查。

### 1. Http方式的Health Check

K8S启动的Nginx+PHP 和 Nginx+Tomcat类型应用容器，以及ai-agent类型应用容器，可以通过以下方式做Health Check

`/apps/sh/k8s_node/health_check.sh http <port> <timeout>`

```yaml
livenessProbe:
  exec:
    command:
    - bash
    - /apps/sh/k8s_node/health_check.sh
    - http
    - "80"
    - "5"
  failureThreshold: 3
  initialDelaySeconds: 300
  periodSeconds: 3
  successThreshold: 1
  timeoutSeconds: 5
name: php-k6zq82
readinessProbe:
  exec:
    command:
    - bash
    - /apps/sh/k8s_node/health_check.sh
    - http
    - "80"
    - "1"
  failureThreshold: 3
  initialDelaySeconds: 10
  periodSeconds: 3
  successThreshold: 1
  timeoutSeconds: 1
```

默认配置定义了应用类型与其http健康检查对应的端口
`probe.httpget.apps=${PROBE_HTTPGET_APPS:{"web":80,"ai-agent":8079}}`

可以通过配置中心下发PROBE_HTTPGET_APPS来修改

### 2. Tcp方式的Health Check

K8S启动的OSP服务，Saturn服务以及ai-model类型应用容器，可以通过以下方式做Health Check

`/apps/sh/k8s_node/health_check.sh tcp <port> <timeout>`

```yaml
livenessProbe:
  exec:
    command:
    - bash
    - /apps/sh/k8s_node/health_check.sh
    - tcp
    - "8080"
    - "5"
  failureThreshold: 3
  initialDelaySeconds: 300
  periodSeconds: 3
  successThreshold: 1
  timeoutSeconds: 5
name: osp-kdo8d6
readinessProbe:
  exec:
    command:
    - bash
    - /apps/sh/k8s_node/health_check.sh
    - tcp
    - "8080"
    - "1"
  failureThreshold: 3
  initialDelaySeconds: 10
  periodSeconds: 3
  successThreshold: 1
  timeoutSeconds: 1
```

Noah默认配置定义了应用类型与其tcp健康检查对应的端口
`probe.tcpsocket.apps=${NOAH_PROBE_TCPSOCKET_APPS:{"osp":8080,"saturn":24500,"ai-model":8500}}	`

可以通过配置中心下发NOAH_PROBE_TCPSOCKET_APPS来修改

### 3. Other方式的Health Check

如果应用类型不在probe.httpget.apps和probe.tcpsocket.apps的配置中，则使用other类型的健康检查

other类型的健康检查实现为touch /tmp/health，容器健康检查默认成功。

`/apps/sh/k8s_node/health_check.sh other <timeout>`

```yaml
livenessProbe:
  exec:
    command:
    - bash
    - /apps/sh/k8s_node/health_check.sh
    - other
    - "5"
  failureThreshold: 3
  initialDelaySeconds: 300
  periodSeconds: 3
  successThreshold: 1
  timeoutSeconds: 5
name: php-k6zq82
readinessProbe:
  exec:
    command:
    - bash
    - /apps/sh/k8s_node/health_check.sh
    - other
    - "1"
  failureThreshold: 3
  initialDelaySeconds: 10
  periodSeconds: 3
  successThreshold: 1
  timeoutSeconds: 1
```

### 4. 自定义Health Check

Noah容器自定义健康检查，在构建容器镜像时可以通过moana extend的方式在容器启动时将custom_health_check.sh放到/apps/sh目录下：

在http/tcp/other类型的hc过程中，如果容器内存在文件/apps/sh/custom_health_check.sh时，则调用/apps/sh/custom_health_check.sh脚本；入参为timeout，脚本需要实现检查超时；如果脚本输出ok（一定要小写），则代表健康检查成功。

## 四. 关于容器ReadinessProbe和LivenessProbe的说明

1. ReadinessProbe通过，则容器进入Ready状态；
2. Liveness如果连续失败failureThreshold指定的次数，则认为容器不能正常服务，kubelet会重启容器；
3. initialDeplaySeconds设定probe的延迟秒数，即容器启动后等待多少秒开始执行probe；
4. periodSeconds设定probe的间隔，即开始执行probe后每隔多少秒执行一次；
5. timeoutSeconds设定一次probe的超时时间，由于ExecProbe实际上是没有使用这个值的，所以health_check.sh和other类型所用到的custom_health_check.sh需要自己实现超时；