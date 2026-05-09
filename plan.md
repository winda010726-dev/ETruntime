# Edge AgentOps: 工业诊断大脑

## 多 Agent 编排 + 三层递进诊断 + eBPF 内核级观测

---

## 一、项目定位

### 一句话定义

面向 Jetson 视觉测振系统的工业诊断大脑——多 Agent 编排架构，双触发模式（问答 + 告警），三层递进式诊断（Trace → Metrics → eBPF），覆盖从 LLM 推理到内核系统调用的全栈可观测性。

### 这个项目不是什么

| 它不是 | 它是 |
|--------|------|
| 运维监控面板 | 多 Agent 协作的智能诊断系统 |
| 套框架的 demo | 从零手写的 ReAct 引擎 + Agent 编排器 |
| 只看 Dashboard | 能从应用层追到内核层的全栈诊断 |
| 单一 Agent 项目 | Orchestrator + 专项 Agent 的可扩展架构 |
| 只能问答的机器人 | 告警驱动 + 问答双模式，不需要人值守 |

### 核心竞争力分层

```
┌─────────────────────────────────────────────┐
│  Layer 3: eBPF 内核级诊断 (护城河)            │  ← 别人做不到
│  把 8s 慢请求拆解为: CPU调度350ms +           │
│  磁盘IO 3.85s + GPU推理 3.6s                 │
├─────────────────────────────────────────────┤
│  Layer 2: 可观测性架构 (差异化)               │  ← 别人不做
│  Trace + Metrics + Snapshot 三支柱           │
│  Agent 每步决策可追溯、可复现、可审计          │
├─────────────────────────────────────────────┤
│  Layer 1: Agent 推理引擎 (入场券)             │  ← 别人也做
│  ReAct + Function Calling + 多 Agent 编排    │
│  手写，不用框架                               │
└─────────────────────────────────────────────┘
```

### 每个技术亮点对应的落地痛点

| 落地痛点 | 解法 | 面试怎么讲 |
|---------|------|-----------|
| LLM 输出不可控，可能乱调工具 | 工具注册表 + JSON 校验 + 错误回传 LLM | "LLM 幻觉出不存在的工具名，我的注册表会拦截并把错误当作 observation 返回，让 LLM 换方案" |
| Agent 死循环，无限调工具不停 | 步数限制 + Token 预算 + 超时熔断 | "步数护栏 + Token 预算双重保护，触发后降级输出已收集的部分证据" |
| 单个 Agent 能力有限 | 多 Agent 编排，按领域路由 | "单 Agent 准确率 60%，引入 Orchestrator 路由后提升到 85%" |
| LLM 诊断结论不可信（幻觉） | 规则引擎确定性归因 + LLM 只写报告 | "确定性因果用规则（camera_fps=0 → 相机故障），LLM 只负责写人话" |
| Agent 出错了无法排查 | OpenTelemetry Trace + 请求快照 | "拿 trace_id 在 Jaeger 看瀑布图，5 分钟定位是 LLM 的问题还是工具的问题" |
| 应用层指标看不出根因 | eBPF 内核级追踪 | "Trace 说工具超时 8s，eBPF 告诉我其中 45% 是磁盘 IO 阻塞" |
| 危险操作不可控 | 工具三级分类 + Human-in-the-Loop | "Level 2 操作必须人类确认，执行后自动验证恢复状态" |
| 没人盯着终端问 Agent | 告警驱动模式 + 自动触发诊断 | "工业现场不需要人去问，系统自动监控指标，超阈值自动触发 Agent 诊断并推送" |
| Agent 修复方案不确定能否生效 | 仿真环境先验证再执行 | "修复方案先在仿真环境执行，验证指标恢复后，人确认才在真实设备执行" |

### 面试叙事

> 我设计并实现了一个面向边缘视觉系统的工业诊断大脑。架构上采用 Orchestrator + 专项 Agent 的编排模式，支持插件化扩展。系统有两种触发模式：交互式问答和告警驱动自动诊断。
> 诊断能力分三层递进：第一层用 OpenTelemetry 做 Agent 决策链路追踪，定位"哪一步慢了"；第二层用 Prometheus 做系统指标分析，定位"哪个资源出了问题"；第三层用 eBPF 做内核级追踪，把一个 8 秒的慢请求精确拆解为 CPU 调度延迟、磁盘 IO 阻塞和 GPU 推理各占多少。
> 修复方案先在仿真环境验证，通过后人工确认才在真实设备执行。全程可追溯、可复现、可审计。

---

## 二、目标体系

### 近期目标（5 个月，2026.05 - 2026.09）

