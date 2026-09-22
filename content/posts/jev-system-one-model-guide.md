+++
title = "Jev 深度解析：ChatGPT 共同作者做了个不生成文本的模型，附 Hermes / Codex 接入实战"
date = "2026-09-22T21:30:00+08:00"
draft = false
toc = true
description = "TypeSafe AI 在 2026 年 9 月发布的 Jev 不是聊天模型，而是专门给软件做结构化决策的 System One 模型：输入状态、输出带校准概率的类型安全结果。本文讲清它的原理、与 LLM 的本质差异、七个真实应用场景，并给出把 Jev 接进 Hermes Agent 和 Codex CLI 的完整可执行方案（MCP 服务 + Codex PreToolUse 风险闸门）。"
summary = "Jev 不写一个字，却在分类、路由、审查这类判断任务上比前沿大模型快 40-200 倍、便宜几百倍。它把 AI 从「生成文字」拉回「做决策」。这篇讲清楚它是什么、能干什么，以及怎么真正接进你的 Hermes 和 Codex 工作流。"
categories = [
    "AI知识技能",
]
tags = [
    "Jev",
    "TypeSafe",
    "System One",
    "RLCD",
    "MCP",
    "Hermes",
    "Codex",
    "AI Agent",
]
+++

## 先说结论

Jev 是一类"不会说话的模型"：你给它一段上下文（state）和几个问题（questions），它直接返回带概率的类型安全答案，不生成任何文字。

三句话版本：

- **它解决的是"AI 在代码里不好用"的问题**，不是"AI 不够聪明"的问题。字符串灵活但贵、慢、需要解析验证，还会跑偏；Jev 干脆放弃生成文字。
- **它的定位是"智能 if 语句"**：分类、路由、打分、审查，插在普通软件里当模糊判断的分支条件。
- **它不能替代你的主模型**，但能让主模型少干脏活。官方数据是同类判断任务快 40-200 倍、便宜两个数量级。

本文后半部分给出把 Jev 真正接进 Hermes Agent 和 Codex CLI 的完整方案，包括一个可以直接跑的 MCP 服务和 Codex 的命令风险闸门，代码我都验证过了。

## 一、Jev 是什么：把"判断"从"生成"里拆出来

Jev 由 TypeSafe AI 在 2026 年 9 月 15 日发布。这家公司 2024 年成立，隐身两年后出关，创始人是 Diogo Almeida——前 OpenAI 研究员、InstructGPT 论文共同作者，也就是 RLHF 那条技术路线的核心参与者之一。公司在发布同期宣布拿到 4000 万美元种子轮。

有个细节很能说明他们的思路：Almeida 说他在 OpenAI 做完 RLHF、眼看着聊天模型超越人类水平之后，一直在问自己一个问题——**既然对话模型这么强，为什么自动化还是没发生？**

他的答案是：语言模型为"跟人交流"优化，而计算机系统之间的交互根本不用自然语言。让一个 LLM 嵌在生产代码里做判断，你要忍受三件事：

1. 它对每个决策都要生成一串 token，慢且贵；
2. 它的输出需要解析、校验，还有一种"类型错误"的风险；
3. 它给的置信度基本不能信，过度自信且不稳定。

于是有了 System One 模型这一类东西。名字来自卡尼曼《思考，快与慢》里的系统一：快速、直觉、不做长篇推理。

Jev 的几块技术底座：

- **RLCD（Reinforcement Learning for Calibrated Decisions）**：不是 RLHF（对齐人类偏好），也不是 RLVR（可验证奖励），而是专门训练模型输出"校准过的概率"——置信度高的时候准确率真的更高，这样程序才敢拿阈值做自动分支。
- **并行采样**：传统 LLM 逐 token 自回归生成；Jev 在单次查询里把所有答案一起算出来。这是速度量级差异的来源。
- **类型安全**：可能的输出空间由你事先定义，模型不会产生类型错误，官方直接标注 0%。这不是"实测误差为 0"，而是 schema 匹配有数学保证。
- **只用合成数据训练**，且不在客户请求上训练，权重对所有账号一致，不能微调——你要定制，就通过 state 和 instructions 来"喂"它。

## 二、和 LLM 的本质差别

官方给了一张对照表，我按自己的理解重排了一下：

