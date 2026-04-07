---
title: Pod生命周期、探针与资源限制
top_img: false
tags:
  - Kubernetes
  - Pod
abbrlink: f6a4b8c3
date: 2026-04-07 23:40:00
updated:
categories:
keywords:
  - Kubernetes
  - Pod
  - Probe
  - requests
  - limits
description: 讲清 Kubernetes 中 Pod 的生命周期、liveness/readiness/startup 探针，以及资源 requests/limits 的作用和常见问题。
cover:
highlight_shrink:
---

# Pod生命周期、探针与资源限制

如果说 `Deployment` 解决的是“我要跑几个副本”，`Service` 解决的是“应用怎么被访问”，那么 `Pod` 生命周期、健康探针和资源限制解决的就是另一个更贴近运行时的问题：

- 这个 Pod 现在处于什么阶段
- Kubernetes 怎么判断它是否健康
- 什么时候给它流量
- 什么时候重启它
- 它最多能用多少 CPU 和内存

这部分内容非常关键，因为很多线上问题并不是 YAML 写错了，而是：

- 应用启动慢，被错误地重启
- 明明容器进程还活着，但服务其实不可用
- Pod 被 Service 转发流量太早，导致大量 502 / 503
- 内存上限配太小，频繁 OOMKilled
- CPU limit 配得太死，导致响应变慢

所以这篇文章的重点不是只解释概念，而是把这些机制和真实运行问题对应起来。

---

## 一、Pod 生命周期是什么

官方文档对 Pod 生命周期的定义很明确：Pod 是相对短暂的对象，它从创建开始，会经历一组状态变化，直到最终结束。

一个 Pod 的生命周期中，最常见的阶段（phase）有：

- `Pending`
- `Running`
- `Succeeded`
- `Failed`
- `Unknown`

这些阶段是 Pod 的大致状态，不是容器的全部细节，但足够帮助我们快速判断大方向。

---

## 二、Pod 的几个常见阶段

### 1. Pending

`Pending` 表示 Pod 已经被创建，但还没有真正运行起来。

常见原因包括：

- 还在调度节点
- 镜像还没拉取完成
- PVC 还没绑定成功
- 节点资源不足
- 依赖的初始化流程还没完成

也就是说，`Pending` 不一定是“有故障”，它可能只是还在准备阶段；但如果长时间 Pending，就通常说明调度或资源有问题。

---

### 2. Running

`Running` 表示 Pod 已经被调度到某个节点上，并且至少有一个主容器已经启动成功。

但要注意：

> `Running` 不等于“业务已经可用”。

这点特别重要。很多初学者看到 Pod 是 Running，就以为应用已经完全正常了，其实不一定。因为：

- 容器进程可能只是启动了
- 应用可能还没完成初始化
- 数据库连接可能还没建立
- 缓存可能还没预热
- readiness probe 可能仍未通过

所以在生产环境里，判断是否可接收流量，不能只看 `Running`，还要看 `Ready` 状态和探针结果。

---

### 3. Succeeded

`Succeeded` 一般出现在一次性任务中，例如：

- Job 执行完成
- 数据迁移脚本执行成功
- 批处理任务正常退出

含义是：

- Pod 中所有容器都已经正常结束
- 并且不会再重启

---

### 4. Failed

`Failed` 表示 Pod 中至少有一个容器异常结束，且不会再恢复到正常运行状态。

常见场景：

- Job 执行失败
- 容器异常退出且不再重启
- 节点层面发生严重异常

---

### 5. Unknown

`Unknown` 表示控制平面无法确定 Pod 当前状态，通常和节点通信异常有关。

这个状态不算常见，但出现时通常要优先关注：

- 节点是否宕机
- kubelet 是否异常
- 网络是否有问题

---

## 三、Pod phase 和 Pod condition 不是一回事

这是一个很容易混淆的点。

- `phase` 是大致阶段
- `condition` 是更细粒度的条件状态

官方文档中，常见 Pod condition 包括：

- `PodScheduled`
- `Initialized`
- `ContainersReady`
- `Ready`

在较新的 Kubernetes 版本中，还可以看到：

- `PodReadyToStartContainers`

你可以这样理解：

- `Running` 是“Pod 起来了”
- `Ready` 是“Pod 可以接收业务流量了”

实际排查时，经常会看到：

- Pod 是 `Running`
- 但 `READY` 列显示 `0/1`

这通常就意味着：

- 容器活着
- 但还没准备好处理请求

---

## 四、为什么需要探针

Kubernetes 不会自动知道你的应用什么时候真的可用，它只知道：

