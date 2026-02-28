# OpenClaw: WhatsApp Group Setup & Troubleshooting

> Cara mengaktifkan bot OpenClaw di grup WhatsApp, termasuk gotcha dan discovery dari debugging.

---

## Konsep Dasar

OpenClaw bisa menerima dan menjawab pesan di grup WhatsApp. Ada 3 config key utama yang mengatur ini:

| Config Key | Fungsi | Tipe |
|------------|--------|------|
| `channels.whatsapp.groupPolicy` | Kebijakan akses grup | `"open"` / `"disabled"` / `"allowlist"` |
| `channels.whatsapp.groups` | Daftar grup yang diizinkan + setting per-grup | Object/Record `{ "jid": {} }` |
| `channels.whatsapp.groupAllowFrom` | Daftar **pengirim** (nomor HP) yang boleh trigger bot di grup | Array `["+62xxx"]` |

**PENTING:** Jangan tertukar antara `groups` dan `groupAllowFrom`!
- `groups` = **grup mana** yang diizinkan (pakai group JID)
- `groupAllowFrom` = **siapa** (nomor HP) yang boleh trigger bot di grup

---

## Config Key Detail

### groupPolicy

```json
{
  "channels": {
    "whatsapp": {
      "groupPolicy": "open"
    }
  }
}
```

| Value | Perilaku |
|-------|----------|
| `"open"` | Semua grup bisa chat bot |
| `"disabled"` | Bot mengabaikan semua pesan grup |
| `"allowlist"` | Hanya grup di `groups` yang diizinkan (default) |

### groups (Record, BUKAN Array!)

```json
{
  "channels": {
    "whatsapp": {
      "groups": {
        "<GROUP_JID_1>@g.us": {},
        "<GROUP_JID_2>@g.us": {}
      }
    }
  }
}
```

**GOTCHA KRITIS:** `groups` harus berupa **object/record**, bukan array!

```json
// SALAH - akan error "expected record, received array"
"groups": ["<GROUP_JID_1>@g.us"]

// BENAR
"groups": { "<GROUP_JID_1>@g.us": {} }
```

Value tiap grup adalah object kosong `{}`. Jangan isi key tambahan seperti `activation` - itu bukan config key yang valid.

### groupAllowFrom (Sender Filter)

```json
{
  "channels": {
    "whatsapp": {
      "groupAllowFrom": ["+628xxxxxxxxx1", "+628xxxxxxxxx2"]
    }
  }
}
```

Ini membatasi **siapa** yang bisa trigger bot di grup. Jika tidak diset, fallback ke `allowFrom` (DM allowlist).

**GOTCHA:** Jangan isi `groupAllowFrom` dengan group JID! Isinya harus nomor HP format E.164.

---

## Activation Mode (Mention vs Always)

> Referensi resmi: https://docs.openclaw.ai/channels/whatsapp#groups

Menentukan kapan bot merespons di grup:

| Mode | Perilaku |
|------|----------|
| `mention` (default) | Bot jawab kalau di-@mention, reply ke chat bot, atau regex match |
| `always` | Bot jawab semua pesan di grup |

### Kapan bot merespons di mode `mention`?

1. **@mention bot** di grup (tag nomor bot)
2. **Reply/quote pesan bot** sebelumnya (implicit mention - otomatis dianggap mention)
3. **Regex match** dari `mentionPatterns` (kalau dikonfigurasi)

Jadi kalau kamu reply ke jawaban bot, bot akan jawab lagi tanpa perlu @mention.

### Cara ganti activation mode

Kirim pesan **standalone** (pesan sendiri, bukan reply) di grup. **Harus owner:**
```
/activation always
```
atau
```
/activation mention
```

**Owner** = nomor di `channels.whatsapp.allowFrom`, atau nomor bot sendiri kalau tidak diset.

### Config di `openclaw.json` (per-grup)

`requireMention` bisa diset per-grup di `groups` record:

