---
title: client-go入门到控制器实战
top_img: false
tags:
  - Kubernetes
  - client-go
  - Go
abbrlink: c7a1d9e4
date: 2026-04-08 09:10:00
updated:
categories:
keywords:
  - Kubernetes
  - client-go
  - Informer
  - WorkQueue
  - Controller
description: 从 client-go 的基本操作流程讲起，系统梳理 Informer、WorkQueue 和控制器模式，并给出一个可在两节点 playground 上实操的完整示例。
cover:
highlight_shrink:
---

# client-go入门到控制器实战

如果说前面的文章更多是在讲“怎么使用 Kubernetes”，那 `client-go` 这部分就开始进入另一个层面：**怎么用程序去操作 Kubernetes。**

很多人第一次接触 `client-go`，会觉得它有点绕：

- 为什么不能直接一直 `List`？
- `Watch` 和 `Informer` 到底差在哪？
- 为什么大家都在讲控制器模式？
- `WorkQueue` 又是干什么的？

这些问题如果分开看，会显得零碎；但如果把它们放到一个完整执行链路里，就会清楚很多。

这篇文章就按这个顺序来讲：

1. `client-go` 怎么连接 Kubernetes
2. 它怎么做增删改查
3. 为什么真实项目里不会停留在裸 `List/Watch`
4. `Informer` 内部到底做了什么
5. `WorkQueue` 为什么是控制器的标配
6. 一个可以在两节点 playground 上直接实践的小控制器案例

---

## 一、什么是 client-go

`client-go` 是 Kubernetes 官方提供的 Go 客户端库。

它的核心作用可以概括成一句话：

> 用 Go 程序去访问 Kubernetes API，并基于 Kubernetes 的对象模型构建自动化逻辑。

你可以用它做的事情包括：

- 查询 Pod、Deployment、Service 等资源
- 创建、更新、删除资源
- 监听资源变化
- 编写自定义控制器
- 实现 Operator 的基础能力

所以从定位上看：

- `kubectl` 是给人用的命令行工具
- `client-go` 是给程序用的客户端库

很多 Kubernetes 生态里的控制器，本质上都是在 `client-go` 之上构建出来的。

---

## 二、client-go 的基本操作链路

无论你最后是写一个简单工具，还是写一个完整控制器，入口基本都差不多：

```text
kubeconfig / in-cluster config
        ↓
rest.Config
        ↓
Clientset / Dynamic Client
        ↓
调用 Kubernetes API
```

最常见的是 `Clientset`。

你可以把它理解成一组按资源类型分类好的客户端集合，例如：

- `clientset.CoreV1().Pods(...)`
- `clientset.AppsV1().Deployments(...)`
- `clientset.BatchV1().Jobs(...)`

这类客户端是强类型的，写起来比较顺手，也更适合日常开发。

---

## 三、先从一个最小可用例子开始

先看一个最常见的例子：读取集群里的 Pod。

### 1. 依赖准备

初始化模块：

```bash
go mod init client-go-demo
```

依赖建议遵循一个原则：

> `client-go` 的主次版本尽量和你的 Kubernetes 集群小版本保持一致。

例如：

- 集群是 `1.33.x`
- 那通常就选 `client-go v0.33.x`

你在 playground 里练习时，可以先看一下：

```bash
kubectl version --short
```

再决定依赖版本。

---

### 2. 读取 kubeconfig 并列出 Pod

```go
package main

import (
	"context"
	"flag"
	"fmt"
	"path/filepath"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
)

func main() {
	var kubeconfig string
	if home := homedir.HomeDir(); home != "" {
		flag.StringVar(&kubeconfig, "kubeconfig", filepath.Join(home, ".kube", "config"), "path to kubeconfig")
	}
	flag.Parse()

	config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
	if err != nil {
		panic(err)
	}

	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}

	pods, err := clientset.CoreV1().Pods("default").List(context.Background(), metav1.ListOptions{})
	if err != nil {
		panic(err)
	}

	for _, pod := range pods.Items {
		fmt.Printf("pod=%s phase=%s node=%s\n", pod.Name, pod.Status.Phase, pod.Spec.NodeName)
	}
}
```

这个例子里最核心的两步是：

1. `clientcmd.BuildConfigFromFlags` 把 kubeconfig 转成 `rest.Config`
2. `kubernetes.NewForConfig` 基于这个配置创建 `Clientset`