- 容器进程是否存在
- 某个命令是否执行成功
- 某个端口或 HTTP 接口是否返回正常

所以它引入了探针（Probe）机制，让 kubelet 周期性检查容器状态。

官方文档把探针分成三类：

- `livenessProbe`
- `readinessProbe`
- `startupProbe`

这三者解决的是三个不同的问题。

---

## 五、livenessProbe：活着吗

`livenessProbe` 用来判断：

> 这个容器是不是“虽然进程还在，但已经卡死或失去继续工作的能力”？

如果 liveness probe 持续失败，kubelet 会重启容器。

典型使用场景：

- 应用发生死锁
- 主线程卡住
- 进程没有退出，但已经无法提供正常服务

例如一个 Java / Go / Python 服务，进程没有挂，但内部已经僵死，这时候 liveness probe 就能帮助 kubelet 把它拉起来。

### 一个常见误区

不是所有服务都必须配置 liveness probe。

如果你的程序在出现严重问题时本来就会自己崩溃退出，那么 kubelet 会根据重启策略自动处理。这种情况下，liveness probe 不一定是必须的。

更重要的是：

**不要把 liveness probe 配得过于激进。**

否则可能出现：

- 应用只是暂时慢了一下
- 探针失败
- kubelet 误以为它死了
- 直接重启容器

结果反而让服务更不稳定。

---

## 六、readinessProbe：准备好接流量了吗

`readinessProbe` 用来判断：

> 这个容器是否已经准备好接收请求？

如果 readiness probe 失败，Kubernetes 不会重启容器，但会把这个 Pod 从 Service 的后端列表里摘掉。

这点很关键：

- liveness 失败，容器会被重启
- readiness 失败，容器不一定重启，只是不再接流量

这就是 readiness probe 的真正价值：**流量控制**。

典型场景：

- 应用刚启动，还没完成初始化
- 数据库暂时不可用
- 依赖服务异常
- 应用过载，暂时不想继续接流量

在这些场景下，readiness probe 可以防止流量打到一个“虽然还活着，但当前不适合服务请求”的实例上。

---

## 七、startupProbe：启动完成了吗

`startupProbe` 是为“启动很慢的应用”准备的。

官方文档明确说明：如果配置了 startup probe，那么在它成功之前，liveness 和 readiness 都会被禁用。

这意味着：

- Kubernetes 会先等启动探针通过
- 通过之前，不会急着做 liveness / readiness 检查

这非常适合：

- 启动时间长的应用
- 需要加载大量配置的应用
- 需要执行迁移或预热的应用

它解决的是一个经典问题：

> 应用其实只是启动慢，还没来得及起来，就被 liveness probe 提前判死了。

所以如果你的服务启动时间明显长于正常探针周期，应该优先考虑 startup probe，而不是一味把 liveness 的延迟拉得特别长。

---

## 八、三种探针怎么配合

可以这样记：

- `startupProbe`：先确认“启动完成了没有”
- `readinessProbe`：再确认“能不能接流量”
- `livenessProbe`：最后持续确认“有没有卡死，需要不需要重启”

一个比较稳妥的思路是：

- 启动慢的应用，加 `startupProbe`
- 大多数对外服务，加 `readinessProbe`
- 明确存在“卡死不退出”风险的应用，再加 `livenessProbe`

不要一上来三种全配，而且参数都照抄。探针配置一定要结合应用启动耗时、健康检查接口和故障模型来调。

---

## 九、探针支持哪些检查方式

官方文档中，探针支持四种机制：

- `httpGet`
- `tcpSocket`
- `exec`
- `grpc`

最常见的还是前三种。

### 1. httpGet

最常用。让 kubelet 请求容器内的某个 HTTP 路径，例如：

- `/healthz`
- `/ready`
- `/live`

适合大多数 Web 服务。

---

### 2. tcpSocket

检查某个端口是否能建立 TCP 连接。

优点是简单，缺点是只能说明端口开着，不能说明业务真的健康。

---

### 3. exec

在容器里执行一个命令，退出码为 0 就认为成功。

适合：

- 检查本地文件
- 检查进程状态
- 执行简单健康脚本

---

### 4. grpc

较新的 Kubernetes 已支持 gRPC 探针。适合基于 gRPC 健康检查协议的服务。

如果你的系统本身是 gRPC 服务，这种方式会更自然。

---

## 十、探针里最常用的几个参数

探针常见字段包括：

- `initialDelaySeconds`
- `periodSeconds`
- `timeoutSeconds`
- `successThreshold`
- `failureThreshold`

它们的含义分别是：

