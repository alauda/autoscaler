# OCP 与社区 Cluster Autoscaler 实现调研

本文调研 OpenShift Container Platform（OCP）Cluster Autoscaler、社区 Kubernetes Cluster Autoscaler 以及 Cluster API（CAPI）autoscaling 的实现方式，为 ACP 设计 Cluster Autoscaler 产品化方案提供依据。

本文只讨论节点自动扩缩容，不覆盖 HPA、VPA、KEDA 等 workload 级弹性能力。

## 1. 摘要

核心结论：

- OCP 的价值在于产品化体验：用户通过 `ClusterAutoscaler` 和 `MachineAutoscaler` 两类 CR 管理集群级策略和节点组级 min/max。
- OCP 的底层依赖 OpenShift Machine API，不适合作为 ACP 的直接底层模型，除非 ACP 也采用 OpenShift Machine API。
- 社区 Cluster Autoscaler 已内置 `clusterapi` cloud provider，可以通过 CAPI `MachineDeployment` / `MachineSet` / `MachinePool` 管理节点组。
- ACP 基础设施 provider 已基于 CAPI 实现，因此更适合复用社区 Cluster Autoscaler + `clusterapi` provider，而不是开发自己的 cloud provider。
- 从 0 扩容是特殊能力，需要准确的新节点调度画像；不能默认认为所有 provider 都天然支持。

一句话概括：**ACP 应借鉴 OCP 的 CRD 产品体验，但底层应复用社区 Cluster Autoscaler 的 CAPI 集成。**

## 2. Cluster Autoscaler 通用机制

Cluster Autoscaler 解决的是“节点数量是否需要变化”的问题。它不直接根据节点 CPU 使用率扩容，也不替代 workload 级弹性组件。

它主要关注两类信号：

```text
扩容：是否存在因资源或调度约束无法调度的 Pending Pod
缩容：是否存在长期低利用率且可以安全清空的 Node
```

### 2.1 扩容机制

典型扩容链路：

```text
发现 Pending Pod
  -> 模拟不同 node group 新增节点后能否承载这些 Pod
  -> 根据 expander 策略选择 node group
  -> 调用 provider / machine API 增加节点
  -> 新节点加入集群
  -> Pending Pod 被调度
```

扩容判断会考虑：

- CPU / memory requests。
- GPU / extended resources。
- `nodeSelector`。
- node affinity / anti-affinity。
- taint / toleration。
- topology spread。
- volume topology。
- Pod affinity / anti-affinity。

如果 Pending Pod 的根因是错误的调度约束，而不是新增节点可以解决的容量不足，Cluster Autoscaler 不一定会扩容。

### 2.2 缩容机制

典型缩容链路：

```text
发现低利用率 Node
  -> 判断低利用率是否持续足够长时间
  -> 判断 Node 上的 Pod 是否可以被驱逐
  -> 模拟这些 Pod 是否能在其他 Node 重新调度
  -> cordon / drain Node
  -> 删除底层 Machine / VM / 实例
  -> Node 从集群中移除
```

缩容比扩容更容易被阻塞，常见原因包括：

- PodDisruptionBudget 过于严格。
- Pod 使用本地存储。
- Pod 不是由 Deployment、ReplicaSet、StatefulSet、DaemonSet、Job 等控制器管理。
- Pod 标记了 `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"`。
- 节点上存在系统关键 Pod。
- Pod 搬迁后无法满足 affinity、anti-affinity、topology spread 等约束。
- 其他节点没有足够资源承载被驱逐 Pod。
- node group 已达到最小副本数。

### 2.3 expander 策略

当多个 node group 都能承载 Pending Pod 时，Cluster Autoscaler 通过 expander 选择扩容目标。

常见策略：

| expander | 含义 |
|---|---|
| `random` | 在可行 node group 中随机选择。 |
| `least-waste` | 选择新增节点后 CPU / memory 浪费更少的 node group。 |
| `priority` | 按配置的 node group 优先级选择。 |
| `most-pods` | 优先选择能承载最多 Pending Pod 的 node group。 |
| `price` | 基于成本选择，依赖 provider 能提供价格信息。 |
| `grpc` | 通过外部 gRPC expander 决策。 |

`least-waste` 优化的是装箱效率，不等同于最低成本。GPU、label、taint、zone 等通常先决定 node group 是否可行，CPU / memory waste 再决定可行候选中谁更省。

平台常见组合是：

```text
priority,least-waste
```

即先表达业务或成本偏好，再在同优先级候选中选择资源浪费更少的方案。