有了 `clientset`，你就可以开始访问 Kubernetes 的各种资源。

---

## 四、client-go 常见操作

### 1. 查询资源

查询 Pod 列表：

```go
pods, err := clientset.CoreV1().Pods("default").List(ctx, metav1.ListOptions{})
```

查询单个 Deployment：

```go
deploy, err := clientset.AppsV1().Deployments("default").Get(ctx, "nginx", metav1.GetOptions{})
```

---

### 2. 创建资源

```go
import (
	corev1 "k8s.io/api/core/v1"
)

pod := &corev1.Pod{
	ObjectMeta: metav1.ObjectMeta{
		Name:      "demo-pod",
		Namespace: "default",
	},
	Spec: corev1.PodSpec{
		Containers: []corev1.Container{
			{
				Name:  "nginx",
				Image: "nginx:1.27",
			},
		},
	},
}

created, err := clientset.CoreV1().Pods("default").Create(ctx, pod, metav1.CreateOptions{})
```

---

### 3. 更新资源

先 `Get`，再改对象，再 `Update`：

```go
deploy, err := clientset.AppsV1().Deployments("default").Get(ctx, "nginx", metav1.GetOptions{})
if err != nil {
	panic(err)
}

var replicas int32 = 3
deploy.Spec.Replicas = &replicas

_, err = clientset.AppsV1().Deployments("default").Update(ctx, deploy, metav1.UpdateOptions{})
```

---

### 4. 删除资源

```go
err := clientset.CoreV1().Pods("default").Delete(ctx, "demo-pod", metav1.DeleteOptions{})
```

---

### 5. Watch 资源变化

```go
watcher, err := clientset.CoreV1().Pods("default").Watch(ctx, metav1.ListOptions{})
if err != nil {
	panic(err)
}

for event := range watcher.ResultChan() {
	fmt.Printf("type=%s obj=%T\n", event.Type, event.Object)
}
```

这时候你已经能监听资源变化了，但真实项目通常不会停在这里。

因为裸 `Watch` 虽然能用，但它还不够。

---

## 五、为什么不能只靠裸 List / Watch

先说结论：

> 在真正的控制器开发里，裸 `List/Watch` 只是基础能力，生产上更常见的模式是 `Informer + WorkQueue + Reconcile`。

原因主要有几个。

### 1. 直接 Watch 容易断

网络抖动、连接中断、超时、资源版本过期，这些情况都会让你自己管理 `Watch` 变得麻烦。

### 2. 你需要本地缓存

控制器通常会反复读取资源状态。如果每次都打 API Server：

- 压力大
- 延迟高
- 容易被限流

### 3. 你需要去重和削峰

资源更新可能非常频繁。如果每来一次事件就立即同步一次：

- 容易抖动
- 容易产生重复处理
- 容易把下游 API 打爆

### 4. 你需要失败重试

实际业务里，同步逻辑不会每次都成功。没有重试队列，控制器很快就会变得脆弱。

这也是为什么 Kubernetes 生态里，几乎所有经典控制器都遵循下面这条链路：

```text
List + Watch
    ↓
Reflector
    ↓
DeltaFIFO
    ↓
Informer 本地缓存
    ↓
事件处理器
    ↓
WorkQueue
    ↓
Worker
    ↓
Reconcile
```

---

## 六、Informer 到底是什么

可以先给一个工程化一点的定义：

> `Informer` 是建立在 `List/Watch` 之上的本地缓存与事件分发机制，它负责持续同步资源、维护缓存，并把变化事件通知给注册的处理器。

你平时最常见的，是 `SharedInformerFactory` 创建出来的一组共享 Informer。

例如：

```go
factory := informers.NewSharedInformerFactory(clientset, 30*time.Second)
podInformer := factory.Core().V1().Pods()
```

这里的 `30*time.Second` 是 resync period，它不是“每 30 秒重新 watch 一次”，而是：

- 在正常 watch 的基础上
- 周期性触发一次全量对象的同步机会

它更像是“重新把缓存里的对象过一遍事件处理逻辑”，而不是简单重连。

---

## 七、Informer 的几个关键组件

如果只记结论，可以记住下面这几个名字：

- `Reflector`
- `DeltaFIFO`
- `Indexer`
- `SharedIndexInformer`

