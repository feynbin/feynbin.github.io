---
title: kubectl常用命令与YAML生成
top_img: false
tags:
  - Kubernetes
  - kubectl
abbrlink: e5f9a3b2
date: 2026-04-07 23:10:00
updated:
categories:
keywords:
  - kubectl
  - Kubernetes
  - YAML
  - bash completion
description: 梳理常用 kubectl 命令、如何快速生成基础 YAML，以及 kubectl 的 Bash 自动补全配置方法。
cover:
highlight_shrink:
---

# kubectl常用命令与YAML生成

学习 Kubernetes 的过程中，`kubectl` 是绕不过去的核心工具。它既是我们和集群交互的命令行入口，也是日常排查、创建资源、查看状态、调试应用的基础工具。

很多初学者一开始会遇到两个问题：

- 命令太多，不知道哪些最常用
- YAML 太长，每次从头手写很容易出错

其实 `kubectl` 本身就提供了很多很好用的能力：

- 查看资源状态
- 创建基础资源
- 通过命令快速生成 YAML
- 导出对象定义
- 配置自动补全，提高输入效率

这篇文章就把这些最常用的内容串起来。

---

## 一、kubectl 是什么

`kubectl` 是 Kubernetes 官方提供的命令行客户端，用来和集群中的 API Server 通信。

你可以用它做几乎所有日常操作：

- 查看集群信息
- 创建、更新、删除资源
- 查看日志
- 进入容器
- 排查故障
- 导出 YAML

可以把它理解成：

> 如果说 Kubernetes 是整个控制系统，那么 `kubectl` 就是你最常用的操作面板。

---

## 二、最常用的 kubectl 命令

下面这些命令，是平时最常用也最值得优先记住的。

### 1. 查看集群与节点

```bash
kubectl cluster-info
kubectl get nodes
kubectl get nodes -o wide
```

作用：

- `cluster-info` 查看集群控制平面信息
- `get nodes` 查看节点列表
- `-o wide` 输出更多字段，比如节点 IP

---

### 2. 查看资源列表

```bash
kubectl get pods
kubectl get svc
kubectl get deploy
kubectl get ing
kubectl get pvc
kubectl get all
```

常见写法：

```bash
kubectl get pods -A
kubectl get pods -n blog-prod
kubectl get pods -o wide
kubectl get pods --show-labels
```

说明：

- `-A` 表示所有命名空间
- `-n` 指定命名空间
- `get all` 不是“真的所有资源”，只是常见核心资源

---

### 3. 查看资源详情

```bash
kubectl describe pod <pod-name>
kubectl describe deploy <deploy-name>
kubectl describe svc <service-name>
```

`describe` 比 `get` 更适合排查问题，因为它会显示：

- 事件
- 调度信息
- 镜像
- 挂载卷
- 探针配置
- 最近异常

如果 Pod 起不来，第一反应通常就是：

```bash
kubectl describe pod <pod-name>
```

---

### 4. 查看日志

```bash
kubectl logs <pod-name>
kubectl logs -f <pod-name>
kubectl logs <pod-name> -c <container-name>
kubectl logs deploy/<deploy-name>
```

说明：

- `-f` 类似 `tail -f`，持续跟踪日志
- 一个 Pod 有多个容器时，通常要加 `-c`
- 对 Deployment 看日志时，`kubectl` 会帮你找到对应 Pod

---

### 5. 进入容器

```bash
kubectl exec -it <pod-name> -- /bin/sh
kubectl exec -it <pod-name> -- /bin/bash
```

常见用途：

- 看配置文件
- 检查环境变量
- 测试网络连通性
- 看应用进程

如果镜像很精简，没有 `bash`，一般用：

```bash
/bin/sh
```

---

### 6. 删除资源

```bash
kubectl delete pod <pod-name>
kubectl delete deploy <deploy-name>
kubectl delete svc <service-name>
kubectl delete -f app.yaml
```

注意：

- 删除 Pod 不一定等于应用停止
- 如果 Pod 是由 Deployment 管理的，删掉后还会自动重建

所以删除单个 Pod 常常只是“重启一个实例”。

---

### 7. 查看 YAML

```bash
kubectl get deploy <deploy-name> -o yaml
kubectl get svc <service-name> -o yaml
kubectl get pod <pod-name> -o yaml
```

这个命令非常重要，因为它可以帮助你：

- 查看对象当前完整配置
- 学习字段结构
- 导出后再修改

---

### 8. 应用配置文件

```bash
kubectl apply -f app.yaml
kubectl apply -f ./manifests
```

这是最常见的资源创建和更新方式。

一般来说：

- 开发测试时，可以先用 `kubectl create`
- 真正持续维护资源时，更推荐 `kubectl apply -f`

---

### 9. 查看事件

```bash
kubectl get events
kubectl get events -n blog-prod
kubectl get events --sort-by=.metadata.creationTimestamp
```

