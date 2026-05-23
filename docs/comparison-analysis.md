# AI Sandbox 方案对比分析

## 概述

本文档对比分析三个主流 AI Agent 沙箱方案：Agent Sandbox、E2B Infra、CubeSandbox。

---

## 核心定位对比

| 项目 | 定位 | 开发者 | License | Stars |
|------|------|--------|---------|-------|
| **Agent Sandbox** | K8s 原生框架，自建部署 | Kubernetes SIG Apps | Apache 2.0 | ~100 |
| **E2B Infra** | 商业托管服务基础设施 | e2b.dev | Apache 2.0 | 1.1k |
| **CubeSandbox** | 高性能开源沙箱服务 | Tencent Cloud | Apache 2.0 | 5.8k |

---

## 核心差异速览

| 维度 | Agent Sandbox | E2B Infra | CubeSandbox |
|------|---------------|-----------|-------------|
| **冷启动时间** | 分钟级 | ~100ms | **<60ms** |
| **内存开销** | 50-100MB | ~128MB | **<5MB** |
| **单节点密度** | ~100-500 | ~1000 | **数千** |
| **数据位置** | 本地化 | 云端托管 | 本地化 |
| **K8s 集成** | ✅ 原生 | ❌ | ❌ |
| **E2B SDK兼容** | ❌ 需适配 | ✅ 原生 | ✅ **零成本迁移** |
| **运维复杂度** | 中等（需K8s） | 低（托管） | 低（一键部署） |

---

## 技术架构对比

### 底层技术栈

| 维度 | Agent Sandbox | E2B Infra | CubeSandbox |
|------|---------------|-----------|-------------|
| **虚拟化技术** | gVisor/Kata | Firecracker microVM | KVM + RustVMM |
| **调度系统** | Kubernetes Scheduler | Nomad + Consul | 自研 CubeMaster |
| **网络隔离** | NetworkPolicy (iptables) | 自研网络层 | CubeVS (eBPF) |
| **运行时** | containerd | Firecracker | containerd-shim v2 |
| **主要语言** | Go | Go + Terraform | Rust + Go |

#### 网络隔离性能优势

| 维度 | Agent Sandbox | E2B Infra | CubeSandbox |
|------|---------------|-----------|-------------|
| **单包延迟** | 10-50μs | 5-20μs | **<1μs** |
| **查表复杂度** | O(n) 线性匹配 | 自研算法 | **O(1) 哈希查表** |
| **上下文切换** | 需要（用户态） | 需要（用户态） | **不需要（内核态）** |
| **策略更新延迟** | 秒级（iptables重载） | 毫秒级 | **微秒级（BPF map）** |
| **1000规则延迟** | ~10ms | ~1ms | **<100μs** |
| **10000规则延迟** | ~100ms（性能下降） | ~10ms | **<100μs（稳定）** |

| 项目 | 性能优势 | 劣势 |
|------|---------|------|
| Agent Sandbox | K8s原生集成，标准化管理 | iptables线性匹配，大规模规则性能下降明显 |
| E2B Infra | 自研网络层优化较好 | 用户态实现，仍需上下文切换 |
| CubeSandbox | eBPF内核态+O(1)查表，无上下文切换，万级规则仍保持<100μs | 需要Linux内核支持 |

### 调度组件对比

| 组件 | Agent Sandbox | E2B Infra | CubeSandbox |
|------|---------------|-----------|-------------|
| **API 网关** | K8s API Server（Go） | E2B Gateway | **CubeAPI（Rust）** |
| **调度器** | K8s Scheduler | Nomad Server | **CubeMaster** |
| **数据存储** | etcd（Raft共识） | Consul KV | 内存（无共识） |
| **节点代理** | kubelet | Nomad Client | **Cubelet** |
| **运行时** | containerd → gVisor/Kata | Firecracker | **CubeHypervisor** |
| **网络** | CNI（用户态） | 自研 | **CubeVS（eBPF内核态）** |
| **预热池** | WarmPool Controller | Orchestrator | **资源池预分配** |

### 调度延迟分析

