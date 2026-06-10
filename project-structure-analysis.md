# Pi Monorepo 项目结构分析

> **Pi** — 一个开源的交互式编码智能体 CLI（类似 Claude Code / Copilot CLI），由 Mario Zechner 创建，基于 MIT 许可证发布。

---

## 一、顶层结构

```
pi/
├── .github/             # GitHub 配置：ISSUE_TEMPLATE、workflows (CI/CD)
├── .husky/              # Git hooks (pre-commit 运行 lint/typecheck/lockfile 检查)
├── .pi/                 # 项目级 Pi 配置目录
│   ├── extensions/      # 内置扩展 (prompt-url-widget, redraws, tps)
│   ├── git/             # git 相关配置
│   ├── npm/             # npm 相关配置
│   ├── prompts/         # 可复用提示模板 (cl, is, pr, sa, wr)
│   └── skills/          # Agent Skills (add-llm-provider 清单)
├── .vscode/             # VSCode 调试配置 (11 个 launch config) + 工作区设置
├── packages/            # ← 核心源码：4 个包
│   ├── tui/             #   终端 UI 库 (差分渲染)
│   ├── ai/              #   统一多供应商 LLM API
│   ├── agent/           #   智能体运行时 (工具调用/状态管理/传输层)
│   └── coding-agent/    #   Pi CLI 主应用
├── scripts/             # 构建/发布/检查脚本 (21 个)
├── node_modules/        # 依赖
├── package.json         # monorepo 根配置 (npm workspaces)
├── package-lock.json    # 锁定文件
├── tsconfig.base.json   # TypeScript 基础配置
├── tsconfig.json        # 根 tsconfig (path aliases + noEmit)
├── biome.json           # Biome 格式化/lint 配置 (3-space tab, 120 width)
├── AGENTS.md            # 核心开发规则 (用于 AI 开发参照)
├── CONTRIBUTING.md      # 贡献指南
├── SECURITY.md          # 安全策略
├── README.md            # 项目介绍 (英文)
├── README.zh-CN.md      # 项目介绍 (中文)
├── test.sh              # 运行全部测试
├── pi-test.sh           # 从源码启动 Pi
├── pi-test.bat / .ps1   # Windows 启动脚本
├── .nvmrc               # Node.js 版本 22.19.0
├── .npmrc               # npm 配置 (save-exact=true)
├── .gitignore / .gitattributes
└── LICENSE              # MIT
```

---

## 二、核心架构：4 个包

四个包**锁步版本号**（当前 0.79.0），构建顺序有严格依赖：**`tui → ai → agent → coding-agent`**

### 1. `@earendil-works/pi-tui` — 终端 UI 框架

```
packages/tui/
├── src/
│   ├── index.ts            # 导出入口
│   ├── tui.ts              # TUI 核心
│   ├── terminal.ts         # 终端抽象 (ProcessTerminal / VirtualTerminal)
│   ├── terminal-image.ts   # 图片渲染
│   ├── stdin-buffer.ts     # 标准输入缓冲
│   ├── keys.ts             # 按键常量
│   ├── keybindings.ts      # 按键绑定
│   ├── kill-ring.ts        # 剪贴环 (Emacs 风格)
│   ├── undo-stack.ts       # 撤销栈
│   ├── fuzzy.ts            # 模糊搜索
│   ├── autocomplete.ts     # 自动补全 (CombinedAutocompleteProvider)
│   ├── editor-component.ts # 编辑器组件框架
│   ├── native-modifiers.ts # 原生修饰键
│   ├── word-navigation.ts  # 单词导航
│   └── components/         # 内置 UI 组件
│       ├── box.ts          #   盒子容器
│       ├── text.ts         #   文本
│       ├── truncated-text.ts #   截断文本
│       ├── input.ts        #   输入框
│       ├── editor.ts       #   编辑器
│       ├── markdown.ts     #   Markdown 渲染
│       ├── loader.ts       #   加载动画
│       ├── cancellable-loader.ts # 可取消加载
│       ├── select-list.ts  #   选择列表
│       ├── settings-list.ts #   设置列表
│       ├── spacer.ts       #   间距
│       ├── image.ts        #   图片 (Kitty/iTerm2 协议)
│       ├── cancellable-loader.ts
│       └── ...
├── native/                 # 原生平台预编译包
├── test/                   # 测试 (使用 node:test)
├── vitest.config.ts
└── tsconfig.build.json
```