- 完成多 Agent 编排的诊断系统，覆盖 4 个 Demo 场景
- 实现双触发模式（问答 + 告警驱动）
- eBPF 实现 2 个内核追踪脚本
- 秋招面试能讲 15 分钟不重复，经得住追问
- GitHub 仓库完整可展示

### 中期目标（1-2 年）

- 进入 AI Infra / 性能工程岗位
- eBPF 深入到 GPU 驱动层追踪
- 项目持续迭代，接入更多诊断 Agent

### 远期目标

- 成为 AI 系统可靠性 + 性能工程领域的专家

---

## 三、双触发模式

### 这是你的项目和普通 Agent demo 的关键区别

```
模式 1: 被动问答（Demo / 调试用）
  用户提问 → Agent 诊断 → 返回报告
  适用: 开发调试、Demo 展示

模式 2: 主动告警（工业现场用）
  监控循环 → 指标异常 → 自动触发 Agent 诊断 → 推送告警
  适用: 无人值守的工业部署
```

### 告警驱动流程

```
Prometheus / 监控循环 每 N 秒采集指标
  ↓
阈值检查:
  gpu_load > 80% 持续 30s?
  temperature > 70°C?
  fps < 15?
  camera_fps = 0?
  ↓
触发 → 自动生成问题描述
  "系统告警: GPU负载85%, 温度75°C。请诊断原因。"
  ↓
Agent 自动执行诊断
  ↓
推送结果（钉钉/企业微信/日志）
  ↓
如果需要修复 → 等待人工确认
```

### 告警规则（可配置）

```yaml
rules:
  - name: gpu_overload
    condition: gpu_load > 80 AND duration > 30s
    severity: warning
    auto_diagnose: true

  - name: thermal_throttling_risk
    condition: temperature > 70
    severity: critical
    auto_diagnose: true

  - name: camera_failure
    condition: camera_fps == 0
    severity: critical
    auto_diagnose: true

  - name: vision_degraded
    condition: fps < 15 OR inference_latency_ms > 100
    severity: warning
    auto_diagnose: true

  - name: agent_loop_risk
    condition: agent_step_count > 6
    severity: warning
    auto_diagnose: false
```

---

## 四、仿真环境 + 故障注入

### 为什么需要仿真

```
问题 1: Jetson 不在身边 → 用仿真器开发和测试
问题 2: Jetson 运行正常时没有故障 → 用故障注入制造故障
问题 3: 修复操作有风险 → 先在仿真环境验证
```

### 仿真器架构

```
JetsonSimulator（开发阶段）
  ├── 场景切换: load_scenario("GPU过载" / "相机故障" / "正常")
  ├── 读操作: get_jetson_status() → 从状态字典读
  ├── 写操作: restart_vision_service() → 修改状态字典
  └── 验证: verify() → 检查状态是否恢复正常

↓ Jetson 到手后升级为 ↓

RecordReplaySimulator（生产测试）
  ├── 录制: 录制真实 Jetson 运行数据（时序）
  ├── 回放: 按时间序列回放真实数据
  └── 故障注入: 在回放数据上叠加异常
```

### 故障注入方法（Jetson 到手后）

| 故障类型 | 注入方法 |
|---------|---------|
| GPU 过载 | 额外推理任务 / stress-ng 压 GPU |
| 相机断连 | 拔 USB / kill 相机进程 |
| 磁盘 IO 高 | dd 写大文件 |
| 内存不足 | 启动吃内存进程 |
| 网络延迟 | tc netem 模拟延迟 |
| 视觉服务崩溃 | kill 推理进程 |

### 仿真 → 真实执行流程

```
Agent 诊断完成 → 提出修复方案
  ↓
仿真环境执行修复 → 检查仿真指标
  ├── 仿真未通过 → Agent 换方案 → 再仿真
  └── 仿真通过 → 展示预期效果
                    ↓
              人工确认
                ├── 拒绝 → 结束
                └── 确认 → 真实设备执行 → 执行后验证
```

### 预设故障场景

