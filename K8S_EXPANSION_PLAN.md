# SEED Emulator Kubernetes Native: 高性能智能仿真架构演进规划

## 1. 愿景与核心目标 (Vision & Core Objectives)

本文档旨在规划 **SEED Emulator** 的下一代架构演进，使其从一个基于静态编译的仿真工具，进化为一个 **Kubernetes Native 的智能仿真平台**。

我们的核心目标并非局限于某个具体的数字（如“百万节点”），而是构建一个 **高度泛化、极致资源理性 (Resource Rationality)、且具备 AI 驱动能力的通用仿真底座**。该底座需同时满足以下场景：
1.  **极致单机密度：** 在单台服务器上榨干硬件性能，支撑超大规模拓扑（如国家级互联网仿真）。
2.  **弹性多机扩展：** 无缝横向扩展至大规模 Kubernetes 集群，利用云原生优势。
3.  **智能运行体验：** 通过 AI 介入，实现仿真保真度与资源消耗的动态平衡，提供“即插即用、按需计算”的优越体验。

---

## 2. 现状与挑战：从“能用”到“好用”的跨越

### 2.1 现有架构的局限性 (Limitations)
当前的 Docker-Compose/Static-K8s 模式本质上是 **"基础设施即全量 (Infrastructure as All)"**。一旦编译完成，无论节点是否在通信，它都必须占用完整的操作系统资源（进程、命名空间、文件句柄）。

这种模式面临三大根本性挑战：
1.  **资源僵化 (Resource Rigidity)：** 即使 90% 的节点处于空闲状态，它们依然占用了 90% 的内核资源（RTNL 锁、内存）。
2.  **扩展瓶颈 (Scalability Wall)：** 依赖 Host Kernel 的网络栈导致单机上限被锁死（通常在数千节点），无法通过简单堆砌硬件突破内核锁竞争。
3.  **运维黑盒 (Operational Blackbox)：** 仿真启动后，缺乏动态调整能力。无法在不重启的情况下修改链路属性或扩缩容。

### 2.2 原始 K8s 迁移方案的再思考
之前关于 "K8s + Multus + VXLAN" 的讨论虽然指出了方向，但视角主要集中在“如何连通”。未来的架构需要解决更深层的问题：**“如何高效地计算”**。仅仅把容器搬到 K8s 上并不能解决资源浪费问题，我们需要更智能的**聚合**与**调度**。

---

## 3. 核心架构变革：资源理性与智能聚合 (Architecture Evolution)

为了实现从“运行容器”到“运行仿真”的范式转变，我们提出 **Kubernetes Native 聚合架构**。

### 3.1 动态聚合模型 (Dynamic Aggregation Model)
打破 "1 Node = 1 Pod" 的僵化映射，引入 **"仿真区域 (Simulation Zone)"** 概念作为基本调度单元。

*   **智能切片 (Intelligent Slicing)：**
    *   Operator 不再盲目生成 Pod，而是根据节点功能（核心路由 vs 边缘主机）和即时负载，将逻辑节点打包进不同的 Zone。
    *   *高保真区 (High-Fidelity Zone):* 对于需要运行复杂安全工具（如 IDS/IPS）的节点，仍分配独立 Pod 甚至独占 CPU。
    *   *高密度区 (High-Density Zone):* 对于数万个仅做背景流量的僵尸网络节点，聚合到单个 Pod 内，共享用户态协议栈。
*   **自适应保真度 (Adaptive Fidelity)：**
    *   **Level 1 (Stateless):** 仅模拟连通性，无协议栈（适合大规模背景节点）。
    *   **Level 2 (User-Space):** 使用 VPP/LwIP 用户态协议栈，高性能，无内核开销。
    *   **Level 3 (Full-Kernel):** 完整的 Linux Netns，支持所有标准工具（tcpdump, iptables）。
    *   *关键能力：* 系统可根据用户关注点，**动态**将节点在不同保真度级别间热迁移。

