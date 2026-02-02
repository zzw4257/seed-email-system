# SEED Emulator 单机百万节点 Kubernetes 深度架构演进规划

## 1. 执行摘要 (Executive Summary)

本文档针对 **单台多核高配置服务器上模拟 1,000,000 (一百万) 个网络节点** 这一极端工程目标，提供一份深度的技术分析与架构演进路线图。

本文首先深入剖析现有 **Docker/Compose 模式** 的物理边界，随后完整收录并批判性评估了关于 **"迁移至 Kubernetes" 的原始技术分析**。基于此，我们提出了一套彻底重构的 **Kubernetes Native 聚合架构**，旨在通过用户态网络 (User-Space Networking) 和 AI 驱动的资源调度，实现真正的“资源理性 (Resource Rationality)”。

---

## 2. 现状深度剖析：从 Docker 到 Kubernetes (Deep Analysis)

### 2.1 Docker-Compose 模式的能力边界 (Capabilities & Hard Limits)

在讨论未来之前，必须明确现在的 Docker-Compose 模式到底能做什么，以及绝对做不到什么。

*   **能做到的 (Capabilities):**
    *   **中小规模高保真模拟：** 在 100-1000 节点规模下，提供极高的真实度（每个节点都是完整的 Linux 环境）。
    *   **开发体验友好：** 也就是 "Laptop Scale"，单机即可运行，调试方便，文件系统直观。
    *   **拓扑所见即所得：** Linux Bridge 直观地对应虚拟网线。

*   **绝对做不到的 (Hard Limits):**
    1.  **Linux Bridge 端口限制：** 标准 Linux Bridge 性能在端口数超过 1024 后急剧下降，且有硬性上限。模拟大规模二层网络（如大型 IXP）时会直接失效。
    2.  **RTNL (Routing Netlink) 锁死：** 这是内核级的全局互斥锁。每当创建/删除网卡、修改 IP 或路由时，内核都会加锁。
        *   *现象：* 当启动 1000+ 容器时，Docker 并非并行启动，而是因为等待 RTNL 锁而串行化。
        *   *推论：* 启动 100 万个节点意味着数千万次 Netlink 调用。单机内核会因为争抢这把锁而完全瘫痪，启动时间可能长达数周甚至死锁。
    3.  **PID 与文件句柄耗尽：** 100 万个容器意味着至少 100 万个进程（即便只是 pause 容器）。Linux 默认 PID 上限通常为 32768 或 4194304，文件句柄上限也受内存限制。单纯的 Docker 模式无法跨越此物理墙。

### 2.2 关于 "迁移至 Kubernetes" 的原始分析回顾 (Review of Original Analysis)

在项目演进讨论中，曾有一份关于迁移至 Kubernetes 的技术分析。为了确保架构延续性，我们将该分析的核心观点收录如下：

> **原始分析摘要 (Original Analysis Summary):**
>
> *   **可行性：** "Can you do it? Yes. Should you do it? Yes." 它是超越单机限制的唯一途径。
> *   **核心挑战：** 标准 K8s 网络（扁平化 IP）会破坏仿真器对拓扑、L2 网段和 BGP IP 的精确控制。
> *   **解决方案 (Multus CNI):**
>     *   **双网卡架构：** `eth0` 走 K8s 管理网（API/Metrics），`net1..N` 走 Multus 数据面（仿真流量）。
>     *   **CRD 定义链路：** 使用 `NetworkAttachmentDefinition` 定义虚拟网线。
> *   **跨机互联 (VXLAN):**
>     *   Docker Compose 只能用 Linux Bridge（单机）。
>     *   K8s 多机环境必须用 VXLAN Overlay 将 L2 网络延伸到物理机之外。
>     *   *效果：* 容器 A (节点 1) 发给 容器 B (节点 2) 的包被 UDP 封装，BIRD 守护进程对此无感知，认为是一根直连网线。
> *   **核心收益 (打破 RTNL 锁):**
>     *   "3000 容器 vs 1 内核锁" 问题被解决。
>     *   通过将仿真分散到 10 台物理机，拥有了 10 个独立的 Linux 内核，10 把 RTNL 锁。BGP 收敛速度因物理并行而大幅提升。

### 2.3 对原始分析的深度批判与场景纠偏 (Critical Re-evaluation)

上述原始分析在 **分布式多机集群** 场景下是 **完全正确且极具洞见** 的。然而，针对我们当前的 **"单机百万节点"** 目标，该分析存在致命的不适用性，必须进行纠偏：

1.  **RTNL 锁并未消失 (The Lock is Still There):**
    *   *原始分析假设：* 通过增加物理机来分摊 RTNL 锁压力。
    *   *单机现状：* 我们只有一台机器，一个内核。如果依然采用 Kubernetes 原生模式（1 Pod = 1 Node），我们将试图在一个内核上创建 100 万个 Network Namespaces。这不仅不能解决 RTNL 锁问题，反而因为 Kubelet 的 API 调用开销而雪上加霜。
    *   *结论：* **单机百万节点必须绕过 Host Kernel，不能使用内核级 Network Namespace。**

2.  **VXLAN 是纯粹的浪费 (VXLAN Overhead):**
    *   *原始分析假设：* 流量需要跨越物理网络。
    *   *单机现状：* 所有流量都在同一台机器的内存中流转。
    *   *问题：* 在单机内部使用 VXLAN = "用户态数据 -> 内核协议栈 -> VXLAN 封包 -> 内核环回 -> VXLAN 解包 -> 内核协议栈 -> 用户态"。这是极其昂贵的 CPU 上下文切换。
    *   *结论：* **单机内部必须使用共享内存 (Shared Memory) 或零拷贝技术，严禁使用 Overlay 协议。**

