# AI Agent 沙箱产品设计文档（PRD）

**文档版本**: v1.0
**创建日期**: 2026-03-05
**文档状态**: 草稿
**负责团队**: AI Agent 平台组

---

## 目录

1. [产品背景与目标](#1-产品背景与目标)
2. [用户角色与核心场景](#2-用户角色与核心场景)
3. [产品全景架构](#3-产品全景架构)
4. [核心概念定义](#4-核心概念定义)
5. [功能模块详细设计](#5-功能模块详细设计)
   - 5.1 [沙箱生命周期管理](#51-沙箱生命周期管理)
   - 5.2 [IDE（GUI）形态](#52-idegui形态)
   - 5.3 [CLI 形态](#53-cli-形态)
   - 5.4 [Web UI 形态](#54-web-ui-形态)
   - 5.5 [工具调用与 MCP 集成](#55-工具调用与-mcp-集成)
   - 5.6 [安全与隔离机制](#56-安全与隔离机制)
   - 5.7 [资源配额与计费](#57-资源配额与计费)
   - 5.8 [可观测性与审计](#58-可观测性与审计)
6. [API 设计规范](#6-api-设计规范)
7. [数据模型设计](#7-数据模型设计)
8. [非功能需求](#8-非功能需求)
9. [成功指标（OKR / KPI）](#9-成功指标okr--kpi)
10. [竞品分析](#10-竞品分析)
11. [发布路线图（Roadmap）](#11-发布路线图roadmap)
12. [风险与依赖](#12-风险与依赖)
13. [附录：术语表](#13-附录术语表)

---

## 1. 产品背景与目标

### 1.1 背景

随着大型语言模型（LLM）的能力边界不断扩展，AI Agent 正在从"对话助手"演进为"自主行动者"：它能够调用工具、执行代码、操作文件系统、访问外部 API，甚至在多 Agent 协同场景下完成复杂的软件工程任务。

然而，**Agent 的自主行动能力是一把双刃剑**：

| 能力 | 潜在风险 |
|------|----------|
| 执行任意代码 | 恶意代码、无限循环、资源耗尽 |
| 读写文件系统 | 敏感数据泄露、数据损坏 |
| 调用外部网络 | 数据外传、供应链攻击 |
| 运行系统命令 | 权限提升、宿主机破坏 |

**AI Agent 沙箱**正是解决上述风险的核心基础设施，它为 Agent 的每一次"行动"提供一个**安全、隔离、可审计、可重现**的执行环境。

### 1.2 产品愿景

> **让每一个 AI Agent 的行动都在可控、可信、可追溯的环境中发生。**

### 1.3 产品目标

**短期目标（0-6 月）**：
- 建立沙箱核心基础设施，支持代码执行、文件操作、网络访问三大核心能力
- 提供 CLI 和 API 两种接入方式，覆盖开发者自集成场景
- 完成安全基线：进程隔离、资源限制、审计日志

**中期目标（6-12 月）**：
- 推出 Web UI 控制台，支持可视化创建、管理、监控沙箱
- 发布 IDE 插件（VS Code / JetBrains），实现 Agent 在本地 IDE 内的沙箱化执行
- 支持多 Agent 协同沙箱，构建 Agent 间通信隔离机制

**长期目标（12-24 月）**：
- 成为企业级 AI Agent 平台的标准沙箱基础设施
- 支持自定义沙箱模板市场
- 提供合规报告生成，满足金融、医疗等强监管行业需求

### 1.4 成功定义

- 单沙箱冷启动 P99 < 3s
- 沙箱安全逃逸率 = 0（Season Review 通过 CVE 红队测试）
- 开发者接入 MVP 所需时间 < 30 分钟
- 月活沙箱实例 > 10 万（上线 12 个月内）

---

## 2. 用户角色与核心场景

### 2.1 用户角色矩阵

| 角色 | 描述 | 核心诉求 | 主要使用形态 |
|------|------|----------|-------------|
| **Agent 开发者** | 构建 AI Agent 应用的工程师 | 快速集成、调试 Agent 行为 | CLI、IDE、API |
| **平台运维工程师** | 管理 Agent 基础设施的 SRE | 资源监控、配额管理、故障排查 | Web UI、API |
| **AI 产品经理** | 设计 Agent 产品流程 | 可视化 Agent 行为回放、测试用例管理 | Web UI |
| **安全合规专员** | 审计 Agent 操作合规性 | 完整操作日志、敏感数据检测报告 | Web UI |
| **企业管理员** | 管理组织级沙箱资源 | 成本控制、权限管理、策略配置 | Web UI |
| **终端用户** | 使用内嵌 Agent 能力的业务用户 | 无感知的安全保障 | 透明（无直接交互） |

### 2.2 核心用户故事

#### 场景 A：Agent 开发者 —— 本地调试代码生成 Agent

```
作为一名 Agent 开发者，
我希望在本地 IDE 中运行 Claude Code Agent，
当 Agent 尝试执行 bash 命令时，能在一个隔离的沙箱容器中运行，
而不是直接操作我的本机环境，
以便我可以放心地让 Agent 尝试各种危险操作而不担心破坏本地环境。
```

**关键路径**：
1. 开发者在 VS Code 中安装 Agent Sandbox 插件
2. 配置沙箱镜像（选择 ubuntu-24.04 + python3.12 模板）
3. 启动 Agent，Agent 发出 `bash("rm -rf /tmp/*")` 工具调用
4. 插件拦截调用，将其路由到沙箱容器执行
5. 沙箱内执行完成，结果返回 Agent
6. 开发者在 IDE 侧边栏实时查看沙箱内文件系统状态

---

#### 场景 B：平台运维 —— 批量管理生产环境沙箱

```
作为平台运维工程师，
我需要在 Web 控制台监控当前运行中的所有沙箱实例，
查看各沙箱的 CPU/内存使用率，
对异常沙箱执行强制终止，
并导出过去 30 天的资源消耗报告。
```

---

#### 场景 C：安全合规专员 —— 审计 Agent 操作

```
作为安全合规专员，
我需要查阅某次 Agent 任务执行的完整操作记录，
包括所有文件读写、网络请求、命令执行，
并对可疑操作生成合规告警报告，
以满足等保三级的审计要求。
```

---

#### 场景 D：企业管理员 —— 配置组织沙箱策略

```
作为企业管理员，
我需要为不同团队配置差异化的沙箱策略：
- 生产环境 Agent：禁止出站网络、只读文件系统
- 测试环境 Agent：允许访问内网，限速 10Mbps
- 研发环境 Agent：完全开放（仅限 VPN 内）
```

---

## 3. 产品全景架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        用户接入层（Client Layer）                      │
│                                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐   │
│  │  IDE 插件     │  │  CLI 工具    │  │      Web UI 控制台        │   │
│  │ (VS Code /   │  │ (sandbox-    │  │  (管理 / 监控 / 审计)     │   │
│  │  JetBrains)  │  │  ctl)        │  │                          │   │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────────┘   │
└─────────┼─────────────────┼──────────────────────┼───────────────────┘
          │                 │                      │
          └─────────────────┼──────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      API 网关层（Gateway Layer）                       │
│              RESTful API + WebSocket（实时日志/事件流）                │
│                   认证（API Key / OAuth2 / OIDC）                     │
└──────────────────────────────┬──────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     沙箱控制面（Control Plane）                        │
│                                                                       │
│  ┌────────────────┐  ┌──────────────┐  ┌──────────────────────┐     │
│  │ 沙箱生命周期    │  │  调度器      │  │  策略引擎             │     │
│  │ 管理器         │  │ (Scheduler)  │  │ (Policy Engine)      │     │
│  │ (Lifecycle Mgr)│  │              │  │                      │     │
│  └────────────────┘  └──────────────┘  └──────────────────────┘     │
│                                                                       │
│  ┌────────────────┐  ┌──────────────┐  ┌──────────────────────┐     │
│  │ 镜像仓库管理    │  │  配额管理器  │  │  审计日志服务          │     │
│  │ (Registry Mgr) │  │ (Quota Mgr)  │  │ (Audit Service)      │     │
│  └────────────────┘  └──────────────┘  └──────────────────────┘     │
└──────────────────────────────┬──────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     沙箱数据面（Data Plane）                           │
│                                                                       │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    沙箱实例集群                                │   │
│  │                                                              │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │   │
│  │  │  Sandbox #1  │  │  Sandbox #2  │  │  Sandbox #N  │  ...   │   │
│  │  │             │  │             │  │             │         │   │
│  │  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │         │   │
│  │  │ │ gVisor  │ │  │ │ gVisor  │ │  │ │ gVisor  │ │         │   │
│  │  │ │ (runsc) │ │  │ │ (runsc) │ │  │ │ (runsc) │ │         │   │
│  │  │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │         │   │
│  │  │  Network NS │  │  Network NS │  │  Network NS │         │   │
│  │  │  PID NS     │  │  PID NS     │  │  PID NS     │         │   │
│  │  │  Mount NS   │  │  Mount NS   │  │  Mount NS   │         │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘         │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐   │
│  │ 网络代理层    │  │ 文件系统层   │  │   Sidecar 监控代理        │   │
│  │ (Egress Proxy)│  │ (OverlayFS) │  │   (ebpf-based)           │   │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. 核心概念定义

### 4.1 沙箱（Sandbox）

沙箱是一个**轻量级、隔离的执行环境**，具有以下特征：

| 属性 | 描述 |
|------|------|
| **隔离性** | 进程、网络、文件系统三重命名空间隔离 |
| **短生命周期** | 默认 TTL 30 分钟，任务完成自动回收 |
| **不可变基础层** | 基础镜像只读，写入通过 OverlayFS 隔离 |
| **可快照** | 支持在任意时刻对文件系统状态创建快照 |
| **可重现** | 给定相同镜像 + 相同输入，行为完全确定 |

### 4.2 沙箱模板（Sandbox Template）

预配置的沙箱基础镜像 + 初始化脚本 + 策略配置的组合：

```yaml
template:
  id: python-data-science-v2
  display_name: "Python 数据科学环境"
  base_image: ubuntu:24.04
  pre_installed:
    - python3.12
    - pip
    - numpy
    - pandas
    - matplotlib
  policy:
    network: restricted        # 仅允许访问 PyPI 镜像源
    filesystem: read_write     # 允许在 /workspace 下读写
    cpu_limit: 2               # 2 vCPU
    memory_limit: 4Gi
    timeout: 1800              # 30分钟
```

### 4.3 工具调用拦截（Tool Call Interception）

Agent 发出工具调用（如 `bash`, `write_file`, `http_request`）时，沙箱 SDK 拦截调用，将其路由至沙箱内执行，并将结果返回给 Agent，**对 Agent 透明**。

### 4.4 沙箱会话（Sandbox Session）

一次 Agent 任务对应一个沙箱会话，会话内的沙箱实例可以被复用，保持状态连续性（如安装的依赖包、写入的文件）。

---

## 5. 功能模块详细设计

### 5.1 沙箱生命周期管理

#### 5.1.1 状态机

```
                  ┌──────────┐
                  │ PENDING  │  ← 调度中，等待资源
                  └────┬─────┘
                       │ 资源就绪
                       ▼
                  ┌──────────┐
                  │ STARTING │  ← 镜像拉取 + 容器启动
                  └────┬─────┘
                       │ 健康检查通过
                       ▼
              ┌────────────────┐
     ┌───────►│    RUNNING     │◄───────┐
     │        └────┬───────────┘        │
     │             │                    │
     │    工具调用  │         暂停恢复   │
     │             ▼                    │
     │        ┌──────────┐              │
     │        │  BUSY    │──────────────┘
     │        └──────────┘
     │
     │ 超时 / 主动停止
     ▼
┌──────────┐          ┌──────────┐
│ STOPPING │─────────►│ STOPPED  │
└──────────┘          └──────────┘
                           │
                    保留期结束（默认 1h）
                           ▼
                      ┌──────────┐
                      │ DELETED  │
                      └──────────┘
```

#### 5.1.2 生命周期操作 API

| 操作 | 方法 | 路径 | 描述 |
|------|------|------|------|
| 创建沙箱 | POST | `/v1/sandboxes` | 基于模板创建沙箱实例 |
| 查询沙箱 | GET | `/v1/sandboxes/{id}` | 获取沙箱状态和元信息 |
| 列出沙箱 | GET | `/v1/sandboxes` | 分页列出当前用户沙箱 |
| 执行命令 | POST | `/v1/sandboxes/{id}/exec` | 在沙箱内执行 shell 命令 |
| 文件操作 | * | `/v1/sandboxes/{id}/files` | 读写沙箱内文件 |
| 暂停沙箱 | POST | `/v1/sandboxes/{id}/pause` | 暂停（冻结进程，保留内存） |
| 恢复沙箱 | POST | `/v1/sandboxes/{id}/resume` | 恢复暂停的沙箱 |
| 快照 | POST | `/v1/sandboxes/{id}/snapshots` | 创建文件系统快照 |
| 终止沙箱 | DELETE | `/v1/sandboxes/{id}` | 强制终止并回收资源 |

#### 5.1.3 启动性能要求

| 指标 | 目标值 | 极限目标（12M） |
|------|--------|---------------|
| 冷启动（镜像已缓存）P50 | < 1s | < 500ms |
| 冷启动 P99 | < 3s | < 2s |
| 热启动（实例池预热） | < 200ms | < 100ms |
| 快照恢复 | < 5s | < 2s |

**关键实现机制**：
- **实例预热池（Warm Pool）**：预先启动一定数量的通用沙箱实例，收到请求后直接分配
- **镜像分层缓存**：在节点上缓存常用镜像层，避免重复拉取
- **快照 + CRIU**：使用 CRIU（Checkpoint/Restore In Userspace）在检查点处快速恢复状态

---

### 5.2 IDE（GUI）形态

#### 5.2.1 产品定位

面向 **Agent 开发者**的本地开发增强工具，以 IDE 插件形式提供沙箱能力，让开发者在熟悉的开发环境中无缝使用沙箱，**不改变已有开发习惯**。

#### 5.2.2 支持的 IDE

- **优先级 P0**：VS Code（扩展市场占比最高）
- **优先级 P1**：JetBrains 系列（IntelliJ IDEA、PyCharm、WebStorm）
- **优先级 P2**：Cursor、Windsurf（AI 原生 IDE）

#### 5.2.3 核心功能设计

##### A. 沙箱面板（Sandbox Panel）

在 IDE 左侧活动栏新增"沙箱"图标，点击展开沙箱面板：

```
┌────────────────────────────────────────┐
│ 🗂 沙箱管理                        [+] │
├────────────────────────────────────────┤
│ ▼ 当前会话                             │
│   📦 sandbox-xk9p2 [RUNNING] ●        │
│      模板: python-data-science-v2      │
│      运行: 00:12:34 | CPU: 23%         │
│      [打开终端] [文件浏览器] [停止]     │
├────────────────────────────────────────┤
│ ▼ 历史会话（今天）                      │
│   📦 sandbox-abc12 [STOPPED]           │
│      2026-03-05 14:23 | 持续 45min    │
│      [查看日志] [恢复快照]             │
├────────────────────────────────────────┤
│ ▼ 模板                                 │
│   🐍 python-data-science-v2            │
│   🟢 node-18-standard                  │
│   🦀 rust-1.75                         │
│   ➕ 从市场安装模板...                  │
└────────────────────────────────────────┘
```

##### B. 工具调用拦截与可视化

当 Agent 在 IDE 内（如 Claude Code 插件）发出工具调用时，沙箱插件自动拦截并在 IDE 内显示执行状态：

```
┌────────────────────────────────────────────────┐
│ 🔧 Agent 工具调用 — 沙箱执行中                  │
├────────────────────────────────────────────────┤
│ 工具: bash                                      │
│ 命令: pip install pandas matplotlib             │
│                                                │
│ 执行环境: sandbox-xk9p2 (python-ds-v2)         │
│ 状态: ████████░░ 运行中...                      │
│                                                │
│ 输出:                                          │
│ > Collecting pandas                            │
│ > Downloading pandas-2.2.0.tar.gz (4.0 MB)    │
│   ████████████████████ 4.0 MB/s               │
│                                                │
│ [在终端中查看] [允许] [拒绝] [查看策略]         │
└────────────────────────────────────────────────┘
```

##### C. 沙箱内文件浏览器

集成到 IDE 原生文件树，显示沙箱内文件系统状态，支持：
- 浏览 `/workspace` 目录内容
- 双击在 IDE 编辑器内直接编辑沙箱内文件
- 右键菜单：上传到沙箱 / 从沙箱下载
- 文件变更实时高亮（红/绿标注增删）

##### D. 集成终端

在 IDE 内嵌终端面板中，提供"连接到沙箱终端"选项：
- 一键 attach 到沙箱容器的 shell
- 终端标题栏显示沙箱 ID 和状态
- 支持多个沙箱终端并行开启

##### E. 差异比较器（Diff Viewer）

沙箱执行完毕后，可视化展示文件系统变更：
```
工作区变更摘要 (sandbox-xk9p2, 12:34 - 12:46)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
+ /workspace/output/analysis.csv         (新建, 128KB)
+ /workspace/charts/revenue_trend.png    (新建, 45KB)
~ /workspace/config.yaml                 (修改, +3/-1 行)
- /tmp/cache_old.pkl                     (删除)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[导出变更] [应用到本地] [丢弃] [创建快照]
```

#### 5.2.4 配置项（settings.json）

```json
{
  "agentSandbox.serverEndpoint": "https://sandbox.example.com",
  "agentSandbox.apiKey": "${env:SANDBOX_API_KEY}",
  "agentSandbox.defaultTemplate": "python-data-science-v2",
  "agentSandbox.autoIntercept": true,
  "agentSandbox.confirmBeforeExec": "dangerous",
  "agentSandbox.warmPool.enabled": true,
  "agentSandbox.warmPool.size": 2
}
```

`confirmBeforeExec` 取值：
- `always`：每次工具调用前弹确认框
- `dangerous`：仅对高风险操作（rm、curl、sudo 等）弹框
- `never`：全部自动执行（静默模式）

---

### 5.3 CLI 形态

#### 5.3.1 产品定位

面向**偏好命令行**的 Agent 开发者和 CI/CD 流水线场景，提供完整的沙箱管理能力。

#### 5.3.2 CLI 工具命名

```bash
sandbox-ctl
```

#### 5.3.3 命令结构

```
sandbox-ctl
├── sandbox          # 沙箱管理
│   ├── create       # 创建沙箱
│   ├── list         # 列出沙箱
│   ├── get          # 查看沙箱详情
│   ├── delete       # 删除沙箱
│   ├── pause        # 暂停沙箱
│   ├── resume       # 恢复沙箱
│   └── logs         # 查看日志流
├── exec             # 在沙箱内执行命令
├── cp               # 文件传输（本地 <-> 沙箱）
├── snapshot         # 快照管理
│   ├── create
│   ├── list
│   ├── restore
│   └── delete
├── template         # 模板管理
│   ├── list
│   ├── get
│   ├── create
│   └── delete
├── policy           # 策略管理
│   ├── list
│   ├── get
│   └── apply
├── config           # 本地配置
│   ├── set
│   ├── get
│   └── view
└── version          # 版本信息
```

#### 5.3.4 核心命令详细设计

##### `sandbox create`

```bash
# 基础用法
$ sandbox-ctl sandbox create \
    --template python-data-science-v2 \
    --name my-analysis-sandbox \
    --timeout 3600 \
    --env PYTHONPATH=/workspace/lib \
    --mount /local/data:/workspace/data:ro

# 输出
Creating sandbox... ✓
Sandbox ID:   sandbox-xk9p2
Template:     python-data-science-v2
Status:       RUNNING
Started at:   2026-03-05 14:23:01 CST
Expires at:   2026-03-05 15:23:01 CST
Endpoint:     grpc://sandbox-xk9p2.internal:50051
```

##### `sandbox exec`

```bash
# 同步执行（等待完成并返回输出）
$ sandbox-ctl exec sandbox-xk9p2 -- python3 analysis.py

# 异步执行（返回 job ID，后续查询）
$ sandbox-ctl exec sandbox-xk9p2 --async -- pip install -r requirements.txt
Job ID: job-abc123

# 交互式 shell
$ sandbox-ctl exec sandbox-xk9p2 -it -- bash

# 超时控制
$ sandbox-ctl exec sandbox-xk9p2 --timeout 60 -- python3 slow_script.py

# 环境变量
$ sandbox-ctl exec sandbox-xk9p2 -e API_KEY=xxx -e DEBUG=1 -- python3 main.py
```

##### `cp`（文件传输）

```bash
# 本地 -> 沙箱
$ sandbox-ctl cp ./data.csv sandbox-xk9p2:/workspace/data.csv

# 沙箱 -> 本地
$ sandbox-ctl cp sandbox-xk9p2:/workspace/output.json ./output.json

# 递归复制目录
$ sandbox-ctl cp -r ./src/ sandbox-xk9p2:/workspace/src/

# 支持通配符
$ sandbox-ctl cp sandbox-xk9p2:/workspace/results/*.csv ./results/
```

##### `sandbox logs`

```bash
# 实时日志流（tail -f 模式）
$ sandbox-ctl sandbox logs sandbox-xk9p2 -f

# 过滤日志级别
$ sandbox-ctl sandbox logs sandbox-xk9p2 --level warn,error

# 指定时间范围
$ sandbox-ctl sandbox logs sandbox-xk9p2 \
    --since "2026-03-05T14:00:00Z" \
    --until "2026-03-05T15:00:00Z"

# 过滤工具调用类型
$ sandbox-ctl sandbox logs sandbox-xk9p2 --type tool_call

# 导出 JSON 格式（用于审计）
$ sandbox-ctl sandbox logs sandbox-xk9p2 --format json > audit.json
```

##### Pipeline 集成示例

```bash
#!/bin/bash
# CI/CD 流水线中使用沙箱执行 Agent 任务

# 1. 创建沙箱
SANDBOX_ID=$(sandbox-ctl sandbox create \
    --template python-data-science-v2 \
    --timeout 1800 \
    --output id)

# 2. 上传代码
sandbox-ctl cp -r ./agent_code/ $SANDBOX_ID:/workspace/

# 3. 执行 Agent 任务
sandbox-ctl exec $SANDBOX_ID \
    --timeout 1200 \
    -- python3 /workspace/run_agent.py \
    --task "分析 Q1 销售数据并生成报告"

EXIT_CODE=$?

# 4. 下载结果
sandbox-ctl cp $SANDBOX_ID:/workspace/reports/ ./ci-outputs/

# 5. 清理
sandbox-ctl sandbox delete $SANDBOX_ID

exit $EXIT_CODE
```

#### 5.3.5 配置文件（~/.sandbox-ctl/config.yaml）

```yaml
server:
  endpoint: "https://sandbox.example.com"
  api_key: "${SANDBOX_API_KEY}"
  timeout: 30s

defaults:
  template: "python-data-science-v2"
  sandbox_timeout: "30m"
  confirm_dangerous: true   # 执行 rm/sudo 等前确认

output:
  format: "human"           # human | json | yaml
  color: true

plugins:
  - name: sandbox-hooks
    path: "~/.sandbox-ctl/plugins/hooks.so"
```

---

### 5.4 Web UI 形态

#### 5.4.1 产品定位

面向**非技术用户**（产品经理、安全合规、企业管理员）的可视化控制台，提供完整的沙箱管理、监控、审计能力。

#### 5.4.2 整体导航结构

```
顶部导航: [Logo] [概览] [沙箱] [任务] [模板] [策略] [审计] [设置]
                                               [组织: Acme Corp ▾] [用户 ▾]
```

#### 5.4.3 页面详细设计

##### A. 概览仪表盘（Dashboard）

```
┌─────────────────────────────────────────────────────────────────────┐
│ AI Agent 沙箱控制台                              今日: 2026-03-05   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌───────────┐ │
│  │  运行中沙箱  │ │  今日任务数  │ │ 平均启动时长 │ │ 安全告警  │ │
│  │    1,234     │ │    8,891     │ │   1.2s P50   │ │     3     │ │
│  │  ↑12% vs昨日│ │  ↑5% vs昨日 │ │  ↓0.3s vs昨  │ │ ⚠ 需处理  │ │
│  └──────────────┘ └──────────────┘ └──────────────┘ └───────────┘ │
│                                                                     │
│  资源使用趋势（最近 24 小时）                 [1h] [6h] [24h] [7d] │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │  CPU ────────────────────────────────────── 63%              │ │
│  │  内存 ──────────────────────────── 48%                       │ │
│  │  网络出口 ─────── 12%                                         │ │
│  │  [折线图]                                                     │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  最近告警                                           [查看全部 →]    │
│  ⚠ 2026-03-05 14:15  sandbox-xk9p2 内存超过 80% 阈值             │
│  ⚠ 2026-03-05 13:42  sandbox-abc11 尝试访问被禁止的外网域名       │
│  ✅ 2026-03-05 12:30  sandbox-zz001 安全策略更新已应用             │
└─────────────────────────────────────────────────────────────────────┘
```

##### B. 沙箱列表页

```
┌─────────────────────────────────────────────────────────────────────┐
│ 沙箱列表                                              [+ 新建沙箱]  │
├─────────────────────────────────────────────────────────────────────┤
│ 🔍 搜索沙箱...  状态▾  模板▾  创建者▾  时间范围▾          [导出] │
├──────────┬───────────────┬───────────┬──────────┬────────┬─────────┤
│ 沙箱 ID  │ 名称          │ 状态      │ 模板     │ 运行时 │ 操作    │
├──────────┼───────────────┼───────────┼──────────┼────────┼─────────┤
│ xk9p2    │ 数据分析-03   │ ● RUNNING │ python-ds│ 12:34  │ 终端    │
│          │               │           │          │        │ 停止    │
│          │               │           │          │        │ 详情 ›  │
├──────────┼───────────────┼───────────┼──────────┼────────┼─────────┤
│ abc11    │ 代码生成任务  │ ◌ PAUSED  │ node-18  │ 05:11  │ 恢复    │
│          │               │           │          │        │ 终止    │
│          │               │           │          │        │ 详情 ›  │
├──────────┼───────────────┼───────────┼──────────┼────────┼─────────┤
│ zz001    │ 安全扫描任务  │ ○ STOPPED │ ubuntu-  │ 23:45  │ 详情 ›  │
│          │               │           │ base     │        │ 删除    │
└──────────┴───────────────┴───────────┴──────────┴────────┴─────────┘
```

##### C. 沙箱详情页

```
┌─────────────────────────────────────────────────────────────────────┐
│ ← 返回列表   sandbox-xk9p2   ● RUNNING         [暂停] [终止] [···] │
├──────────┬──────────────────────────────────────────────────────────┤
│          │                                                          │
│ [概览]   │ 基本信息                                                 │
│ [终端]   │ 沙箱 ID:    sandbox-xk9p2                               │
│ [文件]   │ 模板:       python-data-science-v2                      │
│ [日志]   │ 创建时间:   2026-03-05 14:23:01                         │
│ [网络]   │ 到期时间:   2026-03-05 15:23:01 (剩余 47:26)            │
│ [快照]   │ 创建者:     user@example.com                            │
│ [策略]   │                                                         │
│          │ 资源使用                                                 │
│          │ CPU:    ████████░░ 82%  (上限 2 vCPU)                  │
│          │ 内存:   ████░░░░░░ 43%  (上限 4GiB)                    │
│          │ 磁盘:   ██░░░░░░░░ 23%  (上限 10GiB)                   │
│          │ 网络:   出口 1.2 MB/s / 入口 0.3 MB/s                  │
│          │                                                         │
│          │ 工具调用统计（本次会话）                                  │
│          │ bash: 47次  |  write_file: 12次  |  http_get: 8次      │
└──────────┴──────────────────────────────────────────────────────────┘
```

##### D. Web 终端（WebShell）

在浏览器内直接连接沙箱 Shell，基于 xterm.js 实现：
- 完整的 ANSI 颜色和键盘支持
- 多标签页终端（一个沙箱可开多个 Shell）
- 终端录制（Record & Replay）
- 复制粘贴、字体大小调整

##### E. 审计日志页

```
┌─────────────────────────────────────────────────────────────────────┐
│ 审计日志                                                            │
├─────────────────────────────────────────────────────────────────────┤
│ 沙箱▾  时间范围▾  操作类型▾  风险等级▾    [导出 CSV] [导出 JSON] │
├─────────────────────────────────────────────────────────────────────┤
│ 时间                │ 沙箱      │ 操作类型    │ 详情          │ 风险│
├─────────────────────┼───────────┼─────────────┼───────────────┼─────┤
│ 14:35:22            │ xk9p2     │ 命令执行    │ bash: curl    │ ⚠中  │
│                     │           │             │ http://ext... │     │
├─────────────────────┼───────────┼─────────────┼───────────────┼─────┤
│ 14:34:11            │ xk9p2     │ 文件写入    │ /workspace/   │ 低  │
│                     │           │             │ output.csv    │     │
├─────────────────────┼───────────┼─────────────┼───────────────┼─────┤
│ 14:33:05            │ xk9p2     │ 网络请求    │ GET pypi.org  │ 低  │
│                     │           │             │ /simple/numpy │     │
└─────────────────────┴───────────┴─────────────┴───────────────┴─────┘
```

##### F. 策略管理页

提供可视化的策略编辑器，支持：
- 网络策略：白名单域名、出口带宽限制、禁止外网
- 文件系统策略：只读目录、禁止访问路径
- 进程策略：禁止的命令列表、最大进程数
- 时间策略：最大存活时间、空闲超时

##### G. 模板市场

```
┌─────────────────────────────────────────────────────────────────────┐
│ 沙箱模板市场                                        [发布我的模板]  │
├─────────────────────────────────────────────────────────────────────┤
│ 🔍 搜索模板...    [全部] [官方] [社区] [我的]                      │
│                                                                     │
│ 🏷 按语言: Python  Node.js  Java  Go  Rust  Ruby  ...              │
│ 🏷 按用途: 数据分析  Web开发  安全测试  机器学习  DevOps  ...       │
│                                                                     │
│ ┌──────────────────────────┐  ┌──────────────────────────┐         │
│ │ 🐍 Python 数据科学 v2     │  │ 🟢 Node.js 18 全栈       │         │
│ │ 官方出品 ⭐4.9 | 12.3k用  │  │ 官方出品 ⭐4.8 | 8.1k用  │         │
│ │ Python 3.12, pandas,     │  │ Node 18, npm, Git       │         │
│ │ numpy, matplotlib, Jupyter│  │ 内置 TypeScript 支持     │         │
│ │              [使用模板]   │  │              [使用模板]  │         │
│ └──────────────────────────┘  └──────────────────────────┘         │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 5.5 工具调用与 MCP 集成

#### 5.5.1 内置工具集

沙箱默认暴露以下工具给 Agent 调用：

| 工具名 | 描述 | 示例 |
|--------|------|------|
| `sandbox_exec` | 执行 Shell 命令 | `{"cmd": "python3 main.py", "timeout": 60}` |
| `sandbox_write_file` | 写入文件 | `{"path": "/workspace/a.py", "content": "..."}` |
| `sandbox_read_file` | 读取文件 | `{"path": "/workspace/a.py"}` |
| `sandbox_list_dir` | 列出目录 | `{"path": "/workspace"}` |
| `sandbox_http_request` | 发起 HTTP 请求（受策略约束） | `{"url": "...", "method": "GET"}` |
| `sandbox_install_package` | 安装软件包 | `{"package": "pandas", "manager": "pip"}` |
| `sandbox_snapshot` | 创建文件系统快照 | `{"label": "before-refactor"}` |

#### 5.5.2 MCP（Model Context Protocol）服务端

沙箱暴露标准 MCP 服务端，任何支持 MCP 的 Agent 框架（LangChain、CrewAI、AutoGen 等）可直接接入：

```json
{
  "mcpServers": {
    "agent-sandbox": {
      "command": "sandbox-mcp-server",
      "args": ["--sandbox-id", "sandbox-xk9p2"],
      "env": {
        "SANDBOX_API_KEY": "${SANDBOX_API_KEY}"
      }
    }
  }
}
```

#### 5.5.3 SDK 集成示例

```python
from anthropic import Anthropic
from sandbox_sdk import SandboxClient, SandboxTools

# 初始化
client = Anthropic()
sandbox = SandboxClient(api_key="sk-...")

# 创建沙箱
with sandbox.create(template="python-data-science-v2") as sb:
    tools = SandboxTools(sb)

    # 将沙箱工具传递给 Claude
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=4096,
        tools=tools.to_anthropic_tools(),
        messages=[{
            "role": "user",
            "content": "分析 /workspace/sales.csv 文件，生成可视化图表"
        }]
    )

    # 处理工具调用（自动路由到沙箱执行）
    result = tools.process_tool_calls(response)
```

---

### 5.6 安全与隔离机制

#### 5.6.1 隔离层次

```
┌─────────────────────────────────────────────────────┐
│  Layer 4: 网络策略（出口白名单 + 带宽限速）           │
│  ─────────────────────────────────────────────────  │
│  Layer 3: 文件系统隔离（OverlayFS + 路径白名单）      │
│  ─────────────────────────────────────────────────  │
│  Layer 2: 进程隔离（Namespace + Cgroups + Seccomp）  │
│  ─────────────────────────────────────────────────  │
│  Layer 1: 内核隔离（gVisor 用户态内核）               │
└─────────────────────────────────────────────────────┘
```

**Layer 1 - gVisor**：使用 Google gVisor 作为容器运行时，提供用户态内核（Sentry），将沙箱的系统调用截获在用户态处理，防止内核漏洞利用。

**Layer 2 - Namespace + Cgroups**：
- PID Namespace：进程不可见宿主机进程
- Network Namespace：完全独立网络栈
- Mount Namespace：独立文件系统视图
- Cgroups v2：CPU、内存、IO 硬限制

**Layer 3 - Seccomp Filter**：白名单系统调用，默认禁止 `ptrace`, `kexec_load`, `perf_event_open` 等危险调用

**Layer 4 - 网络出口控制**：
- 出口流量经过统一代理（Envoy Proxy）
- 基于域名白名单的出口控制（DNS + SNI 过滤）
- 禁止访问宿主机元数据服务（169.254.169.254）

#### 5.6.2 敏感数据检测

在工具调用执行前后，自动检测：
- API Key 泄露（正则 + 语义）
- 信用卡号、身份证号等 PII 数据
- 私钥文件（`-----BEGIN PRIVATE KEY-----`）

检测到后动作可配置：`block` / `alert` / `redact`（脱敏后放行）

#### 5.6.3 危险命令拦截

预设危险命令规则引擎（可扩展）：

| 级别 | 规则示例 | 默认行为 |
|------|----------|----------|
| CRITICAL | `rm -rf /`，`mkfs.*`，`dd if=/dev/zero` | 直接拒绝 |
| HIGH | `curl * \| bash`，`wget * \| sh`，`chmod 777 /` | 需用户确认 |
| MEDIUM | `sudo *`，`su -`，`passwd` | 记录告警 |
| LOW | `pip install *`，`apt install *` | 仅记录 |

#### 5.6.4 镜像安全扫描

- 所有模板镜像在发布前经过 Trivy 漏洞扫描
- 高危 CVE（CVSS ≥ 9.0）阻断发布
- 中危 CVE 标注警告，定期自动更新

---

### 5.7 资源配额与计费

#### 5.7.1 配额维度

| 维度 | 描述 | 配置层级 |
|------|------|----------|
| 并发沙箱数 | 同时运行的沙箱实例上限 | 组织 / 团队 / 用户 |
| 单沙箱 CPU 上限 | vCPU 数量（0.25 - 32） | 模板 / 策略 |
| 单沙箱内存上限 | GB（0.5 - 128） | 模板 / 策略 |
| 单沙箱磁盘上限 | GB（1 - 500） | 模板 / 策略 |
| 单沙箱存活时间 | 最大 TTL（1min - 24h） | 策略 |
| 月度 CPU 时 | 累计使用的 vCPU·小时 | 组织级 |
| 月度出口流量 | GB | 组织级 |

#### 5.7.2 计费模型

```
月度账单 = Σ (沙箱运行时长 × vCPU 数 × vCPU 单价)
         + Σ (沙箱运行时长 × 内存 GB × 内存单价)
         + Σ (出口流量 GB × 流量单价)
         + 快照存储 GB × 存储单价
```

计费精度：**按秒计费**，最小计费单位 1 秒。

#### 5.7.3 超配额行为

| 配额类型 | 超配额行为 |
|----------|-----------|
| 并发数上限 | 新建请求排队，等待超时返回 429 |
| CPU 上限 | Cgroups 硬限制，CPU 被限速 |
| 内存上限 | 触发 OOM Killer，沙箱进入 ERROR 状态 |
| 磁盘上限 | 写入失败，返回 ENOSPC |
| TTL 超限 | 自动发送 SIGTERM，30s 后 SIGKILL |
| 月度配额超限 | 告警 + 可选自动暂停新建 |

---

### 5.8 可观测性与审计

#### 5.8.1 指标体系（Metrics）

**黄金指标（RED）**：
- **Request Rate**：沙箱创建 QPS、工具调用 QPS
- **Error Rate**：沙箱启动失败率、工具执行错误率
- **Duration**：沙箱启动延迟、工具执行延迟 P50/P95/P99

**资源指标**：
- 沙箱 CPU/内存使用率（实时 + 趋势）
- 节点级资源水位（用于调度决策）

#### 5.8.2 日志分级

| 日志类型 | 内容 | 保留期 |
|----------|------|--------|
| 系统日志 | 沙箱创建/删除事件 | 90 天 |
| 审计日志 | 全量工具调用记录（含参数、结果摘要） | 365 天 |
| 安全日志 | 策略违规、敏感数据检测 | 365 天 |
| 调试日志 | 沙箱内 stdout/stderr | 7 天 |

#### 5.8.3 审计日志格式（NDJSON）

```json
{
  "timestamp": "2026-03-05T14:35:22.123Z",
  "sandbox_id": "sandbox-xk9p2",
  "session_id": "session-yyy",
  "org_id": "org-acme",
  "user_id": "user-123",
  "event_type": "tool_call",
  "tool_name": "sandbox_exec",
  "tool_input": {
    "cmd": "curl https://api.example.com/data",
    "timeout": 30
  },
  "result": {
    "exit_code": 0,
    "stdout_size": 1024,
    "stderr_size": 0,
    "duration_ms": 823
  },
  "risk_level": "MEDIUM",
  "policy_actions": [],
  "trace_id": "trace-zzz"
}
```

#### 5.8.4 告警规则（默认预置）

| 告警名 | 触发条件 | 严重度 |
|--------|----------|--------|
| 沙箱启动超时 | 启动时间 > 10s | WARNING |
| 内存使用率高 | 内存 > 85% 持续 5min | WARNING |
| 高危命令执行 | CRITICAL 级命令被执行 | ERROR |
| 网络策略违规 | 访问白名单外域名 | ERROR |
| 批量沙箱创建 | 1min 内创建 > 100 个沙箱 | WARNING |
| 沙箱逃逸检测 | 检测到容器逃逸特征 | CRITICAL |

---

## 6. API 设计规范

### 6.1 API 风格

- **协议**：RESTful HTTP/1.1 + WebSocket（实时事件流）
- **数据格式**：JSON（Content-Type: application/json）
- **版本控制**：URL 路径版本（/v1/）
- **认证**：Bearer Token（API Key） + OAuth2（用户授权）
- **限流**：令牌桶算法，默认 1000 req/min/key

### 6.2 通用响应格式

**成功响应**：
```json
{
  "data": { ... },
  "request_id": "req-abc123"
}
```

**错误响应**：
```json
{
  "error": {
    "code": "SANDBOX_NOT_FOUND",
    "message": "Sandbox sandbox-xk9p2 not found",
    "details": { ... }
  },
  "request_id": "req-abc123"
}
```

### 6.3 核心 API 示例

**创建沙箱**：
```http
POST /v1/sandboxes
Authorization: Bearer sk-...
Content-Type: application/json

{
  "template_id": "python-data-science-v2",
  "name": "数据分析任务",
  "timeout": 3600,
  "env": {
    "PYTHONPATH": "/workspace/lib"
  },
  "metadata": {
    "agent_id": "agent-xyz",
    "task_id": "task-001"
  }
}

Response 201:
{
  "data": {
    "id": "sandbox-xk9p2",
    "status": "STARTING",
    "created_at": "2026-03-05T14:23:01Z",
    "expires_at": "2026-03-05T15:23:01Z",
    "endpoints": {
      "grpc": "grpc://sandbox-xk9p2.internal:50051",
      "websocket": "wss://sandbox-xk9p2.internal/ws"
    }
  }
}
```

**执行命令**：
```http
POST /v1/sandboxes/{id}/exec
Content-Type: application/json

{
  "cmd": ["python3", "-c", "print('hello world')"],
  "timeout": 30,
  "env": {"FOO": "bar"},
  "stdin": null
}

Response 200:
{
  "data": {
    "exit_code": 0,
    "stdout": "hello world\n",
    "stderr": "",
    "duration_ms": 45
  }
}
```

**WebSocket 实时日志**：
```
ws://api.sandbox.example.com/v1/sandboxes/{id}/logs?token=...

服务端推送 JSON 事件：
{"type": "stdout", "data": "Installing numpy...\n", "ts": "2026-..."}
{"type": "tool_call", "tool": "bash", "input": {...}, "ts": "2026-..."}
{"type": "sandbox_status", "status": "RUNNING", "ts": "2026-..."}
```

---

## 7. 数据模型设计

### 7.1 核心实体关系

```
Organization (1) ──── (*) Team (1) ──── (*) User
     │                      │
     └──── (*) Policy        └──── (*) SandboxSession
                                          │
                              (*) ──── SandboxInstance ──── (*) ToolCallRecord
                                          │
                                    (*) ──── Snapshot
```

### 7.2 SandboxInstance 模型

```typescript
interface SandboxInstance {
  id: string;                      // "sandbox-xk9p2"
  name?: string;                   // 用户自定义名称
  status: SandboxStatus;           // PENDING|STARTING|RUNNING|BUSY|PAUSED|STOPPING|STOPPED|ERROR|DELETED
  template_id: string;             // 基础模板
  org_id: string;
  team_id?: string;
  created_by: string;              // user_id
  session_id?: string;             // 关联的 Agent 会话

  spec: {
    cpu_limit: number;             // vCPU
    memory_limit: string;          // "4Gi"
    disk_limit: string;            // "10Gi"
    timeout: number;               // seconds
    env: Record<string, string>;
    network_policy: NetworkPolicy;
  };

  status_detail: {
    node_id: string;               // 运行在哪个物理节点
    container_id: string;
    started_at?: Date;
    stopped_at?: Date;
    exit_code?: number;
    error_message?: string;
  };

  metrics: {
    cpu_usage_percent: number;
    memory_usage_bytes: number;
    disk_usage_bytes: number;
    network_rx_bytes: number;
    network_tx_bytes: number;
    tool_call_count: number;
  };

  created_at: Date;
  updated_at: Date;
  expires_at: Date;
}
```

---

## 8. 非功能需求

### 8.1 性能指标

| 指标 | 目标 |
|------|------|
| 沙箱冷启动（P99） | < 3s |
| 沙箱热启动（P99） | < 300ms |
| API 响应延迟（P95，非执行类） | < 100ms |
| 单集群最大并发沙箱数 | 50,000 |
| 系统可用性 | 99.9% / 月（≤ 43.2 分钟故障） |

### 8.2 可靠性

- **沙箱节点故障**：在 30 秒内自动迁移或重建沙箱（有状态场景通过快照恢复）
- **控制面高可用**：关键服务 3 副本，跨可用区部署
- **数据持久性**：快照数据三副本存储，持久性 99.999999999%（11 个 9）

### 8.3 安全合规

- **等保三级**：满足国内等保三级要求
- **数据本地化**：支持私有化部署，数据不出境
- **RBAC**：基于角色的访问控制，最小权限原则
- **加密**：传输加密（TLS 1.3），存储加密（AES-256）
- **漏洞响应**：P0 级安全漏洞 4 小时内响应，24 小时内修复

### 8.4 可维护性

- 所有服务提供 `/health`, `/ready`, `/metrics` 端点
- 蓝绿部署，零停机升级
- 支持混沌工程（集成 Chaos Monkey）

---

## 9. 成功指标（OKR / KPI）

### 9.1 用户体验指标

| 指标 | 基线 | 6 月目标 | 12 月目标 |
|------|------|----------|----------|
| 开发者首次接入时间（Time-to-Hello-World） | N/A | < 30min | < 10min |
| NPS（开发者净推荐值） | N/A | ≥ 40 | ≥ 60 |
| 沙箱创建成功率 | N/A | ≥ 99% | ≥ 99.9% |

### 9.2 增长指标

| 指标 | 6 月目标 | 12 月目标 |
|------|----------|----------|
| 月活沙箱实例数（MAI） | 10,000 | 100,000 |
| 注册开发者数 | 500 | 5,000 |
| 接入 Agent 框架数 | 3 | 10 |

### 9.3 安全指标

| 指标 | 目标 |
|------|------|
| 沙箱逃逸事件 | 0 |
| 高危漏洞修复时间（P0 CVE） | ≤ 24h |
| 安全策略覆盖率 | 100% 沙箱受策略管控 |

### 9.4 商业指标

| 指标 | 12 月目标 |
|------|----------|
| 月度 ARR | ¥500万 |
| 付费组织数 | 50 |
| 平均单客 CPU 时消耗 | > 1000 vCPU·h/月 |

---

## 10. 竞品分析

| 维度 | **本产品** | E2B | Daytona | Modal | GitHub Codespaces |
|------|-----------|-----|---------|-------|-------------------|
| **定位** | 企业级 Agent 沙箱 | 开发者 Agent 沙箱 | 开发环境管理 | 无服务器计算 | 云开发环境 |
| **冷启动** | < 1s（目标） | ~150ms | ~5s | ~1s | ~30s |
| **安全隔离** | gVisor + Seccomp | gVisor | Docker | gVisor | Docker |
| **MCP 支持** | ✅ 原生 | ✅ | ❌ | ❌ | ❌ |
| **Web UI** | ✅ 完整控制台 | 基础 | ✅ | 基础 | ✅ |
| **审计合规** | ✅ 等保三级 | ❌ | ❌ | ❌ | ❌ |
| **私有化部署** | ✅ | ❌ | ✅ | ❌ | ❌ |
| **中文支持** | ✅ 原生 | ❌ | ❌ | ❌ | 部分 |
| **定价** | 按秒 | 按秒 | 按月席位 | 按秒 | 按月席位 |

**核心差异化**：
1. **企业合规**：面向中国市场的等保三级合规能力，是竞品普遍缺失的
2. **IDE 深度集成**：提供 VS Code 插件级别的集成，而非仅 API
3. **私有化部署**：满足金融、政务等强监管客户的数据本地化要求

---

## 11. 发布路线图（Roadmap）

### Phase 0：地基（M1-M2）

**目标**：搭建核心基础设施，内部验证可行性

- [ ] 沙箱核心引擎（基于 gVisor + Docker）
- [ ] 基础 REST API（创建/执行/删除）
- [ ] 简单的 Python SDK
- [ ] 内部红队安全测试

**里程碑**：内部 Demo 可跑通"Claude Code Agent 安全执行代码"场景

---

### Phase 1：MVP（M3-M4）

**目标**：对外开放内测，获取首批开发者反馈

- [ ] CLI 工具（sandbox-ctl）核心命令
- [ ] Web UI 基础版（沙箱列表、详情、日志）
- [ ] 沙箱模板系统（内置 5 个官方模板）
- [ ] 基础审计日志
- [ ] VS Code 插件 Alpha 版

**里程碑**：50 名内测开发者，首批付费用户

---

### Phase 2：生产就绪（M5-M8）

**目标**：达到生产环境可用标准，启动商业化

- [ ] 高可用架构（多副本、跨 AZ）
- [ ] 完整 Web UI 控制台（监控、策略、审计）
- [ ] VS Code 插件正式版 + JetBrains 插件
- [ ] MCP 服务端完整实现
- [ ] 资源配额与计费系统
- [ ] 等保三级合规报告
- [ ] SLA 99.9% 保证

**里程碑**：5 家付费企业客户，MAI > 10,000

---

### Phase 3：规模化（M9-M12）

**目标**：规模化增长，拓展生态

- [ ] 沙箱模板市场（开放社区投稿）
- [ ] 多 Agent 协同沙箱（Agent 间隔离通信）
- [ ] 快照市场（可共享的环境快照）
- [ ] 私有化部署版本（K8s Helm Chart）
- [ ] 更多 Agent 框架集成（LangChain、AutoGen、CrewAI）
- [ ] 沙箱 Benchmark 工具

**里程碑**：MAI > 100,000，ARR > ¥500万

---

### Phase 4：平台化（M13-M24）

**目标**：成为 AI Agent 基础设施平台

- [ ] Serverless 沙箱（按需冷启动，无需预先创建）
- [ ] 跨区域沙箱网络（全球节点）
- [ ] 合规报告自动化（SOC2、ISO 27001）
- [ ] AI 驱动的沙箱行为异常检测
- [ ] 沙箱编排（Workflow：多个沙箱串联/并联）

---

## 12. 风险与依赖

### 12.1 技术风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| gVisor 安全漏洞 | 中 | 高 | 持续 CVE 监控 + 多层防御 |
| 冷启动性能不达标 | 中 | 高 | 早期 PoC 验证 + 预热池设计 |
| 大规模调度性能瓶颈 | 低 | 高 | 分片调度 + 压测验证 |
| 容器逃逸 0-Day | 低 | 极高 | gVisor 额外隔离 + 入侵检测 |

### 12.2 依赖项

| 依赖 | 类型 | 风险 |
|------|------|------|
| gVisor | 开源技术 | 低（Google 维护，活跃社区） |
| Kubernetes | 基础设施 | 低（行业标准） |
| Claude API | 外部 API | 中（需处理限流和可用性） |
| MCP 协议 | 开放协议 | 低（Anthropic 主导，标准化） |

### 12.3 商业风险

| 风险 | 缓解措施 |
|------|----------|
| E2B 等竞品快速跟进企业市场 | 加速私有化部署和合规认证 |
| 大客户安全审计周期长 | 提前准备等保材料，配套咨询服务 |
| LLM API 成本上升 | 沙箱本身不调用 LLM，成本可控 |

---

## 13. 附录：术语表

| 术语 | 解释 |
|------|------|
| **沙箱（Sandbox）** | 隔离的代码执行环境，防止 Agent 操作影响宿主机 |
| **Agent** | 能够自主调用工具、执行任务的 AI 应用 |
| **MCP** | Model Context Protocol，Anthropic 提出的 AI 工具调用标准协议 |
| **gVisor** | Google 开源的用户态内核，提供比 Docker 更强的隔离性 |
| **工具调用（Tool Call）** | Agent 发出的调用外部工具的请求（如 bash、write_file） |
| **CRIU** | Checkpoint/Restore In Userspace，进程快照与恢复工具 |
| **Seccomp** | Linux 内核安全机制，限制进程可调用的系统调用 |
| **OverlayFS** | 联合文件系统，实现容器写时复制的文件系统层 |
| **TTL** | Time-To-Live，沙箱最大存活时间 |
| **Warm Pool** | 预热实例池，预先启动的沙箱实例，用于快速分配 |
| **PRD** | Product Requirements Document，产品需求文档 |
| **RED 指标** | Rate（请求率）、Error（错误率）、Duration（延迟），黄金监控指标 |
| **MAI** | Monthly Active Instance，月活沙箱实例数 |
| **ARR** | Annual Recurring Revenue，年度经常性收入 |

---

*文档结束*

*最后更新: 2026-03-05 | 版本: v1.0 | 下次评审: 2026-04-05*
