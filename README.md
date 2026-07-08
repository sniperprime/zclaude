# zclaude

Run [Claude Code](https://docs.claude.com/en/docs/claude-code) against
[z.ai](https://z.ai)'s Anthropic-compatible GLM API.

Install once with a single command, enter your z.ai API key once (it asks
you interactively if you haven't included it yet), and from then on just run
`zclaude` — it sets all the env vars and launches `claude` for you.

> Requires the `claude` CLI to already be installed
> ([instructions](https://docs.claude.com/en/docs/claude-code)).
> Get a z.ai key by subscribing to the
> [GLM Coding Plan](https://docs.z.ai/devpack/tool/claude).

## Install (one command)

### macOS / Linux

```bash
curl -fsSL https://raw.githubusercontent.com/sniperprime/zclaude/main/install.sh | bash
```

Installs a single `zclaude` script into `~/.local/bin`. If that directory
isn't on your `PATH`, the installer prints the line to add.

> **Getting a `429` from `raw.githubusercontent.com`?** GitHub rate-limits
> that endpoint per source IP — wait a minute and retry, or clone the repo
> instead and run the installer locally:
>
> ```bash
> git clone --depth 1 https://github.com/sniperprime/zclaude.git /tmp/zclaude
> bash /tmp/zclaude/install.sh
> ```

### Windows (PowerShell)

```powershell
irm https://raw.githubusercontent.com/sniperprime/zclaude/main/install.ps1 | iex
```

Installs `zclaude` into `%LOCALAPPDATA%\Programs\zclaude` and adds it to
your user `PATH`. Open a **new** terminal afterward.

> **Getting a `429` from `raw.githubusercontent.com`?** GitHub rate-limits
> that endpoint per source IP — wait a minute and retry, or clone the repo
> instead and run the installer locally:
>
> ```powershell
> git clone --depth 1 https://github.com/sniperprime/zclaude.git $env:TEMP\zclaude
> & "$env:TEMP\zclaude\install.ps1"
> ```

> On Windows you can also use the macOS/Linux command above from **Git Bash**
> or **WSL**.

## Use

First run asks for your z.ai API key (hidden input) and saves it:

```bash
zclaude
```

Every run after that just works — same key, no prompt. Any arguments pass
straight through to `claude`:

```bash
zclaude "refactor this module"
zclaude --help
```

### Ways to provide the key

The key is resolved in this order:

1. `zclaude config <KEY>` — set it inline, no prompt.
2. The stored config file — set on a previous run.
3. `ZAI_API_KEY` environment variable — used and saved for next time.
4. Interactive prompt — asked for automatically if none of the above is set.

## Manage your key

```bash
zclaude change-key        # change the stored key (interactive prompt)
zclaude change-key <KEY>  # change the key without a prompt
zclaude reset             # delete the stored key
```

`config`, `set-key`, and `change` are accepted as aliases for `change-key`.

## Update

```bash
zclaude update    # pull the latest version (re-runs the installer)
```

| Platform        | Where the key is stored                       |
| --------------- | --------------------------------------------- |
| macOS / Linux   | `~/.config/zclaude/config` (perms `600`)      |
| Windows         | `%APPDATA%\zclaude\config` (ACL: you only)    |

It is stored in plaintext on your machine — anyone with access to your user
account can read it. Treat it like any other local credential.

## What it sets

```sh
ANTHROPIC_BASE_URL="https://api.z.ai/api/anthropic"
ANTHROPIC_AUTH_TOKEN="<your z.ai API key>"
ANTHROPIC_MODEL="glm-5.2"
ANTHROPIC_DEFAULT_OPUS_MODEL="glm-5.2"
ANTHROPIC_DEFAULT_SONNET_MODEL="glm-5.1"
ANTHROPIC_DEFAULT_HAIKU_MODEL="glm-4.7"
CLAUDE_CODE_SUBAGENT_MODEL="glm-4.7"
CLAUDE_CODE_EFFORT_LEVEL="max"
API_TIMEOUT_MS="3000000"
```

Then runs: `claude --dangerously-skip-permissions "$@"`

Model tiers: `glm-5.2` for the main session and opus tier, `glm-5.1` for the
sonnet tier, and `glm-4.7` for the haiku tier and spawned subagents.
`API_TIMEOUT_MS` raises Claude Code's request timeout so z.ai's long-context
agentic calls don't get cut off mid-flight.

> **Note:** `--dangerously-skip-permissions` lets Claude run tools without
> per-action approval prompts. Convenient, but it means commands and file
> edits execute without asking. Use it in a directory you trust.

## Uninstall

**macOS / Linux**

```bash
rm ~/.local/bin/zclaude
rm -rf ~/.config/zclaude
```

**Windows (PowerShell)**

```powershell
Remove-Item -Recurse -Force "$env:LOCALAPPDATA\Programs\zclaude"
Remove-Item -Recurse -Force "$env:APPDATA\zclaude"
```
