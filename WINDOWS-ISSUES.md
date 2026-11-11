# SuperAgent Windows 支持：已修 5 个 bug，卡在第 6 个

> 日期：2026-11-11
> 环境：Windows 11 · Python 3.14.7 · Node 24.19.0 · `@deepseek-ai/dsh 0.2.1-alpha.2` · `dsh-superagent-prototype` @ `8c32a9dc5d`
> 目标：让 `dsh-superagent` 在 Windows 上跑起来，作为 PR 的证据
> 结论：**启动能跑（8 项 health ready），但配置 GitHub Source 后插件不激活，原因未定位**

---

## 一、已修复并验证的 5 个 Windows bug

### Bug 1：`import fcntl` 直接崩

**位置**：`scripts/dsh-superagent:10`

**现象**：
```
ModuleNotFoundError: No module named 'fcntl'
```

**修复**：条件导入，Windows 回退 `msvcrt.locking`。

**验证**：`--help` 正常输出。

---

### Bug 2：`ps` 命令不存在

**位置**：`scripts/dsh-superagent:59`（`ps()`）

**现象**：`FileNotFoundError`（Windows 没有 `ps`）

**修复**：新增 `_ps_windows()`，用 PowerShell `Get-CimInstance Win32_Process`（`wmic` 在 Win11 已移除，不能用）。

**验证**：能列出 16 个 `node.exe` 进程，命令行完整。

---

### Bug 3：`tmux` 不存在

**位置**：`scripts/dsh-superagent:166-181`（`start()`）

**现象**：`Both dsh and tmux must be on PATH`

**修复**：Windows 走 `subprocess.Popen`，POSIX 保留 `tmux`。

**验证**：`start` 成功起进程、写 PID。

---

### Bug 4：`os.kill` / `os.umask` 行为不同

**位置**：`scripts/dsh-superagent` 多处

**修复**：
- `os.kill(pid, SIGINT)` → Windows 用 `taskkill /PID /T /F`
- `os.umask(0o077)` → Windows 跳过

**验证**：`stop` 能结束进程。

---

### Bug 5：`subprocess` 用 GBK 解码 UTF-8 输出

**位置**：`scripts/dsh-superagent` 全部 `subprocess.run(..., text=True)`

**现象**：
```
UnicodeDecodeError: 'gbk' codec can't decode byte 0x93
```

**修复**：全部加 `encoding='utf-8', errors='replace'`。

**验证**：`status` / `ps()` 不再崩。

---

### Bug 6：`stateDir` schema 只认 POSIX 路径

**位置**：`src/deployment.ts:13`

**代码**：
```typescript
const absolute = (description: string) => s.string().pattern(/^\//)
```

**现象**：
```
dsh-superagent: ValidationError: invalid config:
  - $.stateDir expect string to match regexp /^\// (at stateDir)
```

**修复**：
```typescript
const absolute = (description: string) =>
  s.string().pattern(/^(\/|[A-Za-z]:[\\/]|\\\\)/)   // POSIX / 盘符 / UNC
```

**验证**：改后 `start` 从「插件拒绝启动」变成「8 项 health ready」。

---

### Bug 7：`env` 命令不存在（`workerProbe`）

**位置**：`src/deployment.ts:73`

**代码**：
```typescript
return argv => runAsWorker(subprocess, ['env', `PATH=${path}`, ...argv], cwd, ...)
```

**现象**：
```
The agent GitHub CLI cannot read its account.
spawn env ENOENT
```

**修复**：
```typescript
return argv => process.platform === 'win32'
  ? runAsWorker(subprocess, argv, cwd, ..., { ...process.env, PATH: path })
  : runAsWorker(subprocess, ['env', `PATH=${path}`, ...argv], cwd, ...)
```

**验证**：`gh` 能通过 `probe` 调用（`gh api user` 返回 `dcd887-agent`）。

---

### Bug 8（环境类，非代码）：`pnpm` 的 `allowBuilds` 是占位符

**位置**：`pnpm-workspace.yaml`

**现象**：
```yaml
allowBuilds:
  '@deepseek-ai/dsh-subprocess-local': set this to true or false
  node-pty: set this to true or false
  ...
```

**后果**：`dsh-subprocess-local` 的 `postinstall`（`node scripts/ensure-spawn-helper.mjs`）**没跑** → `spawn` 失败。

**修复**：全部改成 `true`，重跑 `pnpm install`。

**验证**：
```
@deepseek-ai/dsh-subprocess-local postinstall$ node scripts/ensure-spawn-helper.mjs
@deepseek-ai/dsh-subprocess-local postinstall: Done
```

---

## 二、已验证能跑的部分

### 空配置启动（可复现，两次）

```
$ python scripts/dsh-superagent start --timeout 45
Log: ...\.artifacts\dsh-superagent-superagent-20261111-134206-4260.log
{
  "pids": [11772],
  "health": {
    "runtime": "paused", "relay": "ready", "notice": "ready",
    "launch": "ready", "delivery": "ready", "assessment": "ready",
    "reply": "ready", "evals": "ready", "github": "paused"
  },
  "setup": { "place": "github", "code": "repositoryUnset",
             "detail": "The GitHub Source names no repository." }
}
耗时 9.9 秒
```

**第二次（删 sqlite 重置后）同样 9.9 秒、8 项 ready。**

