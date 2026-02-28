# OpenClaw: Browser, Media & Nodes

> Browser tool untuk web automation, media support untuk gambar/audio, dan nodes untuk control perangkat remote.

---

## Konsep Dasar

| Istilah | Penjelasan |
|---------|-----------|
| **Browser** | Chromium yang dikelola OpenClaw — bisa navigate, screenshot, interact |
| **Node** | Perangkat (macOS/iOS/Android/headless) yang terkoneksi ke gateway |
| **Media** | Dukungan gambar, audio, voice notes, file |
| **Canvas** | Render HTML/CSS di node — untuk present UI ke user |

---

## Browser Tool — Web Automation

OpenClaw punya browser bawaan (Chromium/Playwright) yang bisa dikontrol agent.

### Enable Browser

Browser default sudah enabled. Untuk disable:

```json
{
  "browser": {
    "enabled": false
  }
}
```

### Actions yang Tersedia

| Action | Fungsi |
|--------|--------|
| `status` | Cek status browser |
| `start` | Start browser |
| `stop` | Stop browser |
| `tabs` | List open tabs |
| `open` | Buka URL baru |
| `focus` | Focus ke tab tertentu |
| `close` | Tutup tab |
| `snapshot` | Ambil accessibility tree (untuk AI "lihat" halaman) |
| `screenshot` | Ambil screenshot (return image) |
| `act` | UI action: click, type, press, hover, drag, select, fill |
| `navigate` | Navigate (back, forward, reload) |
| `console` | Baca browser console logs |
| `pdf` | Generate PDF dari halaman |
| `upload` | Upload file ke input element |
| `dialog` | Handle dialog/popup |

### Flow Recommended untuk Browser Automation

```
1. browser → status / start
2. browser → open (URL)
3. browser → snapshot (ai atau aria) → agent "lihat" halaman
4. browser → act (click/type/press) → interact
5. browser → screenshot (kalau perlu visual confirmation)
```

### Snapshot Modes

| Mode | Penjelasan |
|------|-----------|
| `ai` (default) | AI-friendly snapshot — cocok untuk reasoning |
| `aria` | Accessibility tree — lebih detail, ada refs untuk `act` |

Snapshot `aria` mengembalikan refs seperti `e12` yang bisa dipakai di `act`:
```
act: click ref=e12
act: type ref=e3 text="Hello"
```

### Multi-Profile Browser

OpenClaw support multiple browser profiles:

```json
{
  "browser": {
    "defaultProfile": "chrome",
    "profiles": {
      "chrome": { "port": 18800 },
      "firefox": { "port": 18801, "cdpUrl": "ws://..." }
    }
  }
}
```

| Action | Fungsi |
|--------|--------|
| `profiles` | List semua profiles + status |
| `create-profile` | Buat profile baru (auto-allocate port) |
| `delete-profile` | Hapus profile |
| `reset-profile` | Kill orphan process di port profile |

Rules:
- Profile names: lowercase alphanumeric + hyphens (max 64 chars)
- Port range: 18800-18899 (~100 profiles max)
- Remote profiles: attach-only (no start/stop/reset)

### Browser di Docker

Kalau pakai Docker, browser perlu beberapa setup tambahan:

```bash
# Install browser dependencies di Docker
ssh my-vps "docker exec openclaw-openclaw-gateway-1 npx playwright install chromium"
```

**GOTCHA:** Browser di Docker bisa butuh memory ekstra. Pastikan container punya cukup RAM.

---

## Image & Media Support

### Image Processing

Agent bisa analyze gambar pakai `image` tool:

| Param | Required | Penjelasan |
|-------|----------|-----------|
| `image` | Ya | Path file atau URL |
| `prompt` | Tidak | Default: "Describe the image." |
| `model` | Tidak | Override image model |
| `maxBytesMb` | Tidak | Max file size cap |

**Require:** `agents.defaults.imageModel` harus dikonfigurasi, atau model default punya vision capability.

### Media di WhatsApp

Config media:
```json
{
  "channels": {
    "whatsapp": {
      "mediaMaxMb": 50
    }
  }
}
```

Tipe media yang didukung:
- **Gambar** — JPEG, PNG, GIF, WebP
- **Audio** — Voice notes (OGG Opus), MP3, WAV
- **Video** — MP4
- **Dokumen** — PDF, DOCX, dll

### Kirim Media via Agent

Agent pakai `message` tool untuk kirim media:
```json
{
  "action": "send",
  "media": "/path/to/image.png",
  "caption": "Ini gambar yang diminta"
}
```

