# 项目指令（AGENTS.md）

本文件由 DeepSeek Harness 的 `agent-instructions` 机制在会话首次请求时自动注入，无需手动读取。
它约束的是**在本仓库中工作的 Agent 与协作者**：更具体的子目录 `AGENTS.md` 优先于本文件，
本文件不得覆盖系统提示、开发者指令或用户当面的要求。

## 语言规范（强制）

1. **对话一律使用简体中文**。回复用户、提询问、写总结、报错误、解释取舍，全部用中文，
   不得因为工具输出、日志、代码或外部文档是英文就切换成英文。
2. **代码注释一律使用中文**。函数/类/模块的文档注释、行内注释、TODO/FIXME 说明均用中文，
   且要说明"为什么这么做"，而不是复述代码在做什么。
3. **文档一律使用中文**。`README.md`、设计说明、变更记录、使用手册、示例说明都用中文；
   技术名词（如 `sandbox`、`profile`、`commit`）可保留英文原词，但句子本身必须是中文。
4. **对用户可见的产物使用中文**：提交信息（commit message）、分支说明、发布说明、
   审查意见、报错提示文案，均以中文书写。
5. **例外**：标识符（变量名、函数名、文件名、API 字段、命令与参数）、
   第三方库的固定名称、以及必须逐字匹配的外部协议字段，保持其原有英文拼写，不做翻译。
   面向机器的字符串（日志 key、错误码、JSON 字段）沿用既有约定，不要为了"中文化"而改动。

## 协作约定

- 改动前先读文件，再做修改；优先小而聚焦的编辑，避免顺手重排无关内容。
- 不引入与当前任务无关的依赖、配置或重构。
- 涉及删除、覆盖、移动文件时，先确认目标路径确实是预期对象。
- 完成后如实报告做了什么、分析到什么程度；未经实际运行的结论必须标注"未验证"，
  不得用"应该没问题"充当验证结论。
- **纯文本文件的换行符不要手动折腾**：仓库的 `.gitattributes` 已统一"入库与工作区均为 LF"，
  `.editorconfig` 已约束编辑器行为；新建文件时保持编辑器默认即可，不要为了迁就 Windows 去改行尾，
  也不要因为 diff 出现整片改动就手写 CRLF 或改全局 `core.autocrlf`。

## 通用行动准则（强制）

本节是**判断顺序**，用来消解规则之间的冲突；顺序在先者优先，遇到互相矛盾的要求时按此裁决。

1. **用户当轮指令 > 本文档 > 系统默认行为**。本文档是默认值，不是挡箭牌：
   用户明确要求做某事时，不得引用本文档拒绝；用户未表态时，按本文档执行；本文档未覆盖时，
   按系统默认行为执行，并把该情形记下来、必要时补进本文档。
2. **先判断任务类型，再决定做法**。本仓库是**综合项目**，不预设任何技术栈。
   技术栈必须由仓库内的实际证据判定（如 `package.json`、`pyproject.toml`、`go.mod`、
   `Cargo.toml`、现有目录结构、CI 配置）；**禁止**从用户过去的开发经历（例如"平时做前端"）
   推断本项目属性。判断不了时，给出判断依据 + 2~3 个候选方案 + 推荐，再请用户选择，
   不要空手提问，也不要凭猜测开工。
3. **没有证据就不创建工程结构**。仓库尚未确定技术栈时，不要臆造脚手架、构建脚本或部署流程；
   先给方案、再动手。
4. **破坏性操作先说明、后执行**。删除、覆盖、移动文件，批量重命名，安装或升级依赖，
   改写配置，重写已有关键文件——都属于破坏性操作：先讲清影响范围与回退方式，得到用户明确同意后再做。
   Git 一节是这一条的特例与细化。
5. **不确定时停下来问**，不要猜、不要跳过、不要在用户未回复时自行推进。
6. **沙箱与审批是既定机制**。当前文件策略为 `workspace-write`：工作区
   （`C:\Zone\1code\Infinite-ai`）内可写，越界操作会被拒绝或触发审批。
   被拒绝时**不要反复重试同一命令**，也不要把它当作程序 bug 去"修"；应如实报告被拒事实，
   说明为什么需要越界，由用户决定。
