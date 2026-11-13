# dsh-superagent（Windows 支持分支）

> 上游：[deepseek-ai/dsh-superagent-prototype](https://github.com/deepseek-ai/dsh-superagent-prototype) @ `8c32a9dc`（2026-10-31，MIT）
> 本仓库：`dcd887/dsh-superagent-prototype`，tag `v0.1.0-windows`
> 非官方分支，与 DSH 团队无关。

## 一句话

上游在 Windows 上从「`--help` 都跑不起来」修到「能启动、8 项 health ready、API 返回 200」——改了 8 处，全是兼容层，没碰核心逻辑。

## 为什么有这个分支

上游的启动脚本和部署层是按 POSIX 写的，在原生 Windows 上会连续撞上八处平台差异：`fcntl` 模块不存在、`ps`/`tmux`/`env` 命令不存在、`os.kill`/`os.umask` 语义不同、`subprocess` 用 GBK 解码 UTF-8 输出、`stateDir` 的 schema 只认 `/` 开头的路径、pnpm 的 `allowBuilds` 是占位符导致原生模块没编译。

这些不是设计缺陷，是平台假设。修法也不改架构：Windows 分支用 `msvcrt.locking`、`Get-CimInstance Win32_Process`、`subprocess.Popen`、显式 `encoding='utf-8'`、路径判断兼容盘符，以及把 `allowBuilds` 填实。

官方仓库当前不接受 Issue 与 PR（`has_issues: false`），所以这份兼容性工作以 fork 形式维护，没有走 upstream 流程。

## 改了什么

| # | 问题 | 根因 | 修复 |
|---|---|---|---|
| 1 | 启动即崩 `ModuleNotFoundError: No module named 'fcntl'` | POSIX 专有模块 | 条件导入，Windows 回退 `msvcrt.locking` |
| 2 | `ps` 命令不存在 | Windows 无 `ps` | `_ps_windows()` 走 `Get-CimInstance Win32_Process`（`wmic` 在 Win11 已移除，不可用） |
| 3 | `start` 报 `Both dsh and tmux must be on PATH` | 依赖 tmux 做进程托管 | Windows 走 `subprocess.Popen`，POSIX 保留 tmux |
| 4 | `os.kill(pid, SIGINT)` 无效、`os.umask` 无意义 | 信号语义不同 | Windows 用 `taskkill /PID /T /F`；跳过 `umask` |
| 5 | `UnicodeDecodeError: 'gbk' codec can't decode byte 0x93` | `subprocess.run(text=True)` 用系统 ANSI 码页解码 | 所有调用显式 `encoding='utf-8'` |
| 6 | `stateDir` 校验拒绝启动（插件不激活） | schema 用 `/^\//` 判断绝对路径，`C:\...` 不匹配 | 改为同时接受盘符与 `/` |
| 7 | `workerProbe` 失败 | 用 `env` 命令读环境 | 改为直接读进程环境 |
| 8 | 原生模块没编译（`node-pty` / `dsh-subprocess-local`） | `pnpm-workspace.yaml` 的 `allowBuilds` 是占位符 | 填实包名，重跑 install |

文件位置：1-5 在 `scripts/dsh-superagent`；6-7 在 `src/deployment.ts`；8 在 `pnpm-workspace.yaml`。

## 怎么用

### 方式一：下载预构建包（推荐，不用自己 build）

1. 下载 [dsh-superagent-win-v0.1.0.zip](https://github.com/dcd887/dsh-superagent-prototype/releases/download/v0.1.0-windows/dsh-superagent-win-v0.1.0.zip)（330 KB），解压到任意目录
2. 在解压目录里右键 `install.ps1` → 使用 PowerShell 运行
   - 脚本会自动找 DSH profile、在该 profile 的 `node_modules/@local` 下建 junction、然后打印配置片段
3. 按提示把配置片段加进 profile 的 `cordis.patch.yml`，重启 DSH

`stateDir` 是必填项，不填插件拒绝启动（上游行为，本补丁没改）。脚本会给出默认建议路径。

### 方式二：从源码 build

```bash
git clone https://github.com/dcd887/dsh-superagent-prototype
cd dsh-superagent-prototype
pnpm install
pnpm build

# 验证启动
node scripts/dsh-superagent start
node scripts/dsh-superagent status
curl http://127.0.0.1:<port>/orchestrator/api/state
node scripts/dsh-superagent stop
```

环境：Windows 11 · Node 24.19.0 · Python 3.14.7 · `@deepseek-ai/dsh 0.2.1-alpha.2`

### 版本兼容性（重要）

预构建包是按 `@deepseek-ai/dsh 0.2.1-alpha.2` 编译的，不保证跨版本兼容。

- DSH 升级后如果内部接口变了，这个包可能加载失败、插件不激活，或行为异常。
- 遇到这种情况，请改用「方式二：从源码 build」，并把依赖对齐到你当前的 dsh 版本。
- 本分支只在这个版本上做过完整验证（8 项 health ready），其他版本未测试。
- 上游 `dsh-superagent` 本身就是 research prototype，它自己的版本兼容性也不保证。

### 关于包名

本分支把 package 名从 `dsh-superagent` 改成了 `dsh-superagent-win`，author 改为本仓库维护者，`cordis.patch.yml` 里三处模块名同步改了。原因：避免与官方包名冲突，也避免解压后在 DSH 里显示成 DeepSeek 团队的作品。

源码里 `delivery.ts` 等处的 `dsh-superagent` 字符串是内部数据标识（消息来源类型、前端路由 key），不是模块引用，未改动。

## 验证结果（可复现）

| 检查 | 结果 |
|---|---|
| `dsh --version` | 0.2.1-alpha.2 |
| `--help` / `status` | 正常输出 |
| `ps()` Windows 分支 | 列出 16 个 `node.exe` 进程，命令行完整 |
| `start`（空配置） | 9.9 秒，8 项 health ready（两次独立复现） |
| `GET /orchestrator/api/state` | HTTP 200 |
| `stop` | 正常退出 |

原始记录见 `WINDOWS-ISSUES.md`（含每条的报错原文与验证命令）。

## 已知未解决

配了 GitHub Source 的 `repository` 之后，`dsh-superagent-github` 插件不激活。`gh` 本身正常，错误被 cordis loader 吞了，日志里看不到。

如果继续查，方向是：给 loader 加 debug 输出，或直接调 `resolveDeployment` 断点，看插件解析在哪一步返回空。

这个没修好，所以本分支只保证「能启动、能跑空配置」，不保证 GitHub 集成可用。

## 与上游同步

上游目前只有一个 commit（`8c32a9dc`，2026-10-31），本分支基于它。上游若更新，需要 rebase 后重跑上面那套验证。

## 反馈

遇到其他 Windows 问题，欢迎在本仓库开 Issue。请附上：

- 报错原文（文本，不要截图）
- 环境版本：Windows 版本 / Node / Python / `@deepseek-ai/dsh` 版本
- 复现命令

有完整复现步骤的优先看。

## 许可

MIT（与上游一致）。上游项目名与商标归其所有者。
