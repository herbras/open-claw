# OpenClaw: Install via Docker di VPS (Remote Server)

> Dokumentasi step-by-step install OpenClaw di VPS via SSH + Docker.
> Server: my-vps (Ubuntu, Docker 29.1.5, Compose v5.0.2)

---

## Prerequisites

Pastikan server punya:
- Docker Engine + Docker Compose v2+
- Git
- Disk space cukup (minimal ~5GB untuk image + logs)

Cek di server:
```bash
ssh my-vps "docker --version && docker compose version && git --version"
```

---

## Step 1: Clone Repo

```bash
ssh my-vps "git clone https://github.com/openclaw/openclaw.git ~/openclaw"
```

---

## Step 2: Build Docker Image

```bash
ssh my-vps "cd ~/openclaw && docker build -t openclaw:local -f Dockerfile ."
```

Proses ini butuh waktu beberapa menit (download base image node:22-bookworm, install deps, build).

---

## Step 3: Buat Directory Config

```bash
ssh my-vps "mkdir -p ~/.openclaw ~/.openclaw/workspace"
```

**PENTING - Fix Permission:**
Container jalan sebagai user `node` (uid 1000). Pastikan directory dimiliki uid 1000:

```bash
ssh my-vps "sudo chown -R 1000:1000 ~/.openclaw"
```

---

## Step 4: Generate Token & Tulis .env

```bash
ssh my-vps 'cd ~/openclaw && TOKEN=$(openssl rand -hex 32) && cat > .env << EOF
OPENCLAW_CONFIG_DIR=$HOME/.openclaw
OPENCLAW_WORKSPACE_DIR=$HOME/.openclaw/workspace
OPENCLAW_GATEWAY_PORT=18789
OPENCLAW_BRIDGE_PORT=18790
OPENCLAW_GATEWAY_BIND=lan
OPENCLAW_GATEWAY_TOKEN=$TOKEN
OPENCLAW_IMAGE=openclaw:local
OPENCLAW_EXTRA_MOUNTS=
OPENCLAW_HOME_VOLUME=
OPENCLAW_DOCKER_APT_PACKAGES=
EOF
echo "Token: $TOKEN"'
```

Catat token yang muncul - ini dipakai untuk akses dashboard.

---

## Step 5: Onboarding Wizard (Interaktif)

Ini HARUS dijalankan langsung karena butuh input interaktif:

```bash
ssh -t my-vps "cd ~/openclaw && docker compose run --rm openclaw-cli onboard --no-install-daemon"
```

Wizard akan tanya:
- **Gateway bind**: pilih `lan` (PENTING! jangan `loopback`)
- **Gateway auth**: `token`
- **Gateway token**: paste token dari Step 4
- **Tailscale exposure**: `Off`
- **Install Gateway daemon**: `No`
- **AI provider**: pilih sesuai kebutuhan (misal OpenRouter)

### Gotcha: bind loopback vs lan

Kalau wizard set bind ke `loopback`, gateway cuma listen di 127.0.0.1 di dalam container.
Ini bikin:
- CLI container tidak bisa connect ke gateway
- Dashboard tidak bisa diakses dari luar

**Fix manual kalau terlanjur `loopback`:**
```bash
ssh my-vps "sed -i 's/\"bind\": \"loopback\"/\"bind\": \"lan\"/' ~/.openclaw/openclaw.json"
ssh my-vps "cd ~/openclaw && docker compose restart openclaw-gateway"
```

---

## Step 6: Start Gateway

```bash
ssh my-vps "cd ~/openclaw && docker compose up -d openclaw-gateway"
```

Verifikasi:
```bash
# Cek container jalan
ssh my-vps "cd ~/openclaw && docker compose ps"

# Cek logs
ssh my-vps "cd ~/openclaw && docker compose logs --tail=10 openclaw-gateway"
```

Output yang diharapkan:
```
[gateway] listening on ws://0.0.0.0:18789 (PID ...)
[whatsapp] Listening for personal WhatsApp inbound messages.
```

Warning tentang `CLAUDE_AI_SESSION_KEY`, `CLAUDE_WEB_SESSION_KEY`, `CLAUDE_WEB_COOKIE` bisa diabaikan (fitur opsional).

---

## Step 7: Akses Dashboard via SSH Tunnel

Karena server remote, akses dashboard lewat SSH port forwarding.

Di **terminal lokal** (laptop/PC kamu, BUKAN di server):
```bash
ssh -N -L 18789:127.0.0.1:18789 my-vps
```