### 2.4 node group 抽象

Cluster Autoscaler 不直接管理所有 Node，而是管理它能识别的 node group。Node 是否属于 autoscaler 管理范围，取决于它能否映射到已发现、可扩缩容的 node group。

不同平台的 node group 抽象不同：

```text
云厂商 ASG 模型：
Cluster Autoscaler -> 云厂商 ASG / node pool -> VM -> Node

OCP 模型：
ClusterAutoscaler -> MachineAutoscaler -> MachineSet -> Machine -> Node

CAPI 模型：
Cluster Autoscaler -> MachineDeployment / MachineSet / MachinePool -> Machine -> InfraMachine -> Node
```

在 CAPI 场景中，Node 通常需要通过 `providerID` 映射到 CAPI `Machine`，再映射到对应的 node group。映射不稳定会直接影响扩缩容可靠性。

### 2.5 从 0 扩容为什么特殊

从 0 扩容的关键问题是：node group 当前没有 Node，Cluster Autoscaler 无法从现有 Node 推断新节点能力。

因此它必须从其他来源获得“虚拟新节点”的调度信息，例如：

- CPU / memory capacity。
- max pods。
- labels。
- taints。
- GPU / extended resources。
- zone / topology 信息。
- volume limit / CSI driver 信息。

如果这些信息不准确，autoscaler 可能认为新增节点可以承载 Pending Pod，但真实节点加入后 Pod 仍然无法调度。

不同 provider 获取这些信息的方式不同：有的能从 instance type 或机器模板推断，有的依赖 provider status，有的需要额外 annotations 或产品层标准化对象。

## 3. OCP Cluster Autoscaler 实现

OCP Cluster Autoscaler 是 OpenShift 面向 Machine API 的产品化集成。它不直接管理云厂商 ASG，而是通过 OpenShift Machine API 管理节点生命周期。

典型链路：

```text
ClusterAutoscaler
  -> MachineAutoscaler
  -> MachineSet
  -> Machine
  -> Node
```

### 3.1 OCP 的核心 CR

| CR | 作用范围 | 主要职责 |
|---|---|---|
| `ClusterAutoscaler` | 集群级 | 配置集群资源上限、缩容参数、部分扩容策略等全局行为。 |
| `MachineAutoscaler` | 单个 `MachineSet` | 指定某个 `MachineSet` 是否允许自动扩缩容，以及 min / max 副本数。 |
| `MachineSet` | 单个节点组 | 保存新机器模板，包括 providerSpec、failure domain、labels、taints、bootstrap 引用等。 |

`ClusterAutoscaler` 示例：

```yaml
apiVersion: autoscaling.openshift.io/v1
kind: ClusterAutoscaler
metadata:
  name: default
spec:
  resourceLimits:
    maxNodesTotal: 20
    cores:
      min: 8
      max: 200
    memory:
      min: 32
      max: 1024
  scaleDown:
    enabled: true
    unneededTime: 10m
    utilizationThreshold: "0.5"
```

`MachineAutoscaler` 示例：

```yaml
apiVersion: autoscaling.openshift.io/v1beta1
kind: MachineAutoscaler
metadata:
  name: worker-us-east-1a
  namespace: openshift-machine-api
spec:
  minReplicas: 1
  maxReplicas: 5
  scaleTargetRef:
    apiVersion: machine.openshift.io/v1beta1
    kind: MachineSet
    name: worker-us-east-1a
```

如果只创建 `ClusterAutoscaler`，但没有为目标 `MachineSet` 创建 `MachineAutoscaler`，该 `MachineSet` 通常不会被自动扩缩容。

### 3.2 OCP 如何保持模板与 Node 一致

OCP 的一致性来自同一个 `MachineSet` 同时服务于两件事：

```text
Cluster Autoscaler 调度模拟
  -> 读取 MachineSet / 现有 Node 信息

Machine API 创建真实机器
  -> 使用同一个 MachineSet.spec.template
```

如果 `MachineSet` 当前已有 Node，autoscaler 可以从真实 Node 推断 CPU、memory、labels、taints、topology、GPU 等调度信息。如果副本数为 0，则需要 OCP 对对应平台的 `MachineSet.providerSpec` 和 node template 有内置支持。

因此，`MachineAutoscaler` 本身不需要理解 AWS、Azure、vSphere 或 bare metal 模板字段；不同 provider 的模板语义由 OCP Machine API provider / autoscaler 集成处理。