- `initialDelaySeconds`：容器启动后，延迟多久才开始探测
- `periodSeconds`：探测间隔
- `timeoutSeconds`：单次探测超时时间
- `successThreshold`：连续成功多少次才算成功
- `failureThreshold`：连续失败多少次才算失败

一个非常实用的理解方式是：

```text
最终容器会不会被判失败，不是看一次检查，而是看“多久、失败几次、超时多久”
```

所以探针不是开关，而是一组时间策略。

---

## 十一、一个常见的探针配置示例

下面是一个比较典型的 HTTP 服务探针配置：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blog-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: blog-api
  template:
    metadata:
      labels:
        app: blog-api
    spec:
      containers:
      - name: blog-api
        image: registry.example.com/blog-api:v1
        ports:
        - containerPort: 8080
        startupProbe:
          httpGet:
            path: /healthz
            port: 8080
          failureThreshold: 30
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 5
          timeoutSeconds: 2
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 2
          failureThreshold: 3
```

这个配置表达的语义是：

- 启动阶段先看 `/healthz`，允许较长启动时间
- 就绪阶段看 `/ready`，只有通过后才接流量
- 运行过程中持续检查 `/healthz`，失败多次才重启

这比只配一个 liveness probe 更符合真实服务的运行逻辑。

---

## 十二、资源 requests 和 limits 是什么

除了探针，另一个极其重要的部分是资源限制。

官方文档把这两个概念区分得很清楚：

- `requests`：容器至少需要多少资源
- `limits`：容器最多能用多少资源

最常见的资源是：

- `cpu`
- `memory`

你可以简单理解为：

- `request` 主要影响调度
- `limit` 主要影响运行时约束

---

## 十三、requests 的作用：调度时看它

当你创建 Pod 时，scheduler 会看每个节点还有多少可分配资源，然后根据 Pod 的 `requests` 决定这个 Pod 能不能调度上去。

例如你声明：

```yaml
requests:
  cpu: "250m"
  memory: "256Mi"
```

意思是：

- 至少需要 0.25 个 CPU
- 至少需要 256Mi 内存

如果节点剩余可分配资源不够，Pod 就可能一直停留在 `Pending`。

这也是为什么很多 Pod 明明“机器看起来还很空”，却调度不上去。因为调度器看的是 `requests` 和 allocatable，不是你肉眼看见的瞬时使用率。

---

## 十四、limits 的作用：运行时看它

`limits` 表示容器运行时允许使用的最大资源。

例如：

```yaml
limits:
  cpu: "500m"
  memory: "512Mi"
```

意思是：

- CPU 最多用到 0.5 核
- 内存最多用到 512Mi

官方文档指出：

- CPU limit 通过限流实现
- memory limit 通常通过 OOM kill 的方式体现，属于“事后强制”

这两个行为差别很大：

### CPU 超 limit

通常不会把容器直接杀掉，而是被限流，表现为：

- 请求变慢
- 延迟变高
- 吞吐下降

### Memory 超 limit

则可能直接触发 OOMKilled，表现为：

- 容器被杀
- Pod 重启
- 服务不稳定

这也是为什么内存 limit 的配置要比 CPU 更谨慎。

---

## 十五、如果只写 limit，不写 request，会怎样

官方文档明确说明：

> 如果你给某种资源设置了 limit，但没有设置 request，并且没有别的默认机制补上 request，那么 Kubernetes 会把 limit 的值复制一份作为 request。

这意味着：

- 你本来只是想限制上限
- 结果调度时也按这个值占坑了

例如：

```yaml
limits:
  memory: "1Gi"