---

## Audio & Voice Notes

### Receive Voice Notes

Kalau user kirim voice note, OpenClaw bisa:
1. **Transcribe** — convert audio ke text (butuh transcription model)
2. **Forward as file** — kirim audio file ke agent

### Config Audio

```json
{
  "agents": {
    "defaults": {
      "audio": {
        "transcription": {
          "enabled": true,
          "model": "openai/whisper-1"
        }
      }
    }
  }
}
```

### Voice Notes dari Agent

Agent bisa bikin voice notes via TTS (Text-to-Speech) — butuh node yang support audio.

---

## Nodes — Control Perangkat Remote

Nodes adalah **perangkat** yang terkoneksi ke gateway via WebSocket. Bisa macOS, iOS, Android, atau headless.

### Apa yang Bisa Dilakukan Node

| Action | Fungsi | Platform |
|--------|--------|----------|
| `status` | Cek node online/offline | Semua |
| `describe` | Detail capabilities node | Semua |
| `notify` | Kirim notifikasi | macOS |
| `run` | Jalankan command | macOS |
| `camera_snap` | Ambil foto dari kamera | macOS/iOS |
| `camera_clip` | Record video pendek | macOS/iOS |
| `screen_record` | Record layar | macOS |
| `location_get` | Dapatkan lokasi GPS | iOS/Android |

### Node Connection

Nodes connect ke gateway via WebSocket:
```
Node (macOS app) → WebSocket → Gateway (18789) → Agent
```

Pairing flow:
1. Node connect, kirim device identity
2. Gateway minta approval (pending)
3. Admin approve device

### Manage Nodes lewat CLI

```bash
# List nodes
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js nodes status"

# Approve pending node
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js nodes approve <node-id>"
```

### Contoh: Run Command di macOS Node

```json
{
  "action": "run",
  "node": "office-mac",
  "command": ["echo", "Hello from OpenClaw"],
  "env": ["FOO=bar"],
  "commandTimeoutMs": 12000,
  "invokeTimeoutMs": 45000,
  "needsScreenRecording": false
}
```

### Camera Capture

```json
{
  "action": "camera_snap",
  "node": "office-mac"
}
```

Returns: image block (bisa di-analyze pakai `image` tool).

**PENTING:** Camera/screen commands butuh node app yang foregrounded. Kalau app di background, capture gagal.

---

## Canvas — Render UI di Node

Canvas render HTML/CSS/JS di node device:

| Action | Fungsi |
|--------|--------|
| `present` | Tampilkan HTML content |
| `hide` | Sembunyikan canvas |
| `navigate` | Navigate dalam canvas |
| `eval` | Execute JavaScript di canvas |
| `snapshot` | Screenshot canvas (return image) |
| `a2ui_push` | Push A2UI content |
| `a2ui_reset` | Reset A2UI state |

### Contoh: Present HTML

```json
{
  "action": "present",
  "html": "<h1>Hello!</h1><p>Status: OK</p>"
}
```

Canvas served di gateway HTTP server:
- `http://localhost:18789/__openclaw__/canvas/` — Agent-editable HTML/CSS/JS
- `http://localhost:18789/__openclaw__/a2ui/` — A2UI host

---

## Troubleshooting

### Browser tidak bisa start

1. Cek `browser.enabled` di config
2. Cek Chromium terinstall:
   ```bash
   ssh my-vps "docker exec openclaw-openclaw-gateway-1 npx playwright install --dry-run"
   ```
3. Cek memory cukup — Chromium butuh ~500MB+

### Media tidak terkirim

1. Cek `mediaMaxMb` — file mungkin terlalu besar
2. Cek format didukung
3. Cek path file accessible dari dalam Docker container

### Node tidak terdeteksi

1. Cek node app running dan connected
2. Cek port 18789 accessible dari node
3. Cek pairing status: `nodes status`
4. Approve kalau masih pending

### Screenshot/camera return error

1. Node app harus foregrounded
2. Cek permissions (camera access di macOS perlu approval)
3. Cek node capabilities: `nodes describe <node-id>`

---

## Referensi

- Browser: https://docs.openclaw.ai/tools/browser
- Browser Login: https://docs.openclaw.ai/tools/browser-login
- Image & Media: https://docs.openclaw.ai/nodes/images
- Audio & Voice: https://docs.openclaw.ai/nodes/audio
- Nodes: https://docs.openclaw.ai/nodes
- Camera: https://docs.openclaw.ai/nodes/camera
- Canvas: https://docs.openclaw.ai/tools/canvas