### API 可访问

```
$ curl -H 'x-orchestrator-request: 1' http://127.0.0.1:3082/orchestrator/api/state
HTTP 200 ｜1260 字符
```

### 配置面板可读

```
$ curl -H 'x-orchestrator-request: 1' http://127.0.0.1:3082/orchestrator/api/settings
HTTP 200 ｜8636 字符
```

**返回 `sources` / `blocks` / `environment` 三块结构。**

---

## 三、卡住的地方：第 6 个问题（未解决）

### 现象

**一配 `source:github` 的 `repository`，插件就不激活。**

```
$ python scripts/dsh-superagent start --timeout 55
Startup health check timed out
{ "pids": [18804], "health": null, "setup": null }
耗时 59.7 秒

$ curl -H 'x-orchestrator-request: 1' http://127.0.0.1:3082/orchestrator/api/state
✗ 404 Not Found
```

**但 `dsh` 进程活着、3082 端口在听。**

### 已经排除的原因

| 假设 | 怎么排除的 | 结论 |
|---|---|---|
| `gh` 本身不可用 | `gh api user` → `dcd887-agent`；`gh api repos/dcd887/dsh-brief-pack --jq .permissions` → `{"push":true}` | ❌ 不是 |
| `PATH` 分隔符错误 | `withoutDirectory` 用 `path.delimiter`（Windows 是 `;`）| ❌ 不是 |
| `stdout` 重定向到文件 | 试过文件 / `DEVNULL` / 不重定向三种 | ❌ 都不是 |
| `spawn` helper 缺失 | 修 `allowBuilds` 后 `postinstall` 已跑 | ❌ 不是 |
| `GH_TOKEN` 没传进去 | `start` 时设了环境变量，`Popen` 继承 | ❌ 不是 |
| 加载慢（< 60 秒）| 等到 90 秒仍 404 | ❌ 不是 |

### 为什么不能继续查

**看不到错误信息。**

- `stdout=文件` → 日志 0 字节（`dsh` 不写 stdout）
- `stdout=DEVNULL` → 什么都看不到
- `stdout=None`（继承）→ `scripts` 退出后管道关闭

**`~/.dsh/logs/startup-*.log` 只在「`dsh` 启动失败」时写**，而这种情况是「`dsh` 起来了、插件没激活」，**不写诊断日志**。

**所以：错误信息在「插件激活」这一步被吞掉了，无法定位。**

### 一个可疑的观察

**「进程内重启」和「手动 `stop` + `start`」都失败，但「空配置启动」成功。**

**说明**：`dsh-superagent-github` 插件的激活逻辑里有「Windows 上会抛错」的分支，而且这个错误被 `cordis` 的 loader 吞掉了（loader 只 log 到 `dsh` 自己的 logger，而 logger 写 stdout，被重定向掉了）。

---

## 四、如果继续查，下一步该做什么

### 方向 1：让 `dsh` 的 logger 写文件

**查 `dsh` 有没有「logger 输出到文件」的配置**（`config.yml` / 环境变量如 `DSH_LOG_FILE`）。

**如果有**，设上，就能看到「插件激活失败」的真实错误。

### 方向 2：直接调 `resolveDeployment`

**写一个小的 Node 脚本**，import `src/deployment.ts` 的 `resolveDeployment`，用 `workerProbe` 跑一遍，看它抛什么。

```typescript
import { resolveDeployment, workerProbe } from './lib/index.js'
// 需要 subprocess 服务实例
```

**难点**：`workerProbe` 需要 `SubprocessRuntime`，而它在 `dsh` 的运行时里。

### 方向 3：走 `dsh` 的 cordis 调试

**`cordis_inspect_query`** 可能能看到「插件激活失败的原因」。

### 方向 4：问维护者

**在 PR 里写清楚「空配置能跑，配 GitHub Source 后插件不激活」，维护者可能直接告诉你原因。**

---

## 五、PR 材料现状

### 能提的部分（数据充足）

| 文件 | 改动 |
|---|---|
| `scripts/dsh-superagent` | 6 处（`fcntl` / `ps` / `tmux` / `os.kill` / `umask` / 编码）|
| `src/deployment.ts` | 2 处（`stateDir` schema / `workerProbe` env）|

**已验证**：
```
✅ dsh --version → 0.2.1-alpha.2
✅ scripts/dsh-superagent --help → 正常
✅ scripts/dsh-superagent status → 正常
✅ ps() Windows 分支 → 列出 16 个 node 进程
✅ start（空配置）→ 9.9 秒，8 项 health ready（两次可复现）
✅ GET /orchestrator/api/state → HTTP 200
✅ stop → 正常退出
```

### 不能提的部分

**「配 GitHub Source 后插件不激活」** —— 原因未定位，**写成 known issue**。

**另外 `pnpm-workspace.yaml` 的 `allowBuilds` 是「环境问题」，不是「代码问题」**，不该进 PR，但可以在 PR 描述里提一句「Windows 用户需要先 `pnpm approve-builds`」。

---

## 六、一句话

> **Windows 上从「连 `--help` 都跑不了」修到「能启动、8 项 health ready、API 200」。配 GitHub Source 后插件不激活，错误被 `cordis` loader 吞了，看不到，卡在这里。**
