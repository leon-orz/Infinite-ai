# AI / Agent 提效方法调研（2026-11）

> 调研时间：2026-11-23 ~ 2026-11-24（Asia/Shanghai）
> 方法：7 路并行联网检索 + 一手来源复核；star 数经 `img.shields.io` 独立通道抓取
> 语言约定：对话与文档用简体中文，标识符与协议字段保持英文原拼写

## 0. 阅读说明

**证据分级**（本文所有条目都带这个标记，请务必区分）：

| 标记 | 含义 |
| --- | --- |
| ★★★ | 一手核实：本轮实际打开了官方文档 / 原始论文 / 规范原文 |
| ★★ | 二手转述：来源可信（厂商博客、权威媒体、HN 原帖），但是转述或厂商自报数字 |
| ★ | 未能核实：只见到搜索摘要或标题，**不应作为结论引用** |

**本轮检索环境的重要限制**（会影响可核实范围，记录下来避免重复踩坑）：

- `api.github.com`：对本出口 IP 全端点 403 rate limit（未认证配额被打满）
- `lite.duckduckgo.com` / `html.duckduckgo.com`：返回 202 人机验证（选鸭子）
- `mojeek.com`、`ecosia.org`：403；`search.brave.com`：429；`bing.com`：返回无关结果；`api.star-history.com`：403
- `bloomberg.com`：403 反爬；`zhihu.com`：403 登录墙；`gartner.com`：403
- **唯一稳定可用的替代通道**：`https://img.shields.io/github/stars/<owner>/<repo>.json`（取 star 数）
  与 `https://hn.algolia.com/api/v1/search`（取 HN 讨论）——记下来，下次直接用

**本文的核心结论一句话**：2026 年真正稳定的提效来源几乎全是**减法和工程纪律**（约定文件、计划先行、门禁用钩子、上下文做减法、工具做减法、沙箱常开），而绝大多数"新概念红利"（多智能体、大而全的 skill 包、MCP 全量接入、SDD 全流程）在实测里收益有限甚至有反效果。

---

## 一、方法层：不花钱、收益最确定的部分

### 1.1 上下文做减法（最高性价比）★★★

- **不要一上来加东西，先砍**。Anthropic 自家在新一代模型上把系统提示删掉 >80%，评测无可测损失 ★★（来源域名 `claude.dev` 非官方主域，建议自行复核）。
- **上下文退化主要来自"竞争性干扰"，而不是长度本身** ★★。2026 年《Length Is Cheap, Competition Is Not》用 126,473 次实验证明：中性填充几乎无损（准确率 0.97–1.00），真正致命的是**与证据同格式的干扰内容**（掉到 0.16–0.33）。→ 清理"看起来像答案但不是答案"的内容，比压缩 token 数更重要。
- **"截断上下文可缓解退化"被证伪** ★★。《How Fast Do Agents Rot?》：agentic 任务上所有被测模型都在 **16 步内**从近乎满分跌到近乎零，且**限制上下文窗口会让衰减更陡**。→ 可靠性要按**步数**预算，不是按 token 预算。
- **工具输出必须硬截断**。Claude Code 默认 25,000 token 上限是这个思路的落地 ★★★。

### 1.2 程序化工具调用（PTC / Code Mode）★★★

- Anthropic《Code execution with MCP》：把工具暴露成文件树里的代码 API，工具定义从 **150,000 token 降到 2,000（省 98.7%）**，中间结果留在执行环境里不进上下文。
- 独立验证：BFCL v4 上 14 个模型中 11 个"工具即 Python stub"不劣于 JSON 调用，GPT-5.6 系提升 10.6%，且在 context rot 下更稳 ★★。
- **代价**：需要沙箱、资源限制、监控。**没有沙箱就别走这条路。**

### 1.3 长任务要有外部落盘笔记 ★★

- 实测：把工具历史裁到最后 5 轮 + 摘要，完成率从 71.0% 升到 **91.6%**，token 从 1,480,996 降到 553,374（arXiv 2606.10209）。
- 结构性做法：`NOTES.md` / 结构化笔记，而不是指望压缩历史记住细节。

### 1.4 记忆与检索：按任务选，不是谁取代谁 ★★★

| 方案 | star | 定位 | 关键数字 |
| --- | --- | --- | --- |
| mem0 | 69k | 生产级记忆层 | LOCOMO 上相对 OpenAI 记忆提升 26%，p95 延迟 -91%，token -90%（**厂商自评** ★★） |
| Zep / Graphiti | 33k | 时序知识图 | LongMemEval 最高 +18.5%，延迟 -90%（**厂商自评** ★★） |
| Letta（原 MemGPT） | 26k | 有状态常驻 agent | — |
| context7 | 65k | 文档检索 MCP | MCP 生态里最普及的文档注入 |

**反直觉但重要的实测** ★★：

- 在可索引的只读问答上，**语义检索 65.2% 打败 deep agentic search 46.2%（且后者贵一倍多）**；后者 41.8% 的失败发生在 planner→subagent 交接处，且多是"流畅自信的错答"（arXiv 2608.01507）。
- 编码场景反过来：用文件系统 + grep 处理超长文本比语义检索高 **17.3%**（arXiv 2603.20432）。
- **结论：个人开发者先试 grep / 文件系统，别一上手就上向量库。**

---

## 二、约定与记忆文件（跨工具的"事实标准"）

### 2.1 AGENTS.md 已经是事实标准，且被正式收编 ★★★

