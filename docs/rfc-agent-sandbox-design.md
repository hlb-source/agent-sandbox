# RFC: Agent Sandbox - 单例有状态工作负载管理框架

## 摘要

Agent Sandbox 是 Kubernetes 原生框架，用于管理单例、有状态、稳定身份的工作负载。提供核心 CRD（Sandbox）和扩展 CRDs（SandboxTemplate、SandboxClaim、SandboxWarmPool），简化AI Agent Runtime的创建过程，提升效率

---

## 动机

### 问题陈述

| 控制器 | 特性 | 局限性 |
|--------|------|--------|
| **Deployment** | 无状态、多副本、不稳定身份 | 无法满足稳定 hostname + 持久存储的单例 |
| **StatefulSet** | 有状态、有序编号、多副本 | 单例场景过于复杂，编号语义不匹配 |
| **Pod** | 单例，但无生命周期管理 | 缺少定时过期、暂停恢复机制 |

### 目标场景

1. **AI Agent Runtime**：执行 LLM 生成的代码，需要隔离环境 + 快速启动 + 稳定访问
2. **开发环境**：每个开发者独立、持久、网络可达的环境

### 现有方案缺陷

手动组合 Pod + Service + PVC + 生命周期脚本：
- 配置繁琐，易出错
- 无标准化 API
- 缺少预热机制，冷启动延迟高
- 缺少安全默认配置

---

## 目标

### 主要目标

1. 声明式 API 管理单例有状态 Pod
2. 稳定身份（固定 FQDN：Fully Qualified Domain Name，完全限定域名）
3. 持久存储支持
4. 生命周期管理（定时过期、暂停/恢复）
5. 预热池机制，消除冷启动延迟
6. 安全默认配置（Secure-by-default）

### 非目标

1. 支持多副本（replicas 限制为 0 或 1）
2. 替代 StatefulSet
3. 跨集群管理
4. 多容器编排

---

## 设计

### 架构概览

```
用户层:  SandboxClaim → SandboxTemplate → SandboxWarmPool
                     ↓
控制器层: SandboxClaim Controller ── adopt ──→ Sandbox
         SandboxWarmPool Controller ── 预热 ──→ Sandbox
         Sandbox Controller ──→ Pod + Service + PVC
         SandboxTemplate Controller ──→ NetworkPolicy
                     ↓
资源层:   Sandbox CR ──→ Pod + Headless Service + PVC
```

### 核心 CRD

#### Sandbox（核心资源）

管理单个 Pod + Service + PVC 的声明式资源。

**关键特性**：
| 特性 | 说明 |
|------|------|
| 稳定身份 | Pod 名称 = Sandbox.Name，FQDN（Fully Qualified Domain Name，完全限定域名）固定 |
| 持久存储 | `volumeClaimTemplates` 创建 PVC |
| 生命周期 | `shutdownTime` 定时过期，`shutdownPolicy` 删除/保留 |
| 暂停/恢复 | `replicas=0` 删除 Pod，`replicas=1` 恢复，PVC 保留 |

**生命周期管理详解**：

1. **定时过期机制**：
   - `shutdownTime`：指定过期时间点（RFC3339格式）
   - `shutdownPolicy`：过期后行为
     - `Delete`：Pod + Service + PVC + Sandbox CR 全部删除（一次性任务自动清理）
     - `Retain`：Pod + Service + PVC 删除，Sandbox CR 保留（状态标记Expired，审计追踪）

2. **暂停/恢复机制**：
   - `replicas=0`：Pod删除（释放计算资源），PVC + Service + Sandbox CR 保留（数据持久化）
   - `replicas=1`：Pod重建，PVC数据恢复，Service FQDN不变（网络身份稳定）

**对比原生资源**：
| 机制 | Deployment | StatefulSet | Pod | **Sandbox** |
|------|------------|-------------|-----|-------------|
| 定时过期 | ❌ 无 | ❌ 无 | ❌ 无（需Job/TTL） | ✅ **shutdownTime自动触发** |
| 暂停（replicas=0） | ✅ Pod删除 | ✅ Pod删除 | ❌ 需手动删除 | ✅ **Pod删除 + PVC保留** |
| 恢复（replicas=1） | ✅ Pod新建（无状态） | ✅ Pod重建（编号固定） | ❌ 需手动创建 | ✅ **Pod重建 + PVC数据恢复** |
| 数据持久化 | ❌ 无状态 | ✅ PVC保留 | ✅ 需手动配置 | ✅ **自动管理PVC** |
| 网络身份稳定 | ❌ 随机名称 | ✅ 编号固定 | ✅ 固定名称 | ✅ **固定名称 + FQDN** |

