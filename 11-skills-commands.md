# OpenClaw: Skills & Slash Commands

> Slash commands untuk kontrol cepat, skills untuk extend kemampuan agent, dan ClawHub sebagai marketplace.

---

## Konsep Dasar

| Istilah | Penjelasan |
|---------|-----------|
| **Slash Command** | Command chat yang dimulai dengan `/` — langsung dieksekusi |
| **Skill** | Paket instruksi (`SKILL.md`) yang mengajari agent cara melakukan sesuatu |
| **ClawHub** | Marketplace skills (seperti npm tapi untuk OpenClaw skills) |
| **Native Command** | Slash command bawaan OpenClaw |

---

## Slash Commands — Kontrol Cepat via Chat

Kirim sebagai **standalone message** (bukan reply ke pesan lain). Harus pesan sendiri.

### Commands yang Sudah Kita Pakai

| Command | Fungsi | Siapa Bisa |
|---------|--------|-----------|
| `/activation mention` | Set mode mention (harus @mention) | Owner |
| `/activation always` | Set mode always (jawab semua) | Owner |
| `/model grok-4.1` | Ganti model di session ini | Owner |
| `/new` | Mulai session baru | Semua |
| `/new grok-4.1` | Session baru + ganti model | Owner |
| `/reset` | Reset session (= `/new`) | Semua |
| `/status` | Info session (context, model, tokens) | Semua |
| `/stop` | Abort run + clear queue | Semua |
| `/compact` | Force compaction | Semua |
| `/send on\|off\|inherit` | Override send policy per session | Owner |

### Commands Lainnya

| Command | Fungsi |
|---------|--------|
| `/context list` | Lihat isi system prompt |
| `/context detail` | Detail context contributors |
| `/verbose on\|off` | Toggle verbose mode |
| `/thinking low\|medium\|high` | Set thinking level |
| `/help` | Lihat available commands |

### Config Native Commands

```json
{
  "commands": {
    "native": "auto",
    "nativeSkills": "auto"
  }
}
```

| Value | Perilaku |
|-------|----------|
| `auto` (default) | Commands aktif otomatis |
| `on` | Explicitly enabled |
| `off` | Disabled |

**GOTCHA:** Owner = nomor yang ada di `channels.whatsapp.allowFrom`. Kalau `allowFrom` tidak diset, nomor bot sendiri jadi owner.

---

## Skills — Extend Kemampuan Agent

Skills bukan tools. Skills adalah **instruksi** yang mengajari agent cara melakukan sesuatu. Ditulis dalam `SKILL.md` dan di-inject ke system prompt.

### Perbedaan Tool vs Skill

| | Tool | Skill |
|---|------|-------|
| **Apa** | Fungsi yang bisa dipanggil | Instruksi/panduan |
| **Format** | TypeScript code | Markdown file (`SKILL.md`) |
| **Lokasi** | Plugin di `extensions/` | `workspace/skills/<name>/` |
| **Inject** | Tool schema ke model API | Text ke system prompt |
| **Contoh** | `generate_image` (function) | "Cara bikin gambar yang bagus" (instructions) |

### Struktur Skill

```
~/.openclaw/workspace/skills/
├── netlify-deploy/
│   └── SKILL.md        # Instruksi cara deploy ke Netlify
├── copywriting/
│   └── SKILL.md        # Aturan copywriting
└── daily-report/
    └── SKILL.md        # Template daily report
```

### SKILL.md Format

SKILL.md pakai **YAML frontmatter** untuk metadata:

```markdown
---
name: netlify-deploy
description: Deploy static site ke Netlify
user-invocable: true
---

# Netlify Deploy

Skill untuk deploy static site ke Netlify.

## Cara Pakai

1. User minta deploy website/HTML
2. Pastikan file HTML sudah jadi
3. Jalankan: `npx netlify deploy --dir=./output --prod`
4. Return URL yang berhasil di-deploy

## Rules

- Selalu confirm dulu sebelum deploy
- Jangan deploy kalau ada error di HTML
- Include preview link sebelum production deploy
```

### Frontmatter Options

| Key | Default | Penjelasan |
|-----|---------|-----------|
| `name` | (required) | Nama skill |
| `description` | — | Deskripsi singkat |
| `user-invocable` | `true` | Expose sebagai slash command untuk user |
| `disable-model-invocation` | `false` | Exclude dari model prompt (agent tidak bisa invoke) |
| `command-dispatch` | — | `"tool"` = bypass model, dispatch langsung ke tool |
| `command-tool` | — | Tool name untuk tool dispatch |

### Skill Discovery — Urutan Precedence

Skills di-discover dari 3 lokasi (highest → lowest):

