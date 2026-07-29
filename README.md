# Bitto-Codex-MultiAgentPrompt

> 一套**生产级的多 Agent 协作规范**（prompt），面向 GLM-5-2 / Bitto 网关场景。
> 把本仓库的文件放进你的 Agent 框架，主 Agent 就会按规范自动拆分任务、并行派发给子 Agent、做独立审查，最后给你一份整合好的结果。

---

## 这是什么？

前沿大模型什么都能干——但要拿最贵的模型从规划写到测试再到自查一遍，**烧钱且浪费**。贵的模型该做"决策和把关"，代码执行、文件扫描、机械检查可以交给更便宜的模型并行跑。

本仓库提供的是一份**协作规则书**（`AGENTS.md`）。它告诉主 Agent：

- 什么情况下必须把任务**拆给多个子 Agent 并行做**（而不是自己闷头干）；
- 三个子 Agent 各自的**专长与边界**（谁擅长写代码、谁擅长挑错、谁擅长啃硬骨头）；
- 子 Agent 之间**怎么传信、怎么确认、怎么防卡死**；
- 出问题了**怎么回退、怎么换人、怎么上报**。

你不需要理解里面所有细节——**把它当成一个"会自己带团队的 Agent 说明书"丢进项目即可**。规范本身用的是 v5 mailbox 协议（基于 Git 目录的确定性文件系统信箱），稳定、可审计、不依赖特定平台。

---

## 文件地图

| 文件 | 是什么 | 给谁用 |
|---|---|---|
| `AGENTS.md` | 多 Agent 协作规范（v5 完整版，44K） — 含信箱协议、心跳机制、审查体系等全套工程保障 | **首选**。大多数 Agent 框架直接读项目根目录的 `AGENTS.md` |
| `multi-agent-spec-bitto.txt` | Bitto 网关适配版（14K） — 精简了 v5 信箱协议层，保留核心派单/回退/路由逻辑 | 粘贴到 Bitto 网关等 **不支持 Markdown** 的 System Prompt 输入框使用 |

**两个文件是同一套规范的两个版本，不是完全相同**：`AGENTS.md` 更完整、更工程化（v5 信箱协议、确定性哈希、版本化回执）；`.txt` 版去掉了网关环境不适用的底层通信协议，更适合粘贴到纯文本输入框。根据你的使用方式**二选一**即可。

---

## 推荐搭配：多模型路由工具

本规范需要三个可用模型（deepseek-flash / deepseek-pro / GLM-5-2）。如果你用 **Codex** 作为 Agent 运行环境，又想把多个模型源（官方 OpenAI 订阅、DeepSeek、GLM、Qwen、本地模型等）统一挂到同一个 Codex Provider 下面，推荐搭配：