**Service 机制（Headless Service）**：

每个 Sandbox 自动创建一个 Headless Service（`ClusterIP: None`），提供稳定的网络身份。

| 类型 | ClusterIP | DNS解析 | 适用场景 |
|------|-----------|---------|----------|
| 普通 Service | 虚拟IP（如10.96.0.1） | 解析到ClusterIP | 多副本、负载均衡 |
| **Headless Service** | **None** | **直接解析到Pod IP** | **单例、稳定身份** |

**原理**：
```
Sandbox创建 → Controller创建Headless Service
    ↓
Service.Name = Sandbox.Name（固定）
    ↓
CoreDNS注册：sandbox-name.namespace.svc.cluster.local → Pod IP
    ↓
客户端查询DNS → 直接获取Pod IP → 连接Pod
```

**FQDN结构**：
```
my-sandbox.default.svc.cluster.local
│         │      │      │      │
│         │      │      │      └── 集群域名（默认cluster.local）
│         │      │      └── Service类型标识
│         │      └── Pod/Service名称
│         └── Namespace
└── Sandbox名称（Pod名称=Sandbox名称）
```

**好处**：
| 好处 | 说明 |
|------|------|
| **稳定DNS** | Pod重建后名称不变，DNS记录不变，客户端无需修改 |
| **无需负载均衡** | 单例Pod无需ClusterIP转发，直接连接 |
| **节省资源** | 无虚拟IP、无kube-proxy规则，仅DNS记录 |
| **StatefulSet体验** | 类似StatefulSet的稳定身份，但更简单（无编号） |

**对比原生资源**：
| 资源 | Pod名称 | Service类型 | DNS稳定性 |
|------|---------|-------------|-----------|
| Deployment | 随机（deploy-xxx-yyy） | 普通 Service | 不稳定（Pod重建后名称变化） |
| StatefulSet | 编号（sts-0, sts-1） | Headless | 稳定（编号固定） |
| **Sandbox** | **固定（sandbox-name）** | **Headless** | **稳定（名称固定）** |

**API 示例**：
```yaml
kind: Sandbox
spec:
  podTemplate: {spec: PodSpec, metadata: {labels, annotations}}
  volumeClaimTemplates: []
  replicas: 0 | 1
  shutdownTime: Time
  shutdownPolicy: Delete | Retain
status:
  serviceFQDN: string
  conditions: []
```

#### SandboxTemplate（配置模板）

可复用的 Pod 配置 + 网络策略模板。

**关键特性**：
| 特性 | 说明 |
|------|------|
| 模板复用 | 统一 Pod 配置，多处引用 |
| 自动 NetworkPolicy | 每个 Template 对应一个共享 NetworkPolicy |
| Secure-by-default | 默认阻断私有网段、Metadata Server |

**API 示例**：
```yaml
kind: SandboxTemplate
spec:
  podTemplate: PodTemplate
  networkPolicy: NetworkPolicySpec
  networkPolicyManagement: Managed | Unmanaged
```

#### SandboxWarmPool（预热池）

维护预热的 Sandbox 池，消除冷启动延迟。

**关键特性**：
| 特性 | 说明 |
|------|------|
| 预热创建 | 预先创建 N 个 Ready 的 Sandbox |
| 快速分配 | Claim 通过 adopt 1-3秒获取（Pod已Ready） |
| 自动伸缩 | 支持 HPA 自动调整池大小 |
| 健康检查 | 超过 5 分钟不 Ready 自动删除重建 |

**API 示例**：
```yaml
kind: SandboxWarmPool
spec:
  replicas: int32
  sandboxTemplateRef: {name: string}
status:
  readyReplicas: int32
```

#### SandboxClaim（用户申请）

声明式申请 Sandbox 的用户入口。

**关键特性**：
| 特性 | 说明 |
|------|------|
| 模板引用 | 引用 SandboxTemplate |
| 自动分配 | 优先 adopt WarmPool，无可用则冷启动 |
| 生命周期管理 | 控制 Sandbox 的过期、删除 |

