# Setup Lengkap OpenClaw — Contoh Konfigurasi

> Dokumentasi lengkap sistem OpenClaw yang sedang berjalan.
> Terakhir update: 17 Februari 2026

---

## Overview

OpenClaw adalah platform AI agent self-hosted yang konek ke WhatsApp. Satu nomor WA bisa punya banyak agent, masing-masing punya personality, tools, dan workspace sendiri.

**Infra:**
- VPS: my-vps (Ubuntu)
- Runtime: Docker Compose
- LLM: Kimi K2.5 via OpenRouter
- Channel: WhatsApp (1 nomor, beberapa grup aktif, beberapa kontak DM)

---

## Agent yang Aktif (4)

### 1. Huda (default)
- **Fungsi:** Mentor bisnis, asisten riset Islam, PM, checkout assistant, dan segala-galanya
- **40 tools** dari 9 plugin
- **Workspace:** Punya SOUL.md (personality + aturan), TOOLS.md, BOOTSTRAP.md, knowledge graph SQLite
- **Bisa:** Jual course, riset kitab klasik, review CV, generate gambar, scrape social media, baca repo production, bikin GitLab issue, deploy website ke Netlify, spawn 8 sub-agent paralel

### 2. Sales Agent (Contoh: Warung/Bisnis)
- **Fungsi:** Sales agent khusus grup bisnis tertentu
- **Workspace:** Terpisah (`workspace-sales/`)
- **Tools:** Default (tidak di-restrict khusus)

### 3. PM Agent
- **Fungsi:** Project management — tracking issues dan progress tim
- **12 tools:** Code reader (3) + GitLab (5) + Fizzy board (4)
- **Binding:** Auto-handle semua pesan di grup `<GROUP_JID_FEEDBACK>`

### 4. Turath Research
- **Fungsi:** Riset kitab klasik Islam
- **7 tools:** Turath only (search, get book, get page, get author, filter, list categories, list authors)

---

## Plugin yang Aktif (9)

### founderplus (14 tools)
Commerce — jual course/event FounderPlus Academy langsung dari WhatsApp. Bisa search produk, validasi voucher, proses payment, cek status bayar.

Tools: `fp_list_courses`, `fp_get_course`, `fp_search_courses`, `fp_list_events`, `fp_get_event`, `fp_list_learning_programs`, `fp_get_learning_program`, `fp_search_products`, `fp_list_newsletter_files`, `fp_list_payment_channels`, `fp_validate_voucher`, `fp_check_payment_status`, `fp_submit_guest_payment`, `fp_format_phone`

### turath (7 tools)
Perpustakaan digital Islam — akses ke 100,000+ kitab klasik dari Turath.io. Bisa cari kitab, baca halaman, lihat biografi ulama, filter per kategori.

Tools: `turath_search`, `turath_get_book`, `turath_get_page`, `turath_get_author`, `turath_filter_ids`, `turath_list_categories`, `turath_list_authors`

### gitlab-issues (5 tools)
Buat dan kelola issues di GitLab. List projects, buat issue, tambah comment.

Tools: `gitlab_list_projects`, `gitlab_list_issues`, `gitlab_get_issue`, `gitlab_create_issue`, `gitlab_add_comment`

### apify-social (4 tools)
Scrape social media real-time via Apify. TikTok, Instagram, Facebook, LinkedIn. Hasil sudah diformat rapi untuk WhatsApp.

Tools: `apify_tiktok`, `apify_instagram`, `apify_facebook`, `apify_linkedin`

### fizzy (4 tools)
Project board — lihat progress tim, kartu aktif/selesai, laporan mingguan.

Tools: `fizzy_list_users`, `fizzy_list_boards`, `fizzy_user_cards`, `fizzy_user_weekly_report`

### codebase-reader (3 tools)
Baca 7 repo production secara read-only. Bisa lihat struktur folder, grep pattern, baca file.

Tools: `code_tree`, `code_grep`, `code_read`

Repo: project-1, project-2, project-3, dll. (sesuai repo yang kamu clone)

### cv-advisor (2 tools)
Review dan scoring CV. Kasih feedback dan scorecard.

Tools: `cv_review`, `cv_scorecard`

### pollinations (1 tool)
Generate gambar AI dari prompt teks via Pollinations.ai.

Tools: `generate_image`

### whatsapp
Channel connector — menghubungkan OpenClaw ke WhatsApp.

---

## Settingan Config (`openclaw.json`)

### Auth & Model
```
Provider:       OpenRouter (API key)
Model utama:    Kimi K2.5 (kimi-coding/k2p5)
Model cadangan: Grok 4.1 Fast (openrouter/x-ai/grok-4.1-fast)
```

### Agent Defaults (berlaku untuk semua agent kecuali di-override)
```
maxConcurrent:      1        → 1 pesan diproses sekaligus per agent
bootstrapMaxChars:  40000    → Max karakter SOUL.md + TOOLS.md yang di-load
subagents.max:      8        → Max 8 sub-agent paralel
subagents.model:    Kimi K2.5
```