**核心特性：**
- **差分渲染**：三种渲染策略（首次、宽度变化/视口外全重绘、普通更新）
- **CSI 2026 同步输出**：无闪烁渲染
- **组件接口**：`render()`、`handleInput()`、`invalidate()`
- **叠加层系统**：可配置定位/锚点/边距/可见性
- **IME 支持**：通过 `CURSOR_MARKER` 和 Focusable 接口
- **括号粘贴模式**

### 2. `@earendil-works/pi-ai` — 统一 LLM API

```
packages/ai/
├── src/
│   ├── index.ts             # 导出入口
│   ├── types.ts             # 核心类型定义
│   ├── models.ts            # 模型管理
│   ├── models.generated.ts  # 自动生成的模型注册表
│   ├── image-models.ts
│   ├── image-models.generated.ts
│   ├── stream.ts            # 流式 API (stream/complete)
│   ├── images.ts            # 图片生成 API
│   ├── api-registry.ts      # 供应商注册中心
│   ├── images-api-registry.ts
│   ├── cli.ts               # CLI 入口 (pi-ai 命令)
│   ├── oauth.ts             # OAuth 入口
│   ├── env-api-keys.ts      # 环境变量 API Key 检测
│   ├── session-resources.ts # 会话资源文件管理
│   ├── providers/           # 各 LLM 供应商实现
│   │   ├── anthropic.ts
│   │   ├── openai-responses.ts
│   │   ├── openai-completions.ts
│   │   ├── openai-codex-responses.ts
│   │   ├── openai-responses-shared.ts
│   │   ├── openai-prompt-cache.ts
│   │   ├── google.ts        # Google Gemini
│   │   ├── google-vertex.ts # Google Vertex AI
│   │   ├── google-shared.ts
│   │   ├── amazon-bedrock.ts
│   │   ├── azure-openai-responses.ts
│   │   ├── mistral.ts
│   │   ├── cloudflare.ts
│   │   ├── github-copilot-headers.ts
│   │   ├── faux.ts          # 伪供应商 (测试用)
│   │   ├── simple-options.ts
│   │   ├── transform-messages.ts
│   │   ├── register-builtins.ts
│   │   └── images/          # 图片生成供应商
│   │       ├── openrouter.ts
│   │       └── register-builtins.ts
│   └── utils/
│       ├── abort-signals.ts
│       ├── diagnostics.ts
│       ├── event-stream.ts
│       ├── hash.ts
│       ├── headers.ts
│       ├── json-parse.ts
│       ├── node-http-proxy.ts
│       ├── overflow.ts
│       ├── sanitize-unicode.ts
│       ├── typebox-helpers.ts
│       ├── validation.ts
│       └── oauth/           # OAuth 实现
│           ├── anthropic.ts
│           ├── device-code.ts
│           ├── github-copilot.ts
│           ├── openai-codex.ts
│           ├── pkce.ts
│           └── types.ts
├── scripts/
│   ├── generate-models.ts      # 模型注册表生成器
│   └── generate-image-models.ts
├── test/
├── README.md
├── vitest.config.ts
└── tsconfig.build.json
```

**核心特性：**
- 支持 20+ 供应商：Anthropic、OpenAI、Google、Vertex AI、Mistral、DeepSeek、Groq、Cerebras、xAI、OpenRouter、NVIDIA NIM、Together AI、Cloudflare、Fireworks、Amazon Bedrock、Azure OpenAI、OpenCode、MiniMax、Kimi、小米 MiMo 等
- 统一 API：`stream()` / `complete()` / `streamSimple()` / `completeSimple()`
- 工具调用（TypeBox schema 验证）
- 思考/推理（跨供应商支持）
- 图片输入（视觉模型）
- 图片生成（`generateImages()` / `getImageModel()`）
- 跨供应商切换（自动消息格式转换）
- OAuth 流程（Anthropic、OpenAI Codex、GitHub Copilot）
- 模型注册表自动生成（`generate-models.ts` 脚本）
- Faux 供应商用于确定性测试

