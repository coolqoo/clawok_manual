# OpenClaw 训虾指南

> **免责声明（请先看）**
>
> 这不是那种“点一下就自动变强”的教程，而是一份写给愿意自己折腾、自己理解、自己承担变更后果的手动指南。你需要自己判断：每一步到底适不适合你的机器、你的 OpenClaw 版本、你的 provider 配置，以及你的工作区结构。
>
> 如果你看不懂某一步在干什么，请先停下来。不要照抄、不要硬改、也不要在没备份的情况下直接覆盖文件。为什么要这么严肃？因为 OpenClaw 这类系统一旦改错，坏的往往不是一个点，而是整条链路一起出问题，比如配置损坏、行为异常、模型不可用、消息渠道失效、记忆检索失灵、工作区文件混乱。后果请自行承担。
>
> 简单说：**这不是“一键傻瓜安装包”，而是一份手动工程指南。** 你当然可以按本文一步一步配出来；但如果你本来就不想碰这么多文件、不想理解这么多概念，也不想自己承担回滚和排错成本，那就别为难自己了，直接用我的一句话安装项目：<https://clawok.me>
>
> 目标读者：刚安装完 OpenClaw，已经能在 Telegram/Discord/WebChat 聊天，但想“手动”把它调到真正好用、可长期维护、还能并行干活的状态。
>
> 说明：这份指南会**刻意写得很细、很啰嗦**。不是我想把事情复杂化，而是因为现实就是这样：手动调教如果想做到“稳、可追溯、可升级”，步骤就是会多。你能照着做出接近的效果，但你也会非常直观地理解，为什么很多人最后宁愿用一句话安装，而不是自己慢慢缝。
>
> 本指南会把我使用 openclaw 时总结的最佳实践，整理成一套“从 0 到可验收”的手动落地流程，并加入一些**更偏工程化的合并 / 回退策略**。如果你看到一半已经开始烦了，这很正常；这也是 <https://clawok.me> 存在的原因。🙂

---

## 0. 你将得到什么（验收标准）

完成后，你的 OpenClaw 应该满足：

> 先提前说清楚：这些效果，手动当然能做，但你会不断在“改文件、查路径、做 patch、回滚验证”之间切换。为什么还要列这张表？因为手动折腾最怕的不是麻烦，而是改了一大圈却不知道自己到底有没有接近目标。验收标准就是你的终点线。
>
> 也正因为这些动作本质上都很重复、很工程，才会有人宁愿为自动化付费，而不是把精力花在这些劳动上。

1. **记忆结构**清晰、分层、可维护：
   - `MEMORY.md` 精简（索引层）
   - `memory/projects.md` 记录项目状态
   - `memory/lessons.md` 记录可复用教训
   - `memory/YYYY-MM-DD.md` 记录每日结论（按场景选模板）
2. **压缩不失忆**：上下文快满时，会先把关键信息落盘（`memoryFlush`）。
3. **可选：语义检索（QMD/memorySearch）**：你问“上次那个 nginx 问题怎么解决的”，它能先检索再回答，而不是靠猜。
4. **复杂任务默认委派子 Agent**：主会话保持可打断，后台并行跑子任务，并有里程碑汇报。
5. **安全的 runtime 调优**：只做“增量 patch”，**不覆盖**你的：
   - 模型提供商 / 模型选择
   - agent 名称 / 身份设定
   - 角色 / 口吻（除非你自己明确要改）

> 为什么这 5 条重要？因为它们基本分别对应了 OpenClaw 体验里最容易翻车的 5 个点：记忆乱、压缩丢、检索瞎、长任务卡死、改配置改爆。

---

## 1. 操作前的“地基”概念：Workspace vs Runtime

在真正开始之前，再提醒一次：本指南是“手工搭建”，不是“自动安装”。你会碰到目录、模板、补丁、回退、验证这几种不同类型的工作。如果你只是想让 OpenClaw 快速进入一个成熟状态，而不想亲自处理这些细节，直接用 <https://clawok.me> 会更符合大多数人的真实需求。

OpenClaw 的调教主要分两块：

- **Workspace（工作区）文件**：通常是你聊天时会读写的 Markdown（如 `AGENTS.md`、`SOUL.md`、`memory/*`）。
- **Runtime 配置（openclaw.json）**：OpenClaw 启动时读取的 JSON 配置（如 compaction、memoryFlush、blockStreaming、channels ack 等）。