1. **Workspace skills** — `<workspace>/skills/` (per-agent, tertinggi)
2. **Managed/local skills** — `~/.openclaw/skills/` (shared, install via ClawHub)
3. **Bundled skills** — bawaan OpenClaw (terendah)

Workspace skills override managed, managed override bundled. Extra folders via `skills.load.extraDirs` (lowest precedence).

**PENTING:** Skills di-snapshot saat session start. Perubahan skill mid-session TIDAK langsung berlaku — butuh session baru (`/new`).

### Install Skill dari ClawHub

```bash
# Install skill
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js skills install <skill-name>"
```

### Install Skill Manual

```bash
# Buat folder skill
ssh my-vps "mkdir -p ~/.openclaw/workspace/skills/my-skill"

# Tulis SKILL.md
ssh my-vps "cat > ~/.openclaw/workspace/skills/my-skill/SKILL.md << 'EOF'
# My Custom Skill

Instruksi di sini...
EOF"
```

Tidak perlu restart — skill di-load setiap session baru.

### Config Skills

```json
{
  "commands": {
    "nativeSkills": "auto",
    "nodeManager": "bun"
  }
}
```

| Key | Penjelasan |
|-----|-----------|
| `nativeSkills` | `auto`/`on`/`off` — enable native skills |
| `nodeManager` | `bun`/`npm`/`pnpm` — package manager untuk install dependencies |

---

## ClawHub — Marketplace Skills

ClawHub (https://clawhub.com) adalah tempat untuk discover dan install skills yang dibuat komunitas.

### Browse Skills

```bash
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js skills search <keyword>"
```

### Install dari ClawHub

```bash
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js skills install <skill-id>"
```

### Publish Skill ke ClawHub

```bash
# Di dalam folder skill
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js skills publish ./workspace/skills/my-skill"
```

---

## Skills yang Sudah Terinstall (my-vps)

```
~/.openclaw/workspace/
├── SKILL.md             # Main skill file
├── skills/
│   └── netlify-deploy/  # Deploy ke Netlify
│       └── SKILL.md
```

Huda agent punya akses ke semua skills di workspace-nya. Agent lain (Sales, PM, Turath) punya workspace terpisah dengan skills sendiri.

---

## Native Skills vs Custom Skills

### Native Skills

Bawaan OpenClaw — langsung tersedia tanpa install:
- Command handling (`/activation`, `/model`, dll)
- Session management (`/new`, `/reset`, `/status`)
- Context inspection (`/context list`, `/context detail`)
- Compaction (`/compact`)

### Custom Skills

Yang kamu buat sendiri atau install dari ClawHub:
- Ditulis di `SKILL.md`
- Di-load dari `workspace/skills/<name>/`
- Bisa include file pendukung (templates, data, dll)

---

## Tips: Kapan Pakai Skill vs Tool vs SOUL.md

| Situasi | Pakai |
|---------|-------|
| "Agent harus bisa call API X" | **Tool** (plugin) |
| "Agent harus ikuti SOP tertentu" | **Skill** (SKILL.md) |
| "Agent harus punya personality Y" | **SOUL.md** |
| "Agent harus tahu info tentang user" | **USER.md** |
| "Agent harus punya knowledge base" | **Skill + Tool** (SKILL.md instruksi + tool untuk query) |

---

## Troubleshooting

### Slash command tidak direspon

1. **Harus standalone message** — bukan reply ke pesan lain
2. **Cek owner** — beberapa command hanya owner (`/activation`, `/model`)
3. **Cek `commands.native`** — pastikan bukan `"off"`
4. **Slash command tertulis benar** — case-sensitive, no space sebelum `/`

### Skill tidak ter-load

1. Cek folder ada: `ls ~/.openclaw/workspace/skills/my-skill/`
2. Cek file `SKILL.md` ada (bukan `skill.md` — case matters di Linux)
3. Cek ukuran — skill terlalu besar bisa ke-truncate karena `bootstrapMaxChars`

### "/activation always" tidak bekerja

1. Harus dikirim oleh **owner** (nomor di `allowFrom`)
2. Harus **standalone message** (pesan sendiri, bukan reply)
3. Efeknya per-grup — harus dikirim di grup yang mau diubah
4. Cek log: `docker compose logs --tail=10 openclaw-gateway | grep "activation"`

---

## Referensi

- Slash Commands: https://docs.openclaw.ai/tools/slash-commands
- Skills: https://docs.openclaw.ai/tools/skills
- Skills Config: https://docs.openclaw.ai/tools/skills-config
- ClawHub: https://docs.openclaw.ai/tools/clawhub
- Plugins: https://docs.openclaw.ai/tools/plugin
