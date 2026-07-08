# zclaude — Technical Guide

**Running Claude Code against z.ai's GLM API.**

---

## Table of Contents

1. [Architecture & Concepts](#1-architecture--concepts)
2. [Prerequisites](#2-prerequisites)
3. [Installation](#3-installation)
4. [API Key Configuration](#4-api-key-configuration)
5. [Injected Environment Variables](#5-injected-environment-variables)
6. [Model Mapping](#6-model-mapping)
7. [Commands & Subcommands](#7-commands--subcommands)
8. [Security & Risk Profile](#8-security--risk-profile)
9. [Troubleshooting](#9-troubleshooting)
10. [Uninstall](#10-uninstall)

---

## 1. Architecture & Concepts

`zclaude` is a shell wrapper that configures the Claude Code CLI to target z.ai's
Anthropic-compatible GLM API instead of the native Anthropic endpoint.

### Execution Flow

```
zclaude "task"
  → resolve API key (config file / env / interactive prompt)
  → export env vars (BASE_URL, AUTH_TOKEN, model overrides)
  → exec claude --dangerously-skip-permissions "$@"
```

The wrapper uses `exec` to replace its own process with Claude Code, so all
arguments are forwarded transparently with no additional overhead.

### Components

| Component | Role |
|-----------|------|
| **Claude Code CLI** | Primary LLM harness — handles the agentic loop, tool use, and session management |
| **z.ai GLM API** | Anthropic-compatible inference endpoint — accepts requests in the same format as the Anthropic API |
| **zclaude** | Configurator — injects env vars so Claude Code targets z.ai |

---

## 2. Prerequisites

| Prerequisite | Details |
|-------------|---------|
| Claude Code CLI | Must be installed on `PATH`. Setup guide: [docs.claude.com/en/docs/claude-code](https://docs.claude.com/en/docs/claude-code) |
| z.ai API Key | Required for authentication. Obtain via the [GLM Coding Plan](https://docs.z.ai/devpack/tool/claude) |
| Shell | Bash or Zsh (macOS/Linux); PowerShell, CMD, Git Bash, or WSL (Windows) |

---

## 3. Installation

### macOS / Linux

```bash
curl -fsSL https://raw.githubusercontent.com/sniperprime/zclaude/main/install.sh | bash
```

The installer script:
- Downloads the `zclaude` binary to `~/.local/bin` (default, or `$ZCLAUDE_BIN_DIR` if set)
- Sets the executable permission (`chmod +x`)
- Validates that the target directory is on `PATH` — if not, prints the `export`
  line to add to your shell profile

### Windows (PowerShell)

```powershell
irm https://raw.githubusercontent.com/sniperprime/zclaude/main/install.ps1 | iex
```

The installer script:
- Downloads `zclaude.ps1` to `%LOCALAPPDATA%\Programs\zclaude`
- Creates a `.cmd` shim for CMD and PowerShell compatibility
- Adds the install directory to the user `PATH` persistently

> On Windows, Git Bash and WSL can use the macOS/Linux installer.

---

## 4. API Key Configuration

### Key Resolution Order

Zclaude resolves the API key with the following priority:

```
1. zclaude config <KEY>     — inline via subcommand
2. config file              — stored from a previous session
3. ZAI_API_KEY env var      — fallback environment variable
4. interactive prompt       — requested interactively
```

If the key is found via the `ZAI_API_KEY` environment variable, it is automatically
saved to the config file for subsequent sessions.

### Storage Locations

| Platform | Path |
|----------|------|
| macOS / Linux | `$XDG_CONFIG_HOME/zclaude/config` (fallback: `~/.config/zclaude/config`) |
| Windows | `%APPDATA%\zclaude\config` |

The file is stored as plaintext with `600` permissions (Unix) or user-only ACL
(Windows).

### Key Management

```bash
zclaude change-key           # interactive prompt
zclaude change-key <KEY>     # no prompt
zclaude reset                # delete stored key
```

Accepted aliases: `config`, `set-key`, `change`.

---

## 5. Injected Environment Variables

Zclaude exports the following variables before launching Claude Code:

```sh
ANTHROPIC_BASE_URL="https://api.z.ai/api/anthropic"
ANTHROPIC_AUTH_TOKEN="<resolved API key>"
ANTHROPIC_MODEL="glm-5.2"
ANTHROPIC_DEFAULT_OPUS_MODEL="glm-5.2"
ANTHROPIC_DEFAULT_SONNET_MODEL="glm-5.1"
ANTHROPIC_DEFAULT_HAIKU_MODEL="glm-4.7"
CLAUDE_CODE_SUBAGENT_MODEL="glm-4.7"
CLAUDE_CODE_EFFORT_LEVEL="max"
API_TIMEOUT_MS="3000000"
```

| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_BASE_URL` | Redirects all API calls from Anthropic to the z.ai endpoint |
| `ANTHROPIC_AUTH_TOKEN` | Authentication credential |
| `ANTHROPIC_MODEL` / `*_DEFAULT_*_MODEL` | Override default models at each tier |
| `CLAUDE_CODE_SUBAGENT_MODEL` | Model used for spawned subagents |
| `CLAUDE_CODE_EFFORT_LEVEL` | Set to `max` — maximizes reasoning effort |
| `API_TIMEOUT_MS` | 50-minute timeout — accommodates long-context agentic calls on z.ai |

---

## 6. Model Mapping

| Model | Claude Code Tier | Use Case |
|-------|-----------------|----------|
| `glm-5.2` | Default / Opus | Primary session, complex reasoning tasks |
| `glm-5.1` | Sonnet | Mid-complexity subtasks — balances latency and quality |
| `glm-4.7` | Haiku / Subagent | Lightweight tasks, code search, helper agents |

This mapping ensures Claude Code uses the appropriate model for the tier
requested by its internal routing, with no manual intervention required.

---

## 7. Commands & Subcommands

### Standard Usage

```bash
zclaude                          # interactive session
zclaude "refactor this module"   # single-turn prompt
zclaude --help                   # delegates to claude --help
```

All positional arguments and flags are forwarded directly to Claude Code.

### Subcommands

| Command | Aliases | Description |
|---------|---------|-------------|
| `change-key [KEY]` | `config`, `set-key`, `change` | Set or update the API key |
| `reset` | — | Delete the config file |
| `update` | `upgrade` | Pull the latest version (re-runs the installer) |

---

## 8. Security & Risk Profile

### `--dangerously-skip-permissions`

Zclaude always launches Claude Code with this flag. All tool use (file writes,
shell execution, etc.) is executed without an approval prompt.

**Implications:**
- Claude has full access to the filesystem within the working directory scope
- Shell commands are executed without confirmation
- File changes are immediate and irreversible (no sandboxing)

**Recommended mitigations:**
- Run within an isolated project directory
- Always commit to version control before a session
- Avoid using `zclaude` on repositories containing secrets or production configs

### Credential Storage

The API key is stored as plaintext. Although file permissions are restricted to
user-only, the credential is accessible to any process running under the same
user. Treat it like any other local credential.

---

## 9. Troubleshooting

### `zclaude: command not found`

The installation directory is not on `PATH`. Add it to your shell profile:

```bash
# Bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc

# Zsh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
```

Windows: open a new terminal after installation. If the issue persists, manually
add `%LOCALAPPDATA%\Programs\zclaude` to the user `Path` via System Properties
→ Environment Variables.

### `claude: command not found`

Claude CLI is not installed. Install it first:
[docs.claude.com/en/docs/claude-code](https://docs.claude.com/en/docs/claude-code).

### `Invalid API key` / `Authentication failed`

The key is invalid or expired. Verify and rotate:

```bash
zclaude change-key
```

### Timeout on long responses

`API_TIMEOUT_MS` is set to 3,000,000ms (50 minutes). If timeouts persist, there may
be a network issue or rate limiting from z.ai. Check the service status and consider
reducing the context window size.

---

## 10. Uninstall

### macOS / Linux

```bash
rm ~/.local/bin/zclaude
rm -rf ~/.config/zclaude
```

### Windows (PowerShell)

```powershell
Remove-Item -Recurse -Force "$env:LOCALAPPDATA\Programs\zclaude"
Remove-Item -Recurse -Force "$env:APPDATA\zclaude"
```

Uninstalling zclaude does not remove the Claude Code CLI. To uninstall Claude CLI,
refer to the Anthropic documentation.

---

*Repository: [github.com/sniperprime/zclaude](https://github.com/sniperprime/zclaude)*