它们各自负责的事情大致如下。

### 1. Reflector

`Reflector` 负责跟 API Server 打交道：

- 先 `List`
- 再 `Watch`
- 不断把最新变更推给下游

它解决的是“如何持续从 API Server 拿到资源变化”。

---

### 2. DeltaFIFO

`DeltaFIFO` 是一个带变更语义的队列。

它保存的不只是“这个对象来了”，而是：

- 新增
- 更新
- 删除
- 同步

也就是说，它更关注“对象发生了什么变化”，而不仅仅是对象本身。

---

### 3. Indexer

`Indexer` 本质上是带索引能力的本地缓存。

你可以把它理解成：

- 把最新对象缓存在本地
- 支持按 key 查
- 也支持按索引查

这样控制器大部分读操作就不需要每次都打 API Server 了。

---

### 4. SharedIndexInformer

这是真正日常开发最常打交道的对象。

它把前面几层能力封装起来，对外提供：

- 本地缓存
- 事件回调
- 下游 `Lister`

所以开发者平时更多是直接用：

- `Informer().AddEventHandler(...)`
- `Lister().Get(...)`

而不是自己手搓 `Reflector`。

---

## 八、Informer 的典型使用方式

通常你会这样写：

```go
factory := informers.NewSharedInformerFactory(clientset, 30*time.Second)
deploymentInformer := factory.Apps().V1().Deployments()

deploymentInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
	AddFunc: func(obj interface{}) {},
	UpdateFunc: func(oldObj, newObj interface{}) {},
	DeleteFunc: func(obj interface{}) {},
})

factory.Start(stopCh)
factory.WaitForCacheSync(stopCh)
```

这里有两个细节非常重要。

### 1. 先 `Start`，再 `WaitForCacheSync`

如果缓存还没同步完成，你就开始从 `Lister` 里读数据，很容易读到不完整状态。

### 2. 事件回调里不要做重活

这一点非常关键。

事件处理器最好的职责通常只有一个：

> 把对象 key 放进队列。

真正的业务处理，应该留给 worker。

不然你很快就会遇到：

- 事件阻塞
- 并发混乱
- 重试困难

---

## 九、为什么还需要 WorkQueue

很多初学者的第一个疑问是：

> Informer 都已经能收到事件了，为什么还要再多一层队列？

因为事件通知和业务处理不是一回事。

`WorkQueue` 解决的是下面这些问题：

- 削峰
- 去重
- 顺序消费
- 并发 worker 管理
- 失败重试
- 限速退避

换句话说：

- `Informer` 负责“感知变化”
- `WorkQueue` 负责“有节制地处理变化”

这也是控制器模式里非常经典的解耦。

---

## 十、WorkQueue 的核心思想

控制器里通常不会把整个对象直接丢进队列，而是把它的 key 丢进去：

```text
namespace/name
```

例如：

```text
default/nginx
```

这样做有几个好处：

- 队列元素更轻
- 相同 key 容易合并
- 真正处理时再从缓存里拿最新对象

这一点非常重要，因为事件到来时对象状态可能已经过时了，**真正可靠的是处理当下从缓存里读到的最新状态**。

---

## 十一、WorkQueue 的常见类型

`client-go` 里常见的队列主要有这几类。

### 1. 普通队列

最基础的 FIFO 队列。

适合简单异步处理，但没有额外的失败重试和限速能力。

### 2. DelayingQueue

支持延迟入队。

适合某些需要“过几秒再试一次”的场景。

### 3. RateLimitingQueue

这是控制器最常用的一种。

它支持：

- 出错后重试
- 指数退避
- 限制重试频率

你平时在控制器里最常看到的就是这一类。

---

## 十二、控制器里的标准处理流程

一个典型控制器通常长这样：

1. Informer 监听资源
2. 事件到来后，把 key 加进 `WorkQueue`
3. Worker 从队列里取 key
4. 通过 `Lister` 从缓存读取最新对象
5. 执行对比和协调逻辑，也就是 `Reconcile`
6. 成功则 `Forget`
7. 失败则 `AddRateLimited` 进入重试

这就是最经典的控制器模式。

所以很多文章里说：

> Kubernetes 控制器本质上是在不断做“期望状态”和“当前状态”的对比，然后把当前状态拉回期望状态。

真正落地到代码里，通常就是通过 `Informer + WorkQueue + Reconcile` 完成的。