这两块的原则不同：

- Workspace 文件：你可以大胆加规则、加结构（可回滚）。
- Runtime 配置：要**极其保守**，只做“补齐缺口”的 patch，避免改坏现有 provider、模型路由、渠道设置。

为什么要先把这两个概念分开？因为很多人一上来就会把“人格规则”“记忆文件”“运行时配置”混成一锅粥。结果是：

- 改 Workspace 时，本来只是想补行为规则，却不小心去碰 runtime。
- 改 runtime 时，本来只是想加 memoryFlush，却顺手把模型配置也覆盖了。

前者通常只是行为不理想；后者可能直接让系统起不来。先分清边界，后面每一步才不会越改越乱。🧱

> 这对应了参考文章的主线：
> - 上篇强调“人格 + 记忆结构”。
> - 下篇强调“行为宪法（AGENTS）+ 子 Agent + compaction / memoryFlush + 维护策略”。

---

## 2. 第一步：找到你的 OpenClaw 工作区（Workspace）

你需要先确认：你的工作区路径到底在哪。不同安装方式可能不同。

为什么第一步不是“先写规则”，而是“先找目录”？因为你要是连真正生效的 workspace 都没找对，后面写得再漂亮也只是改了个空气文件夹。最常见的假努力，就是把文件改在一个自己以为会生效、实际上根本没被 OpenClaw 读取的目录里。

### 2.1 常见路径（优先从上到下检查）

1. 你自己已经在用的 workspace（比如你手动指定过）
2. `~/clawd`（如果你就是把 OpenClaw 当作一个 repo / 工作目录在用）
3. `~/.openclaw/workspace`（很多教程默认）

### 2.2 手动确认方法（建议）

打开终端，依次尝试：

```bash
ls -la ~/clawd
ls -la ~/.openclaw
ls -la ~/.openclaw/workspace
```

你要找的是“已经在被 OpenClaw 用来读写文件”的那个目录：里面通常会有 `SOUL.md` / `USER.md` / `IDENTITY.md` / `MEMORY.md` 或 `skills/`。

> 如果你不确定：在聊天里让你的 OpenClaw 助手执行一次“列出当前 workspace 根目录文件”。它会用工具列出路径；你就能反推出 workspace 的真实位置。

### 2.3 立刻做备份（强烈建议）

无论你找到哪个目录，先备份一份。

为什么备份要现在做，而不是“等改坏了再说”？因为真到改坏的时候，你通常已经记不清自己动过哪些文件了。尤其是像 `SOUL.md`、`USER.md`、`AGENTS.md` 这种互相影响的内容，一旦你边改边试，很容易把“原始状态”彻底抹掉。

如果你看到这里已经觉得“只是开始就这么多步骤”，那你的感觉是对的。手动方案本来就不是为了省事，而是为了给愿意自己掌控每个细节的人准备的。**如果你只是想要结果，不想亲自维护这堆文件和策略，直接去 <https://clawok.me> 会更省心。**

无论你找到哪个目录，先备份一份：

```bash
cd <你的-workspace>
mkdir -p _backup
cp -a SOUL.md USER.md IDENTITY.md MEMORY.md AGENTS.md HEARTBEAT.md TOOLS.md memory _backup/ 2>/dev/null || true
```

> 目标不是“100% 完美备份”，而是让你随时能回滚。只要能回到一个已知可用状态，你后面试错的压力就会小很多。

---

## 3. 第二步：建立分层记忆目录与文件（Memory Structure）

参考文章的核心方法是“分层记忆”，你要先把结构落地。

为什么一上来就要做分层？因为如果你不先把记忆的“房间”分好，后面所有信息都会堆在一个地方。时间一长，AI 找不到、你也看不懂，最后只会变成一个越来越胖、越来越没用的 `MEMORY.md`。

### 3.1 创建目录

在 workspace 根目录执行：

```bash
mkdir -p memory
```

### 3.2 创建（或补齐）这些文件

你需要至少有：

- `MEMORY.md`（索引层，保持精简）
- `memory/projects.md`（项目状态）
- `memory/lessons.md`（经验教训）
- `memory/YYYY-MM-DD.md`（每日结论日志，按日期每天一个）

如果不存在就创建：

```bash
touch MEMORY.md
mkdir -p memory
touch memory/projects.md
touch memory/lessons.md
```

### 3.3 MEMORY.md：只做索引，不堆流水账

