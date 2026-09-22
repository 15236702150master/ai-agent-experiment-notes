# 实验 2-1：本地 LLM 服务部署与工具调用

## 实验目标

验证一个本地小型语言模型能否完成完整的工具调用 Agent 闭环，并观察固定提示词前缀对首 token 延迟（TTFT）的影响。

## 实验条件

### 运行环境

- 操作系统：Windows 原生环境
- 推理服务：Ollama `0.34.2`
- 服务地址：`http://localhost:11434`
- 模型：`qwen3:0.6b`
- 模型格式：GGUF，`Q4_K_M`
- 模型参数规模：约 `751.63M`
- 上下文长度：`40960`
- tokenizer：Qwen3-0.6B，与本地模型配套的 chat template
- 推理参数：`temperature=0`、`num_predict=512`、`seed=21`
- 请求模式：Ollama 原生 `/api/generate`，`raw=true`，流式输出

### 工具与数据

- `get_current_time`：根据 IANA 时区返回当前时间
- `get_current_temperature`：通过 Open-Meteo 获取温度和天气
- 测试地点：Vancouver
- 除天气只读查询外，模型推理和工具编排均在本机完成
- 实验不保存或发送凭据

## 实验要求

1. 服务端必须报告目标模型及非空、不可变的模型 digest。
2. 渲染后的 prompt 必须保留 chat template 特殊 token 和工具 schema。
3. 第一轮必须恰好生成两个工具调用：时间和天气。
4. 两个工具必须并行执行，并保存可审计的返回结果。
5. 第二轮必须消费两个工具结果，并在没有新工具调用的情况下结束。
6. 保留流式响应块、prompt、耗时、token 数、服务端 duration 和哈希。
7. 完成 5 组匹配的缓存命中 / 未命中 TTFT 对照。

## 怎么才算完成实验

以下条件全部满足才算正式完成：

- `manifest.json` 中 `official_complete` 为 `true`。
- 模型 digest 已记录且本地模型校验通过。
- 工具调用验收项全部为 `true`：两个工具准确、参数正确、并行执行、结果有效、第二轮终止。
- 5 组缓存命中 / 未命中样本均已保存；无论命中是否更快，都报告实际分布。
- 原始流、请求 prompt、token 统计、服务端耗时和哈希可在 `evidence.json` 中复核。
- 凭据扫描通过，`credential_scan_findings` 为空。

## 实验结果

### 完成状态

- 正式状态：`official_complete = true`
- 工具调用：通过
- 缓存对照：5 组完整
- 凭据扫描：通过，无发现项
- 推理成本：`0 USD`（本地推理）

### 工具调用结果

第一轮生成了两个调用：

```text
get_current_time(timezone="America/Vancouver")
get_current_temperature(location="Vancouver", unit="celsius")
```

两个工具使用 `ThreadPoolExecutor` 并行执行：

| 工具 | 耗时 |
|---|---:|
| 当前时间 | `0.027 s` |
| 当前天气 | `2.416 s` |
| 并行总耗时 | `2.417 s` |

第二轮模型消费工具结果后停止调用工具，输出了温哥华时间、温度、天气和湿度。

### 性能结果

| 指标 | 结果 |
|---|---:|
| 工具场景平均解码速度 | `57.92 tokens/s` |
| 第二轮解码速度 | `55.45 tokens/s` |
| 缓存命中平均 TTFT | `2.46 s` |
| 缓存未命中平均 TTFT | `9.40 s` |
| 未命中 / 命中平均 TTFT | `3.82x` |
| 命中更快的配对数 | `5/5` |

### 可复核摘要

```json
{
  "experiment_id": "2-1",
  "official_complete": true,
  "model": "qwen3:0.6b",
  "ollama_version": "0.34.2",
  "decode_tokens_per_second": 57.91995691696848,
  "cache_hit_mean_ttft_s": 2.4602889800000414,
  "cache_miss_mean_ttft_s": 9.400364360000093,
  "cache_miss_over_hit": 3.820837485521695,
  "matched_pairs": 5,
  "credential_scan_passed": true
}
```

## 结果解读

### 直接结论

本地 Qwen3-0.6B 已完成“理解请求 → 选择多个工具 → 并行执行 → 注入工具结果 → 生成最终回答”的完整 Agent 闭环。这个结果验证的是系统链路，而不只是模型能否回答一个问题。

缓存对照显示，当前服务在提示词前缀完全一致时能显著降低 TTFT。5 组实验中命中全部更快，平均等待时间从 `9.40 s` 降至 `2.46 s`。

### 性能如何理解

本机解码速度约 `57.92 tokens/s`，低于书中 M2 级别机器的 `100 tokens/s` 参考观察值。这不是工具调用失败，而是当前硬件、操作系统和推理后端组合的实测性能。实验协议要求报告实测值，不能把 `100 tokens/s` 当作所有机器的硬性通过线。

天气查询占用了并行工具阶段的主要时间，因为它需要访问 Open-Meteo；时间工具本身几乎立即返回。并行执行使总耗时接近较慢工具的耗时，而不是两个工具耗时的简单相加。

### 局限

- 只有 5 组缓存配对，适合验证现象，不足以建立跨机器的统计基准。
- 结果对应 `qwen3:0.6b` 和本机 Ollama 配置，不能直接代表更大模型或 vLLM 的表现。
- TTFT 会受模型是否已加载、系统负载和天气网络请求影响；复现实验时应保留硬件和服务版本。

## 复现步骤

在源项目目录执行：

```powershell
cd D:\Data\01_Projects\2026\ai_agent\chapter2\local_llm_serving
D:\Data\01_Projects\2026\ai_agent\.venv\Scripts\python.exe run_experiment.py `
  --output runs\exp2-1-qwen3-0.6b-<timestamp> `
  --tokenizer tokenizer-qwen3-0.6b
```

完成后检查输出目录中的 `manifest.json` 和 `evidence.json`。