```python
SCENARIOS = {
    "正常": {
        "temperature": 45, "gpu_load": 30, "cpu_usage": 25,
        "fps": 30, "inference_latency_ms": 25,
        "camera_connected": True, "camera_fps": 30,
        "ring_buffer_usage": 20, "shm_lock_wait_ms": 1,
        "vision_service_running": True,
    },
    "GPU过载": {
        "temperature": 75, "gpu_load": 85, "cpu_usage": 45,
        "fps": 12, "inference_latency_ms": 180,
        "camera_connected": True, "camera_fps": 30,
        "ring_buffer_usage": 75, "shm_lock_wait_ms": 3,
        "vision_service_running": True,
    },
    "相机故障": {
        "temperature": 42, "gpu_load": 20, "cpu_usage": 15,
        "fps": 0, "inference_latency_ms": 0,
        "camera_connected": False, "camera_fps": 0,
        "ring_buffer_usage": 0, "shm_lock_wait_ms": 0,
        "vision_service_running": True,
    },
    "缓冲区满": {
        "temperature": 55, "gpu_load": 70, "cpu_usage": 60,
        "fps": 5, "inference_latency_ms": 350,
        "camera_connected": True, "camera_fps": 30,
        "ring_buffer_usage": 98, "shm_lock_wait_ms": 25,
        "vision_service_running": True,
    },
    "视觉服务崩溃": {
        "temperature": 40, "gpu_load": 5, "cpu_usage": 10,
        "fps": 0, "inference_latency_ms": 0,
        "camera_connected": True, "camera_fps": 30,
        "ring_buffer_usage": 100, "shm_lock_wait_ms": 0,
        "vision_service_running": False,
    },
}
```

---

## 五、系统架构

### 整体架构

```
┌──────────────────────────────────────────────────────────────┐
│              触发层（双模式）                                   │
│                                                              │
│  模式 1: 用户问答                 模式 2: 告警驱动              │
│  POST /chat                     监控循环 → 阈值检查            │
│  "视觉变慢了"                    gpu_load > 80% → 自动触发     │
└──────────────────────────┬───────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                    Agent Server (FastAPI)                     │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │              Orchestrator Agent (调度大脑)               │  │
│  │  1. 理解问题  2. 路由到专项 Agent  3. 汇总结果          │  │
│  └───┬──────────────┬──────────────┬─────────────────────┘  │
│      │              │              │                         │
│      ▼              ▼              ▼                         │
│  ┌────────┐  ┌────────────┐  ┌────────────┐                │
│  │ Vision │  │  Hardware   │  │  System    │  ← 可插拔      │
│  │ Diag   │  │  Diag      │  │  Diag      │                │
│  │ Agent  │  │  Agent     │  │  Agent     │                │
│  │        │  │            │  │            │                │
│  │ 工具:  │  │ 工具:      │  │ 工具:      │                │
│  │ -视觉  │  │ -相机状态  │  │ -Agent     │                │
│  │  管线  │  │ -日志查询  │  │  Trace     │                │
│  │ -GPU   │  │ -硬件FAQ   │  │ -LLM延迟   │                │
│  │  指标  │  │ -设备状态  │  │ -工具延迟   │                │
│  │ -视觉  │  │            │  │ -系统资源   │                │
│  │  FAQ   │  │            │  │            │                │
│  └───┬────┘  └─────┬──────┘  └─────┬──────┘                │
│      └─────────────┼───────────────┘                         │
│                    ▼                                         │
│  ┌─────────────────────────────────────────────────────┐    │
│  │   Diagnosis Engine (规则引擎 + LLM 报告生成)          │    │
│  └────────────────────────┬────────────────────────────┘    │
│                           ▼                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │   Action Agent (仿真验证 → 人工确认 → 执行 → 验证)    │    │
│  │     Level 1 只读 → 自动                              │    │
│  │     Level 2 可逆 → 仿真 → 确认 → 执行 → 验证          │    │
│  │     Level 3 不可逆 → 只建议                           │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────── 共享基础设施 ────────────────────────────┐    │
│  │ RAG 知识库 │ 请求快照 │ 结构化日志 │ 告警管理器       │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────────────────────┬───────────────────────────────────┘
                           │ HTTP
               ┌───────────┼───────────┐
               ▼                       ▼
┌──────────────────────┐  ┌──────────────────────────────┐
│  JetsonSimulator     │  │  Jetson Runtime API (FastAPI) │
│  (开发阶段)           │  │  (部署阶段)                    │
│  场景切换 + 故障注入  │  │  真实指标 + 视觉管线           │
└──────────────────────┘  └──────────────────────────────┘
```

### 视觉管线集成（你的真实工程经验）

```
你的视觉系统架构:
  相机 → 适配器模式 → 共享内存(环形缓冲区) → 视觉算法 → 结果输出

Agent 可观测的管线指标:
  ├── ring_buffer_usage      环形缓冲区占用率
  ├── ring_buffer_capacity   总容量（帧）
  ├── shm_lock_wait_ms       共享内存锁等待时间
  ├── producer_fps           相机写入速度
  ├── consumer_fps           算法消费速度
  └── frame_drop_reason      丢帧原因（consumer_slow / buffer_full / camera_disconnect）

普通 Agent 的诊断: "FPS 低了，可能是 GPU 问题"
你的 Agent 的诊断: "FPS 低是因为消费者速度跟不上生产者，
                   环形缓冲区占用 85%，共享内存锁等待 12ms，
                   瓶颈在推理速度，不是相机输入"
```