### 3.3 OCP 从 0 扩容的平台支持

按 OCP 4.21 文档，`MachineAutoscaler.spec.minReplicas` 可以在以下平台上设为 `0`：

- AWS。
- Google Cloud / GCP。
- Azure。
- RHOSP / OpenStack。
- VMware vSphere。

这表示 OCP 对这些平台的 Machine API provider 和 Cluster Autoscaler 集成已经具备必要的新节点模板推断能力。未列入支持范围的平台，不应默认支持 `minReplicas: 0`。

OCP 文档也提醒不要把 IPI 安装过程中创建的三个默认 compute `MachineSet` 的 `minReplicas` 设为 `0`。更合理的用法是：默认 worker 池保留基础容量，把 `minReplicas: 0` 用在 GPU、大规格、昂贵或低频使用的专用节点组上。

### 3.4 对 ACP 的启示

OCP 值得 ACP 借鉴的是：

- 使用集群级 CR 管理 autoscaler 实例配置。
- 使用节点组级 CR 管理单个 node group 的 min/max。
- 用户不直接编辑底层 node group annotations 或 provider 参数。
- 从 0 扩容需要明确的平台支持边界。

OCP 不适合 ACP 直接复用的是：

- OCP 绑定 OpenShift Machine API。
- OCP 的 `MachineAutoscaler` 目标是 OpenShift `MachineSet`，而 ACP 的基础设施抽象是 CAPI。
- OCP 的 operator 和权限模型与 ACP global 管理集群模型不同。

## 4. 社区 Cluster Autoscaler 与 CAPI provider

社区 Cluster Autoscaler 是通用 Kubernetes 节点自动扩缩容组件。它通过 cloud provider / node group provider 接口对接不同基础设施。

常见 provider 包括 AWS、Azure、GCE / GKE、Cluster API、OpenStack Magnum、DigitalOcean、Hetzner、Linode、Oracle Cloud、Scaleway、TencentCloud、Vultr、Rancher 和 external gRPC provider。

### 4.1 配置入口

社区版通常通过 Deployment 参数、flags、provider 配置和 ConfigMap 管理。

常见参数包括：

```text
--cloud-provider
--nodes
--node-group-auto-discovery
--expander
--scale-down-enabled
--scale-down-unneeded-time
--scale-down-utilization-threshold
--skip-nodes-with-local-storage
--skip-nodes-with-system-pods
```

在 CAPI 场景中，核心参数是：

```text
--cloud-provider=clusterapi
--node-group-auto-discovery=clusterapi:namespace=<capi-cluster-namespace>,clusterName=<workload-cluster>
```

`clusterapi` 是社区 Cluster Autoscaler 内置 cloud provider 名称。它负责读取 CAPI `MachineDeployment`、`MachineSet`、`MachinePool` 等对象，并在扩缩容时调整这些对象的副本数。

不同 Cluster Autoscaler / CAPI provider 版本支持的对象类型可能不同。ACP 初版如果追求稳定产品边界，建议只暴露 `MachineDeployment`。

### 4.2 CAPI min/max 表达

社区 `clusterapi` provider 通常通过 CAPI node group 对象上的 annotations 表达 min/max。以 `MachineDeployment` 为例：

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: worker-md-0
  annotations:
    cluster.x-k8s.io/cluster-api-autoscaler-node-group-min-size: "1"
    cluster.x-k8s.io/cluster-api-autoscaler-node-group-max-size: "10"
spec:
  replicas: 3
