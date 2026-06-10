# Pi 调试手册

---

## 一、VSCode 调试配置

项目已预置 11 个 launch config（`.vscode/launch.json`），在 VSCode 中按 `F5` 即可选择使用。

### 1.1 Pi CLI 调试

| 配置名称 | 入口 | 说明 |
|----------|------|------|
| **Debug pi (交互模式)** | `packages/coding-agent/src/cli.ts` | 默认交互模式，`PI_OFFLINE=1` 离线运行 |
| **Debug pi (RPC 模式)** | 同上，加 `--mode rpc --no-session` | RPC JSONL 协议模式，使用外部终端 |
| **Debug pi (无 API key, 无 session)** | 同上，`--no-session` | 清空所有 API Key 环境变量，测试无认证场景 |
| **Debug pi (Benchmark 模式)** | 同上，`--no-session` | `PI_STARTUP_BENCHMARK=1`，测量启动性能 |

所有 Pi 调试配置共用：
- 运行时：`npx tsx`（源码直接执行，无需预构建）
- 环境变量：`PI_SKIP_VERSION_CHECK=1` 跳过版本检查
- Source Maps 开启，跳过 `node_modules` 和 Node 内部模块

### 1.2 测试调试

| 配置名称 | 框架 | 工作目录 | 说明 |
|----------|------|---------|------|
| **Debug 当前测试文件** | vitest | 仓库根 | 调试当前打开的测试文件 |
| **Debug coding-agent 全部测试** | vitest | `packages/coding-agent` | 设置 `PI_NO_LOCAL_LLM=1` |
| **Debug AI 包测试** | vitest | `packages/ai` | AI 包全部测试 |
| **Debug Agent 包测试** | vitest | `packages/agent` | Agent 包全部测试 |
| **Debug TUI 包测试** | `node:test` | `packages/tui` | TUI 使用 `npx tsx --test` 运行 |

### 1.3 其他调试

| 配置名称 | 说明 |
|----------|------|
| **Debug TUI 演示 (chat-simple)** | 运行 TUI 示例应用，验证 TUI 渲染 |
| **Attach to Node 进程** | 附加到已运行的 Node 进程（端口 9229），适用于远程或生产环境调试 |

---

## 二、运行时调试工具

### 2.1 `/debug` 命令（交互模式）

在 Pi 交互界面输入 `/debug`，将生成调试日志文件：

**输出路径：** `~/.pi/agent/pi-debug.log`

**日志内容：**
1. 时间戳 + 终端尺寸（宽×高）
2. 所有渲染行的原始 ANSI 字符串（带 visibleWidth）
3. 当前会话的完整消息列表（JSONL 格式）

**用途：** 排查 TUI 渲染问题、检查发给 LLM 的实际消息内容

### 2.2 环境变量

| 变量 | 用途 |
|------|------|
| `PI_OFFLINE=1` | 离线模式，不连接任何 LLM（开发时避免意外计费） |
| `PI_SKIP_VERSION_CHECK=1` | 跳过版本更新检查 |
| `PI_STARTUP_BENCHMARK=1` | 启动性能基准测量，输出各阶段耗时到 stderr |
| `PI_NO_LOCAL_LLM=1` | 测试时禁用本地 LLM 调用 |
| `PI_CODING_AGENT_DIR` | 自定义 agent 配置目录 |
| `PI_ALLOW_LOCKFILE_CHANGE=1` | 允许提交 lockfile 变更 |

### 2.3 交互模式热键

交互模式下常用调试相关快捷键：

| 快捷键 | 功能 |
|--------|------|
| `/debug` | 导出调试日志 |
| `/reload` | 重新加载配置和扩展 |
| `/compact` | 手动触发上下文压缩 |
| `Esc` | 中止当前操作 / 取消 |

---

## 三、启动性能分析

### 3.1 Benchmark 模式