---

## 六、三层递进诊断模型

### 为什么要分层

- 简单问题快速解决（Layer 1 秒级定位）
- 复杂问题逐步深入（Layer 2 → Layer 3）
- eBPF 只在需要时触发（避免不必要的开销）

### Layer 1：应用层诊断（OpenTelemetry Trace）

```
能力: 看到 Agent 每一步做了什么、花了多少时间
能定位: LLM 推理慢 / 工具调用超时 / Agent 步数过多 / RAG 检索慢
不能定位: 为什么 Jetson API 响应慢？ → 需要 Layer 2
```

### Layer 2：系统层诊断（Prometheus Metrics）

```
能力: 看到设备资源状态和趋势
能定位: GPU 过高 / 温度过高 / 内存不足 / 相机丢帧 / 缓冲区满
不能定位: GPU 95% 是推理慢还是别的进程抢了 GPU？ → 需要 Layer 3
```

### Layer 3：内核层诊断（eBPF）

```
能力: 看到进程在内核里到底在干什么
能定位 (Layer 1+2 定位不了的):
  - "8s = CPU调度350ms + 磁盘IO 3.85s + GPU推理 3.6s + 网络 200ms"
  - "Jetson API 进程被 journald 抢占了 350ms"
  - "vfs_read 每次等 480ms，正常应该 < 1ms"
  - "共享内存 mmap 映射的页面被换出到 swap"
```

### 触发逻辑

```
诊断开始
  ├─ Layer 1 → 能定位? → Yes → 输出报告
  │                       No ↓
  ├─ Layer 2 → 能定位? → Yes → 输出报告
  │                       No ↓
  └─ Layer 3 → 输出精确时间分解
```

---

## 七、功能清单

### 核心功能

| 编号 | 功能 | 描述 | 对应周 | 状态 |
|------|------|------|--------|------|
| F1 | 自然语言问诊 | 用户用中文描述问题 | Week 1 | ✅ 完成 |
| F2 | 自动工具选择 | Agent 自主决定调哪些工具 | Week 2 | ✅ 完成 |
| F3 | ReAct 多步推理 | 循环调工具直到获得足够信息 | Week 2 | ✅ 完成 |
| F4 | 请求快照 | 每次诊断完整记录 Token、工具调用、结果 | Week 2 | ✅ 完成 |
| F5 | 工具调用去重 | LLM 返回重复 tool_call 时自动去重 | Week 2 | ✅ 完成 |
| F6 | 推理护栏 | 步数限制 + Token 滑动窗口 + 超时熔断 | Week 3-4 | ✅ 完成 |
| F7 | ReActAgent 类 | 可复用的 Agent 引擎，支持实例化多个 Agent | Week 3 | ✅ 完成 |
| F8 | 仿真环境 | JetsonSimulator + 5 个场景切换 | Week 3 | ✅ 完成 |
| F9 | FastAPI 服务化 | 9 个 HTTP 端点 + Swagger 文档 | Week 4 | ✅ 完成 |
| F10 | 告警驱动模式 | 6 条规则 + 去抖动 + 告警聚合 + 自动触发 Agent | Week 4 | ✅ 完成 |
| F11 | 多 Agent 编排 | Orchestrator LLM 路由 + 3 个专项 Agent + 结果汇总 | Week 5-6 | ✅ 完成 |
| F12 | Agent 注册机制 | AgentSpec + AgentRegistry 插件化扩展 | Week 6 | ✅ 完成 |
| F13 | RAG 知识库 | Chroma + 故障排查文档 + search_wiki 工具 | Week 7 | 待做 |
| F14 | 规则诊断引擎 | 阈值 + 组合条件自动归因 + 置信度 | Week 8 | 待做 |
| F15 | 全链路 Trace | OpenTelemetry 三级 Span + Jaeger 可视化 | Week 9 | 待做 |
| F16 | Prometheus + Grafana | Agent + 设备指标暴露 + Dashboard | Week 10-11 | 待做 |
| F17 | Action Agent | 仿真沙盒验证 → 确认 → 执行 → 验证闭环 | Week 13 | 待做 |
| F18 | 工具三级分类 | Level 1 自动 / Level 2 确认 / Level 3 只建议 | Week 13 | 待做 |
| F19 | eBPF 系统调用追踪 | 追踪目标进程的 syscall 和耗时 | Week 15 | 待做 |
| F20 | eBPF 延迟分解 | 慢请求精确拆解为各层耗时 | Week 15 | 待做 |

### 后续迭代功能（5 个月后）

