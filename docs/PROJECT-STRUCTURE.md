# Pi Agent 项目结构文档

## 概述

Pi 是一个自扩展编码助手（Coding Agent）CLI 工具，采用 npm workspaces 单仓库架构，包含 4 个核心包。

- **仓库**: `https://github.com/earendil-works/pi.git`
- **License**: MIT
- **Node 要求**: >=22.19.0
- **模块系统**: ESM (`"type": "module"`)
- **TypeScript**: 5.9.3，使用 `tsgo` 原生编译

---

## 包依赖关系

```
@earendil-works/pi-tui          (无内部依赖)
       │
       ▼
@earendil-works/pi-ai           (无内部依赖)
       │
       ▼
@earendil-works/pi-agent-core   (依赖 pi-ai)
       │
       ▼
@earendil-works/pi-coding-agent (依赖 pi-ai, pi-agent-core, pi-tui)
```

构建顺序: `tui → ai → agent → coding-agent`

---

## 顶层目录结构

```
pi/
├── .github/                    # CI/CD、issue 模板、贡献者审批
├── .husky/                     # Git hooks (pre-commit)
├── .pi/                        # 项目本地配置（extensions, skills, prompts）
├── packages/                   # 4 个核心包
│   ├── tui/                    # 终端 UI 库
│   ├── ai/                     # LLM 多提供商抽象层
│   ├── agent/                  # Agent 运行时
│   └── coding-agent/           # 编码助手 CLI（主包）
├── scripts/                    # 发布、构建、检查脚本
├── AGENTS.md                   # AI agent 开发规则
├── biome.json                  # Linter/formatter 配置
├── CONTRIBUTING.md             # 贡献指南
├── package.json                # 根 workspace 配置
├── tsconfig.json               # 根 TypeScript 配置
├── tsconfig.base.json          # 共享编译器选项
├── test.sh                     # 测试运行器
└── .npmrc                      # save-exact=true, min-release-age=2
```

---

## 包详解

### 1. `@earendil-works/pi-tui` (packages/tui)

**用途**: 终端用户界面库，提供差分渲染（differential rendering）能力。

**核心文件**:

| 文件 | 说明 |
|------|------|
| `src/tui.ts` | TUI 核心类（Container, Component, Focusable, Overlay） |
| `src/terminal.ts` | 终端抽象层（ProcessTerminal） |
| `src/keys.ts` | 按键解析，Kitty 协议支持 |
| `src/keybindings.ts` | 快捷键系统 |
| `src/autocomplete.ts` | 自动补全 |
| `src/fuzzy.ts` | 模糊匹配 |
| `src/stdin-buffer.ts` | stdin 输入缓冲 |
| `src/kill-ring.ts` | 剪贴板历史（kill ring） |
| `src/undo-stack.ts` | 撤销/重做栈 |
| `src/terminal-image.ts` | 终端图片渲染（Kitty, iTerm2 协议） |
| `src/editor-component.ts` | 编辑器组件接口 |

**UI 组件** (`src/components/`):

| 组件 | 说明 |
|------|------|
| `box.ts` | 盒子容器 |
| `editor.ts` | 文本编辑器 |
| `image.ts` | 图片显示 |
| `input.ts` | 输入框 |
| `markdown.ts` | Markdown 渲染器 |
| `select-list.ts` | 可选列表 |
| `settings-list.ts` | 设置列表 |
| `text.ts` | 文本显示 |
| `truncated-text.ts` | 截断文本 |
| `spacer.ts` | 间隔 |
| `loader.ts` | 加载指示器 |

---

### 2. `@earendil-works/pi-ai` (packages/ai)

**用途**: 统一的多提供商 LLM API 抽象层，自动模型发现和提供商配置。

**核心文件**:

| 文件 | 说明 |
|------|------|
| `src/types.ts` | 核心类型: `Model<Api>`, `Message`, `StreamFunction`, `Api`, `Provider` |
| `src/stream.ts` | `streamSimple()` — 统一流式调用函数 |
| `src/models.ts` | 模型定义与注册表 |
| `src/models.generated.ts` | 自动生成的模型元数据（**勿手动编辑**） |
| `src/image-models.generated.ts` | 自动生成的图像模型元数据 |
| `src/env-api-keys.ts` | 环境变量 API 密钥解析 |
| `src/api-registry.ts` | API/提供商注册系统 |
| `src/session-resources.ts` | 会话资源管理 |
| `src/oauth.ts` | OAuth 导出 |

**支持的 API 协议**:

| 协议 | 提供商 |
|------|--------|
| `anthropic-messages` | Anthropic |
| `openai-completions` | OpenAI Chat Completions |
| `openai-responses` | OpenAI Responses |
| `openai-codex-responses` | OpenAI Codex (WebSocket) |
| `azure-openai-responses` | Azure OpenAI |
| `google-generativeai` | Google Generative AI |
| `google-vertex` | Google Vertex AI |
| `bedrock-converse-stream` | AWS Bedrock |
| `mistral-conversations` | Mistral AI |

**提供商实现** (`src/providers/`):