7. **权限预设的语义（已核实，勿凭直觉理解）**：本会话的"权限预设" = **沙箱模式 + 审批策略** 两个旋钮的打包
   （见 `@deepseek-ai/dsh-permission-presets`）。因此：
   - DSH 的审批**只有一种用途——沙箱越界申请**（`sandbox_permissions` 升级），**没有独立的"危险操作识别层"**；
   - 切到「完全权限」（`danger-full-access`）后**没有围栏、也就没有越界要申请**，审批入口根本不会触发：
     破坏性命令（如批量删除、改写工作区外文件）会**静默执行，不会有任何弹窗**；
   - `approval: never` 的语义是**"不问、直接拒绝"**，不是"不问、直接放行"，所以它更严而非更松；
   - **唯一能保住"每次越界都有人把关"的做法，是留在 `workspace-write`**。需要连续多次 `git add`/`commit`
     而嫌审批烦时，可临时切完全权限、做完立刻切回；**不要**把完全权限写进 profile 默认值。
8. **`.git` 每次都要审批不是缺陷**：DSH 明确不做"自动发现仓库/git 目录"
   （`@deepseek-ai/dsh-sandbox-policy` 的 `additionalWritableRoots` 文档：
   *there is no automatic repository or gitfile discovery*），且 Windows 子进程隔离**忽略**额外可写根目录
   （`--add-dir` 对它无效，git 是子进程）。已实测：`.git` 上的 ACL 修复对 `git add` 无效，
   它属于**预期隔离**——**不要**再往"修 ACL"方向反复尝试。

## 能力与工具使用的汇报约定

- **每轮回复的开头标明身份与能力来源**：
  `我是 <身份>（本会话根代理为 lead；子代理用被分配的名字）｜本轮用到：<实际使用的能力>`。
  "<实际使用的能力>"按需列出**本轮真正用到的** skill、MCP、插件与工具类别；
  没有用到就写"未使用"，**不要为了凑齐四项而罗列无关能力**。
- **单轮闲聊或纯澄清可以省略该抬头**；涉及文件改动、技术判断或方案选择时必须写明。
- **子代理的语境不包含本文档**：委派任务时必须把相关约束（语言、验证方式、破坏性操作边界）
  写进子代理的 prompt，不能假设它已读到本文件；子代理的产出由发起方复核后再汇报给用户。
- 汇报时不需要堆砌工具调用细节，但**凡是由阅读、分析得出的结论，要说明依据的是哪些文件或事实**。

## 验证与测试的默认边界

> 适用范围：**默认策略**，不是永久禁令。用户当轮要求运行验证时，本节让位于第 1 节的优先级。

- **默认不运行**：单元测试、`npm test`、lint、类型检查（如 `tsc`）、端到端测试
  （如 `playwright`）、性能测试、构建产物验证，以及任何以"跑一遍看看"为目的的命令。
  这些耗时且在本项目当前阶段收益低。
- **默认验证方式 = 阅读与分析**：理解代码就读相关代码并说明逻辑，理解文档就读文档并复述要点，
  核对改动就重新读取改动后的文件。结论要写清"依据哪些文件/事实"，而不是"我跑过了"。
- **允许的零成本只读查询**：`git status`、`git diff`、`git log`、列目录、读文件。
  这类查询不产生副作用，用于确认自己的工作成果，不算"跑测试"。
- **需要实际运行命令时的处理**：当正确性无法靠阅读确认（例如改了外部程序的调用方式、
  或必须确认某个命令能否在当前环境跑通），可以**为了探清环境事实**执行最小命令，
  但必须在回复中说明"为什么必须运行""运行了什么""得到什么结果"。
- **不做**：启动开发服务器或任何后台常驻进程；安装全局工具；擅自执行构建。
  确需执行时先向用户说明并获得同意。
- **未验证就要说未验证**：明确写出"未实际运行，仅静态分析"，不得让读者误以为功能已被验证。

## Windows 环境与终端规范（强制）