```json
{
  "channels": {
    "whatsapp": {
      "groups": {
        "<GROUP_JID_1>@g.us": { "requireMention": false },
        "<GROUP_JID_2>@g.us": {}
      }
    }
  }
}
```

| `requireMention` | Perilaku |
|-------------------|----------|
| `true` atau `{}` (default) | Mode mention - butuh @mention atau reply ke bot |
| `false` | Mode always - jawab semua pesan |

**GOTCHA:** `activation` BUKAN config key yang valid. Pakai `requireMention` (boolean) di config, atau `/activation` (chat command) di grup.

---

## Cara Dapat Group JID

Group JID format: `<GROUP_JID_1>@g.us`

Cara paling mudah: kirim pesan di grup, lalu cek log:
```bash
ssh my-vps "cd ~/openclaw && docker compose logs --tail=20 openclaw-gateway" | grep "group"
```

Output akan menunjukkan:
```
Inbound message <GROUP_JID_1>@g.us -> +628xxxxxxxxx3 (group, 118 chars)
```

Group JID = `<GROUP_JID_1>@g.us`

---

## Config Minimal (Recommended)

Untuk bot yang menerima pesan di semua grup, respond saat di-mention:

```json
{
  "channels": {
    "whatsapp": {
      "dmPolicy": "pairing",
      "allowFrom": ["+628xxxxxxxxx1", "+628xxxxxxxxx2"],
      "groupPolicy": "open",
      "mediaMaxMb": 50
    }
  }
}
```

Untuk bot yang hanya aktif di grup tertentu:

```json
{
  "channels": {
    "whatsapp": {
      "dmPolicy": "pairing",
      "allowFrom": ["+628xxxxxxxxx1", "+628xxxxxxxxx2"],
      "groupPolicy": "allowlist",
      "groups": {
        "<GROUP_JID_1>@g.us": {},
        "<GROUP_JID_2>@g.us": {}
      },
      "mediaMaxMb": 50
    }
  }
}
```

---

## Acknowledgment Reactions (Opsional)

Bot bisa auto-react emoji saat terima pesan (sebelum jawab):

```json
{
  "channels": {
    "whatsapp": {
      "ackReaction": {
        "emoji": "👀",
        "direct": true,
        "group": "mentions"
      }
    }
  }
}
```

| Key | Nilai |
|-----|-------|
| `emoji` | Emoji yang dipakai ("👀", "✅", dll) |
| `direct` | `true`/`false` - react di DM? |
| `group` | `"always"` / `"mentions"` / `"never"` |

**GOTCHA:** Pakai `channels.whatsapp.ackReaction` (object), BUKAN `messages.ackReactionScope` (string). Yang terakhir tidak valid.

---

## History Injection di Grup

Bot otomatis inject pesan-pesan terakhir di grup sebagai konteks:

```
[Chat messages since your last reply - for context]
[from: User1 (+628xxxxxxxxx2)] pesan 1...
[from: User2 (+628xxxxxxxxx1)] pesan 2...
[Current message - respond to this]
[from: User1 (+628xxxxxxxxx2)] pertanyaan baru...
```

Default: 50 pesan terakhir. Ubah dengan:
```json
{
  "channels": {
    "whatsapp": {
      "historyLimit": 50
    }
  }
}
```

Set `0` untuk disable.

---

## Troubleshooting

### Bot tidak terima pesan grup (paling sering!)

**Masalah:** Log hanya menunjukkan DM, tidak ada "(group, X chars)".

**Checklist:**
1. **groupPolicy:** Pastikan bukan `"disabled"`. Pakai `"open"` untuk test.
2. **groups format:** Harus **object**, bukan array!
   ```json
   // BENAR
   "groups": { "120363...@g.us": {} }
   // SALAH
   "groups": ["120363...@g.us"]
   ```
3. **groupAllowFrom:** Jangan isi dengan group JID! Isi dengan nomor HP.
   ```json
   // BENAR (nomor pengirim)
   "groupAllowFrom": ["+628xxxxxxxxx2"]
   // SALAH (group JID)
   "groupAllowFrom": ["120363...@g.us"]
   ```