### 3. `@earendil-works/pi-agent-core` — 智能体运行时

```
packages/agent/
├── src/
│   ├── index.ts        # 导出入口
│   ├── types.ts        # 类型定义
│   ├── agent.ts        # Agent 类 (高层 API)
│   ├── agent-loop.ts   # agentLoop() 流式 API
│   ├── node.ts         # Node.js 环境入口
│   ├── proxy.ts        # 代理支持
│   ├── harness/        # 测试用 Harness（完整测试基础设施）
│   │   ├── agent-harness.ts      # 核心测试工具
│   │   ├── compaction/           # 上下文压缩
│   │   │   ├── compaction.ts
│   │   │   ├── branch-summarization.ts
│   │   │   └── utils.ts
│   │   ├── env/
│   │   │   └── nodejs.ts
│   │   ├── messages.ts
│   │   ├── prompt-templates.ts
│   │   ├── session/              # 会话存储
│   │   │   ├── session.ts
│   │   │   ├── jsonl-repo.ts     # JSONL 文件存储
│   │   │   ├── jsonl-storage.ts
│   │   │   ├── memory-repo.ts    # 内存存储
│   │   │   ├── memory-storage.ts
│   │   │   ├── repo-utils.ts
│   │   │   └── uuid.ts
│   │   ├── skills.ts             # Agent Skills 集成
│   │   ├── system-prompt.ts
│   │   ├── types.ts
│   │   └── utils/
│   │       ├── shell-output.ts
│   │       └── truncate.ts
├── docs/               # 文档 (harness, hooks, observability)
│   ├── agent-harness.md
│   ├── durable-harness.md
│   ├── hooks.md
│   └── observability.md
├── test/
├── README.md
├── vitest.config.ts
├── vitest.harness.config.ts
└── tsconfig.build.json
```

**核心特性：**
- **事件驱动循环**：`agent_start` / `turn_start` / `message_start` / `message_update` / `message_end` / `tool_execution_*`
- **AgentMessage**：通过声明合并可扩展
- **消息处理**：`transformContext()` + `convertToLlm()`
- **转向 (Steering)**：工具运行时中断并插入消息
- **后续 (Follow-up)**：完成后排队处理（one-at-a-time / all 模式）
- **工具执行模式**：`parallel`（默认，并发执行）或 `sequential`
- **钩子 (Hooks)**：`beforeToolCall`（拦截阻断）、`afterToolCall`（后处理，`terminate: true`）
- **完整测试 Harness**：上下文压缩（分支摘要）、会话存储（JSONL/内存）、Skills、提示模板

### 4. `@earendil-works/pi-coding-agent` — Pi CLI 主应用

这是最终产出的 CLI 工具，合并了 tui 的界面 + ai 的 LLM 能力 + agent 的运行时。

