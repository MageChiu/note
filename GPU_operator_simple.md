# NVIDIA GPU Operator 完全指南

## 一、概述

NVIDIA GPU Operator 是一个 **Kubernetes Operator**，用于自动化管理 Kubernetes 集群中 GPU 资源所需的所有 NVIDIA 软件组件。它的核心目标是让管理员像管理 CPU 节点一样轻松地管理 GPU 节点，无需在每个节点上手动安装和维护 GPU 驱动、运行时等底层依赖。

## 二、解决的核心问题

在传统的 Kubernetes GPU 集群中，管理员需要在每个 GPU 节点上手动完成以下工作：

- 安装 NVIDIA GPU 驱动
- 安装 NVIDIA Container Toolkit（容器运行时）
- 配置 Kubernetes Device Plugin
- 配置监控（DCGM Exporter）等

当集群规模增长、驱动需要升级或节点异构时，这些工作会变得极其繁琐且容易出错。GPU Operator 将这些组件全部 **容器化**，以 DaemonSet 等 Kubernetes 原生方式自动部署和管理。

## 三、架构与核心组件

GPU Operator 基于 Kubernetes 的 **Operator Framework** 构建，通过 CRD（`ClusterPolicy`）来声明式管理以下组件：

| 组件 | 作用 |
|---|---|
| **NVIDIA Driver** | 以容器方式在节点上安装/管理 GPU 内核驱动 |
| **NVIDIA Container Toolkit** | 让容器运行时（containerd/CRI-O）能够访问 GPU |
| **Kubernetes Device Plugin** | 向 Kubernetes 注册 `nvidia.com/gpu` 资源，实现 GPU 调度 |
| **DCGM Exporter** | 导出 GPU 指标（温度、利用率、显存等）到 Prometheus |
| **Node Feature Discovery (NFD)** | 自动发现节点的 GPU 硬件特征并打标签 |
| **GPU Feature Discovery (GFD)** | 发现 GPU 型号、驱动版本等详细特征并打标签 |
| **MIG Manager** | 管理 NVIDIA Multi-Instance GPU（MIG）配置 |
| **VGPU Manager** | 管理 vGPU 场景下的许可和配置 |
| **Sandbox Device Plugin** | 支持虚拟化/沙箱环境中的 GPU 直通 |

工作流简化表示：

```
ClusterPolicy CR → GPU Operator Controller
    ├── DaemonSet: nvidia-driver
    ├── DaemonSet: nvidia-container-toolkit
    ├── DaemonSet: nvidia-device-plugin
    ├── DaemonSet: dcgm-exporter
    ├── DaemonSet: gpu-feature-discovery
    └── DaemonSet: nvidia-mig-manager
```

## 四、关键特性

1. **Day-0 自动化**：新 GPU 节点加入集群后，Operator 自动完成驱动安装和环境配置，实现即插即用。
2. **声明式管理**：通过单一 `ClusterPolicy` CRD 定义所有 GPU 软件栈的期望状态，Operator 负责收敛。
3. **驱动容器化**：GPU 驱动以容器方式运行，支持版本升级、回滚，无需重新制作节点镜像。
4. **MIG 支持**：可通过配置文件自动管理 A100/H100 等支持 MIG 的 GPU 的分区策略。
5. **多平台支持**：支持 x86_64 和 ARM64，兼容主流 Linux 发行版（Ubuntu、RHEL/CentOS、CoreOS 等）。
6. **与 GPU 生态集成**：原生集成 NVIDIA AI Enterprise、vGPU、Network Operator（RDMA/GPUDirect）等。

## 五、典型部署方式

通过 Helm 安装：

```bash
# 添加 NVIDIA Helm 仓库
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

# 安装 GPU Operator
helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --create-namespace
```

> 在私有的环境中需要做一些镜的替换，这里的修改就不直接写了

安装后，Operator 会自动在所有 GPU 节点上部署所需组件。可通过自定义 `ClusterPolicy` 调整行为，例如：

```yaml
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata:
  name: cluster-policy
spec:
  driver:
    enabled: true
    version: "550.90.07"
  devicePlugin:
    enabled: true
  dcgmExporter:
    enabled: true
  migManager:
    enabled: true
```

## 六、适用场景

- **大规模 GPU 集群管理**：数十到数千个 GPU 节点的统一生命周期管理
- **云原生 AI/ML 平台**：为 Kubeflow、Ray on K8s 等平台提供 GPU 基础设施层
- **多租户推理/训练集群**：结合 MIG 和 Time-Slicing 实现 GPU 细粒度共享
- **边缘计算**：在边缘 Kubernetes 集群中自动化管理 GPU 资源
- **混合云 / 多云**：跨不同基础设施统一 GPU 软件栈版本