4. **Config valid?** Cek log saat startup - jika ada "Config invalid", fix dulu.
5. **Jalankan doctor:** `docker compose exec openclaw-gateway node dist/index.js doctor --fix`

### Bot terima pesan tapi tidak jawab di grup

**Kemungkinan:** Activation mode = `mention` (default).

Bot hanya jawab kalau salah satu ini terpenuhi:
1. User **@mention** bot di grup
2. User **reply/quote pesan bot** sebelumnya (implicit mention)
3. Body pesan cocok dengan **mentionPatterns** regex

**Fix kalau mau bot jawab semua pesan:**
- Kirim `/activation always` di grup (harus owner), atau
- Set `"requireMention": false` di `groups` config per-grup

### "Listening for personal WhatsApp inbound messages" - normal?

**YA, ini normal!** Pesan ini SELALU muncul, baik grup aktif atau tidak. "Personal" berarti "akun WhatsApp personal kamu" (bukan Business API). Bukan indikator grup disabled.

### Log "Config invalid: expected record, received array"

Format `groups` salah. Ganti array ke object:
```bash
ssh my-vps "python3 -c \"
import json
with open('/home/ubuntu/.openclaw/openclaw.json') as f:
    cfg = json.load(f)
groups = cfg['channels']['whatsapp'].get('groups', [])
if isinstance(groups, list):
    cfg['channels']['whatsapp']['groups'] = {g: {} for g in groups}
    with open('/home/ubuntu/.openclaw/openclaw.json', 'w') as f:
        json.dump(cfg, f, indent=2)
    print('Fixed!')
\""
```

### Grup tiba-tiba berhenti bekerja setelah restart

Kalau grup pernah jalan lalu stop setelah restart:
1. Cek apakah config berubah (mungkin hot-reload atau manual edit)
2. Jalankan `doctor --fix`: `docker compose exec openclaw-gateway node dist/index.js doctor --fix`
3. Restart gateway: `docker compose restart openclaw-gateway`

---

## Discovery & Gotcha (Catatan dari Debugging)

### 1. `groups` vs `groupAllowFrom` - PALING MEMBINGUNGKAN

Ini sumber bug #1 kami. Kedua key ini namanya mirip tapi fungsinya beda total:

| | `groups` | `groupAllowFrom` |
|---|---------|-------------------|
| **Fungsi** | Grup mana yang boleh | Siapa yang boleh trigger |
| **Isi** | Group JID (`@g.us`) | Nomor HP (`+62xxx`) |
| **Format** | Object `{ "jid": {} }` | Array `["+62xxx"]` |
| **Dipakai saat** | `groupPolicy: "allowlist"` | Selalu (filter sender) |

### 2. Invalid groupPolicy value = "open" behavior

Kami sempat set `groupPolicy: "mentions"` (typo/invalid). Ternyata value invalid tidak di-reject, tapi JUGA tidak match dengan `"disabled"` atau `"allowlist"`, sehingga code path-nya sama dengan `"open"`. **Bot tetap jalan tapi secara tidak sengaja.**

Begitu kami "fix" ke `"allowlist"` dengan format `groupAllowFrom` yang salah (isi group JID bukan nomor HP), bot malah mati karena:
- `groupPolicy: "allowlist"` → cek `groupAllowFrom`
- `groupAllowFrom` diisi JID, bukan E.164 → normalize gagal
- Semua pengirim di-block

### 3. "Listening for personal WhatsApp inbound messages" = RED HERRING

Pesan ini selalu muncul di log. Banyak waktu terbuang mengira ini indikator grup tidak aktif. Padahal "personal" merujuk ke "personal WhatsApp account" (vs Business API).

### 4. Config hot-reload bisa jadi sumber masalah

OpenClaw auto-detect perubahan `openclaw.json` dan apply secara dinamis. Kalau config baru invalid, gateway tetap jalan tapi dengan config lama/fallback. Ini bisa bikin bingung karena:
- Edit config → gateway bilang "config change applied"
- Tapi sebenarnya pakai best-effort mode karena ada validation error
- Restart gateway → error muncul baru ketahuan