| 维度 | 传统 LLM | System One / Jev |
| --- | --- | --- |
| 训练方式 | RLHF / RLVR | RLCD |
| 优化目标 | 人类偏好、可验证奖励 | 校准决策 |
| 输入侧重 | 对话消息序列 | 结构化程序状态 |
| 输出 | 字符串（需要解析和校验） | 类型安全的结构化值 |
| 采样 | 顺序、逐 token | 并行、一次出全部 |
| 价格 | 输入 $0.20–10 / Mtok，输出约为输入的 5 倍 | 输入 $0.042 / Mtok，输出免费 |
| 延迟 | 前沿模型 3–329 秒 | 70–500 毫秒 |
| 置信度 | 过自信、不一致 | 每次输出都带校准概率 |
| 典型场景 | 人在环路的对话、写作、编码 | 工作流分支、大数据打标、实时应用、验证与护栏 |

注意最后一行：Jev 不是"更小的 LLM"。它放弃了字符串生成这个能力，换来的是速度、成本和确定性。它**不能**写文案、不能写代码、不会解释理由——官方 FAQ 里这条是明说的。

价格上还有个容易被忽略的点：**按输入计费、输出完全免费**。这意味着你可以往 state 里塞大量上下文（合同、日志、用户历史），只要问题设计得好，边际成本低到可以忽略——这正好是 map-reduce 打标的理想形态。

## 三、接口长什么样

一个 HTTP 端点：`POST https://api.typesafe.ai/v1/systemone`。请求只有两块：state 和 questions。

提问用三种原语：

- **noul**：是/否判断，返回成立的概率（例：这条消息是否表达紧迫性）
- **choice**：从你定义好的选项里选一个，返回选项概率分布 + 置信度（例：该分给哪个团队）
- **score**：按有序档位打分，返回连续分数和分布（例：客户愤怒程度 0-2）

一次请求可以并行问多个问题。真实请求体长这样：

```bash
curl -X POST https://api.typesafe.ai/v1/systemone \
  -H "Authorization: Bearer $TYPESAFE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "jev-latest",
    "state": "用了三天还是连不上 Stripe，一直在丢单，麻烦尽快处理！",
    "questions": {
      "department": {
        "type": "choice",
        "instructions": "这个工单该由哪个团队处理",
        "criteria": {
          "billing": "付款或订阅问题",
          "technical": "缺陷或集成问题",
          "sales": "价格或账号问题"
        }
      },
      "is_urgent": {
        "type": "noul",
        "instructions": "这条消息是否表达了紧迫性"
      }
    }
  }'
```

返回是纯结构化的：

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "department": {
      "type": "choice",
      "choice": "technical",
      "confidence": 0.78,
      "probabilities": { "technical": 0.85, "billing": 0.15, "sales": 0.0 }
    },
    "is_urgent": { "type": "noul", "noul": 0.999, "confidence": 0.9 }
  }
}
```

`0.85` 和 `0.999` 就是它的全部价值：你的代码可以直接写 `if urgent > 0.9: 优先处理`，不需要任何解析，也不会拿到"抱歉，我无法判断"这种话。

当前模型规格：`jev-1.13.0`（别名 `jev-latest` / `jev-preview`），上下文 64k（state 加最长问题限 32k），限流 250k tokens/秒、1200 请求/分钟，而且官方明确说限流因为需求太猛会动态调整——发布当天 API 一度被打到服务不过来。（以上规格与价格截至 2026 年 9 月 22 日，以官网为准。）

## 四、它能干什么：七个已经跑起来的场景

我把官方和社区的案例整理成七类，前四类是最容易落地的。

**1. 智能 if 语句 / 工作流分支**
把手写的脆弱规则（`if "退款" in text`）换成带概率的模糊判断。适合"规则写起来太啰嗦、但又不值得叫人看"的中间地带。

**2. 分类与路由**
客服工单分流是官方演示场景。Bryo AI 的 CTO 拿它和 Gemini 做业务邮件分类对比：Gemini 略准一点，但成本贵 10-20 倍；而 Jev 返回的置信度可以直接触发自动化工单流转——按他的说法，这是"唯一真正返回概率"的方案。

**3. 护栏与安全审查（最实用的一个）**
Vercel 的工程师原来用一个大模型检查命令安全性，切到 Jev 后响应快了 5-18 倍，准确率还提高了。同样的思路可以做 prompt 注入检测、越狱检测、agent 行为审计。LangChain 已经把它做成了 `AutoModeMiddleware`：在工具调用真正执行前拦一道。

**4. 模型路由**
用 Jev 先看任务难度，再决定派给便宜模型还是贵模型。Earendil 的 CTO Armin Ronacher 专门提到这点。LangChain 的 `ModelRouterMiddleware` 是现成实现。

**5. 大数据 map-reduce 打标**
按输入计费、输出免费、并行出结果——把海量文本变成特征和洞察，这类活它比 LLM 便宜一到两个数量级。

**6. 实时应用**
官方 demo 里有个用 Jev 打 Doom 的：10 次调用/秒，一小时 7 美元。还有 Wikiracing（在维基百科链接间跳转找目标页），每一步要在几百到几千个链接里选，考的就是"高基数选择下不幻觉"。

**7. 复核 LLM 的输出**
让 Jev 当裁判，给另一个模型的输出打分或做校验——这比让模型自己检查自己靠谱得多。

## 五、怎么接进 Hermes 和 Codex

先说一个必须先讲清的坑：**Jev 不是 OpenAI 兼容的 chat completions 接口**。它的端点是 `/v1/systemone`，请求体是 state + questions 而不是 messages。所以你不能把它填进 Hermes 的 `model.default`，也不能填进 Codex 的 `model`——那样必然报错。

正确的做法是把它包成一个 **MCP 服务**。MCP 是 Hermes 和 Codex 都原生支持的扩展方式，写一次两边都能用。

### 5.1 写一个 Jev MCP 服务

环境准备：

```bash
mkdir -p ~/proj/jev-mcp && cd ~/proj/jev-mcp
pip install mcp httpx     # 建议在 venv 里装
```

服务主体（完整可运行，我实际跑过握手和工具调用）：

```python
#!/usr/bin/env python3
"""把 Jev 暴露成 MCP 工具。"""
import os, sys
sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
from jev_client import jev_call, route, screen_command