## 七、与手动安装的对比

| 对比项 | GPU Operator | 手动安装 |
|---|---|---|
| 驱动管理 | 容器化，自动部署 | 需要在每个节点手动安装 |
| 升级方式 | 修改 CR，滚动更新 | 逐节点手动升级 |
| 一致性 | 集群级声明式保证 | 依赖运维纪律 |
| 新节点上线 | 自动配置 | 需要预装或自定义镜像 |

---

## 八、驱动安装与版本管理机制

### 8.1 核心设计理念

GPU Operator 采用 **驱动容器化（Driver Container）** 的方式，将 NVIDIA GPU 驱动封装在容器镜像中，以 DaemonSet 形式部署到集群的每个 GPU 节点上。这意味着驱动的安装、升级、回滚都变成了 Kubernetes 原生的容器生命周期管理操作。

### 8.2 驱动安装的两种模式

#### 模式一：Operator 管理驱动（默认模式）

Operator 自动在每个 GPU 节点上以容器方式安装驱动，节点本身**无需预装任何 NVIDIA 驱动**。

```yaml
apiVersion: nvidia.com/v1
kind: ClusterPolicy
spec:
  driver:
    enabled: true
    version: "550.90.07"        # 指定驱动版本
    repository: "nvcr.io/nvidia"
    image: "driver"
```

**工作原理：**

- Driver Container 启动后，在容器内编译内核模块（或使用预编译版本）
- 通过 `insmod` 将驱动模块加载到宿主机内核
- 容器持续运行，维持驱动的生命周期

#### 模式二：预装驱动模式（Pre-installed Driver）

节点已通过操作系统包管理器或自定义镜像预装了驱动，Operator 跳过驱动安装，仅管理其余组件。

```yaml
spec:
  driver:
    enabled: false   # 禁用 Operator 管理的驱动
  # 其余组件（device plugin、toolkit 等）仍然由 Operator 管理
```

**适用场景：** 使用云厂商提供的 GPU 优化镜像（如 GKE COS、EKS 优化 AMI），或有特殊内核/驱动兼容性要求时。

### 8.3 异构 GPU 型号的处理

#### 驱动的向后兼容性

NVIDIA 驱动天然具有**向后兼容性**——一个较新版本的驱动通常可以同时支持当前代和前几代 GPU。例如：

| 驱动版本 | 支持的 GPU 架构 |
|---|---|
| R550 分支 | Hopper (H100/H200)、Ada Lovelace (L40S)、Ampere (A100)、Turing、Volta 等 |
| R535 分支 | Hopper、Ampere、Turing、Volta 等 |
| R525 分支 | Hopper（初始支持）、Ampere、Turing、Volta |

因此，对于大多数异构集群，**一个统一的驱动版本就能覆盖所有 GPU 型号**。这是默认也是最简单的管理方式。

#### 需要不同驱动版本的场景

某些情况下必须对不同节点使用不同驱动版本：

- **极老的 GPU**（如 Kepler 架构）不被新驱动支持
- **特定工作负载**对某个驱动分支有严格依赖
- **vGPU 节点 vs 裸金属节点**需要不同驱动类型
- **新 GPU（如 Blackwell B200）** 需要更新的驱动分支

#### 通过 nodeSelector / nodeAffinity 实现分组

GPU Operator 利用 **Node Feature Discovery (NFD)** 和 **GPU Feature Discovery (GFD)** 自动为节点打上硬件标签：

```
nvidia.com/gpu.product=NVIDIA-A100-SXM4-80GB
nvidia.com/gpu.family=ampere
nvidia.com/gpu.compute.major=8
nvidia.com/gpu.compute.minor=0
nvidia.com/cuda.driver.major=550
nvidia.com/gpu.memory=81920
```

从 **GPU Operator v23.9+** 开始，支持通过 `ClusterPolicy` 中的 `nodeSelector` 为不同节点组指定不同驱动配置：

```yaml
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata:
  name: cluster-policy
spec:
  driver:
    enabled: true
    version: "550.90.07"       # 默认驱动版本
    nodeSelector:
      nvidia.com/gpu.family: ampere   # 仅应用于 Ampere 节点
```

#### 方案 A：多 ClusterPolicy（v24.3+）

较新版本的 GPU Operator 支持创建多个 ClusterPolicy 资源，每个绑定不同的 nodeSelector：