| 编号 | 功能 |
|------|------|
| F21 | 录制回放仿真（真实 Jetson 数据） |
| F22 | eBPF 追踪 GPU 驱动层 |
| F23 | 历史诊断自动沉淀到知识库 |
| F24 | Prometheus AlertManager 告警联动 |
| F25 | 更多专项 Agent (网络诊断 / 存储诊断) |
| F26 | 自动修复策略 (高置信度 → 免确认) |

### 5 个核心 Demo 场景

**Demo 1：GPU 过载 → 三层递进诊断**

```
触发: 用户问"视觉变慢了" 或 告警 gpu_load > 80%

Agent 调工具:
  get_vision_metrics → latency=180ms, fps=12
  get_jetson_status(gpu_load) → 85%
  get_jetson_status(temperature) → 75°C
  get_vision_pipeline_status → ring_buffer_usage=75%

诊断引擎:
  规则匹配: gpu_high + temp_high → gpu_bottleneck + thermal_risk
  置信度: 0.85

结论: "GPU 负载 85%，温度 75°C 接近降频阈值。
       视觉管线环形缓冲区 75%，消费速度跟不上生产速度。
       建议降低推理分辨率。"
```

**Demo 2：相机故障 → 硬件诊断**

```
触发: 告警 camera_fps = 0

Agent 调工具:
  check_camera_status → connected=False, fps=0
  get_vision_metrics → fps=0
  get_jetson_status → CPU/GPU 正常
  query_logs("camera") → "camera timeout on /dev/video0"

诊断引擎:
  规则匹配: camera_fps=0 + camera_disconnected → camera_failure
  置信度: 0.92

结论: "Jetson 资源正常，问题在相机采集链路。
       日志显示 camera timeout。检查相机连接/驱动/USB带宽。"
```

**Demo 3：缓冲区满 → 视觉管线深度诊断**

```
触发: 告警 fps < 15

Agent 调工具:
  get_vision_pipeline_status → ring_buffer_usage=98%, shm_lock_wait=25ms
  get_vision_metrics → fps=5, latency=350ms
  get_jetson_status(gpu_load) → 70%

诊断引擎:
  规则匹配: buffer_full + lock_wait_high → pipeline_bottleneck
  置信度: 0.88

结论: "环形缓冲区占用 98%，共享内存锁等待 25ms（正常 < 5ms）。
       相机以 30fps 写入但算法只能 5fps 消费。
       瓶颈在推理速度，不是相机输入，也不是 GPU 过载（70%）。
       建议: 降低输入帧率或优化推理模型。"
```

**Demo 4：Agent 响应慢 → 内核级定位**

```
触发: 用户问"Agent 为什么这么慢"

Layer 1 (Trace):
  llm_latency: 1.2s（正常）
  tool_call 超时: 8.5s（异常）

Layer 2 (Metrics):
  CPU: 85%（偏高）, 磁盘 IO 高（可疑）

Layer 3 (eBPF):
  cpu_scheduling: 350ms (4%)
  disk_io: 3850ms (45%)  ← 主因
  gpu_inference: 3600ms (42%)
  network: 700ms (9%)

结论: "8.5s 中 45% 是磁盘 IO 阻塞。
       有其他进程在写大文件。建议检查磁盘 IO 占用。"
```

**Demo 5：诊断 → 仿真 → 修复 → 验证闭环**

```
诊断: vision_service_crashed, confidence=0.93

Agent 提出: restart_vision_service (Level 2)

仿真执行:
  仿真器状态变化: inference_latency 从 0 → 25ms, fps 从 0 → 30
  仿真验证: 通过

用户确认 → 真实执行 → 执行后验证:
  真实 latency: 25ms（恢复正常）

输出: {
  "diagnosis": "vision_service_crashed",
  "action": "restart_vision_service",
  "simulation_result": "passed",
  "real_result": "success",
  "verification": "latency 25ms, 恢复正常"
}
```

---

## 八、完整技术栈

### 必须使用的技术（按学习顺序）

#### Phase 1 — Agent 引擎 + 仿真 (Week 1-4)

| 技术 | 用途 |
|------|------|
| Python 3.10+ | 主语言 |
| openai (compatible mode) | 调 LLM API（Qwen/DeepSeek） |
| pydantic | 数据模型定义 |
| python-dotenv | 环境变量管理 |
| FastAPI + uvicorn | Agent HTTP 服务 |

#### Phase 2 — 多 Agent + 可观测性 (Week 5-12)

| 技术 | 用途 |
|------|------|
| opentelemetry-sdk | 链路追踪 |
| opentelemetry-exporter-otlp | Trace 导出到 Jaeger |
| prometheus_client | Python 指标暴露 |
| Docker + Docker Compose | 运行 Jaeger / Prometheus / Grafana |
| chromadb | RAG 向量数据库 |
| httpx | 异步 HTTP 调 Jetson API |
| jtop / tegrastats | Jetson 指标采集（到手后） |