### Memory & Compaction
```
memorySearch.sources:          memory, sessions
memorySearch.sessionMemory:    true (eksperimental)
compaction.mode:               safeguard (auto-compress kalau context kepanjangan)
compaction.memoryFlush:        true (simpan ke memory sebelum compress)
```

### WhatsApp Channel
```
dmPolicy:       allowlist     → DM hanya dari nomor yang terdaftar
selfChatMode:   false         → Chat ke diri sendiri tidak diproses
groupPolicy:    open          → Terima pesan dari semua grup
                                (tapi WAJIB daftarkan di groups: {} supaya dibalas)
mediaMaxMb:     50            → Max file attachment 50MB
debounceMs:     5000          → Tunggu 5 detik gabung pesan beruntun sebelum proses
groups:         N grup aktif
allowFrom:      N nomor DM
```

### Messages
```
ackReactionScope:  group-mentions  → Kasih emoji reaction saat di-mention di grup
                                     (tanda "lagi mikir/proses")
```

### Bindings (Routing Agent ke Grup)
```
PM Agent → grup <GROUP_JID_FEEDBACK>  (otomatis handle semua pesan di grup ini)
Sisanya  → Huda (default agent)
```

### Commands & Skills
```
native:        auto   → Command bawaan OpenClaw aktif (/activation, /model, dll)
nativeSkills:  auto   → Skills bawaan juga aktif
nodeManager:   bun    → Pakai Bun untuk install dependencies
```

### Gateway (Server)
```
port:       18789
mode:       local
bind:       lan
auth:       token-based
tailscale:  off
```

---

## Workspace Files (Huda)

File-file di workspace Huda yang membentuk personality dan kemampuannya:

```
/home/node/.openclaw/workspace/
├── SOUL.md          → Personality, aturan, cara jawab, semua instruksi
├── TOOLS.md         → Dokumentasi tools untuk agent
├── BOOTSTRAP.md     → Konteks awal saat boot
├── IDENTITY.md      → Identitas agent
├── HEARTBEAT.md     → Heartbeat config
├── AGENTS.md        → Info multi-agent
├── SKILL.md         → Skill definitions
├── USER.md          → Info tentang user
├── REWRITE-PLAN.md  → Plan untuk rewrite
├── memory/          → Long-term memory agent
├── skills/          → Installed skills (netlify-deploy, dll)
├── knowledge-base/  → Knowledge graph bisnis (SQLite)
└── ...              → Files lain (HTML deployments, cookies, dll)
```

---

## Arsitektur Ringkas

```
┌──────────────┐
│   WhatsApp   │ ← DM + grup
└──────┬───────┘
       │
┌──────▼───────┐     ┌────────────────┐
│   OpenClaw   │────→│  Binding Rules  │
│   Gateway    │     │  (route to agent)│
│  (Docker)    │     └────────────────┘
└──────┬───────┘
       │
  ┌────┼────────────────┐
  │    │                 │
┌─▼──┐ ┌──▼───┐ ┌───▼────┐ ┌──▼─────┐
│Huda│ │Sales │ │PM Agent│ │Turath  │
│    │ │Agent │ │        │ │Research│
└─┬──┘ └──────┘ └────────┘ └────────┘
  │
  ├─ founderplus (14 tools)
  ├─ turath (7 tools)
  ├─ gitlab-issues (5 tools)
  ├─ apify-social (4 tools)
  ├─ fizzy (4 tools)
  ├─ codebase-reader (3 tools)
  ├─ cv-advisor (2 tools)
  ├─ pollinations (1 tool)
  │
  ├─ Knowledge Graph (SQLite)
  ├─ Memory (long-term)
  └─ Sub-agents (max 8 paralel)
       │
       ▼
    Kimi K2.5 via OpenRouter
```

---

## Gotchas & Tips

1. **Grup WAJIB didaftarkan** di `channels.whatsapp.groups` meski `groupPolicy: "open"`. Kalau tidak, pesan masuk tapi bot tidak akan balas.

2. **Activation mode** di grup default `mention` — harus @mention bot. Ubah ke `always` via command `/activation always` di grup (owner-only).

3. **`activation` BUKAN valid key** di config grup — akan crash gateway. Atur via command di grup.

4. **debounceMs: 5000** — Kalau kirim 3 pesan cepat berturut-turut, OpenClaw gabung jadi 1 dan proses sekali.

5. **maxConcurrent: 1** — Agent proses 1 pesan per waktu. Pesan lain masuk antrian.

6. **bootstrapMaxChars: 40000** — SOUL.md yang terlalu panjang akan di-truncate. Keep it concise.

7. **Format WhatsApp** — LLM harus output format native WA (`*bold*`, `_italic_`), BUKAN markdown (`**bold**`, `# heading`).
