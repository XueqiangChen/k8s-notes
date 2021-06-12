---
title: "K8s Go 客户端浅析"
date: 2021-04-06T19:58:51+08:00
draft: false
tags: ["kubernetes","client-go"]
categories: ["云计算"]
---

在使用 [Kubernetes REST API](https://kubernetes.io/zh/docs/reference/using-api/) 编写应用程序时， 您并不需要自己实现 API 调用和 “请求/响应” 类型。 您可以根据自己的编程语言需要选择使用合适的客户端库。

客户端库通常为您处理诸如身份验证之类的常见任务。 如果 API 客户端在 Kubernetes 集群中运行，大多数客户端库可以发现并使用 Kubernetes 服务帐户进行身份验证， 或者能够理解 [kubeconfig 文件](https://kubernetes.io/zh/docs/tasks/access-application-cluster/configure-access-multiple-clusters/) 格式来读取凭据和 API 服务器地址。

kubernetes 官方支持的客户端有 go/python/java/dotnet/js 等，今天我们要讨论的是其中的 [go 客户端](https://github.com/kubernetes/client-go/)。

首先下载源代码，进入到`examples`目录：

```bash
➜ git clone git@github.com:kubernetes/client-go.git
➜  ~ cd client-go/examples
➜  examples git:(master) ✗ ll
total 32K
-rwxr-xr-x 1 root root 2.0K Apr  6 17:33 README.md
drwxr-xr-x 2 root root 4.0K Apr  6 17:46 create-update-delete-deployment
drwxr-xr-x 2 root root 4.0K Apr  6 17:33 dynamic-create-update-delete-deployment
drwxr-xr-x 2 root root 4.0K Apr  6 17:33 fake-client
drwxr-xr-x 2 root root 4.0K Apr  6 17:33 in-cluster-client-configuration
drwxr-xr-x 2 root root 4.0K Apr  6 17:33 leader-election
drwxr-xr-x 2 root root 4.0K Apr  6 17:33 out-of-cluster-client-configuration
drwxr-xr-x 2 root root 4.0K Apr  6 17:33 workqueue
```

go 客户端提供了很多的使用样例，我们重点来看一下[create-update-delete-deployment](https://github.com/kubernetes/client-go/tree/master/examples/create-update-delete-deployment) 这个目录，这个目录中是关于操作deployment资源对象的样例，包括创建，查询，更新，删除。

> This example program demonstrates the fundamental operations for managing on [Deployment](https://kubernetes.io/docs/user-guide/deployments/) resources, such as `Create`, `List`, `Update` and `Delete`.
>
> You can adopt the source code from this example to write programs that manage other types of resources through the Kubernetes API.

我们从 `main` 函数开始看起，这里尝试使用 `go-callvis` 将整个调用关系可视化出来：

![](https://chenxqblog-1258795182.cos.ap-guangzhou.myqcloud.com/frame_generic_dark.png)

总的来看，`main` 使用 `kubeconfig` 的配置来生成 rest client，通过 rest client 调用 k8s api 进行资源的操作。

接下来，具体来看一下读取 `kubeconfig` 配置的代码：

```go
var kubeconfig *string
if home := homedir.HomeDir(); home != "" {
    kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file")
} else {
    kubeconfig = flag.String("kubeconfig", "", "absolute path to the kubeconfig file")
}
flag.Parse()

config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
```

命令行参数指定`kubeconfig`的绝对路径，调用`clientcmd.BuildConfigFromFlags("", *kubeconfig)`来解析 config 的配置信息。 `BuildConfigFromFlags` 函数是用来从master地址或者kubeconfig文件地址来构建config配置，这个函数做了两件事情：

1. 调用`func NewNonInteractiveDeferredLoadingClientConfig(loader ClientConfigLoader, overrides *ConfigOverrides) ClientConfig`
2. 调用 `func (config *DeferredLoadingClientConfig) ClientConfig() (*restclient.Config, error)` 函数

```go
func BuildConfigFromFlags(masterUrl, kubeconfigPath string) (*restclient.Config, error) {
    ......
    return NewNonInteractiveDeferredLoadingClientConfig(
        &ClientConfigLoadingRules{ExplicitPath: kubeconfigPath},
        &ConfigOverrides{ClusterInfo: clientcmdapi.Cluster{Server: masterUrl}}).ClientConfig()
}
```

我们来看看这两个链式函数做了什么事情：

```go
func NewNonInteractiveDeferredLoadingClientConfig(loader ClientConfigLoader, overrides *ConfigOverrides) ClientConfig {
	return &DeferredLoadingClientConfig{loader: loader, overrides: overrides, icc: &inClusterClientConfig{overrides: overrides}}
}
```

`NewNonInteractiveDeferredLoadingClientConfig` 函数返回一个 `ClientConfig` 接口的实现: `DeferredLoadingClientConfig`。`DeferredLoadingClientConfig` 主要工作是确保装载的 `Config` 实例使用的是最新 `kubeconfig` 数据（对于配置了多个集群的，`export KUBECONFIG=cluster1-config:cluster2-config`，需要执行 merge）。在这个结构体的注释中写道：

```
It is used in cases where the loading rules may change after you've instantiated them and you want to be sure that the most recent rules are used.  This is useful in cases where you bind flags to loading rule parameters before the parse happens and you want your calling code to be ignorant of how the values are being mutated to avoid passing extraneous information down a call stack

实例化加载规则后，如果要更改加载规则，并且要确保使用最新的规则，则可以使用它。这在以下情况下很有用：
在解析发生之前将标志绑定到加载规则参数，并且您希望您的调用代码不知道值的变化方式，以避免在调用堆栈中传递无关的信息
```

上一个函数返回了 ClientConfig 接口实例。然后调用 ClientConfig 接口定义的 ClientConfig() 方法。ClientConfig() 工作是解析、处理 kubeconfig 文件里的认证信息，并返回一个完整的 rest#Config 实例。

```go
// ClientConfig implements ClientConfig
func (config *DeferredLoadingClientConfig) ClientConfig() (*restclient.Config, error) {
	mergedClientConfig, err := config.createClientConfig()
	......
    
	// load the configuration and return on non-empty errors and if the
	// content differs from the default config
	mergedConfig, err := mergedClientConfig.ClientConfig()
	......

	// check for in-cluster configuration and use it
	if config.icc.Possible() {
		klog.V(4).Infof("Using in-cluster configuration")
		return config.icc.ClientConfig()
	}

	// return the result of the merged client config
	return mergedConfig, err
}
```

这个函数主要有两个重要部分：

1.`mergedClientConfig, err := config.createClientConfig()`

内部执行遍历 kubeconfig files （如果有多个）， 对每个 kubeconfig 执行 LoadFromFile 返回 tools/clientcmd/api#Config 实例。api#Config 顾名思义 api 包下的 Config，是把 kubeconfig （eg. $HOME/.kube/config） 序列化为一个 API 资源对象。

现在,我们看到了几种结构体或接口命名相似，不要混淆了：

- api#Config：序列化 kubeconfig 文件后生成的对象
- tools/clientcmd#ClientConfig：负责用 api#Config 真正创建 rest#Config。处理、解析 kubeconfig 中的认证信息，有了它才能创建 rest#Config，所以命名叫 **Client**Config
- rest#Config：用于创建 http 客户端

对于 merge 后的 api#Config，调用 NewNonInteractiveClientConfig 创建一个 ClientConfig 接口的实现。

2.`mergedConfig, err := mergedClientConfig.ClientConfig()`

真正创建 rest#Config 的地方。在这里解析、处理 kubeconfig 中的认证信息。

完成 rest client 创建之后，就需要创建 `clientset` :

```go
clientset, err := kubernetes.NewForConfig(config)
```

这里的 `func NewForConfig(c *rest.Config) (*Clientset, error)` 函数会根据 api group 为不同的资源生成客户端。例如这里用到的 deployment 的客户端，定义了deployment的namespace：

```go
deploymentsClient := clientset.AppsV1().Deployments(apiv1.NamespaceDefault)
```

接下来创建 deployment 的描述信息：

```go
deployment := &appsv1.Deployment{
    ObjectMeta: metav1.ObjectMeta{
        Name: "demo-deployment",
    },
    Spec: appsv1.DeploymentSpec{
        Replicas: int32Ptr(2),
        Selector: &metav1.LabelSelector{
            MatchLabels: map[string]string{
                "app": "demo",
            },
        },
        Template: apiv1.PodTemplateSpec{
            ObjectMeta: metav1.ObjectMeta{
                Labels: map[string]string{
                    "app": "demo",
                },
            },
            Spec: apiv1.PodSpec{
                Containers: []apiv1.Container{
                    {
                        Name:  "web",
                        Image: "nginx:1.12",
                        Ports: []apiv1.ContainerPort{
                            {
                                Name:          "http",
                                Protocol:      apiv1.ProtocolTCP,
                                ContainerPort: 80,
                            },
                        },
                    },
                },
            },
        },
    },
}
```

创建deployment：

```go
// Create Deployment
fmt.Println("Creating deployment...")
result, err := deploymentsClient.Create(context.TODO(), deployment, metav1.CreateOptions{})
if err != nil {
   panic(err)
}
fmt.Printf("Created deployment %q.\n", result.GetObjectMeta().GetName())
```

这个创建的函数实际上就是调用 HTTP 的 POST 请求来跟K8s集群进行通信的。

```go
func (c *deployments) Create(ctx context.Context, deployment *v1.Deployment, opts metav1.CreateOptions) (result *v1.Deployment, err error) {
	result = &v1.Deployment{}
	err = c.client.Post().
		Namespace(c.ns).
		Resource("deployments").
		VersionedParams(&opts, scheme.ParameterCodec).
		Body(deployment).
		Do(ctx).
		Into(result)
	return
}
```

至于post的请求细节，这里不做详细的阐述。

**参考：**

[k8s informer](https://mp.weixin.qq.com/s?__biz=MzkzMzE2ODg1MQ==&mid=2247489358&idx=1&sn=7045014044efa1767d6c88340c2d398f&source=41#wechat_redirect)

[Kubernetes: Controllers, Informers, Reflectors and Stores](http://borismattijssen.github.io/articles/kubernetes-informers-controllers-reflectors-stores)

[解读 kubernetes client-go 官方 examples](https://segmentfault.com/a/1190000018953168)](https://segmentfault.com/a/1190000018953168)

[client-go学习](https://qiankunli.github.io/2020/07/20/client_go.html#%E7%AE%80%E4%BB%8B)

[go-callvis](https://github.com/ofabry/go-callvis)：callvis 查看代码调用关系工具，一个不错的调用链可视化工具，可以方便我们分析代码