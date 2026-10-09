---
name: dsh-orca-computer-use
description: "DSH 沙箱下 Orca computer use 环境技能 — 在 DeepSeek Harness (DSH) 的 bwrap 沙箱命令里启动并验证 Orca 的 `orca computer` 通道（accessibility 树、点击、输入、截图），含本机固定路径、必须带的 --no-sandbox / HOME、进程保活方式（daemon 空闲自动退出）、/tmp 隔离等坑；附当前 live profile 的压缩/截断/输出参数全集；附带 agent-browser 快速自检。Verified 2026-10-01 on Ubuntu 24.04, 360Browser, DSH web GUI 127.0.0.1:3080."
whenToUse: "当需要在 DSH 的 bash 命令里用 `orca computer` 驱动桌面窗口（360 浏览器、原生应用、webview）、Orca CLI 报 runtime_unavailable / runtime_access_denied、Orca 后台 job 无故 completed、或要快速自检 agent-browser 是否可用时使用。"
---

# DSH 沙箱下的 Orca computer use（本机环境技能）

`computer-use` skill 是 Orca 官方通用 stub；本 skill 记录**本机 + DSH 沙箱**的具体路径、启动方式和坑。通用操作循环（list-apps → get-app-state → click/press/type）以 `computer-use` skill 里 `skills get computer-use` 的 guide 为准。

## 0. 本机固定值（直接抄用）

| 项 | 值 |
|---|---|
| ORCA 可执行文件 | `/home/qiang65/deepseek工作目录/orca-app/opt/Orca/orca-ide` |
| 必须带的参数 | `--no-sandbox`（见坑 1） |
| Orca 的 HOME | `/home/qiang65/deepseek工作目录/orca-home`（**CLI 和 Orca 都要**，runtime 元数据在 `$HOME/.config/orca/orca-runtime.json`） |
| Orca daemon 日志 | `$HOME/.config/orca/logs/daemon.log`（JSON 行：startup / ready / client-hello-accepted / **shutdown reason:idle**） |
| DSH profile（live） | `/home/qiang65/.dsh/profiles/web/cordis.patch.yml`（`patchReload: live`，改完即热加载） |
| runtime socket | 见 runtime 文件内；CLI 连接走 6768 端口 + unix socket |
| agent-browser | `<工作区>/ab`（wrapper，socket 目录 `.ab-sock`） |

`ORCA_CLI_COMMAND` 未设置、`ORCA_DEV_REPO_ROOT` 未设置；`/usr/bin/orca` 是 GNOME Orca 读屏器（别裸跑，会开语音）。

## 1. 诊断：Orca 在不在

```bash
ORCA="/home/qiang65/deepseek工作目录/orca-app/opt/Orca/orca-ide"
export HOME="/home/qiang65/deepseek工作目录/orca-home"
timeout 10 $ORCA --no-sandbox computer capabilities --json
```

（**用 `timeout` 包**：runtime 文件残留但 socket 已死时 CLI 会挂起不返回。）

- `"ok": true` + provider `orca-computer-use-linux` → 通道正常，直接去用（`list-apps` / `get-app-state`）
- `runtime_unavailable`（快返回）→ Orca 没在跑（或 CLI 的 HOME 不对）
- `runtime_access_denied` → 沙箱挡了连接，escalate 权限重跑，**不要** `ORCA open` 或重启
- 挂起/超时 → runtime 文件指向死 socket（Orca 死了没删干净），看 daemon.log 确认

看死因（**空闲自动退出是正常现象，exit 0**）：

```bash
tail -5 $HOME/.config/orca/logs/daemon.log
# {"event":"shutdown","reason":"idle"} = daemon 空闲超时自退，非崩溃
```

## 2. 启动 Orca（DSH 里最稳的方式：后台 job）

**用 DSH 后台 job 跑，不要 setsid nohup**（坑 2）：

```bash
export HOME="/home/qiang65/deepseek工作目录/orca-home"
cd /home/qiang65/deepseek工作目录/orca-app/opt/Orca
./orca-ide --no-sandbox > <工作区>/orca-bgt.log 2>&1
```

