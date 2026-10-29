# JetBrains Git Control 仓库整理

整理日期：2026-10-29  
本地路径：`/Users/saku/Desktop/jetbrains-version-control`  
远程：https://github.com/witt-bit/jetbrains-version-control  
核对提交：`d27522ca852e5556524dbad606cb5ec86be3e5be`（2026-10-25，`docs: refresh changelog and READMEs`）  
分支：`main`，与 `origin/main` 一致

这份说明只依据本地检出、`package.json`、changelog、README，以及 GitHub 公开 issue。没有跑测试，也没有打开 Extension Development Host。

---

## 1. 这是什么

VS Code / Cursor 扩展，把 IntelliJ IDEA / JetBrains 的 Git 界面搬进编辑器，用来替代 VS Code 自带的源代码管理界面，不是在自带 Git 上再叠一层小功能。

| 项 | 值 |
|---|---|
| 仓库名 | `jetbrains-version-control` |
| 包名 | `jetbrains-git-control` |
| 显示名 | JetBrains Git Control |
| Marketplace | `witt-bit.jetbrains-git-control` |
| 发布者 | `witt-bit` |
| 当前版本 | `1.1.3` |
| 引擎 | VS Code `^1.91.0` |
| 语言 | TypeScript，webview 为 React |
| 许可 | MIT |
| 分类 | SCM Providers, Other |
| 激活 | `onStartupFinished`（随窗口启动，而不是第一次点开视图才激活） |
| 星标 / fork | 整理时公开页面约为 13 star、1 fork |
| 体积 | 约 5.6 MB（GitHub `size` 字段，单位 KB） |

产品口号（`package.json` description）：Git graph、IDEA 风格提交面板（含 shelf/stash）、分支管理、cherry-pick、rebase、merge、三路合并、行内 blame、右键菜单。

### 来源

不是从零写的，链路是：

