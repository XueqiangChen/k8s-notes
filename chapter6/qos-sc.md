---
title: "QoS 源代码分析"
date: 2021-04-20T20:02:37+08:00
draft: false
tags: ["kubernetes","QoS"]
categories: ["云计算"]
---

QOS 的作用请参考上面的几篇文章[Kubernetes的资源模型和资源管理](https://chenxq.xyz/post/cloud-k8s-resource-model-and-resource-manager/)

<!--more-->

**qos**

```go
// pkg/apis/core/v1/helper/qos/qos.go
func GetPodQOS(pod *v1.Pod) v1.PodQOSClass {
	requests := v1.ResourceList{}
	limits := v1.ResourceList{}
	zeroQuantity := resource.MustParse("0")
	isGuaranteed := true
	allContainers := []v1.Container{}
	allContainers = append(allContainers, pod.Spec.Containers...)
	allContainers = append(allContainers, pod.Spec.InitContainers...)
	for _, container := range allContainers {
		// process requests
		for name, quantity := range container.Resources.Requests {
			if !isSupportedQoSComputeResource(name) {
				continue
			}
			if quantity.Cmp(zeroQuantity) == 1 {
				delta := quantity.DeepCopy()
				if _, exists := requests[name]; !exists {
					requests[name] = delta
				} else {
					delta.Add(requests[name])
					requests[name] = delta
				}
			}
		}
		// process limits
		qosLimitsFound := sets.NewString()
		for name, quantity := range container.Resources.Limits {
			if !isSupportedQoSComputeResource(name) {
				continue
			}
			if quantity.Cmp(zeroQuantity) == 1 {
				qosLimitsFound.Insert(string(name))
				delta := quantity.DeepCopy()
				if _, exists := limits[name]; !exists {
					limits[name] = delta
				} else {
					delta.Add(limits[name])
					limits[name] = delta
				}
			}
		}

		if !qosLimitsFound.HasAll(string(v1.ResourceMemory), string(v1.ResourceCPU)) {
			isGuaranteed = false
		}
	}
	if len(requests) == 0 && len(limits) == 0 {
		return v1.PodQOSBestEffort
	}
	// Check is requests match limits for all resources.
	if isGuaranteed {
		for name, req := range requests {
			if lim, exists := limits[name]; !exists || lim.Cmp(req) != 0 {
				isGuaranteed = false
				break
			}
		}
	}
	if isGuaranteed &&
		len(requests) == len(limits) {
		return v1.PodQOSGuaranteed
	}
	return v1.PodQOSBurstable
}
```

上面有注释我就不过多介绍，非常的简单。

下面这里是QOS OOM打分机制，通过给不同的pod打分来判断，哪些pod可以被优先kill掉，分数越高的越容易被kill。

**policy**

```go
//\pkg\kubelet\qos\policy.go
// 分值越高越容易被kill
const (
    // KubeletOOMScoreAdj is the OOM score adjustment for Kubelet
    KubeletOOMScoreAdj int = -999
    // KubeProxyOOMScoreAdj is the OOM score adjustment for kube-proxy
    KubeProxyOOMScoreAdj  int = -999
    guaranteedOOMScoreAdj int = -998
    besteffortOOMScoreAdj int = 1000
)
```

**policy#GetContainerOOMScoreAdjust**

```go
// pkg/kubelet/qos/policy.go
// GetContainerOOMScoreAdjust returns the amount by which the OOM score of all processes in the
// container should be adjusted.
// The OOM score of a process is the percentage of memory it consumes
// multiplied by 10 (barring exceptional cases) + a configurable quantity which is between -1000
// and 1000. Containers with higher OOM scores are killed if the system runs out of memory.
// See https://lwn.net/Articles/391222/ for more information.
func GetContainerOOMScoreAdjust(pod *v1.Pod, container *v1.Container, memoryCapacity int64) int {
	if types.IsNodeCriticalPod(pod) {
		// Only node critical pod should be the last to get killed.
		return guaranteedOOMScoreAdj
	}

	switch v1qos.GetPodQOS(pod) {
	case v1.PodQOSGuaranteed:
		// Guaranteed containers should be the last to get killed.
		return guaranteedOOMScoreAdj
	case v1.PodQOSBestEffort:
		return besteffortOOMScoreAdj
	}

	// Burstable containers are a middle tier, between Guaranteed and Best-Effort. Ideally,
	// we want to protect Burstable containers that consume less memory than requested.
	// The formula below is a heuristic. A container requesting for 10% of a system's
	// memory will have an OOM score adjust of 900. If a process in container Y
	// uses over 10% of memory, its OOM score will be 1000. The idea is that containers
	// which use more than their request will have an OOM score of 1000 and will be prime
	// targets for OOM kills.
	// Note that this is a heuristic, it won't work if a container has many small processes.
	memoryRequest := container.Resources.Requests.Memory().Value()
	oomScoreAdjust := 1000 - (1000*memoryRequest)/memoryCapacity
	// A guaranteed pod using 100% of memory can have an OOM score of 10. Ensure
	// that burstable pods have a higher OOM score adjustment.
	if int(oomScoreAdjust) < (1000 + guaranteedOOMScoreAdj) {
		return (1000 + guaranteedOOMScoreAdj)
	}
	// Give burstable pods a higher chance of survival over besteffort pods.
	if int(oomScoreAdjust) == besteffortOOMScoreAdj {
		return int(oomScoreAdjust - 1)
	}
	return int(oomScoreAdjust)
}
```

这个函数返回容器中所有进程的OOM分数。进程的OOM分数是其消耗的内存百分比乘以10(除非有特殊情况)再加上一个可配置的数，这个数在 -1000 到 1000 之间。当系统内存不足时，OOM 分数越高的容器越容易被kill掉。

对于静态Pod、镜像Pod和高优先级Pod，QoS直接被设置成为Guaranteed，而 Guaranteed 等级的容器最后被 kill。Besteffort 等级的容器最容易被删掉，而处于中间位置的 Burstable 等级的 Pod。

理想情况下，我们要保护消耗少于请求的内存的Burstable容器。这里采用了一种启发式计算：

```
oomScoreAdjust = 1000 - (1000*memoryRequest)/memoryCapacity
```

> 问题：这里的memoryRequest 单位是什么？为什么要乘以1000

请求系统内存的10％的容器的OOM得分调整为900。如果容器Y中的进程使用了超过10％的内存，则其OOM得分将为1000。容器使用的内存比其请求多，那么OOM得分将为1000，并且将是OOM kill的主要目标。如果分数小于`1000 + guaranteedOOMScoreAdj`，也就是2分，那么被直接设置成2分，避免分数过低。

**参考文献**

* https://lwn.net/Articles/391222/