| 文件 | 提供商 |
|------|--------|
| `anthropic.ts` | Anthropic Messages API |
| `openai-completions.ts` | OpenAI Chat Completions |
| `openai-responses.ts` | OpenAI Responses |
| `openai-codex-responses.ts` | OpenAI Codex Responses (WebSocket) |
| `azure-openai-responses.ts` | Azure OpenAI Responses |
| `google.ts` | Google Generative AI |
| `google-vertex.ts` | Google Vertex AI |
| `amazon-bedrock.ts` | AWS Bedrock |
| `mistral.ts` | Mistral AI |
| `cloudflare.ts` | Cloudflare Workers AI |
| `faux.ts` | 测试用 provider |

**工具函数** (`src/utils/`):

| 文件 | 说明 |
|------|------|
| `json-parse.ts` | 部分 JSON 解析器 |
| `event-stream.ts` | AssistantMessageEventStream |
| `oauth/` | OAuth 提供商实现 |
| `abort-signals.ts` | 中止信号处理 |
| `headers.ts` | HTTP header 工具 |

---

### 3. `@earendil-works/pi-agent-core` (packages/agent)

**用途**: 通用 Agent 运行时，提供传输抽象、状态管理和附件支持。

**核心文件**:

| 文件 | 说明 |
|------|------|
| `src/agent.ts` | `Agent` 类 — 有状态的 agent 包装器 |
| `src/agent-loop.ts` | 底层 agent 循环 (`runAgentLoop`) |
| `src/types.ts` | 核心类型: `AgentEvent`, `AgentTool`, `AgentState`, `AgentContext` |

**Harness 层** (`src/harness/`):

| 文件 | 说明 |
|------|------|
| `agent-harness.ts` | `AgentHarness` — 高级 agent，带 session/compaction/skills |
| `messages.ts` | 消息转换 (`convertToLlm`) |
| `prompt-templates.ts` | 提示词模板系统 |
| `skills.ts` | Skill 加载与格式化 |
| `system-prompt.ts` | 系统提示词构建 |

**上下文压缩** (`src/harness/compaction/`):

| 文件 | 说明 |
|------|------|
| `compaction.ts` | 上下文压缩（token 估算、摘要） |
| `branch-summarization.ts` | 分支摘要生成 |

**会话存储** (`src/harness/session/`):

| 文件 | 说明 |
|------|------|
| `session.ts` | Session 接口 |
| `jsonl-repo.ts` | 基于 JSONL 文件的会话存储 |
| `memory-repo.ts` | 内存会话存储 |
| `uuid.ts` | UUID v7 生成 |

**Agent 生命周期事件**:

```
agent_start → turn_start → message_start/update/end
    → tool_execution_start/update/end → turn_end → agent_end
```

---

### 4. `@earendil-works/pi-coding-agent` (packages/coding-agent)

**用途**: 交互式编码助手 CLI，这是用户直接使用的主包。

**二进制**: `pi` (dist/cli.js)

**运行模式**:

| 模式 | 说明 |
|------|------|
| `interactive` (默认) | 完整 TUI，差分渲染，主题支持，快捷键 |
| `print` (`--print`) | 非交互式文本/JSON 输出 |
| `json` (`--json`) | JSON 输出模式 |
| `rpc` (`--rpc`) | JSON-RPC 协议模式，用于外部 UI 集成 |

**入口文件**:

| 文件 | 说明 |
|------|------|
| `src/cli.ts` | CLI 入口点 |
| `src/main.ts` | 主函数（参数解析、会话管理、模式分发） |
| `src/index.ts` | SDK 导出（366 行） |

**核心模块** (`src/core/`):

| 文件 | 说明 |
|------|------|
| `agent-session.ts` | `AgentSession` — 主会话编排器 |
| `model-registry.ts` | `ModelRegistry` — 模型发现与注册 |
| `model-resolver.ts` | 模型解析（从 CLI 参数） |
| `auth-storage.ts` | API 密钥存储（环境变量 → auth.json → CLI） |
| `settings-manager.ts` | 设置持久化 (`~/.pi/agent/settings.json`) |
| `session-manager.ts` | 会话文件管理 (.jsonl) |
| `system-prompt.ts` | 系统提示词构建 |
| `resource-loader.ts` | 加载 extensions, skills, themes, context files |
| `skills.ts` | Skill 加载（从 `.pi/skills/`） |
| `slash-commands.ts` | 斜杠命令 (`/compact`, `/help` 等) |
| `keybindings.ts` | 应用快捷键 |
| `extensions/` | 扩展系统 |
| `compaction/` | 上下文压缩 |
| `export-html/` | HTML 会话导出 |

**内置工具** (`src/core/tools/`):

| 工具 | 说明 |
|------|------|
| `bash.ts` | Bash shell 执行 |
| `read.ts` | 文件读取 |
| `edit.ts` | 文件编辑（基于 diff） |
| `write.ts` | 文件写入 |
| `find.ts` | 文件查找（基于 fd） |
| `grep.ts` | 内容搜索（基于 ripgrep） |
| `ls.ts` | 目录列表 |

