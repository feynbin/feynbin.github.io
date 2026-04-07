---
title: 三台Ubuntu搭建高可用Kubernetes实验集群
top_img: false
tags:
  - Kubernetes
  - 云原生
  - 高可用
abbrlink: e5b7c2a9
date: 2026-04-08 08:20:00
updated:
categories:
keywords:
  - Kubernetes
  - kubeadm
  - HAProxy
  - Keepalived
  - Cilium
description: 基于三台 Ubuntu 主机，记录一次实验性质的高可用 Kubernetes 控制平面部署过程，并补充一套更完整的生产级高可用方案。
cover:
highlight_shrink:
---

# 三台Ubuntu搭建高可用Kubernetes实验集群

前面几篇文章更多是在拆解 Kubernetes 的资源对象、Pod 生命周期和探针机制，这篇文章换一个角度，记录一次更偏基础设施层面的实验：**在三台 Ubuntu 主机上，手动搭建一个带控制平面高可用入口的 Kubernetes 集群。**

这次环境本质上是一个**实验版高可用方案**。它已经具备了控制平面的基本高可用能力：

- 三台控制节点
- `HAProxy` 做 API Server 转发
- `Keepalived` 提供虚拟 IP
- `kubeadm` 初始化集群
- `Cilium` 作为网络插件

但它还不是一套完全意义上的生产级“完整高可用 Kubernetes”。原因也很直接：**实验资源有限，我这次只完成了控制面的关键链路验证，没有把工作节点、外部 etcd、监控、备份、灾备等能力全部铺满。**

所以这篇文章会分成两部分：

- 第一部分：按照这次实验的真实过程，整理出一套可复现的部署步骤
- 第二部分：在文章最后补全一套更完整的高可用 Kubernetes 应该长什么样

---

## 一、实验环境说明

这次实验环境一共三台 Ubuntu 主机，全部承担控制平面相关角色：

| 主机名 | CPU / 内存 | IP |
|------|------|------|
| `master1` | 4 核 / 4GB | `10.102.213.94` |
| `master2` | 4 核 / 10GB | `10.102.213.185` |
| `master3` | 4 核 / 10GB | `10.102.213.43` |

另外还规划了一个虚拟 IP：

- `10.102.213.100`

这个 VIP 由 `Keepalived` 漂移，对外统一提供 Kubernetes API 入口；`HAProxy` 则监听 `16443` 端口，把请求转发到三台控制节点的 `6443`。

也就是说，这次控制平面的访问路径是：

```text
kubeadm / kubectl
        ↓
10.102.213.100:16443 (VIP)
        ↓
HAProxy
        ↓
master1:6443 / master2:6443 / master3:6443
```

---

## 二、部署目标和思路

这次实验要解决的核心问题，不是“单机把 Kubernetes 装起来”，而是下面这几个点：

- 控制平面不能只依赖单台机器
- API Server 入口需要是稳定地址，不能写死某一台 master
- 节点重启后，基础运行时和 kubelet 能自动拉起
- 网络插件要能正常接管 Pod 网络

所以整体思路是：

1. 先把三台机器的系统前置条件处理好
2. 手动安装 `containerd`、`runc`、`kubeadm`、`kubelet`、`kubectl`
3. 用 `HAProxy + Keepalived` 做控制面的统一入口
4. 通过 `kubeadm init` 初始化第一台控制节点
5. 再让其他节点加入控制平面
6. 安装 `Cilium` 作为 CNI，并替代 `kube-proxy`

---

## 三、主机初始化

这部分需要三台机器都执行。

### 1. 配置 hosts

为了让节点之间的名字解析更直接，先写好 `/etc/hosts`：

```bash
cat >> /etc/hosts << EOF
10.102.213.94 master1
10.102.213.185 master2
10.102.213.43 master3
EOF
```

这样后面无论是排障还是查看组件状态，都会比只看 IP 清楚很多。

---

### 2. 关闭 swap

Kubernetes 默认要求关闭 swap，否则 kubelet 在很多场景下会直接报错或行为不符合预期。

```bash
swapoff -a
sed -i '/swap/d' /etc/fstab
```

这里要注意两点：

- `swapoff -a` 只是临时关闭
- `/etc/fstab` 里的 swap 项如果不去掉，机器重启后还会重新挂载

---

### 3. 加载内核模块

容器网络和 Kubernetes 转发依赖一些内核模块，至少要准备：

```bash
cat > /etc/modules-load.d/k8s.conf << EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

lsmod | grep -E "overlay|br_netfilter"
```