```

如果某个 `MachineDeployment` 没有 min/max annotations，即使它也是 CAPI 管理的节点组，也通常不会被 Cluster Autoscaler 当成可扩缩容 node group。

### 4.3 CAPI scale-from-zero 信息来源

从 0 扩容需要额外的 template node 信息。在 Cluster Autoscaler `cluster-autoscaler-release-1.35` 中，`clusterapi` provider 可以从两类来源构造 scale-from-zero 所需的 template node：

- 目标 CAPI node group 上的 capacity annotations。
- infrastructure template 的 `status.capacity` / `status.nodeSystemInfo` 等状态字段。

常用 annotations：

| annotation | 必需性 | 作用 |
|---|---|---|
| `cluster.x-k8s.io/cluster-api-autoscaler-node-group-min-size` | 必需 | node group 最小规模；从 0 扩容时通常设为 `"0"`。 |
| `cluster.x-k8s.io/cluster-api-autoscaler-node-group-max-size` | 必需 | node group 最大规模。 |
| `capacity.cluster-autoscaler.kubernetes.io/cpu` | 条件必需 | 新节点 CPU capacity；provider 不能提供 capacity 时必须设置。 |
| `capacity.cluster-autoscaler.kubernetes.io/memory` | 条件必需 | 新节点 memory capacity；provider 不能提供 capacity 时必须设置。 |
| `capacity.cluster-autoscaler.kubernetes.io/ephemeral-disk` | 可选 | 新节点临时磁盘 capacity。 |
| `capacity.cluster-autoscaler.kubernetes.io/maxPods` | 可选 | 新节点最大 Pod 数；未设置时通常按 `110` 处理。 |
| `capacity.cluster-autoscaler.kubernetes.io/labels` | 可选 | 预定义节点 labels，用于模拟 `nodeSelector` / node affinity。 |
| `capacity.cluster-autoscaler.kubernetes.io/taints` | 可选 | 预定义节点 taints，用于模拟 tolerations。 |
| `capacity.cluster-autoscaler.kubernetes.io/csi-driver` | 可选 | 预定义 CSI driver 及 volume limit，例如 `ebs.csi.aws.com=25`。 |
| `capacity.cluster-autoscaler.kubernetes.io/gpu-type` | GPU 可选 | 传统 device plugin 场景的 GPU extended resource 名称，例如 `nvidia.com/gpu`。 |
| `capacity.cluster-autoscaler.kubernetes.io/dra-driver` | GPU 可选 | Kubernetes DRA 场景的 driver 名称；通常与 `gpu-type` 二选一。 |
| `capacity.cluster-autoscaler.kubernetes.io/gpu-count` | GPU 可选 | GPU 数量。 |

示例：

```yaml
metadata:
  annotations:
    cluster.x-k8s.io/cluster-api-autoscaler-node-group-min-size: "0"
    cluster.x-k8s.io/cluster-api-autoscaler-node-group-max-size: "5"
    capacity.cluster-autoscaler.kubernetes.io/cpu: "16"
    capacity.cluster-autoscaler.kubernetes.io/memory: "128Gi"
    capacity.cluster-autoscaler.kubernetes.io/ephemeral-disk: "100Gi"
    capacity.cluster-autoscaler.kubernetes.io/maxPods: "200"
    capacity.cluster-autoscaler.kubernetes.io/labels: "node-type=gpu,workload=batch,topology.kubernetes.io/zone=zone-a"
    capacity.cluster-autoscaler.kubernetes.io/taints: "workload=batch:NoSchedule"
    capacity.cluster-autoscaler.kubernetes.io/gpu-type: "nvidia.com/gpu"
    capacity.cluster-autoscaler.kubernetes.io/gpu-count: "2"