#### Phase 3 — eBPF (Week 15)

| 技术 | 用途 |
|------|------|
| BCC (Python) | eBPF 脚本编写 |
| bpftrace | 快速原型验证 |

### 明确不需要的技术

| 技术 | 为什么不需要 |
|------|-------------|
| LangChain / LangGraph | 手写更深，框架封装了你需要观测的内部状态 |
| MCP | 单系统内部调用，HTTP + JSON 即可 |
| 模型训练 / 微调 | Prompt + RAG 完全够用 |
| CUDA 编程 | 用 eBPF 从外部追踪，不写核函数 |
| Kubernetes | Docker Compose 够用 |
| React / Vue | Grafana 够用 |

---

## 九、需要理解的原理

### Agent 核心原理

| 原理 | 一句话 |
|------|--------|
| LLM 是无状态的 | 每次调 API 都要发完整对话历史 |
| LLM 不执行代码 | 它只输出 JSON 告诉你"我想调什么"，执行的是你的代码 |
| messages 列表是 LLM 的记忆 | 你 append 什么它就看到什么 |
| role 区分信息来源 | user=人，assistant=AI，tool=工具返回 |
| ReAct 就是 while 循环 | 每轮让 LLM 决定继续调工具还是给最终答案 |
| Prompt 是 Agent 的操作手册 | 写得越具体，LLM 行为越可控 |

### 可观测性原理

| 知识点 | 核心问题 |
|--------|---------|
| Trace / Span | TraceID 怎么跨服务传播？父子 Span 怎么形成瀑布图？ |
| Metrics | Counter / Gauge / Histogram 各适合什么场景？ |
| Agent Trace 的难点 | 调用链是动态的，每次不一样，Span 要动态创建 |
| 请求快照 | 为什么 LLM 系统需要快照而传统系统不需要？（不确定性） |

### eBPF 原理

| 知识点 | 核心问题 |
|--------|---------|
| 内核态 vs 用户态 | 为什么普通监控只能看用户态？ |
| kprobe / tracepoint | 怎么在内核函数上挂探针？ |
| CPU 调度 | sched_switch 事件代表什么？ |
| 文件 IO | vfs_read/vfs_write 的延迟来自哪里？ |
| 共享内存 | mmap 在内核层怎么工作？（关联你的视觉管线） |

---

## 十、工业级五大约束

| 约束 | 实现方式 |
|------|---------|
| 可追踪 | 每次诊断有 trace_id，Jaeger 可回放 |
| 可复现 | 请求快照保存完整 messages + 工具结果 |
| 可回滚 | 工具三级分类，Level 2 需确认，仿真先验证 |
| 有护栏 | 步数限制 8 步 / Token 预算 / 超时 10s / 重试 2 次 |
| 可审计 | 所有操作记录到 JSONL：谁确认、何时执行、结果 |

---

## 十一、5 个月详细路线图

### Month 1：Agent 引擎 + 仿真环境（Week 1-4）

```
Week 1-2: LLM API + Function Calling（✅ 已完成）
  已做: 多轮对话, 2 个 mock 工具, ReAct 循环, 请求快照, 工具去重
  产出: hello_world.py (单文件脚本)

Week 3: ReActAgent 类 + 仿真器 + 更多工具（✅ 已完成）
  已做:
    - ReActAgent 类 (agent_core/react_agent.py)
    - JetsonSimulator + 5 个场景 (simulator/jetson_sim.py)
    - 6 个工具 (tools/all_tools.py)
    - AgentConfig 统一配置 (agent_core/config.py)
    - 请求快照管理器 (agent_core/snapshot.py)
  产出: agent_core/, simulator/, tools/ 三个模块
  验收: ✅ 5 个场景切换正常, Agent 给出不同诊断

Week 4: FastAPI 服务化 + 告警监控 + 上下文管理（✅ 已完成）
  已做:
    - FastAPI 9 个 HTTP 端点 + Swagger 文档 (server.py)
    - 告警驱动: 6 条规则 + 去抖动 + 告警聚合 + 后台监控循环 (monitor/)
    - Token 滑动窗口 + 上下文压缩 (agent_core/context_manager.py)
    - 请求超时机制 + on_step 回调
    - run() 返回结构化 dict (answer/steps/stop_reason/compression_count)
  产出: server.py, monitor/alert_rules.py, monitor/alert_monitor.py, agent_core/context_manager.py
  验收: ✅ curl 调通, ✅ 告警自动触发诊断, ✅ 上下文压缩正常工作
```