（以 `run_in_background` 方式执行该命令，Orca 进程随 job 存活。）

等待就绪（约 20-30 秒）：runtime 文件出现 + `capabilities` 返回 `ok:true` 即可：

```bash
sleep 20
timeout 10 $ORCA --no-sandbox computer capabilities --json
```

**注意**：daemon 空闲会自退（实测 4h+ 空闲后 `shutdown reason:idle`，job 显示 completed exit 0）。job 无故 completed 时不用找 bug，重启即可。

旧脚本 `orca-launch.sh`（setsid nohup）会"悄悄死"，别再依赖它。

## 3. 验证（端到端四连）

```bash
$ORCA --no-sandbox computer list-apps --json                                   # 能列出桌面应用
$ORCA --no-sandbox computer get-app-state --app 360Browser --no-screenshot --json  # 能拿窗口+树
$ORCA --no-sandbox computer get-app-state --app 360Browser --json               # 含截图
```

截图在**执行命令的沙箱**的 /tmp 里，要用就**同一条命令内**拷进工作区（见坑 3）。

2026-10-01 验证记录：capabilities / list-apps / get-app-state / 截图（1325x874 真实桌面，360Browser 窗口"test — DeepSeek Harness - 360安全浏览器"）全部通过。

## 4. 当前 live profile 参数全集（2026-10-03 生效值，改参数前先看这里）

**生效位置**（重要，见坑 10）：web profile 的 compaction/pruner 配置必须写在
`~/.npm/_npx/1e7f6d9597241db0/node_modules/@deepseek-ai/dsh-web-app/presets/{standard,ptc,cordis}.patch.yml`
里嵌套的条目上；`~/.dsh/profiles/web/cordis.patch.yml` 的同名条目只作用于顶层
（web-app 已置 `disabled: true`），**改它不生效**。两边目前保持同值，便于对照。

```yaml
- id: tool-result-pruner        # 单条工具输出 > 6144 字符就裁
  config:
    thresholdChars: 6144
    headChars: 3072              # 保留开头
    tailChars: 2048              # 保留结尾（日志末尾报错最重要）
- id: compaction-basic
  config:
    thresholdRatio: 0.75         # 窗口 75% 触发（比方案原值 0.8 更早）
    headroomRatio: 0.02          # 安全余量 = contextWindow×2%（256k 窗口 → 5120）
    reservedRatio: 0.25          # 输出预留 = contextWindow×25%（256k 窗口 → 64000）
    retainRatio: 0.3             # 压缩后至少保留 messageBudget 的 30%
    maxTokens: 8192              # 摘要输出上限（4096 会截断大 checkpoint，见下）
    compactionRetries: 2         # 失败自动重试（方案原值 1）
    # 不指定 summarizationProvider/Model = 摘要跟随会话当前模型
    # （本地模型→本地摘要；云端模型→摘要也上云。见下）
```

**摘要目标：不指定，跟随会话当前模型**（2026-10-03 定稿，用户要求「本地走本地、云端走上云」）。插件源码行为：`summarizeWithLlm` 里 `configured = config.summarizationProvider.length === 0 ? undefined : {...}`，为空时 `target = agent.session.requestHeader()?.config`（会话最新请求的 provider/model）。

| 会话主模型 | 摘要走哪 |
|---|---|
| kvmem 27B（18200，256k） | 本地 kvmem（256k 窗口，能吞大区间 replay） |
| strata 64k（8080） | 本地 strata |
| 远程 llama（192.168.18.48:8080） | 局域网 llama |
| 云端 deepseek-flash / v4-pro | 同样上云（deepseek-official api-key 路由） |

好处：机器上一次只能开一个本地模型（GPU 限制），换模型时摘要自动跟上，**不需要改配置、不需要重启**；用云端主模型时也不再依赖本地服务。

本地服务启动（kvmem 27B，`--host 127.0.0.1 --port 18200 -c 262144`，catalog 写 `contextWindow: 256000 / maxTokens: 8192`）：