---

## 十三、Informer 的真实使用场景

`Informer` 适合几乎所有“需要持续感知资源变化”的场景。

例如：

- 监听 Pod 变化，做调度辅助或状态聚合
- 监听 Deployment 变化，自动修正副本数
- 监听 ConfigMap / Secret 变化，触发配置刷新
- 监听自定义资源 CRD，做 Operator
- 监听 Node 状态，做节点治理

只要你的逻辑不是“一次性查完就结束”，而是：

> 资源变化后我要持续响应

那基本就应该优先考虑 Informer，而不是手写一个无限循环的 `List`。

---

## 十四、除了 Informer 和 WorkQueue，还要理解哪些关键点

如果你准备继续往控制器方向深入，下面这些点也很关键。

### 1. Lister

`Lister` 是 Informer 缓存的只读访问入口。

控制器里大部分读取都应该优先走 `Lister`，而不是每次直连 API Server。

---

### 2. cache key

最常见的 key 是：

```go
key, err := cache.MetaNamespaceKeyFunc(obj)
```

它会生成：

```text
namespace/name
```

这个 key 是控制器队列里最常见的元素格式。

---

### 3. Tombstone

删除事件有时拿到的不是完整对象，而是 `DeletedFinalStateUnknown`。

所以 `DeleteFunc` 里通常要兼容这种场景，否则删对象时容易 panic。

---

### 4. 幂等

控制器的同步逻辑必须尽量幂等。

原因很简单：

- 同一个 key 可能被重复处理
- resync 会重复触发
- 更新事件可能很多

如果你的逻辑每执行一次都会造成额外副作用，那控制器很快就会失控。

---

### 5. 最终一致性

Kubernetes 控制器不是事务系统，它追求的是最终一致性。

这意味着你不能假设：

- 一个事件只来一次
- 来了就立刻处理成功
- 当前读取一定是最新远端状态

你应该接受“可能重复、可能延迟、可能重试”，然后把逻辑写稳。

---

## 十五、实战案例：一个最小 Deployment 控制器

下面这个案例适合你在出差路上的 playground 里练习。

目标很简单：

> 监听某个命名空间下的 Deployment，如果它带有标签 `demo/min-replicas=true`，并且副本数小于 2，就自动把它修正为 2。

这个案例的好处是：

- 能完整用到 `Informer`
- 能完整用到 `WorkQueue`
- 逻辑足够简单
- 在两节点集群里非常容易验证

---

## 十六、项目依赖

你可以先初始化一个目录：

```bash
mkdir client-go-controller-demo
cd client-go-controller-demo
go mod init client-go-controller-demo
```

然后安装依赖：

```bash
go get k8s.io/client-go@与你集群匹配的版本
go get k8s.io/api@与你集群匹配的版本
go get k8s.io/apimachinery@与你集群匹配的版本
```

为了避免版本打架，实际操作里更建议直接统一到同一个 minor 版本。

---

## 十七、完整示例代码

下面给出一个单文件可运行示例：

