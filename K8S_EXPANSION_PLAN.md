# SEED Emulator 单机百万节点 Kubernetes 深度架构演进规划

## 1. 执行摘要 (Executive Summary)

本文档旨在针对 **单台多核高配置服务器上模拟 1,000,000 (一百万) 个网络节点** 这一极端目标，提供一份深度的技术分析与架构演进路线图。

虽然 `seed-emulator` 目前的 Docker/Kubernetes 编译器提供了基础的仿真能力，但其核心的 **1:1 映射模型**（即 1 个仿真节点对应 1 个容器/Pod）在面临百万级规模时将遭遇不可逾越的内核与管控瓶颈。之前的分析虽然正确地指出了 Kubernetes 和 Multus 的必要性，但它们主要解决的是 *分布式互联* 问题，并未解决 *单机资源极限* 问题。

为了实现“单机百万节点”并达到“资源理性 (Resource Rationality)”，我们需要彻底重构底层执行模型，从 **静态编译架构** 转向 **动态 Operator 聚合架构**，并引入 **用户态网络 (User-Space Networking)** 技术，最终通过 Kubeflow 实现 AI 驱动的仿真生命周期管理。

---

## 2. 现状深度剖析与瓶颈分析 (Deep Analysis of Current State)

### 2.1 现有架构 (Kubernetes Compiler)
目前的 `seedemu/compiler/Kubernetes.py` 实际上是一个“翻译器”。
*   **逻辑：** 它遍历 `Emulator` 对象中的图结构，为每个 `Node` 生成一个 K8s `Deployment`，为每个 `Network` 生成一个 `NetworkAttachmentDefinition`。
*   **数据平面：** 依赖 Linux Kernel 的网络命名空间 (`netns`) 和 `veth` 对，或者 CNI 插件（如 macvlan）。
*   **局限性：** 这种模式在几千个节点时表现良好，但在十万、百万级时会直接崩溃。

### 2.2 “之前分析”的再评估与批判
之前的分析提到了 *"Multus CNI + VXLAN 是解决方案"*，这一点在 *分布式多机* 环境下是完全正确的，但在 *单机百万节点* 场景下存在致命误区：

| 关键论点 | 分布式场景 (多机) | 单机百万节点场景 (单机) | 结论 |
| :--- | :--- | :--- | :--- |
| **"K8s 打破 RTNL 锁"** | ✅ 正确。10 台机器有 10 个内核，RTNL 锁压力被物理分摊。 | ❌ **错误**。单机只有一个内核。创建 100万个 `netns` 和 `veth` 对会导致 RTNL (Routing Netlink) 锁竞争极其严重，任何网络变更（如链路启停）都将导致系统卡死。 | 单机必须绕过内核协议栈。 |
| **"Multus + VXLAN"** | ✅ 正确。跨机 L2 互联的标准解法。 | ❌ **低效**。单机内两点通信，若还要经过 VXLAN 封包/解包，是纯粹的 CPU 浪费。 | 单机内应使用共享内存 (Shared Memory) 通信。 |
| **"1 Node = 1 Pod"** | ✅ 可行 (在几百台机器上)。 | ❌ **不可能**。Kubelet 默认上限约 110 Pods，即便调优也难超 500。百万 Pod 需要几千个 Kubelet 进程，这是不现实的。 | 必须采用 **聚合 (Aggregation)** 模型。 |

### 2.3 核心瓶颈：为什么单机跑不了 100 万个容器？
1.  **进程开销 (Process Overhead)：** 即使容器只是休眠，100万个 Pause 容器进程也会消耗几十 GB 的内存用于页表和内核结构，且调度器 (Scheduler) 压力巨大。
2.  **文件系统 IO (FS/IO)：** 目前的编译器为每个节点生成独立的 Docker Context 和 Dockerfile。生成 100万个文件夹和文件的 IO 操作是不可接受的。
3.  **网络栈内存 (TCP/IP Stack)：** 每个 Linux 网络命名空间都有独立的 TCP/IP 栈副本。百万个副本将耗尽内核内存。