1. [aotemj/jetbrains-git-graph](https://github.com/aotemj/jetbrains-git-graph)
2. [zhyc9de/jet-git](https://github.com/zhyc9de/jet-git)（IDEA 风格右键菜单和 UI）
3. 本仓库。这个 fork 增加了行内 Git blame 和提交图导航。

图标来自 IntelliJ IDEA Icons（Apache 2.0）。

### 环境

- VS Code 1.91.0 或更高。1.91 起 `vscode.lm` 消费端 API 稳定，AI 提交说明依赖它。
- 系统已安装 Git，且在 `PATH` 中。
- 包管理：pnpm。根目录和 `webview/` 要各装一次依赖。

本地开发：

```bash
git clone https://github.com/witt-bit/jetbrains-version-control.git
cd jetbrains-version-control
pnpm install
cd webview && pnpm install && cd ..
```

然后在 VS Code 里按 F5 打开扩展开发宿主。

常用脚本：

| 脚本 | 作用 |
|---|---|
| `pnpm run watch` | 扩展 + webview 监听 |
| `pnpm run build` | 类型检查、Biome、esbuild，再构建 webview |
| `pnpm run test:unit` | `node --test`，只跑 `src/**/*.test.ts` |
| `pnpm test` | `vscode-test`（扩展宿主测试；`pretest` 会先编译和 lint） |
| `pnpm run vsce:package` | 打 `.vsix` |
| `pnpm run lint` | `biome check src webview/src` |

---

## 2. 代码怎么拆

约 47 个宿主 TypeScript 文件，约 75 个 webview TS/TSX 文件。界面是 webview，和扩展进程之间用 `src/messages` 的 bridge，不共用一个 React 运行时。

```text
src/                  扩展宿主（Node / VS Code API）
  extension.ts        激活入口
  git/                git 调用、缓存、提交图布局、路径解码、类型
  blame/              行内 blame、光标行尾 blame
  decorations/        资源管理器着色、行号槽标记、悬停卡片
  views/              各 webview 的宿主侧 provider / manager
  ai/                 提交说明 prompt 与生成
  commands/           跳到源码、变更块动作
  messages/           webview 协议
  watchers/           git 状态监听
  statusBar.ts        状态栏分支项
  builtinGitPrompt.ts 询问是否关闭内置 Git
  dev/aiProbe.ts      仅 Development 模式注册的探测命令

webview/              独立前端
  commit/             Commit / Shelf / IDEA Shelf
  panel/              Git Log：图、分支树、详情、右键菜单
  conflicts/          冲突列表、三路合并
  push/               推送面板
  rollback/           回滚面板
  worktree/           worktree 面板
  shared/             bridge、store、主题、通用组件
```

构建：根目录 esbuild 打扩展，webview 自己的 pnpm 脚本打前端。Biome 做 lint。

---

## 3. 界面入口

| 位置 | ID | 内容 |
|---|---|---|
| 底部面板 | `git-brains-panel` | Git Log（`git-brains.gitLog`）、Worktrees（`git-brains.worktrees`） |
| 活动栏 | `git-brains-commit` | Commit webview；另有一个隐藏 TreeView 专门扛角标 |
| 状态栏 | `git-brains.status.menu` | 当前分支、ahead/behind；点击开常用操作 |
| 编辑器 | `jgc.editorGit` 等 | 右键收成一个一级 Git 子菜单 |

快捷键（macOS 用 Cmd，其它平台用 Ctrl）：

| 动作 | 键 |
|---|---|
| Push 面板 | `Cmd+Shift+K` / `Ctrl+Shift+K` |
| 下一个文件 diff | `Cmd+F7` |
| 上一个文件 diff | `Cmd+Shift+F7` |
| 跳到源码 | `F4`，以及 `Cmd+↓` / `Ctrl+↓` |
| 关掉变更块浮层 | `Escape`（仅浮层可见时） |

`Cmd+↓` 在 Commit / Git Log 面板里被从这个键上拿掉了 VS Code 的 `cursorBottom`，避免和「跳到源码」抢。

状态栏会跟着当前编辑器所在的工作区文件夹走（多根工作区）。detached HEAD 会显示出来，检出远程分支前提示会进入 detached。点击菜单：更新项目、提交、推送、新建分支、切换分支、打开 Git Log。

活动栏角标在窗口加载后就出现，不需要先点开 Commit。原因是 webview 第一次解析前没有 `WebviewView.badge`，所以用同容器里一个隐藏 TreeView 的 `TreeView.badge`。

---

## 4. 已发布能力（1.1.3 及更早，changelog 已记录）

版本号仍是 1.1.3。下面这些写在 1.1.3 和更早的 changelog 里，属于已经发布过的行为。第 5 节的三项写在 Unreleased，本地 `main` 有代码和文档，但没有新的 version / release。

### 4.1 Commit 面板

- 按勾选提交。已暂存但取消勾选的文件会在提交前移出 index，提交后再放回去。Commit 和 Commit and Push 同一规则。
- Amend：输入框已经有字时，不用上一次提交说明覆盖。
- 三个页：Commit、Shelf（stash）、IDEA Shelf（读取 / 恢复 JetBrains shelf）。
- 文件列表：单击选中，双击 diff，三击打开源文件；Shift 连选。
- 删除文件可以打开 diff：删除前内容对空文档。已暂存的删除对比 index，未暂存的对比空文档。`HEAD ↔ Staged` 的右侧是 index，不是工作区。
- 非 ASCII 路径会先解码。中文文件名不再显示成八进制转义，带空格、引号、字面量 `a -> b` 的路径也能操作。
- IDEA shelf 恢复时，反斜杠转义单遍解析，避免把字符串里的 `\n` 变成真换行；untracked 文件不再多一个空行；路径被 Git 加引号的文件不再从 shelf 里漏掉。
- 局部提交（Commit this change）从 `HEAD` 继承文件 mode，不再把可执行文件写成 `100644`。

### 4.2 编辑器变更标记

- 资源管理器文件名按 Git 状态着色：added、modified、deleted、untracked、ignored、conflict。
- 行号槽：新增/修改是色条，删除是短标记。被 ignore 的文件和非仓库目录不画。
- 行内 blame 打开时，标记固定在 blame 文字右侧同一列，没有标签的行也留出这列。
- 颜色 ID 是 `jgc.decoration.*`，可用 `workbench.colorCustomizations` 改。默认接近 IDEA File Status Colors。
- 三个开关：`jgc.decorations.enabled`、`fileColors`、`gutterMarkers`。
- 悬停变更行或标记，弹出该块的卡片：
  - 修改行做词级 diff
  - 删除块显示 `HEAD` 里的内容
  - 新增块主要是操作按钮
- 六个动作：上一块、下一块（文件内循环）、只回滚这一块、打开这一块的 diff、复制、只提交这一块。
- 只回滚这一块时，如果文件本来就有未保存修改，不会把整个文件强制保存。
- 只提交这一块走临时 index，文件其余暂存/未暂存状态保持原样。
- 右键 Git 菜单里的 Manage Selection...，以及行号右键，打开同一张浮层，直到你点一个动作。
- 行分类基于 jsdiff 的块语义（`removed+added` 配成一块修改）。中间只隔空行、任一侧超过 5 行时合成一块，和 VS Code quick diff 的启发式同类。

### 4.3 Blame

- 显示范围：`all` / `around` / 仅部分场景由 `jgc.blame.displayMode` 控制。`around` 的上下文行数默认 3。
- 列：revision、date、author、commitNumber。显示顺序固定，默认 `date` + `author`。
- 作者名风格、日期格式（`yyyy MM dd HH mm ss`）、按提交新旧着色（`jgc.blame.colorMode`，默认 `order`）。
- 浅色/深色前景色可配。
- diff 左右两侧各自开关。行号菜单只有一个 Toggle Git Blame Annotations，作用在打开菜单的那一列，而不是 `activeTextEditor`。
- 虚拟侧用该侧的 ref 做 `git blame <ref> -- <path>`。没有 git 版本的一侧（空占位、index、shelf 内容）会说明，而不是静默失败。
- 大文件的逐行标签用预建的行号映射，不再线性重扫。

### 4.4 Git Log

- 分支树：Local / Remote / Tags，可搜索。
- 提交列表带彩色分支线，Message / Author / Date / Hash 列宽可调。
- 详情面板：提交说明和变更文件树。
- 过滤：分支、作者、日期范围。
- 保存文件产生的 `status` 事件不再重建整张图，避免选中提交、折叠状态和滚动位置被清掉。只有提交、分支、标签真正变化才重建。
- 分支右键：Checkout、从该分支新建、Checkout 并 rebase 到当前、把当前 rebase 到该分支、合并进当前、重命名（本地）、删除（未合并可强制）、Update、Push。
- 提交右键：复制完整 hash、cherry-pick、checkout 该修订（detached）、reset 到这里（mixed / soft / hard）、revert、从这里建分支、建标签。
- 变更文件右键：Show Diff、Edit Source、打开该修订的文件版本、还原选中变更、把变更 cherry-pick 到工作区、复制路径、复制文件名。
- 带斜杠的本地分支（`dev/sub`）按实际 remote 名字判断，不再被硬编码前缀当成远程分支。
- Compare with Current 使用与当前分支的 merge-base patch，不再打开两个没有 ref 的空文档。
- 快速切换提交时，旧的文件列表请求不会画到新提交下面。

### 4.5 合并、冲突、rebase

- 三栏：Theirs | Result | Yours，冲突高亮，按块操作，带语法高亮。
- 冲突列表，并可从 SCM 资源上打开 JGC 合并编辑器。
- cherry-pick / rebase / merge 的 Continue 只暂存仍未合并的路径，不再 `git add -u` 把所有修改都收进去。
- Commit 面板在 rebase 中显示横幅：分支名、步数，以及 continue / skip / abort。
- Drop Commit 不再在 30 秒后假报超时；进度会等到 rebase 结束。不在当前分支历史上的提交会事先拒绝。

### 4.6 Worktree

- 侧边栏列表，显示分支和路径。
- 新建：可搜索的分支/标签（本地和 `origin/*`），有输入校验。
- 打开：双击后弹出原生对话框，当前窗口或新窗口。`jgc.worktree.openBehavior`：`ask`（默认）/ `thisWindow` / `newWindow`。
- 清理过期 worktree 元数据。
- Refresh 命令通知面板重新拉取，不再自己调用自己造成死循环。

### 4.7 回滚面板

按文件是否存在于 `HEAD` 分类。一个「已暂存的新文件又有未暂存修改」不再被当成 modified 然后从磁盘删掉。选择保留本地副本时，只是移出暂存区。

### 4.8 其它已发布的交互修正

- 面板标题不再强制全大写（`Unversioned Files`、`12 files`，而不是 `UNVERSIONED FILES`）。
- Commit、Git Log、Worktree 空白处右键不再冒出 webview 原生菜单；列表行仍用自定义菜单；输入框保留复制粘贴。这项写在 Unreleased 的 Fixed 里，和下面三块新功能放在同一次文档更新中，是否已进 1.1.3 的 Marketplace 包要以当时的 vsix 为准。本地 `main` 包含这次文档更新。

---

## 5. 本地 main 已有、但版本号仍是 1.1.3 的改动

changelog 顶部是 `## [Unreleased]`。`package.json` 的 `version` 没有抬高。因此：

- 克隆 `main` 能看到这些代码和 README 描述。
- Marketplace / Releases 上的 1.1.3（changelog 日期 2026-10-25）不应被当成已经包含下面三项。README 徽章仍写 Version 1.1.3，正文却已经描述这些功能，读文档时容易以为商店版也有。

### 5.1 询问关闭内置 Git

配置：`jgc.onboarding.builtinGitPrompt`，`ask` | `never`，默认 `ask`。

默认会弹一次窗，提议把全局 `git.enabled` 设为 false。理由是两套 Git UI 会重复状态栏、装饰和文件监听。确认后禁用，并提示可以一键重载。重载后源代码管理视图是空的，活动栏图标仍在（内置 Changes 视图没有 provider 也会留着）。对话框会说明：右键活动栏，取消勾选 Source Control。

选任一明确答案后，设置写成 `never`。改回 `ask` 会再次出现，不需要重载。

实现：`src/builtinGitPrompt.ts`。

### 5.2 光标行尾 blame

光标所在行末尾显示 `Who, 日期 * 摘要`，跟着光标走，不是全文件标注。

- 自己的提交显示 `You`
- 未提交的新行不显示
- 模板 `jgc.caretBlame.format`，默认 `${who}, ${date} * ${message}`，另有 `${hash}`
- message 默认最多 50 字，hash 最多 8 位，超出加省略号
- 总开关 `jgc.caretBlame.enabled`，默认 true
- blame 快照按文档版本缓存，保存和 Git 变化时刷新

有单元测试：`src/blame/caretBlameText.test.ts`。

### 5.3 AI 提交说明

Commit 按钮旁的 ✨。只为勾选的文件生成说明，流式写入 JGC 自己的输入框，提交前可改。

- 使用 `vscode.lm.selectChatModels` / `sendRequest`。JGC 自己没有 API key、base URL 配置项。
- GitHub Copilot 可直接用。其它模型要用户在 VS Code 的 Chat: Manage Language Models 里添加（OpenRouter、Ollama、OpenAI 兼容端点，或别的扩展注册进来的模型）。
- 多个模型时第一次问一次并记住；✨ 旁的下拉可以重选。
- 生成过程中再点一次是取消。没有任何模型时，提示去哪里添加。
- diff 范围：已暂存对比 index，未暂存对比工作区，未跟踪文件取内容。不会把没勾选的文件写进说明。
- 文风参照：当前提交者自己的最近提交；这个人在本仓库还没有提交时，退回仓库最近历史。生成语言跟随参照提交。
- prompt 有测试：`src/ai/prompt.test.ts`。

这是 issue #5 讨论之后落地的 Option A，不是「调用 Qoder/CodeGeeX 的生成命令」。

---

## 6. 配置一览

在设置里搜 `JGC`。

| 键 | 默认 | 作用 |
|---|---|---|
| `jgc.shortcut.jumpToSourceLabel` | `F4` | 跳到源码在 UI 上显示的标签 |
| `jgc.decorations.enabled` | true | 装饰总开关 |
| `jgc.decorations.fileColors` | true | 资源管理器着色 |
| `jgc.decorations.gutterMarkers` | true | 行号槽标记 |
| `jgc.blame.displayMode` | `all` | 行内 blame 显示范围 |
| `jgc.blame.aroundLines` | 3 | around 模式上下文 |
| `jgc.blame.columns` | date, author | 列 |
| `jgc.blame.colorMode` | `order` | 着色 |
| `jgc.blame.nameStyle` | `fullName` | 作者名风格 |
| `jgc.blame.dateFormat` | `yyyy-MM-dd HH:mm` | 日期格式 |
| `jgc.blame.foregroundColorLight` | `#57606a` | 浅色主题 |
| `jgc.blame.foregroundColorDark` | `#8b949e` | 深色主题 |
| `jgc.caretBlame.enabled` | true | 光标行尾 blame |
| `jgc.caretBlame.format` | `${who}, ${date} * ${message}` | 行尾模板 |
| `jgc.caretBlame.messageMaxLength` | 50 | 摘要长度 |
| `jgc.worktree.openBehavior` | `ask` | worktree 打开方式 |
| `jgc.onboarding.builtinGitPrompt` | `ask` | 是否再问关闭内置 Git |

---

## 7. 命令

用户向：

| 命令 | 标题 |
|---|---|
| `jgc.marker.prevChange` | Previous Change |
| `jgc.marker.nextChange` | Next Change |
| `jgc.marker.rollbackLines` | Rollback Lines |
| `jgc.marker.showDiffForLines` | Show Diff for Lines |
| `jgc.marker.copyBlock` | Copy Change Block |
| `jgc.marker.commitBlock` | Commit this Change |
| `jgc.marker.manageSelection` | Manage Selection... |
| `jgc.marker.closePopup` | Close Change Popup |
| `git-brains.status.menu` | Git Actions |
| `git-brains.refreshLog` | Refresh Git Log |
| `git-brains.openMergeEditor` | Open Merge Editor |
| `git-brains.openConflicts` | Conflicts |
| `git-brains.openMergeEditorFromSCM` | Open in JGC Merge Editor |
| `git-brains.showFileHistory` | Show File History |
| `git-brains.nextDiff` | Next File Diff |
| `git-brains.prevDiff` | Previous File Diff |
| `git-brains.commitPanel.refresh` | Refresh Changes |
| `git-brains.commitPanel.commit` | Commit |
| `git-brains.commitPanel.commitAndPush` | Commit and Push |
| `git-brains.commitPanel.shelve` | Shelve Changes |
| `git-brains.commitPanel.unshelve` | Unshelve Changes |
| `git-brains.commit.jumpToSource` | Jump to Source |
| `git-brains.openPushPanel` | Push... |
| `git-brains.editSource` | Edit Source |
| `git-brains.annotateWithGitBlame` | Toggle Git Blame Annotations |
| `git-brains.worktrees.new` | New Worktree... |
| `git-brains.worktrees.refresh` | Refresh Worktrees |

开发残留（见第 8 节）：`jgc.dev.aiProbe.hijack.*`、`jgc.dev.aiProbe.vendor.*`、`jgc.dev.aiProbe.generate`。代码里还有 `jgc.dev.aiProbe.test`，只在 Development 模式注册，`package.json` 的命令表里没有这一条。

贡献点菜单包括：`view/title`、`editor/title`、`scm/title`、`scm/resourceState/context`、`editor/context`、`jgc.editorGit`、`editor/lineNumber/context`、`comments/commentThread/context`、`commandPalette`、`explorer/context`。

---

## 8. 问题和未完成项

公开 issue（整理时 GitHub API 返回 9 条，其中 4 条是 PR）：

| 编号 | 状态 | 类型 | 内容 |
|---|---|---|---|
| #1 | closed | PR | worktree |
| #2 | closed | issue | Commit 窗口禁用鼠标选中样式 |
| #3 | closed | issue | 中文文件名显示为八进制转义，Git 操作失败 |
| #4 | closed | PR | 解码 Git 转义路径 |
| #5 | **open** | issue | AI 提交说明的调研和选项，讨论帖 |
| #6 | closed | PR | 中英国际化。closed，但主线界面仍然是英文，没有语言包 |
| #7 | closed | issue | 删除文件打不开 diff |
| #8 | closed | PR | 删除文件改为打开 diff |
| #9 | closed | PR | 接管 Git（和关闭内置 Git 相关） |

没有一串未关的功能 bug。代码里也几乎没有 `TODO` / `FIXME`。缺口是产品状态和发布卫生，不是某个已知崩溃。

### 8.1 版本号和文档脱节

`package.json` 是 1.1.3。changelog 的 Unreleased 已包含关闭内置 Git、光标行尾 blame、AI 提交说明。README 中英文都按这些功能来写，徽章仍是 1.1.3。下一个该做的发布动作是抬版本、写 release note、打 vsix，而不是再实现一遍。

### 8.2 AI 能力边界，和仓库里的旧结论矛盾

#5 调查过、也写进了 `docs/ai-commit-message-research.md`：

- Qoder（`Alibaba-Cloud.tongyi-lingma`）和 CodeGeeX（`AMiner.codegeex`）没有注册 `languageModelChatProviders`。它们的「生成提交说明」命令不返回文本，只写进 VS Code 自带 SCM 输入框。
- 这些命令自己决定 diff 范围（已暂存的全部，否则工作区全部），不能替用户只描述勾选的文件。
- 用临时改写 index 去迁就它们，会弄脏 Git 状态，维护者明确不这么做。
- SecretStorage 按扩展隔离，不能去读它们的 token。

文档开头仍是「已暂停，等待后续决策」，最后更新写着 2026-10-08。issue 在 2026-10-25 的追评里已经改口：引擎下限是 1.91 不是 1.104；用户自己加的模型由 VS Code BYOK 提供，所以 Option B 并进 Option A。代码已按这个做了。文档和 issue 标题没有改成「已实现，但只走 vscode.lm」。

结果：

- 有 Copilot，或在 VS Code 里加过模型的人，可以用 ✨。
- 只用 Qoder / CodeGeeX、没有把模型注册进 `vscode.lm` 的人，这个按钮没有通路。这是平台限制，不是少写一个适配器。

### 8.3 开发探测命令进了正式贡献点

`src/dev/aiProbe.ts` 文件头写明 TEMP、Dev-only, never ships。`registerAiProbe` 在 `extensionMode !== Development` 时直接 return。

但 `package.json` 的 `contributes.commands` 仍注册了 7 条：

- Test Hijack: Qoder / Copilot / CodeGeeX
- Test AI Commit Message Generation
- Test Vendor: Qoder / Copilot / CodeGeeX

正式安装的 vsix 里，这些标题会出现在命令面板，点击没有处理器。应该从贡献点删掉，只留开发模式动态注册。

探测内容是：`vscode.lm` 能否列出模型、自己发请求能否得到提交说明、调用上述三家命令时返回值是什么。注释写了它们经常返回 `undefined`，文字不回到调用方。

### 8.4 关闭内置 Git 的副作用

默认 `ask`。关掉 `git.enabled` 之后，任何依赖 VS Code 内置 Git 扩展 API 或 SCM 视图的其它扩展会失去仓库。JGC 自己的设计前提就是不依赖那套 UI，所以对它自己是自洽的，对用户的其它插件不是。选择会被记住，不会每次提醒。

### 8.5 测试覆盖

12 个 `*.test.ts`，没有 webview 组件测试，也没有 git 集成测试：

- `src/decorations/lineDiff.test.ts`
- `src/decorations/markerCard.test.ts`
- `src/decorations/statusColors.test.ts`
- `src/decorations/png.test.ts`
- `src/blame/caretBlameText.test.ts`
- `src/ai/prompt.test.ts`
- `src/git/pathUtils.test.ts`
- `webview/src/shared/hooks/useMultiClick.test.ts`
- `webview/src/shared/keyboard.test.ts`
- `webview/src/push/utils/dividerUtils.test.ts`
- `webview/src/push/utils/setup.test.ts`
- `webview/src/push/hooks/useDraggableDivider.test.ts`

提交、rebase、shelf、worktree、三路合并、状态栏、角标都没有自动化。changelog 里修过的回归（假超时、shelf 转义、斜杠分支、可执行位、勾选与 index 不一致）说明这些路径以前靠手测，漏过。

`pnpm run test:unit` 只包含 `src/**/*.test.ts`，webview 里那 5 个测试不在这条命令里。

### 8.6 国际化没落地

#6 已关闭，但界面文案、命令标题、右键菜单仍是英文。README 有 `README.zh_CN.md`。不能把「仓库有中文说明」当成「产品有中文 UI」。

### 8.7 没有下一阶段路线图

仓库里没有 roadmap。从 #5 和 Unreleased 能看出的未收尾只有：

1. 把 Unreleased 收成下一个版本并发布。
2. 更新 `docs/ai-commit-message-research.md` 和 issue #5 的状态，写明已实现范围和不支持 Qoder/CodeGeeX。
3. 去掉 `package.json` 里的 dev probe 命令。
4. 如果还要中文界面，需要新的 i18n，不要以为 #6 已经做完。

### 8.8 这次没有核对的事

- 没有执行 `pnpm test` 或 `pnpm run test:unit`，所以不知道当前提交测试是红是绿。
- 没有安装依赖，没有 F5。
- 没有对 diff 上游 `zhyc9de/jet-git`，因此不能列出这个 fork 相对上游的完整补丁列表，只能按 README 和 changelog 归纳。
- GitHub issue 列表来自公开 API，不包含 Discussions 以外的私密内容。整理时只有 #5 开着。

---

## 9. 一句话结论

这是一个功能已经相当全的 IDEA 风格 Git 扩展：提交图、勾选提交、shelf、变更块卡片、blame、三路合并、worktree 都在。公开 bug 基本修完了。当前真正的问题是发布和说明：`main` 比 1.1.3 商店版多了三块功能却没抬版本；AI 按钮只能用 VS Code 语言模型 API，旧调研文档还写着暂停；7 条开发探测命令不该出现在用户的命令面板里。