这两个模块的作用可以简单理解为：

- `overlay`：为容器镜像和文件系统能力做准备
- `br_netfilter`：让桥接流量进入 iptables / netfilter 处理链

---

### 4. 配置网络参数

Kubernetes 常见的几个内核参数也需要提前设置好：

```bash
cat > /etc/sysctl.d/99-k8s.conf << EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system
sysctl net.ipv4.ip_forward
```

这里最关键的是：

- 开启 IPv4 转发
- 让桥接流量能被 iptables 处理

如果这一步没做好，后面即使集群能起来，Pod 网络、Service 转发也很容易出现问题。

---

## 四、安装容器运行时和 Kubernetes 组件

这次实验采用的是**手动安装二进制**的方式，而不是直接用系统包管理器。

这种方式的优点是：

- 组件版本更可控
- 适合理解 Kubernetes 各组件到底依赖什么
- 排查问题时更容易看清楚 systemd、二进制路径和配置文件之间的关系

缺点也很明显：

- 步骤更繁琐
- 更容易漏掉 systemd 单元文件
- 后续升级要自己负责

---

### 1. 安装 containerd 和 runc

先下载并安装 `containerd`：

```bash
wget https://github.com/containerd/containerd/releases/download/v2.2.2/containerd-2.2.2-linux-amd64.tar.gz
tar xvf containerd-2.2.2-linux-amd64.tar.gz -C /usr/local

/usr/local/bin/containerd --version
```

再安装 `runc`：

```bash
wget https://github.com/opencontainers/runc/releases/download/v1.2.5/runc.amd64
install -m 755 runc.amd64 /usr/local/sbin/runc

runc --version
```

---

### 2. 下载 kubeadm、kubelet、kubectl

```bash
export KUBERNETES_VERSION=$(curl -L -s https://dl.k8s.io/release/stable.txt)
echo "Latest Kubernetes version: $KUBERNETES_VERSION"

wget https://dl.k8s.io/release/$KUBERNETES_VERSION/bin/linux/amd64/kubeadm
wget https://dl.k8s.io/release/$KUBERNETES_VERSION/bin/linux/amd64/kubectl
wget https://dl.k8s.io/release/$KUBERNETES_VERSION/bin/linux/amd64/kubelet
```

下载完成后放到可执行路径：

```bash
chmod +x kubeadm kubectl kubelet
mv kubeadm kubectl kubelet /usr/local/bin/
```

---

## 五、配置 containerd

`containerd` 是这次实验里最底层的运行时。它的配置如果不提前理顺，后面的 `kubeadm init` 往往会踩很多坑。

### 1. 生成默认配置并切到 systemd cgroup

```bash
mkdir -p /etc/containerd
/usr/local/bin/containerd config default > /etc/containerd/config.toml

sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sed -i 's|'registry.k8s.io/pause:3.10.1'|'registry.aliyuncs.com/google_containers/pause:3.10.1'|' /etc/containerd/config.toml
```

这里最重要的是 `SystemdCgroup = true`。

因为当前主流 Linux 发行版上，`kubelet + containerd` 最稳妥的组合通常就是 `systemd` cgroup 驱动。如果 kubelet 和容器运行时的 cgroup 驱动不一致，经常会出现节点异常、资源统计不准甚至 kubelet 启动失败的问题。

---

### 2. 配置镜像加速

实验环境里，镜像拉取往往是最容易卡住的一步，所以这里额外配置了 containerd 的 registry mirror：

```bash
#!/bin/bash

CERT_DIR="/etc/containerd/certs.d"
mkdir -p $CERT_DIR

REGISTRIES=(
    "docker.io|https://docker.m.daocloud.io,https://docker.1ms.run"
    "registry.k8s.io|https://k8s.m.daocloud.io,https://k8s.1ms.run"
    "gcr.io|https://gcr.m.daocloud.io,https://gcr.1ms.run"
    "ghcr.io|https://ghcr.m.daocloud.io,https://ghcr.1ms.run"
    "quay.io|https://quay.m.daocloud.io,https://quay.1ms.run"
)

for item in "${REGISTRIES[@]}"; do
    IFS="|" read -r domain endpoints <<< "$item"
    mkdir -p "$CERT_DIR/$domain"

    cat <<EOF > "$CERT_DIR/$domain/hosts.toml"
server = "https://$domain"

EOF

    IFS="," read -ra ADDR <<< "$endpoints"
    for endpoint in "${ADDR[@]}"; do
        cat <<EOF >> "$CERT_DIR/$domain/hosts.toml"
[host."$endpoint"]
  capabilities = ["pull", "resolve"]

EOF
    done
done

sed -i "s|config_path = .*|config_path = \"$CERT_DIR\"|g" /etc/containerd/config.toml
```