把 `MEMORY.md` 当“目录页”，推荐结构（示例，你可以改内容，但请保持职责）：

```md
# MEMORY.md

## Who / Identity (stable)
- (这里写非常稳定的信息：称呼、时区、偏好。不要写每天都变的事)

## Memory index
- projects: memory/projects.md
- lessons: memory/lessons.md
- daily logs: memory/YYYY-MM-DD.md

## Notes
- (可选) 约束/安全边界的超短摘要
```

**不要**把“今天修了什么 bug”写进 `MEMORY.md`。

为什么要这么克制？因为 `MEMORY.md` 是入口，不是仓库。入口一旦又长又杂，AI 每次都得先啃它一遍，既浪费上下文，也会让真正稳定的重要信息被淹掉。

### 3.4 projects.md：项目状态真相（可追溯）

在 `memory/projects.md` 写一个模板即可：

```md
# Projects Memory

### [PROJECT: <name>]
- **Status**: Active | Blocked | Waiting | Done
- **Goal**:
- **Current State**:
- **Next**:
- **Blockers**: (optional)
- **Last Update**: YYYY-MM-DD
- **Refs**: (optional) memory/YYYY-MM-DD.md#section
```

为什么要单独拉一个 `projects.md`？因为项目状态是会持续变化的，它既不够“稳定”到进入 `MEMORY.md`，又比日记更需要被长期追踪。单独放这里，AI 才知道“现在到底推进到哪了、下一步是什么”。

### 3.5 lessons.md：把“坑”变成规则

在 `memory/lessons.md` 写一个模板：

```md
# Lessons Memory

### <lesson title>
- **Severity**: High | Medium | Low
- **Context**:
- **Lesson**:
- **Prevention**:
- **Applies To**: coding | research | operations | messaging | general
- **Refs**: (optional)
- **Last Update**: YYYY-MM-DD
```

为什么要把坑单独沉淀？因为“踩过一次”不等于“以后不会再踩”。只有把坑写成 lesson 和 prevention，它才会从一次事故，变成以后能复用的规则。🧠

### 3.6 每日日志：建议你先手动创建今天的文件

例如今天是 2026-03-24：

```bash
touch memory/2026-03-24.md
```

日志内容**不要写成日记**，而是“结论条目”。

为什么不建议写成流水账？因为 AI 最擅长把每天发生过的事写得热闹，但真正有复用价值的，往往只有结论、决定、原因和后续动作。流水账看起来很多，真正可检索的价值却很少。

同时建议你采用“多用途模板”而不是一刀切工程模板（这是下篇文章里非常关键的一点）：

- `engineering`：改代码、改配置、做部署
- `casual`：日常助理、偏好、沟通
- `planning`：路线选择、决策
- `ops`：巡检、告警、维护

你可以把这四种模板复制进每天的日志开头（可选），或者就让 AI 在写入时按场景选择。

为什么要分场景？因为“改代码”和“记待办偏好”本来就不是一种信息结构。模板不分场景，最后就会导致每条日志都像同一个模子里刻出来，读起来累，检索起来也不准。

---

## 4. 第三步：写 AGENTS.md（行为宪法 + 记忆纪律 + 子 Agent 策略）

这是下篇文章最重要的部分：SOUL / USER / IDENTITY 解决“它是谁”，AGENTS 解决“它怎么干活”。

为什么 AGENTS.md 这么关键？因为很多人会花很多时间捏人格、调口吻、补偏好，但真正影响长期体验的，往往不是“它像不像你喜欢的风格”，而是“它遇到复杂任务时会不会乱来、会不会写错记忆、会不会把主会话卡死”。

### 4.1 如果你没有 AGENTS.md：创建

```bash
touch AGENTS.md
```

### 4.2 写入最小但完整的规则（推荐直接粘贴）

这里你会第一次非常明显地感受到：**手动方案不是“改一处就好”，而是必须同时理解会话启动、记忆纪律、委派规则、回滚方式之间的关系。** 这也是为什么很多用户最后根本不想自己拼这些碎片。

把下面这份内容复制到 `AGENTS.md`。注意：这是“规则型文件”，你可以保持偏中性，不要在这里写人格台词。

