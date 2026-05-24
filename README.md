# DeepSeek Cache Optimizer

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.10+-blue.svg" alt="Python">
  <img src="https://img.shields.io/badge/License-MIT-green.svg" alt="License">
  <img src="https://img.shields.io/badge/Hermes-%3E%3D0.14.0-orange.svg" alt="Hermes">
  <img src="https://img.shields.io/badge/version-1.1.0-blue.svg" alt="Version">
</p>

DeepSeek/MiMo prefix-cache 优化器 + Reasonix 三层优化机制。最大化 DeepSeek API 的 prefix-cache 缓存命中率，降低 token 消耗和延迟。

## 三大支柱

### Pillar 1 — 缓存优先循环

| 机制 | 钩子 | 说明 |
|------|------|------|
| 工具排序 | `pre_llm_call` | tools 按 name 字典序排列，字节级稳定 |
| 前缀保护压缩 | `pre_llm_call` | 消息 >14 条时压缩中间历史，保留 system + 最近 4 轮 |

### Pillar 2 — 工具调用修复

| 机制 | 钩子 | 说明 |
|------|------|------|
| Call-Storm 检测 | `post_tool_call` | 相同 (tool, args) 在 5 次窗口内重复 ≥3 次 → 告警 |
| 失败信号升级 | `pre_llm_call` | 连续 3 次工具失败 → 自动升级模型 |

升级映射：
- mimo-v2.5 → mimo-v2.5-pro
- deepseek-v4-flash → deepseek-v4-pro
- gpt-4o-mini → gpt-4o

### Pillar 3 — 成本控制

| 机制 | 钩子 | 说明 |
|------|------|------|
| 轮末自动压缩 | `transform_tool_result` | 工具结果 >3000 token → 截断保留头尾 60%+20% |

## 安装

### 前置条件

- Python 3.10+
- [Hermes Agent](https://github.com/weksbwrx62862/hermes) >= 0.14.0

### 从源码安装

```bash
git clone https://github.com/weksbwrx62862/deepseek-cache-optimizer.git
cd deepseek-cache-optimizer
pip install -e .
```

## 使用

自动启用，无需额外配置。在 Hermes 中添加插件路径即可：

```yaml
# hermes_config.yaml
plugins:
  - name: deepseek-cache-optimizer
    path: ./deepseek-cache-optimizer
```

## 可调参数

常量可在 `__init__.py` 顶部调整：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `TOOL_RESULT_CAP_TOKENS` | 3000 | 工具结果压缩阈值 |
| `FAILURE_ESCALATION_THRESHOLD` | 3 | 失败升级阈值 |
| `STORM_WINDOW` | 5 | 风暴检测窗口大小 |
| `STORM_THRESHOLD` | 3 | 风暴判定阈值 |

## 统计

- 缓存统计文件：`cache_stats.json`
- 日志输出：每 5 次 API 请求或 30 秒自动保存

## 提供的钩子

| 钩子 | 说明 |
|------|------|
| `pre_llm_call` | 工具排序 + 前缀压缩 + 失败升级 |
| `post_tool_call` | Call-Storm 检测 |
| `transform_tool_result` | 轮末压缩 |
| `post_api_request` | 缓存统计记录 |

## 项目结构

```
deepseek-cache-optimizer/
├── plugin.yaml     # 插件声明
├── __init__.py     # 主入口 + 三大 Pillar 实现
├── README.md       # 本文档
```

## 开发

```bash
git clone https://github.com/weksbwrx62862/deepseek-cache-optimizer.git
cd deepseek-cache-optimizer
pip install -e .
# 通过 Hermes 运行时测试
```

## License

MIT