如果你排查：

- Pod 调度失败
- 镜像拉取失败
- PVC 绑定失败
- 探针失败

事件信息通常很有用。

---

### 10. 端口转发

```bash
kubectl port-forward pod/<pod-name> 8080:80
kubectl port-forward svc/<service-name> 8080:80
```

作用：

- 把本地端口转发到 Pod 或 Service
- 非常适合本地调试和临时访问

---

## 三、常用资源创建命令

虽然实际生产里更推荐维护 YAML 文件，但 `kubectl create` 对初学者和快速起步非常有帮助。

### 1. 创建命名空间

```bash
kubectl create namespace blog-prod
```

---

### 2. 创建 Deployment

```bash
kubectl create deployment nginx --image=nginx
```

这会创建一个最基础的 Deployment。

---

### 3. 创建 Service

```bash
kubectl expose deployment nginx --port=80 --target-port=80 --type=ClusterIP
```

如果已经有 Deployment，可以用 `expose` 很快给它创建 Service。

---

### 4. 创建 ConfigMap

```bash
kubectl create configmap app-config --from-literal=APP_ENV=prod
kubectl create configmap app-config --from-file=app.conf
```

---

### 5. 创建 Secret

```bash
kubectl create secret generic app-secret \
  --from-literal=DB_USER=blog \
  --from-literal=DB_PASSWORD=123456
```

---

### 6. 创建 Job

```bash
kubectl create job hello --image=busybox -- echo hello kubernetes
```

---

### 7. 创建 CronJob

```bash
kubectl create cronjob hello \
  --image=busybox \
  --schedule="*/5 * * * *" \
  -- echo hello kubernetes
```

---

## 四、不会每次都手写复杂 YAML，怎么办

这是非常实际的问题。

答案是：**可以先用 `kubectl create` 生成一个基础对象，再输出成 YAML，最后在这个基础上修改。**

最常用的方式就是：

```bash
kubectl create ... --dry-run=client -o yaml
```

这里有三个关键点：

- `create`：告诉 kubectl 你要创建什么资源
- `--dry-run=client`：只在本地生成，不真正提交到集群
- `-o yaml`：把结果以 YAML 输出

这几乎是写 Kubernetes YAML 最省时间的方法之一。

---

## 五、快速生成基础 YAML 的常用方式

### 1. 生成 Deployment YAML

```bash
kubectl create deployment nginx \
  --image=nginx:1.27 \
  --dry-run=client -o yaml
```

如果想直接保存到文件：

```bash
kubectl create deployment nginx \
  --image=nginx:1.27 \
  --dry-run=client -o yaml > nginx-deploy.yaml
```

然后你再去补：

- `replicas`
- `resources`
- `env`
- `volumeMounts`
- `livenessProbe`
- `readinessProbe`

这样比从零开始写快很多。

---

### 2. 生成 Service YAML

```bash
kubectl create service clusterip nginx \
  --tcp=80:80 \
  --dry-run=client -o yaml
```

也可以生成 NodePort：

```bash
kubectl create service nodeport nginx \
  --tcp=80:80 \
  --dry-run=client -o yaml
```

---

### 3. 生成 ConfigMap YAML

```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=prod \
  --from-literal=SERVER_PORT=8080 \
  --dry-run=client -o yaml
```

---

### 4. 生成 Secret YAML

```bash
kubectl create secret generic app-secret \
  --from-literal=DB_USER=blog \
  --from-literal=DB_PASSWORD=123456 \
  --dry-run=client -o yaml
```

注意：生成后的 YAML 中，Secret 数据通常会以 Base64 形式显示。

---

### 5. 生成 Namespace YAML

```bash
kubectl create namespace blog-prod --dry-run=client -o yaml
```

---

### 6. 生成 Job YAML

```bash
kubectl create job hello \
  --image=busybox \
  --dry-run=client -o yaml \
  -- echo hello kubernetes
```

---

## 六、推荐的 YAML 生成工作流

如果你不想每次从头写复杂 YAML，推荐用下面这套流程：

### 方式一：命令生成基础 YAML，再手工补全

1. 用 `kubectl create ... --dry-run=client -o yaml` 生成骨架
2. 保存为文件
3. 根据业务补充字段
4. 用 `kubectl apply -f` 提交

例如：

```bash
kubectl create deployment blog-api \
  --image=registry.example.com/blog-api:v1 \
  --dry-run=client -o yaml > blog-api.yaml
```

然后编辑 `blog-api.yaml`，补充：

- 副本数
- 环境变量
- 配置挂载
- 资源限制
- 探针
- 存储卷

最后执行：

```bash
kubectl apply -f blog-api.yaml
```

---

### 方式二：先创建真实对象，再导出 YAML

有时候你也可以先快速创建一个最小对象，再导出来改。

例如：