```

如果没有 request，Kubernetes 可能把 request 也当成 `1Gi`。这样就会导致调度比你预期更保守。

所以比较好的习惯是：

- `requests` 明确写
- `limits` 也明确写

而不是只写一个。

---

## 十六、一个典型的资源配置示例

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

可以这样理解：

- 平时至少给我保证 0.1 核和 128Mi
- 高峰时我最多可以吃到 0.5 核
- 内存不能超过 512Mi

这种配置很常见，适合普通中小型 API 服务作为起步值。

当然，真实数值还是要根据：

- 压测结果
- 监控数据
- GC 特征
- 峰值请求

来调整。

---

## 十七、为什么资源配置不能乱写

资源配置最常见的两种错误是：

### 1. request 配太大

后果：

- 调度困难
- 节点资源利用率低
- Pod 长时间 Pending

### 2. limit 配太小

后果：

- CPU 被严重限流
- 内存频繁 OOMKilled
- 服务看起来“偶发性抽风”

所以资源参数的本质不是“随便给个值”，而是对应用资源画像的表达。

---

## 十八、QoS 类别是什么

Kubernetes 会根据 Pod 的 requests 和 limits，把 Pod 分成不同 QoS（Quality of Service）类别。

官方文档给出的三类是：

- `Guaranteed`
- `Burstable`
- `BestEffort`

它们影响的重要一点是：

> 节点资源紧张时，哪些 Pod 更容易被驱逐（evict）

### 1. Guaranteed

最稳定。通常要求每个容器都同时设置 CPU 和内存的 request / limit，且 request 必须等于 limit。

### 2. Burstable

最常见。设置了 request 和/或 limit，但不满足 Guaranteed 条件。

### 3. BestEffort

什么资源都没配。最不稳定，资源紧张时最容易先被驱逐。

这就是为什么生产环境一般不建议完全不配资源限制。

---

## 十九、一个更完整的 Deployment 示例

下面这个例子把探针和资源限制放在一起：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
      - name: api-server
        image: registry.example.com/api-server:v1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "512Mi"
        startupProbe:
          httpGet:
            path: /healthz
            port: 8080
          periodSeconds: 5
          failureThreshold: 24
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          periodSeconds: 5
          timeoutSeconds: 2
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          periodSeconds: 10
          timeoutSeconds: 2
          failureThreshold: 3
```

这个例子表达了几个重要思路：

- 用 `startupProbe` 容忍应用冷启动
- 用 `readinessProbe` 控制什么时候接流量
- 用 `livenessProbe` 防止应用卡死
- 用 requests 保障基础资源
- 用 limits 防止单个容器无限吃资源

这就是一个比较像样的生产起点。

---

## 二十、排查这类问题时常用的 kubectl 命令

如果你怀疑 Pod 生命周期、探针或资源限制有问题，最常用的命令通常是：

```bash
kubectl get pods
kubectl get pods -o wide
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl top pod
kubectl top node
kubectl get events --sort-by=.metadata.creationTimestamp
```

重点看什么：

- `describe pod` 里的探针失败事件
- `OOMKilled`
- `Back-off restarting failed container`
- `FailedScheduling`
- 当前 CPU / 内存使用量

如果 `kubectl describe pod` 中出现类似：

```text
Readiness probe failed
Liveness probe failed
OOMKilled
```

那么排查方向就已经比较明确了。

---

## 二十一、初学者最容易踩的坑

### 1. 把 Running 当成可用

错误理解：

- Pod Running 了，就说明服务没问题

正确理解：

- Running 只表示至少有容器启动成功
- 真正是否可接流量，要看 Ready 和 readiness probe

### 2. 启动慢却没配 startupProbe

结果：

- 应用还没启动完成
- liveness probe 先判失败
- 容器被反复重启

### 3. readiness 和 liveness 共用一个很脆弱的接口

结果：

- 一次短暂抖动
- 不仅被摘流量，还可能被重启

### 4. 完全不配 requests / limits

结果：

- 调度不可控
- 节点压力下行为不可预测
- 最容易变成 BestEffort

### 5. memory limit 配太小

结果：

- OOMKilled
- 服务频繁重启

---

## 二十二、实战中的建议

如果你在做一个普通 Web API 服务，比较稳妥的实践通常是：

- 至少配 `readinessProbe`
- 启动较慢时加 `startupProbe`
- 只有明确需要时再加 `livenessProbe`
- 一定写 `requests`
- `memory limit` 不要瞎压
- 用监控和压测数据来回调资源参数

如果你现在还没有成熟监控，至少先做到：

- 观察应用平时的 CPU / 内存占用
- 看启动需要多久
- 看高峰期是否有明显波动

这些数据比“抄一份网上 YAML”更有价值。

---

## 总结

Pod 生命周期、健康探针和资源限制，决定了 Kubernetes 上的应用是不是“真的稳定运行”。

你可以把这三部分这样串起来理解：

- 生命周期回答“Pod 当前处在哪个阶段”
- 探针回答“它现在健康吗、能接流量吗、要不要重启”
- 资源限制回答“它需要多少资源、最多能用多少资源”

对初学者来说，真正要掌握的不是把所有字段背下来，而是理解这几个机制背后的运行逻辑：

- 为什么 Running 不等于 Ready
- 为什么 readiness 影响流量而不是重启
- 为什么 startup probe 能保护慢启动应用
- 为什么 request 影响调度，limit 影响运行时行为
- 为什么内存超限比 CPU 超限更危险

当这些点都理顺之后，再去看 Deployment YAML，你就不会只是在“填配置项”，而是真正理解这些字段在控制应用如何运行。