---

## 3. 核心变革：聚合模型与用户态网络 (The Aggregation Model)

为了达成目标，我们必须解耦 **仿真逻辑单元 (Simulation Node)** 与 **基础设施执行单元 (Execution Pod)**。

### 3.1 概念：仿真区域 (Simulation Zone)
我们不再为每个路由器创建一个 Pod，而是创建一个 **仿真区域 Pod (Simulation Zone Pod)**，该 Pod 内部托管成百上千个轻量级仿真节点。

*   **1 Pod = 1 AS (或多个 AS)：** 比如 AS100 包含 500 个路由器，这 500 个路由器全部运行在同一个 Pod 的同一个容器进程内（或一组协作进程内）。
*   **虚拟化技术选型：**
    *   **方案 A (进程级虚拟化)：** 修改 BIRD 等路由软件，使其支持在单进程内通过配置文件区分不同“虚拟节点” (类似于 VRF)。
    *   **方案 B (用户态协议栈 - 推荐)：** 使用 **VPP (Vector Packet Processing)** 或 **LwIP** 在用户态实现 TCP/IP 栈。
        *   Host OS 内核只看到一个进程。
        *   该进程内部维护 1000 个虚拟的路由表和邻居表。
        *   数据包在进程内部通过内存拷贝（甚至零拷贝）直接流转，完全不经过 Host Kernel，彻底避开 RTNL 锁。

### 3.2 资源理性 (Resource Rationality)
*   **按需分配：** 空闲的节点（如仅作为背景流量的 Host）不应占用 CPU 时间片。在用户态轮询模式下，可以轻松实现几万个空闲节点几乎零 CPU 占用。
*   **动态伸缩：** 如果某个 AS 遭受 DDoS 攻击模拟，流量激增，K8s Operator 可以动态将该 AS 从“高密度 Pod”迁移到“独占高性能 Pod”。

---

## 4. 架构蓝图：Kubernetes Native Evolution

我们将废弃静态的 `Compiler`，构建一个动态的 **Seed Emulator Operator**。

### 4.1 自定义资源定义 (CRDs)
我们将定义一套声明式的 API 来描述网络仿真：

1.  **`Topology` (全局拓扑):** 定义整个网络的图结构（AS 关系、链路）。
2.  **`SimulationZone` (仿真区域):** Operator 计算出的“切片”。例如 `Zone-A` 包含 AS1-AS10。
3.  **`VirtualLink` (虚拟链路):** 描述跨 Zone 的连接。

### 4.2 控制器逻辑 (Controller Logic)
Operator 的核心是一个智能调度器：
1.  **输入：** 用户提交 `Topology` YAML（包含 100万节点定义）。
2.  **切片与装箱 (Bin-packing)：** Operator 根据节点类型（核心路由 vs 边缘主机）和预估负载，将 100万个节点分配到 N 个 `SimulationZone` 中。
    *   *策略：* 核心路由器放入低密度 Zone (1 Pod = 10 Router)。
    *   *策略：* 边缘僵尸网络主机放入高密度 Zone (1 Pod = 5000 Hosts)。
3.  **部署：** Operator 创建 `StatefulSet` 来运行这些 Zone。
4.  **布线：**
    *   **Zone 内布线：** 纯内存指针传递。
    *   **同机跨 Zone 布线：** 利用共享内存 (Shared Memory / Memif) 接口，极高吞吐。
    *   **跨机 Zone 布线：** 自动配置 VXLAN/Geneve 隧道（仅在不得不跨机时使用）。

---

## 5. Kubeflow 与 AI 深度集成

Kubeflow 不仅仅是“跑实验”的工具，更是实现“资源理性”的大脑。