```
packages/coding-agent/
├── src/
│   ├── cli.ts                    # CLI 入口 (bin: pi)
│   ├── main.ts                   # 主逻辑
│   ├── config.ts                 # 配置加载
│   ├── index.ts                  # 导出入口 (SDK)
│   ├── sdk.ts                    # SDK 接口 (用于嵌入集成)
│   ├── migrations.ts             # 配置迁移
│   ├── package-manager-cli.ts    # Pi 包管理器 CLI
│   ├── cli/                      # CLI 辅助
│   │   ├── args.ts               # 参数解析
│   │   ├── config-selector.ts    # 配置选择器
│   │   ├── file-processor.ts     # 文件处理
│   │   ├── initial-message.ts    # 初始消息
│   │   ├── list-models.ts        # 列出模型
│   │   └── session-picker.ts     # 会话选择器
│   ├── core/                     # 核心逻辑
│   │   ├── agent-session.ts      # 智能体会话
│   │   ├── agent-session-runtime.ts
│   │   ├── agent-session-services.ts
│   │   ├── bash-executor.ts      # Bash 执行器
│   │   ├── session-manager.ts    # 会话管理
│   │   ├── session-cwd.ts        # 会话工作目录
│   │   ├── event-bus.ts          # 事件总线
│   │   ├── settings-manager.ts   # 设置管理
│   │   ├── trust-manager.ts      # 信任管理
│   │   ├── model-registry.ts     # 模型注册表
│   │   ├── model-resolver.ts     # 模型解析
│   │   ├── resource-loader.ts    # 资源加载
│   │   ├── prompt-templates.ts   # 提示模板
│   │   ├── system-prompt.ts      # 系统提示
│   │   ├── keybindings.ts        # 按键绑定
│   │   ├── messages.ts           # 消息
│   │   ├── defaults.ts           # 默认配置
│   │   ├── diagnostics.ts        # 诊断信息
│   │   ├── provider-attribution.ts
│   │   ├── provider-display-names.ts
│   │   ├── resolve-config-value.ts
│   │   ├── output-guard.ts       # 输出保护
│   │   ├── package-manager.ts    # 包管理器
│   │   ├── auth-guidance.ts      # 认证引导
│   │   ├── auth-storage.ts       # 认证存储
│   │   ├── footer-data-provider.ts
│   │   ├── telemetry.ts          # 遥测
│   │   ├── timings.ts            # 计时
│   │   ├── source-info.ts        # 源码信息
│   │   ├── skills.ts             # Skills
│   │   ├── slash-commands.ts     # 斜杠命令
│   │   ├── http-dispatcher.ts    # HTTP 调度
│   │   ├── exec.ts               # 进程执行
│   │   ├── extensions/           # 扩展系统
│   │   │   ├── types.ts
│   │   │   ├── loader.ts
│   │   │   ├── runner.ts
│   │   │   ├── wrapper.ts
│   │   │   └── index.ts
│   │   ├── compaction/           # 上下文压缩
│   │   │   ├── compaction.ts
│   │   │   ├── branch-summarization.ts
│   │   │   ├── utils.ts
│   │   │   └── index.ts
│   │   ├── tools/                # 内置工具
│   │   │   ├── index.ts
│   │   │   ├── read.ts           #   读取文件
│   │   │   ├── write.ts          #   写入文件
│   │   │   ├── edit.ts           #   编辑文件
│   │   │   ├── edit-diff.ts      #   diff 编辑
│   │   │   ├── bash.ts           #   执行命令
│   │   │   ├── grep.ts           #   搜索内容
│   │   │   ├── find.ts           #   搜索文件
│   │   │   ├── ls.ts             #   列出目录
│   │   │   ├── path-utils.ts
│   │   │   ├── truncate.ts
│   │   │   ├── render-utils.ts
│   │   │   ├── file-mutation-queue.ts
│   │   │   ├── output-accumulator.ts
│   │   │   └── tool-definition-wrapper.ts
│   │   └── export-html/          # HTML 导出
│   │       ├── index.ts
│   │       ├── ansi-to-html.ts
│   │       ├── tool-renderer.ts
│   │       ├── template.css
│   │       ├── template.html
│   │       ├── template.js
│   │       └── vendor/
│   │           ├── highlight.min.js
│   │           └── marked.min.js
│   ├── modes/                    # 运行模式
│   │   ├── index.ts
│   │   ├── interactive/          # 交互模式 (默认)
│   │   │   ├── interactive-mode.ts
│   │   │   ├── theme/            # 主题系统
│   │   │   │   ├── dark.json
│   │   │   │   ├── light.json
│   │   │   │   ├── theme-schema.json
│   │   │   │   └── theme.ts
│   │   │   ├── components/       # TUI 界面组件 (30+)
│   │   │   │   ├── footer.ts
│   │   │   │   ├── user-message.ts
│   │   │   │   ├── assistant-message.ts
│   │   │   │   ├── tool-execution.ts
│   │   │   │   ├── bash-execution.ts
│   │   │   │   ├── diff.ts
│   │   │   │   ├── model-selector.ts
│   │   │   │   ├── session-selector.ts
│   │   │   │   ├── theme-selector.ts
│   │   │   │   ├── settings-selector.ts
│   │   │   │   ├── config-selector.ts
│   │   │   │   ├── extension-*.ts
│   │   │   │   ├── login-dialog.ts
│   │   │   │   ├── trust-selector.ts
│   │   │   │   ├── compaction-summary-message.ts
│   │   │   │   ├── keybinding-hints.ts
│   │   │   │   └── ... (30+ 个组件)
│   │   │   └── assets/
│   │   ├── print-mode.ts         # Print 模式 (-p)
│   │   └── rpc/                  # RPC 模式
│   │       ├── rpc-mode.ts
│   │       ├── rpc-client.ts
│   │       ├── rpc-types.ts
│   │       └── jsonl.ts
│   ├── bun/                      # Bun 平台支持
│   │   ├── cli.ts
│   │   ├── register-bedrock.ts
│   │   └── restore-sandbox-env.ts
│   └── utils/                    # 通用工具 (20+)
│       ├── ansi.ts
│       ├── changelog.ts
│       ├── child-process.ts
│       ├── clipboard*.ts
│       ├── exif-orientation.ts
│       ├── frontmatter.ts
│       ├── fs-watch.ts
│       ├── git.ts
│       ├── html.ts
│       ├── image-convert.ts / image-resize*.ts
│       ├── json.ts
│       ├── mime.ts
│       ├── open-browser.ts
│       ├── paths.ts
│       ├── photon.ts
│       ├── pi-user-agent.ts
│       ├── shell.ts
│       ├── sleep.ts
│       ├── syntax-highlight.ts
│       ├── tools-manager.ts
│       ├── version-check.ts
│       └── windows-self-update.ts
├── examples/                     # 示例
│   ├── extensions/               #   扩展开发示例
│   │   ├── with-deps/            #     带依赖的扩展
│   │   ├── custom-provider-anthropic/
│   │   ├── custom-provider-gitlab-duo/
│   │   ├── sandbox/              #     沙箱扩展
│   │   └── gondolin/             #     Gondolin 扩展
│   ├── sdk/                      # SDK 使用示例
│   └── rpc-extension-ui.ts
├── docs/                         # 用户文档 (30 篇)
│   ├── quickstart.md
│   ├── usage.md
│   ├── sessions.md
│   ├── models.md
│   ├── providers.md
│   ├── settings.md
│   ├── extensions.md
│   ├── skills.md
│   ├── themes.md
│   ├── keybindings.md
│   ├── images/
│   └── ...
├── test/
│   └── suite/                    # 测试套件
│       ├── harness.ts            # 测试 Harness
│       └── regressions/          # Issue 回归测试
├── scripts/
├── npm-shrinkwrap.json           # 独立锁定文件 (发布用)
├── CHANGELOG.md
├── README.md
├── vitest.config.ts
├── tsconfig.build.json
├── tsconfig.examples.json
└── .gitignore
```