这一步的核心不是“必须用哪一家镜像加速”，而是要理解：

- `containerd` 可以通过 `certs.d/<registry>/hosts.toml` 配置镜像源
- `config.toml` 里的 `config_path` 要正确指向这个目录
- 如果这一步没生效，后续拉取控制平面镜像和 CNI 镜像时会非常痛苦

---

### 3. 配置 containerd 的 systemd 服务

手动安装二进制时，`containerd.service` 往往要自己补：

```ini
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd
Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
```

配置完成后记得：

```bash
systemctl daemon-reload
systemctl enable --now containerd
systemctl status containerd
```

---

## 六、配置 kubelet

如果说手动安装里最容易漏的是哪一步，`kubelet` 的 systemd 配置基本一定排得上号。

### 1. 创建 kubelet.service

```bash
cat <<EOF > /etc/systemd/system/kubelet.service
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=https://kubernetes.io/docs/
Wants=network-online.target
After=network-online.target

[Service]
ExecStart=/usr/local/bin/kubelet
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

### 2. 创建 kubeadm 依赖的 drop-in 配置

```bash
mkdir -p /etc/systemd/system/kubelet.service.d

cat <<EOF > /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
ExecStart=
ExecStart=/usr/local/bin/kubelet \$KUBELET_KUBECONFIG_ARGS \$KUBELET_CONFIG_ARGS
EOF
```

最后开启服务：

```bash
systemctl daemon-reload
systemctl enable kubelet
```

这里即使 kubelet 先启动报错也没关系，因为在 `kubeadm init` 之前，它缺少真正的 bootstrap 配置文件是正常现象。

---

## 七、配置控制平面高可用入口

这一步是这次实验的重点。严格来说，三台 master 本身并不自动等于“高可用”，**你还需要一个稳定的 API Server 访问入口**。

这次实验采用的是：

- `HAProxy`：四层负载均衡
- `Keepalived`：VIP 漂移

---

### 1. 配置 HAProxy

`HAProxy` 负责把访问 VIP 的请求转发到三个 API Server：

```haproxy
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    timeout connect 5000
    timeout client 50000
    timeout server 50000

frontend k8s-api
    bind *:16443
    mode tcp
    option tcplog
    default_backend k8s-api-backend

backend k8s-api-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server master1 10.102.213.94:6443 check
    server master2 10.102.213.185:6443 check
    server master3 10.102.213.43:6443 check
```

这里用 `mode tcp` 很重要，因为 Kubernetes API Server 是 HTTPS/TLS 流量，这里做的是四层转发，不是七层 HTTP 代理。

---

### 2. 配置 Keepalived

三台机器都配置 `Keepalived`，但优先级不同。

`master1`：

```conf
vrrp_script check_haproxy {
    script "killall -0 haproxy"
    interval 3
    weight -20
}

vrrp_instance VI_1 {
    state MASTER
    interface ens32
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass k8s_secret
    }
    virtual_ipaddress {
        10.102.213.100
    }
    track_script {
        check_haproxy
    }
}
```

`master2`：

```conf
vrrp_script check_haproxy {
    script "killall -0 haproxy"
    interval 3
    weight -20
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens32
    virtual_router_id 51
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass k8s_secret
    }
    virtual_ipaddress {
        10.102.213.100
    }
    track_script {
        check_haproxy
    }
}
```

`master3`：

```conf
vrrp_script check_haproxy {
    script "killall -0 haproxy"
    interval 3
    weight -20
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens32
    virtual_router_id 51
    priority 80
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass k8s_secret
    }
    virtual_ipaddress {
        10.102.213.100
    }
    track_script {
        check_haproxy
    }
}
```

然后启动服务：

```bash
systemctl enable haproxy keepalived
systemctl restart haproxy keepalived
```

这套设计的核心逻辑是：

- 正常情况下 VIP 落在优先级最高的节点上
- 如果 `haproxy` 挂了，优先级会下降
- `Keepalived` 会把 VIP 漂移到其他节点

这就让外部始终只需要访问一个固定地址：`10.102.213.100:16443`。

---

## 八、初始化 Kubernetes 控制平面

高可用入口准备好之后，就可以在 `master1` 上进行首次初始化。

### 1. 生成 kubeadm 配置

先导出默认配置，再按自己的环境修改：

```bash
kubeadm config print init-defaults > kubeadm-config.yaml
```

这次实验使用的关键配置如下：

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.35.3
imageRepository: registry.aliyuncs.com/google_containers
controlPlaneEndpoint: "10.102.213.100:16443"
networking:
  podSubnet: "10.244.0.0/16"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```