```bash
# TUI 模式启动基准
npm run profile:tui

# RPC 模式启动基准
npm run profile:rpc

# 自定义参数
node scripts/profile-coding-agent-node.mjs \
  --mode tui \
  --runs 5 \
  --warmup 2 \
  --cpu-profile \
  --isolated-agent-dir
```

**参数说明：**

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--mode` | `tui` | `tui` 或 `rpc` |
| `--runs` | `1` | 测量次数 |
| `--warmup` | `0` | 预热次数（不计入结果） |
| `--label` | `{mode}-startup` | 输出文件前缀 |
| `--runtime` | `auto` | `node` / `bun` / `auto` |
| `--agent-dir` | - | 指定 agent 配置目录 |
| `--isolated-agent-dir` | - | 使用临时独立目录 |
| `--cpu-profile` | - | 生成 CPU profile 文件 |
| `--skip-build` | - | 跳过构建（Node 模式） |
| `--no-offline` | - | 不强制离线模式 |

**输出示例：**
```
[run 1] elapsed=452.3ms
  runtime:          node
  mode:             tui
  elapsed:          452.3ms
  init: 85.2ms
  config: 42.1ms
  extensions: 120.5ms
  session: 98.3ms
  tui_ready: 106.2ms
```

### 3.2 CPU Profile 分析

使用 `--cpu-profile` 生成 `.cpuprofile` 文件，可用以下工具分析：
- **Chrome DevTools**：`chrome://inspect` → 打开 CPU profile
- **VS Code**：安装 `ms-vscode.vscode-js-profile-flame` 扩展
- **Flamegraph**：使用 `0x` 或 `flamebearer` 生成火焰图

输出目录：`profiles-node/`（Node）或 `profiles-bun/`（Bun）

---

## 四、测试调试

### 4.1 运行测试

```bash
# 运行所有非 LLM 测试（无需 API Key）
./test.sh

# 运行全部测试
npm test

# 运行单个测试文件
# AI / Agent / Coding-Agent 包（vitest）
node ../../node_modules/vitest/dist/cli.js --run test/specific.test.ts

# TUI 包（node:test）
node --test test/specific.test.ts
```

### 4.2 测试结构

- **常规测试**：各包 `test/` 目录下的 `.test.ts` 文件
- **回归测试**：`packages/coding-agent/test/suite/regressions/<issue-number>-<short-slug>.test.ts`
- **测试 Harness**（agent 包）：
  - `packages/agent/src/harness/agent-harness.ts` — 完整的 agent 测试基础设施
  - 使用 `faux` 供应商（确定性 LLM 模拟），无需真实 API Key
  - 支持会话 JSONL/内存存储、上下文压缩验证

### 4.3 调试单个测试

```bash
# 命令行：使用 --inspect 启动 vitest
node --inspect-brk ../../node_modules/vitest/dist/cli.js --run test/specific.test.ts

# 或使用 VSCode 的 "Debug 当前测试文件" 配置，
# 在测试文件打开状态下按 F5
```

---

## 五、RPC 模式调试

### 5.1 启动 RPC

```bash
pi --mode rpc --no-session
```

通过 stdin/stdout 的 JSONL 协议与 Pi 通信。常用调试命令：

```bash
# 获取状态
echo '{"type":"get_state"}' | pi --mode rpc --no-session

# 发送提示（带 id 关联请求/响应）
echo '{"id":"req-1","type":"prompt","message":"ls"}' | pi --mode rpc --no-session
```

### 5.2 协议调试工具

**Python 脚本示例：**
```python
import subprocess, json
proc = subprocess.Popen(
    ["pi", "--mode", "rpc", "--no-session"],
    stdin=subprocess.PIPE, stdout=subprocess.PIPE, text=True
)
proc.stdin.write(json.dumps({"type": "prompt", "message": "Hello!"}) + "\n")
proc.stdin.flush()
for line in proc.stdout:
    event = json.loads(line)
    print(json.dumps(event, indent=2))  # 打印每个事件
```