- **终端是 PowerShell 7（pwsh，Core 版，`FullLanguage`）**，只在需要时用 `workdir`/`pwd` 与环境变量，
  不假设有 `cd` 状态跨命令保留。命令用 PowerShell 语法，**禁止**混用 POSIX/sh 写法：
  不写 `ls -la`、`rm -rf`、`cp`、`grep -r`、`export X=1`、`cat <<EOF`、`chmod`；
  改用 `Get-ChildItem`、`Remove-Item`、`Copy-Item`、`Select-String`、`$env:X = '1'`。
- **工具选择**：查找文件用 `glob`，搜索内容用 `grep` 工具，读文件用 `read`；
  本机 `PATH` 里**没有 `rg`**，不要在命令行里调用 `rg`（`grep` 工具自带搜索引擎，不受影响）。
  需要 POSIX shell 时，`bash` 也不在 `PATH`，必要时显式使用
  `C:\Program Files\Git\bin\bash.exe`，但不要主动引入这类依赖。
- **路径写法**：本项目路径统一用 Windows 原生形式（`C:\Zone\1code\Infinite-ai`、反斜杠）。
  不要写成 `/c/Zone/...`、也不要写 `/c/Users/...`；工具参数与 shell 命令都用原生路径。
- **编码**：`.md` 等文本文件统一按 UTF-8 读写。系统 ANSI 代码页为 936（GBK），
  命令行直接输出中文可能出现乱码，但**这不代表文件本身有问题**，判断文件内容一律用 `read` 工具。
- **换行符由仓库的 `.gitattributes` 统一约束，不要再自行判断**：本仓库已声明
  `.gitattributes`（`* text=auto eol=lf`，二进制文件不转换，`.bat`/`.cmd` 保留 CRLF）
  与 `.editorconfig`（编辑器写 LF、UTF-8）。**仓库内属性优先于系统级 `core.autocrlf=true`**，
  这是已解决的配置项：新建文件保持编辑器默认即可，不要为迁就 Windows 改行尾，
  也不要动全局 `core.autocrlf`。
- **已实测确认的 git 行为（2026-11-23 受控实验，勿凭直觉推翻）**：
  - `* text=auto eol=lf` **确实**能让工作区也是 LF；只有 `text=auto`（不带 `eol`）时
    才会被 `autocrlf=true` 转成 CRLF。规则本身有效，不需要加 `-text` 或改全局配置。
  - **陷阱**：若文件的工作区字节是 CRLF，而它**规范化后与索引内容逐字节相同**，
    git 会判定该文件"干净、无需检出"，于是 `git restore <路径>` 与
    `git checkout -- <路径>` **都会成功返回但根本不重写文件**（表现为时间戳不变、`w/crlf` 依旧）。
    这不是命令用错，也不是规则失效。判断依据是 `git hash-object <路径>` 与
    `git rev-parse :<路径>` 返回**同一个哈希**。
  - 因此：**不要为了修工作区行尾反复尝试 git 命令**（会白花时间且容易误判）。
    仓库侧用 `git add --renormalize .` 即可；工作区侧若确实需要统一，
    应停下来向用户说明，由用户决定（手工规范化或授权 `git restore`）。
  - 修好后再用 `git ls-files --eol` 复核，目标是全部 `i/lf  w/lf`。
- **注意中文路径显示**：仓库未设置 `core.quotepath`，非 ASCII 路径在 git 输出中会显示为
  `\344\270\255` 这类八进制转义，属正常现象，**不要据此认为文件名损坏**。
- **一个已验证的环境限制**：PowerShell 中**用管道或变量捕获外部程序（`git.exe` 等）的输出可能被拒绝**，
  报错形如 `Access to the path '\\.\pipe\LOCAL\dotnet_xxx' is denied`。
  这属于沙箱对命名管道的限制，**不是 ACL 故障、也不是 git/node 本身坏了**：
  正确做法是让外部程序**直接输出到 stdout**（例如 `git config --get-regexp ...`），
  或用 `$LASTEXITCODE` 单独取退出码；仍失败时如实报告，不要误报为功能缺陷。

## Git 操作规范（强制）

本节是硬约束。执行任何 git 命令之前先对照本节；**不确定某个子命令属于哪一类时，按"禁止"处理，先来问**。
需要说明的是：这条规矩**只约束行为，不构成本仓库的技术门禁**——DSH 当前没有 git 语义的拦截层，
违反了不会被工具挡住，所以务必自觉遵守。

