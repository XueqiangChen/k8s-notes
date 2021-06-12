---
title: "[译]Kubernetes深入研究：CustomResources的代码生成"
date: 2021-03-12T10:57:25+08:00
draft: false
tags: ["kubernetes","codegen"]
categories: ["云计算"]
---

> 原文链接：[Kubernetes Deep Dive: Code Generation for CustomResources](https://www.openshift.com/blog/kubernetes-deep-dive-code-generation-customresources)

<!--more-->

CustomResourceDefinitions (CRDs)是kubernetes 1.7 中引入的，并在1.8中从alpha版本升级为beta版本了，最新版本的kubernetes 1.20 中，CRDs的版本已经是v1了。由于CRDs的易用性，在很多实现了控制器模式的例子中都可以看到它的身影。

在Kubernetes 1.8中， CRDs 在基于golang的项目中的使用也变得更加自然：通过用户提供的 CustomResources，我们可以利用在Kubernetes提供的代码生成工具来生成代码。这篇文章展示了代码生成器的工作方式，以及如何以最少的代码行将其应用到自己的项目中，为您提供了生成的Deepcopy函数，内置的clients，listers和informers，所有这些都带有一个Shell脚本调用和几个代码注释。

> 注：deepcopy，意为”深拷贝“，深拷贝意味着会重新生成对象并拷贝对象中的所有字段、地址等数据；浅拷贝仅仅是对象的引用，并没有生成新的对象。

## 为什么要使用代码生成？

那些在golang中原生使用ThirdPartyResources或CustomResourceDefinition的人可能会惊讶于突然在Kubernetes 1.8中需要生成client-go。更具体地说，client-go要求 runtime.Object 类型（golang中的CustomResources必须实现runtime.Object接口）必须具有DeepCopy方法。这里的代码生成通过deepcopy-gen生成器起作用，可以在[k8s.io/code-generator](https://github.com/kubernetes/code-generator)存储库中找到。

除了deepcopy-gen 生成器外，还有几个代码生成器是大多数CustomResources用户都想使用的：

* deepcopy-gen - 为每个T类型的方法创建`func (t* T) DeepCopy() *T` 函数
* client-gen - 为 CustomResource APIGroups 创建内置的客户端集
* informer-gen - 为CustomResources创建informer，该informer提供基于事件的界面以对服务器上CustomResources的更改做出反应
* lister-gen - 为CustomResources创建listers，该listers为GET和LIST请求提供只读缓存层。

最后两个是构建控制器（或Operators）的基础。在后续博客中，我们将更详细地介绍控制器。这四个代码生成器使用与Kubernetes上游控制器所使用的相同的机制和软件包，构成了构建功能齐全，可用于生产环境的控制器的强大基础。

`k8s.io/code-generator` 中还有其他用于其他上下文的生成器，例如，如果您构建自己的聚合API服务器，则除了版本化类型外，还将使用内部类型。Conversion-gen 将在这些内部和外部类型之间创建转换函数。Defaulter-gen将负责默认某些字段。

## 在你的项目中调用代码生成器

所有的Kubernetes代码生成器都是在[k8s.io/gengo](https://github.com/kubernetes/gengo)之上实现的。它们共享许多公共命令行标志。基本上，所有生成器都获得输入包`（--input-dirs）`的列表，它们通过类型进行检查，并输出生成的代码。生成的代码：

* 要么进入与输入文件相同的目录，例如deepcopy-gen （ `--output-file-base "zz_generated.deepcopy"`定义文件名）
* 或者它们生成一个或多个输出包(`--output-package`)例如 client-, informer- 和 lister-gen 所做的（通常将生成的代码放在 pkg/client 目录下）

k8s.io/code-generator附带了一个shell脚本`generator-group.sh`，这个脚本会帮我们处理这些繁重的工作，通常只要调用`hack/update-codegen.sh`就行，例如：

```bash
$ vendor/k8s.io/code-generator/generate-groups.sh all \
github.com/openshift-evangelist/crd-code-generation/pkg/client \ 
github.com/openshift-evangelist/crd-code-generation/pkg/apis \
example.com:v1
```

运行这个命令后生成的目录如下所示：

![](https://chenxqblog-1258795182.cos.ap-guangzhou.myqcloud.com/Screen-Shot-2017-10-16-at-17_58_28.webp)

所有的 APIs 都被创建到 `pkg/apis` 目录下，clientsets，informers 以及 listers 被创建到 `pkg/client` 这个目录下。总而言之，`pkg/client` 文件夹完全都是生成的，并且包括`zz_generated.deepcopy.go`这个文件。两者都不应该手动修改，而是通过运行以下命令创建：

```bash
$ hack/update-codegen.sh
```

这个脚本的[附近](https://github.com/kubernetes/code-generator/tree/master/hack)还有一个 `hack/verify-codegen.sh` 脚本，如果生成的任何文件不是最新的，该脚本都将以非零的返回码终止。这对于放入CI脚本中非常有帮助：如果开发人员无意中修改了文件，或者如果文件刚刚过时，CI会注意到并报错。

## 控制生成的代码–标签

如上所述，虽然代码生成器的某些行为是通过命令行标志（尤其是要处理的程序包）来控制的，但更多的属性是通过golang文件中的标签来控制的。

标签有两种：

* Global tags 相关的在`package`目录下的`doc.go`文件中
* Local tags 在它需要处理的类型中定义

标签通常是 `// +tag-name` 或者 `// +tag-name=value` 这两种形式，写在注释中。根据标签，注释的位置变得很重要。大部分的注释的标签应该直接标注在类型上（对于全局标签来说应该在`package`行中），其他的必须与类型分开，中间至少要有一条空线。

### GLOBAL TAGS

Global tags 被写入到包的 `doc.go` 文件中，一个典型的`pkg/apis/<apigroup>/<version>/doc.go`文件如下所示：

```go
// +k8s:deepcopy-gen=package,register
// Package v1 is the v1 version of the API.
// +groupName=example.com
package v1
```

它告诉deepcopy-gen默认为该包中的每种类型创建Deepcopy方法。如果你的类型中不需要生成deepcopy方法，可以使用local tag来关闭`// +k8s:deepcopy-gen=false`。如果你不在包的声明中使用生成 deepcopy 方法，就需要通过在每个类型的注释中定义 `// +k8s:deepcopy-gen=false`。

> 注意：上例中的值中的 `register` 关键字将使deepcopy方法注册到该scheme中。在Kubernetes 1.9中被取消了，因为该scheme将不再负责执行`runtime.Objects`的深层复制。取而代之的是调用 `yourobject.DeepCopy()` 或者 `yourobject.DeepCopyObject()`。
>
> Scheme定义了序列化和反序列化API对象的方法，用于将group、版本和类型信息转换为Go模式和从Go模式转换为Go模式的类型注册表，以及不同版本的Go模式之间的映射。

最后，`// +groupName=example.com` 定义了API 组的全限定名称。如果你写错了这个名称，client-gen 会生成错误的代码。另外这个标签必须在包上的注释块中。

### LACAL TAGS

本地标记要么直接写在API类型上方，要么写在它上方的第二个注释块中。如下的 `types.go` 文件所示：

```go
// +genclient

// +genclient:noStatus

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
// Database describes a database.
 type Database struct {
     metav1.TypeMeta `json:",inline"`
     metav1.ObjectMeta `json:"metadata,omitempty"`
     Spec DatabaseSpec `json:"spec"`
 }


// DatabaseSpec is the spec for a Foo resource
 type DatabaseSpec struct {
     User string `json:"user"`
     Password string `json:"password"`
     Encoding string `json:"encoding,omitempty"`
 }


// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object


// DatabaseList is a list of Database resources
 type DatabaseList struct {
     metav1.TypeMeta `json:",inline"`
     metav1.ListMeta `json:"metadata"`
     Items []Database `json:"items"`
 }
 
```

请注意，默认情况下，我们为所有类型启用了deepcopy，也就是说，可以选择不使用 deepcopy。但是，这些类型都是API类型，需要深度复制。因此，在此示例types.go中，我们不必打开或关闭deepcopy，而仅在`doc.go`中的程序包范围内即可。

### runtime.Object and DeepCopyObject

有一个特殊的 deepcopy 标签，需要更多说明：

```go
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
```

如果您尝试将CustomResources与基于Kubernetes 1.8的客户端一起使用，有些人可能已经很高兴，因为他们不小心出售了master分支的k8s.op/apimachinery，您遇到了由于CustomResource类型未实现runtime.Object而导致的编译器错误，因为未在您的类型上定义DeepCopyObject（）runtime.Object。原因是在1.8中，`runtime.Object`接口使用此方法签名进行了扩展，因此每个`runtime.Objec`t都必须实现`DeepCopyObject`。 `DeepCopyObject（）runtime.Object`的实现很简单：

```go
func (in *T) DeepCopyObject() runtime.Object {
    if c := in.DeepCopy(); c != nil {
        return c
    } else {
        return nil
    }
}
```

但幸运的是，您不必为每种类型都实现此功能，而只需将以下本地标记放在顶级API类型的上方：

```go
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
```

在上面的示例中，`Database` 和`DatabaseList`都是顶级类型，因为它们被用作runtime.Objects。根据经验，顶级类型是那些嵌入了`metav1.TypeMeta`的类型。同样，这些是客户端使用client-gen创建的类型。

请注意，`// + k8s：deepcopy-gen：interfaces` 标记可以并且也应该在定义具有某些接口类型的字段（例如，`field SomeInterface`）的API类型的情况下使用。然后`// + k8s：deepcopy-gen：interfaces=example.com/pkg/apis/example.SomeInterface`将导致`DeepeepSomeInterface（）SomeInterface`方法的生成。这允许它以类型正确的方式对这些字段进行深度复制。

### Client-gen 标签

最后，有许多标记可控制client-gen，在我们的示例中可以看到其中两个：

```go
// +genclient

// +genclient:noStatus

```

第一个标记告诉client-gen为该类型创建一个客户端（始终启用）。请注意，您不必将其放在API对象的列表类型上方。

第二个标记告诉client-gen该类型未通过`/status`子资源使用规范状态分隔。生成的客户端将没有`UpdateStatus`方法（client-gen一旦在您的结构中找到`Status`字段，就会盲目生成该方法）。` /status`子资源仅在1.8中才适用于本地（在golang中）实现的资源。但是，随着PR 913中为CustomResources讨论子资源，这种情况可能很快就会改变。

对于群集范围的资源，必须使用标签：

```go
// +genclient:nonNamespaced

```

对于特殊用途的客户端，您可能还希望详细控制客户端提供哪些HTTP方法。可以使用几个标签来完成此操作，例如：

```go
// +genclient:noVerbs

// +genclient:onlyVerbs=create,delete

// +genclient:skipVerbs=get,list,create,update,patch,delete,deleteCollection,watch

// +genclient:method=Create,verb=create,result=k8s.io/apimachinery/pkg/apis/meta/v1.Status

```

前三个应该是不言自明的，但是最后一个需要一些解释。上面写入此标记的类型将是仅创建的，并且不会返回API类型本身，而是metav1.Status。对于CustomResources来说，这没有多大意义，但是对于用golang编写的用户提供的API服务器，这些资源可以存在，并且实际上可以在OpenShift API中使用。

## 使用类型客户端的主要功能

尽管大多数基于Kubernetes 1.7和更早版本的示例都使用了`Client-go dynamic client` 作为CustomResources，但在很长一段时间内，本地Kubernetes API类型的类型化客户端都更加方便。在1.8版中进行了更改：如上所述，client-gen还为您的自定义类型创建了native，功能齐全且易于使用的类型化客户端。实际上，client-gen不知道您是将其应用于CustomResource类型还是native类型。
因此，使用此客户端与使用客户端Gober客户端完全等效。这是一个非常简单的示例：

```go
import (
    ...
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/tools/clientcmd"
    examplecomclientset "github.com/openshift-evangelist/crd-code-generation/pkg/client/clientset/versioned"

)

var (
 kuberconfig = flag.String("kubeconfig", "", "Path to a kubeconfig. Only required if out-of-cluster.")
 master = flag.String("master", "", "The address of the Kubernetes API server. Overrides any value in kubeconfig. Only required if out-of-cluster.")
 )


func main() {
 flag.Parse()


 cfg, err := clientcmd.BuildConfigFromFlags(*master, *kuberconfig)
 if err != nil {
 	glog.Fatalf("Error building kubeconfig: %v", err)
 }


 exampleClient, err := examplecomclientset.NewForConfig(cfg)
 if err != nil {
 	glog.Fatalf("Error building example clientset: %v", err)
 }


 list, err :=  exampleClient.ExampleV1().Databases("default").List(metav1.ListOpti ons{})
 if err != nil {
 	glog.Fatalf("Error listing all databases: %v", err)
 }


 for _, db := range list.Items {
 	fmt.Printf("database %s with user %q\n", db.Name, db.Spec.User)
 }
 }
 
```

它与kubeconfig文件一起使用，实际上可以与kubectl和Kubernetes客户端一起使用。

与动态客户端使用的旧版TPR或CustomResource代码相比，您无需进行类型转换。相反，实际的客户端调用看起来完全是本地的，它是：

```go
list, err := exampleClient.ExampleV1().Databases("default").List(metav1.ListOptions{})
```

在此示例中，结果是群集中所有数据库的`DatabaseList`。如果您将类型切换为集群范围（即没有命名空间；请不要忘记使用`// + genclientnonNamespaced`标记告诉client-gen！），调用将变成

```go
list, err := exampleClient.ExampleV1().Databases().List(metav1.ListOptions{})
```

### 以编程方式在GOLANG创建自定义资源

由于这个问题经常出现，因此请您谈谈如何从您的golang代码中以编程方式创建CRD的几句话。

客户代总是创建所谓的clinetsets。客户端集将一个或多个API组捆绑到一个客户端中。通常，这些API组来自一个存储库，并位于一个基本程序包中，例如，如本博文示例中的`pkg/apis`；对于Kubernetes，则来自`k8s.io/api`。

CustomResourceDefinitions由
[kubernetes/apiextensions-apiserver存储库](https://github.com/kubernetes/apiextensions-apiserver)。该API服务器（也可以独立启动）是由kube-apiserver嵌入的，因此CRD在每个Kubernetes群集上都可用。但是创建CRD的客户端会创建到apiextensions-apiserver存储库中，当然也要使用client-gen。阅读此博客后，您可以在`kubernetes/apiextensions-apiserver/tree/master/pkg/client`上找到客户端也不会感到惊讶，创建客户端实例以及如何创建CRD看起来也不奇怪：

```go
import (
    ...
    apiextensionsclientset "k8s.io/apiextensions-apiserver/pkg/client/clientset/clientset”

)
apiextensionsClient, err := apiextensionsclientset.NewForConfig(cfg)
 ...
 createdCRD, err := apiextensionsClient.ApiextensionsV1beta1().CustomResourceDefinitions().Create(yourCRD)
 
```



请注意，创建完成后，您将必须等待在新CRD上设置“已建立”条件。只有这样，kube-apiserver才会开始提供资源。如果您不等待该条件，则每次CR操作都会返回404 HTTP状态代码。