```

需要注意：

- capacity annotations 会覆盖 provider template 中由 provider 提供的 capacity 信息。
- labels 会与目标 CAPI node group 中可传播到 Node 的 labels 合并；同名 key 的冲突行为需要以目标版本为准。
- taints 的合并行为依赖目标 CAPI / Cluster Autoscaler 版本以及 taint propagation 能力。
- `maxPods` 未设置时通常按 `110` 处理。
- DRA 场景使用 `dra-driver`；传统 device plugin 场景使用 `gpu-type`，不要让两者同时代表同一个 GPU 资源。

### 4.4 priority expander ConfigMap

当 `--expander` 包含 `priority` 时，社区 priority expander 会读取固定名称的 ConfigMap：

```text
cluster-autoscaler-priority-expander
```

该 ConfigMap 位于 Cluster Autoscaler 通过 `--kubeconfig` 访问的 workload cluster 中，namespace 由 `--namespace` 指定。名称是 upstream 实现中的固定常量，不是通过启动参数为每个 autoscaler 实例单独指定。

如果该 ConfigMap 缺失或格式错误，社区 Cluster Autoscaler 会跳过 priority expander，并继续使用 expander 链中的后续策略，例如 `priority,least-waste` 中的 `least-waste`。

## 5. OCP 与社区实现差异

| 对比项 | OCP Cluster Autoscaler | 社区 Cluster Autoscaler |
|---|---|---|
| 定位 | OpenShift 集成版节点自动扩缩容 | 通用 Kubernetes 节点自动扩缩容组件 |
| 节点组抽象 | OpenShift `MachineSet` | cloud provider node group、CAPI `MachineDeployment` / `MachineSet` / `MachinePool`、external provider 等 |
| 扩缩容接口 | OpenShift Machine API | cloud provider API、CAPI API、external gRPC provider 等 |
| 单个节点组 min/max | `MachineAutoscaler.spec.minReplicas` / `maxReplicas` | provider 配置、`--nodes`、auto discovery、CAPI annotations 等 |
| 集群级配置 | `ClusterAutoscaler` CR | Deployment flags / provider config / ConfigMap |
| 运维方式 | Operator / CRD 管理 | 用户或平台维护 Deployment、RBAC、参数、凭据 |
| 产品绑定 | 绑定 OpenShift / OCP | 不绑定发行版 |
| provider 覆盖 | 以 OCP Machine API 支持范围为准 | 覆盖多云、CAPI、external provider 等 |
| 参数可调性 | 取决于 OCP API 暴露 | 可直接调整更多 flags |
| 从 0 扩容 | 依赖 MachineSet 模板和 OCP 平台支持 | 依赖 provider 能提供新节点调度属性；CAPI 场景可能依赖模板和 capacity annotations |
| 对 ACP 的适配性 | 不推荐作为底层方案 | 推荐，尤其是 `clusterapi` cloud provider |

## 6. 对 ACP 方案的输入

基于以上调研，ACP 方案设计应遵循以下判断：

1. 底层使用社区 Cluster Autoscaler + `--cloud-provider=clusterapi`。
2. 产品层借鉴 OCP，提供集群级 `ClusterAutoscaler` 和节点组级 `MachineAutoscaler`。
3. ACP 初版只把 CAPI `MachineDeployment` 作为 worker node group，不把 `MachineSet` / `MachinePool` 纳入产品 API 范围。
4. `MachineAutoscaler` 不直接做扩缩容决策，只作为 min/max 的产品层单一事实源。
5. 需要一个 ACP controller 将产品层 CR 翻译为社区 Cluster Autoscaler Deployment、ConfigMap、RBAC、flags 和 CAPI annotations。
6. 普通扩缩容不强制要求额外节点画像对象；从 0 扩容需要 provider 提供准确的新节点调度画像。
7. baremetal provider 如果要纳入 autoscaling，需要单独确认节点库存、capacity、删除语义、Machine / Node `providerID` 映射以及从 0 扩容模板来源。

详细 ACP 方案见第二篇文档：`cluster-autoscaler-acp-design.md`。

## 7. 参考资料

- [社区 Kubernetes Autoscaler 项目](https://github.com/kubernetes/autoscaler)
- [社区 Cluster Autoscaler 目录（cluster-autoscaler-release-1.35）](https://github.com/kubernetes/autoscaler/tree/cluster-autoscaler-release-1.35/cluster-autoscaler)
- [OpenShift Cluster Autoscaler Operator](https://github.com/openshift/cluster-autoscaler-operator)
- [OpenShift Kubernetes Autoscaler fork](https://github.com/openshift/kubernetes-autoscaler)
- [OpenShift Container Platform: Applying autoscaling to a cluster](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/machine_management/applying-autoscaling)
- [ClusterAutoscaler API, OCP](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/autoscale_apis/clusterautoscaler-autoscaling-openshift-io-v1)
- [MachineAutoscaler API, OCP](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/autoscale_apis/machineautoscaler-autoscaling-openshift-io-v1beta1)
- [Kubernetes Node Autoscaling](https://kubernetes.io/docs/concepts/cluster-administration/node-autoscaling/)
- [Kubernetes Cluster Autoscaler FAQ（cluster-autoscaler-release-1.35）](https://github.com/kubernetes/autoscaler/blob/cluster-autoscaler-release-1.35/cluster-autoscaler/FAQ.md)
- [Cluster Autoscaler README（cluster-autoscaler-release-1.35）](https://github.com/kubernetes/autoscaler/blob/cluster-autoscaler-release-1.35/cluster-autoscaler/README.md)
- [Cluster Autoscaler Cluster API provider README（cluster-autoscaler-release-1.35）](https://github.com/kubernetes/autoscaler/blob/cluster-autoscaler-release-1.35/cluster-autoscaler/cloudprovider/clusterapi/README.md)
- [Cluster API Book: Autoscaling](https://cluster-api.sigs.k8s.io/tasks/automated-machine-management/autoscaling)
- [Cluster API Book: MachineDeployment](https://cluster-api.sigs.k8s.io/developer/core/controllers/machine-deployment)
- [Cluster API Book: Metadata propagation](https://cluster-api.sigs.k8s.io/reference/api/metadata-propagation)
- [Kubernetes Pod Disruptions and PDB](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