try:                      # mcp >= 2.0
    from mcp.server.mcpserver import MCPServer
except ImportError:       # mcp 1.x
    from mcp.server.fastmcp import FastMCP as MCPServer

server = MCPServer(
    name="jev",
    version="0.2.0",
    instructions=(
        "Jev 返回结构化决策和校准置信度,不生成文本。"
        "适合做:命令风险判定、请求分类与路由、打分排序、复核结论。"
    ),
)

@server.tool(description="判定一条 shell 命令的风险等级(safe/review/danger)")
def jev_screen_command(command: str, context: str = "") -> dict:
    return screen_command(command, context)

@server.tool(description="为任务挑选最合适的模型档位")
def jev_route(task: str, candidates: dict) -> dict:
    return route(task, candidates)

@server.tool(description="通用判定:state + questions(支持 noul/choice/score)")
def jev_decide(state: str, questions: dict) -> dict:
    return jev_call(state, questions)

if __name__ == "__main__":
    server.run()   # 默认 stdio
```

底层调用和三个"决策菜谱"放在 `jev_client.py` 里，核心就一个函数：

```python
def jev_call(state, questions, timeout=25.0):
    if MOCK:                       # JEV_MOCK=1 时离线自测,不发请求
        return _mock(state, questions)
    resp = httpx.post(
        API_URL,
        headers={"Authorization": f"Bearer {API_KEY}",
                 "Content-Type": "application/json"},
        json={"model": MODEL, "state": state, "questions": questions},
        timeout=timeout,
    )
    resp.raise_for_status()
    return resp.json()
```

装上 `mcp` 包之后，用官方客户端库可以自测（不消耗 API 额度）：

```bash
JEV_MOCK=1 python3 test_smoke.py
# server: jev 0.2.0
# tools: ['jev_screen_command', 'jev_route', 'jev_decide']
# SMOKE OK
```

整个项目一共四个文件，放在 `~/proj/jev-mcp/` 下就行：

```text
~/proj/jev-mcp/
├── jev_client.py          # 共享逻辑:HTTP 调用 + 三个决策菜谱
├── jev_mcp_server.py      # MCP 服务(Hermes / Codex 共用)
├── codex_pre_tool_use.py  # Codex PreToolUse 风险闸门
└── test_smoke.py          # MCP 链路冒烟测试
```

### 5.2 接进 Hermes Agent

Hermes 有原生 MCP 客户端，在 `~/.hermes/config.yaml` 里加一段就行：

```yaml
mcp_servers:
  jev:
    command: "/home/scott/.hermes/hermes-agent/venv/bin/python"
    args: ["/home/scott/proj/jev-mcp/jev_mcp_server.py"]
    env:
      TYPESAFE_API_KEY: "你的 key"
    timeout: 60
    connect_timeout: 30
