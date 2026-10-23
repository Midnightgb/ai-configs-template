# ai-configs-template

Keep your AI coding agent setup the same on every machine you work on: Claude Code
settings, `CLAUDE.md`, skills, agents, hooks, output styles, any other tool config you
choose, plus the tools themselves, pinned by version.

It is a [chezmoi](https://www.chezmoi.io/) source repo with a small sync engine on top
(`ai-sync`). Create your own **private** copy from this template; your config never
touches anyone else's repo.

```
          your private GitHub repo  (the single source of truth)
                 ▲  ai-sync push            │  ai-sync pull (timer, every 30 min)
                 │  (secret scan first)     ▼
   ┌─────────────┴──────────┐      ┌────────────────────────┐
   │ machine A              │      │ machine B              │
   │ ~/.claude, tools …     │      │ ~/.claude, tools …     │
   └────────────────────────┘      └────────────────────────┘
```

## Contents

- [How it works](#how-it-works)
- [Requirements](#requirements)
- [Setup](#setup)
- [Daily use](#daily-use)
- [Pinning tools: `deps.toml`](#pinning-tools-depstoml)
- [Per-machine differences](#per-machine-differences)
- [What is never synced](#what-is-never-synced)
- [Troubleshooting](#troubleshooting)
- [Platform support](#platform-support)
- [Repository layout](#repository-layout)
- [Uninstall](#uninstall)

## How it works

- **The repo is the truth.** Machines do not talk to each other. Each one pulls from and
  pushes to your repo.
- **There is no primary machine.** The first machine you capture defines the initial
  content. After that, every machine is equal: whoever pushes a change first wins, and
  the others receive it on their next pull.
- **Local edits are never overwritten silently.** Every 30 minutes a systemd timer runs
  `ai-sync pull`. If the machine has local changes that are not pushed yet ("drift"), it
  does **not** pull or apply. It notifies you instead, and you run `ai-sync push`.
- **Tools are part of the config.** `.chezmoidata/deps.toml` lists the tools and versions
  you want. Every apply installs or upgrades what is missing. Installed tools are left
  alone.
- **Home paths are portable.** Captured files that mention your home directory become
  templates (`{{ .chezmoi.homeDir }}`), so they work on machines with another username.

Applying is idempotent: a second `chezmoi apply` with no new changes does nothing, and the
install scripts only re-run when their rendered content changes.

## Requirements

| Tool | Why |
|---|---|
| [chezmoi](https://www.chezmoi.io/install/) ≥ 2.40 | applies the repo to your home directory |
| git, with access to your private repo | transport |
| bash, python3 | `ai-sync` and the installer |
| systemd user session (Linux, WSL with systemd enabled) | the 30-minute timer |

Optional: `notify-send` (Linux desktop notifications), herdr
(in-app toasts), `brew` / `cargo` / `npm` if your `deps.toml` uses them.

## Setup

### 1. Create your private repo

On GitHub, click **Use this template → Create a new repository** and make it **Private**.
Or use the CLI:

```sh
gh repo create my-ai-configs --private --template <owner>/ai-configs-template
```

### 2. First machine: capture your current config

Pick the machine whose setup you trust most.

```sh
chezmoi init git@github.com:<you>/my-ai-configs.git
```

`init` asks two questions and stores the answers in `~/.config/chezmoi/chezmoi.toml`:

| Prompt | Example | Used for |
|---|---|---|
| Short name for this machine | `laptop`, `desktop`, `wsl` | per-machine rules in `.chezmoiignore` and templates |
| Claude Code permission mode | `default`, `acceptEdits`, `auto`, `bypassPermissions` | written into `settings.json` on this machine only |

Then install the engine and capture what you have:

```sh
chezmoi apply      # installs ai-sync, the timer and your deps; touches none of ~/.claude yet
ai-sync capture    # copies CLAUDE.md, settings.json, agents, commands, skills, output-styles, hooks
git -C "$(chezmoi source-path)" status    # review what was captured
ai-sync push -m "feat: capture config from laptop"
```

`ai-sync capture` with no arguments takes the default Claude Code set. Pass paths to
capture anything else, for example `ai-sync capture ~/.config/herdr/config.toml`.

What capture does to `settings.json`:

- `permissions.defaultMode` becomes `{{ .defaultMode }}`, so each machine keeps its own mode.
- It adds a `SessionStart` hook that runs `ai-sync hook`. Claude Code then shows a warning
  at session start whenever the machine is out of sync.

### 3. Every other machine: become a mirror

> **This replaces the managed files on this machine with the repo's version.** Back up
> first, and look at the diff before applying.

```sh
tar czf ~/claude-backup-$(date +%F).tgz -C ~ .claude    # backup
chezmoi init git@github.com:<you>/my-ai-configs.git      # answer the two prompts for this machine
chezmoi diff                                             # what would change
chezmoi apply
```

Skills, agents and the like that exist **only** on this machine are not deleted. They show
up as drift, and `ai-sync push` adds them to the repo. That is how you merge two existing
setups: capture on A, mirror on B, then push B's extras.

## Daily use

| Command | What it does |
|---|---|
| `ai-sync status` | Result of the last pull, and local changes not pushed yet |
| `ai-sync push [-m MSG]` | Adds new and changed files, scans for secrets, commits, pushes |
| `ai-sync pull` | Pulls and applies, unless there is drift (the timer runs this) |
| `ai-sync capture [PATH...]` | Puts a file or directory under sync for the first time |
| `ai-sync hook` | Used by the Claude Code SessionStart hook; prints nothing when in sync |

Typical loop: edit a skill on any machine, then run `ai-sync push`. Within 30 minutes every
other machine has it. Run `ai-sync pull` if you do not want to wait.

### What counts as drift

- A managed file that changed locally (`chezmoi status`).
- A new, non-empty file or directory under `~/.claude/{skills,agents,commands,output-styles,hooks}`.

Two `settings.json` cases are **not** drift, because Claude Code rewrites that file on its own:

- Formatting or key-order changes (compared as JSON).
- A changed `permissions.defaultMode`. The next pull restores this machine's mode.

### Notifications

When a pull stops (drift, uncommitted repo changes, failed pull or apply, dependency
warnings), `ai-sync` notifies you through every channel available on the machine:

- `notify-send` on a Linux desktop,
- a herdr toast, when herdr is installed,
- a native Windows toast on WSL (`~/.local/share/ai-sync/toast.ps1`),
- a macOS notification through `osascript`.

It also records the state in `~/.local/state/ai-sync/status`. The Claude Code hook reads that
file at session start, and the log of the last pull is in `~/.local/state/ai-sync/last-pull.log`.

### Secret scan

`ai-sync push` refuses to commit when the staged diff matches common credential shapes:
Anthropic and GitHub tokens, Slack tokens, AWS access keys, private key blocks, and JSON
fields named `api_key`, `token`, `secret` or `password` with a long value. On a hit it
unstages everything and prints the diff line numbers.

This is a safety net, not a guarantee. Keep the repo private.

## Pinning tools: `deps.toml`

Edit `.chezmoidata/deps.toml` in the repo (`chezmoi cd` takes you there), then push. Every
machine installs the change on its next pull.

```toml
[deps]

# Homebrew: version is a minimum (older => upgrade, newer => ok)
[[deps.brew]]
formula = "jq"
version = "1.7"

# Cargo crates: exact version
[deps.cargo]
ripgrep = "14.1.1"

# Global npm packages: exact version
[deps.npm]
"@anthropic-ai/claude-code" = "2.1.0"

# herdr plugins from GitHub, pinned by tag or commit
[[deps.herdr_plugins]]
source = "owner/repo/subdir"
ref = "v1.0.0"
```

Every section is optional. The installer never fails the apply: missing package managers
and failed installs become `deps: WARN …` lines, and `ai-sync` turns those into a
notification.

For a tool that fits none of these, add your own `run_onchange_after_30-<name>.sh.tmpl`
next to the deps script. Check the installed version first, so that re-runs do nothing.

## Per-machine differences

Each machine exposes these values to templates:

| Value | Example | Source |
|---|---|---|
| `.machine` | `laptop` | your answer at `chezmoi init` |
| `.platform` | `linux`, `wsl`, `darwin` | detected |
| `.defaultMode` | `auto` | your answer at `chezmoi init` |
| `.nodeDir` | `/home/me/.nvm/versions/node/v20.11.0/bin` | `node` found on `PATH` at init |

**Keep something off a machine.** Add a rule to `.chezmoiignore`:

```
{{- if eq .machine "laptop" }}
.claude/skills/gpu-benchmarks
{{- end }}
```

**Make a value differ per machine.** Turn the file into a template (`chezmoi chattr +template <file>`)
and branch on the machine:

```
{{ if eq .platform "wsl" }}delivery = "herdr"{{ else }}delivery = "system"{{ end }}
```

**Change your answers** with `chezmoi init --prompt`, then `chezmoi apply`.

### Do not sync generated content

Some tools rewrite their own config files: plugins that inject managed blocks, apps that
re-save settings with their own state. If you sync such a file as is, every machine keeps
fighting the tool, the file shows as drift forever, and the timer stops pulling. Sync only
the parts you write by hand: move them to a template, or let the tool regenerate its block
on each machine and keep it out of the repo.

## What is never synced

`.chezmoiignore` excludes these on purpose:

- **Credentials:** `~/.claude/.credentials.json` and `~/.claude.json`. The second one holds
  your OAuth account and per-machine MCP servers. Log in on each machine.
- **History and session state:** `~/.claude/projects`, `todos`, `shell-snapshots`, `statsig`, `history.jsonl`.
- **Build junk inside skills:** `node_modules`, `__pycache__`, nested `.git`, `*.pyc`, `*Zone.Identifier`.

Never capture SSH keys, `.env` files or token stores, even into a private repo.

## Troubleshooting

**`ai-sync status` shows drift you did not make.** A tool rewrote a managed file. Run
`chezmoi diff <file>` to see the difference. If the change is good, run `ai-sync push`. If it
is not, run `chezmoi apply --force <file>`. If it keeps coming back, see
[Do not sync generated content](#do-not-sync-generated-content).

**The push was blocked by the secret scan.** Remove the secret from the file. Then
un-stage the capture: `chezmoi forget <file>`, or edit the file in `chezmoi cd` and push again.
While the repo has uncommitted changes, pulls stop with `source-dirty`.

**"These come from templates and need a manual edit".** Templated files (such as
`settings.json`) cannot be re-added automatically. Edit them in the repo (`chezmoi edit <file>`)
and push.

**A captured file contains `{{` and stayed a plain file.** `ai-sync` will not template a file
that already uses `{{`, because chezmoi would try to render it. If it also contains your home
path, edit it by hand in the repo and escape the braces as `{{ "{{" }}`.

**`systemctl --user unavailable`.** On WSL, enable systemd in `/etc/wsl.conf`
(`[boot]` → `systemd=true`), run `wsl --shutdown` from Windows, then `chezmoi apply` again.
On a headless server, run `loginctl enable-linger $USER`.

**`git pull failed`.** Two machines pushed diverging commits. Run `chezmoi cd`,
`git pull --rebase`, resolve, `git push`, then `ai-sync pull`.

**Check the timer:** `systemctl --user list-timers ai-sync.timer` and
`journalctl --user -u ai-sync --since today`.

## Platform support

| Platform | Status |
|---|---|
| Linux (systemd) | Supported |
| WSL 2 (systemd enabled) | Supported, with Windows toasts |
| macOS | Config sync and `ai-sync` commands should work but are untested. There is no scheduler yet: run `ai-sync pull` by hand, or add a launchd agent |

## Repository layout

```
.chezmoi.toml.tmpl                      init prompts and per-machine data
.chezmoiignore                          what never reaches $HOME, and per-machine exclusions
.chezmoidata/deps.toml                  pinned tools
run_onchange_after_10-deps.sh.tmpl      installer for deps.toml (runs when it changes)
run_onchange_after_20-ai-sync-timer.sh.tmpl  enables the timer
dot_local/bin/executable_ai-sync        the sync engine
dot_local/share/ai-sync/toast.ps1       Windows toast for WSL
private_dot_config/systemd/user/        ai-sync.service + ai-sync.timer
dot_claude/                             your captured Claude Code config (created by ai-sync capture)
```

## Uninstall

On one machine:

```sh
systemctl --user disable --now ai-sync.timer
rm -f ~/.config/systemd/user/ai-sync.{service,timer} ~/.local/bin/ai-sync
rm -rf ~/.local/state/ai-sync
chezmoi purge    # removes chezmoi's config and source copy; your ~/.claude files stay as they are
```

Then remove the `ai-sync hook` entry from `~/.claude/settings.json`.