这里最关键的几个点分别是：

- `controlPlaneEndpoint` 不能写某台 master 的真实 IP，而应该写 VIP
- `imageRepository` 用镜像加速源，避免拉镜像卡住
- `cgroupDriver` 与前面的 containerd 保持一致

---

### 2. 执行初始化

```bash
kubeadm init --config kubeadm-config.yaml --upload-certs --skip-phases=addon/kube-proxy
```

这里显式跳过了 `kube-proxy`，因为后面准备用 `Cilium` 的 `kubeProxyReplacement=true` 模式直接接管 Service 转发能力。

初始化完成后，把管理员配置拷到当前用户目录：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get nodes
```

---

## 九、安装 Cilium 网络插件

如果不装 CNI，控制平面虽然能初始化成功，但 Pod 网络不会真正可用。

这次选择的是 `Cilium`，并启用了 kube-proxy replacement：

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)

curl -L --fail --remote-name-all "https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz"

sudo tar xzvf cilium-linux-amd64.tar.gz -C /usr/local/bin
rm cilium-linux-amd64.tar.gz
cilium version
```

安装命令如下：

```bash
cilium install \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=10.102.213.100 \
  --set k8sServicePort=16443 \
  --set image.repository=quay.m.daocloud.io/cilium/cilium \
  --set operator.image.repository=quay.m.daocloud.io/cilium/operator-generic
```

这里有两个关键点：

- `k8sServiceHost` 和 `k8sServicePort` 明确指向高可用 API 入口
- `kubeProxyReplacement=true` 说明 Service 相关能力交给 Cilium 处理，而不是再依赖 kube-proxy

安装完成后，可以继续用下面这些命令确认状态：

```bash
kubectl get pods -A
cilium status
kubectl get nodes -o wide
```

---

## 十、这次实验方案解决了什么问题

虽然这是实验版集群，但它已经把高可用控制面的几个核心问题打通了。

### 1. API Server 不再依赖单点

如果你把 `kubectl`、`kubeadm join` 或其他控制面访问都直接写成某一台 master 的地址，那么这台机器一旦故障，整个控制面入口就会失效。

通过 `VIP + HAProxy`，现在外部只关心：

```text
10.102.213.100:16443
```

底层到底是哪台节点接管请求，对使用者是透明的。

### 2. 控制节点可以横向扩展

只要 `kubeadm init` 完成后生成 join 命令，后续其他 master 节点就可以继续加入控制平面。这样比一开始就把整个集群压在单节点上更稳。

### 3. 网络层具备继续扩展的基础

`Cilium` 接管之后，后续再加 worker 节点、再跑业务 Pod，会比单纯靠默认组件更灵活。

---

## 十一、这套实验方案还不算完整高可用的地方

这部分也必须说清楚。很多“实验环境里的高可用”其实只是做到了**入口高可用**或者**控制平面多副本**，但距离真正生产可用还差一截。

这次实验主要还缺下面这些部分：

- 没有单独规划 worker 节点池
- etcd 没有单独拆成独立集群进行管理说明
- 没有覆盖证书备份、etcd 快照和恢复演练
- 没有补监控、日志、告警体系
- 没有补存储高可用方案
- 没有补应用层的发布、回滚、限流与容灾策略

也就是说，这次更准确的表述应该是：

> 我完成的是一套控制平面实验级高可用部署，而不是一套生产级全栈高可用 Kubernetes 平台。

这个区别在面试或者实际工作里都非常重要，因为很多故障根本不是 API Server 多副本就能解决的。

---

## 十二、完整的高可用 Kubernetes 应该是什么样

如果资源和时间充足，一套更完整、更接近生产的高可用 Kubernetes，通常至少应该补齐下面这些层面。

### 1. 控制平面高可用

这是你这次实验已经覆盖到的核心部分：

- 至少 3 台 control plane 节点
- API Server 前面有稳定负载均衡入口
- 推荐使用 `HAProxy + Keepalived`，或者云厂商的 SLB / NLB
- 所有节点通过统一 `controlPlaneEndpoint` 接入

这一层解决的是：**控制面组件不要因为单机故障就整体不可访问。**

---

### 2. etcd 高可用和备份能力