```go
package main

import (
	"context"
	"flag"
	"fmt"
	"path/filepath"
	"time"

	apierrors "k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/informers"
	appslisters "k8s.io/client-go/listers/apps/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
	"k8s.io/client-go/util/workqueue"
)

type Controller struct {
	clientset kubernetes.Interface
	lister    appslisters.DeploymentLister
	queue     workqueue.TypedRateLimitingInterface[string]
}

func NewController(clientset kubernetes.Interface, factory informers.SharedInformerFactory) *Controller {
	deploymentInformer := factory.Apps().V1().Deployments()

	c := &Controller{
		clientset: clientset,
		lister:    deploymentInformer.Lister(),
		queue:     workqueue.NewTypedRateLimitingQueue(workqueue.DefaultTypedControllerRateLimiter[string]()),
	}

	deploymentInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    c.enqueue,
		UpdateFunc: func(oldObj, newObj interface{}) { c.enqueue(newObj) },
		DeleteFunc: c.enqueue,
	})

	return c
}

func (c *Controller) enqueue(obj interface{}) {
	key, err := cache.MetaNamespaceKeyFunc(obj)
	if err != nil {
		return
	}
	c.queue.Add(key)
}

func (c *Controller) Run(ctx context.Context, workers int) {
	defer c.queue.ShutDown()

	for i := 0; i < workers; i++ {
		go func() {
			for c.processNextItem(ctx) {
			}
		}()
	}

	<-ctx.Done()
}

func (c *Controller) processNextItem(ctx context.Context) bool {
	key, shutdown := c.queue.Get()
	if shutdown {
		return false
	}
	defer c.queue.Done(key)

	if err := c.sync(ctx, key); err != nil {
		fmt.Printf("sync failed for %s: %v\n", key, err)
		c.queue.AddRateLimited(key)
		return true
	}

	c.queue.Forget(key)
	return true
}

func (c *Controller) sync(ctx context.Context, key string) error {
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}

	deploy, err := c.lister.Deployments(namespace).Get(name)
	if err != nil {
		if apierrors.IsNotFound(err) {
			fmt.Printf("deployment %s no longer exists\n", key)
			return nil
		}
		return err
	}

	if deploy.Labels["demo/min-replicas"] != "true" {
		return nil
	}

	var current int32 = 1
	if deploy.Spec.Replicas != nil {
		current = *deploy.Spec.Replicas
	}

	const desired int32 = 2
	if current >= desired {
		fmt.Printf("deployment %s already satisfies replicas=%d\n", key, current)
		return nil
	}

	copy := deploy.DeepCopy()
	copy.Spec.Replicas = int32Ptr(desired)

	_, err = c.clientset.AppsV1().Deployments(namespace).Update(ctx, copy, metav1.UpdateOptions{})
	if err != nil {
		return err
	}

	fmt.Printf("deployment %s scaled from %d to %d\n", key, current, desired)
	return nil
}

func int32Ptr(v int32) *int32 {
	return &v
}

func main() {
	var kubeconfig string
	if home := homedir.HomeDir(); home != "" {
		flag.StringVar(&kubeconfig, "kubeconfig", filepath.Join(home, ".kube", "config"), "path to kubeconfig")
	}
	flag.Parse()

	config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
	if err != nil {
		panic(err)
	}

	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}

	factory := informers.NewSharedInformerFactory(clientset, 30*time.Second)
	controller := NewController(clientset, factory)

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	factory.Start(ctx.Done())
	factory.WaitForCacheSync(ctx.Done())

	fmt.Println("controller started")
	controller.Run(ctx, 2)
}
```

---

## 十八、这个示例里每一部分在做什么

为了避免代码看起来像一团，我们把关键路径拆开看。

### 1. `SharedInformerFactory`

```go
factory := informers.NewSharedInformerFactory(clientset, 30*time.Second)
```

它负责创建共享缓存和共享 watch 逻辑，避免同类资源每个组件都自己 watch 一遍。

---

### 2. `DeploymentInformer`

```go
deploymentInformer := factory.Apps().V1().Deployments()
```

它提供了两个最重要的能力：

- `Informer()`：注册事件处理器
- `Lister()`：从本地缓存读 Deployment

---

### 3. 事件处理函数

```go
deploymentInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
	AddFunc:    c.enqueue,
	UpdateFunc: func(oldObj, newObj interface{}) { c.enqueue(newObj) },
	DeleteFunc: c.enqueue,
})
```

这里没有直接处理业务，而是统一 `enqueue`。

这是控制器里非常重要的习惯：

- 事件回调尽量轻
- 真正逻辑放到 worker

---

### 4. 队列

```go
queue: workqueue.NewTypedRateLimitingQueue(...)
```

这个队列负责：

- 保存待处理 key
- 支持失败重试
- 支持限速退避

如果 `sync` 出错，就：

```go
c.queue.AddRateLimited(key)
```

如果成功，就：

```go
c.queue.Forget(key)
```

---

### 5. `Lister`

```go
deploy, err := c.lister.Deployments(namespace).Get(name)
```

这里从本地缓存读取最新 Deployment，而不是每次都直接打 API Server。

这也是 Informer 体系非常重要的价值之一。

---

### 6. Reconcile 逻辑

```go
if deploy.Labels["demo/min-replicas"] != "true" {
	return nil
}
```

只有命中特定标签的 Deployment 才处理。

然后判断副本数：

- 小于 2，就更新
- 大于等于 2，就跳过

这就是一个最小版的“声明式控制器”。

---

## 十九、在 playground 上怎么验证

先运行控制器：

```bash
go run main.go
```

