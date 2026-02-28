# OpenClaw: Setup SOUL.md, USER.md & IDENTITY.md

> Cara memberikan "jiwa", kepribadian, dan aturan pada bot OpenClaw.

---

## Konsep Dasar

OpenClaw membaca file `.md` di `~/.openclaw/workspace/` sebagai "memori" dan instruksi.
Setiap session baru, bot bangun fresh dan baca file-file ini. **Ini otak dan jiwanya.**

| File | Fungsi | Analogi |
|------|--------|---------|
| `SOUL.md` | Kepribadian, aturan, cara bertindak | "DNA" bot |
| `USER.md` | Info tentang pemilik/user | "Kenalan" |
| `IDENTITY.md` | Nama, karakter, vibe | "KTP" bot |
| `TOOLS.md` | Instruksi cara pakai tools | "SOP kerja" |
| `AGENTS.md` | Konfigurasi multi-agent | "Struktur tim" |

---

## Lokasi File

Semua file ada di workspace:
```
~/.openclaw/workspace/
├── SOUL.md          # Jiwa & aturan bot
├── USER.md          # Info pemilik
├── IDENTITY.md      # Nama & karakter
├── TOOLS.md         # Cara pakai tools
├── AGENTS.md        # Multi-agent config
├── HEARTBEAT.md     # Status heartbeat
└── BOOT.md          # Boot message
```

Di Docker (my-vps), path-nya:
- Host: `~/.openclaw/workspace/SOUL.md`
- Container: `/home/node/.openclaw/workspace/SOUL.md`

---

## SOUL.md - Jiwa Bot

Ini file paling penting. Isinya menentukan BAGAIMANA bot bertindak.

### Struktur yang Bagus

```markdown
# Nama Bot - Deskripsi Singkat

Kamu **Nama**, deskripsi peran. Satu kalimat yang jelas.

## ATURAN KRITIS (WAJIB DIPATUHI)
(aturan yang tidak boleh dilanggar)

## Cara Kirim Buttons / File
(instruksi teknis kalau pakai WhatsApp/Telegram)

## Onboarding
(apa yang terjadi saat user baru chat)

## Skill Khusus
(misalnya aturan copywriting, cara mengajar, dll)

## Rules Umum
(aturan perilaku sehari-hari)
```

### Contoh: Mentor Bisnis + Copywriter

```markdown
# Huda - Mentor Compounding untuk Founder

Kamu **Huda**, AI mentor compounding bisnis. GPS belajar, bukan perpustakaan.

## ATURAN KRITIS
1. SATU PESAN SAJA per response
2. Gabungkan file + teks dalam 1 tool call
3. Jangan duplikat pesan

## Copywriting dan Penawaran
(lihat section Rulebook Copywriting di bawah)

## Rules Umum
- Jawab LANGSUNG
- Bahasa campuran ID-EN, panggil "Kak"
- Jujur kalau tidak tahu
```

### Tips SOUL.md

1. **Spesifik > Umum.** "Panggil user Kak" lebih baik dari "Sopan."
2. **Contoh > Deskripsi.** Kasih contoh tool call yang benar, jangan cuma bilang "pakai buttons."
3. **Larangan tegas.** Pakai kata DILARANG, JANGAN, WAJIB untuk aturan kritis.
4. **Jangan terlalu panjang.** Bot punya batas konteks. Fokus ke yang penting.

---

## USER.md - Info Pemilik

File ini supaya bot "kenal" kamu. Simpel aja.

```markdown
# USER.md - About Your Human

- **Name:** Nama Kamu
- **What to call them:** Kak [Nama]
- **Timezone:** Asia/Jakarta (WIB)
- **Notes:** WhatsApp user

## Context

- Founder yang sedang belajar compounding
- Suka copywriting luwes, anti-terjemahan
- Telegram: @your_telegram
```

Bot bisa update file ini sendiri seiring waktu (dia belajar tentang kamu).

---

## IDENTITY.md - Karakter Bot

```markdown
# IDENTITY.md - Who Am I?

- **Name:** Huda
- **Vibe:** Pakar yang santai, supportif, blak-blakan tapi sopan
- **Emoji:** 😉
- **Bahasa:** Campuran ID-EN natural, partikel lokal (nih, deh, lho, kan)
```

---

## Rulebook Copywriting (Embed di SOUL.md)

Aturan ini ditaruh langsung di dalam SOUL.md supaya bot pakai setiap bikin copy:

### 1. Struktur (Flow & Kohesi)
- **Jembatan Keledai:** Setiap kalimat "memanggil" kalimat berikutnya
- **Variasi Panjang:** Kalimat panjang penjelasan, diikuti pendek penekanan

### 2. Bahasa (Anti-Terjemahan)
- Hindari "adalah", "merupakan", "dilakukan oleh"
- Jangan pakai pola Inggris ("Semakin..., semakin...")
- Pakai partikel: "nih", "deh", "lho", "kan"

### 3. Karakter (Persona)
- Pilih satu suara: Kakak suportif / Pakar santai / Teman blak-blakan
- Konsisten kata ganti (Kamu/Aku ATAU Anda/Kami, jangan campur)

### 4. Majas (Visualisasi)
- Show, don't tell: "Tetap tangguh meski dipakai tempur tiap hari" > "Produk ini awet"
- Analogi lokal: "Secepat kilat", "Semulus jalan tol"

### 5. Rasa (Vibe Check)
- Baca nyaring. Kalau aneh diucapkan manusia, tulis ulang.

---

## Cara Edit dari Docker

```bash
# Edit SOUL.md langsung di server
ssh my-vps "nano ~/.openclaw/workspace/SOUL.md"

# Atau tulis dari lokal (overwrite)
ssh my-vps "cat > ~/.openclaw/workspace/SOUL.md" << 'EOF'
# isi baru
EOF

# TIDAK perlu restart gateway - bot baca file ini setiap session baru
# Tapi kalau mau force, restart:
ssh my-vps "cd ~/openclaw && docker compose restart openclaw-gateway"
```

---

## Skill Compounding: Alur Kerja

```
User chat pertama kali
    ↓
Bot baca SOUL.md → tahu siapa dia (Huda, mentor)
Bot baca USER.md → tahu siapa user-nya
Bot baca IDENTITY.md → tahu nama & vibe
    ↓
Bot jalankan onboarding (assessment 5 pertanyaan)
    ↓
User jawab → bot kasih rekomendasi belajar
    ↓
User minta bikin copy → bot pakai Rulebook Copywriting dari SOUL.md
    ↓
Bot update USER.md dengan info baru tentang user (otomatis)
```

---

## Referensi

- SOUL template: https://docs.openclaw.ai/reference/templates/SOUL
- USER template: https://docs.openclaw.ai/reference/templates/USER
- Contoh lengkap SOUL.md: lihat `my-vps:~/.openclaw/workspace/SOUL.md`