### 5.1 实验编排 (Experiment Orchestration)
利用 **Kubeflow Pipelines (KFP)** 管理仿真全生命周期：
*   **Stage 1: Gen (生成):** 调用 Python 脚本生成超大规模拓扑数据。
*   **Stage 2: Plan (规划):** AI 模型预测各 AS 流量负载，生成最优的 Zone 切分方案。
*   **Stage 3: Deploy (部署):** 提交 CRD 给 Operator，拉起百万节点。
*   **Stage 4: Inject (注入):** 启动大规模流量发生器 (Traffic Generator)。
*   **Stage 5: Train (训练):** 实时采集数据，在线训练网络防御模型。

### 5.2 智能调度闭环
1.  **Prometheus/eBPF 监控：** 实时监控每个 Zone Pod 的 CPU/内存/包转发率。
2.  **AI 推理 (Inference)：** Kubeflow 中的模型判断某节点是否过载。
3.  **Re-scheduling：** 触发 Operator 动态调整：将过载的虚拟路由器“热迁移”到新的 Pod 中。

---

## 6. 实施路线图与阶段拆解 (Detailed Roadmap)

### 第一阶段：Operator 基础架构建设 (Phase 1: Foundation)
目标：建立 K8s Operator 骨架，验证 CRD 驱动的部署模式，暂时保持 1:1 映射以跑通流程。
*   **任务 1.1:** 设计 CRD (`Topology`, `EmulatedNode`, `EmulatedLink`) 的 Go/Python 结构体。
*   **任务 1.2:** 使用 Kubebuilder 或 Kopf 构建 Operator 框架。
*   **任务 1.3:** 实现 `Reconcile` 逻辑，使其能读取 CRD 并生成原本由 Python Compiler 生成的 Deployment/Service。
*   **任务 1.4:** 验证 Multus CNI 在 Operator 模式下的自动化配置。

### 第二阶段：聚合运行时研发 (Phase 2: Aggregation Runtime)
目标：打破 1:1 限制，实现 1 Pod 运行多个 Node 的“富容器”模式。
*   **任务 2.1:** 开发 `seed-runtime-supervisor`。这是一个轻量级守护进程，负责在容器内启动和管理多个 BIRD 进程。
*   **任务 2.2:** 实现基于 Linux Network Namespace 的轻量级隔离（在容器内再开 Netns，不依赖 K8s 管理）。
*   **任务 2.3:** 改造 Operator，增加 `AggregationPolicy`，支持将同一个 AS 的所有路由器调度到一个 Pod 中。

### 第三阶段：用户态网络高性能改造 (Phase 3: High-Performance Data Plane)
目标：引入用户态协议栈，彻底解决单机百万节点的内核瓶颈。
*   **任务 3.1:** 调研并选型用户态网络栈 (VPP vs LwIP vs User-mode Linux)。对于仿真场景，兼容性（能否运行标准 BIRD）是关键。
    *   *尝试方向：* LD_PRELOAD 劫持 socket 调用，对接用户态栈。
*   **任务 3.2:** 开发 `Memif` (Memory Interface) CNI 插件，实现同机 Pod 间的纳秒级通信。
*   **任务 3.3:** 实现“虚拟时间”同步机制，确保在 CPU 跑满时仿真结果依然准确（可选）。

### 第四阶段：Kubeflow 与智能化 (Phase 4: AI Integration)
目标：实现“资源理性”，让 AI 接管资源调度。
*   **任务 4.1:** 封装 `SeedEmulatorOp` 为 Kubeflow Component。
*   **任务 4.2:** 构建“流量预测模型”，输入拓扑图，输出各节点预估负载热力图。
*   **任务 4.3:** 实现 Operator 的“动态重平衡 (Rebalancing)” 逻辑，根据实时负载调整 Pod 副本数和节点分布。

### 第五阶段：彻底解放 (Phase 5: Liberation)
目标：开源社区化与生态建设。
*   **任务 5.1:** 构建 Web UI (基于 Backstage 或独立前端)，可视化展示百万节点的实时状态（利用 WebGL/Canvas）。
*   **任务 5.2:** 建立插件市场，允许用户上传自定义的“虚拟网元”镜像（如防火墙、IDS）。
