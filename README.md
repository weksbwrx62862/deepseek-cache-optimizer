<h1 align="center">DeepSeek Cache Optimizer</h1>
<p align="center"><strong>DeepSeek/MiMo prefix-cache 优化器 — 工具排序 + 前缀保护 + Storm 检测 + 轮末压缩</strong></p>

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.10+-blue.svg" alt="Python">
  <img src="https://img.shields.io/badge/License-MIT-green.svg" alt="License">
  <img src="https://img.shields.io/badge/Hermes-%3E%3D0.14.0-orange.svg" alt="Hermes">
  <img src="https://img.shields.io/badge/version-1.1.0-blue.svg" alt="Version">
  <img src="https://img.shields.io/badge/Stars-★-yellow.svg" alt="Stars">
  <img src="https://img.shields.io/badge/Last_Commit-2026--05-blue.svg" alt="Last Commit">
</p>

---

DeepSeek Cache Optimizer 是面向 DeepSeek / MiMo API 的 prefix-cache 优化插件，基于 Reasonix 三层优化机制，通过工具排序、前缀保护压缩、Call-Storm 检测与轮末自动压缩，最大化缓存命中率、降低 token 消耗与延迟。作为 Hermes Agent 插件，安装即生效，无需额外配置。

## 功能矩阵

| 支柱 | 机制 | 钩子 | 核心效果 |
|------|------|------|----------|
| **Pillar 1 — 缓存优先** | 工具排序 | `pre_llm_call` | tools 按 name 字典序排列，字节级稳定前缀 |
| **Pillar 1 — 缓存优先** | 前缀保护压缩 | `pre_llm_call` | 消息 >14 条时压缩中间历史，保留 system + 最近 4 轮 |
| **Pillar 2 — 调用修复** | Call-Storm 检测 | `post_tool_call` | 相同 (tool, args) 在 5 次窗口内重复 ≥3 次 → 告警 |
| **Pillar 2 — 调用修复** | 失败信号升级 | `pre_llm_call` | 连续 3 次工具失败 → 自动升级模型 |
| **Pillar 3 — 成本控制** | 轮末自动压缩 | `transform_tool_result` | 工具结果 >3000 token → 截断保留头 60% + 尾 20% |
| — | 缓存统计收集 | `post_api_request` | 记录命中/未命中 token，自动持久化 |

## 架构图

```
                         ┌──────────────────────────────────┐
                         │     DeepSeek Cache Optimizer      │
                         │           v1.1.0                  │
                         └──────────────┬───────────────────┘
                                        │
               ┌────────────────────────┼────────────────────────┐
               │                        │                        │
    ┌──────────▼──────────┐  ┌──────────▼──────────┐  ┌──────────▼──────────┐
    │   Pillar 1          │  │   Pillar 2          │  │   Pillar 3          │
    │   缓存优先循环       │  │   工具调用修复       │  │   成本控制           │
    ├─────────────────────┤  ├─────────────────────┤  ├─────────────────────┤
    │                     │  │                     │  │                     │
    │  ┌───────────────┐  │  │  ┌───────────────┐  │  │  ┌───────────────┐  │
    │  │  工具排序      │  │  │  │ Call-Storm    │  │  │  │ 轮末自动压缩   │  │
    │  │  (name 字典序) │  │  │  │ 检测          │  │  │  │ (>3000 token) │  │
    │  └───────┬───────┘  │  │  └───────┬───────┘  │  │  └───────┬───────┘  │
    │          │          │  │          │          │  │          │          │
    │  ┌───────▼───────┐  │  │  ┌───────▼───────┐  │  │  ┌───────▼───────┐  │
    │  │ 前缀保护压缩   │  │  │  │ 失败信号升级   │  │  │  │ 头60%+尾20%  │  │
    │  │ (>14 条消息)  │  │  │  │ (3次→升级模型) │  │  │  │ 中间省略标记   │  │
    │  └───────────────┘  │  │  └───────────────┘  │  │  └───────────────┘  │
    └─────────────────────┘  └─────────────────────┘  └─────────────────────┘
               │                        │                        │
               ▼                        ▼                        ▼
         pre_llm_call            post_tool_call         transform_tool_result
         post_api_request        pre_llm_call
```