### 3.2 泛化网络互联 (Generalized Networking)
不再依赖单一的 VXLAN 或 Bridge，而是建立分层互联体系：
1.  **Tier 0 (进程内):** 聚合在同一 Pod 内的节点通信，直接走内存指针拷贝 (Zero-Copy)。
2.  **Tier 1 (同机跨 Pod):** 利用 **共享内存 (Shared Memory / Memif)** 技术，绕过内核协议栈，实现纳秒级通信。
3.  **Tier 2 (跨机集群):** 自动配置 VXLAN/Geneve 或 SR-v6 隧道，由 K8s CNI 插件透明管理。

---

## 4. Kubernetes Native Operator 体系

我们将构建 **Seed Emulator Operator**，它是整个仿真系统的“大脑”。

### 4.1 声明式 API (CRDs)
*   **`SimulationGraph`:** 定义宏观拓扑、链路属性和业务意图（如“模拟一次 BGP 劫持攻击”）。
*   **`ResourceProfile`:** 定义仿真对资源的需求等级（如“低延迟优先”或“最大规模优先”）。
*   **`RuntimePolicy`:** 定义运行时行为（如“当 CPU > 80% 时自动降级非核心区域保真度”）。

### 4.2 智能控制器 (Intelligent Controller)
Operator 不仅负责部署，更负责**生命周期管理**：
*   **自动装箱 (Auto-Binpacking):** 结合物理机拓扑感知 (Topology Awareness)，将频繁通信的 Zone 调度到同一物理机甚至同一 NUMA 节点上。
*   **无感自愈 (Seamless Healing):** 当某物理节点故障，Operator 自动将其上的 Simulation Zone 迁移至其他节点，并重新建立隧道连接。

---

## 5. AI 驱动的智能运行体验 (AI-Driven Experience)

引入 **Kubeflow** 作为仿真平台的“副驾驶 (Co-Pilot)”，将仿真提升到“数字孪生”的高度。

### 5.1 仿真即代码 (Simulation as Code) 与 MLOps
*   **实验流水线:** 使用 Kubeflow Pipelines 编排“生成 -> 部署 -> 攻击 -> 数据清洗 -> 模型训练”的全流程。
*   **参数超参优化 (Hyperparameter Tuning):** 利用 Katib 自动搜索最优的仿真参数（如 BIRD 路由器的 Keepalive 时间、队列长度），以在有限硬件上获得最稳定的仿真效果。

### 5.2 实时智能调度 (Real-time Intelligent Scheduling)
*   **流量预测模型:** 基于历史数据或 GNN 模型，预测未来 N 分钟的网络热点。
*   **抢占式优化:** 在热点形成前，Operator 提前扩容相关 Zone 的资源限额，或将流量牵引至轻载路径。
*   **异常检测:** 实时分析全网流量，自动识别非预期的路由黑洞或环路（这些可能是配置错误，也可能是仿真本身发现的协议漏洞）。

---

## 6. 演进路线图 (Evolution Roadmap)

### Phase 1: 基础架构云原生化 (Cloud-Native Foundation)
*   构建 Operator 骨架，定义 `Topology` CRD。
*   实现 Operator 对现有 Docker/K8s 资源的纳管。
*   **目标：** 用户提交 YAML，集群自动拉起仿真，无需运行 Python 编译脚本。

### Phase 2: 聚合运行时与混合调度 (Aggregation & Hybrid Scheduling)
*   开发支持多进程/多租户的 `SeedRuntime` 容器底座。
*   实现“高保真”与“高密度”两种 Pod 类型的混合部署。
*   **目标：** 单机支持 10万+ 节点，且保持核心业务的高真实度。

### Phase 3: 用户态高性能网络 (High-Performance Data Plane)
*   引入 VPP/LwIP，实现 Pod 间共享内存通信 (Memif CNI)。
*   彻底解除 Host Kernel RTNL 锁限制。
*   **目标：** 网络吞吐量提升 10-100 倍，支持超大规模单机仿真。

### Phase 4: 智能化闭环 (Intelligence Loop)
*   集成 Kubeflow，建立 Traffic Generator 与 Monitoring 的闭环。
*   实现基于 AI 的动态资源调整与保真度切换。
*   **目标：** 仿真系统具备“自我优化”能力，提供极致的运行体验。

### Phase 5: 生态化与服务化 (Ecosystem)
*   提供 SaaS 化服务接口，支持多租户共享大集群。
*   建立仿真模型市场 (Marketplace)。
*   **目标：** 成为网络安全研究与教学的通用云原生底座。