然后另开一个终端，创建测试 Deployment：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  namespace: default
  labels:
    demo/min-replicas: "true"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
```

应用它：

```bash
kubectl apply -f deploy.yaml
```

然后观察：

```bash
kubectl get deploy nginx-demo -w
```

你应该能看到副本数从 `1` 被自动修正到 `2`。

这就说明你的整个链路已经跑通了：

- Deployment 变化被 Informer 感知
- 事件被放入 WorkQueue
- Worker 消费 key
- `sync` 执行协调逻辑
- Deployment 被更新

---

## 二十、这个案例能练到什么

虽然这个例子简单，但它已经覆盖了控制器最核心的骨架：

- kubeconfig 连接集群
- `Clientset` 调 Kubernetes API
- `SharedInformerFactory`
- `Lister`
- `WorkQueue`
- worker 并发处理
- 失败重试
- reconcile 思路

只要这套骨架你理解了，后面无论你写的是：

- Pod 控制器
- ConfigMap 控制器
- Secret 刷新器
- 自定义资源 Operator

本质上都是在这个模型上继续扩展。

---

## 二十一、开发 client-go 控制器时最容易踩的坑

### 1. 在事件回调里直接写业务逻辑

结果往往是：

- 事件阻塞
- 并发混乱
- 不好重试

正确思路是：**回调里只入队。**

---

### 2. 不等缓存同步完成就开始处理

如果 `WaitForCacheSync` 没做好，启动初期容易读到不完整状态。

---

### 3. 每次都直接请求 API Server

控制器读多写少，读取应该优先走缓存，真正更新再打 API。

---

### 4. 忽略删除事件的特殊对象

删除事件里可能拿到 tombstone，不做兼容容易 panic。

---

### 5. 逻辑不幂等

控制器天然会重复处理，所以同步逻辑必须能重复执行而不出错。

---

### 6. 忘记处理重试和 `Forget`

如果失败不 `AddRateLimited`，错误就直接丢了；如果成功后不 `Forget`，重试计数会一直累积。

---

## 二十二、什么时候该用 client-go，什么时候该看 controller-runtime

这个问题也很常见。

如果你的目标是：

- 真正理解 Kubernetes 控制器底层工作方式
- 学清楚 Informer、Lister、WorkQueue 的原理
- 写一些相对轻量的自定义控制器

那直接学 `client-go` 很有价值。

如果你的目标是：

- 更快开发 Operator
- 减少样板代码
- 使用更抽象的 Reconciler 模式

那后面通常会继续看 `controller-runtime`。

可以把两者关系理解成：

- `client-go` 更底层
- `controller-runtime` 更工程化、更高层

但不管你以后是不是会转向 `controller-runtime`，**把 `client-go` 这套骨架搞明白，收益都很大。**

---

## 二十三、路上练习时我建议你这样安排

既然你现在是在出差路上，用的是两节点 playground，那我建议练习顺序尽量短平快：

1. 先写一个只做 `List Pod` 的程序，确认 kubeconfig、依赖和访问链路没问题
2. 再写一个 `Watch Pod` 的程序，确认事件能持续收到
3. 然后切到 `Informer`，观察缓存同步和事件回调
4. 最后把 `WorkQueue` 和 worker 加上，跑我上面这个 Deployment 最小副本控制器

这样你每一步都能看到明确结果，不容易在路上被复杂细节拖住。

---

## 总结

`client-go` 真正难的地方，不是某个 API 调用本身，而是要理解它背后的控制器思维。

你可以把这篇文章压缩成一句话：

> 用 `client-go` 写 Kubernetes 自动化程序，最核心的模型就是：连接集群、感知资源变化、把变化放进队列、由 worker 做幂等协调。

对应到具体组件上，就是：

- `Clientset` 负责访问 API
- `Informer` 负责缓存和事件分发
- `Lister` 负责从缓存读取最新对象
- `WorkQueue` 负责削峰、重试和调度处理
- `Reconcile` 负责把当前状态拉回期望状态

只要这条主线你顺了，后面的控制器、Operator、自定义资源处理，其实都只是这套模式的不同变体。

如果你接下来准备继续练，我建议先把文中的最小 Deployment 控制器亲手跑通。因为一旦这个例子跑起来，你对 `Informer`、`WorkQueue` 和控制器模式的理解会立刻从“概念”变成“手感”。