```bash
nohup bash "/media/qiang65/data/ubuntusoftware/kvmem-v0.16.0-rc3-linux-x86_64-cuda13.2.86/scripts/linux/start-iq3.sh" > ~/kvmem-18200.log 2>&1 &
curl -s http://127.0.0.1:18200/v1/models | head -c 120   # 验证
# 64k 那个是 Strata（python），由它自己的方式起：/media/qiang65/data/Strata/serve/server.py --engine strata --config ... --port 8080
```

⚠️ `/home/qiang65/start-iq3.sh` 是错位拷贝（脚本内 `ROOT="$(dirname $0)/../.."` 会算成 `/`），**不要用**。摘要目标对应的服务没起来时，`compaction/end` 会记 `Connection error.`，会话不会瘦身（旧对话看起来还是"满"）——见坑 11。

**不能用** `deepseek-account/deepseek-chat` 作为固定摘要目标：`~/.dsh/.credentials.yaml` 里只有 `client-connection/browser-session` 一条 record（无 `deepseek-account-platform/default`），走它每次摘要都会抛 `ACCOUNT_SIGN_IN_REQUIRED` → 压缩永远失败。云端要走就走 api-key 路由 `deepseek-official/*`（`DEEPSEEK_API_KEY` 在 credentials refs 里）。

**headroomRatio / reservedRatio 是 2026-10-03 补的插件代码补丁**（支持比例制，优先于 headroomTokens/reservedTokens 绝对值）：补丁位置 `~/.npm/_npx/1e7f6d9597241db0/node_modules/@deepseek-ai/dsh-compaction-basic/lib/index.js`（`resolveCompactSpec`），**dsh 升级换 npx 缓存后补丁丢失**，需重打；改前旧值（headroomTokens=4096、无 reservedRatio，reserved 默认取模型 maxTokens）记录在本节末尾的"实际触发点"注释里。公式（源码 `resolveCompactSpec`）：

```text
headroom   = headroomRatio  ? round(contextWindow × headroomRatio)   : headroomTokens(默认值)
reserved   = reservedRatio  ? round(contextWindow × reservedRatio)   : 模型 maxTokens
messageBudget = contextWindow − reserved
trigger  = floor(min(contextWindow × thresholdRatio, messageBudget − headroom))
retain   = floor(messageBudget × retainRatio)
```

模型条目与 provider 级超时（`llm-pi-ai` 的 kvmem）：

```yaml
      kvmem:
        # 本地 27B 做大 replay 摘要时 prefill 很慢（实测 17.6k token 的 replay 要 87.6s），
        # 默认 5 分钟 stream idle 超时会误杀（历史 "pi-ai stream idle timeout after 300000ms"），
        # 故放宽到 15 分钟。
        streamIdleTimeoutMs: 900000
        models:
          - id: Qwen3.8-27B-Uncensored-YMQ-XS-Pro.gguf
            contextWindow: 256000
            maxTokens: 8192      # 单次输出上限，设了即成为每请求默认值
```

实际触发点（由上面公式算出，2026-10-03）：

```text
kvmem 256k：reserved=64000 headroom=5120 → trigger = min(192000, 186880) = 186880
            retain  = 192000×0.3 = 57600
64k 模型：  reserved=16000 headroom=1280 → trigger = min(48000, 46720) = 46720
            retain  = 48000×0.3 = 14400
```

（改 ratio 前是绝对值 headroomTokens=4096 + reserved=模型 maxTokens，64k 窗口预算被吃死会报 `exceed_context_size_error`；比例制后换任意 contextWindow 都自动缩，见 2026-10-03 session-188cca1d 的 72909/64000 报错。）

**执行顺序**（源码 `compactIfNeeded`）：① 先跑 `pruneToolOutputs`（把会话里所有 >6144 字符的工具日志裁成头 3072 + `[... tool result middle pruned ...]` + 尾 2048）→ ② 裁完低于阈值就结束，**不用 LLM 摘要** → ③ 不够才用**会话当前模型**做摘要压缩。所以**工具结果 > 阈值时当轮 tool_result 是完整的**，裁剪在下一轮 compaction 才发生——别拿"刚跑的大输出没被裁"当故障。