### Month 2：多 Agent 编排 + 诊断引擎（Week 5-8）

```
Week 5-6: 多 Agent 编排 + 注册机制（✅ 已完成）
  已做:
    - AgentSpec 元信息定义 + AgentRegistry 插件化注册 (agents/registry.py)
    - 3 个专项 Agent: vision_diag / hardware_diag / system_diag (agents/specialists.py)
    - Orchestrator: LLM 路由决策 → 派发专项 Agent → 汇总报告 (agents/orchestrator.py)
    - 工具按领域分组: VISION_TOOLS / HARDWARE_TOOLS / SYSTEM_TOOLS
    - server.py 新增: /api/diagnose (Orchestrator), /api/diagnose/simple (单Agent), /api/agents
    - main.py 支持 --simple (单Agent) / 默认 Orchestrator 两种模式
  产出: agents/ 模块 (4 个文件)
  验收: ✅ 11 个端点全部测试通过, ✅ 3 个 Agent 注册成功

Week 7: RAG 知识库
  做:
    - 写 5 篇 Wiki 文档（Jetson 故障排查、视觉管线架构、相机 FAQ 等）
    - Chroma 向量化存储
    - search_wiki 工具注册
  产出: rag/wiki_search.py, docs/*.md
  验收: 问"camera timeout 怎么办", Agent 从你的文档里找到答案

Week 8: 诊断引擎
  做:
    - 8-10 条诊断规则
    - 置信度计算
    - LLM 报告生成
  产出: diagnosis/engine.py, diagnosis/rules.py
  验收: 跑通 5 个 Demo 场景, 规则引擎输出正确根因
```

### Month 3：可观测性层（Week 9-12）

```
Week 9: OpenTelemetry Trace
  做: Orchestrator → SubAgent → Tool 三级 Span
  产出: observability/tracer.py
  验收: Jaeger 瀑布图展示完整诊断链路

Week 10: Prometheus + Grafana
  做: Agent 指标暴露 + Dashboard
  指标: agent_request_total, agent_step_count, agent_token_usage,
        agent_llm_latency, agent_tool_latency, agent_tool_error_count,
        agent_max_steps_hit_count, agent_cost_per_request
  产出: observability/metrics.py, dashboards/
  验收: Grafana 实时反映 Agent 行为

Week 11-12: Jetson 指标 + Docker Compose
  做: Jetson Prometheus Exporter + 一键部署
  产出: docker-compose.yml, jetson_runtime/prometheus_metrics.py
  验收: docker compose up 跑通全部服务
```

### Month 4：Action Agent + eBPF（Week 13-16）

```
Week 13: Action Agent + 仿真验证
  做:
    - 工具注册增加 level 字段
    - 仿真环境执行 + 验证
    - 人工确认 + 执行 + 执行后验证
    - 操作审计日志
  产出: agents/action_agent.py
  验收: 诊断 → 仿真通过 → 确认 → 执行 → 验证恢复

Week 14: 5 个 Demo 端到端打磨
  做: 所有 Demo 场景可录制, Trace + Metrics 全程可见
  产出: P50/P90/P99 延迟数据, Token 消耗统计

Week 15: eBPF 内核级诊断
  学: BCC Python, kprobe/tracepoint
  做:
    脚本 1: trace_syscalls.py (追踪 syscall 和耗时)
    脚本 2: trace_latency.py (请求延迟分解)
  验收: eBPF 看到 Trace 看不到的东西

Week 16: 文档 + 录制
  做: README, 架构图, Demo 视频
  验收: GitHub 仓库完整可展示
```

### Month 5：面试准备（Week 17-20）

```
Week 17-18: 算法 + 系统设计刷题
Week 19: 项目表达练习 (模拟面试, 15 分钟讲项目)
Week 20: 秋招投递
```

---

## 十二、项目目录结构