3.  **Pod 带来的额外重负 (The Cost of a Pod):**
    *   K8s Pod 不是免费的。每个 Pod 包含 pause 容器、cgroups 组、secret 挂载等。
    *   Kubelet 也就是个 Go 程序，它无法在一台机器上管理 100 万个 Pod 对象（心跳、状态同步会由 O(N) 变为灾难级）。

**总结：** 原始分析解决了“如何利用多台机器扩展规模”的问题，而我们现在要解决的是“如何在一台机器上榨干每一滴性能以达到数量级突破”的问题。这需要完全不同的架构。

---

## 3. 核心变革：聚合模型与用户态网络 (The Aggregation Model)

为了达成单机百万节点，我们必须解耦 **仿真逻辑单元 (Simulation Node)** 与 **基础设施执行单元 (Execution Pod)**。

### 3.1 概念：仿真区域 (Simulation Zone)
我们不再为每个路由器创建一个 Pod，而是创建一个 **仿真区域 Pod (Simulation Zone Pod)**。
*   **1 Pod = 1 AS (或多个 AS) = N 个仿真节点**。
*   **Infrastructure (Pod):** 申请 32GB 内存，8 核 CPU。
*   **Logic (Nodes):** 内部运行 5000 个轻量级路由器实例。

### 3.2 关键技术：用户态网络 (User-Space Networking)
这是打破 RTNL 锁的唯一解法。
*   **技术选型：** 使用 **VPP (Vector Packet Processing)** 或 **LwIP**。
*   **架构原理：**
    *   Host Kernel 只看到 **一个** 进程（Zone Runner）。
    *   该进程在 **用户态内存** 中维护 5000 张路由表、5000 个 ARP 表。
    *   **虚拟网线：** 变成进程内部的内存指针拷贝。
    *   **RTNL 锁：** 完全无关。我们在用户态可以并行修改几万个路由表，互不干扰。

---

## 4. 架构蓝图：Kubernetes Native Evolution

我们将废弃静态的 `Compiler`，构建一个动态的 **Seed Emulator Operator**。

### 4.1 自定义资源定义 (CRDs)
1.  **`Topology` (全局拓扑):** 用户视角的百万节点图。
2.  **`SimulationZone` (切片):** Operator 计算出的物理部署单元。
3.  **`VirtualLink` (虚拟链路):**
    *   *Intra-Zone:* 内存直连。
    *   *Inter-Zone (同机):* **Memif (Shared Memory Packet Interface)**。这是一种高性能、零拷贝的容器间通信接口，专门配合 VPP 使用。

### 4.2 控制器逻辑 (Controller Logic)
*   **Bin-packing 调度：** 将 100 万节点根据 CPU 预估消耗，“装箱”到 100-200 个 Simulation Zone Pod 中。
*   **动态拓扑变更：** 用户修改 CRD，Operator 仅通知受影响的 Zone Pod 更新内部路由表，无需重启容器。

---

## 5. Kubeflow 与 AI 深度集成 (Resource Rationality)

Kubeflow 将实现“资源理性”闭环：

1.  **流量预测与初始调度 (Prediction):**
    *   输入：拓扑结构。
    *   模型：GNN (图神经网络)。
    *   输出：预测哪些 AS 是核心枢纽，哪些是边缘。
    *   行动：Operator 将核心 AS 分配给 "High-Perf Zone" (独占 CPU)，边缘 AS 分配给 "Low-Power Zone" (高密度聚合)。

2.  **运行时重平衡 (Runtime Rebalancing):**
    *   Prometheus 监控到 Zone A 的 CPU 使用率超过 90% 导致丢包。
    *   Kubeflow Pipeline 触发 **Live Migration**。
    *   Operator 动态分裂 Zone A，将其中的一半 AS 迁移到新启动的 Zone B 中。

---

## 6. 实施路线图与阶段拆解 (Detailed Roadmap)

### 第一阶段：Operator 基础架构建设 (Phase 1: Foundation)
*   **1.1:** 定义 CRD (`Topology`, `EmulatedNode`, `EmulatedLink`)。
*   **1.2:** 构建 Operator 骨架，实现基本的 `Reconcile` 循环。
*   **1.3:** **验证性原型：** 暂时保留 Docker 容器模式，但由 Operator 管理，跑通 1000 节点。

### 第二阶段：聚合运行时研发 (Phase 2: Aggregation Runtime)
*   **2.1:** 开发 `seed-runtime-supervisor` (Python/Go)，在一个容器内启动多个 BIRD 进程。
*   **2.2:** 实现基于 `unshare -n` 的轻量级命名空间管理（绕过 K8s）。
*   **2.3:** Operator 升级支持 `AggregationPolicy`。

### 第三阶段：用户态网络高性能改造 (Phase 3: High-Performance Data Plane)
*   **3.1:** **VPP 集成：** 构建包含 VPP 的 Base Image。
*   **3.2:** **BIRD 适配：** 修改 BIRD 或开发中间件，使其能通过 VPP API 而不是 Kernel Netlink 注入路由。
*   **3.3:** **Memif CNI 开发：** 实现 Pod 间的高性能共享内存通信。

### 第四阶段：Kubeflow 与智能化 (Phase 4: AI Integration)
*   **4.1:** 流量数据采集 (基于 eBPF 或 VPP Telemetry)。
*   **4.2:** 训练负载预测模型。
*   **4.3:** 实现 Operator 的动态扩缩容 (Auto-scaling) 逻辑。

### 第五阶段：彻底解放 (Phase 5: Liberation)
*   **5.1:** 构建 WebGL 可视化前端，展示百万节点。
*   **5.2:** 社区化插件市场建设。