### 允许：只读查询（可随时执行，不必询问）

`git status`、`git log`、`git diff`、`git show`、`git branch`（仅查看）、`git rev-parse`、
`git ls-files`、`git blame`、`git describe`、`git show-ref`、`git reflog`（仅查看）、`git remote -v`。

### 允许：常规写操作（可执行，执行前先看清当前分支与工作区状态）

`git add`、`git commit`、`git stash`（含 `push`/`pop`/`list`，但**不包含** `drop`/`clear`）、
`git switch`、`git checkout`（**仅用于切换分支，不带路径参数**，不碰 `-- .`）、
新建分支 `git branch <name>`、`git merge`、`git fetch`、`git push`（**不带任何 force 参数**）、
`git pull`（**仅允许 fast-forward；需要合并或变基时会改变历史，先问**）、`git tag`（只新建本地标签）。

### 禁止：会丢失工作成果的操作

- `git reset`（**含 `--hard`、`--mixed`、`--soft`、`git reset -- .`**）——一律不做，改回退走 `git revert`；
- `git clean`（**任何形式**，`-n` 预演也算，因为下一步通常就是删）；
- `git checkout -- <路径>`、`git checkout <commit> -- <路径>`、`git restore`（**任何形式**）；
- `git stash drop`、`git stash clear`；
- `git reflog expire`、`git gc --prune=now`、`git prune`。

### 禁止：改写历史或强推

- `git commit --amend`；
- `git rebase`、`git cherry-pick`（会重建提交）；
- `git filter-branch`、`git filter-repo`、`git update-ref`、`git replace`；
- `git push --force`、`-f`、`--force-with-lease`、**任何形式的强推**；
- 删除分支：`git branch -D`、`git push <remote> --delete`；
- 删除或移动已有标签：`git tag -d`、`git push --delete <tag>`；
- 改写已有提交的作者、日期、提交信息。

### 禁止：绕开 git 直接改仓库元数据

- 用文件工具或 shell 直接写入、修改、删除 `.git/` 目录内的任何内容
  （`HEAD`、`refs/`、`objects/`、`index`、`config`、`packed-refs` 等）——
  这在效果上等同于回退代码或改历史，同样不做；
- 改动 `.dsh-acl-recovery/`（DSH 沙箱工具链维护的目录，不属于项目内容）；
- 把工作区内的路径替换成指向工作区外的符号链接或目录联接，再借它操作外部仓库。

### 规则覆盖范围与例外

- **载体无关**：无论经由 pwsh、bash、脚本文件，还是委派给子代理执行，本节规则同样有效；
  不要通过"换个地方跑"来规避。
- **必须当面确认**：本节禁止清单中的任何操作，都必须由用户先看到说明、再明确同意，然后才可以执行。
  不得先做后报，也不得在用户未回复时自行推进。
- **授权只对当次有效**：一次同意只覆盖那一次具体操作，不构成长期或默认许可。
  以下说法一律不算授权：通常都这么干、自动模式、反正有权限、之前同意过类似的、你说过"看着办"。
- **未列入清单但有风险的操作**（例如批量改文件名、删除工作区外的文件、改动远端仓库设置），
  同样要先说明风险与影响，得到用户确认后再做。
- 若某项被禁操作确实是解决问题的最优路径，正确做法是：**停下来说明原因、影响范围与回退方案，
  请用户决定**（由用户手动执行，或由用户明确授权后你代为执行）。

## 仓库现状

- 仓库内容：`README.md`（占位）、`AGENTS.md`（本文件）、`.gitattributes` 与 `.editorconfig`
  （换行符与编辑器约定）、`.gitignore`（忽略 DSH 沙箱维护目录），
  以及工具链自动生成但不纳管的 `.dsh-acl-recovery/` 目录。
- **Git 事实**（2026-11-24 更新）：当前分支 `master`；历史为
  `16fa2b6 Initial commit` → `c71409e docs: 纳入项目约定与换行符规范` →
  `4fe2cec chore: 新增 .gitignore，忽略 DSH 沙箱维护的 .dsh-acl-recovery/` →
  `99325d9 docs: 记录 git 换行符的实测行为与陷阱`；
  远程 `origin` = `git@github.com:leon-orz/Infinite-ai.git`（SSH）；
  **`99325d9` 已推送，本地与 `origin/master` 同步**（以 `git status -sb` 为准，不要照抄本节的历史快照）。