| 环节 | Agent Sandbox | E2B Infra | CubeSandbox |
|------|---------------|-----------|-------------|
| API 入口 | ~100ms（etcd Raft） | ~10ms | **<1ms（Rust）** |
| 调度决策 | ~1-5s（Filter+Score+Bind） | ~50ms | **<10ms（资源池匹配）** |
| 运行时启动 | ~200ms | ~50ms | **<50ms（快照克隆）** |
| **总计** | 分钟级 | ~100ms | **<60ms** |

---

## 性能对比

### 启动性能

| 维度 | Agent Sandbox | E2B Infra | CubeSandbox |
|------|---------------|-----------|-------------|
| **冷启动** | 分钟级（K8s调度） | ~100ms | **<60ms** |
| **热启动** | 亚秒级（预热池adopt） | 预热池支持 | 预热池+快照克隆 |
| **50并发P95** | 受API Server限制 | ~150ms | **90ms** |
| **50并发P99** | - | - | **137ms** |

### 资源开销

| 维度 | Agent Sandbox | E2B Infra | CubeSandbox |
|------|---------------|-----------|-------------|
| **内存开销** | 50-100MB | ~128MB | **<5MB** |
| **单节点密度** | ~100-500 Pod | ~1000 VM | **数千实例** |
| **镜像/rootfs** | 用户自定义 | ~5MB极简rootfs | 极简Guest OS |

内存开销指每个沙箱实例运行时的额外内存占用：

| 项目 | 内存开销 | 说明 |
|------|---------|------|
| **Agent Sandbox** | 50-100MB | gVisor Sentry进程或Kata代理进程开销 |
| **E2B Infra** | ~128MB | Firecracker microVM的VMM进程+Guest OS |
| **CubeSandbox** | <5MB | 极简运行时，无独立Guest OS进程 |

关键区别：Agent Sandbox使用gVisor需要Sentry进程拦截syscall；E2B每个microVM需要独立的VMM进程和Guest内核；CubeSandbox通过共享内核+轻量级隔离实现最小开销。

### 性能优势原因

| 优势 | CubeSandbox/E2B 实现 |
|------|----------------------|
| **绕过K8s控制平面** | 无API Server/etcd/Scheduler开销 |
| **预分配资源池** | 避免动态调度决策 |
| **快照克隆** | 跳过完整VM启动流程 |
| **极简镜像** | 无镜像拉取延迟 |

---

## 安全隔离级别

| 维度 | Agent Sandbox | E2B Infra | CubeSandbox |
|------|---------------|-----------|-------------|
| **隔离技术** | gVisor/Kata | Firecracker microVM | KVM |
| **隔离级别** | 内核级（gVisor拦截syscall） | 硬件级（独立VM） | **硬件级+eBPF网络** |
| **内核共享** | gVisor:不共享；Kata:独立 | 独立Guest OS | **独立Guest OS** |
| **逃逸风险** | 低 | 极低 | **极低** |

---

## 部署模式对比

| 模式 | Agent Sandbox | E2B Infra | CubeSandbox |
|------|---------------|-----------|-------------|
| **托管服务** | 无 | ✅ e2b.dev | 无（腾讯云可托管） |
| **自建部署** | ✅ 需K8s集群 | ✅ Terraform(GCP/AWS) | ✅ 一键部署 |
| **云平台** | GKE/EKS/自建 | GCP(主)/AWS(Beta) | 普通云VM(PVM) |
| **硬件要求** | 标准K8s节点 | 裸金属/KVM | x86_64 Linux+KVM |
| **部署复杂度** | 中等 | 低（托管）/高（自建） | **低** |

---

## 结论

### 定位总结

| 方案 | 定位 | 最佳场景 |
|------|------|----------|
| **Agent Sandbox** | K8s原生运维框架 | 企业K8s用户、标准化管理 |
| **E2B** | 托管沙箱服务 | 快速验证、不想运维 |
| **CubeSandbox** | 极致性能自建方案 | RL训练、高性能、E2B降本 |

### 核心差异一句话

- **Agent Sandbox**：运维标准化，K8s生态融合，但冷启动慢
- **E2B**：托管服务，无需运维，但数据在云端、成本高
- **CubeSandbox**：性能最优，E2B兼容，适合极致性能+自建部署