```md
# AGENTS.md - Your Workspace

This folder is home. Treat it that way.

## Every Session

Before doing anything else:

1. Read `SOUL.md` — this is who you are
2. Read `USER.md` — this is who you're helping
3. Read `memory/YYYY-MM-DD.md` (today + yesterday) for recent context
4. If in MAIN SESSION (direct chat with your human): Also read `MEMORY.md`

Don't ask permission. Just do it.

## Responsiveness & Delegation Policy (Hard Rule)

- Main session stays interruptible: never block the user behind long-running work.
- If a task is multi-step, uncertain, or may take >20 seconds: spawn a background sub-agent.
- Long tasks must post milestones every 5–10 minutes.
- If user changes direction: stop the current branch immediately.

### Sub-agent brief template (required)

When spawning a sub-agent, include:
- Goal
- Inputs
- Outputs
- File scope
- Constraints / risks
- Acceptance criteria

## Memory (4-layer system)

### Layers
- `MEMORY.md`: stable facts + index (keep lean)
- `memory/projects.md`: project status + next actions
- `memory/lessons.md`: reusable lessons + prevention
- `memory/YYYY-MM-DD.md`: daily conclusions (choose template by scenario)

### Writing rules (strict)
- Write conclusions, not play-by-play.
- Project status changes => update `memory/projects.md`.
- New reusable pitfall => update `memory/lessons.md`.
- Update `MEMORY.md` only when the index changes.

## Safety

- Don't exfiltrate private data.
- Don't run destructive commands without asking.
- Prefer recoverable deletes (trash) over rm.
- External actions (emails, public posts, messages) require confirmation.

## Group Chats

You are a participant — not the user's voice.
Do not leak private context in groups.
```

> 注意：这里的“>20 seconds”只是一个很激进的触发线，你可以改成 “>2 minutes / >8 minutes”，但建议保留“主会话可打断”这一条。
>
> 为什么一定要保留这条？因为用户对 AI 最大的不满之一，就是一发任务过去，主会话像死了一样没响应。可打断，意味着它像助手；不可打断，体验就更像一段堵住你的脚本。

---

## 5. 第四步：HEARTBEAT.md（低噪音，别把心跳当重活）

心跳的目的：轻量巡检、必要提醒、无事就 `HEARTBEAT_OK`。

为什么要专门强调“轻量”？因为 heartbeat 如果做重活，就会把一个本该低成本、低风险的后台动作，变成高噪音、高干扰、还容易误触发的大杂烩。

创建或编辑 `HEARTBEAT.md`：

```md
# HEARTBEAT.md

## Rule
If nothing needs attention, reply exactly: HEARTBEAT_OK

## What to check (lightweight)
- urgent inbox/mentions (if configured)
- upcoming calendar events (if configured)
- critical service status signals (only if you know what to check)

## Memory maintenance policy
- Heartbeat does NOT run heavy memory maintenance.
- Heavy maintenance must be done by cron (daily check + due-date gate).
```

> 对应下篇文章的建议：重维护放 cron，不放 heartbeat。
>
> 这样拆开的原因很简单：heartbeat 追求的是“随时能跑、跑完就走”；而重维护追求的是“有计划地整理和压缩”。这两件事的节奏完全不一样，硬塞在一起只会互相拖累。

---

## 6. 第五步（可选但推荐）：QMD / memorySearch 语义检索

这一步是“从能用到离不开”的关键体验：**先检索，再回答**。

为什么它这么重要？因为没有检索时，AI 面对“上次那个 nginx 问题怎么弄的”这类问题，很容易开始“凭感觉补全”。而一旦先检索，它回答时就更像在翻你的资料，而不是在现场编故事。🔍

### 6.1 你需要先接受一个事实：Embedding 不是凭空来的

语义检索需要 embedding 模型（向量化）。你有三条路：

1. **远程 embedding API（最省事）**：比如 SiliconFlow 的 `BAAI/bge-m3`（参考文章推荐）。
2. 你自己的 embedding 服务（自建 / 本地运行）。
3. 暂时不做语义检索，只靠分层结构 + 手动查文件（也能用，但体验差一截）。

本指南不会强迫你必须开启 memorySearch；会提供“缺啥就跳过”的手动策略。

为什么这里不强推？因为很多人会被“高级功能”这四个字诱惑，结果把基础结构都没打稳，就先去折腾 embedding。现实里更稳的顺序恰恰相反：先把记忆结构做好，再加检索，体验提升才是叠加的。

### 6.2 配置前置检查清单（你要自己确认）

在你动 runtime 之前，先确认：