## 快速开始

### 前置条件

- Python 3.10+
- [Hermes Agent](https://github.com/weksbwrx62862/hermes) >= 0.14.0

### 安装

```bash
git clone https://github.com/weksbwrx62862/deepseek-cache-optimizer.git
cd deepseek-cache-optimizer
pip install -e .
```

### 最小示例

在 Hermes 配置中添加插件路径即可自动启用：

```yaml
# hermes_config.yaml
plugins:
  - name: deepseek-cache-optimizer
    path: ./deepseek-cache-optimizer
```

无需额外配置，所有优化机制开箱即用。

## 核心功能详解

### Pillar 1 — 缓存优先循环

prefix-cache 的核心原理是：请求前缀越稳定，缓存命中率越高。Pillar 1 从两个维度保证前缀稳定性：

**工具排序**：在 `pre_llm_call` 钩子中，将 `tools` 数组按 `function.name` 字典序排列。这确保无论业务逻辑如何调整工具注册顺序，发送给 API 的 tools 始终保持字节级一致，最大化 prefix-cache 命中。

**前缀保护压缩**：当消息数超过 14 条时，自动压缩中间历史消息。保留策略：
- 保留所有 `system` 消息（不可变前缀）
- 保留前 2 条非 system 消息（对话开头）
- 保留最近 4 轮（8 条消息）
- 中间部分压缩为摘要（每条取前 150 字符预览）

### Pillar 2 — 工具调用修复

当 Agent 陷入重复调用或持续失败时，Pillar 2 提供自动修复机制：

**Call-Storm 检测**：在 `post_tool_call` 钩子中，使用滑动窗口（默认 5 次）检测相同 `(tool_name, args_hash)` 的重复调用。当重复次数 ≥3 时触发告警日志，标记 storm 状态供后续决策使用。

**失败信号升级**：在 `pre_llm_call` 钩子中，当连续工具失败次数达到阈值（默认 3 次）时，自动将当前模型升级为更强版本。升级映射：

| 源模型 | 升级目标 |
|--------|----------|
| mimo-v2.5 | mimo-v2.5-pro |
| mimo-v2 / mimo-v2-pro | mimo-v2.5-pro |
| deepseek-v4-flash | deepseek-v4-pro |
| gpt-4o-mini | gpt-4o |
| claude-3-haiku | claude-3-sonnet |

### Pillar 3 — 成本控制

**轮末自动压缩**：在 `transform_tool_result` 钩子中，当工具返回结果超过阈值（默认 3000 token / ~7500 字符）时，自动截断：
- 保留头部 60%（通常包含关键信息）
- 保留尾部 20%（通常包含结论/状态）
- 中间插入省略标记，标注省略字符数和 token 数

单次压缩可减少 50%+ 的 token 消耗，需要完整结果时可重新调用工具。

### 缓存统计

通过 `post_api_request` 钩子自动收集每次 API 请求的缓存命中数据：
- 支持 DeepSeek (`prompt_cache_hit_tokens`)、Anthropic (`cache_read_input_tokens`)、OpenAI (`cached_tokens`) 等多种字段格式
- 统计数据自动持久化至 `~/.hermes/deepseek_cache_stats.json`
- 每 5 次请求或 30 秒自动保存

## 技术栈

| 类别 | 技术 |
|------|------|
| 语言 | Python 3.10+ |
| 插件框架 | Hermes Agent >= 0.14.0 |
| 并发安全 | threading.Lock |
| 数据持久化 | JSON |
| Token 估算 | 字符比例估算（中英混合 ~2.5 字符/token） |

## 项目结构

```
deepseek-cache-optimizer/
├── plugin.yaml       # 插件声明（名称、版本、钩子、依赖）
├── __init__.py       # 主入口 + 三大 Pillar 实现 + 缓存统计
└── README.md         # 本文档
```

## 开发指南

### 本地开发

```bash
git clone https://github.com/weksbwrx62862/deepseek-cache-optimizer.git
cd deepseek-cache-optimizer
pip install -e .
```

### 可调参数

常量可在 `__init__.py` 顶部调整：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `TOOL_RESULT_CAP_TOKENS` | 3000 | 工具结果压缩阈值（token） |
| `CHARS_PER_TOKEN` | 2.5 | 字符到 token 的估算比例 |
| `FAILURE_ESCALATION_THRESHOLD` | 3 | 连续失败触发升级的阈值 |
| `STORM_WINDOW` | 5 | Call-Storm 检测滑动窗口大小 |
| `STORM_THRESHOLD` | 3 | 窗口内重复调用判定阈值 |

### 添加升级模型映射

在 `__init__.py` 的 `ESCALATION_MODEL_MAP` 字典中添加新的映射即可：

```python
ESCALATION_MODEL_MAP = {
    "your-model-lite": "your-model-pro",
}
```

### 测试

通过 Hermes Agent 运行时测试，观察日志输出验证各 Pillar 行为：

```
DeepSeek Cache Optimizer v1.1 registered: 4 hooks ...
Cache: hit=1200 miss=300 rate=80.0% | 累计: 42请求 78.5%命中 15万token
Call-storm detected: read_file (args_hash=a1b2c3d4) repeated 3 times in 5 calls
⚠️ Failure escalation: deepseek-v4-flash → deepseek-v4-pro (failures=3)
Tool result compacted: search 15000 → 6000 chars (60.0% reduction)
```

## 路线图

- [ ] 支持从 `plugin.yaml` 或环境变量读取配置参数
- [ ] 添加 Call-Storm 自动终止（当前仅告警）
- [ ] 支持自定义压缩策略（可配置头尾保留比例）
- [ ] 添加 Prometheus 指标导出
- [ ] 支持多模型并行对比缓存效率

## 常见问题

**Q: 安装后需要额外配置吗？**

不需要。添加插件路径到 Hermes 配置即可，所有优化机制开箱即用。

**Q: 前缀保护压缩会丢失重要上下文吗？**

压缩策略保留 system 消息、对话开头和最近 4 轮，中间部分仅做摘要。大多数 Agent 场景下，近期上下文已足够决策。

**Q: 失败升级后模型会一直保持升级状态吗？**

不会。升级仅在当次 `pre_llm_call` 生效，失败计数重置后恢复原始模型。

**Q: 缓存统计数据存在哪里？**

默认存储在 `~/.hermes/deepseek_cache_stats.json`，重启后自动加载历史数据。

**Q: 支持哪些 API Provider 的缓存统计？**

支持 DeepSeek、Anthropic、OpenAI 三种字段格式，自动识别并提取缓存命中数据。

## Contributing

欢迎贡献！请遵循以下流程：

1. Fork 本仓库
2. 创建 Feature Branch：`git checkout -b agent/<task-id>-<brief-description>`
3. 提交变更，遵循 Conventional Commits 规范
4. 创建 Pull Request，按模板填写描述

## License

MIT License

## Security

- 本插件不收集或传输任何用户数据到外部服务
- 缓存统计数据仅存储在本地文件系统
- 不在任何日志中暴露 API Key 或 Token
- 如发现安全漏洞，请通过 GitHub Issues 私密报告

## 致谢

- [Reasonix](https://github.com/anthropics/anthropic-cookbook) — 三层优化机制的设计灵感来源
- [Hermes Agent](https://github.com/weksbwrx62862/hermes) — 插件运行框架
- DeepSeek — prefix-cache API 支持

<p align="center"><strong>最大化缓存命中，最小化 token 消耗</strong></p>