```
edge-agentops-demo/
│
├── agent_core/                     # ✅ 通用 Agent 引擎（与业务无关）
│   ├── __init__.py
│   ├── config.py                   # ✅ 配置管理 (API/模型/护栏/超时)
│   ├── react_agent.py              # ✅ ReAct 循环引擎 (可复用的类)
│   ├── context_manager.py          # ✅ Token 滑动窗口 + 上下文压缩
│   └── snapshot.py                 # ✅ 请求快照管理器
│
├── server.py                       # ✅ FastAPI HTTP 服务 (9 个端点)
├── main.py                         # ✅ CLI 入口
│
├── simulator/                      # ✅ 仿真环境
│   ├── __init__.py
│   └── jetson_sim.py               # ✅ JetsonSimulator + 5 个预设场景
│
├── monitor/                        # ✅ 告警监控
│   ├── __init__.py
│   ├── alert_monitor.py            # ✅ 后台监控循环 + 去抖动 + 告警聚合
│   └── alert_rules.py              # ✅ 6 条告警规则定义
│
├── tools/                          # ✅ 工具层（6 个工具）
│   ├── __init__.py
│   └── all_tools.py                # ✅ 工具定义 + 注册表
│
├── agents/                         # 🔲 多 Agent 编排层（Week 5-6）
│   ├── orchestrator.py             # 🔲 Orchestrator 调度大脑
│   ├── registry.py                 # 🔲 Agent 注册表
│   ├── vision_agent.py             # 🔲 视觉诊断 Agent
│   ├── hardware_agent.py           # 🔲 硬件诊断 Agent
│   ├── system_agent.py             # 🔲 系统诊断 Agent
│   └── action_agent.py             # 🔲 执行修复 Agent（含仿真沙盒）
│
├── diagnosis/                      # 🔲 诊断引擎（Week 8）
│   ├── engine.py                   # 🔲 规则引擎 + 确定性归因
│   ├── rules.py                    # 🔲 诊断规则定义
│   └── report.py                   # 🔲 LLM 报告生成
│
├── observability/                  # 🔲 可观测性（Week 9-11）
│   ├── tracer.py                   # 🔲 OpenTelemetry Span
│   ├── metrics.py                  # 🔲 Prometheus 指标
│   └── middleware.py               # 🔲 FastAPI 中间件
│
├── rag/                            # 🔲 知识库（Week 7）
│   ├── wiki_search.py              # 🔲 Chroma 检索
│   └── indexer.py                  # 🔲 文档索引
│
├── ebpf/                           # 🔲 eBPF 内核级诊断（Week 15）
│   ├── trace_syscalls.py           # 🔲 系统调用追踪
│   └── trace_latency.py            # 🔲 请求延迟分解
│
├── jetson_runtime/                 # 🔲 Jetson 端（部署阶段用）
│
├── docs/                           # 🔲 Wiki 知识库文档（Week 7）
│
├── snapshots/                      # ✅ 请求快照（运行时自动生成）
├── requirements.txt                # ✅ 依赖清单
├── PROJECT_PLAN.md                 # ✅ 本文档
└── .env                            # API Key 配置
```

---

## 十三、简历写法

```
Edge AgentOps 工业诊断平台
- 手写 ReAct 引擎，不依赖 LangChain，支持工具注册、三级分类、Token 滑动窗口
- 多 Agent 编排架构，Orchestrator 按领域路由，支持插件化扩展
- 双触发模式：交互式问答 + 告警驱动自动诊断，适配工业无人值守场景
- 基于 OpenTelemetry + Jaeger 实现 Agent 全链路追踪，每步决策可复现可审计
- 三层递进诊断：Trace 定位慢步骤 → Prometheus 定位资源瓶颈 → eBPF 内核级延迟拆解
- 修复方案先在仿真环境验证，通过后人工确认执行，执行后自动验证恢复
```

---

## 十四、面试高频问题

| 问题 | 回答要点 |
|------|---------|
| Agent 怎么实现？ | 手写 ReAct while 循环 + 工具注册表 + JSON Schema，不用框架 |
| 为什么不用 LangChain？ | 需要观测内部状态，框架封装了；手写可以在每个节点插 Trace Span |
| 为什么不微调模型？ | Prompt + RAG 够用，微调成本高迭代慢 |
| 需要 MCP 吗？ | 单系统内部调用，HTTP + JSON 即可，MCP 是多生态互通用的 |
| 多 Agent 怎么编排？ | Orchestrator LLM 分类路由，Agent 注册表，结果汇总 |
| Agent 死循环怎么办？ | 步数限制 + Token 预算 + 工具去重 + 降级兜底 |
| 怎么定位性能瓶颈？ | 三层递进: Trace → Metrics → eBPF |
| 和普通监控区别？ | Prometheus 看到"磁盘 IO 高"，eBPF 告诉你"vfs_read 每次等 480ms" |
| 工业 Agent 和 Cursor 区别？ | Cursor 追求自主性，工业追求可控性。工具分级、人工确认、审计日志 |
| 错误怎么排查？ | trace_id + Jaeger 瀑布图 + 请求快照，5 分钟定位 |
| 怎么保证诊断准确？ | 规则引擎确定性归因，LLM 只写报告。不让 LLM 自由判断 |
| 怎么测试 Agent？ | 仿真环境 + 故障注入（Chaos Engineering），场景切换验证诊断准确率 |
| 告警怎么做？ | 监控循环 + 阈值规则 + 自动触发 Agent 诊断 + 推送通知 |