**交互模式 UI** (`src/modes/interactive/`):

| 组件 | 说明 |
|------|------|
| `interactive-mode.ts` | 交互模式主逻辑 |
| `components/assistant-message.ts` | 助手消息渲染 |
| `components/bash-execution.ts` | Bash 执行显示 |
| `components/model-selector.ts` | 模型选择器 |
| `components/session-selector.ts` | 会话选择器 |
| `components/settings-selector.ts` | 设置选择器 |
| `components/theme-selector.ts` | 主题选择器 |
| `components/tool-execution.ts` | 工具执行显示 |
| `components/diff.ts` | Diff 显示 |
| `components/login-dialog.ts` | 登录对话框 |
| `theme/*.json` | 主题定义 |

**扩展系统** (`src/core/extensions/`):

| 文件 | 说明 |
|------|------|
| `types.ts` | Extension, ExtensionFactory, ExtensionRuntime 类型 |
| `loader.ts` | 扩展发现与加载 |
| `runner.ts` | 扩展事件执行 |
| `wrapper.ts` | 工具包装 |

扩展可注册: 工具、命令、快捷键、事件处理器、widgets、overlays、消息渲染器、自动补全提供者。

---

## 配置文件

| 文件 | 路径 | 用途 |
|------|------|------|
| `models.json` | `~/.pi/agent/models.json` | 自定义提供商、模型覆盖 |
| `auth.json` | `~/.pi/agent/auth.json` | API 密钥和 OAuth token 存储 |
| `settings.json` | `~/.pi/agent/settings.json` | 全局设置（默认模型、思考级别等） |
| `settings.json` | `<project>/.pi/settings.json` | 项目级设置覆盖 |

**环境变量**:

| 变量 | 用途 |
|------|------|
| `ANTHROPIC_API_KEY` | Anthropic API 密钥 |
| `OPENAI_API_KEY` | OpenAI API 密钥 |
| `GEMINI_API_KEY` | Google Gemini API 密钥 |
| `DEEPSEEK_API_KEY` | DeepSeek API 密钥 |
| `OPENROUTER_API_KEY` | OpenRouter API 密钥 |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | AWS Bedrock |
| `PI_CODING_AGENT_DIR` | 配置目录覆盖 |
| `PI_OFFLINE` | 离线模式 |
| `PI_TELEMETRY` | 遥测控制 |

---

## API 密钥解析优先级

1. CLI `--api-key` 参数
2. `auth.json` 中存储的凭据
3. 环境变量
4. `models.json` 中的 fallback resolver

---

## 模型选择优先级

1. CLI 参数 (`--provider` + `--model`)
2. 首个可用的 scoped 模型
3. `settings.json` 中保存的 `defaultProvider` / `defaultModel`
4. 第一个有有效 API 密钥的可用模型

---

## CI/CD

| 工作流 | 触发条件 | 说明 |
|--------|----------|------|
| `ci.yml` | push/PR to main | 构建、检查、测试 |
| `build-binaries.yml` | Tag push (v*) | 构建二进制、创建 Release、发布 npm |
| `pr-gate.yml` | PR opened | 自动关闭未批准贡献者的 PR |
| `npm-audit.yml` | Daily cron | 生产依赖审计 |

**发布流程**: Tag push → 构建 Bun 二进制（6 个平台）→ GitHub Release → npm trusted publishing

---

## 测试

**运行测试**:

```bash
# 根目录（安全模式，不泄露 API key）
./test.sh

# 单个包
cd packages/ai && npx vitest --run

# 特定测试文件
cd packages/coding-agent && node ../../node_modules/vitest/dist/cli.js --run test/specific.test.ts
```

**测试统计**:

| 包 | 测试文件数 |
|----|-----------|
| pi-tui | 31 |
| pi-ai | 84 |
| pi-agent-core | 6 |
| pi-coding-agent | 115 |

**注意**: 永远不要直接运行完整的 vitest 套件（包含 e2e 测试，会检测 API key 并激活）。

---

## 关键架构模式

### 分层架构

1. **pi-ai**: 纯 LLM 抽象层 — 无 UI，无会话管理
2. **pi-agent-core**: Agent 运行时 — 工具执行循环、事件系统、会话存储
3. **pi-coding-agent**: 应用层 — CLI、TUI、扩展、工具、会话
4. **pi-tui**: UI 原语 — 独立于 agent 逻辑

### 扩展系统

扩展是从以下位置发现的 TypeScript 文件:
- `.pi/extensions/`（项目本地）
- `~/.pi/agent/tools/`（全局）
- CLI `--extensions` 标志

扩展可注册: 工具、命令、快捷键、事件处理器、widgets、overlays、消息渲染器、自动补全提供者。

### 会话管理

- 会话存储为 `.jsonl` 文件
- 支持: 会话分叉、分支、树导航
- 上下文压缩: token 估算 + LLM 驱动的摘要

---

## 构建命令

```bash
npm run build              # 构建所有包
npm run check              # Lint + 类型检查
npm run release:patch      # 发布补丁版本
npm run release:minor      # 发布次版本
npm run release:local      # 本地发布测试
```