- **提交信息规范**：采用 `类型: 中文简述` 的标题（常用 `feat`、`fix`、`docs`、`chore`、`refactor`、
  `test`、`ci`、`build`、`perf`、`style`、`revert`），标题用中文、不加句号；
  正文用中文分条写明**改了什么、为什么这么改**（必要处附依据）。
  已推送的提交信息**不要**为了"更规范"而改写（`--amend` 是禁止项）。
- **版本控制范围**：`AGENTS.md` 已按用户决定**纳入版本控制**，`.gitattributes`、`.editorconfig`、
  `.gitignore` 同样纳管；`.dsh-acl-recovery/` 由工具链维护，**不删除、不重命名、不纳入版本控制**
  （已由 `.gitignore` 挡在 `git status` 之外）。
- **行尾状态基线**：`git ls-files --eol` 应全部为 `i/lf  w/lf`（2026-11-23 已达成）；
  若出现 `w/crlf`，先读「Windows 环境与终端规范」里的"已实测确认的 git 行为"，**不要盲目试命令**。
- 提交时**显式列出路径**（如 `git add AGENTS.md .gitattributes .editorconfig`），
  不要使用 `git add -A` / `git add .`，避免把 `.dsh-acl-recovery/` 等无关内容一并带入。
- 技术栈、构建命令、测试命令、目录结构约定尚未确定。等代码落地后，在本节补充：
  语言与运行时版本、包管理器、常用命令（安装/构建/测试/格式化）、产物与忽略规则。

### 已核实的环境事实（2026-11-23 实测，非推测）

| 项 | 实测结果 |
| --- | --- |
| 操作系统 | Windows 11（`Microsoft Windows NT 10.0.26100.0`，24H2） |
| 终端 | PowerShell **7.7.0 Core**，`FullLanguage`，工作目录 `C:\Zone\1code\Infinite-ai` |
| shell 工具 | `$DSH_SHELL=1`；每个 `pwsh` 调用是独立进程，**不保留 `cd` 状态**，需要目录时用 `workdir` |
| 编码 | 系统 ANSI 代码页 936（GBK）；控制台输出编码 UTF-8；文本文件按 UTF-8 读写 |
| Node/npm | `node 24.13.0`、`npm`，经 Volta 管理 |
| 包管理器 | `pnpm` 可用（`C:\Program Files\Volta\pnpm.exe`） |
| Python | `python`、`uv` 可用 |
| 其他可用 | `git`（`C:\Program Files\Git\cmd\git.exe`）、`wsl`、`code` |
| **不可用/不在 PATH** | `rg`、`bash`、`cargo`、`go`。`bash` 仅存在于 `C:\Program Files\Git\bin\bash.exe`，需显式路径调用 |
| 文件策略 | `workspace-write`，工作区 `C:\Zone\1code\Infinite-ai`；越界操作会被拒绝或触发审批 |
| git 配置 | `core.autocrlf=true`、`core.ignorecase=true`、`core.filemode=false`、`init.defaultBranch=master`；`core.quotepath` 未设置 |
| 已知限制 | PowerShell 管道/变量捕获外部程序输出可能被拒（命名管道错误），详见「Windows 环境与终端规范」 |

- **本文档的维护时点**：环境事实变化时、技术栈落地时、新增硬性约定时，由当前 Agent 提出更新建议
  并在本节同步；不要静默修改已生效的强制条款。

## 本文件的加载位置（供排查参考）

- 用户全局链：`$DSH_HOME/AGENTS.md`（通常是 `~/.dsh/AGENTS.md`），
  以及 `$DSH_AGENTS_HOME/AGENTS.md`（通常是 `~/.agents/AGENTS.md`）；
- 项目链：从项目根（以 `.git` 等标记判定）到会话工作目录，逐层加载
  `AGENTS.md`、`CLAUDE.md`，随后叠加 `AGENTS.local.md`、`CLAUDE.local.md`。
