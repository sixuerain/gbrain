# Agents working on GBrain

This is your install + operating protocol. Claude Code reads `./CLAUDE.md` automatically.
Everyone else (Codex, Cursor, OpenClaw, Aider, Continue, or an LLM fetching via URL):
start here.

> **Becoming someone's persistent personal agent** (identity + memory + private repo)?
> Follow [`BOOTSTRAP_FOR_AGENTS.md`](./BOOTSTRAP_FOR_AGENTS.md) — the `gbrain bootstrap`
> flow — instead of the plain install below, then come back here for the operating
> protocol. Connecting to an EXISTING remote brain from a laptop agent?
> `gbrain connect https://your-host/mcp --token gbrain_xxx --install` (see the MCP
> table in [`README.md`](./README.md)).

## Install (5 min)

<!-- npm-trap + #218 recovery: canonical copy lives in README.md ("Install" warning) — sync edits. -->
1. Install gbrain via Bun (the canonical path):
   ```bash
   curl -fsSL https://bun.sh/install | bash
   export PATH="$HOME/.bun/bin:$PATH"
   bun install -g github:garrytan/gbrain
   ```
   If `bun install -g` aborts or `gbrain doctor` reports `schema_version: 0`,
   the CLI prints a recovery hint pointing at [#218](https://github.com/garrytan/gbrain/issues/218).
   Run `gbrain apply-migrations --yes` to recover, or fall back to the
   deterministic install: `git clone https://github.com/garrytan/gbrain.git ~/gbrain && cd ~/gbrain && bun install && bun link`.
2. Init the brain: `gbrain init` (defaults to PGLite, zero-config). For 1000+ files or
   multi-machine sync, init suggests Postgres + pgvector via Supabase.
3. **STOP — ask the user about search mode.** `gbrain init` auto-applied a
   default but printed a 9-cell cost matrix (mode × downstream model)
   preceded by `[AGENT]` markers. You MUST relay the matrix to the operator
   and confirm their choice before continuing. Cost spread between corners
   is 25x — silent acceptance is the wrong default. See
   [`./INSTALL_FOR_AGENTS.md`](./INSTALL_FOR_AGENTS.md) Step 3.5 for the
   exact ask-the-user protocol. Same banner fires on `gbrain post-upgrade`
   for existing users (search modes were added in v0.32.3).
4. Read [`./INSTALL_FOR_AGENTS.md`](./INSTALL_FOR_AGENTS.md) for the full step-by-step
   flow (API keys, identity, cron, verification).

## Read this order

1. `./AGENTS.md` (this file) — install + operating protocol.
2. [`./CLAUDE.md`](./CLAUDE.md) — orientation + resolver: architecture, cross-cutting
   invariants, the reference map, inline ship rules. It routes to on-demand detail docs:
   [`./docs/architecture/KEY_FILES.md`](./docs/architecture/KEY_FILES.md) (per-file index —
   read a file's entry before editing it), [`./docs/TESTING.md`](./docs/TESTING.md) (test
   tiers + isolation lint + E2E lifecycle), and
   [`./docs/architecture/thin-client.md`](./docs/architecture/thin-client.md) (remote-MCP seam).
3. [`./docs/architecture/brains-and-sources.md`](./docs/architecture/brains-and-sources.md)
   — the two-axis mental model (brain = which DB, source = which repo in the DB). Every
   query routes on both axes. Read before writing anything that touches brain ops.
4. [`./skills/conventions/brain-routing.md`](./skills/conventions/brain-routing.md) —
   agent-facing decision table: when to switch brain, when to switch source, how
   cross-brain federation works (latent-space only; the agent decides).
5. [`./skills/RESOLVER.md`](./skills/RESOLVER.md) — skill dispatcher. Read before any task.

## Trust boundary (critical)

GBrain distinguishes **trusted local CLI callers** (`OperationContext.remote = false`,
set by `src/cli.ts`) from **untrusted agent-facing callers** (`remote = true`, set by
`src/mcp/server.ts`). Security-sensitive operations like `file_upload` tighten filesystem
confinement when `remote = true` and default to strict behavior when unset. If you are
writing or reviewing an operation, consult `src/core/operations.ts` for the contract.

## Common tasks

- **Configure:** [`docs/ENGINES.md`](./docs/ENGINES.md),
  [`docs/guides/live-sync.md`](./docs/guides/live-sync.md),
  [`docs/mcp/DEPLOY.md`](./docs/mcp/DEPLOY.md).
- **Bring in your chat history:** `gbrain transcripts ingest` imports a
  downloaded ChatGPT / Claude export (or agent session logs); `gbrain connectors`
  connects the account and syncs new conversations live, incrementally and on an
  opt-in schedule (cookie/OAuth credentials stay on your machine, 0600). Full
  guide: [`docs/guides/chat-connectors.md`](./docs/guides/chat-connectors.md).
- **Debug:** [`docs/GBRAIN_VERIFY.md`](./docs/GBRAIN_VERIFY.md),
  [`docs/guides/minions-fix.md`](./docs/guides/minions-fix.md), `gbrain doctor --fix`.
  Database unreachable — or any `GBRAIN_DB_ACCESS <reason>` marker in gbrain
  output: `gbrain engine status --probe` (which engine, where its URL comes from,
  classified reachability), then `gbrain db-repair` to diagnose and
  `gbrain db-repair --yes` to apply safe fixes. All three are engine-free — they
  work while the database is down. Full loop:
  [`docs/ENGINES.md`](./docs/ENGINES.md#engine-detection-and-access-repair).
- **Migrate / upgrade:** `gbrain upgrade` (binary self-update + schema migrations + post-upgrade prompts),
  [`docs/UPGRADING_DOWNSTREAM_AGENTS.md`](./docs/UPGRADING_DOWNSTREAM_AGENTS.md),
  [`skills/migrations/`](./skills/migrations/), `gbrain apply-migrations --yes` (manual schema-only).
- **Eval retrieval changes:** capture is off by default. To benchmark a
  retrieval change against real captured queries, set
  `GBRAIN_CONTRIBUTOR_MODE=1`, then `gbrain eval export --since 7d > base.ndjson`
  and `gbrain eval replay --against base.ndjson`. For public benchmark
  coverage (LongMemEval, ground-truth scoring), `gbrain eval longmemeval
  <dataset.jsonl>` runs against an isolated in-memory PGLite
  per question — your `~/.gbrain` is never opened. Full guide:
  [`docs/eval-bench.md`](./docs/eval-bench.md).
- **Drive the brain to a target health score:** the one-command
  loop. `gbrain doctor --remediation-plan --json` previews what would be
  fixed; `gbrain doctor --remediate --yes --target-score 90 --max-usd 5`
  walks a dependency-ordered plan (sync before extract, embed after
  consolidate), re-checking score between every step, refusing to spend
  past the cost cap. Empty brains (no entity pages) or unconfigured embedding
  keys hit a `max_reachable_score` ceiling and bail with what's missing.
  Three phase handlers (synthesize / patterns / consolidate) are
  PROTECTED — only trusted local callers can submit them; MCP cannot.
  Reference: [`docs/architecture/topologies.md`](./docs/architecture/topologies.md).
- **Track a founder/company over time:** when an entity has
  typed metric claims in its `## Facts` fence (`metric: mrr`, `value: 50000`,
  `unit: USD`, `period: monthly` columns), run
  `gbrain eval trajectory <entity-slug>` for the chronological history
  with regressions auto-flagged, or `gbrain founder scorecard <entity-slug>`
  for a four-signal JSON rollup (claim_accuracy / consistency /
  growth_trajectory / red_flags). MCP op `find_trajectory` exposes the
  same data — read scope, visibility-filtered for remote callers.
  `gbrain think` uses this substrate automatically on temporal /
  knowledge_update intent (default ON; flip `think.trajectory_enabled=false`
  to opt out). Non-metric event rows (`meeting`, `job_change`,
  `location_change`) ride through the same pipeline via `facts.event_type`;
  pass `kind: 'event'` or `'all'` to `find_trajectory` to query them.
- **Answer "who is waiting on me?":** connect the user's Google account once
  (`gbrain google setup` — two user interactions; relay the `[SHOW USER]`
  blocks verbatim), then `gbrain waiting --json` returns the ranked people
  waiting on the user, what they promised, evidence quotes, and Gmail deep
  links. Manage loops with `gbrain loops done|drop|mute`. It refuses on
  stale data and names the exact sync command to run first. Guides:
  [`docs/guides/google-connect.md`](./docs/guides/google-connect.md) (setup +
  every error and its fix),
  [`docs/guides/open-loops.md`](./docs/guides/open-loops.md) (how detection
  works); the harness protocol lives in
  [`skills/google-loops/SKILL.md`](./skills/google-loops/SKILL.md).
- **Everything else:** [`./llms.txt`](./llms.txt) is the full documentation map.
  [`./llms-full.txt`](./llms-full.txt) is the same map with core docs inlined for
  single-fetch ingestion.

## Before shipping

Easiest path: `bun run ci:local` runs the full CI gate inside Docker (gitleaks,
guards + typecheck, then 4-shard parallel unit + E2E against four pgvector
containers plus a transaction-mode PgBouncer; unit phase keeps `DATABASE_URL`
unset) and tears down. Use `bun run ci:local:diff` for the
diff-aware subset during fast iteration on a focused branch. Requires Docker
(Docker Desktop / OrbStack / Colima) and `gitleaks` (`brew install gitleaks`).

Manual path: `bun test` plus the E2E lifecycle described in `./CLAUDE.md` (spin
up the test Postgres container, run `bun run test:e2e`, tear it down).

Ship via the `/ship` skill, not by hand. The full release + contributor process
(CHANGELOG voice, version-locations sync, PR conventions, community-PR-wave) lives in
[`./docs/RELEASING.md`](./docs/RELEASING.md); read it before shipping.

## Privacy

Never commit real names of people, companies, or funds into public artifacts. See the
Privacy rule in `./CLAUDE.md`. GBrain pages reference real contacts; public docs must
use generic placeholders (`alice-example`, `acme-example`, `fund-a`).

## Forks

If you are a fork, regenerate `llms.txt` + `llms-full.txt` with your own URL base before
publishing: `LLMS_REPO_BASE=https://raw.githubusercontent.com/your-org/your-fork/main bun run build:llms`.

## COP-SLT 项目工作规则(优先级高)

这是 lhe 的 COP-SLT(T268 CoP SLT release package)项目跟踪约定。

- **收尾绑定到本会话所属项目**:一次会话只对本会话实际触碰的项目(如 cop-slt / Pulsar / 其他)
  做收尾;不跨项目把无关库也一起“顺便收尾”,也不把 A 项目的状态/问题混入 B 项目的收尾。
  同一会话里若确需处理多个项目,各自的收尾分开做、分开 commit。这是通用原则,避免库越来越乱。
- **会话/会议收尾时,必须**把该次新提出的问题追加到
  `~/notes/workspace/cop-slt/analysis/luke-he-question-log` 并按分类更新计数
  (分类: 工具/领域/策略)。这是强制执行,不能只声明而不做。
- **Pulsar 任务收尾时,必须**按 `~/notes/workspace/pulsar/wrap-up/SKILL.md` 的强制
  清单同步项目内所有相关文档状态(新 evidence→sources、综合→analysis、看板行→
  tracking/ongoing-followup、landing page→home.md 快速入口/主线/时间线/目录导航、
  worklog、回链、验证、commit & push),不能只把新内容写进单个 source/analysis 就提交。
  这是强制执行,不能只声明而不做。
- 涉及 cop-slt 知识库写入/修改后,**及时 commit & push** 到 GitLab(`~/notes`)。
- cop-slt 页面只在 Linux 侧写入;Windows/Obsidian 只读。
- 冲突时保留 Linux(brain 权威)版本。

### Windows/Obsidian 同步模型(只读查看 + 仅手动 git pull)
Windows A repo = `C:\Users\lhe\Projects\gitlab\my-docs`,与 `~/notes` 同一 GitLab(remote
`ssh://git@gitlab-master.nvidia.com:12051/lhe/my-docs.git`)。**单写者**:Linux 唯一写源,
Windows 只读(Obsidian 查看/编辑,冲突保留 Linux 版)。

- **2026-09-14 用户新约定：关闭所有自动提交/同步，仅手动 `git pull`**，替代此前保留 auto-pull 的约定。
  Obsidian Git：`autoSaveInterval=0`、`autoPushInterval=0`、`autoPullInterval=0`、
  `autoPullOnBoot=false`、`autoBackupAfterFileChange=false`、`disablePush=true`；
  `syncMethod=rebase` 仅保留为插件非 merge 设置，不启用自动同步。
  Windows 仓库设置 `pull.ff=only`、`pull.rebase=false`、`branch.main.rebase=false`：
  手动 `git pull` 仅允许快进，分叉时停止，不自动 merge/rebase，不恢复 auto-pull。
- **`.obsidian/workspace.json` 不入库**(已 untrack 进 `.gitignore`):这是 Obsidian 每次
  切换界面就重写的高频脏文件,是历史大部分 `M`/conflict 的元凶。不要重新 add 它。
- 遇 Windows 分叉/冲突(如 `[ahead N, behind N]`、`UU`、`MERGING`):**abort/`reset --hard
  origin/main` 对齐 origin**,保留 Linux 版;勿在 Windows 侧制造本地 merge commit。
- 访问 Windows A 用 ob-acp-gate 的 unix socket HTTP(`%XDG_RUNTIME_DIR%/ob-acp/*.sock`);
  授权 8h 过期时 `ob-acp-gate request` 让用户在 launcher 读新 passcode 再 `passcode <CODE>`。
- 修改 Windows 上 obsidian-git 的 `data.json` 走 `ConvertTo-Json`(勿手改 JSON),保持上列非默认项。