- 你是否有一个可用的 embedding endpoint（`baseUrl`）
- 你是否有对应的 `apiKey`
- 你是否知道要用哪个 embedding 模型（比如 `BAAI/bge-m3`）

如果这些你都没有：
- **先跳过 memorySearch**，把前面 1–5 步做扎实。
- 等你拿到 embedding 配置后，再回来补上。

为什么建议这么做？因为没有 endpoint、key、model 的情况下硬配，只会多制造一层“看起来已经开了、实际上根本不能用”的假配置。那种状态最折磨人。

---

## 7. 第六步：安全地调 runtime（openclaw.json）——只 patch，不覆盖你的模型 / 身份

为什么这一节要格外小心？因为 runtime 一旦改错，影响的是“系统怎么跑”；而不是“文件写得好不好看”。你前面所有工作区文件都还有机会慢慢修，这里一旦覆盖错字段，可能直接把 provider、模型路由、渠道配置一起带崩。🛡️

### 7.1 先定位你的 runtime 配置文件在哪

不同机器路径不同。按优先级找：

1. 环境变量 `OPENCLAW_CONFIG_PATH` 指向的文件（如果你设置过）
2. `~/.openclaw/openclaw.json`
3. `~/.openclaw-*/openclaw.json`（多个 profile 时可能存在）

你可以用：

```bash
ls -la ~/.openclaw/openclaw.json
ls -la ~/.openclaw-*/openclaw.json 2>/dev/null || true
```

> 找不到就不要硬改。你需要先搞清楚 OpenClaw 实际在用哪个配置文件。
>
> 为什么不能“先改一个试试”？因为 profile 一多、路径一多，你很可能改到了一个完全不会被读取的文件，然后误以为配置没效果，最后越改越乱。

### 7.2 做一个 runtime 备份

```bash
cp -a <openclaw.json路径> <openclaw.json路径>.bak.$(date +%Y%m%d-%H%M%S)
```

为什么这里也要备份？因为 JSON 覆盖不像 Markdown 那么容易肉眼找回差异。备份能让你在“改出问题但不知道是哪一项导致的”时，快速退回原状。

### 7.3 你要加的 runtime 基线（建议项）

下面这些是“收益大、风险低”的调优项（来自参考文章与 `ref/openclaw.setup_instructions` 的组合）：

1. **block streaming 分块输出**：长回复分块发，体验好。
2. **compaction + memoryFlush**：防止对话压缩导致丢关键细节。
3. **ack reaction（按渠道）**：让你知道它收到消息了。

为什么先加这几项？因为它们解决的都是“日常体感问题”：

- 不分块时，长回复容易像卡住一样迟迟不出来。
- 没有 `memoryFlush` 时，长对话一压缩，前面辛苦得到的结论可能直接蒸发。
- 没有 ack 时，你会经常分不清它是没收到，还是在处理。

#### 7.3.1 推荐 JSON 片段（只新增这些字段；不要动你已有的 models / provider）

把这些字段以“合并”的方式加进去：

```json
{
  "agents": {
    "defaults": {
      "blockStreamingDefault": "on",
      "blockStreamingBreak": "text_end",
      "blockStreamingChunk": {
        "minChars": 200,
        "maxChars": 1500
      },
      "compaction": {
        "reserveTokensFloor": 20000,
        "memoryFlush": {
          "enabled": true,
          "softThresholdTokens": 4000,
          "systemPrompt": "会话接近压缩。只保存可长期复用记忆。",
          "prompt": "把可复用结论写入 memory/YYYY-MM-DD.md；若无内容回复 NO_REPLY。"
        }
      }
    }
  },
  "channels": {
    "discord": {
      "ackReaction": "🫐"
    },
    "telegram": {
      "ackReaction": "👀"
    }
  }
}
```

**重要提醒（请认真看）：**
- 你可能已经在 `channels.discord` / `channels.telegram` 里有 token、allowFrom 等配置。
  - 你要做的是“在原有对象里加一个字段”，不是覆盖整个对象。
- 你可能已经有 `agents.defaults` 里其它设置。
  - 同理，合并即可。

> 如果你不擅长手动合并 JSON：建议使用编辑器的“JSON 合并 / 格式化”功能，或先复制到一个临时文件里对比。
>
> 这里最容易犯的错，不是字段写错，而是“把整段对象复制进去以后把自己原本的配置洗掉”。所以我才一直强调：**patch，不是 replace。**

### 7.4（可选）开启 memorySearch（只有当前置条件满足时）