真正生产环境里，`etcd` 是整个集群最关键的数据平面之一。

比较稳妥的做法通常是：

- 使用 3 台或 5 台 etcd 成员
- 与业务流量隔离，尽量不要和高负载业务混布
- 定期做 etcd snapshot
- 验证快照恢复流程，而不是只“以为备份了”

如果只做了 control plane 多副本，却没有 etcd 备份与恢复方案，那么集群实际上还是脆弱的。

---

### 3. 工作节点池高可用

完整集群不能只有 master，还应该有独立 worker 节点池：

- 至少 2 到 3 台 worker 起步
- 不同业务按节点池或标签做隔离
- 关键业务尽量跨节点、跨故障域部署
- 配合 `taint`、`toleration`、`nodeSelector`、`affinity` 做调度约束

这层解决的是：**业务负载不要和控制面混跑，也不要因为单台 worker 故障就掉服务。**

---

### 4. 网络高可用

一个完整 HA Kubernetes 不只是“Pod 能互通”，还要考虑网络组件本身的稳定性：

- CNI 插件要支持多节点稳定运行
- CoreDNS 至少双副本
- Ingress Controller 至少双副本
- 南北向流量入口要有负载均衡能力
- 网络策略要能限制横向访问风险

换句话说，网络不仅要通，还要稳、可控、可观测。

---

### 5. 存储高可用

如果集群里跑数据库、中间件或者任何需要持久化的数据服务，就必须考虑存储层：

- 动态存储供应器
- 后端共享存储或分布式存储
- 卷的快照和恢复
- 跨节点挂载能力

常见方向包括：

- NFS 作为实验起点
- Ceph / Rook 作为更完整的分布式存储方案
- 云厂商块存储 / 文件存储作为托管方案

没有存储高可用，很多“应用高可用”其实也只是表面高可用。

---

### 6. 可观测性和告警

生产环境至少要知道集群现在是不是在出问题，而不是等用户反馈：

- `Prometheus + Alertmanager`
- `Grafana`
- 日志采集方案，比如 `Loki` 或 `EFK`
- 节点、Pod、容器、控制面组件的关键指标监控

如果没有这一层，集群出故障时你只能靠 `kubectl describe` 和运气排查。

---

### 7. 安全和权限治理

完整高可用不只是“服务别挂”，还包括“别因为权限和安全问题出事故”。

至少要考虑：

- RBAC 最小权限
- Secret 管理
- 审计日志
- 节点基线加固
- 镜像仓库访问控制
- NetworkPolicy

很多时候集群不是因为不可用出问题，而是因为权限过大、配置过宽、边界缺失导致风险扩大。

---

### 8. 备份、恢复和灾备

这是很多人最容易忽略、但实际上最接近“真正高可用”的部分。

要补齐这一层，至少要有：

- etcd 备份与恢复演练
- Kubernetes 关键 YAML / Helm values 版本管理
- 镜像仓库可追溯
- 持久化数据备份
- 跨机房或跨区域灾备预案

因为真正的高可用，不是“永远不坏”，而是：

> 出问题之后，能快速恢复，而且恢复过程是可验证的。

---

## 十三、如果让我把这套实验继续补完整，我会怎么扩

如果后面资源允许，我会按下面这个顺序继续往上补：

1. 增加独立 worker 节点，把业务负载和 control plane 分开
2. 补齐三节点 etcd 的备份、恢复和快照策略
3. 部署 Ingress Controller、CoreDNS 双副本和监控体系
4. 增加动态存储方案，至少先补一个可用的实验存储类
5. 给关键应用补 `PDB`、反亲和、探针、资源限制和 HPA
6. 补日志、告警和日常巡检手段

这样整套环境才会从“能搭起来”逐步走向“能长期运行”。

---

## 总结

这次实验的价值，不在于把 Kubernetes “装上去”这么简单，而是在有限资源下，把控制平面高可用里最核心的一条链路跑通了：

- 三台控制节点
- `HAProxy + Keepalived` 提供统一入口
- `kubeadm` 负责控制平面初始化
- `Cilium` 提供网络能力

它已经足够帮助我们理解一件事：**Kubernetes 的高可用，不是多加几台机器这么简单，而是要把控制面入口、运行时、网络、数据、监控和恢复能力串成一整套系统。**

所以更准确地说，这次做成的是一套**实验版高可用 Kubernetes 控制平面**；而一套真正完整的高可用 Kubernetes，还应该继续补上 worker、etcd、存储、监控、备份和灾备这些能力。

这也是实验环境和生产环境之间最本质的差别。
