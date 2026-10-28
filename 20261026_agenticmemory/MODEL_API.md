# 面试模型 API 接入说明

面试方提供以下模型接口和 AK，可用于记忆构建或回答。具体接入方式自行决定，题目接口、交付要求与评分规则仍以 `README.md` 和 `INTERFACE.md` 为准。

## 接口与配置

| 项目 | 值 |
|---|---|
| HTTP 方法 | `POST` |
| 完整请求地址 | `https://api.siliconflow.cn/v1/chat/completions` |
| API Base URL | `https://api.siliconflow.cn/v1` |
| 请求模型标识 | `deepseek-ai/DeepSeek-V4-Flash` |
| 内容类型 | `application/json` |
| 认证 | `Authorization: Bearer <AK>` |

私密发放包中的 `.env` 已填写面试方提供的 AK。GitHub 版本仅含 `.env.example`；从仓库获取材料时，将面试方另行发放的 AK 填入自己的 `.env`。实际 AK 不提交到仓库、HTML、日志、`answers.jsonl` 或 `AI_NOTES.md`；提交代码从环境变量读取即可。

配置变量：

```dotenv
SILICONFLOW_API_KEY=YOUR_API_KEY
SILICONFLOW_BASE_URL=https://api.siliconflow.cn/v1
SILICONFLOW_MODEL=deepseek-ai/DeepSeek-V4-Flash
```

`.env` 不会被 Python 或其他程序自动加载。可以使用自己选择的配置加载方式，或先在项目目录执行下列 shell 命令，再启动程序。

```bash
set -a
. ./.env
set +a
```

## 请求示例

以下示例直接使用环境变量中的 AK：

```bash
curl --request POST \
  --url https://api.siliconflow.cn/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${SILICONFLOW_API_KEY}" \
  -d '{
    "model": "deepseek-ai/DeepSeek-V4-Flash",
    "messages": [
      {"role": "system", "content": "你是一个有用的助手"},
      {"role": "user", "content": "你好，请介绍一下你自己"}
    ]
  }'
```

这里的系统指令和问题仅演示协议，实际构建和回答所用的指令、上下文及问题由你的程序组织。

## 返回示例

以下是面试方提供的响应格式示例，不是本次任务的实际调用结果：

```json
{
  "id": "019bdaa55225ef854b320e9b838f77ce",
  "object": "chat.completion",
  "created": 1768899826,
  "model": "deepseek-ai/DeepSeek-V4-Flash",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "你好！...",
        "reasoning_content": "..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 15,
    "completion_tokens": 1540,
    "total_tokens": 1555,
    "completion_tokens_details": {
      "reasoning_tokens": 1190
    },
    "prompt_tokens_details": {
      "cached_tokens": 0
    },
    "prompt_cache_hit_tokens": 0,
    "prompt_cache_miss_tokens": 15
  },
  "system_fingerprint": ""
}
```

最终文本从 `choices[0].message.content` 读取。`reasoning_content` 若存在，不替代最终回答；题目不要求保存或展示模型完整推理过程。`created` 是这次 API 响应的时间字段，不能用来补全题目原始对话的消息时间或事件时间。

该响应是模型服务协议，不是 `run.sh query` 的最终返回格式。你的程序仍需按 `INTERFACE.md` 返回具体 `answer`、召回记忆及原文证据等字段。

## 构建 token 统计对应关系

| 题目 usage 字段 | 服务响应字段 |
|---|---|
| `model` | 本次实际调用的模型标识 |
| `kind` | 本接口调用填写 `generation` |
| `input_tokens` | `usage.prompt_tokens` |
| `output_tokens` | `usage.completion_tokens` |
| `cached_input_tokens` | 优先 `usage.prompt_tokens_details.cached_tokens`，缺失时可取 `usage.prompt_cache_hit_tokens`；两者不相加 |
| `measurement` | 有真实服务 usage 时填写 `api_usage` |

`cached_input_tokens` 是输入总量的子集，不再加到 `input_tokens`。`reasoning_tokens` 是输出明细，已经计入 `completion_tokens` 的部分不重复加；不要以可见回答字符数代替服务报告的生成 token。

按上面的格式示例，一次调用的输入是 15、输出是 1540、缓存输入是 0；不能把 1190 再加到 1540 上。这些数字只解释字段，实际报告必须统计你自己的调用，包括构建阶段的合并、修复和重试。

每次 `ingest` 按模型汇总该命令的所有调用，并如实填写 `usage.complete`；缺失的用量不能当作 0。查询用量与构建用量分开。模型 tokenizer 分母与倍率由面试官核算；成本及耗时仍不直接加扣分。

调用失败、响应解析失败时，保留真实错误并按题目接口返回，不用本页示例内容替代真实运行结果。

本说明依据面试方提供的协议编写，未实际调用服务验证 AK 可用性、额度或模型权限。