当你有 embedding 配置后，再把这段加入（同样是 merge）：

```json
{
  "memory": {
    "backend": "qmd",
    "citations": "auto"
  },
  "memorySearch": {
    "enabled": true,
    "provider": "openai",
    "remote": {
      "baseUrl": "https://api.siliconflow.cn/v1",
      "apiKey": "<YOUR_EMBEDDING_API_KEY>"
    },
    "model": "BAAI/bge-m3"
  }
}
```

- `memory.backend: qmd`：表示使用 QMD 作为检索后端（如果你的 OpenClaw 版本支持）。
- 如果你的版本不支持 `qmd` / 或字段名不同：请不要乱加；先保持只启用 `memorySearch`。

为什么这里要这么保守？因为语义检索这块最怕“看了别人的配置就整段照搬”。你的版本、字段名、后端支持情况只要有一点不一样，就可能让你进到一个很难排查的半坏状态。

### 7.5 你应该刻意“不做”的 runtime 修改

为了避免毁掉你已经能跑的配置：

- 不要动你的 `models`（除非你明确知道自己在干嘛）
- 不要改你的默认模型 / 路由
- 不要改你的 agent 名称 / 身份
- 不要把人格内容写进 runtime

本指南的定位是：**在不动你身份与模型选择的前提下，把核心体验补齐**。

为什么要刻意列出“不做什么”？因为很多配置事故，不是因为少做了，而是因为顺手多做了。会控制住手，本身就是一种高级能力。

---

## 8. 第七步：让“写记忆”真正发生（你要训练它按规则落盘）

做到 1–7 步以后，结构有了，但你还需要一个现实认知：

- 你写的规则不是魔法。
- AI 有时会偷懒，有时会写错位置。

所以你需要一个“手动校准期”（建议 3–7 天）。

为什么这里不能配完就撒手？因为规则写在那里，不等于模型已经稳定学会执行。你得给它一点“扶上马再送一程”的过程，不然很多规则会停留在文件里，看起来很完整，用起来却不稳定。

### 8.1 每次重要任务结束，你都加一句收尾指令

例如：

- “把今天这次结论写进 `memory/YYYY-MM-DD.md`，按 planning 模板。”
- “把这个坑写到 `memory/lessons.md`，Severity=High，并给 Prevention。”
- “更新 `memory/projects.md` 的 Next，并标注 Last Update。”

这样做的原因很简单：AI 在收尾阶段最容易偷懒，给一句明确的落盘指令，相当于把“写记忆”从可选项变成了验收动作。

### 8.2 主动检查它写得对不对

每天花 2 分钟打开：

- `memory/YYYY-MM-DD.md` 是否只有结论，没有流水账
- `memory/projects.md` 是否能看出“下一步”
- `memory/lessons.md` 是否能当“可执行规则”用

如果发现它写得烂：不要骂它，直接把标准贴回去，让它重写。

为什么要亲自看这 2 分钟？因为记忆系统一旦在早期就开始“写歪”，后面只会越积越难救。早点纠偏，成本最低。

---

## 9. 第八步（推荐）：用 Cron 做“每日检查 + 到期才重维护”的记忆维护

参考文章强调：

- heartbeat 保持轻
- 重维护用 cron
- cron 也不要每天都跑重任务，而是“每天检查一次，到期才执行”

为什么要搞这层 gate？因为“每天都重维护”听起来勤快，实际上常常是过度整理。你会消耗资源、打断节奏，还可能把本来还没沉淀好的信息过早压缩掉。

### 9.1 你需要一个状态文件

创建：`memory/heartbeat-state.json`

```json
{
  "lastMemoryMaintenance": 0
}
```

这个状态文件的作用，是让系统知道“上次认真整理记忆是什么时候”。没有它，cron 就很难做基于时间的判断，只能一股脑重复干活。

### 9.2 Cron 任务做什么

每天固定时间（比如凌晨 03:30）跑一次：

1. 读 `memory/heartbeat-state.json:lastMemoryMaintenance`
2. 如果距离上次维护 < 7 天：直接退出
3. 如果 >= 7 天：
   - 读最近 7 天日志
   - 提炼到 `projects.md` / `lessons.md`
   - 压缩无价值流水
   - 更新 `lastMemoryMaintenance`