**运行模式：**
| 模式 | 命令 | 说明 |
|------|------|------|
| 交互模式 | `pi` (默认) | 全 TUI 交互，支持所有功能 |
| Print 模式 | `pi -p` | 非交互单次执行，输出到 stdout |
| JSON 模式 | `pi --mode json` | JSON 格式输出 |
| RPC 模式 | `pi --mode rpc` | 通过 JSONL 协议提供远程过程调用 |

**核心能力：**
- **内置工具**：read、write、edit、edit-diff、bash、grep、find、ls
- **扩展系统**：TypeScript 模块，可注册工具/命令/UI 组件/事件处理器
- **Skills**：按需加载的能力包（`/skill:名称`）
- **提示模板**：可复用 Markdown 模板（`/名称`）
- **主题系统**：JSON 格式，暗/亮内置，热重载
- **Pi 包管理**：通过 npm/git 打包分享扩展/skills/提示/主题
- **会话管理**：JSONL 存储，树形分支（`/tree`、`/fork`、`/clone`），自动压缩
- **认证**：支持订阅计划（Anthropic、OpenAI Codex、GitHub Copilot OAuth）+ 30+ API Key 供应商
- **HTML 导出**：将完整会话导出为独立 HTML 页面
- **SDK**：将 Pi 嵌入到其他应用中