### 5. `activation` bukan config key, tapi `requireMention` iya

Kami coba `"groups": { "jid": { "activation": "always" } }` → error "Unrecognized key".

Yang benar:
- **Config file:** pakai `"requireMention": false` di dalam group record
- **Chat command:** pakai `/activation always` di grup (owner-only, standalone message)
- Reply/quote ke pesan bot = **implicit mention** (otomatis dijawab tanpa @mention)

### 6. `ackReaction` vs `ackReactionScope`

Docs menunjukkan `channels.whatsapp.ackReaction` (object dengan `emoji`, `direct`, `group`). Jangan pakai `messages.ackReactionScope` (string) - itu format lama/invalid.

### 7. Urutan debugging yang efektif

```
1. Cek log startup → ada "Config invalid"? Fix dulu
2. Jalankan `openclaw doctor` → lihat status WhatsApp & sessions
3. Set `groupPolicy: "open"` dulu untuk test (hilangkan variabel allowlist)
4. Kirim pesan di grup → cek log ada "Inbound message ... (group)"?
5. Kalau ada inbound tapi gak dijawab → masalah activation (mention vs always)
6. Kalau gak ada inbound → masalah config atau koneksi WhatsApp
```

---

## Menambahkan Grup Baru (Step-by-Step)

### Langkah 1: Dapat Group JID

Kirim pesan apapun di grup (bot harus sudah jadi member), lalu cek log:

```bash
ssh my-vps "cd ~/openclaw && docker compose logs --tail=20 openclaw-gateway" | grep "Inbound.*group"
```

Output:
```
Inbound message <GROUP_JID_6>@g.us -> +628xxxxxxxxx3 (group, 154 chars)
```

Group JID = `<GROUP_JID_6>@g.us`

### Langkah 2: Tambahkan ke `openclaw.json`

```bash
ssh my-vps "python3 -c \"
import json
with open('/home/ubuntu/.openclaw/openclaw.json') as f:
    cfg = json.load(f)
cfg['channels']['whatsapp']['groups']['<GROUP_JID_6>@g.us'] = {}
with open('/home/ubuntu/.openclaw/openclaw.json', 'w') as f:
    json.dump(cfg, f, indent=2)
print('Done!')
\""
```

### Langkah 3: Restart gateway

```bash
ssh my-vps "cd ~/openclaw && docker compose restart openclaw-gateway"
```

### Langkah 4: Test

Kirim pesan di grup (mention bot kalau mode mention). Cek log untuk auto-reply:
```bash
ssh my-vps "cd ~/openclaw && docker compose logs --tail=10 openclaw-gateway" | grep "Auto-replied"
```

---

## Per-Group Agent & Workspace

Setiap grup bisa diarahkan ke **agent yang berbeda** dengan workspace (SOUL.md) yang berbeda. Ini berguna kalau kamu punya beberapa persona/bot dalam satu gateway.

### Konsep

| Istilah | Penjelasan |
|---------|-----------|
| **Agent** | Satu identitas AI dengan SOUL.md, tools, dan model sendiri |
| **Workspace** | Folder yang berisi SOUL.md, TOOLS.md, skills, dll untuk agent itu |
| **Group routing** | Mengarahkan grup tertentu ke agent tertentu |

### Cara Setup Multi-Agent

#### 1. Definisikan agents di `openclaw.json`

```json
{
  "agents": {
    "defaults": {
      "model": { "primary": "kimi-coding/k2p5" }
    },
    "entries": [
      {
        "id": "main",
        "name": "Huda",
        "workspace": "/home/node/.openclaw/workspace",
        "tools": {
          "allow": ["generate_image", "turath_search", "turath_get_book", "..."]
        }
      },
      {
        "id": "turath",
        "name": "Turath Research",
        "workspace": "/home/node/.openclaw/workspace-turath",
        "tools": {
          "allow": ["turath_search", "turath_get_book", "turath_get_page", "..."]
        }
      }
    ]
  }
}
```

