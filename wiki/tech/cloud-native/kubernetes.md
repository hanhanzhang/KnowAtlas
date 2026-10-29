---
title: "Kubernetes 核心概念与架构详解"
domain: tech/cloud-native
type: deep-dive
tags: [kubernetes, scheduler, cni, cgroup, rbac]
source: "Kubernetes v1.37.0 源码"
updated: 2026-10-16
---

# Kubernetes 核心概念与架构详解

> 本文核对基准：**Kubernetes v1.37.0 源码**。涉及版本差异的论断（1.26/1.32/1.33/1.37）均以该基准为准；带「参见」的条目可直接在源码树中定位。

## 目录

- [一、集群基础架构](#一集群基础架构)
  - [1.1 集群服务进程](#11-集群服务进程)
  - [1.2 组件交互与 Pod 调度启动流程](#12-组件交互与-pod-调度启动流程)
  - [1.3 组件如何感知变化：informer 与 watch](#13-组件如何感知变化informer-与-watch)
- [二、核心组件详解](#二核心组件详解)
  - [2.1 kube-apiserver](#21-kube-apiserver)
  - [2.2 etcd](#22-etcd)
  - [2.3 kube-controller-manager](#23-kube-controller-manager)
  - [2.4 kube-scheduler](#24-kube-scheduler)
  - [2.5 kubelet](#25-kubelet)
  - [2.6 kube-proxy](#26-kube-proxy)
  - [2.7 cloud-controller-manager](#27-cloud-controller-manager)
  - [2.8 CoreDNS 与 kubectl](#28-coredns-与-kubectl)
- [三、组件部署与通信](#三组件部署与通信)
  - [3.1 部署形态](#31-部署形态)
  - [3.2 选主机制（Leader Election）](#32-选主机制leader-election)
  - [3.3 kubelet 上报节点资源](#33-kubelet-上报节点资源)
- [四、认证与鉴权机制](#四认证与鉴权机制)
  - [4.1 信任材料：三个 CA 与一组 SA 密钥](#41-信任材料三个-ca-与一组-sa-密钥)
  - [4.2 认证器链](#42-认证器链)
  - [4.3 身份提取：CN 是用户名，O 是组](#43-身份提取cn-是用户名o-是组)
  - [4.4 五条交互路径](#44-五条交互路径)
    - [路径 A：kubectl → apiserver](#路径-akubectl--apiserver)
    - [路径 B：controller-manager → apiserver（双向职责）](#路径-bcontroller-manager--apiserver双向职责)
    - [路径 C：kubelet → apiserver（两阶段 bootstrap）](#路径-ckubelet--apiserver两阶段-bootstrap)
    - [路径 D：apiserver ↔ etcd](#路径-dapiserver--etcd)
    - [路径 E：apiserver ↔ 聚合 API（TLS 断开的两段）](#路径-eapiserver--聚合-apitls-断开的两段)
  - [4.5 三条信任隔离边界](#45-三条信任隔离边界)
  - [4.6 认证 → 授权 → 准入控制](#46-认证--授权--准入控制)
- [五、资源组织与配额](#五资源组织与配额)
  - [5.1 资源管理组件](#51-资源管理组件)
  - [5.2 QoS 等级](#52-qos-等级)
- [六、调度机制](#六调度机制)
  - [6.1 调度考量因素](#61-调度考量因素)
  - [6.2 拓扑分布](#62-拓扑分布)
  - [6.3 调度框架扩展点与默认插件](#63-调度框架扩展点与默认插件)
  - [6.4 调度队列与排序](#64-调度队列与排序)
- [七、网络与负载均衡](#七网络与负载均衡)
  - [7.1 ClusterIP 负载均衡](#71-clusterip-负载均衡)
  - [7.2 CoreDNS 负责 DNS 解析](#72-coredns-负责-dns-解析)
  - [7.3 ClusterIP 是虚拟 IP](#73-clusterip-是虚拟-ip)
  - [7.4 kube-proxy 不处理流量](#74-kube-proxy-不处理流量)
  - [7.5 ClusterIP 生成流程](#75-clusterip-生成流程)
  - [7.6 kube-proxy 规则同步](#76-kube-proxy-规则同步)
  - [7.7 Linux 网络规则管理](#77-linux-网络规则管理)
- [八、节点管理与心跳检测](#八节点管理与心跳检测)
  - [8.1 心跳机制](#81-心跳机制)
  - [8.2 心跳超时判断](#82-心跳超时判断)
  - [8.3 节点退役](#83-节点退役)
- [九、资源隔离技术](#九资源隔离技术)
  - [9.1 cgroup (Control Groups)](#91-cgroup-control-groups)
  - [9.2 资源隔离类型](#92-资源隔离类型)
  - [9.3 kubelet 资源管理组件](#93-kubelet-资源管理组件)
- [十、工作负载类型](#十工作负载类型)
  - [10.1 分类](#101-分类)
  - [10.2 对比](#102-对比)
  - [10.3 选择指南](#103-选择指南)
- [十一、Pod 生命周期](#十一pod-生命周期)
  - [11.1 Pod 生命周期阶段](#111-pod-生命周期阶段)
  - [11.2 容器生命周期阶段](#112-容器生命周期阶段)
  - [11.3 容器类型](#113-容器类型)
  - [11.4 健康检查影响](#114-健康检查影响)
  - [11.5 RestartPolicy](#115-restartpolicy)
- [总结](#总结)

---

## 一、集群基础架构

### 1.1 集群服务进程

一个标准的 Kubernetes 集群由 **控制面 4 个 + 工作节点 2 个 + 可选 1 个** 服务进程构成：

| 类型 | 服务 | 职责 |
|------|------|------|
| **控制平面** | kube-apiserver | API 网关，所有组件通过它交互 |
| **控制平面** | etcd | 分布式键值存储，持久化集群状态 |
| **控制平面** | kube-scheduler | 调度 Pod 到合适的节点 |
| **控制平面** | kube-controller-manager | 运行控制器（注册 49 个，默认启动约 43 个），维护期望状态 |
| **工作节点** | kubelet | 节点代理，管理 Pod 生命周期 |
| **工作节点** | kube-proxy | 维护网络规则，实现 Service 负载均衡 |
| **可选** | cloud-controller-manager | 云平台特定逻辑（云环境必需） |
| *客户端* | kubectl | 命令行工具，非守护进程；经 apiserver 读写集群（见 2.8） |

> **参见**：`cmd/kube-controller-manager/app/controller_descriptor.go:138`（`NewControllerDescriptors` 注册 49 个；默认关闭 bootstrapsigner/tokencleaner/selinux-warning，另有个别受 feature gate 控制）

### 1.2 组件交互与 Pod 调度启动流程

组件交互总览：

```
用户创建 Pod
    ↓
kube-apiserver（认证 → 授权 → 准入控制 → 存储到 etcd）
    │  （详见第四章 4.6「认证 → 授权 → 准入控制」）
    ↓
kube-scheduler（Filter → Score → Bind，分配节点；详见 2.4 与第六章）
    ↓
kubelet（通过 CRI 创建容器，启动健康检查）
    ↓
kube-proxy（维护网络规则，实现 Service 负载均衡）
```

Pod 调度与启动细分步骤：

```
Step 1: 用户创建 Pod → kube-apiserver → etcd

Step 2: kube-scheduler 监听到未调度 Pod
  - 从调度队列（PriorityQueue）取出 Pod
  - Filter：默认启用的 20 个插件中，14 个实现 Filter 扩展点
    （v1.37，随 feature gate 变动，见 6.3）
  - Score：对候选节点评分
  - assume + Reserve + 异步 Bind：将 Pod 绑定到最优节点
  - 写回 kube-apiserver → etcd

  注：Filter→Score→Bind 是主干简化，调度框架的 15 个扩展点见 6.3，
  完整调用链见 2.4。

Step 3: kubelet 监听到分配到本节点的 Pod
  - kubelet/config/ 获取 Pod 配置
  - kubelet/pod_workers.go 创建 Pod 工作线程
  - kubelet/kuberuntime/ 创建容器（通过 CRI 接口）
  - kubelet/prober/ 启动健康检查
  - kubelet/nodestatus/ 更新节点状态
```

### 1.3 组件如何感知变化：informer 与 watch

上文反复出现的「监听到」「Watch 到」，其统一机制是 **List/Watch**，这是理解所有组件协作的钥匙：

```
                    kube-apiserver
                         │ ① List（首次全量快照，resourceVersion=0 可走缓存）
                         │ ② Watch（增量事件流，从指定 revision 起）
                         ▼
客户端（Reflector）──> 本地缓存 (Store/Indexer) ──> 事件分发 (handler)
  KCM 各控制器            读多写少时不再打 apiserver        入 workqueue
  kube-scheduler          controller 读对象走本地缓存        （只放对象 key）
  kube-proxy
  kubelet（Pod 事件）
```

关键事实：

| 要点 | 说明 |
|------|------|
| **resourceVersion = etcd revision** | 不是时间戳，是 etcd 的全局单调递增版本号；watch 从指定 revision 回补错过的事件 |
| **List/Watch 分离** | List 拿全量快照，Watch 只收增量；断线后带上次 revision 重新 Watch 即可续传 |
| **apiserver 侧有 watch cache** | 读请求（List/Get/Watch 起点）先走内存环形缓冲，未命中才回源 etcd；缓存未就绪时降级直读 etcd |
| **乐观并发控制** | 更新带 resourceVersion 做条件写（etcd 事务 `OptimisticPut`），冲突则 apiserver 自动回读重试 |
| **事件只传 key** | informer 回调通常只把对象 key 放进 workqueue，处理时再从本地缓存读最新状态——保证幂等与解耦 |

这条链路在两侧的实现：

- **apiserver 侧（推）**：etcd watch → Reflector 喂 watchCache → 分发协程 → 每个客户端 watcher 独立 send goroutine → HTTP 编码推送
  > **参见**：`staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go`（watch cache 与分发）、`staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go`
- **客户端侧（收）**：client-go 的 `Reflector.ListAndWatch` 循环 + 本地 cache + handler 分发；各组件均基于此构建
  > **参见**：`staging/src/k8s.io/client-go/tools/cache/reflector.go`

**各组件的 watch 对象速查**（详见各自章节）：

| 组件 | Watch 的资源 |
|------|-------------|
| kube-scheduler | Pod（未调度）、Node、PV/PVC、StorageClass/CSI*、PodGroup、DRA 资源等（`pkg/scheduler/eventhandlers.go:641`） |
| kube-controller-manager | 各控制器对应资源（Deployment、EndpointSlice、Node、Lease…） |
| kube-proxy | Service（排除 headless）、EndpointSlice、本节点 Node（见 2.6） |
| kubelet | 分配给本节点的 Pod（按 `spec.nodeName` 过滤，见 2.5） |
| cloud-controller-manager | Node、Service、云资源状态（见 2.7） |

---

## 二、核心组件详解

本章按「每个组件具体做什么」组织，统一模板：**职责 → 内部结构 → 主循环/工作方式 → 与其他组件的交互 → 源码出处**。机制深潜保留在第四章（认证）、第六章（调度）、第七章（网络），本章与之交叉引用。

### 2.1 kube-apiserver

**职责**：集群唯一的 API 入口，也是**唯一读写 etcd 业务数据的组件**——所有组件（包括 kubectl）都只与它通信。

**三层委派链**（外 → 内）：

```
请求 → apiextensions-apiserver   处理 CRD 资源
        └→ kube-apiserver        处理内置资源（Pod/Node/Service/...）
              └→ kube-aggregator 按 APIService 把非本机资源反向代理到扩展 apiserver
```

> **参见**：`cmd/kube-apiserver/app/server.go:185-206`（"aggregator comes last in the chain"）

**请求处理链**（节选，完整约 25 层中间件）：

```
WithRequestInfo（解析 verb/group/resource）
  → 认证 authn → tracing → 审计 audit → impersonation
  → 优先级与公平性 APF（流控排队）→ 授权 authz
  → 路由 Director → 具体资源 handler
```

第四章只展开了其中「认证」「授权」两层；超时、HSTS、CORS、panic recovery、优雅关闭等其余中间件见源码。

> **参见**：`staging/src/k8s.io/apiserver/pkg/server/config.go:1036-1116`（`DefaultBuildHandlerChain`，由外到内）

**单个写请求的内部顺序**（以 Create 为例）：

| 步骤 | 动作 |
|------|------|
| 1 | 解码 + `fieldValidation`（Strict/Warn/Ignore）未知/非法字段检查 |
| 2 | **变异型准入**（mutating admission：默认值注入、webhook 等） |
| 3 | `rest.BeforeCreate` → `strategy.Validate`（内置资源真正的语义校验层） |
| 4 | **校验型准入**（validating admission / ValidatingAdmissionWebhook / ResourceQuota） |
| 5 | 写入 etcd（乐观并发事务），watch cache 同步更新 |

> **参见**：`staging/src/k8s.io/apiserver/pkg/endpoints/handlers/create.go:119-219`、`.../registry/generic/registry/store.go:545-612`

**主要子系统**：

| 子系统 | 职责 |
|--------|------|
| REST 存储 + watch cache | 每个资源一个 `genericregistry.Store`；读路径先走内存环形缓存，未命中/未就绪回源 etcd |
| watch 推送 | 四段流水线（见 1.3）：etcd watch → Reflector → watchCache → 每 watcher 独立 send goroutine |
| 聚合层 | 按 APIService 把请求代理到扩展 apiserver，合并其 discovery/OpenAPI（metrics-server 即挂在这层） |
| API discovery | 提供 `/api`、`/apis` 及聚合式 `discovery.k8s.io/v2`；kubectl 靠它知道"集群有哪些资源" |
| OpenAPI | `/openapi/v2`、`/openapi/v3`；供客户端校验与 Server-Side Apply 类型转换 |
| APF 流控 | 按 FlowSchema / PriorityLevelConfiguration 对请求排队调度（`--enable-priority-and-fairness` 默认开；关闭则退化为固定并发闸门） |
| admission 装配 | `AllOrderedPlugins` 有序插件表（`pkg/kubeapiserver/options/plugins.go:72`） |
| 授权链 | RBAC、Node、Webhook 等 authorizer 可组合（`--authorization-mode`，kubeadm 默认 `Node,RBAC`） |

**与其他组件的交互**：

- 常态数据是各组件**主动推/拉**进 apiserver 的（watch + status 上报），apiserver **不维持**到 kubelet 的常驻连接
- 仅当客户端访问 `nodes/proxy`、`pods/log`、`pods/exec|attach|portforward` 这类**节点代理型子资源**时，apiserver 才按 `--kubelet-*` 配置临时拨号到 kubelet
- kubeadm 启动 apiserver 的 flag 组（证书、SA 签发、front-proxy、`--authorization-mode=Node,RBAC`、`--enable-admission-plugins=NodeRestriction`）见 `cmd/kubeadm/app/phases/controlplane/manifests.go:185`

> **参见**：`pkg/registry/core/node/rest/proxy.go`、`pkg/registry/core/pod/rest/subresources.go`（exec/attach 的 `ExecLocation`）

### 2.2 etcd

**职责**：分布式强一致 KV 存储，Kubernetes 的**唯一持久化后端**（k8s 只使用其 KV 能力；Raft 选举、成员管理属于 etcd 自身领域）。

**访问边界**（谁能直连 etcd）：

| 组件 | 直连 | 用途 |
|------|------|------|
| kube-apiserver | ✅（唯一业务读写者） | 所有资源对象的 CRUD / watch |
| kubeadm | ✅（集群外工具） | etcd 成员管理、健康检查、部署 etcd 静态 Pod |
| 其余所有组件 | ❌ | 只与 apiserver 交互，均不引用 etcd client |

**k8s 使用 etcd 的方式**：

| 机制 | 说明 |
|------|------|
| resourceVersion = etcd revision | 全局单调递增版本号；乐观并发控制——更新按原 revision 做条件写（事务 `OptimisticPut`），冲突则回读重试 |
| watch | 从指定 revision 回补事件，支持分页/流式续传（`WithRev` + `WithPrevKV`） |
| compaction | 默认保留最近 **5 分钟**历史版本，防止历史无限膨胀（watch 落后太久需全量 List 重建） |
| lease / TTL | events 等对象的自动过期 |
| 读路径 | List/Get 先走 apiserver 的 watch cache，未命中才回源 etcd（见 1.3） |

> **参见**：`staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go`（CRUD + 乐观锁重试）、同目录 `watcher.go`（watch）、`compact.go`（历史压缩）、`.../storage/cacher/delegator.go`（cache/etcd 路由）

### 2.3 kube-controller-manager

**职责**：运行所有**需要集群全局视野的控制循环**，把实际状态不断拉向期望状态（无状态，靠 Lease 选主保证多副本时只有 Leader 干活，见 3.2）。

**控制器通用模式（informer → workqueue → 对账）**：

```go
// 1. informer 事件回调：只把对象 key 放进队列，不带对象数据
informer.AddEventHandler(ResourceEventHandlerFuncs{
    AddFunc:    func(obj) { c.queue.Add(key) },
    UpdateFunc: func(_, obj) { c.queue.Add(key) },
    DeleteFunc: func(obj) { c.queue.Add(key) },
})

// 2. worker 取 key → 从 informer 本地缓存读最新对象 → 对账
func (c *Controller) processNextWorkItem() {
    key, quit := c.queue.Get()          // key 形如 "default/my-nginx"
    if err := c.syncHandler(key); err != nil {
        c.queue.AddRateLimited(key)     // 失败：指数退避后重新入队
        return
    }
    c.queue.Forget(key)                 // 成功：清除重试计数
}
```

三个关键设计：

- **幂等**：同一 key 处理任意多次结果一致，事件丢失/重复无害
- **只传 key 不传对象**：处理时从本地缓存读，保证看到的是最新状态，也避免队列膨胀
- **失败退避**：workqueue 自带 rate limit，防止热点对象打爆 apiserver

**控制器分类**：

| 类别 | 控制器 |
|------|--------|
| **工作负载** | Deployment, ReplicaSet, StatefulSet, DaemonSet, Job, CronJob |
| **网络** | Endpoint, EndpointSlice, Node IPAM（Pod CIDR 分配） |
| **节点** | Node Lifecycle, Taint Eviction |
| **存储** | PV Controller, Storage Version Migrator |
| **服务账户** | ServiceAccount, Token, RootCA ConfigMap |
| **命名空间** | Namespace, GC (Garbage Collection) |
| **其他** | HPA, PDB, Resource Quota |

> **参见**：`cmd/kube-controller-manager/app/controller_descriptor.go:138`（注册）、`staging/src/k8s.io/client-go/util/workqueue/`（队列与退避）、各控制器构造在 `cmd/kube-controller-manager/app/*.go`

### 2.4 kube-scheduler

**职责**：为未调度的 Pod 选择一个合适的节点，只做决策与绑定，不管理已运行的 Pod。

**主循环**在 `pkg/scheduler/schedule_one.go` 中实现：

```
ScheduleOne
  └─ schedulingCycle（调度周期）
       ├─ PreFilter → findNodesThatPassFilters（并行过滤 + extender 过滤）
       ├─ 无可行节点 → RunPostFilterPlugins（默认抢占）
       ├─ prioritizeNodes（PreScore + Score）→ 取最高分节点
       └─ assume + Reserve + Permit（写入调度器缓存）
  └─ bindingCycle（绑定周期，独立 goroutine 异步执行）
       └─ PreBind → Bind（写回 apiserver）→ PostBind
```

> **参见**：`pkg/scheduler/schedule_one.go`（`ScheduleOne`/`schedulingCycle`/`bindingCycle`）

**与其他组件的交互**：watch Pod/Node/PV 等（清单见 1.3），决策结果通过**写回 apiserver 的 binding 子资源**生效（不直连 kubelet）；多副本经 Lease 选主（见 3.2）。

### 2.5 kubelet

**职责**：节点代理——把 apiserver 中期望的 Pod 落成本机容器，并持续上报状态。每个节点运行一个。

**配置来源（三源 mux）**：

| 来源 | 用途 | 机制 |
|------|------|------|
| **apiserver watch** | 常规 Pod | List/Watch `pods?spec.nodeName=<本节点>`，先等本节点注册完成才开始（1s 轮询） |
| **file** | static pod | 扫描 `--pod-manifest-path`，20s 周期 + fsnotify 即时触发 |
| **http** | 远程清单 | 20s 拉取 URL |

三个来源经 `mux` 合并进一个 `updates` channel，统一喂给主循环。static pod 会在 apiserver 创建同名 **mirror pod**（只读镜像，供调度器/控制器可见）。

> **参见**：`pkg/kubelet/config/`（`mux.go` 多源合并）、`config/apiserver.go:36`、`kubelet.go:523`（`makePodSourceConfig`）

**主循环 syncLoop**（事件驱动的 select 分发）：

```
syncLoop
  ├─ configCh        → HandlePodAdditions / Updates / Removes / Reconcile
  ├─ plegCh          → 容器状态同步（HandlePodSyncs）
  ├─ syncCh（每 1s） → 扫 workQueue 中到期的 Pod
  ├─ liveness/readiness/startup 探针结果通道
  └─ housekeeping（每 2s）→ HandlePodCleanups（清理已终止 Pod）
```

**pod_workers**：每个 Pod 一个独立 goroutine 串行处理其生命周期操作；成功后按 `SyncFrequency`（默认 1 分钟）重同步，失败按 10s 指数退避（transient 错误 1s）；**驱逐**表达为 `SyncPodKill` + `evicted` 标记。

> **参见**：`pkg/kubelet/kubelet.go:2671`（`syncLoop`）、`pkg/kubelet/pod_workers.go:1245`（`podWorkerLoop`）

**容器状态检测**（谁检测容器状态——kubelet）：

```
kubelet
  ├── PLEG (Pod Lifecycle Event Generator)
  │   └── 默认每 1 秒 relist 一次容器状态（健康阈值 3 分钟），
  │       单轮耗时受 runtime 影响可能超过 1 秒；
  │       启用 EventedPLEG 时改为 CRI 事件驱动 + 按需 relist
  ├── Prober (健康检查)
  │   ├── Liveness Probe: 存活探针
  │   ├── Readiness Probe: 就绪探针
  │   └── Startup Probe: 启动探针
  └── CRI 调用容器运行时
      └── 获取容器状态信息
```

**CRI 边界**：kubelet 不直接操作容器运行时，全部经 CRI gRPC 接口（`kuberuntime/` 封装）：

| 阶段 | 典型 CRI 调用 |
|------|--------------|
| Pod 网络/隔离就绪 | RunPodSandbox / StopPodSandbox / RemovePodSandbox |
| 容器生命周期 | CreateContainer → StartContainer → StopContainer → RemoveContainer |
| 状态查询 | ListContainers / ContainerStatus / ListPodSandbox |
| 镜像 | ImagePull / ImageStatus / RemoveImage |
| 流式操作 | Exec / Attach / PortForward（返回流式 URL 或执行） |

**后台周期（精选）**：

| 循环 | 默认周期 |
|------|---------|
| syncCh 扫 workQueue / housekeeping | 1s / 2s |
| PLEG relist（阈值 3min） | 1s |
| Pod status 批量同步（statusManager，事件触发立即） | 10s |
| 探针（timeout 1s、failureThreshold 3） | 10s |
| eviction 监控 | 10s |
| node status 计算 / 实际上报 | 10s / 5min（见 3.3） |
| Node Lease 续约（duration 40s） | 10s（见 8.1） |
| container GC / image GC | 1min / 5min |
| volume reconciler / populator | 100ms / 100ms |
| runtime 健康检查 | 5s |
| file/http 配置源扫描 | 20s / 20s |

**streaming server**：`exec`、`logs`、`portforward` 的 HTTP 服务由 kubelet 直接提供（apiserver 只做转发，见 2.1）；`pods/log` 直接读取 CRI 落盘的日志文件。

**与其他组件的交互**：三套客户端（kube/event/heartbeat）——watch 本节点 Pod、Patch PodStatus（10s 批量 + 事件触发）、Patch NodeStatus（见 3.3）、续约 Node Lease（见 8.1）、上报 Event、维护 mirror pod。

> **参见**：`cmd/kubelet/app/server.go:656`（启动流程）、`pkg/kubelet/kuberuntime/`（CRI 封装）、`pkg/kubelet/server/server.go`（streaming/metrics 端点）

### 2.6 kube-proxy

**职责**：每个节点上的网络代理——把 Service/EndpointSlice 状态翻译成本机内核转发规则（规则机制详见第七章）。

**watch 与同步**：

| Watch 对象 | 过滤条件 |
|-----------|---------|
| Service | 排除 headless（`clusterIP=None`，交给 DNS）、排除带 `service.kubernetes.io/proxy-name` 标签的（由其他代理接管） |
| EndpointSlice | 排除 headless Service 的 slice |
| 本节点 Node | `metadata.name` 字段过滤；拓扑标签变化触发全量同步 |

事件回调（`OnServiceUpdate`/`OnEndpointSliceUpdate`）只标记「有变化」，真正下发由 **bounded-frequency-runner** 合流：

- `minSyncPeriod` **1s**（合并突发变更）、`syncPeriod` **30s**、全量重算 `FullSyncPeriod` **1h**
- 同步失败按退避重试；增量同步 = serviceChanges + endpointsChanges 局部计算

**进程边界**：

- healthz 端口 **10256**（应答本地 endpoint 存活，不转发流量）、metrics 端口 **10249**
- 配置是**本地 `--config` 文件 + fsnotify**：文件被修改/改名即**报错退出、由上层（DaemonSet 重启）生效**——1.37 的 kube-proxy 不 watch ConfigMap

**模式现状（v1.37）**：iptables 为当前默认模式（**1.40 起默认切换到 nftables**）；**ipvs 已废弃**（1.40 默认禁用、1.43 移除）；userspace 已于 1.26 移除（见 7.4）。

> **参见**：`cmd/kube-proxy/app/server.go:527`（Run/informer 装配）、`pkg/proxy/iptables/proxier.go:454`（事件→Sync）、`pkg/proxy/runner/bounded_frequency_runner.go`、`pkg/proxy/apis/config/v1alpha1/defaults.go:62`（30s/1s）

### 2.7 cloud-controller-manager

**职责**：运行**依赖云厂商 API** 的控制循环，从 kube-controller-manager 中剥离而来（KEP-2395）；云环境部署，自建机房通常不部署。

| 控制器 | 职责 |
|--------|------|
| **cloud-node** | 新节点注册后回填 providerID、内外网地址、云相关标签（`UpdateNodeStatus`） |
| **cloud-node-lifecycle** | 云上实例已删除 → 删除 Node 对象；已关机 → 打 shutdown taint |
| **service** | 同步 `type=LoadBalancer` 的 Service 与云 LB（创建/更新/删除监听器与后端） |
| **route** | 为 PodCIDR 在云上创建路由条目（需 `--configure-cloud-routes` 且云支持 Routes API） |
| **nodeipam** | 分配/下发节点 Pod CIDR（复用 KCM 的 IPAM 代码） |

**与 kube-controller-manager 的边界**：

- KCM 的 `--cloud-provider` **只接受空串或 `external`**（in-tree provider 自 v1.31 起禁用）
- node lifecycle **未整体迁移**：KCM 保留基于租约/驱逐的纯集群版，CCM 只有依赖云 API 的判定版（实例删除/关机）
- 注：persistent volume label、clustername 两个控制器 **1.37 已删除**，勿再引用

> **参见**：`staging/src/k8s.io/cloud-provider/app/controllermanager.go:442`（默认注册表）、`cmd/cloud-controller-manager/main.go:62`（nodeipam）、`cmd/kube-controller-manager/app/options/options.go:477`（仅接受 external）

### 2.8 CoreDNS 与 kubectl

**CoreDNS**：集群 DNS 服务，负责 Service 名称解析（机制见 7.2）。

- **部署**：kubeadm addon 生成 ConfigMap（Corefile）+ Deployment + Service（即 `kube-dns`，其 ClusterIP 写入 kubelet 的 `clusterDNS`）；传统部署方式见 `cluster/addons/dns/coredns/`
- **Corefile 要点**（`cmd/kubeadm/app/phases/addons/dns/manifests.go:170`）：`kubernetes`（集群域反查，fallthrough 到上游）、`forward . /etc/resolv.conf`、`cache 30`（对集群域禁用缓存）、`prometheus :9153`、`loop`/`reload`/`loadbalance`、`health`/`ready`
- **下发链路**：kubeadm 把 `clusterDNS`（CoreDNS Service IP）与 `clusterDomain` 写进 kubelet 配置 → kubelet 的 DNS Configurer 在创建 sandbox 时注入 Pod 的 dnsConfig——`ClusterFirst` 策略下 nameserver 即 clusterDNS，search 域自动生成 `<ns>.svc.<domain>`、`svc.<domain>`、`<domain>`

> **参见**：`pkg/kubelet/network/dns/dns.go:385`（`GetPodDNS`）、`pkg/kubelet/kuberuntime/kuberuntime_sandbox.go:98`（注入 CRI dnsConfig）

**kubectl**：**薄客户端**，本身只做参数解析、命令树与输出格式化；真正逻辑在 `k8s.io/kubectl`、`k8s.io/cli-runtime`、`k8s.io/client-go`。

- **discovery**：磁盘缓存（TTL 6h）+ `DeferredDiscoveryRESTMapper` 把资源名解析成 GVR，再经 Builder 构造/读取对象
- **apply**：读取 `kubectl.kubernetes.io/last-applied-configuration` 注解 → **三方合并**（original=上次 apply、modified=本地文件、current=服务器对象）→ 生成 PATCH；无法合并时降级 delete+create；`--server-side` 走 SSA
- **patch**：`--type` 支持 json/merge/strategic（默认 strategic merge）
- 与 apiserver 的交互见 4.4 路径 A

> **参见**：`staging/src/k8s.io/kubectl/pkg/cmd/apply/patcher.go:367`（`CreateThreeWayMergePatch`）、`staging/src/k8s.io/cli-runtime/pkg/resource/builder.go`

---

## 三、组件部署与通信

### 3.1 部署形态

| 部署方式 | 形态 | 适用场景 | 组件间通信 |
|---------|------|---------|-----------|
| **同机二进制** | 各组件直接运行 | 开发、测试、小规模生产 | localhost (127.0.0.1) |
| **异机二进制** | 各组件分布在不同机器 | 生产环境、大规模集群 | 内网 IP (10.x.x.x) |
| **kubeadm** | 控制面组件为静态 Pod（etcd 可选 stacked/外部），kube-proxy/CoreDNS 为 DaemonSet | 生产默认安装方式 | 同集群 Service/节点网络 |

### 3.2 选主机制（Leader Election）

kube-scheduler 和 kube-controller-manager 的选主通过 **Lease 资源**实现：

```
实例 1 (Leader)    ← Lease →    kube-apiserver    →    etcd
实例 2 (Standby)   ← 监听 Lease
实例 3 (Standby)   ← 监听 Lease
```

- 不是直接通过 etcd，而是通过 kube-apiserver 间接使用 etcd
- Lease 资源类型为 `coordination.k8s.io/v1`，存储在 `kube-system` 命名空间：
  - kube-scheduler 的 Lease 名为 `kube-scheduler`
  - kube-controller-manager 的 Lease 名为 `kube-controller-manager`
- Leader 默认每 2 秒尝试续约（RetryPeriod=2s），单轮续约放弃期限 10 秒（RenewDeadline），超过 15 秒（LeaseDuration）未续约则 Lease 过期
- 默认值定义在 `component-base/config/v1alpha1/defaults.go`（client-go 的 `leaderelection.go` 只做合法性校验，如 LeaseDuration > RenewDeadline）

> **参见**：`cmd/kube-scheduler/app/options/options.go:112`、`cmd/kube-controller-manager/app/options/options.go:247`、`staging/src/k8s.io/client-go/tools/leaderelection/leaderelection.go`

### 3.3 kubelet 上报节点资源

kubelet 上报节点资源信息给 **kube-apiserver**，大部分信息**持久化到 etcd**（Node/Pod 对象），唯独资源使用率不持久化：

| 数据类型 | 上报频率 | 是否持久化 | 说明 |
|---------|---------|-----------|------|
| 节点容量 | 启动时 + 变化时 | 是（但很少变） | CPU/Memory/Disk 总量 |
| 可分配资源 | 启动时 + 变化时 | 是（但很少变） | 减去系统预留后的资源 |
| 节点条件 | 每 10 秒重算（带 4% 抖动）；有变化则下一个周期立即上报，无变化每 5 分钟上报一次 | 是 | Ready/MemoryPressure 等 |
| 容器状态 | 每 10 秒批量同步；状态变化时立即触发 | 是 | 运行/等待/终止状态 |
| 资源使用率 | 每 10 秒采样（cAdvisor housekeeping，内存缓存 2 分钟） | 否 | 暴露在 kubelet 本地端点（`/stats/summary` 按需组装、`/metrics/resource`），由 metrics-server 等拉取；卷统计另按 `volumeStatsAggPeriod`（1 分钟）聚合 |

**不需要持久化的原因：**

1. 数据是临时的，会频繁变化
2. 数据可再生，kubelet 会重新上报
3. 更新频率高，持久化会导致性能问题

> **参见**：默认值 `pkg/kubelet/apis/config/v1beta1/defaults.go`（`NodeStatusUpdateFrequency=10s`、`NodeStatusReportFrequency=5m`）；容器状态同步 `pkg/kubelet/status/status_manager.go:215`（`syncPeriod = 10s`）；上报判定 `pkg/kubelet/kubelet_node_status.go:512`。注：`syncFrequency`（默认 1 分钟）是运行容器/配置的定期重同步，与上述频率不同。

---

## 四、认证与鉴权机制

### 4.1 信任材料：三个 CA 与一组 SA 密钥

集群的信任体系建立在 `kubeadm init` 阶段生成的一组材料上，它们被固化到磁盘并挂载进控制面容器：

| 根 | 签发对象 | 说明 |
|----|---------|------|
| **ca** | apiserver 服务端证书、各组件客户端证书 | Kubernetes API 层的身份根 |
| **front-proxy-ca** | front-proxy-client 证书 | 聚合 API 的身份宣告根 |
| **etcd-ca** | etcd 服务端/peer/健康检查/apiserver 客户端证书 | 存储层身份根（仅 stacked etcd 时生成） |

此外还有一组**独立于 CA 体系**的 ServiceAccount 密钥（`sa.key` / `sa.pub`），用于签发和校验 Pod 身份令牌。它们不在证书清单里，由 `CreatePKIAssets` 单独生成；外部 CA 场景走 `UsingExternal*` 分支。

SA 密钥在三个 flag 上的分工：

| Flag | 持有者 | 材料 | 用途 |
|------|--------|------|------|
| `--service-account-private-key-file` | kube-controller-manager | sa.key（私钥） | 签发遗留 secret 型 SA Token（TokensController，LegacyIssuer） |
| `--service-account-signing-key-file` | kube-apiserver | sa.key（私钥） | 签发 TokenRequest / 投影 token 的 JWT |
| `--service-account-key-file` | kube-apiserver | sa.pub（公钥，可配多把） | **验证** SA token 签名 |

> **参见**：`cmd/kubeadm/app/phases/certs/certlist.go`（证书清单与归属声明）、`cmd/kubeadm/app/phases/certs/certs.go:74`（SA 密钥生成，注释「Service accounts are not x509 certs」）、`pkg/kubeapiserver/options/authentication.go:430`、`pkg/controlplane/apiserver/options/options.go:205`

**两套体系的本质区别：**

| 对比项 | X.509 证书体系 | ServiceAccount 体系 |
|-------|----------------|-------------------|
| 材料 | 三个 CA 根及其签发的证书 | 一对裸密钥 |
| 结构 | 证书链 + EKU + KeyUsage | 仅签名，无证书对象 |
| 生效方式 | `--client-ca-file` 等参数 | `--service-account-key-file` 等 |
| 覆盖范围 | 谁能连接 apiserver | 哪个 Pod 是谁 |

ServiceAccount 密钥不是 CA：它没有证书链概念，也不参与 X.509 验证，仅用于 JWT 的签发与验签。

### 4.2 认证器链

请求进入 kube-apiserver 后，按固定顺序依次尝试认证器：

```
请求进入 kube-apiserver
    │
    ├─① requestHeader    验 front-proxy-ca 签发的证书
    │                     且 CN 必须在 allowed-names 白名单内
    │                     （kubeadm 默认设为 front-proxy-client；
    │                       白名单为空 = 不限制任何 CN）
    │                     命中 → 身份取自 X-Remote-User 请求头
    │
    ├─② x509             验 ca 签发的客户端证书
    │                     CN → 用户名，O → 用户组
    │
    ├─③ token union      静态 token → legacy SA JWT → SA issuer JWT →
    │                     bootstrap token → OIDC/JWT → webhook
    │                     （OIDC 刻意排在 SA 之后）
    │
    ├─④ anonymous        仅当整链「未认证且无错误」时才兜底
    │
    └→ 追加 system:authenticated 组（anonymous 拿不到该组）
```

**union 语义**：前一个未认证**或报错**都会继续尝试下一个（错误被聚合、不短路），只有成功才返回。anonymous 不是普通一环，而是用 `NewFailOnError` 接在链尾——链上已有错误时**不会**降级为匿名，只有整链「未认证且无错」才落到 anonymous。

`system:authenticated` 由 `AuthenticatedGroupAdder` 在 union 外层统一追加（在 anonymous 之前），实现里显式跳过匿名用户。

> **参见**：`pkg/kubeapiserver/authenticator/config.go:115-246`（认证器装配顺序）、`staging/src/k8s.io/apiserver/pkg/authentication/request/union/union.go:53`、`staging/src/k8s.io/apiserver/pkg/authentication/group/authenticated_group_adder.go`

### 4.3 身份提取：CN 是用户名，O 是组

客户端证书通过验证后，apiserver 从**链上叶子证书**的主题中提取身份：

| 证书字段 | 映射为 | 作用 |
|---------|-------|------|
| **CN** (Common Name) | 用户名 | RBAC 匹配 `kind: User` |
| **O** (Organization) | 用户组（每个 O 一个组） | RBAC 匹配 `kind: Group` |

典型分配：

| 组件 | CN | O |
|------|-----|---|
| controller-manager | `system:kube-controller-manager` | — |
| scheduler | `system:kube-scheduler` | — |
| 管理员 | `system:admin` | `system:masters` |
| kubelet | `system:node:<节点名>` | `system:nodes` |
| kube-proxy | 使用 ServiceAccount token 认证（kubeadm 方式），不使用客户端证书 | — |

**认证与授权是两件事**：认证（CA/证书）确认"你是谁"，CN/O 只是身份标识的一部分；"你能干什么"由授权（RBAC）依据该身份决定。持有合法证书只是通过了第一道关。

> **参见**：`staging/src/k8s.io/apiserver/pkg/authentication/request/x509/x509.go:289`（`CommonNameUserConversion`）

### 4.4 五条交互路径

#### 路径 A：kubectl → apiserver

```
kubectl
  │ ① 验证 apiserver 服务端证书
  │ ② 出示客户端证书（CN=管理员，O=system:masters）
  ▼
kube-apiserver
  │ ③ x509 认证器验签，提取 CN/O
  │ ④ RBAC 授权
  ▼
etcd
```

#### 路径 B：controller-manager → apiserver（双向职责）

```
controller-manager
  │ 使用 controller-manager.kubeconfig 调 apiserver（客户端证书认证）
  ▼
kube-apiserver

反向职责 —— controller-manager 持有两把私钥：
  ├── sa.key      签发遗留 secret 型 ServiceAccount Token
  └── ca.key      签发 CSR（kubelet 证书等）
apiserver 同时持有 sa.pub（--service-account-key-file，验签）
和 sa.key（--service-account-signing-key-file，签发投影 token）
```

同一套 `ca` 根，在这条路径上被用于两个相反方向：controller-manager 调 apiserver 时是**被认证方**，签发证书时是**认证方**。

> 注：未配置 `--cluster-signing-{cert,key}-file` 时 signer 控制器跳过；kubeadm 的 external-CA 模式会置空这两个参数，使 csrsigning 不启动。

#### 路径 C：kubelet → apiserver（两阶段 bootstrap）

kubelet 的证书不由 kubeadm 预生成，而是运行时现场申请：

```
阶段一：kubelet 仅有 bootstrap 凭证，提交 CSR
    │
    ▼
controller-manager 批准（csrapproving）并签发（csrsigning）
（apiserver 侧仅做 RBAC + 准入校验，不产生批准动作）
    │
    ▼
阶段二：kubelet 获得正式证书
        CN=system:node:<节点名>，O=system:nodes
```

证书临近过期时 kubelet 自动轮换，无需人工干预（轮换阈值为证书生命周期的 70%~90%，即 80%±10% 抖动）。

> **参见**：`cmd/kube-controller-manager/app/certificates.go`（csrapproving/csrsigning 注册）、`pkg/kubelet/certificate/bootstrap/bootstrap.go:165`（`requestNodeCertificate`）、`staging/src/k8s.io/client-go/util/certificate/certificate_manager.go:703`

#### 路径 D：apiserver ↔ etcd

```
kube-apiserver
  │ 携带 apiserver-etcd-client 证书
  ▼
etcd ──验 etcd-ca──> 确认 apiserver 身份

etcd 成员之间 ──验 etcd-ca──> 相互认证（peer）
```

`ca` 不参与这条路径，存储层拥有独立的信任根。

#### 路径 E：apiserver ↔ 聚合 API（TLS 断开的两段）

聚合 API 服务（如 metrics-server）**看不到用户的证书** —— 它是独立进程，与 apiserver 之间另建 TLS 连接。因此身份必须通过 HTTP 请求头传递：

```
kubectl ──用户证书──> kube-apiserver
                          │  用户证书在此终结
                          │
                          │  以 front-proxy-client 身份建立新连接
                          │  用户身份写入 X-Remote-User 头
                          ▼
                    聚合 API 服务（独立进程）
                    无独立鉴权能力，完全信任该请求头
```

**为什么必须独立 CA：**

请求头是明文 HTTP 字段，任何能连接 apiserver 的客户端都能伪造 `X-Remote-User: system:masters`。因此判定"能否采信请求头"必须依赖证书：

```
请求进入
  ├─ 证书由 front-proxy-ca 签发？  → 否，请求头全部丢弃
  ├─ CN 等于 front-proxy-client？   → 否，请求头全部丢弃
  └─ 都满足 → 采信 X-Remote-User
```

独立 CA 的意义在于**签发权隔离**：它的私钥仅在初始化时使用，之后无任何组件使用。共用了 `ca` 的话，持有 `ca` 私钥的一方可以签出任意 CN 的证书，CN 白名单形同虚设。

### 4.5 三条信任隔离边界

| CA | 运行时私钥持有者 | 失陷后果 |
|----|----------------|---------|
| **ca** | controller-manager（持续持有，用于签发 CSR） | 伪造任意组件身份、签发任意 CSR |
| **front-proxy-ca** | 无组件使用（初始化后即封存；注：整个 pki/ 目录会只读挂进 apiserver/KCM 容器，但无 flag 引用该私钥） | 伪造身份宣告者 |
| **etcd-ca** | etcd 静态 Pod | 冒充 etcd 成员，可读写全量数据 |

**隔离的技术基础**（修正版）：k8s 代码本身**没有**对 `KeyUsageCertSign` 的显式校验，只强制叶子证书具备 `ExtKeyUsageClientAuth`（EKU）。`keyCertSign` 约束来自 **Go 标准库**的链验证：`Verify → CheckSignatureFrom` 要求签发者 `IsCA` 且（当 `KeyUsage != 0` 时）含 `KeyUsageCertSign`。叶子证书不含 CertSign 是 kubeadm/client-go 的**生成惯例**（`isCA=false` 不加）。真正的边界是「私钥持有」+「上述链验证约束」，而非 k8s 应用层的权限检查。

> **参见**：`staging/src/k8s.io/apiserver/pkg/authentication/request/x509/x509.go:280`（只设 `ExtKeyUsageClientAuth`）、`cmd/kubeadm/app/util/pkiutil/pki_helpers.go:621`（`isCA` 才加 CertSign）

### 4.6 认证 → 授权 → 准入控制

在第一章提到的请求处理流程中，三个阶段职责如下（它们只是 apiserver 完整请求过滤链中的三环，全链约 25 层见 2.1）：

```
请求进入 apiserver
    │
    ├─ 认证（Authentication）：确认你是谁
    │     来源：客户端证书 CN/O、ServiceAccount JWT、请求头
    │     本章 4.2 ~ 4.4 描述的内容
    │
    ├─ 授权（Authorization）：确认你能做什么
    │     RBAC 匹配 Role / RoleBinding / ClusterRole
    │     依据 4.3 提取出的用户名和组
    │
    ├─ 准入控制（Admission）：确认请求本身是否合法
    │     变更操作前的策略校验，如 NamespaceLifecycle、
    │     LimitRanger、ResourceQuota、NodeRestriction 等
    │
    └→ 写入 etcd
```

三者必须全部通过。合法证书只能保证通过第一道，后续两道由完全独立的机制把关。

---

## 五、资源组织与配额

### 5.1 资源管理组件

| 组件 | 作用 |
|------|------|
| **Namespace** | 逻辑隔离、资源组织层 |
| **ResourceQuota** | 限制命名空间资源总量 |
| **LimitRange** | 设置容器资源默认值和范围 |
| **PriorityClass** | 控制调度和抢占优先级 |

### 5.2 QoS 等级

| 等级 | 条件 | 驱逐优先级 |
|------|------|-----------|
| **Guaranteed** | 所有容器都设置了 requests 和 limits，且 cpu/memory 的 requests == limits | 最后被驱逐 |
| **Burstable** | 至少一个容器设置了 requests 或 limits | 其次被驱逐 |
| **BestEffort** | 所有容器都没有设置 requests 和 limits | 首先被驱逐 |

> 注：驱逐排序键实际是「是否超出 requests → Pod 优先级（数值低者先驱逐）→ **超出 requests 部分**的内存用量」，QoS 不参与排序，其驱逐顺序是由此间接产生的实践效果（BestEffort 必然"超 requests"、Guaranteed 通常低于 requests）；磁盘信号同构，PID 信号只有后两键。OOM 场景则由 `GetContainerOOMScoreAdjust` 按 QoS 显式区分：Guaranteed=-997、BestEffort=1000、Burstable=`1000-1000×request/capacity`（下限 3）。
>
> **参见**：`pkg/kubelet/eviction/helpers.go:816`（`rankMemoryPressure`）、`pkg/kubelet/qos/policy.go:47`

---

## 六、调度机制

### 6.1 调度考量因素

| 因素类别 | 具体因素 |
|---------|---------|
| **资源因素** | CPU/Memory/Storage/自定义资源需求 vs 节点可用量 |
| **亲和性/反亲和性** | Node Affinity, Pod Affinity, Pod Anti-Affinity, Taint/Toleration |
| **拓扑分布** | Zone/Region/Node/Rack 分布均匀性 |
| **存储因素** | PV/PVC 绑定条件、StorageClass 要求、CSI 驱动兼容性 |
| **性能因素** | 镜像本地性、资源均衡、历史调度 |

### 6.2 拓扑分布

拓扑分布会考量**机房、机架、节点**等多层级信息：

| 拓扑层级 | 拓扑键 | 使用场景 |
|---------|--------|---------|
| **节点** | `kubernetes.io/hostname` | 跨节点分布，避免单点故障 |
| **可用区** | `topology.kubernetes.io/zone` | 跨可用区分布，提高可用性 |
| **区域** | `topology.kubernetes.io/region` | 跨区域分布，容灾 |
| **机架** | `rack` (自定义) | 跨机架分布，避免机架故障 |
| **自定义** | `disktype`, `gpu-type` 等 | 特定资源分布 |

### 6.3 调度框架扩展点与默认插件

**15 个可配置扩展点**（`config.Plugins` 口径，MultiPoint 仅为聚合配置项）：

```
PreEnqueue、QueueSort、PreFilter、Filter、PostFilter、PreScore、Score、
Reserve、Permit、PreBind、Bind、PostBind、PlacementGenerate、
PlacementScore、PodGroupPostFilter
```

- 传统文档常说的「12 个扩展点」（PreEnqueue→PostBind）在 1.36/1.37 因 PodGroup/Workload 调度新增了 PlacementGenerate、PlacementScore、PodGroupPostFilter 三个
- 框架内部另有 Sign、PlacementFeasible 两个非配置型插件接口

**默认插件与 Filter 数量**：v1.37 默认启用 **20 个插件**（基础 18 个 + feature gate 默认开启的 NodeDeclaredFeatures、DynamicResources），其中实现 Filter 扩展点的恰为 **14 个**：

```
NodeName、NodeUnschedulable、TaintToleration、NodeAffinity、NodePorts、
NodeResourcesFit、VolumeRestrictions、NodeVolumeLimits(CSI)、VolumeBinding、
VolumeZone、PodTopologySpread、InterPodAffinity、NodeDeclaredFeatures、
DynamicResources
```

该数字随 feature gate 波动（如关闭 DRA 相关 gate 则为 12 个经典基础 Filter 插件）。

> **参见**：`pkg/scheduler/apis/config/v1/default_plugins.go:32`（默认插件列表）、`pkg/scheduler/apis/config/v1/types.go:179`（Plugins 结构 16 项）、`pkg/scheduler/framework/runtime/framework.go:127`（运行时 15 个扩展点）

### 6.4 调度队列与排序

**调度队列是调度器进程内的单一 PriorityQueue 实例，不按 Namespace 分组**：

| 对比项 | Namespace | 调度队列 |
|--------|-----------|---------|
| **层级** | 资源组织层 | 调度器内部 |
| **作用** | 逻辑隔离、资源配额 | 管理待调度 Pod |
| **可见性** | 用户可见 | 用户不可见 |
| **是否分组** | 按 Namespace 分组 | **全局单一队列，不按 Namespace 分组** |

- 队列内部含三个子结构：**activeQ**（优先级堆）、**backoffQ**、**unschedulable**（调度失败池；PodGroup 场景另有 pending/incomplete 结构）
- 排序由 `PrioritySort` 插件实现：**优先级降序，同优先级按入队时间戳先后**
- 时间戳是**入队时刻**（`p.clock.Now()`），不是 Pod 的 CreationTimestamp；从 unschedulable 池重新入队时会刷新
- 排序比较函数中**完全没有 Namespace 参与**

> **参见**：`pkg/scheduler/backend/queue/scheduling_queue.go`（PriorityQueue 与三个子结构）、`pkg/scheduler/framework/plugins/queuesort/priority_sort.go:40`

---

## 七、网络与负载均衡

> 本章讲**规则与流量的机制**；kube-proxy 进程本身的职责、watch 对象与同步节奏见 2.6。

### 7.1 ClusterIP 负载均衡

ClusterIP 的负载均衡由 **kube-proxy** 实现：

| 组件 | 职责 |
|------|------|
| **kube-apiserver** | 存储 Service 和 EndpointSlice |
| **kube-controller-manager** | EndpointSlice Controller 维护 Service → Pod 映射 |
| **kube-proxy** | 生成网络规则，实现负载均衡 |
| **CoreDNS** | 提供 Service DNS 解析 |

> **参见**：`pkg/controller/endpointslice/endpointslice_controller.go:368`

### 7.2 CoreDNS 负责 DNS 解析

```
Pod 访问 Service
    ↓
1. DNS 查询: my-service.default.svc.cluster.local
    ↓
2. CoreDNS 解析 → 返回 ClusterIP (10.96.0.100)
    ↓
3. Pod 发送请求到 ClusterIP
    ↓
4. kube-proxy 网络规则拦截 → 转发到后端 Pod
```

### 7.3 ClusterIP 是虚拟 IP

**ClusterIP 是一个虚拟 IP。在 iptables/nftables 模式下，它不存在于任何网络接口上；在 ipvs 模式下，它会被绑定到 dummy 网卡 `kube-ipvs0` 上（使 VIP 只在本机可达；`arp_ignore`/`arp_announce` 仅在 `--ipvs-strict-arp` 开启时才设置，该选项默认 false）。**

```
ClusterIP (10.96.0.100)
    │
    │ 不是一个 Pod，不是一台机器
    │
    ▼
kube-proxy 在每个节点上生成网络规则
    │
    ├── iptables 模式: 生成 iptables 规则（DNAT，不绑定地址）
    ├── ipvs 模式: 生成 ipvs 虚拟服务器（并绑定 ClusterIP 到 kube-ipvs0）
    └── nftables 模式: 生成 nftables 规则
    │
    ▼
拦截访问 10.96.0.100 的流量 → DNAT/转发到后端 Pod
```

> **参见**：`pkg/proxy/ipvs/proxier.go:94`（`defaultDummyDevice = "kube-ipvs0"`）、`:295`（strict-arp 才设 arp 参数）

### 7.4 kube-proxy 不处理流量

**iptables/ipvs/nftables 模式下，kube-proxy 不处理流量，它只编程内核规则。**（数据面转发由内核 netfilter/IPVS 完成。）

例外与边界：

- **userspace 模式**（进程转发流量）已于 **1.26 移除**
- **iptables 模式 localhost NodePort**：走内核 `route_localnet` 路径（IPv4 默认开启 `LocalhostNodePorts`），非用户态
- **nftables 模式 localhost NodePort**：由**用户态 TCP 代理**服务，仅 TCP；需 `KubeProxyNFTablesLocalhostNodePorts` feature gate（**1.37 Alpha，默认关闭**）且 `--nodeport-addresses` 含 loopback，UDP/SCTP 走 reject 映射
- kube-proxy 另维护 healthz 端口，按本地 endpoint 存活应答（不转发流量）

```
Pod A 访问 ClusterIP (10.96.0.100)
    │
    │ 不是发给 kube-proxy 进程
    ▼
Linux 内核网络栈拦截流量
    │
    │ 匹配 kube-proxy 生成的 iptables/ipvs 规则
    ▼
DNAT 转换目标地址 (10.96.0.100 → Pod IP)
    │
    ▼
直接转发到后端 Pod B
```

> **参见**：`pkg/proxy/nftables/proxier.go:293`（gate 判定）、`pkg/proxy/localnodeportproxy/proxy.go`、`pkg/proxy/iptables/proxier.go:223`（route_localnet）

### 7.5 ClusterIP 生成流程

```
用户创建 Service (不指定 clusterIP)
    ↓
kube-apiserver (registry 层 BeginCreate 钩子分配 ClusterIP，
                不是 Admission Controller；
                REST.beginCreate → allocateCreate → txnAllocClusterIPs，
                存储事务内分配，失败回滚)
    ↓
etcd (存储 Service 对象)
    ↓
kube-proxy (Watch 到 Service 变化，生成本地网络规则)
```

> **参见**：`pkg/registry/core/service/storage/storage.go:342`（beginCreate 注释「This has to happen here…」）、`pkg/registry/core/service/storage/alloc.go:85`

### 7.6 kube-proxy 规则同步

**每个节点上的 kube-proxy 都会生成相同的网络规则：**

- kube-proxy 在**每个节点**上运行
- 每个 kube-proxy **独立监听** kube-apiserver
- 所有节点生成**相同的网络规则**
- 没有中心化的"同步"机制，而是**各自监听，各自生成**

### 7.7 Linux 网络规则管理

```
用户空间：kube-proxy → iptables/ipvsadm/nft 命令
    ↓
内核空间：Netfilter 框架 / IPVS 模块
    ↓
网络设备：eth0, cni0, vethxxx ...
```

| 层级 | 组件 | 作用 |
|------|------|------|
| **用户空间** | kube-proxy | 监听 Service/EndpointSlice，生成规则 |
| **用户空间工具** | iptables/ipvsadm/nft | 配置内核规则 |
| **内核空间** | Netfilter | 数据包过滤、NAT、转发 |
| **内核空间** | IPVS | 内核级负载均衡 |

---

## 八、节点管理与心跳检测

### 8.1 心跳机制

kubelet 定期向 kube-apiserver 上报心跳，**不是 kube-apiserver 检查心跳**，而是 **kube-controller-manager 中的 Node Lifecycle Controller** 负责监控。

| 方式 | 更新内容 | 数据量 | 推荐 |
|------|---------|--------|------|
| **NodeStatus** | 完整 Node.status | 大 (几 KB ~ 几十 KB) | 否 |
| **Lease** | 只更新 renewTime | 极小 (几个字节) | 是 |

Lease 细节：

- 命名空间为 **`kube-node-lease`**（不是 kube-system），Lease 名与节点同名
- 资源类型 `coordination.k8s.io/v1`，`ownerReference` 指向 Node 对象
- `leaseDurationSeconds=40`，kubelet 每 **10 秒**续约一次（40 × 0.25）
- Node Lifecycle Controller 以该 Lease 的 `renewTime` 作为心跳信号

> **参见**：`pkg/kubelet/kubelet.go:1155`（lease controller）、`staging/src/k8s.io/component-helpers/apimachinery/lease/controller.go`、`pkg/controller/nodelifecycle/node_lifecycle_controller.go:956`

### 8.2 心跳超时判断

```
配置参数：
  node-monitor-grace-period: 50s   # 节点监控宽限期（1.32 起默认 50s，
                                   # 由 40s 提高；参数在 kube-controller-manager，
                                   # 默认值见 pkg/controller/nodelifecycle/config/v1alpha1/defaults.go）
  默认 tolerationSeconds: 300      # not-ready/unreachable taint 的默认容忍时长，
                                   # 由 apiserver 的 DefaultTolerationSeconds 准入插件注入

判断流程：
  kubelet 停止心跳
      ↓
  50s 后：Node Lifecycle Controller 将节点 Ready 条件置为 Unknown
         并打上 node.kubernetes.io/not-ready 或 unreachable taint
      ↓
  默认 300s（5 分钟）容忍期满后：Taint Eviction 驱逐节点上的 Pod
      ↓
  节点对象保留（NotReady），需要手动删除
```

- taint 的**产生方**是 Node Lifecycle Controller；300s 默认容忍的**注入方**是 `DefaultTolerationSeconds` 准入插件（`plugin/pkg/admission/defaulttolerationseconds/admission.go:44`）
- **例外**：DaemonSet Pod 获得无限期容忍，不被驱逐（`pkg/controller/daemon/util/daemonset_util.go:47`）

> **参见**：`pkg/controller/nodelifecycle/node_lifecycle_controller.go:992`（置 Unknown 的 `tryUpdateNodeHealth`）、`pkg/controller/tainteviction/taint_eviction.go:160`

### 8.3 节点退役

**节点对象不会自动删除，需要手动清理：**

```bash
# 推荐退役流程
kubectl cordon node-1        # 标记不可调度
kubectl drain node-1         # 驱逐 Pod
kubectl delete node node-1   # 删除节点对象
```

---

## 九、资源隔离技术

### 9.1 cgroup (Control Groups)

kubelet 通过 **cgroup** 实现资源细粒度隔离：

```
kubelet → CRI → 容器运行时 (containerd/CRI-O) → cgroup → Linux 内核
```

### 9.2 资源隔离类型

| 资源类型 | cgroup 控制器 | 限制机制 |
|---------|--------------|---------|
| **CPU** | cpu | CFS 配额制 (cpu.cfs_quota_us) |
| **Memory** | memory | 硬限制 (memory.limit_in_bytes) |
| **Disk I/O** | blkio | 读写带宽限制 |
| **Network** | (其他机制) | kube-proxy / CNI |
| **GPU** | Device Plugin | 设备挂载 |

### 9.3 kubelet 资源管理组件

| 组件 | 位置 | 职责 |
|------|------|------|
| **Container Manager** | `pkg/kubelet/cm/` | 管理容器资源分配，配置 cgroup |
| **CPU Manager** | `pkg/kubelet/cm/cpumanager/` | CPU 独占分配 (static) / 共享分配 (default) |
| **Memory Manager** | `pkg/kubelet/cm/memorymanager/` | 内存独占分配，NUMA 感知 |
| **Topology Manager** | `pkg/kubelet/cm/topologymanager/` | 拓扑感知资源分配，NUMA 对齐 |
| **Device Manager** | `pkg/kubelet/cm/devicemanager/` | GPU 等设备管理（另含 DRA 支持 `cm/dra/`） |
| **Eviction Manager** | `pkg/kubelet/eviction/`（**不在 cm/ 下**，由 kubelet.go 直接创建） | 资源压力检测，Pod 驱逐 |

---

## 十、工作负载类型

### 10.1 分类

```
长期运行型 (Long-Running)
├── Deployment          - 无状态应用（Web 服务、API 服务），管理 ReplicaSet
├── ReplicaSet          - 维护 Pod 副本数（通常由 Deployment 管理）
├── StatefulSet         - 有状态应用（数据库、消息队列）
└── DaemonSet           - 守护进程（日志收集、监控代理）

任务型 (Batch)
├── Job                 - 一次性任务（数据迁移、批量处理）
└── CronJob             - 定时任务（定时备份、定时报告）
```

### 10.2 对比

| 工作负载 | 用途 | 副本管理 | 典型场景 |
|---------|------|---------|---------|
| **Deployment** | 无状态应用 | 滚动更新 | Web 服务、API |
| **ReplicaSet** | 维护副本数 | 保持期望副本 | 一般不直接使用，由 Deployment 管理 |
| **StatefulSet** | 有状态应用 | 有序部署 | 数据库、消息队列 |
| **DaemonSet** | 守护进程 | 每节点一个 | 日志、监控 |
| **Job** | 一次性任务 | 完成即停 | 数据迁移 |
| **CronJob** | 定时任务 | 定时创建 | 定时备份 |

### 10.3 选择指南

```
应用是否需要长期运行？
├── 否 → Job 或 CronJob
│   ├── 是否需要定时执行？→ CronJob
│   └── 否 → Job
│
└── 是 → Deployment, StatefulSet, 或 DaemonSet
    ├── 是否需要每个节点运行一个？→ DaemonSet
    ├── 是否有状态（需要稳定存储/网络）？→ StatefulSet
    └── 否 → Deployment
```

---

## 十一、Pod 生命周期

### 11.1 Pod 生命周期阶段

```
Pending → Running → Succeeded/Failed（另有 Unknown，已弃用，2015 年起 kubelet 不再设置）
```

| 阶段 | 说明 |
|------|------|
| **Pending** | Pod 已创建，但容器尚未全部启动 |
| **Running** | 至少一个容器正在运行 |
| **Succeeded** | 所有容器成功终止且不会重启 |
| **Failed** | 所有容器终止，至少一个容器失败 |

> `PodUnknown` 枚举值保留用于兼容历史数据（控制器侧仍处理），但 kubelet 现已永不写入（`getPhase` 只返回其余四个值）。
>
> **参见**：`staging/src/k8s.io/api/core/v1/types.go:3731`（Deprecated 注释）、`pkg/kubelet/kubelet_pods.go:1672`

### 11.2 容器生命周期阶段

```
Waiting → Running → Terminated
```

### 11.3 容器类型

| 类型 | 说明 |
|------|------|
| **Init Container** | 在主容器之前执行，顺序执行 |
| **Ephemeral Container** | 运行时添加的容器，用于调试 |
| **Sidecar Container** | `restartPolicy: Always` 的 Init 容器，与主容器一起运行，提供辅助功能 |

> Sidecar 由 **SidecarContainers** feature gate 管理：1.28 alpha → 1.29 beta 默认开启 → **1.33 GA** → 1.37 起 gate 移除、无条件启用。

### 11.4 健康检查影响

| 探针 | 失败后果 |
|------|---------|
| **Startup Probe** | 容器重启 |
| **Liveness Probe** | 容器重启 |
| **Readiness Probe** | 不接收流量（容器继续运行） |

### 11.5 RestartPolicy

| 策略 | 说明 |
|------|------|
| **Always** (默认) | 容器终止后总是重启 |
| **OnFailure** | 只有容器失败时才重启 |
| **Never** | 容器终止后不重启 |

---

## 总结

| 概念 | 关键点 |
|------|--------|
| **集群架构** | 控制面 4 + 工作节点 2 + 可选 1 个服务进程；KCM 注册 49 个控制器、默认启动约 43 个 |
| **感知变化** | 统一 List/Watch：resourceVersion = etcd revision，事件只传 key，本地缓存 + workqueue 对账（见 1.3） |
| **组件职责** | apiserver 是唯一读写 etcd 者；kubelet 三源 mux + syncLoop；kube-proxy 事件驱动 30s 同步；ccm 承接云 API 控制器（见第二章） |
| **选主机制** | 通过 `coordination.k8s.io/v1` Lease（kube-system），经 kube-apiserver 间接使用 etcd（2s 续约 / 10s 放弃 / 15s 过期） |
| **认证鉴权** | 三个 CA 隔离信任边界，CN 是用户名 O 是组，认证 ≠ 授权；certSign 约束来自 Go stdlib 链验证而非 k8s 校验 |
| **资源管理** | Namespace 隔离 + ResourceQuota 配额 + LimitRange 限制；QoS 不参与驱逐排序，仅间接影响 oom_score_adj |
| **调度机制** | 单一 PriorityQueue 按优先级排序（同级按入队时间），不按 Namespace；15 个可配置扩展点，14 个默认 Filter 插件（v1.37） |
| **负载均衡** | ClusterIP 是虚拟 IP（ipvs 绑到 kube-ipvs0，strict-arp 默认关），kube-proxy 只编程内核规则不转发流量 |
| **心跳检测** | kubelet 每 10s 续约 `kube-node-lease` Lease，Node Lifecycle Controller 监控（宽限期 50s，300s 容忍后驱逐） |
| **资源隔离** | cgroup 实现 CPU/Memory/Disk 等细粒度隔离；Eviction Manager 独立于 cm/ |
| **工作负载** | 6 种类型：Deployment, ReplicaSet, StatefulSet, DaemonSet, Job, CronJob |