```bash
kubectl create deployment nginx --image=nginx
kubectl get deployment nginx -o yaml > nginx.yaml
```

但这种方式有一个问题：导出的 YAML 里通常会带上一些运行时字段，比如：

- `resourceVersion`
- `uid`
- `managedFields`
- `creationTimestamp`
- `status`

这些字段通常不适合直接当成维护用清单继续保存。

所以更推荐：

- **优先用 `--dry-run=client -o yaml` 生成基础 YAML**
- 不足的地方再自己补

---

## 七、几个非常实用的排查命令

除了创建资源，下面这组命令也非常实用。

### 1. 看 Pod 为什么启动失败

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl get events --sort-by=.metadata.creationTimestamp
```

---

### 2. 看 Deployment 发布情况

```bash
kubectl rollout status deployment/<deploy-name>
kubectl rollout history deployment/<deploy-name>
kubectl rollout undo deployment/<deploy-name>
```

---

### 3. 看 Service 和 Endpoint 是否正常

```bash
kubectl get svc
kubectl get endpoints
kubectl get endpointslices
```

如果 Service 没有后端地址，通常要检查：

- selector 是否正确
- Pod label 是否匹配
- Pod 是否 Ready

---

## 八、kubectl Bash 自动补全怎么安装

这部分很值得配，因为你敲 `kubectl` 的频率会非常高。

Kubernetes 官方文档当前对 Bash 自动补全的推荐写法是：

### 1. 当前 shell 临时启用

```bash
source <(kubectl completion bash)
```

这只对当前终端生效，关掉 shell 就没了。

---

### 2. 永久启用

把下面这行追加到 `~/.bashrc`：

```bash
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

然后重新加载：

```bash
source ~/.bashrc
```

---

### 3. 前提条件：先安装 bash-completion

官方文档明确说明，Bash 补全依赖 `bash-completion` 包。

常见安装方式：

Ubuntu / Debian：

```bash
sudo apt-get update
sudo apt-get install -y bash-completion
```

CentOS / Rocky / RHEL：

```bash
sudo yum install -y bash-completion
```

如果是较新的发行版，也可能使用：

```bash
sudo dnf install -y bash-completion
```

---

### 4. 给别名 `k` 也启用补全

很多人喜欢这样写：

```bash
alias k=kubectl
```

如果希望 `k` 也支持自动补全，还要再加一行：

```bash
complete -o default -F __start_kubectl k
```

推荐一起写进 `~/.bashrc`：

```bash
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc
```

这样之后你就可以用：

```bash
k get po<Tab>
```

来快速补全命令。

---

## 九、一个非常实用的入门操作示例

假设你现在想部署一个最简单的 nginx 服务，并且不想从零写 YAML，可以这样做：

### 第一步：生成 Deployment YAML

```bash
kubectl create deployment nginx \
  --image=nginx:1.27 \
  --dry-run=client -o yaml > nginx-deployment.yaml
```

### 第二步：生成 Service YAML

```bash
kubectl create service clusterip nginx \
  --tcp=80:80 \
  --dry-run=client -o yaml > nginx-service.yaml
```

### 第三步：检查并应用

```bash
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml
```

### 第四步：查看资源

```bash
kubectl get deploy
kubectl get pods
kubectl get svc
```

### 第五步：本地端口转发测试

```bash
kubectl port-forward svc/nginx 8080:80
```

然后访问：

```text
http://127.0.0.1:8080
```

这就是一个非常典型的“先生成 YAML，再部署，再验证”的流程。

---

## 十、初学者最该养成的习惯

如果你刚开始学 `kubectl`，最重要的不是一下子记住所有命令，而是养成这几个习惯：

### 1. 先 `get`，再 `describe`

先看对象是否存在，再看详细状态。

### 2. 出问题先看日志和事件

很多问题其实不是 YAML 语法错，而是：

- 镜像拉取失败
- 配置缺失
- 探针失败
- 存储没绑定

### 3. 优先用命令生成基础 YAML

不要每次手写完整对象，尤其是：

- Deployment
- Service
- ConfigMap
- Secret

先生成骨架，再修改，效率更高。

### 4. 尽早配好补全和别名

这会显著减少输入成本。

---

## 总结

`kubectl` 不只是一个“执行命令”的工具，它其实已经提供了一整套很成熟的资源管理能力。

对于初学者来说，最实用的路线是：

1. 先掌握常用查看和排查命令
2. 学会用 `kubectl create` 创建基础资源
3. 学会用 `--dry-run=client -o yaml` 生成 YAML 骨架
4. 配置 Bash 自动补全和 `k` 别名补全

这样你就不会陷入“YAML 太长写不动”或者“命令太多记不住”的状态，而是可以先用工具把骨架搭出来，再逐步补充细节。

这也是学习 Kubernetes 非常重要的一步：**先把工具用顺手，再去追求复杂资源的编排能力。**