```

三个实测细节：

1. **API key 必须写在 `env` 里**。Hermes 启动 MCP 子进程时只继承 `PATH`、`HOME` 这类白名单变量，你 shell 里的环境变量不会自动传进去——这是刻意的防泄漏设计。
2. **工具名会带前缀**。上面三个工具在 Hermes 里叫 `mcp_jev_jev_screen_command` 这样，`mcp_{服务名}_{工具名}`。
3. **改完要重启 Hermes**，MCP 服务目前没有热加载。

接好之后，Jev 就是 agent 手上的一件工具。但工具不会自己找活干，你还要在 `AGENTS.md` 或 skill 里写明使用时机，例如：

```text
处理邮件/工单分流时,先用 mcp_jev_jev_decide 判断 urgency 和 category,
把概率写进决策依据,不要用主模型凭感觉分类。
执行可能破坏数据的 shell 命令前,先调 jev_screen_command 看危险概率。
```

顺带说清楚 Hermes 自身的一个限制：Hermes 的 `approvals.mode: smart` 已经在用辅助模型评估危险命令，但配置里没有指定审批模型的选项，**不能直接换成 Jev**。所以在 Hermes 里，Jev 的定位是"agent 主动调用的判断工具"，不是自动闸门。想要强制的闸门效果，用下面的 Codex hook 方案，或者让 cron 任务里的初筛走 Jev。

### 5.3 接进 Codex CLI

Codex 同样支持 MCP，配置在 `~/.codex/config.toml`：

```toml
[mcp_servers.jev]
command = "python3"
args = ["/home/scott/proj/jev-mcp/jev_mcp_server.py"]
startup_timeout_sec = 20
tool_timeout_sec = 60

[mcp_servers.jev.env]
TYPESAFE_API_KEY = "你的 key"
```

也可以用命令行添加，省得手写 TOML：

```bash
codex mcp add jev --env TYPESAFE_API_KEY=你的key -- python3 ~/proj/jev-mcp/jev_mcp_server.py
codex mcp list
```

进会话后用 `/mcp` 能看到服务是否连上。

### 5.4 Codex 的杀手级用法：给命令装一道 Jev 闸门

Codex 有 lifecycle hooks，其中 `PreToolUse` 可以在工具调用执行**之前**拦截它。这就是 Vercel 那个用法的开源版：本地自己搭一个带概率的命令审查。

hook 的协议很简单：

- stdin 收到 JSON，里面有 `tool_name`、`tool_input`、`cwd` 等字段
- **exit 0** 放行（stdout 还能返回 JSON 追加上下文）
- **exit 2** 拦截，stderr 的内容会作为拦截原因回传给模型

脚本核心（完整版在 `~/proj/jev-mcp/codex_pre_tool_use.py`）：

```python
def main() -> int:
    payload = json.loads(sys.stdin.read() or "{}")
    if payload.get("tool_name") not in {"Bash", "shell", "exec_command"}:
        return 0
    command = (payload.get("tool_input") or {}).get("command", "")
    if not command.strip():
        return 0

    try:
        ans = screen_command(command, context=payload.get("cwd", ""))
    except Exception as exc:            # 判定失败默认放行(fail-open)
        print(f"[jev] 判定失败,放行: {exc}", file=sys.stderr)
        return 0

    risk = ans.get("risk_level", {})
    p_danger = float((risk.get("probabilities") or {}).get("danger", 0.0))
    if risk.get("choice") == "danger" and p_danger >= BLOCK_THRESHOLD:
        print(f"Jev 判定为高风险(危险概率 {p_danger:.2f}):{command[:200]}", file=sys.stderr)
        return 2                        # 拦截
    return 0                            # 放行
```

注册到 Codex：

```toml
[[hooks.PreToolUse]]
matcher = "^Bash$"