> 具体怎么创建 cron 任务，取决于你当前 OpenClaw 版本的 cron 工具 / 配置方式。
> 如果你已经会用 `openclaw cron ...`：照下篇文章的 cron 章节照抄即可。
> 如果你不会：先别硬上，等你把前面的结构跑稳再说。
>
> 为什么这里建议“先别硬上”？因为 cron 的价值，是在系统已经稳定以后帮你省力；不是在系统还没稳定时，再多加一个会自动出错的变量。

---

## 10. 最后：验收清单（你按这张表逐条自测）

### 10.1 文件结构

- [ ] workspace 根目录存在：`AGENTS.md / SOUL.md / USER.md / IDENTITY.md / MEMORY.md`
- [ ] `memory/` 存在：`projects.md / lessons.md / YYYY-MM-DD.md`

### 10.2 行为与委派

- [ ] 你给一个 3+ 步的任务，它会主动拆分 / 委派，而不是把你卡在主会话里
- [ ] 它会在长任务里给里程碑

### 10.3 记忆落盘

- [ ] 重要任务结束后，它会把结论写入当天日志
- [ ] 项目状态变化会同步到 `projects.md`
- [ ] 坑会沉淀到 `lessons.md`

### 10.4 压缩不失忆

- [ ] 长对话后仍能复述关键结论（因为 memoryFlush 落盘）

### 10.5（可选）语义检索

- [ ] 你问“上次那个 X 怎么弄的”，它会先检索再回答（而不是编）

为什么一定要自测这一轮？因为“文件都在”不等于“行为真的对”。这张表的意义，就是把“看起来装好了”变成“真的能验收”。✅

---

## 11. 你会觉得麻烦的点（以及这就是为什么要用生成器）

手动做完你会发现，最烦的其实不是“写文件”，而是这些容易把人磨崩的小事：

1. **合并策略**：同一个 JSON / 同一个 markdown 规则，到底是 merge、patch 还是 replace？
2. **不覆盖个性化**：尤其是 SOUL / USER / IDENTITY，你一覆盖就把自己养出来的助手“抹掉”。
3. **runtime 定位**：openclaw.json 到底用哪个？profile 多了就更烦。
4. **记忆维护的工程化**：要有状态文件、要有 gate、要避免每天重跑。

这四点就是“一键生成 / 一键安装”真正卖的东西：不是模板本身，而是**方法论 + 非破坏式落地 + 可验收**。

换句话说，真正值钱的不是“给你几段文本”，而是“帮你在不踩坑的前提下，把这些文本正确放进正确的位置里”。这也是 <https://clawok.me> 这类方案存在的价值。

---

## 附录 A：一句话策略（适合贴在你主会话里随时提醒 AI）

你可以把下面这段当作你和 AI 的“工作暗号”：

- “按 AGENTS.md 执行：主会话可打断，复杂任务默认委派。”
- “按分层记忆写：结论进 daily log，项目进 projects，坑进 lessons，索引才动 MEMORY。”
- “不确定 runtime 字段就别瞎写；不要覆盖我的模型 / 身份 / 名字。”

这三句为什么有用？因为它们几乎浓缩了整篇文章最重要的行为边界。很多时候，你不需要每次都重讲全套规则，只要把这三句贴出去，AI 就知道自己该往哪条轨道上走。

---

## 附录 B：如果你想更贴近 ClawGen 的参考实现

ClawGen 的 ref 包（你可以自己对照）里，包含了一套更严格的：

- 默认委派触发器
- 多用途日志模板
- heartbeat 与 cron 分工
- runtime patch 基线

你可以在本仓库的 `ref/` 目录查看：

- `ref/openclaw.AGENTS.md`
- `ref/openclaw.HEARTBEAT.md`
- `ref/openclaw.setup_instructions.md`

这些文件本质上就是“你手动也能照着写”的版本，只是 ClawGen 会自动做：
- 文件存在性判断
- 个性化内容保护
- 合并 / patch 策略
- runtime 定位与可写性判断

如果你要纯手动复刻：就按本指南 1–10 步老老实实做完，并用 ref 文件当对照即可。

为什么最后还要看 ref？因为它们相当于你的“标准答案边界线”。你不一定要完全照抄，但至少可以拿它来判断：自己现在做的是不是已经偏离了参考实现太多。

---

## 致谢

本文整理和写作过程中参考的内容，均来自 [Linux.do](https://linux.do) 论坛。

感谢论坛里持续分享经验、方案和踩坑记录的佬友们。这篇手动指南能整理出来，离不开这些一线实践内容的启发和帮助。