- 官方站 [agents.md](https://agents.md) 自称 **60k+ 开源项目**在用（页面自述 ★★），可核验检索式：`path:AGENTS.md NOT is:fork NOT is:archived`
- **已移交 Linux 基金会下的 Agentic AI Foundation（AAIF）托管** —— 这是 2026 年最值得注意的标准事件之一。
- 支持方（官方列表）：Codex、Jules、Factory、Aider、goose、opencode、Zed、Warp、VS Code、Devin、UiPath、Junie、Amp、Cursor、RooCode、Gemini CLI、Kilo Code、Semgrep、GitHub Copilot coding agent、Ona、Windsurf、Augment Code
- **裁决语义（原文）**："The closest AGENTS.md to the edited file wins; explicit user chat prompts override everything."（离被改文件最近的生效，用户当轮指令覆盖一切）
- 大 monorepo 用嵌套 AGENTS.md："at time of writing the main OpenAI repo has **88 AGENTS.md files**"
- **agent 会真的执行 AGENTS.md 里列的测试命令**并修失败（FAQ 原文明确）
- 仓库 star：agents.md 26k ★★★

### 2.2 但"兼容性"有静默陷阱 ★★★

- **Claude Code 读 AGENTS.md 有遥测门**：加载器是内置插件 `agents-md`，`isOnByDefault = false` 且依赖远程 feature flag `tengu_agents_md_mod`。设 `DISABLE_TELEMETRY=1` 或 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` 后，**AGENTS.md 永远不加载且不打印任何警告**（HN 486 分）★★
- Cline 走 `.clinerules`；Cursor 主路径仍是 `.cursor/rules/*.mdc`（**纯 `.md` 会被忽略**，因为缺 frontmatter）★★★
- **Windows 坑**：用 symlink 把 `CLAUDE.md` 指向 `AGENTS.md`，会因 Git `core.symlinks` 未启用而退化成"一行文本的 CLAUDE.md"；应改用 `@AGENTS.md` 导入 ★★

### 2.3 官方对规则文件的量化建议 ★★★

- 单文件目标 **< 200 行**（原文："Longer files consume more context and reduce adherence"）
- `MEMORY.md` 只加载前 **200 行 / 25KB**
- 按需加载的文件在 200K 上下文下超 **40,000 字符**就只加载开头；单文件超 **4 MiB 直接整体跳过**
- **写法必须有可验证性**：写"Use 2-space indentation"而不是"Format code properly"；写"Run `npm test` before committing"而不是"Test your changes"
- 官方点名的反例：用 `@path` 导入"整理"长文件——**不省上下文**，导入文件与导入它的文件一起加载

### 2.4 最关键的一条认知 ★★★

> **CLAUDE.md / AGENTS.md 是 context，不是 enforced configuration。**
> 想"无论 agent 怎么决定都拦住某个动作"，只能用 `PreToolUse` 钩子。

---

## 三、工具层：编码 Agent 现状（star 为 2026-11-23/24 shields.io 快照）

| 工具 | star | 定位与关键事实 |
| --- | --- | --- |
| opencode | 229k | 当前最高 star 的开源编码 agent，200+ provider；`AGENTS.md` 逐级加载并**合并** |
| Claude Code | 157k | 能力面最宽：30+ 生命周期钩子、子代理、官方沙箱 |
| spec-kit（GitHub 官方） | 151k | SDD 工具包，已扩展出 bug 修复与"值不值得做"两个入口 |
| Codex CLI | 136k | 轻、快；争议点是默认常驻自更新守护进程 |
| Pi | 123k | 极简反主流：**明确不带子代理、plan mode、权限系统** |
| browser-use | 122k | 浏览器 agent |
| Gemini CLI | 109k | 需手动改 `.gemini/settings.json` 才认 AGENTS.md |
| OpenHands | 94k | 自主软件工程路线 |
| Cline | 72k | Plan/Act 双模 + checkpoints + headless `--json`；规则走 `.clinerules` |
| Aider | 50k | 自动 lint+test 自修闭环；**推荐模型栈停在 2025 年，生态老化** |
| Continue | 37k | IDE 内换任意模型 |
| Roo Code | 24k | 老牌 Cline 分支，相对声量已掉队 |
| sandbox-runtime | 7.2k | Anthropic 开源沙箱运行时（Seatbelt / bubblewrap+socat） |

### 3.1 真正区分高下的三个能力 ★★★

1. **钩子能不能强制**：`PreToolUse` 可 `permissionDecision: "deny"`；v2.1.295+ 的 `onFailure: "block"` 修掉了"钩子脚本崩了、动作照样放行"这个经典漏洞。钩子类型含 shell / HTTP / MCP 工具 / LLM prompt / 子代理五种。
2. **子代理能不能隔离**：默认并发上限 20、最多嵌套 3 层、`isolation: worktree` 给独立 worktree；子代理输出回传前会做指令形状扫描。
3. **沙箱是不是出厂配置**：`/sandbox` 基于 `sandbox-runtime`；**但官方明确声明 Claude 的文件工具、MCP server、hooks、LSP、statusline 全部跑在沙箱之外**；**原生 Windows 无沙箱（须进 WSL2）**。Pi 则相反——明确声明不带权限系统。

### 3.2 工作流方法论的胜负已分 ★★★/★★

- **赢家：Plan-then-execute**。Cursor 官方给出反直觉但重要的建议：*"Sometimes Agent builds something that doesn't match what you wanted. Instead of trying to fix it through follow-up prompts, go back to the plan. Revert the changes, refine the plan, run it again. This is often faster than fixing an in-progress agent."*
- **输家（降级为按需）：全流程 SDD**。spec-kit 151k★、Kiro 齐备、BMAD 56k★，但 HN 上反复出现"SDD 后来怎样了"的声音，负评关键词高度一致：**fluff、spec 之间互相打架、一致性税**。
  - 最有价值的两处残留：spec-kit 的**硬判据**（bug 修复终判 `verified`/`partial`/`failed`，"only `verified` confirms a successful fix"）与 Kiro 的**依赖图分 wave 并行**。
- **worktree 并行已成默认姿势，但冲突检测仍是空白** ★★：反例原话——"**12 tmux panes, 7 orphaned worktrees**, Claude blowing context on micro-ideas"；worktree 只隔离文件系统，**不隔离数据库、端口、构建缓存**；共享仓库时出现"反正别人会修"的偷懒涌现行为。

---

## 四、扩展层：Skills / 插件

### 4.1 现状：已是跨厂商标准，但生态治理滞后 ★★★/★★

- Agent Skills 于 2025-10 发布，2025-12 开源为 [agentskills.io](https://agentskills.io) 规范；到 2026 年主流客户端基本都支持（多数直接读 `.claude/skills`）★★
- **Agent Plugins 1.0**（2026-08-06，Amazon / Cursor / Microsoft / OpenAI / Vercel 共建）★★★：用 `plugin.json` 把 `skills/` + `mcp.json` 打成一个可移植包；已列兼容客户端 VS Code、Cursor、GitHub Copilot、Codex、Kiro。注意官方声明它**不是 AAIF 项目**。
- 生态规模：skills.sh 约 100 万 skill、2.8 亿次安装，但 **0.04% 的 skill 吃掉 62% 的安装量，近一半只被装过一次** ★★（Vercel 报告）→ **分发层才是瓶颈，不是供给**
- 值得装的仓库（star 已复核 ★★★）：obra/superpowers 325k、anthropics/skills 199k、awesome-claude-code 59k、obsidian-skills 54k、agentskills/agentskills 28k、awesome-claude-code-subagents 27k

### 4.2 实测：skill 到底有没有用？

**关键调和（这是本轮最有价值的一组发现）**：

| 实验 | 结论 |
| --- | --- |
| SkillsBench（arXiv 2602.12670，87 任务 / 18 配置）★★★ | 精选 skill 把平均通过率从 **33.9% 提到 50.5%**；**≤3 个模块的聚焦 skill 优于大而全的 bundle**；小模型 + skill 能追平无 skill 的大模型 |
| Vercel Next.js eval（2026-01-27）★★★ | 被动上下文大胜：baseline 53% → skill 默认行为 53%（**56% 的用例根本没触发 skill**）→ skill + 显式指令 79% → **AGENTS.md 文档索引 100%**（40KB 压到 8KB） |

**调和结论**：**参考型知识放进 AGENTS.md（被动、每轮都在），动作型工作流才做成 skill（主动触发）。** 实测里"未触发的 skill 反而带来噪音"——Vercel 的 tests 指标从基线 63% 掉到 58%。

### 4.3 写 skill 的黄金规则 ★★

1. **描述决定一切**：启动时只加载 name+description（约 100 token），描述写不好 = skill 不存在。用祈使句 "Use this skill when…"，写用户意图而非内部机制，宁可"推一点"。
2. **必做触发 eval**：约 20 条查询（8–10 应触发、8–10 不应触发），**负例要选近邻**；每条跑 3 次算触发率。
3. **只写 agent 不知道的**：自问"没这条指令 agent 会做错吗？"不会就删。不要解释什么是 PDF。
4. **控制体积**：SKILL.md < 500 行 / < 5000 token。
5. **关键信息必须放正文，别放 references/**：Supabase 实测"agent 懒得读引用文件，读了一个就不读第二个"，于是把安全清单搬回正文 ★★。
6. **教它怎么找，别复制内容**："先查最新文档再写代码"比抄文档更耐用。
7. **默认纯知识放 AGENTS.md，动作型工作流才做 skill**（见 4.2）。

---

## 五、连接层：MCP

### 5.1 规范在 2026-07-28 发生方向性变化 ★★★

- **彻底无状态**：删除 `initialize` / `notifications/initialized` 握手与 `Mcp-Session-Id`；版本与客户端能力改为每个请求带在 `_meta` 里
- 新增必实现的 `server/discover`；`subscriptions/listen` 长连接替换 HTTP GET + `resources/subscribe`
- **MRTR（多轮往返）**：用 `resultType: "input_required"` + `inputResponses` 替代服务端主动发起的 `roots/list`/`sampling`/`elicitation`
- **弃用 Roots、Sampling、Logging**（给 12 个月窗口）；DCR 被 CIMD 取代
- **为 prompt cache 做协议级优化**：`tools/list` 必须确定性顺序；list 结果必须带 `ttlMs` / `cacheScope`
- 移除了 SSE 断线重连与消息重投

### 5.2 但生态迁移极慢，且目录数字严重虚高 ★★

- 2026-11-10 普查 14,664 个上报版本的 host：**72% 仍在 2025-11-25**，仅 **5.2%** 报 2026-07-28，其中只有 **6 个**同时满足 CIMD 且不用被删弃特性
- 官方 Registry 有 **59,944** 个 active server，但全量探测显示：**只有 45.5% 的远程 URL 真能返回工具列表**，22.7% 是 DNS/TLS/404 死链；两个运营商占了 28.6% 的"存活服务器"和 39% 的工具
- → **"几万个 MCP server"是提交数，不是可用数。**

### 5.3 "什么时候不该用 MCP" ★★★

1. **宿主已有等价原生工具时别接**：文件读写、shell、git、抓网页。官方 filesystem / fetch / git server 的意义是给**没有**这些能力的宿主补能力。
2. **一条 CLI 就够时优先 CLI**：判据是①无状态、②只做一件事、③有成熟 `--help` 自描述（`gh`、`ffmpeg`、`aws`、`docker`、`ripgrep`）→ 给 agent 一个终端。
3. **多步骤且中间数据不该进模型时，用代码执行而不是逐个工具调用**。
4. **>20–40 个工具就该做减法**：已有 server 返回 1,076 个工具；行业普遍在 40–128 个函数处设闸。做法：按任务动态加载，或加一层 meta 工具。
5. **无法给该 server 单独签发、可轮换、最小权限凭证时，别接。**

### 5.4 值得接的 MCP server（star 已核实）

| 场景 | server | star |
| --- | --- | --- |
| 文档注入 | upstash/context7 | 65k |
| 代码托管 | github/github-mcp-server（官方） | 35k |
| 浏览器 | microsoft/playwright-mcp（无障碍树而非截图，token 友好） | 40k |
| 浏览器调试 | ChromeDevTools/chrome-devtools-mcp | 59k |
| 数据库 | googleapis/mcp-toolbox（官方，多库统一 + 鉴权） | 17k |
| 设计稿 | GLips/Figma-Context-MCP | 16k |
| 生产 SaaS | apify 17.5k / notion 4.7k / stripe 2.1k / sentry 973 | — |
| 自己写 server | jlowin/fastmcp | 29k |
| 调试必备 | modelcontextprotocol/inspector | 11k |
| 接入前扫描 | invariantlabs-ai/mcp-scan | 3.3k |
| 目录 | punkpeye/awesome-mcp-servers | 98k |
| 参考实现 | modelcontextprotocol/servers | 92k |

**建议跳过**：官方 `filesystem`、`fetch`、`git`、`sequential-thinking`、`time`——对已有终端和文件工具的 harness，原生工具更快更省 token。

---

## 六、编排与子代理

### 6.1 框架与采纳 ★★

| 框架 | star | 备注 |
| --- | --- | --- |
| microsoft/autogen | 62k | 子代理报告称**仓库已 archived**（未亲验） |
| CrewAI | 61k | 角色制，demo 最快，生产可观测性弱 |
| LlamaIndex | 53k | Workflows 事件驱动 |
| LangGraph | 45k | 图状态机 + supervisor，生产部署最多 |
| openai-agents-python | 31k | Swarm 继任者，handoff |
| Mastra | 30k | TS 生态 |
| Temporal | 25k | 持久化工作流引擎，长任务最稳 |
| google/adk-python | 22k | GCP 原生 |
| Claude Agent SDK | 8.6k | Anthropic 原生 |

### 6.2 判断规则 ★★★

**该用框架**：需要持久化/断点恢复、人工审批卡点、多租户审计、图状路由、跨小时级长任务。
**不该用（一个 while 循环 + 原生 tool-calling 就够）**：单轮生成、≤10 步固定流程、工具调用循环、纯确定性批处理。

> 判据：**先把"最简可行 harness"跑到成功率/延迟卡住，再引入框架。**

### 6.3 多智能体的真实收益与代价 ★★

- **正面**：Anthropic 内部 research eval 上多智能体比单 Opus 4 高 **90.2%**；并行化把研究耗时最多降 90%
- **代价**：token 消耗约为聊天的 **15×**（单 agent 约 4×）；token 用量单独解释 BrowseComp 上 **80%** 的性能方差
- **反例**：MAST（arXiv:2503.13657，7 框架 1600+ 轨迹）归纳 **14 种失效模式**，结论是"流行 benchmark 上多智能体增益常常很小"；Cognition 主张"默认排除不共享上下文的架构"
- **编码场景可并行度低，多智能体目前不划算**；研究类任务才划算
- 强制委派的实测反效果：冲突事实检索率 50.2%→68.7%，但**冲突检出率 16.2%→13.2%**（信息更多、判断更差）
- 单个恶意 agent 就能击穿：**100% 任务失败**、token 7.7×、运行时 3.3×、泄露 94% 相关历史任务

### 6.4 有效实践 ★★

- 给 lead agent 写明确的**努力预算**（简单事实 1 个子代理 3–10 次工具调用；复杂研究 >10 个）
- 子代理产出**写入文件系统、只回传轻量引用**，避免"传话游戏"
- 子代理**共用同一份额度**；非内置子代理的描述合计超 **15,000 token** 会告警

---

## 七、评估与可观测

| 工具 | star | 自托管 | 免费 |
| --- | --- | --- | --- |
| Langfuse | 38k | ✅ Docker/K8s | ✅ 核心 OSS |
| promptfoo | 28k | ✅ | ✅ 全功能 OSS |
| DeepEval | 20k | ✅ | ✅ |
| OpenAI Evals | 20k | ✅ | ✅ |
| Arize Phoenix | 12k | ✅ | ✅ |
| W&B Weave | 1.2k | 部分 | 有免费额度 |
| Braintrust | — | ❌ 托管为主 | 有免费额度 |

**默认组合**：Langfuse（追踪）+ promptfoo（回归）+ DeepEval（指标）。

### LLM-as-judge 的可靠性已被实证削弱（重要）★★

- few-shot 示例标签偏斜使评审通过率单向跳升约 **30 点**（NExT-QA）/ 20 点（MSVD-QA）；单次抽样波动可达 **18 点**
- 语义等价、仅翻转评分锚点就会改变结论；评分量纲本身系统性影响分数
- 同一批输出：确定性打分（完整性 49.6%、字段正确率 30.8%）vs LLM-judge（66.5%、60.4%）→ **judge 系统性抬高分数**
- **校准做法**：均衡示例标签 + 多次抽样取均值 + 双向锚点平均 + 人工子集验证

### OTel GenAI 语义约定**尚未**统一 ★★

GenAI 约定已迁出主仓库，独立为 `open-telemetry/semantic-conventions-genai`（560★，schema URL 仍标 TODO）。→ "标准已定"在 2026 年不成立，跨厂商 trace 对齐仍需自建映射层。

---

## 八、安全（必读，本轮证据最强的一节）

### 8.1 Skill 市场供应链：扫描器全线失守 ★★★

Trail of Bits 实测（2026-06-03），**不到一小时**绕过 ClawHub（VirusTotal + GPT-5.5）、Cisco skill-scanner、skills.sh 的三个扫描器：

- **10 万个换行**塞进截断窗口，把恶意代码挤出扫描范围
- **`.pyc` 字节码投毒**：源码干净、字节码读环境变量 → skills.sh 与 Cisco 都判 `is_safe: true`
- **`.docx`（本质是 ZIP）内藏 payload** 随真实 XML 一起走
- **话术型 prompt injection**："企业内网镜像"叙事骗过 LLM 分析器（这条花了几小时，因为要试措辞）
- 反向发现：**Anthropic 官方 Office skill 自己就在用 `LD_PRELOAD`** 注入 C 代码编译的 shim（为绕过 LibreOffice socket 限制）——扫描器只能判 LOW

> 原文结论：**"Don't outsource trust to a scanner."** 公共 skill 市场对敏感环境一律视为不可信代码；改用 curation（人工策展）+ pin 版本 + 控制谁能引入更新。

### 8.2 MCP 安全 ★★

- **CVE-2025-6514**（CVSS **9.6**，NVD 已核实 ★★★）：`mcp-remote` 在 `authorization_endpoint` 响应 URL 上做 OS 命令注入，**连上不可信 server 即可 RCE**
- **CVE-2025-54136（MCPoison，Cursor）**：已批准的 MCP 配置不再校验，可持久化 RCE
- **postmark-mcp（2025-09）**：首个在野恶意 MCP server，前 15 版干净，1.0.16 加一行 BCC 抄送攻击者；约 300 家组织接入
- **供应链投毒是主战场**：`iflow-mcp` 组织批量 fork 数百个 MCP server 并重新发布到 npm/PyPI；Zapier npm 40+ 包被篡改
- **教科书式 tool poisoning 几乎不存在**：14 万条真实工具描述里，"先读 `~/.ssh/id_rsa` 再传出"出现 **0 次**。真实流行的是**对用户隐瞒的指令**：47 个 host 写 "do NOT tell the user that the platform or safety checks blocked the action"；406 个声称自己"必须先于其他所有工具调用"（跨 server 抢占执行顺序）
- **`instructions` 字段是没人审的上下文黑洞**：66% 的存活 server 返回 server 级 `instructions`（被拼进系统提示），最长 **68,669 字符**（约 1.7 万 token，**每轮都付费**）
- **注解不是权限系统**：72% 的工具带 `annotations`，78,551 个自称 `readOnlyHint: true`——server 自己填的 JSON 字段而已
- **认证现状**：62% 的 host 允许**未登录**列出工具，**15,064 个自称 destructive 的工具暴露给匿名客户端**
- **密钥在公开仓库里**：24,008 个唯一密钥，**2,117 个仍有效**；2,614 个 MCP 实现中 **82%** 有路径穿越风险

### 8.3 编码 Agent 自身的安全反例 ★★

- **GitSpawn（Manifold Security）**：7 个 agent 的启动期 git 调用可执行任意代码，**发生在你接受工作区信任提示之前、甚至登录之前**，且"the command runs outside the sandbox, without an approval prompt. The permission model never sees it"。发布时 4 项仍未修（含 Claude Code `ultrareview`、Qwen Code、Grok Build、Hermes）。
  - 传染路径很日常：**不是 clone，而是"把含 `.git` 的目录当文件传"**（zip、共享盘、U 盘）→ `git status` 刷新索引时执行 `.git/config` 里 `core.fsmonitor` 指定的程序
  - 防护：收到别人的目录先看 `.git/config`；agent 侧 `git -c core.fsmonitor=false status`
- **`~/Downloads` 里的 AGENTS.md 劫持（2026-10-25）**：浏览器能静默把 `AGENTS.md` 写进下载目录，而很多人在那儿跑 agent。实测对 Codex 成功率 **90%**（无害载荷）；用恶意 CLAUDE.md 把 Claude Code 的 API 流量重定向以窃 OAuth token，约 **50%** 成功。研究者原话："**There is nothing here for OpenAI or Anthropic to patch.**"；某机构统计**仅 Claude Code 就在该目录启动过 1300+ 次会话**
  - → **永远不要从 `~/Downloads` 或家目录启动 agent**
- **下一代依赖风险（slopsquatting 底座）**：57.6 万份样本中，商用模型平均 **≥5.2%**、开源 **≥21.7%** 推荐不存在的包（205,474 个唯一幻觉包名，USENIX Security 2025）

---

## 九、成本

- **Uber**：2026 年 4 月烧完全年 AI 预算（Claude Code + Cursor），人均月 API 成本 **$500–$2,000**；95% 工程师月用、**70% 提交代码来自 AI**；CTO 称"回到画板"，6 月限额 ★★
- **缓存命中比换便宜模型更重要**：同一份上下文，缓存读 vs 未命中输入差 **50 倍**（$0.003 vs $0.15/M）；判断只看 `usage.prompt_tokens_details.cached_tokens` ★★
- **峰谷定价**：DeepSeek 低谷输入 $0.15/M、输出 $0.60/M、缓存读 $0.003/M，高峰 2 倍（高峰 ≈ 北京时间工作日 9:00–12:00、14:00–18:00）★★
- **稳定前缀要单独拆出来**：21.6k 上下文冷 prefill 109 秒、次轮 5 秒 ★★
- **每个工具一把独立 Key + maxSpend 上限**：agent 循环每轮工具调用、每次重试、每次后台摘要都是独立计费请求 ★★

---

## 十、冷水区：被高估与已被证伪的部分

| 概念 | 共识程度 | 证据 |
| --- | --- | --- |
| 开发者自评提效倍数 | **已证伪** | METR RCT：允许用 AI 时**慢 19%**，而开发者自认快了 20%；高估幅度 **40 个百分点** ★★ |
| benchmark 分数 ≈ 真实能力 | 已基本形成共识 | SWE-bench 已污染：只给 issue 描述猜 bug 文件路径 **76%**，换非 SWE-bench 仓库仅 **53%** ★★ |
| 上下文窗口越大越好 | 已基本形成共识 | Chroma context rot：18 模型 7 任务，加单个 distractor 即掉分；**结构连贯的长文本反比乱序更差** ★★★ |
| 通用自主 agent（AutoGPT 类） | 共识 | — |
| 多智能体框架 | 已基本形成共识 | MAST 14 种失效模式；Cognition 主张默认排除 ★★ |
| "一行接入所有工具" | 共识快速形成中 | MCP 自身吃满上下文，社区已转向工具路由/延迟加载 ★★★ |
| Skill / prompt 堆叠 | **少数人质疑，缺统计支撑** | 应与上面严格区分；Vercel 的"未触发 skill 带来噪音"是最接近的证据 ★★★ |

### 两个必须记住的"反常识"

1. **METR 那个著名的"AI 让开发者变慢 19%"实验，2026-02 被 METR 自己承认不可靠**：30%–50% 开发者拒交"不许用 AI"的任务，招募出现选择偏差；新估计 -18% / -4%，CI 跨 0。→ **别拿它当定论，也别拿相反结论当定论。**
2. **长任务天花板仍是硬约束**：METR TH1.1 的 50% 时间视野，Opus 4.5 约 **320 分钟**（CI 170–729，上限是点估计的 2.3 倍），且 **80% 可靠性下的视野远低于 50%**；31 个 8h+ 任务中仅 5 个有人类基线。→ **跨小时任务要按 50% 成功率设预期并插检查点。**

---

## 十一、落到 DSH：现状与可选官方包

### 11.1 现状（本机实测）★★★

- `DSH_PROFILE=web`，profile 目录 `C:\Users\10261\.dsh\profiles\web`
- `package.json` 只声明两个 bundle：`@deepseek-ai/dsh-base`、`@deepseek-ai/dsh-web-app`
- `cordis.patch.yml` 只有 4 条覆盖：两个自定义 provider（rightcode / cubence）、`ui-settings-general`、`ui-chat`、`agent-default-model`
- 默认模型：`cubence` / `deepseek-flash` / `reasoningEffort: high`；本地登记的上下文窗口 1,048,576
- **⚠️ 此处初版结论有误，已勘误**：初版写"能力面基本是出厂状态"是**错的**。实测 `dsh-base` 的 bundle 清单后确认，base 已内置 compaction / plan-mode / skill / subagent / workflow / spill / 循环看护 / 会话检索等一大批能力。详见 §14 勘误。

### 11.2 与公认实践的对应关系（⚠️ 下表多数据已在 base 中，见 §14 勘误）

| 对应的方法 | 可装包 |
| --- | --- |
| 上下文做减法 / 压缩 | `dsh-compaction-basic`、`dsh-compaction-tool-result-pruner`、`dsh-compaction-image-offload`、`dsh-command-compact`、`dsh-spill-policy` |
| 计划先行 | `dsh-plan-mode`、`dsh-tool-todo` |
| 约定文件 | `dsh-agent-instructions`（已随 base 生效） |
| 技能 | `dsh-skill-filesystem`（外部 skill 目录）、`dsh-tool-skill` |
| **门禁用钩子**（对应 2.4 的核心结论） | `dsh-hooks-claude-code`、`dsh-hooks-codex`、`dsh-experimental-claude-code-mods` |
| **跨模型复核** | `dsh-subagent-claude-code`、`dsh-subagent-codex`、`dsh-subagent-fork-in-process` |
| worktree 并行 | `dsh-experimental-worktree`、`dsh-tool-worktree` |
| LSP 代码导航 | `dsh-lsp`、`dsh-lsp-stdio`、`dsh-tool-lsp` |
| 终端 | `dsh-tool-terminal`、`dsh-terminal-bash`、`dsh-terminal-panel` |
| MCP 接入 | `dsh-mcp-client`、`dsh-mcp-manager`、`dsh-mcp-resources` |
| 浏览器 | `dsh-experimental-browser-use-playwright-mcp`、`dsh-experimental-browser-use-chrome-devtools-mcp` |
| **成本/上下文可见性** | `dsh-experimental-client-ui-stats`（每轮/每步的上下文、耗时、缓存、工具成本表） |
| 会话历史检索 | `dsh-session-query-sqlite`、`dsh-tool-session-query` |
| 危险动作前置审查 | `dsh-experimental-auto-review` |
| 循环看护 | `dsh-repeat-tool-reminder`、`dsh-tool-call-timeout-policy` |
| 多智能体 | `dsh-agent-team`、`dsh-tool-agent-team`、`dsh-tool-ralph` |
| 插件市场 | `dsh-plugin-market` |

> 完整可用包清单见 DSH 安装目录下 dsh-doc skill 的 `references/packages.md`（本机：`C:\Users\10261\AppData\Local\Volta\tools\image\packages\@deepseek-ai\dsh\node_modules\@deepseek-ai\dsh\node_modules\@deepseek-ai\dsh-agent-preset\skills\dsh-doc\references\packages.md`）

### 11.3 一个待验证的本机可疑点 ★

`cordis.patch.yml` 里 `rightcode` provider 写的是 `api: openai-completions`，`cubence` 是 `anthropic-messages`（默认走 cubence，目前安全）。

有二手分析称：**在 DSH 里给 provider 写 `api: openai-completions` 会绕过 session-header 包装** —— 请求正常返回、不报错，但**缓存恒为 0、成本按约 50 倍走**。该结论来自第三方博客（80aj.com），**未验证**。排查特征：`usage.prompt_tokens_details.cached_tokens` 恒为 0。

---

## 十二、给个人开发者的分层行动建议

**第 0 层（今天就能做，零成本，收益最确定）**
1. 只维护一份 `AGENTS.md`，写"该怎么做 + 什么会咬人"，控制在 200 行内
2. 关键约束**不要写成规则**，写成钩子（规则是 context，不是 enforcement）
3. 永远不从 `~/Downloads` 或家目录启动 agent；收到别人的目录先看 `.git/config`
4. 工具输出硬截断；长任务用落盘笔记（`NOTES.md`）而不是指望压缩

**第 1 层（一周内，低成本）**
5. 中等以上改动先走 plan mode；输出不对时**回退改计划重跑**，别继续修跑偏的 agent
6. 装 `dsh-experimental-client-ui-stats`，把 token / 缓存 / 成本看见 —— 先量再优化
7. 沙箱常开，关掉"沙箱失败就无沙箱重试"的逃生通道
8. git worktree 并行，但**串行合并**（本地 merge queue）

**第 2 层（按需，有明确触发条件才做）**
9. 接 MCP：只接"宿主没有的能力"（如 context7 文档、playwright 浏览器、数据库），**别接 filesystem/fetch/git**
10. 跨模型复核：用另一个模型（Codex / Claude Code）做 review，避免同源自审
11. 写 skill：只做**动作型工作流**（"升级 Next.js 版本"这类），参考型知识放 AGENTS.md
12. 上评估：先有 20 条真实任务的回归集，再谈 LLM-judge

**第 3 层（谨慎，多数人不需要）**
13. 多智能体编排框架：先证明"最简 harness 卡住了"
14. 语义检索 / 向量库：先试 grep 和文件系统
15. 公共 skill 市场：**默认视为不可信代码**

---

## 十三、来源与核实状态

**我本人一手打开核实的（★★★）**
- https://agents.md
- https://modelcontextprotocol.io/specification/2026-07-28/changelog
- https://modelcontextprotocol.io/docs/2026-07-28/getting-started/intro
- https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals.md
- https://aaif.io/blog/from-skills-and-tools-to-portable-agent-plugins
- https://blog.trailofbits.com/2026/06/03/the-sorry-state-of-skill-distribution/
- https://arxiv.org/abs/2602.12670（SkillsBench）
- `https://img.shields.io/github/stars/<owner>/<repo>.json`（本文所有 star 数的来源）

**子代理一手核实、我未逐篇复验的（★★）**
Anthropic 系列（effective-context-engineering / code-execution-with-mcp / writing-tools-for-agents / multi-agent-research-system）、Chroma context-rot、Claude Code 官方文档（hooks / sub-agents / memory / sandboxing）、Cursor 官方文档（rules / plan-mode）、opencode / Kiro / spec-kit / BMAD / cline / Aider / Pi 的 README、METR 五篇博客、arXiv（2503.13657 MAST、2406.10279 slopsquatting、2506.12286 SWE-bench 污染、2608.01507、2606.10209、2603.20432 等）、fetchgate MCP registry audit、spellwarden MCP 安全普查、NVD CVE-2025-6514、Manifold Security GitSpawn、BleepingComputer（Downloads 劫持 / 沙箱逃逸）、HN 多条原始帖、presenc.ai（框架采纳比例，**厂商自报**）

**明确未核实、不应引用的（★）**
- Bloomberg "Skills over MCP" 一文（403，本轮未打开任何 Bloomberg 页面）
- `claude.dev` 那篇"删掉 80% 系统提示"（域名非官方主域）
- 26.1% 的 skill 漏洞率（来自第三方聚合站 host0.ai，非一手）
- 国内工具横评与价格（router.one / 80aj.com 等二手博客）
- DSH `api: openai-completions` 导致缓存失效的结论（二手博客，见 11.3）
- Gartner "40%+ agentic 项目被取消"（403，仅见标题）
- Microsoft Agent Framework 的 star 数（shields.io 返回 repo not found，**未取到，故不列**）

---

## 十四、勘误与本机实测补充（2026-11-24 追加）

### 14.1 勘误：§11 的"缺口"判断错了

实测 bundle 清单后确认，`dsh-base` **已内置**：

`token-meter`、`compaction-basic`、`compaction-tool-result-pruner`、`compaction-image-offload`、`spill-local`、`spill-policy`、`plan-mode`、`tool-todo`、`tool-goal`、`skill-filesystem`、`tool-skill`、`subagent`（含 spawn / fork / tool / control）、`agent-team` + `tool-agent-team`、`workflow-ptc` + `tool-workflow`、`repeat-tool-reminder`、`tool-call-timeout-policy`、`session-query-sqlite`、`tool-fs`、`tool-fs-search`、`tool-web`、`mcp-resources`

`dsh-web-app` 已内置：`session-stats`（chat stats strip）、`ui-trajectory`（timing overview）。

→ **上下文压缩、计划先行、技能、子代理、编排、循环看护、会话检索都已经有了。** 初版把其中多项列为"待装"，是错的。

**真正的缺口**（base + web-app 里都找不到）：

| 缺口 | 包 |
| --- | --- |
| **门禁钩子桥**（对应 §2.4 最高优先级结论） | `dsh-hooks-claude-code`、`dsh-hooks-codex`、`dsh-experimental-claude-code-mods` |
| MCP 接入 | `dsh-mcp-client`、`dsh-mcp-manager`（注意：只有 `mcp-resources` 在 base，**没有 client**） |
| 代码导航 | `dsh-lsp`、`dsh-lsp-stdio`、`dsh-tool-lsp` |
| 持久终端 | `dsh-tool-terminal`、`dsh-terminal-bash`、`dsh-terminal-panel` |
| worktree 并行 | `dsh-experimental-worktree`、`dsh-tool-worktree` |
| 跨模型复核 | `dsh-subagent-claude-code`、`dsh-subagent-codex` |
| 危险动作审查 / 循环 | `dsh-experimental-auto-review`、`dsh-tool-ralph` |
| 细粒度成本表 | `dsh-experimental-client-ui-stats` |
| 浏览器 | `dsh-experimental-browser-use-playwright-mcp`、`dsh-experimental-browser-use-chrome-devtools-mcp` |

### 14.2 本机 token / 缓存 / 成本的可见性（回答"DSH 是不是本来就显示"）

- **base 有数据层**：`dsh-token-meter`（replay-aware 计量服务）；其消费方是 compaction 插件（`compaction-basic` 用它判断上下文压力）
- **web-app 有基础 UI**：`dsh-session-stats`（"chat stats strip"，整段会话的轮/步计数）+ `ui-trajectory`（耗时总览与工具结果检视）
- **不含** `dsh-experimental-client-ui-stats`——即"每轮/每步的上下文 + 缓存 + 工具成本表"是**未安装的实验性可选包**

### 14.3 rightcode baseURL 缺陷（已实测确诊）

| 配置 | SDK 实际请求 | 结果 |
| --- | --- | --- |
| `https://api.site/v1`（现状） | `POST /v1/v1/messages` | **HTTP 404 `404 page not found`** |
| `https://api.site`（去掉 `/v1`） | `POST /v1/messages` | **HTTP 200，正常返回** |

**根因**：OpenAI SDK 只追加 `/chat/completions`（所以 openai-completions 的 base 必须带 `/v1`），而 Anthropic SDK 追加的是完整 `/v1/messages`。**切换协议时必须同步改 baseURL。**

证据（源码）：`@anthropic-ai/sdk/lib/resources/messages/messages.js:36` → `post('/v1/messages')`；`openai/resources/chat/completions/completions.js:85` → `post('/chat/completions')`；`pi-ai/dist/api/anthropic-messages.js:972` → `baseURL: model.baseUrl`（原样传入）。

### 14.4 缓存实测（结论：缓存不用操心，但 baseURL 必须修）

| 场景 | input_tokens | cache_read | cache_creation |
| --- | --- | --- | --- |
| rightcode 第 1 次（显式 `cache_control`） | 4259 | 0 | 0 |
| rightcode 第 2 次（同前缀） | **35** | **4224** | 0 |
| rightcode 不传 `cache_control` | 35 | 4224 | 0 |
| cubence 首次（同前缀） | 35 | 4224 | 0 |

- 缓存是**服务端按前缀自动命中**，与是否显式传 `cache_control` **无关**；第二次同前缀输入 token 从 4259 降到 35（约 **99% 命中**）
- 观察：cubence 首次使用该前缀即命中（该前缀此前只发给过 rightcode）→ **疑似两者共享上游缓存**（单次观察的推断，未进一步验证）
- pi-ai 侧：默认 `retention=short` 会自动加 `cache_control: {type:"ephemeral"}`（`anthropic-messages.js:37-47`）；`PI_CACHE_RETENTION=long` 可在支持的 provider 上申请 1h TTL

### 14.5 勘误：§11.3 的"openai-completions 绕过缓存"不成立（撤回）

源码层面：

- `openai-completions.js:557-600` **同样会调用 `applyAnthropicCacheControl`**，条件只是 `compat.cacheControlFormat === "anthropic"`
- 真正决定是否加缓存标记的是 **`compat.cacheControlFormat`** 配置项，**不是 `api` 字段**
- `pi-ai/dist/utils/identity.js:90`：`getOpenCodeSessionHeaders` 只在目标是 OpenCode 时加 `x-opencode-session`，与 `api` 无关

→ 该二手说法站不住，撤回。§11.3 不再作为待验证疑点。