[[hooks.PreToolUse.hooks]]
type = "command"
command = "/usr/bin/python3 /home/scott/proj/jev-mcp/codex_pre_tool_use.py"
timeout = 30
statusMessage = "Jev 正在评估命令风险"
```

有个行为要注意：**Codex 会记录 hook 的哈希并要求你确认信任**，新加或改动过的 hook 会被跳过直到你批准，可以用 `/hooks` 查看状态。

## 六、我实测了什么

为了确认这套东西真能跑，我在本机做了两轮验证。需要先说明：TypeSafe 还在 early access，笔者尚未拿到 API key，所以真实 API 的返回细节来自官方文档，而接入链路本身是用 `JEV_MOCK=1` 离线模式（不发网络请求、不消耗额度）打通的——协议、工具注册、hook 的分支逻辑都是真的跑过的。

**MCP 服务链路**——官方 MCP 客户端握手 → 发现 3 个工具 → 逐个调用 → 拿到结构化返回：

```text
server: jev 0.2.0
tools: ['jev_screen_command', 'jev_route', 'jev_decide']
SMOKE OK
```

**Codex hook 的四个分支**：

```text
危险命令 + 默认阈值 0.8   → exit 2,stderr 输出拦截原因  ✅ 拦截
阈值调到 0.99            → exit 0                      ✅ 放行
非 shell 工具(apply_patch) → exit 0                      ✅ 放行
非法 JSON 输入            → exit 0                      ✅ 放行
```

链路是通的。真正接生产前你还需要申请的是一件事：**TypeSafe 的 API key**。Jev 还在 early access 阶段，去 console 注册拿 key，然后把我脚本里的 `JEV_MOCK` 去掉就能打真实请求。

## 七、局限与坑

写这篇之前我把官方文档翻了一遍，几个必须知道的限制：

- **中文不是强项**。文档明确说英文是主要训练语言、准确率最好，包括中日韩在内的其他语言"能处理但不同等好"，建议非英文负载先在自己的数据上测，并且特别关注置信度。这条对我们做中文任务的人很关键——别拿中文数据直接上生产。
- **只吃文本**。图片、音频、视频都不支持（官方标了个"yet"）。多模态输入得先转成文本或结构化字段塞进 state。
- **上下文有限**。单请求 64k，且 state 加最长问题只能用 32k。塞长文档要会做切分。
- **按输入计费是双刃剑**。输出免费听着爽，但 state 越大越贵。合理的做法是把 state 攒得值——一次塞进足够多的相关信息，并行问很多问题，而不是单问。
- **校准是群体层面的**。文档自己也提醒：概率校准是在一批预测上统计出来的，不保证单次答案正确。所以别把 0.95 当"必然正确"，它是"阈值可靠"。
- **不给理由**。它不会解释为什么这么判。要可解释性，得自己在 state 和 instructions 上做文章。
- **限流会抖**。官方说由于需求量大，限流在动态调整，可能随时变。批量任务要按 429 做退避重试（官方 SDK 默认带退避）。
- **不要微调幻觉**。Jev 不支持用你的数据微调/LoRA，定制只能靠 prompt 侧的 state 和 instructions。

## 八、我的判断

Jev 值得试的场景，有这么几个特征：**高频、判断型、有明确选项、需要概率做阈值**。比如每天几百封邮件的分流、每次工具调用前的风险闸门、大批量文本打标、agent 内部的模型路由。

不适合的场景也很清楚：让它写东西、让它解释、要求它对中文做精细语义判断、或者你只是偶尔做一次复杂的开放推理——这些场景它要么做不了，要么优势体现不出来。

最后回到那句最有价值的表述，来自 TypeSafe 自己的官网：

> 把 Jev 想成一次前沿智能的函数调用：非结构化状态进，类型化概率决策出。

这句话其实定义了一种新的分工。以前我们把所有事都扔给一个全能聊天模型，代价是慢、贵、还不可靠；Jev 这类模型提示了另一条路——**让语言模型负责跟人打交道，让决策模型负责跟代码打交道**。对做 agent 的人来说，这个分工可能比模型本身更重要。

## 参考

- TypeSafe AI 官方博客：[Introducing System One Models and Jev](https://typesafe.ai/blog/introducing-system-one-models-and-jev)
- 官方文档：[docs.typesafe.ai](https://docs.typesafe.ai/)（Models / Quick start / System One 概念）
- LangChain：[What Is Jev? A Guide to TypeSafe AI's System One Model](https://www.langchain.com/blog/building-a-harness-with-jev)
- Hermes Agent 文档：[hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/)（MCP 配置见 MCP 章节）
- Codex 文档：[Hooks](https://developers.openai.com/codex/hooks) / [MCP](https://developers.openai.com/codex/mcp/)