Setiap agent punya:
- `id` — identifier unik
- `workspace` — path ke folder yang berisi SOUL.md (persona & instruksi)
- `tools.allow` — daftar tools yang boleh dipakai agent ini

#### 2. Arahkan grup ke agent tertentu

Di dalam `groups` config, tambahkan `agent` key:

```json
{
  "channels": {
    "whatsapp": {
      "groups": {
        "<GROUP_JID_1>@g.us": {},
        "<GROUP_JID_6>@g.us": {
          "agent": "turath"
        }
      }
    }
  }
}
```

- Grup tanpa `agent` key → pakai agent default (`main`)
- Grup dengan `"agent": "turath"` → pakai agent Turath (workspace-turath/SOUL.md)

#### 3. Buat workspace untuk agent baru

```bash
# Di server (dalam Docker)
ssh my-vps "mkdir -p ~/.openclaw/workspace-turath"

# Buat SOUL.md untuk agent baru
ssh my-vps "cat > ~/.openclaw/workspace-turath/SOUL.md << 'EOF'
# Nama Agent

Persona dan instruksi di sini...
EOF"
```

#### 4. Restart & test

```bash
ssh my-vps "cd ~/openclaw && docker compose restart openclaw-gateway"
```

### Contoh: Satu Bot, Dua Persona

```
Gateway (+628xxxxxxxxx)
├── Grup "Bisnis" → Agent "main" (Huda - mentor bisnis)
├── Grup "Riset" → Agent "turath" (Turath Research)
└── DM → Agent "main" (default)
```

Semua pakai nomor WA yang sama, tapi persona berbeda per grup.

### GOTCHA

1. **Path workspace dalam Docker:** Pakai path internal Docker (`/home/node/.openclaw/...`), bukan path host (`/home/ubuntu/.openclaw/...`)
2. **Agent ID harus match:** `"agent": "turath"` di groups harus cocok dengan `"id": "turath"` di agents.entries
3. **Tools per agent:** Setiap agent punya daftar tools sendiri. Agent "turath" tidak bisa pakai `generate_image` kecuali ditambahkan ke `tools.allow`-nya
4. **SOUL.md per workspace:** Setiap workspace punya SOUL.md sendiri — ini yang menentukan persona dan cara bicara agent

---

## Config Final yang Bekerja (my-vps)

```json
{
  "channels": {
    "whatsapp": {
      "dmPolicy": "pairing",
      "selfChatMode": false,
      "allowFrom": ["+628xxxxxxxxx1", "+628xxxxxxxxx2"],
      "groupPolicy": "open",
      "mediaMaxMb": 50,
      "groups": {
        "<GROUP_JID_1>@g.us": {},
        "<GROUP_JID_2>@g.us": {},
        "<GROUP_JID_3>@g.us": {},
        "<GROUP_JID_4>@g.us": {},
        "<GROUP_JID_5>@g.us": {},
        "<GROUP_JID_6>@g.us": {}
      },
      "debounceMs": 0
    }
  }
}
```

---

## SELALU Rujuk Docs Resmi

Dokumentasi ini adalah catatan belajar. **Selalu cross-check dengan docs resmi** karena OpenClaw aktif dikembangkan dan config bisa berubah:

**https://docs.openclaw.ai/channels/whatsapp**

Section penting:
- [Groups](https://docs.openclaw.ai/channels/whatsapp#groups) - groupPolicy, activation, mention
- [Config quick map](https://docs.openclaw.ai/channels/whatsapp#config-quick-map) - semua config key
- [Ack reactions](https://docs.openclaw.ai/channels/whatsapp#acknowledgment-reactions-auto-react-on-receipt) - auto-react emoji
- [Inbound flow](https://docs.openclaw.ai/channels/whatsapp#inbound-flow-dm--group) - DM + group flow
- [Troubleshooting](https://docs.openclaw.ai/channels/whatsapp#troubleshooting-quick) - common issues

Kalau ragu tentang config key atau format, cek docs dulu sebelum trial-and-error.
