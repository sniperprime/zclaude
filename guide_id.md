# zclaude — Panduan Teknis

**Menjalankan Claude Code terhadap GLM API dari z.ai.**

---

## Daftar Isi

1. [Arsitektur & Konsep](#1-arsitektur--konsep)
2. [Persyaratan](#2-persyaratan)
3. [Instalasi](#3-instalasi)
4. [Konfigurasi API Key](#4-konfigurasi-api-key)
5. [Variabel Lingkungan yang Diinjeksi](#5-variabel-lingkungan-yang-diinjeksikan)
6. [Pemetaan Model](#6-pemetaan-model)
7. [Perintah & Subcommand](#7-perintah--subcommand)
8. [Keamanan & Risk Profile](#8-keamanan--risk-profile)
9. [Troubleshooting](#9-troubleshooting)
10. [Uninstall](#10-uninstall)

---

## 1. Arsitektur & Konsep

`zclaude` adalah shell wrapper yang mengkonfigurasi Claude Code CLI agar menargetkan
Anthropic-compatible GLM API milik z.ai, bukan endpoint Anthropic.

### Alur Eksekusi

```
zclaude "task"
  → resolve API key (config file / env / interactive prompt)
  → export env vars (BASE_URL, AUTH_TOKEN, model overrides)
  → exec claude --dangerously-skip-permissions "$@"
```

Wrapper ini menggunakan `exec` untuk menggantikan prosesnya sendiri dengan Claude Code,
sehingga semua argumen diteruskan secara transparan tanpa overhead tambahan.

### Komponen

| Komponen | Peran |
|----------|-------|
| **Claude Code CLI** | LLM harness utama — menangani agentic loop, tool use, dan session management |
| **z.ai GLM API** | Anthropic-compatible inference endpoint — menerima request yang sama dengan format Anthropic API |
| **zclaude** | Konfigurator — menginjeksi env vars agar Claude Code menargetkan z.ai |

---

## 2. Persyaratan

| Prasyarat | Keterangan |
|-----------|-------------|
| Claude Code CLI | Harus terinstall di `PATH`. Panduan: [docs.claude.com/en/docs/claude-code](https://docs.claude.com/en/docs/claude-code) |
| z.ai API Key | Diperlukan untuk autentikasi. Dapatkan melalui [GLM Coding Plan](https://docs.z.ai/devpack/tool/claude) |
| Shell | Bash atau Zsh (macOS/Linux), PowerShell/CMD/Git Bash/WSL (Windows) |

---

## 3. Instalasi

### macOS / Linux

```bash
curl -fsSL https://raw.githubusercontent.com/sniperprime/zclaude/main/install.sh | bash
```

Skrip installer:
- Mengunduh binary `zclaude` ke `~/.local/bin` (default, atau `$ZCLAUDE_BIN_DIR`)
- Menyetel permission executable (`chmod +x`)
- Memvalidasi bahwa target directory ada di `PATH` — jika tidak, mencetak `export`
  line yang perlu ditambahkan ke shell profile

### Windows (PowerShell)

```powershell
irm https://raw.githubusercontent.com/sniperprime/zclaude/main/install.ps1 | iex
```

Skrip installer:
- Mengunduh `zclaude.ps1` ke `%LOCALAPPDATA%\Programs\zclaude`
- Membuat `.cmd` shim untuk kompatibilitas CMD/PowerShell
- Menambahkan directory ke user `PATH` secara persisten

> Pada Windows, Git Bash dan WSL dapat menggunakan installer macOS/Linux.

---

## 4. Konfigurasi API Key

### Urutan Resolusi Key

Zclaude menyelesaikan API key dengan prioritas berikut:

```
1. zclaude config <KEY>     — inline via subcommand
2. config file              — tersimpan dari sesi sebelumnya
3. ZAI_API_KEY env var      — fallback environment variable
4. interactive prompt       — diminta secara interaktif
```

Jika key ditemukan melalui env var `ZAI_API_KEY`, key tersebut otomatis
disimpan ke config file untuk sesi berikutnya.

### Lokasi Penyimpanan

| Platform | Path |
|----------|------|
| macOS / Linux | `$XDG_CONFIG_HOME/zclaude/config` (fallback: `~/.config/zclaude/config`) |
| Windows | `%APPDATA%\zclaude\config` |

File disimpan sebagai plaintext dengan permission `600` (Unix) atau ACL
user-only (Windows).

### Manajemen Key

```bash
zclaude change-key           # prompt interaktif
zclaude change-key <KEY>     # tanpa prompt
zclaude reset                # hapus key tersimpan
```

Alias yang diterima: `config`, `set-key`, `change`.

---

## 5. Variabel Lingkungan yang Diinjeksi

Zclaude mengekspor variabel berikut sebelum menjalankan Claude Code:

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

| Variabel | Fungsi |
|----------|--------|
| `ANTHROPIC_BASE_URL` | Mengalihkan semua API call dari Anthropic ke z.ai endpoint |
| `ANTHROPIC_AUTH_TOKEN` | Kredensial autentikasi |
| `ANTHROPIC_MODEL` / `*_DEFAULT_*_MODEL` | Override model default di setiap tier |
| `CLAUDE_CODE_SUBAGENT_MODEL` | Model yang digunakan untuk spawned subagents |
| `CLAUDE_CODE_EFFORT_LEVEL` | Set ke `max` — memaksimalkan reasoning effort |
| `API_TIMEOUT_MS` | Timeout 50 menit — mengakomodasi long-context agentic calls pada z.ai |

---

## 6. Pemetaan Model

| Model | Tier Claude Code | Use Case |
|-------|-----------------|----------|
| `glm-5.2` | Default / Opus | Sesi utama, tugas reasoning kompleks |
| `glm-5.1` | Sonnet | Sub-tugas mid-complexity — keseimbangan latency dan kualitas |
| `glm-4.7` | Haiku / Subagent | Tugas ringan, pencarian kode, agent pembantu |

Pemetaan ini memastikan Claude Code menggunakan model yang sesuai dengan tier
yang diminta oleh internal routing, tanpa intervensi manual dari user.

---

## 7. Perintah & Subcommand

### Penggunaan Standar

```bash
zclaude                          # masuk ke sesi interaktif
zclaude "refactor this module"   # single-turn prompt
zclaude --help                   # delegasi ke claude --help
```

Semua argumen positional dan flags diteruskan langsung ke Claude Code.

### Subcommand

| Perintah | Alias | Deskripsi |
|----------|-------|-----------|
| `change-key [KEY]` | `config`, `set-key`, `change` | Set atau update API key |
| `reset` | — | Hapus config file |
| `update` | `upgrade` | Pull versi terbaru (re-run installer) |

---

## 8. Keamanan & Risk Profile

### `--dangerously-skip-permissions`

Zclaude selalu menjalankan Claude Code dengan flag ini. Artinya seluruh tool use
(file write, shell execution, dll.) dieksekusi tanpa approval prompt.

**Implikasi:**
- Claude memiliki akses penuh ke filesystem dalam scope working directory
- Shell commands dieksekusi tanpa konfirmasi
- Perubahan file bersifat langsung dan irreversible (tanpa sandboxing)

**Mitigasi yang direkomendasikan:**
- Jalankan dalam directory proyek terisolasi
- Selalu commit ke version control sebelum sesi
- Pertimbangkan untuk tidak menggunakan `zclaude` pada repositori yang berisi
  secrets atau konfigurasi produksi

### Penyimpanan Kredensial

API key disimpan sebagai plaintext. Meskipun file permission dibatasi ke
user-only, kredensial ini dapat diakses oleh proses apapun yang berjalan di
bawah user yang sama. Perlakukan seperti kredensial lokal lainnya.

---

## 9. Troubleshooting

### `zclaude: command not found`

Instalasi belum di `PATH`. Tambahkan directory ke shell profile:

```bash
# Bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc

# Zsh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
```

Windows: buka terminal baru setelah instalasi. Jika masih gagal, tambahkan
`%LOCALAPPDATA%\Programs\zclaude` ke user `Path` secara manual via
System Properties → Environment Variables.

### `claude: command not found`

Claude CLI belum terinstall. Install terlebih dahulu:
[docs.claude.com/en/docs/claude-code](https://docs.claude.com/en/docs/claude-code).

### `Invalid API key` / `Authentication failed`

Key tidak valid atau expired. Verifikasi dan rotasi:

```bash
zclaude change-key
```

### Timeout pada respons panjang

`API_TIMEOUT_MS` sudah diset ke 3.000.000ms (50 menit). Jika masih timeout,
kemungkinan terdapat masalah jaringan atau rate limiting dari z.ai. Periksa
status layanan dan coba kurangi ukuran context window.

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

Uninstall zclaude tidak menghapus Claude Code CLI. Untuk menghapus Claude CLI,
lihat dokumentasi Anthropic.

---

*Repository: [github.com/sniperprime/zclaude](https://github.com/sniperprime/zclaude)*