---

## 三、构建与工具链

| 工具 | 用途 |
|------|------|
| **`tsgo`** | TypeScript 编译器（strip-only，tsc 的替代品） |
| **`tsx`** | 直接运行 .ts 文件（开发/测试用） |
| **`esbuild`** | 浏览器端打包验证 |
| **`Biome`** | 格式化 + lint（3-space tab、120 宽度） |
| **`vitest`** | 测试框架（ai、agent、coding-agent 包） |
| **`node:test`** | tui 包使用 Node 内置测试框架 |
| **`husky`** | Git hooks（pre-commit 自动检查） |

关键构建命令：`npm run build` 按序构建 4 个包，`npm run check` 运行全部检查。

---

## 四、依赖关系图

```
@earendil-works/pi-tui (终端 UI)
        ↓
@earendil-works/pi-ai (LLM API)
        ↓
@earendil-works/pi-agent-core (智能体运行时)
        ↓
@earendil-works/pi-coding-agent (CLI 主应用)
```

每个上层包依赖下层包：
- `pi-agent-core` → `pi-ai`
- `pi-coding-agent` → `pi-agent-core` + `pi-ai` + `pi-tui`

---

## 五、配置体系

```
全局配置 ~/.pi/               项目配置 .pi/ (仓库根目录)
├── agent/                     ├── extensions/
│   ├── settings.json          ├── git/
│   └── sessions.jsonl         ├── npm/
├── extensions/                 ├── prompts/
├── skills/                     └── skills/
└── themes/
```

配置作用域分层：全局（`~/.pi/`）→ 项目（`.pi/`）→ 当前目录。

Pi 还可以通过 `$PIO_CONFIG_DIR` 环境变量读取外部配置，以及通过 `-c` 标志指定备用会话文件。

---

## 六、脚本工具 (`scripts/`)

| 脚本 | 用途 |
|------|------|
| `release.mjs` | 版本发布自动化（bump → changelog → 构建 → commit + tag → push） |
| `release-notes.mjs` | GitHub Release notes 修复 |
| `publish.mjs` | npm 发布 |
| `local-release.mjs` | 本地测试发布 |
| `sync-versions.js` | 工作区间版本同步 |
| `generate-coding-agent-shrinkwrap.mjs` | 生成 coding-agent 独立锁定文件 |
| `check-pinned-deps.mjs` | 检查依赖是否锁定精确版本 |
| `check-ts-relative-imports.mjs` | 检查 TypeScript 相对导入规范 |
| `check-browser-smoke.mjs` | 浏览器端构建验证 |
| `check-lockfile-commit.mjs` | 锁定文件提交检查 |
| `profile-coding-agent-node.mjs` | Node.js 性能分析 |
| `browser-smoke-entry.ts` | 浏览器端入口点 |
| `cost.ts` | 使用成本计算 |
| `stats.ts` / `tool-stats.ts` | 使用统计 |
| `edit-tool-stats.mjs` / `read-tool-stats.mjs` | 编辑/读取工具统计 |
| `session-context-stats.mjs` | 会话上下文统计 |
| `session-transcripts.ts` | 会话转录 |
| `update-source-imports-to-ts.sh` | 导入路径更新脚本 |
| `build-binaries.sh` | 二进制构建 |

---

## 七、总结

Pi 是一个**四层架构**的编码智能体 CLI：

1. **界面层** (`tui`) — 高性能终端 UI 框架，差分渲染
2. **AI 层** (`ai`) — 20+ 供应商的统一 LLM 接口
3. **智能体层** (`agent`) — 通用智能体运行时，工具调用和状态管理
4. **应用层** (`coding-agent`) — 交互式编码助手 CLI，整合所有能力

设计理念强调**最小核心 + 可扩展**：核心功能紧凑，一切可通过扩展（Extensions）、技能（Skills）、提示模板（Prompt Templates）、主题（Themes）和 Pi 包来扩展，不需要的都挪到 `examples/extensions/` 中。
