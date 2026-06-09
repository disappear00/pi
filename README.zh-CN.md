<p align="center">
  <a href="https://pi.dev">
    <img alt="pi logo" src="https://pi.dev/logo-auto.svg" width="128">
  </a>
</p>
<p align="center">
  <a href="https://discord.com/invite/3cU7Bz4UPx"><img alt="Discord" src="https://img.shields.io/badge/discord-community-5865F2?style=flat-square&logo=discord&logoColor=white" /></a>
</p>
<p align="center">
  <a href="https://pi.dev">pi.dev</a> 域名由
  <br /><br />
  <a href="https://exe.dev"><img src="packages/coding-agent/docs/images/exy.png" alt="Exy mascot" width="48" /><br />exe.dev</a>
  慷慨捐赠
</p>

> 新贡献者的 Issues 和 PR 默认会自动关闭。维护者会每天审查自动关闭的 issue。详见 [CONTRIBUTING.md](CONTRIBUTING.md)。

---

# Pi Agent Harness 单体仓库

这是 pi agent harness 项目的主仓库，包含我们可自扩展的编码代理（coding agent）。

* **[@earendil-works/pi-coding-agent](packages/coding-agent)**：交互式编码代理 CLI
* **[@earendil-works/pi-agent-core](packages/agent)**：支持工具调用和状态管理的代理运行时
* **[@earendil-works/pi-ai](packages/ai)**：统一的多提供商 LLM API（OpenAI、Anthropic、Google 等）

了解更多关于 pi 的信息：

* [访问 pi.dev](https://pi.dev)，项目官网含演示
* [阅读文档](https://pi.dev/docs/latest)，你也可以直接让代理解释自己

## 分享你的开源编码代理会话

如果你在开源工作中使用了 pi 或其他编码代理，欢迎分享你的会话。

公开的开源会话数据有助于用真实世界的任务、工具使用、失败和修复来改进编码代理，而非依赖玩具基准测试。

完整说明请参见 [X 上的这篇文章](https://x.com/badlogicgames/status/2037811643774652911)。

发布会话请使用 [`badlogic/pi-share-hf`](https://github.com/badlogic/pi-share-hf)。阅读其 README.md 获取设置说明。你只需要一个 Hugging Face 账号、Hugging Face CLI 和 `pi-share-hf`。

你也可以观看[这个视频](https://x.com/badlogicgames/status/2041151967695634619)，了解我是如何发布我的 `pi-mono` 会话的。

我会定期在此发布自己的 `pi-mono` 工作会话：

- [badlogicgames/pi-mono 在 Hugging Face 上](https://huggingface.co/datasets/badlogicgames/pi-mono)

## 所有包

| 包 | 描述 |
|---------|-------------|
| **[@earendil-works/pi-ai](packages/ai)** | 统一的多提供商 LLM API（OpenAI、Anthropic、Google 等） |
| **[@earendil-works/pi-agent-core](packages/agent)** | 支持工具调用和状态管理的代理运行时 |
| **[@earendil-works/pi-coding-agent](packages/coding-agent)** | 交互式编码代理 CLI |
| **[@earendil-works/pi-tui](packages/tui)** | 支持差异渲染的终端 UI 库 |

Slack/聊天自动化和工作流请参见 [earendil-works/pi-chat](https://github.com/earendil-works/pi-chat)。

## 权限与容器化

Pi 不包含内置权限系统来限制文件系统、进程、网络或凭据访问。默认情况下，它以启动它的用户和进程权限运行。

如果你需要更强的隔离边界，请将 Pi 容器化或沙箱化。详见 [packages/coding-agent/docs/containerization.md](packages/coding-agent/docs/containerization.md)，包含三种模式：

- **OpenShell**：在策略控制的沙箱中运行整个 `pi` 进程。
- **Gondolin 扩展**：将 `pi` 和提供商认证保留在宿主机上，同时将内置工具和 `!` 命令路由到本地 Linux 微型虚拟机中。
- **纯 Docker**：为简单隔离，在本地容器中运行整个 `pi` 进程。

## 贡献

详见 [CONTRIBUTING.md](CONTRIBUTING.md) 了解贡献指南，以及 [AGENTS.md](AGENTS.md) 了解项目特定规则（适用于人类和代理）。

## 开发

```bash
npm install --ignore-scripts  # 安装所有依赖，不运行生命周期脚本
npm run build        # 构建所有包
npm run check        # 代码检查、格式化和类型检查
./test.sh            # 运行测试（无 API 密钥时跳过 LLM 相关测试）
./pi-test.sh         # 从源码运行 pi（可从任何目录执行）
```

## 供应链加固

我们将 npm 依赖变更视为需要审查的代码变更。

- 直接外部依赖固定到精确版本。内部工作区包保持版本范围。
- `.npmrc` 设置了 `save-exact=true` 和 `min-release-age=2`，以避免在 npm 解析期间使用当天发布的依赖。
- `package-lock.json` 是依赖的唯一真实来源。除非设置了 `PI_ALLOW_LOCKFILE_CHANGE=1`，否则预提交钩子会阻止意外的 lockfile 提交。
- `npm run check` 验证固定的直接依赖、原生 TypeScript 导入兼容性以及生成的 coding-agent shrinkwrap。
- 发布的 CLI 包包含 `packages/coding-agent/npm-shrinkwrap.json`，从根 lockfile 生成，用于为 npm 用户固定传递依赖。
- 发布冒烟测试使用 `npm run release:local` 进行构建、打包，并在标记发布之前在仓库外部创建隔离的 npm 和 Bun 安装。
- 本地发布安装、文档化的 npm 安装以及 `pi update --self` 在支持的地方使用 `--ignore-scripts`。
- CI 使用 `npm ci --ignore-scripts` 安装，并有一个定时 GitHub 工作流运行 `npm audit --omit=dev` 和 `npm audit signatures --omit=dev`。
- Shrinkwrap 生成具有明确的依赖生命周期脚本白名单；新的生命周期脚本依赖在审查之前会触发检查失败。

## 许可证

MIT
