# DeepSeek Harness 上下文机制优化
复制给你的agent让它来帮你修改就可以,按照这说明进行优化，以后遇到超出极限的上下文时，新建一个文log,生成对应的对话log.txt
按照这说明进行优化，以后遇到超出极限的上下文，在工作目录的新建一个文件夹来放置这些log,生成对应的对话log.txt
# DeepSeek Harness 优化配置（直接复制进你的 profile，解决 compact 失败 + 工具输出爆 token）

> 
>
>**profile yaml 片段**

```
# ========== 1. 工具返回截断（根治单次tool_result塞满上下文，最关键） ==========
toolResultPrune:
  thresholdChars: 6144       # 单条工具输出超过这个字符就自动裁剪
  maxLeadingChars: 3072      # 保留开头
  maxTrailingChars: 2048     # 保留结尾（日志一般末尾报错最重要）
  ellipsisNote: "\n\n【中间超长内容已省略，如需查看完整日志请写入本地文件读取】"

# ========== 2. Compact 自动压缩配置，减少压缩失败概率 ==========
compaction:
  # 触发阈值：上下文占用达到窗口80%时自动执行compact
  triggerRatio: 0.8
  # 保留最近N轮对话不压缩（最近几轮强制保留，避免摘要丢失当前任务状态）
  retainRounds: 6
  # 备选摘要模型：主模型压缩失败时，换轻量模型做摘要（大幅降低压缩翻车）
  summarizationModel: deepseek-chat
  # 摘要生成时的最大输出token，防止摘要写到一半被截断导致判定无效
  maxSummaryTokens: 2048
  # 压缩重试次数，第一次压缩失败自动重试1次
  retryAttempts: 1
  # 压缩策略：优先丢弃旧工具日志，而不是丢弃思考过程
  priority: dropToolOutputsFirst

# ========== 3. 模型生成参数，解决「输出token上限截断」 ==========
modelParams:
  maxOutputTokens: 8192      # 调高单次输出上限，对应你截图的“已达到输出token上限”
  temperature: 0.3

# ========== 4. 上下文预算兜底保护（防止窗口直接打爆） ==========
contextBudget:
  retainRatio: 0.15          # 至少保留总窗口15%作为最近对话，不压缩
  headroomTokens: 4096       # 预留安全余量，临近窗口提前触发压缩
```

## 配套手动操作方案（遇到`Compaction could not produce a useful summary`报错时）

1. **不要一次性 compact 全部会话**
`compact --before 10`**保留最近 10 轮**
> 
>
**超大日志不再塞进对话**
`log.txt``read_file`
`所有超过3000字符的命令输出，保存到./tmp/xxx.log，不要直接返回文本`
3. 应急重置：
`new`**简短任务目标**

## 重点说明

1. `maxOutputTokens`**单次生成上限**
2. `summarizationModel`**不要用 reasoner 模型做摘要**`useless summary`
3. `toolResultPrune`

## 可选增强插件（如果你想更激进自动压缩）

`@falling-ts/dsh-force-compact`

```
dsh plugin add @falling-ts/dsh-force-compact
```