### 4.1 三个代码补丁（2026-10-03，都在 npx 缓存里，升级即丢）

| 补丁 | 文件 | 改了什么 |
|---|---|---|
| 比例制 headroom/reserved | `@deepseek-ai/dsh-compaction-basic/lib/index.js` `resolveCompactSpec` | 支持 `headroomRatio`/`reservedRatio`（见上公式） |
| **pi-ai max_tokens 钳制** | `@earendil-works/pi-ai/dist/api/simple-options.js` + `dist/utils/estimate.js` | ① 估算补上 `context.tools`；② 启发式部分 ×1.15；③ **可用预算 <1024 时不钳**，交回服务端裁决 |
| **token-meter 压力保守化** | `@deepseek-ai/dsh-token-meter/lib/index.js` `measure()` | 启发式 baseline/delta ×1.25（真实 usage 不放） |

**为什么打这两个估算补丁**：本地 OpenAI 兼容服务（llama.cpp/strata/kvmem）**从不截断**，超窗直接 400 `prompt (N tokens) + max tokens (M) exceeds the context (K); requests are never truncated`。而 pi-ai 算输出预算时：

```text
# 补丁前（dist/api/simple-options.js）
max_tokens = min(请求值, max(1, contextWindow − 估算prompt − 4096))   # CONTEXT_SAFETY_TOKENS
# 估算prompt = estimateContextTokens()，只用 chars/4 数 messages，
# 完全漏掉 context.tools（本机 26 个工具 schema ≈ 20611 字符 ≈ 5153 tokens），
# 且对代码/JSON/中文再低估 ~20%
```

补丁后的实际公式（`dist/api/simple-options.js`）：

```text
est      = estimateContextTokens(context)      # usage 锚点精确；其余 chars/4 + tools
padded   = est.tokens + ceil(heuristic × 0.15) # 只放大启发式那部分
available= contextWindow − padded − 4096
max_tokens = available < 1024 ? 请求值 : min(请求值, available)
```

实测反推（session-fd0effdc，64k 模型）：`50940 + 23763`、`56402 + 19931` 两条报错里的 max_tokens 正好等于 `65536 − est − 4096`，反解 est=37677/41509，而服务端实际 prompt 是 50940/56402 → **低估 26%**（主因：估算完全没算 26 个工具 schema ≈ 5153 tokens）。

反向的坑也在本机出现过（session-ac22093b，2026-10-03）：估算**偏高**时 ×1.3 把 `max_tokens` 压到 1 → 服务端 usage 记 `inputTokens 42929 / outputTokens 1`、`turn/end: max-tokens`，用户看到的就是**空响应**（像"上下文满了"，其实 42929+8192=51121 < 65536 完全塞得下）。所以：放大系数降到 1.15，并加 `MIN_USEFUL_MAX_TOKENS = 1024` 下限——**可用预算不足就不要再钳**，让服务端按真实计数裁决：真超窗会 400，而 `compaction-basic` 监听 `agent/request-error` 会**自动压缩 + 重试**（这才是设计路径）。

**验证方式**（不需要重启，直接 import 补丁后的模块）：

```bash
node /home/qiang65/deepseek工作目录/verify-compaction-patches.mjs
# 覆盖 4 个场景：小 prompt 不钳 / 中等 prompt 交回服务端 / 超大 prompt 交回服务端 /
# usage 锚点精确值不被放大；出现 "⚠️ 危险：只给 1 个 token" 就说明下限失效了
```

**重启方式**（补丁是代码，必须重启才加载；profile YAML 才是热加载）：