**API 示例**：
```yaml
kind: SandboxClaim
spec:
  sandboxTemplateRef: {name: string}
  lifecycle: {shutdownTime, shutdownPolicy}
status:
  sandboxStatus: {name: string}
```

**SandboxClaim 核心价值**：

相比直接创建 Sandbox：
1. **模板抽象**：用户只需引用模板名，无需关心 Pod 配置细节
2. **预热池加速**：可从 WarmPool 直接采纳已就绪的 Sandbox，1-3秒获取（无需等待Pod启动）
3. **统一管理**：网络策略、安全配置在 Template 层统一控制
4. **声明式语义**："我需要一个沙箱"而非"创建一个Pod+Service+PVC"

---

### 资源关联

```
SandboxTemplate ←─ SandboxWarmPool ── creates ──→ Sandbox (pool labels)
                ←─ SandboxClaim ── adopts ──→ Sandbox (owner: Claim)
                                            ↓
                                       Pod + Service + PVC
```

**关键 Label**：
| Label | 用途 |
|-------|------|
| `warm-pool-sandbox` | 标识 Sandbox 属于 WarmPool |
| `sandbox-template-ref-hash` | 匹配 Template |
| `sandbox-name-hash` | Pod/Service 关联 Sandbox |

---

### 安全设计

**Secure-by-default**：
| 安全措施 | 实现 |
|----------|------|
| 禁用 SA Token | `automountServiceAccountToken: false` |
| DNS 隔离 | 公网 DNS (8.8.8.8, 1.1.1.1) |
| NetworkPolicy | Default-Deny |
| 阻断 Metadata | Egress 阻断 169.254.0.0/16 |
| 阻断私有网段 | Egress 阻断 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 |

---

### 性能对比

| 指标 | 冷启动 | 热启动（WarmPool） |
|------|--------|-------------------|
| 启动延迟 | ~5-15秒（小镜像）<br>~30-90秒（大镜像） | ~1-3秒 |
| 主要耗时 | Pod调度 + 镜像拉取 + 容器启动 | Controller同步 + API更新 |
| 用户感知 | 阖塞等待 | 立即可用 |

**数据来源**：
- Metrics Buckets 定义（`internal/metrics/metrics.go`）：热启动预期 50ms-5000ms
- Load Test 结果（`dev/load-test/README.md`）：冷启动约 5-10秒（轻量镜像）
- 实际生产环境：大镜像（如 AI Runtime）拉取可能需要 30-60秒

---

## 使用示例

### 直接创建 Sandbox

```yaml
apiVersion: agents.x-k8s.io/v1alpha1
kind: Sandbox
metadata:
  name: my-sandbox
spec:
  podTemplate:
    spec:
      containers:
      - name: my-container
        image: python:3.11
  volumeClaimTemplates:
  - name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
  shutdownTime: "2025-12-31T00:00:00Z"
  shutdownPolicy: Retain
```

### 使用扩展 CRDs

```yaml
# 1. 定义模板
apiVersion: extensions.agents.x-k8s.io/v1alpha1
kind: SandboxTemplate
metadata:
  name: ai-agent-template
spec:
  podTemplate:
    spec:
      runtimeClassName: gvisor
      securityContext: {runAsNonRoot: true}
      containers:
      - name: agent
        image: python:3.11-slim

---

# 2. 创建预热池
apiVersion: extensions.agents.x-k8s.io/v1alpha1
kind: SandboxWarmPool
metadata:
  name: ai-agent-pool
spec:
  replicas: 5
  sandboxTemplateRef:
    name: ai-agent-template

---

# 3. 用户申请
apiVersion: extensions.agents.x-k8s.io/v1alpha1
kind: SandboxClaim
metadata:
  name: user-alice-agent
spec:
  sandboxTemplateRef:
    name: ai-agent-template
  lifecycle:
    shutdownTime: "2025-12-31T23:59:59Z"
    shutdownPolicy: Retain
```

---

## 未来规划

1. Beta/GA 版本：稳定 API
2. PVC 暂停恢复：Scale-down 保留 PVC
3. SDK 扩展：Python SDK 支持 read/write/run_code
4. 更多 Runtime：QEMU、Firecracker
5. Agent 框架集成：CrewAI、Ray RLlib

---

## 参考

1. [Kubernetes SIG Apps](https://github.com/kubernetes/community/tree/master/sig-apps)
2. [StatefulSet 设计](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
3. [NetworkPolicy 规范](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
4. [gVisor Runtime](https://gvisor.dev/)