Penjelasan flag:
- `-N` = jangan jalankan command, cuma buka tunnel
- `-L 18789:127.0.0.1:18789` = forward port lokal 18789 ke port 18789 di server

Lalu buka browser di laptop:
```
http://localhost:18789/
```

Atau langsung dengan token:
```
http://localhost:18789/#token=<TOKEN_DARI_STEP_4>
```

---

## Step 8: Approve Browser Device (Pairing)

Pertama kali buka dashboard, akan muncul error:
```
disconnected (1008): pairing required
```

Fix - approve device dari dalam container gateway:

```bash
# List pending devices
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js devices list"

# Approve device (ganti REQUEST_ID dengan ID dari list)
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js devices approve <REQUEST_ID>"
```

Refresh browser setelah approve.

---

## Step 9: Approve WhatsApp Users

Ketika seseorang chat ke bot WhatsApp, mereka akan dapat pesan:
```
OpenClaw: access not configured.
Your WhatsApp phone number: +62xxxx
Pairing code: XXXXXXXX
Ask the bot owner to approve with: openclaw pairing approve whatsapp <code>
```

Approve dari Docker:
```bash
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js pairing approve whatsapp <KODE>"
```

Contoh:
```bash
# Approve nomor +628xxxxxxxxx1
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js pairing approve whatsapp XXXXXXXX"

# Approve nomor +628xxxxxxxxx2
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js pairing approve whatsapp YYYYYYYY"
```

---

## Cheat Sheet: Perintah Sehari-hari

```bash
# Start gateway
ssh my-vps "cd ~/openclaw && docker compose up -d openclaw-gateway"

# Stop gateway
ssh my-vps "cd ~/openclaw && docker compose down"

# Restart gateway
ssh my-vps "cd ~/openclaw && docker compose restart openclaw-gateway"

# Lihat logs (live)
ssh my-vps "cd ~/openclaw && docker compose logs -f openclaw-gateway"

# Lihat logs (10 terakhir)
ssh my-vps "cd ~/openclaw && docker compose logs --tail=10 openclaw-gateway"

# Cek status container
ssh my-vps "cd ~/openclaw && docker compose ps"

# Health check
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js health --token <TOKEN>"

# List devices
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js devices list"

# Approve device
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js devices approve <REQUEST_ID>"

# Approve WhatsApp user
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js pairing approve whatsapp <KODE>"

# Edit config
ssh my-vps "nano ~/.openclaw/openclaw.json"

# SSH tunnel untuk dashboard (jalankan di laptop lokal)
ssh -N -L 18789:127.0.0.1:18789 my-vps
```

---

## Troubleshooting

### "Missing config" saat start gateway
Config belum dibuat. Jalankan wizard onboard ulang:
```bash
ssh -t my-vps "cd ~/openclaw && docker compose run --rm openclaw-cli onboard --no-install-daemon"
```

### Permission errors (EACCES)
```bash
ssh my-vps "sudo chown -R 1000:1000 ~/.openclaw"
```

### CLI tidak bisa connect ke gateway
Jangan pakai `docker compose run --rm openclaw-cli ...` untuk command yang perlu connect ke gateway.
Pakai `docker compose exec openclaw-gateway node dist/index.js ...` supaya jalan di dalam container gateway yang sama.

### "disconnected (1008): pairing required"
Browser belum di-approve. Lihat Step 8 di atas.

### Gateway bind loopback (tidak bisa diakses)
```bash
ssh my-vps "sed -i 's/\"bind\": \"loopback\"/\"bind\": \"lan\"/' ~/.openclaw/openclaw.json"
ssh my-vps "cd ~/openclaw && docker compose restart openclaw-gateway"
```

---

## Arsitektur

```
Laptop/PC (browser)
    |
    | SSH Tunnel (port 18789)
    |
my-vps (VPS)
    |
    | Docker
    |
    +-- openclaw-gateway container
    |     - Gateway (ws://0.0.0.0:18789)
    |     - WhatsApp listener
    |     - AI agent (OpenRouter/auto)
    |     - Dashboard web UI
    |
    +-- Volume mounts
          - ~/.openclaw/ --> /home/node/.openclaw (config)
          - ~/.openclaw/workspace/ --> /home/node/.openclaw/workspace
```

---

## Referensi

- Docs resmi: https://docs.openclaw.ai/install/docker
- GitHub: https://github.com/openclaw/openclaw
- Channel setup: https://docs.openclaw.ai/channels
- Dashboard: https://docs.openclaw.ai/web/dashboard
- Remote access: https://docs.openclaw.ai/gateway/remote