```bash
# 推荐：systemd 独立 unit 跑重启脚本（脱离本会话 cgroup，杀旧服务不会连坐）
systemd-run --user --unit=dsh-restart-$(date +%s) --collect --property=KillMode=process \
  --setenv=HOME=/home/qiang65 --setenv=PATH=/home/qiang65/.local/node22/bin:/usr/local/bin:/usr/bin:/bin \
  bash -c 'sleep 15; bash "/home/qiang65/deepseek工作目录/restart-dsh-web.sh"'
# 脚本自己会用 systemd-run --user --unit=dsh-web-server-<ts> 把新服务拉成独立 unit，
# 只搬运 HOME/PATH/DBUS/XDG 等白名单 env（旧服务 /proc/<pid>/environ），实测 3 秒起来。
tail -f /home/qiang65/deepseek工作目录/dsh-web-restart.log   # 看到 ✅ 即成功
```

⚠️ 早先用 `setsid nohup` 直接跑重启脚本会**在杀掉旧服务后被连坐回收**（日志只有首行、"✅" 永远不出现，服务靠用户手动/桌面图标才回来）——必须用 systemd unit，见坑 13。

非 schema 键（方案里写过但实际不存在的，行为已被等价覆盖）：`ellipsisNote`（标记固定）、`retainRounds`（比例制）、`priority: dropToolOutputsFirst`（先裁工具日志就是实际逻辑）、model 条目 `temperature`。

## 5. agent-browser 快速自检（独立于 Orca）

```bash
cd /home/qiang65/deepseek工作目录
./ab open https://www.baidu.com --session checkN     # 开页面
./ab get title --session checkN                      # 应返回 "百度一下，你就知道"
./ab screenshot /tmp/ab-check.png --session checkN   # 截图
./ab close --session checkN                          # 清理
```

失败模式与背景详见 `<工作区>/agent-browser-notes.md`。

## 6. 坑（按踩坑顺序）

1. **`--no-sandbox` 必须带**：SUID `chrome-sandbox` helper 没配 4755，不带就 FATAL 崩（`chrome-sandbox is owned by root and mode 4755`）。GPU 报错 `GLDisplayEGL::Initialize failed` 非致命，回退软渲染。
2. **setsid nohup 会悄悄死**：进程没了、无 FATAL、socket 文件残留。DSH 后台 job 方式存活，且 daemon 空闲自退（exit 0）是**正常**，见 §1 daemon.log。
3. **/tmp 每沙箱独立**：每条 bash 命令和每个 job 一个独立 /tmp。CLI 截图写在"执行该命令的沙箱"的 /tmp 里，命令结束即失效 → **要用截图就同一条命令内拷进工作区**（`result.screenshot.path` 或 inline base64）。
4. **CLI 挂起**：runtime 文件残留但 socket 死时，`capabilities` 会挂住不返回 → 用 `timeout 10` 包。
5. **pasteText / menus 不支持**（capabilities 返回 `false`），输入用 type。
6. **启动后 20 秒内 xdotool 可能搜不到 Orca 窗口**（渲染慢），以 CLI 返回为准。
7. **插件代码补丁 ≠ 配置热加载**（2026-10-03 踩）：headroomRatio/reservedRatio、pi-ai 估算、token-meter 压力都是打在 `~/.npm/_npx/...` 里的代码补丁（§4.1），**改代码要重启 `dsh web` 才生效**；profile YAML 改动才是热加载（preset 补丁文件也是 bundle 层，同样要重启）。dsh 升级换 npx 缓存后代码补丁丢失，要重打。重启用 §4.1 / 坑 13 的 systemd 方式。
8. **`exceeds the context ...; requests are never truncated` 是本地服务端拒收，不是 DSH 报错**（2026-10-03）：出现在 `compaction/end` 里时，说明**摘要请求**超窗；出现在 `assistant/attempt`/`turn/end` 里时，说明**主请求**超窗。三类原因与对策（§4.1）：① pi-ai 估算漏掉 tools/偏低 → 请求的 `max_tokens` 给大了（补丁已补 tools + 轻量放大；可用预算不足时改为**交回服务端**，让 400 触发自动压缩重试）；② DSH token-meter 启发式偏低 → 压缩触发太晚（补丁把启发式压力 ×1.25）；③ 摘要目标不可用（服务没起 → `Connection error.`，或 account 未登录 → `ACCOUNT_SIGN_IN_REQUIRED`）→ 现在默认**跟随会话模型**，把对应本地服务起起来即可；要钉云端只能用 api-key 路由 `deepseek-official/*`。
9. **`summarization truncated at the token cap (incomplete checkpoint)` = compaction `maxTokens` 太小**（2026-10-03 踩）：这是摘要**输出**被截断（不是输入超窗），把 `compaction-basic.config.maxTokens` 从 4096 提到 8192。同一错误别和坑 8 混淆——两者的修法相反（一个要压输出预算，一个要提输出预算）。
10. **profile 补丁打不到 preset 内的条目（本次最大坑，2026-10-03）**：`~/.dsh/profiles/web/cordis.patch.yml` 里的 `- id: compaction-basic` 是**空操作**。web-app 把顶层 `compaction-basic` 置 `disabled: true`，真正生效的实例嵌在 `preset-standard`/`preset-ptc`/`preset-cordis` 的 `config.plugins` → `compaction` 组里；loader 的 `buildMap`（`dsh-app-boot/lib/index.js` 的 `applyEntryPatches`）**只递归 `entry.group && Array.isArray(entry.config)`**，而 preset 是 `@deepseek-ai/dsh-agent-preset`、插件列表在 `config.plugins`，所以同 id 条目匹配不到，配置全落在 disabled 那条上。症状：纸面上写的 `thresholdRatio/maxTokens/summarization*` 全不生效，实际跑默认值（`maxTokens` 默认 65536、无摘要目标 → 摘要落到会话模型 → 被 pi-ai 钳成 `contextWindow−估算−4096`，即 23763/35499 这类怪值）。
    **判断方法**：`dsh --profile web --dump-config | grep -A 13 compaction-basic`，看带 config 的是不是 `disabled: true` 那条、嵌套实例有没有 config（`--dump-config` 与真实启动共用同一套补丁语义，不会漂移）。
    **修法**：把配置写进 `@deepseek-ai/dsh-web-app/presets/{standard,ptc,cordis}.patch.yml` 里嵌套的 `compaction-basic` / `tool-result-pruner` 条目（npx 缓存，升级后要重打）；`--patch` 覆盖层同样匹配不到，实测已验证。
