# git-worktree-setup

[![skills.sh](https://skills.sh/b/imsai-sh/git-worktree-setup)](https://skills.sh/imsai-sh/git-worktree-setup)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

> English version: [README.md](README.md)

一个 [Agent Skill](https://agentskills.io/specification)，教 AI 编码 agent（Claude Code、Codex、Cursor、Gemini CLI 等）**如何把一个全新的 `git worktree` 引导到真正可运行的状态**——`node_modules` 链接好、`.env` 拷贝就位、dev 端口预留、本地数据库视情况共享或隔离——只需要一个仓库级脚本 + 对应 agent 工具的 hook。

## 要解决的问题

`git worktree add` 只 checkout 源代码，**别的什么都不管**。没有 `.env`，没有 `node_modules`，没有 virtualenv，没有端口预留。每次新建 worktree 都得（你自己或你的 agent）重做一遍同样的 setup，并在缺失的模块里来回 debug。

现有解法多半是各人手写的一次性 shell 脚本，里面塞着各种栈相关的坑（比如 hoisted 的根 `node_modules` 不一定能覆盖 `apps/*/node_modules`，因为 bundler 是从 workspace 内部解析；symlink `pgdata` 在并发写入下会损坏；secrets 根本不应该 symlink）。

## 这个 skill 做什么

skill 本身**不是** hook——它不会在每次新建 worktree 时被自动触发。它是个**生成器**，agent 在你说出下面这种话时跑**一次**：

> "帮我把这个仓库的 worktree 自动 setup 弄好。"

随后 agent 会：

1. **先自己 audit 仓库**——读 `package.json`、`.gitignore`、`docker-compose.yml`、agent 工具配置目录、自定义 Makefile / setup 脚本，以及任何相关文件。能从代码和配置推出来的事实，不来烦你。
2. **带着具体的初步方案上桌**——三档资源分类（共享 / 复制 / 生成）、hook 接线建议、手动入口。把它真的拿不准的开放问题（并发模型、自定义资源、多工具配合）单独列出来。
3. **只问 1-3 个尖锐问题**，不问 7 个。
4. **生成一份针对该仓库定制的 `scripts/setup-worktree.sh`** + 对应 hook 配置，然后验证幂等性 + 端到端 `dev` 启动。

后续项目结构改了，再跑一次它会 diff 编辑现有脚本。

### 特别契合 Claude Code Desktop 的 worktree 工作流

Claude Code Desktop（Mac/Windows）会为每个任务自动创建一个 `.claude/worktrees/<task-id>`，让每个对话都跟你的主 checkout 隔离开。如果不做 bootstrap，这些 worktree 一生下来就是死的——没 `node_modules`、没 `.env`、没分配端口，agent 头几轮全在重装依赖，根本还没碰到要改的代码。

每个仓库装一次这个 skill，以后每个 Desktop 任务的 worktree 落地就能直接 `npm run dev`（或你这边等价的命令）。CLI 上手动 `git worktree add` 的流程也吃这套——Desktop 只是因为创建得太频繁，让收益最明显。

## 安装

**推荐方式**——用 [`skills` CLI](https://github.com/vercel-labs/skills)。一条命令搞定 Claude Code、Codex、Cursor、OpenCode 以及 [50+ 其他 agent](https://skills.sh)：

```bash
npx skills add imsai-sh/git-worktree-setup
```

CLI 会自动检测你装了哪些 agent，把 skill 放到正确的目录（`.claude/skills/`、`.codex/skills/` 等）。`-a <agent>` 指定单个 agent，`-g` 全局安装，`-y` 跳过交互。

然后在任意对话里说一句 *"帮我把这个仓库的 worktree 自动 setup 弄好"*，你的 agent 会发现并调用这个 skill。

### 手动安装（备用）

不想跑 CLI 的话，直接把这个仓库 clone 到 agent 的 skills 目录：

```bash
# Claude Code，user 级
git clone https://github.com/imsai-sh/git-worktree-setup.git ~/.claude/skills/git-worktree-setup

# 或者项目级（提交进仓库）
git submodule add https://github.com/imsai-sh/git-worktree-setup.git .claude/skills/git-worktree-setup
```

对没有标准 skills 加载器的 agent，直接指给它看就行：*"读一下 `<路径>` 下的 SKILL.md，按它的工作流执行。"* skill 本质就是 markdown + 几个支持文件——这里没有任何 Claude Code 专属内容。

## 仓库结构

| 文件 | 内容 |
|---|---|
| [`SKILL.md`](SKILL.md) | agent 加载的 skill 规范。**英文，发布版本。** |
| [`SKILL.zh.md`](SKILL.zh.md) | 中文版（与英文版内容等价，便于中文开发者审阅 / 参与维护）。 |
| [`setup-worktree.sh`](setup-worktree.sh) | 可复用的脚本模板。放进 `<repo>/scripts/`，编辑底部资源声明块即可。包含 6 个幂等 helper：`link_resource`、`copy_resource`、`link_glob`、`hash_port`、`clean_branch_name`、`upsert_env`。 |
| [`recipes.md`](recipes.md) | 各栈即贴即用的代码块：npm workspaces / pnpm / Yarn Berry / Poetry / uv / Cargo / Go / Wrangler / Docker Compose / Postgres / WorktreeCreate-hook 严格 stdout 变体 + `worktree-remove.sh` 清理模板。 |
| [`hook-config.json`](hook-config.json) | Claude Code 的三种候选 hook 接线：`SessionStart`（最可移植）、`WorktreeCreate`（最干净，但只 Claude Code 有）、二者并用的双保险方案。 |

## 设计原则

- **先 audit，再问** —— agent 不该上来就丢一份问题清单。先读仓库，给出具体方案，只问需要判断的题。
- **三档分类，不留第四档** —— 每个缺失资源要么 Share、要么 Copy、要么 Generate。强制做选择。
- **幂等不能讨价还价** —— 脚本必须可重复运行。Helper 帮你处理这块；验证 checklist 强制兜底。
- **手动入口永远保留** —— 即便挂了 hook，`bash scripts/setup-worktree.sh` 也必须独立可跑。Hook 会失败、工具会变，脚本才是事实来源。
- **对未知坦诚** —— 对作者没充分了解过的 agent 工具（Gemini CLI、Aider），skill 明确告诉 agent 去问用户要文档，而不是凭空编 hook 配置。
- **发布版尽量精简** —— 大部分文字放在 `recipes.md` 和脚本注释里，不堆在 `SKILL.md`。

## 致谢

提炼自一个多 worktree monorepo 的实战经验，外加对两个先行项目的细读：

- [`tfriedel/claude-worktree-hooks`](https://github.com/tfriedel/claude-worktree-hooks) —— Claude Code `WorktreeCreate` hook 契约的权威参考（stdout 只能是路径、进度信息要写到 `/dev/tty`、`git worktree add` 输出抑制）。"按 branch 名 hash 推导端口"的思路也来自这里。
- [`mittalyashu/git-worktree`](https://github.com/mittalyashu/git-worktree) —— 纯手动脚本路线，每个 worktree 跑 `pnpm install` + `docker-compose up`。"不挂 agent hook、纯脚本"场景的有用基线。

这个 skill 把两者抽象成一个工具无关、结构无关的生成器。

## 许可证

MIT —— 见 [LICENSE](LICENSE)。

## 贡献

欢迎 issue 和 PR。特别欢迎：

- `recipes.md` 里加新的栈方案（Elixir/Phoenix、Java/Gradle、.NET 等）
- 验证非 Claude Code agent 工具（Codex、Cursor、Gemini CLI、Aider）的 hook 机制——把当前的"问用户"占位符换成具体配置
- `setup-worktree.sh` 里加常见模式的 helper
- 值得加进"常见错误"段落的真实失败案例