**通过重定向分析事件流：**
```bash
echo '{"type":"prompt","message":"Hello"}' | pi --mode rpc --no-session > rpc-output.jsonl
# 然后分析 rpc-output.jsonl 文件
```

### 5.3 扩展 UI 协议调试

RPC 模式下的扩展 UI 交互（`select`、`confirm`、`input`、`editor`）会通过以下协议交换：

```
stdout ← {"type":"extension_ui_request","id":"uuid-1","method":"select",...}
stdin  → {"type":"extension_ui_response","id":"uuid-1","value":"Allow"}
```

查看 `examples/rpc-extension-ui.ts` 获取完整示例。

---

## 六、常见调试场景

### 6.1 渲染/UI 问题

```bash
# 1. 导出当前 UI 状态
# 在交互模式输入 /debug
# 查看 ~/.pi/agent/pi-debug.log 中的渲染行

# 2. tmux 环境下的按键问题
# 确保 ~/.tmux.conf 包含：
set -g extended-keys on
set -g extended-keys-format csi-u
# 需要 tmux 3.5+
```

### 6.2 LLM 调用问题

```bash
# 检查发送给 LLM 的消息
# /debug 日志包含 agent.messages 的完整 JSON

# 离线调试（不调用 LLM）
PI_OFFLINE=1 ./pi-test.sh

# 使用 faux 供应商（确定性输出）
PI_OFFLINE=1 ./pi-test.sh --no-env
```

### 6.3 扩展/Skills 加载问题

```bash
# 查看扩展加载诊断信息
# 启动时终端会显示扩展加载状态
# 检查扩展文件语法
npx tsx --check path/to/extension.ts
```

### 6.4 上下文压缩问题

```bash
# 手动触发压缩查看结果
# 在交互模式输入 /compact

# 查看压缩前 token 用量
# 底部 footer 栏显示当前上下文使用率
```

### 6.5 配置/启动问题

```bash
# 跳过版本检查
PI_SKIP_VERSION_CHECK=1 ./pi-test.sh

# 跳过 session 恢复
./pi-test.sh --no-session

# 使用独立 agent 目录
PI_CODING_AGENT_DIR=/tmp/test-pi ./pi-test.sh
```

---

## 七、源码关键入口点

| 用途 | 路径 |
|------|------|
| CLI 入口 | `packages/coding-agent/src/cli.ts` |
| 主流程 | `packages/coding-agent/src/main.ts` |
| 交互模式 | `packages/coding-agent/src/modes/interactive/interactive-mode.ts` |
| RPC 模式 | `packages/coding-agent/src/modes/rpc/rpc-mode.ts` |
| 智能体会话 | `packages/coding-agent/src/core/agent-session.ts` |
| 工具调用 | `packages/coding-agent/src/core/tools/` |
| 扩展加载 | `packages/coding-agent/src/core/extensions/loader.ts` |
| LLM 供应商 | `packages/ai/src/providers/` |
| Agent 循环 | `packages/agent/src/agent-loop.ts` |
| TUI 核心 | `packages/tui/src/tui.ts` |

---

## 八、快速参考

```bash
# 从源码启动调试
./pi-test.sh                           # 正常启动
PI_OFFLINE=1 ./pi-test.sh              # 离线模式
./pi-test.sh --no-session              # 无 session 恢复
./pi-test.sh -p "Hello"                # Print 模式单次执行

# 测试
./test.sh                              # 非 LLM 测试
npm test -- --run test/foo.test.ts     # 单文件测试

# 构建后调试
npm run build                          # 构建所有包
node packages/coding-agent/dist/cli.js # 运行构建产物

# 性能分析
npm run profile:tui                    # TUI 启动基准
npm run profile:rpc                    # RPC 启动基准

# 调试日志路径
~/.pi/agent/pi-debug.log              # /debug 命令输出
profiles-node/                         # CPU profile 输出
```