11. **`compaction/end: "Connection error."` = 摘要服务没起**（2026-10-03 踩）：摘要跟随会话模型，所以摘要服务就是「会话正在用的那个模型服务」；若配置里钉死的摘要目标（早期版本钉过 kvmem/strata）没在跑，压缩就会以 connection error 失败 → 会话不瘦身 → **旧对话看起来一直"满"**。识别：事件序列里有 `compaction/prune`（工具日志裁了）和 `compaction/start`，但 `compaction/end` 带 `Connection error.`。对策：起对应本地服务（§4 命令），或保持「不指定摘要目标」让它跟随会话模型。
12. **钳制过紧 = 静默空响应（比 400 更坏）**（2026-10-03 踩，session-ac22093b）：症状不是报错，而是助手回复空白、`turn/end: max-tokens`，该轮 usage 形如 `inputTokens 42929 / outputTokens 1`。原因是 pi-ai 的输出预算被估算压到 1 个 token。**看到 `outputTokens` 极小就要查这里**，不要误判成"上下文满了"（42929+8192=51121 < 65536，其实塞得下）。修法见 §4.1：×1.15 + `MIN_USEFUL_MAX_TOKENS=1024` 下限，宁可让服务端 400（会触发自动压缩重试）。
13. **重启 dsh web 要用 systemd unit，别用 setsid**（2026-10-03 踩）：`setsid nohup bash restart-dsh-web.sh &` 会在杀掉旧服务后被**连坐回收**（`dsh-web-restart.log` 只有第一行、永远等不到 "✅"），结果用户以为重启失败。改用 `systemd-run --user ... restart-dsh-web.sh`（§4.1），脚本内部再用 systemd-run 把新服务拉成独立 unit，实测 3 秒起来。桌面兜底：`~/桌面/启动dsh-web.desktop` / `~/启动脚本/启动-dsh-web.sh`。