👉 **[CCSwitchMulti](https://github.com/BigStrongSun/ccswitchmulti)** —— 面向 Codex 的多模型路由与 Provider 管理桌面工具（Tauri 跨平台，MIT 协议）。

它做的事：在 Codex 后面加一个**本地多模型路由 Provider**。你登录 ChatGPT 订阅、添加 DeepSeek / GLM / 本地模型源后，Codex 只连本地代理，由本地按 `model` 名字把请求分发到不同上游。效果：

- 主 Agent 用官方强模型做规划/决策/质量把关；
- 执行、审查等可拆分任务路由到廉价 API、本地模型或 DeepSeek / GLM，实测可**省下约一半官方额度**；
- 三个子 Agent 的模型名（deepseek-flash / deepseek-pro / glm-5-2）直接在路由表里映射好即可。

使用要点：保持 CCSwitchMulti 运行；改完路由规则后**完全退出并重开 Codex Desktop**；模型菜单不显示时用它的"模型菜单解锁流程"启动（运行时注入，不改 app.asar）。中文全流程见 [Codex 多路由使用说明](https://github.com/BigStrongSun/ccswitchmulti/blob/main/docs/guides/codex-multirouter-guide-zh.md)。

> 注意：本仓库的规范只定义"怎么派活"，模型从哪来、怎么路由由你的平台/网关决定。CCSwitchMulti 是其中一种（Codex 场景）落地方式，不是必须。

---

## 3 分钟看懂核心概念

不需要记住术语，先有个画面：

```
                  你的一句话需求
                        │
                        ▼
                 ┌─────────────┐
                 │   主 Agent   │  ← 项目经理：拆任务、派活、收尾、整合
                 │  (primary)  │
                 └──────┬──────┘
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
  ┌──────────┐   ┌──────────┐    ┌──────────┐
  │ deepseek │   │ deepseek │    │  GLM-5-2 │
  │  -flash  │   │   -pro   │    │          │
  │ 干活/扫描│   │  挑错/诊断│    │ 啃硬骨头 │
  └──────────┘   └──────────┘    └──────────┘
   并行子 Agent（互不 spawn，只听主 Agent 指挥）
```

- **主 Agent（primary）**：项目经理。负责把任务拆成清晰的工单、派给子 Agent、等他们交活、做验收、最后把结果揉成一份完整答复。**它自己基本不写业务代码。**
- **三个子 Agent**（都是平级、第一层、不能再开小号）：
  - **deepseek-flash**：执行主力。定义明确的功能、跨模块对接、前端页面、测试用例、CI/文档、仓库组装——都由它来写。
  - **deepseek-pro**：只读审查员。专门找问题：架构隐患、安全漏洞、失败路径、边界情况。**它只挑错，不写实现。**
  - **GLM-5-2**：高级攻坚手。遇到模糊、跨系统、难搞的可视化、长上下文、高风险或需要"救场"的实现，交给它。
- **信箱（mailbox）**：主 Agent 和子 Agent 之间不靠"聊天记录"传话，而是靠仓库 `.git` 目录里的一份**结构化工单文件（envelope）**来通信。每件事有唯一编号、版本号、内容指纹，确保不丢、不重、不串。
- **触发条件（M1–M7）**：规范里列了 7 类"必须拆任务"的场景（比如：涉及整个项目、跨多个模块、要做发布、要写前端页面……）。命中任意一条，主 Agent 就必须走正式派单流程，不能自己偷偷干。
- **回退（fallback）**：flash 连续两次搞不定同一块，主 Agent 会把这块转交给 GLM-5-2 救场；再不行才由主 Agent 自己兜底（并如实说明）。

---

## 怎么用（两种方式）

### 方式 A：作为项目里的 `AGENTS.md`（最推荐）

绝大多数现代 Agent 框架（Codex、Claude Code、Cursor、CodeBuddy、各类自托管 Agent 网关等）在启动时会**自动读取项目根目录的 `AGENTS.md`** 作为行为准则。

1. 把本仓库的 `AGENTS.md` 下载/复制到你的**项目根目录**：
   ```bash
   # 在你的项目根目录执行
   curl -O https://raw.githubusercontent.com/zhushihao/Bitto-Codex-MultiAgentPrompt/main/AGENTS.md
   ```
2. 确保你的 Agent 框架里配置了三个可用模型：**deepseek-flash、deepseek-pro、glm-5-2**（或通过网关映射成对应名字）。
3. 正常给 Agent 下任务即可。它会按规范自行决定要不要拆、派给谁、怎么验收。

> 如果你用的框架不读 `AGENTS.md`，看方式 B。

### 方式 B：作为 system prompt 粘贴

1. 打开 `multi-agent-spec-bitto.txt`（纯文本，无格式符号），全选复制。
2. 粘贴到你的 Agent / 网关的 **System Prompt（系统提示词）** 输入框。
3. 保存并起一个会话，直接下任务。

> 提示：Bitto 网关等非富文本环境用 `.txt` 版最稳；本地 IDE 类工具用 `AGENTS.md` 版即可。

---

## 快速上手示例

**场景**：你有一个前端项目，想加一个带筛选的数据表格页面。

1. 项目根目录已有 `AGENTS.md`。
2. 你下任务：*“在 src/pages 下新建一个 DataGrid 组件，支持按状态筛选，要响应式，桌面和手机都能看。”*
3. 规范判定命中 **M7（非平凡前端）** → 主 Agent 派单：
   - `deepseek-flash`（xhigh）写组件 + 样式 + 基本自测；
   - `deepseek-pro`（medium）对成品做正确性审查；
   - 若涉及跨域/复杂可视化难点，`GLM-5-2`（high）会被拉来攻坚。
4. 主 Agent 等两边交活 → 做验收（桌面/手机渲染、交互、控制台报错）→ 给你**一份整合好的结果**，而不是三段零散汇报。

你全程只说了那一句话。

---

## 模型路由速查（主 Agent 内部逻辑，你一般不用管）

| 任务长这样 | 派给谁 |
|---|---|
| 写功能、接模块、写测试、写文档、组装仓库 | `deepseek-flash` (xhigh) |
| 找 bug、审架构、查安全、挑毛病 | `deepseek-pro` (medium，只读) |
| 模糊/跨系统/难可视化/高风险/救场实现 | `GLM-5-2` (high) |
| 只是扫描、盘点、抽证据、机械检查 | `deepseek-flash` (medium) |

> 不确定就用更便宜能干的那个，证据不足再升级——这是规范内置的策略，你不用手动指定。

---

## 小白最常见疑问（FAQ）

**Q：我完全不懂 prompt 工程，能用吗？**
能。你只要把 `AGENTS.md` 放进项目根目录，剩下的交给框架和这份规范。它写的就是"Agent 该怎么带团队"。

**Q：三个子 Agent 我都要自己申请/付费吗？**
是的，你需要这三个模型在你的 Agent 框架或网关里可用（名称映射成 deepseek-flash / deepseek-pro / glm-5-2）。规范只负责"派活"，模型本身由你的平台提供。

**Q：怎么把多个模型源（官方 OpenAI / DeepSeek / GLM / 本地模型）统一挂到 Codex 下面？**
推荐搭配 **[CCSwitchMulti](https://github.com/BigStrongSun/ccswitchmulti)**（基于 CC Switch 的 Tauri 跨平台桌面工具）。它在 Codex 后面加了一个**本地多模型路由 Provider**：你登录 ChatGPT 订阅 + 添加 DeepSeek / GLM / 本地模型源后，Codex 只连本地代理，由本地按 `model` 名字把请求分发到不同上游。这样主 Agent 用官方强模型、执行/审查任务路由到廉价或本地模型，**实测可省下约一半官方额度**。使用要点：保持 CCSwitchMulti 运行、改完路由规则后完全退出并重开 Codex Desktop、模型菜单不显示时用它的"模型菜单解锁流程"启动。详见其 [Codex 多路由使用说明（中文）](https://github.com/BigStrongSun/ccswitchmulti/blob/main/docs/guides/codex-multirouter-guide-zh.md)。

**Q：`AGENTS.md` 和 `multi-agent-spec-bitto.txt` 用哪个？**
AGENTS.md 是完整版（含 v5 信箱协议），适合放进项目根目录让框架自动读取；`.txt` 是精简版，适合粘贴到 Bitto 网关等纯文本 System Prompt 输入框。用哪个取决于你的使用方式，不要两个都塞。

**Q：子 Agent 会互相乱发任务、无限套娃吗？**
不会。规范强制所有子 Agent 都是第一层、禁止再开子 Agent（"never spawns"）。只有主 Agent 能派活、能发起审查。

**Q：任务卡住了怎么办？**
规范里有"inactivity window（活动窗口）"机制：主 Agent 按难度给每个子 Agent 一个合理的等待窗口，到期还没动静才会探活/判定失联，不会一慢就杀。真卡死会按回退链换人并如实告诉你。

**Q：它会不会偷偷改我项目里不相干的文件？**
不会。规范把"保留用户和其他人的改动"列为硬约束（I7），任何冲突都先上报，不经你允许不回退。

**Q：这和原 `myprompt` 仓库什么关系？**
`Bitto-Codex-MultiAgentPrompt` 是从 `myprompt` 的 `Business/`（GLM-5-2 / Bitto 版）单独拆出来的**公开版本**，方便直接拿来用，不含私有（k3）配置。

---

## 术语表（随手查）

| 词 | 人话 |
|---|---|
| primary（主 Agent） | 项目经理，负责拆活、派活、收尾 |
| sub-agent（子 Agent） | 被派活的执行者，平级、第一层 |
| envelope（工单） | 主 Agent 写给子 Agent 的结构化任务单，落在 `.git` 目录里 |
| mailbox（信箱） | 存放工单和回执的文件目录，Agent 间靠它通信 |
| M1–M7 | 7 类"必须正式拆任务"的触发场景 |
| GATE_RECORD | 正式派单前必须公示的一张"任务分配表" |
| fallback（回退） | 一个 Agent 搞不定时的换人/救场流程 |
| TAINTED | 被标"受污染/不可信"的半成品，下游不能直接用 |
| reasoning_effort | 模型思考强度（medium / xhigh / high），主 Agent 派活时设定 |

---

## 进阶：信箱协议简介（不用懂也能用）

规范用的是 **v5 TASK_ENVELOPE** 协议，核心要点：

- 工单路径基于 `GIT_COMMON_DIR`（Git 公共目录的绝对路径），不依赖工作区根，避免多工作树串台。
- 每封工单有：**协议版本、任务 ID、版本号、内容指纹（body_hash）、目标 Agent、范围、排除项、交付物、验收标准**等 27 个必填字段。缺一个，子 Agent 直接写 `BRIEF_INVALID` 退出，绝不瞎干。
- 所有回执（ACK / 心跳 / 结果 / 无效）都带**五元组身份**（task_id, revision, body_hash_confirmed, gate_id, scope_version），保证"对的是同一个任务、同一版内容"。
- 主 Agent 用"临时文件 + 落盘 + 原子替换 + 往返校验"发布工单，杜绝写到一半被读到。

普通使用者无需关心这些——它们是为了**稳定、可审计、可复现**而存在的工程保障。

---

## 更新日志

- 2026-07-29：从 `myprompt` 的 `Business/`（GLM-5-2 / Bitto 版）拆出，独立为公开仓库 `Bitto-Codex-MultiAgentPrompt`；随附面向小白的完整 README。
- 规范内核：v5 mailbox 协议（GIT_COMMON_DIR 信箱、规范 JSON、版本化回执）；适配模型 deepseek-flash / deepseek-pro / GLM-5-2。

---

## License

本仓库内容按原 `myprompt` 仓库约定使用；如需明确授权协议，请在 fork / 使用前与作者确认。