```yaml
# ClusterPolicy for Hopper nodes
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata:
  name: policy-hopper
spec:
  driver:
    version: "560.35.03"
    nodeSelector:
      nvidia.com/gpu.family: hopper
---
# ClusterPolicy for Ampere nodes
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata:
  name: policy-ampere
spec:
  driver:
    version: "550.90.07"
    nodeSelector:
      nvidia.com/gpu.family: ampere
```

#### 方案 B：预装驱动 + Operator 管理混合

对特殊节点使用预装驱动并打标签排除出 Operator 管理范围，其余节点由 Operator 统一管理：

```yaml
spec:
  driver:
    enabled: true
    version: "550.90.07"
    nodeSelector:
      gpu-driver-managed: "operator"   # 仅管理带此标签的节点
```

### 8.4 驱动版本升级策略

#### 滚动升级

修改 `ClusterPolicy` 中的 `driver.version` 字段即触发升级流程：

```bash
kubectl patch clusterpolicy cluster-policy \
  --type merge \
  -p '{"spec":{"driver":{"version":"555.42.02"}}}'
```

**升级过程：**

```
1. Operator 检测到 driver.version 变更
2. 按照 upgradePolicy 逐节点执行：
   a. 标记节点为 SchedulingDisabled（cordon）
   b. 可选：驱逐节点上的 GPU 工作负载（drain）
   c. 停止旧 Driver Container
   d. 启动新版本 Driver Container
   e. 等待驱动加载成功 + 健康检查通过
   f. 恢复节点调度（uncordon）
3. 继续处理下一个节点
```

#### 升级策略配置

```yaml
spec:
  driver:
    upgradePolicy:
      autoUpgrade: true           # 是否自动触发升级
      maxParallelUpgrades: 1      # 最大并行升级节点数
      maxUnavailable: "25%"       # 最大不可用比例
      waitForCompletion:
        timeoutSeconds: 0         # 0 表示无超时
      podDeletion:
        force: false              # 是否强制驱逐
        timeoutSeconds: 300       # 驱逐超时
        deleteEmptyDir: false     # 是否删除 emptyDir Pod
      drain:
        enable: true              # 升级前是否 drain 节点
        force: false
        timeoutSeconds: 300
```

#### 回滚

如果新驱动有问题，直接将 `driver.version` 改回旧版本即可：

```bash
kubectl patch clusterpolicy cluster-policy \
  --type merge \
  -p '{"spec":{"driver":{"version":"550.90.07"}}}'
```

Operator 会按照相同的滚动流程执行降级。

### 8.5 驱动容器的编译机制

Driver Container 内部的驱动安装有两种路径：

| 方式 | 说明 | 优劣 |
|---|---|---|
| **运行时编译** | 容器启动时检测宿主机内核版本，在线编译内核模块 | 灵活，但启动慢（数分钟），依赖内核头文件 |
| **预编译驱动（Pre-compiled）** | 镜像中已包含针对特定内核版本编译好的模块 | 启动快（秒级），但需要为每个内核版本维护镜像 |

配置预编译模式：

```yaml
spec:
  driver:
    usePrecompiled: true
    version: "550.90.07"
    # 镜像 tag 会包含内核版本信息
    # e.g., nvcr.io/nvidia/driver:550.90.07-5.15.0-1057-ubuntu22.04
```

**在大规模集群中推荐使用预编译驱动**——节点重启或新节点上线时，驱动加载时间从数分钟缩短到十几秒，显著减少 GPU 不可用窗口。

## 九、实际运维建议

| 场景 | 推荐策略 |
|---|---|
| 同构集群（全是 A100 或全是 H100） | 统一 `driver.version`，定期跟随 NVIDIA 长期支持分支升级 |
| 异构集群（A100 + H100 混合） | 优先使用同一驱动版本（R550+ 通常兼容），确有冲突时使用多 ClusterPolicy 分组 |
| 引入新一代 GPU（如 B200） | 先在新 GPU 节点上测试高版本驱动，通过 nodeSelector 隔离，验证后再推广 |
| 生产环境升级 | `maxParallelUpgrades: 1` + `drain.enable: true`，逐节点滚动，预留回滚窗口 |
| 超大规模集群（1000+ GPU 节点） | 使用预编译驱动 + 分批标签控制升级顺序，避免同时 drain 大量节点影响训练任务 |

## 十、与相关技术的协同

GPU Operator 与 **NVIDIA Network Operator**（管理 RDMA/GPUDirect 网络）配合使用，可以覆盖分布式训练场景下的 GPU + 高性能网络的完整基础设施栈。