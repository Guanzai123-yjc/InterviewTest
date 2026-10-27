# 验收接口 v3

这里只约定可执行接口和交付格式，不限定内部实现。stdout输出一个JSON值，日志写stderr；正常退出码0，非法参数、解析/模型调用等失败退出非0，返回 `{"status":"error","error":{"message":"具体原因"}}`。失败原因能确认来自外部服务时可标记 `error.category="external_service"`，保留真实依据。

## 输入与构建

JSONL每行会话：`user_id`、`session_id`、正整数`sequence`、`recorded_at`及`messages`；每条消息含`message_id`、`role`、`content`。sequence仅表示数据接收顺序，原文和稳定ID供证据核验。

`ingest --input PATH --store PATH` 输出：

```json
{"successful_sessions":10,"skipped_sessions":0,"failed_sessions":0,"failed_session_ids":[],"usage":{"complete":true,"models":[]}}
```

计数为本次新增、已存在跳过、失败的会话数。重复导入不得改变事实；面试官会依次导入初始、增量，再各重复一次。部分失败保留已知计数与usage并标记不完整。`export --store PATH` 输出可检查内部记忆与来源的JSON，结构自由。存储应能跨命令/进程恢复。

## 查询及回答

`query --store PATH --query TEXT`：接受自然语言问题，依据该存储已经导入的数据回答。空白问题非法。公开题在两批数据全部导入后运行。历史问题通过问题本身的时间描述表达。

示意如下，方括号中的文字是占位内容，**不是样例答案**：

```json
{
  "status":"ok",
  "answer":"[对问题的具体回答；信息不足时说明已知事实与缺口]",
  "memories":[{
    "memory_id":"[稳定ID]",
    "subject":"[主体标签]",
    "text":"[实际召回记忆]",
    "source_ids":["[原始消息ID]"],
    "evidence":[{"message_id":"[原始消息ID]","quote":"[对应消息的连续原文片段]"}]
  }],
  "timings":{"retrieval_ms":0.0},
  "trace":{"summary":"[可选：检索步骤、筛选依据或失败位置的简要说明]"}
}
```

`status` 为 `ok`、`insufficient` 或 `error`。正常查询无论ok还是insufficient都要给非空answer。主体标签自由但含义明确。引用必须来自已导入数据，quote必须是对应消息中的连续原文。信息不足可以返回有用的边界证据；真正无关问题可返回空memories并说明没有相关记录。

本题不限定召回条数、记忆及引文长度、回答长度或查询历史上下文字符数。召回内容是否相关、是否遗漏必要事实仍按实际语义评价。`usage.history_input_chars`可选，用于描述本次查询的历史输入字符量，不设置上限，也不参与评分。

`timings.retrieval_ms` 为实际检索阶段的累计毫秒数：包含查询向量化、候选检索、过滤、重排及证据读取，结束于最终回答生成前；多次检索相加，使用单调时钟。无检索工作可为0，未测量用null并解释，不能编造数字。评测器还独立测量整次CLI耗时，含进程启动、加载、检索和回答生成；两者不等价。缺少自报计时会标记不完整，不能据此声称“检索0ms”。

## answers.jsonl 与 HTML

每道公开题恰好一行，题号、问题文本必须与 `questions.jsonl` 一致，result是两批数据全部导入后该次query的完整真实返回值：

```json
{"question_id":"[题号]","question":"[原问题]","result":{"status":"insufficient","answer":"[本次真实回答]","memories":[],"timings":{"retrieval_ms":null}}}
```

不得用示意占位内容提交。失败也保留真实error并在复盘解释，不把失败伪装成成功。HTML应展示同一批结果，以及构建产物；为自动核对，嵌入两个JSON脚本块：

- `<script type="application/json" id="answers-data">`：JSON数组，内容与answers.jsonl逐行对象一致，顺序可按题号比较。
- `<script type="application/json" id="memory-data">`：真实export JSON。

HTML由候选人自行制作，需满足此约定并能离线打开；不依赖CDN、服务或账户。脚本只核对嵌入数据，面试官还会检查可见展示是否一致。AI_NOTES.md简要记录实际工具/任务、问题及修复、验证命令与结果、经验；无须完整过程日志。

## 构建 token 参考统计

仅累计四次正常导入（初始、增量、重复初始、重复增量），不计开发AI、查询、评测或故障测试。每次ingest按模型汇总：

```json
{"complete":true,"models":[{"model":"[实际模型标识]","kind":"generation","input_tokens":12345,"output_tokens":1234,"cached_input_tokens":0,"measurement":"api_usage"}]}
```

数字仅示意。kind为generation/embedding/rerank；measurement为api_usage/tokenizer_estimate；缓存是总输入子集，不再相加。输入含提示词、旧记忆重读、合并、修复、校验、重试及embedding/rerank；输出含丢弃、覆盖、修复、重试的生成token，计费推理token遵循服务usage，不重复计算已含明细。embedding向量维数和重排分数不是生成输出。没有模型调用明确为 `{"complete":true,"models":[]}`；缺失和部分计量不能填0，使用complete:false并保留已知明细。

面试官以每个模型各自tokenizer编码两份原始完整JSONL文件（含ID/role/元数据/换行）得到固定分母B，绑定输入hash。四次导入输入I、生成输出O的倍率为 `R_in=Σ(I_m/B_m)`、`R_out=Σ_generation(O_m/B_m)`。分母只算一次，不随重复导入变化；不以字符数或持久化记忆大小代替token。缺计量/分母时倍率null且不影响质量评分；倍率不是货币成本。usage默认是提交程序自报，未自动与服务端独立对账。
