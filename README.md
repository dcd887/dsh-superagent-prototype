# dsh-superagent (Windows fork)

> **This fork adds Windows support to `deepseek-ai/dsh-superagent-prototype`.**
>
> The upstream repository does not accept pull requests (`has_issues: false` and the
> `pulls` API returns 404), so the Windows fixes live here.
>
> **What changed** (2 files, +86 −22):
> - `scripts/dsh-superagent` — no `fcntl` / `tmux` / `ps` on Windows: `msvcrt.locking`,
>   `Get-CimInstance`, `Popen`, `taskkill`, skip `umask`, UTF-8 decode for every `subprocess`.
> - `src/deployment.ts` — `stateDir` accepts Windows paths (`C:\\`, `C:/`, `\\server`),
>   and `workerProbe` no longer shells out to `env(1)` on Windows.
>
> **Verified on Windows 11** (Python 3.14.7, Node 24.19.0, DSH 0.2.1-alpha.2):
> `--help` works, `status` works, `ps()` lists 16 node processes, `start` brings the
> profile up in 9.9 s with 8 health components `ready`, and `/orchestrator/api/state`
> returns HTTP 200.
>
> **Known issue**: configuring the GitHub Source's `repository` makes
> `dsh-superagent-github` fail to activate and takes the main plugin down with it.
> `gh` itself works (`gh api user` → the agent account, `push: true`); the failure is
> in plugin activation and the error is swallowed by the cordis loader. Without a
> repository configured everything starts fine.
>
> **Windows setup note**: before the first `pnpm install`, set every entry under
> `allowBuilds` in `pnpm-workspace.yaml` to `true` (pnpm writes
> `set this to true or false` placeholders). Otherwise
> `@deepseek-ai/dsh-subprocess-local`'s `postinstall` never runs, `spawn` fails, and
> the GitHub Source check reports `spawn env ENOENT`.
>
> Full log: [WINDOWS-ISSUES.md](WINDOWS-ISSUES.md)
>
> ---

# dsh-superagent

> **Research prototype for reference only, not an official DeepSeek product.** This code grew out of our team's day-to-day development work, with much of the engineering done by coding agents. Treat it as research code, not production-ready software. It may still contain bugs, and its coding style and engineering practices may be inconsistent. Review and adapt it before using it in your own environment.
>
> We have adapted it for publication, so it is not identical to our internal version or the implementation described in *Exploring Super-agent Design: Towards a More Autonomous Agent System*. Some designs discussed in the article are directions for future development and are not implemented in this prototype.

dsh-superagent is a proactive coding agent for one GitHub repository, packaged as a bundle for [DeepSeek Harness](https://www.deepseek.com/en/harness/). It listens to GitHub and, optionally, Feishu. When someone allowed asks the agent for work, it opens a work item, works on it in its own agent session and directory, and answers in the original conversation, for example with a pull request. A web panel shows what needs a person.

It is also a way of working with an agent: **take as little human attention as possible, and give as much value as possible.** An agent that can act for a whole team can also waste a whole team's attention, so it acts only when asked or when a rule the team wrote allows it, and speaks only when someone must act or can use a result. [docs/000-philosophy.md](docs/000-philosophy.md) explains the principle and what it asks of the operator.

## Setup

Setup has many steps, so an agent does it with you. Ask your coding agent:

> Read SETUP.md in this repository and set up dsh-superagent with me.

It asks one short question at a time, shows the full plan once, and then installs and configures everything. [SETUP.md](SETUP.md) is what it follows.

## Documentation

Read the numbered documents in order; each builds only on the ones before it.

- [000 Philosophy](docs/000-philosophy.md)
- [001 Domain model](docs/001-domain.md)
- [002 Authorization](docs/002-authorization.md)
- [003 Execution](docs/003-execution.md)
- [004 Prompts](docs/004-prompts.md)
- [005 Panel](docs/005-panel.md)
- [006 Deployment](docs/006-deployment.md): manual setup, configuration, upgrades and troubleshooting
- [007 Architecture](docs/007-architecture.md)
- [008 Cookbook](docs/008-cookbook.md): adding a Source

## Development

`pnpm check` runs the typecheck, the tests and the build.

## License

[MIT](LICENSE)
