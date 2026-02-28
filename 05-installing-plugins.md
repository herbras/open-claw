# OpenClaw: Memasang Plugin / Extension

> Cara install plugin custom ke OpenClaw gateway yang berjalan di Docker.

---

## Konsep

| Istilah | Penjelasan |
|---------|-----------|
| **Plugin** | Kode TypeScript yang mendaftarkan tools ke OpenClaw |
| **Extension** | Folder plugin di `~/.openclaw/extensions/<nama>/` |
| **Tool** | Fungsi yang bisa dipanggil AI agent (misal `cv_review`, `generate_image`) |
| **Skill** | Prompt/instruksi di `skills/<nama>/SKILL.md` — panduan cara AI pakai tools |

---

## Struktur Plugin Minimal

```
~/.openclaw/extensions/nama-plugin/
├── openclaw.plugin.json    # Manifest: id, name, description, configSchema
├── package.json            # Dependencies
├── index.ts                # Entry point — export register function
├── src/                    # Source code
│   └── tools/
│       └── my-tool.ts      # Tool implementations
└── skills/                 # (opsional) Skill prompts
    └── my-skill/
        └── SKILL.md
```

### `openclaw.plugin.json` (Manifest)

```json
{
  "id": "nama-plugin",
  "name": "Nama Plugin",
  "description": "Deskripsi singkat",
  "configSchema": {
    "type": "object",
    "properties": {
      "API_KEY": {
        "type": "string",
        "description": "API key untuk service"
      }
    }
  }
}
```

### `index.ts` (Entry Point)

Ada 2 format export yang didukung:

**Format 1: Function (simpel)**
```ts
export default function register(api: any) {
  api.registerTool({ name: "my_tool", ... });
}
```

**Format 2: Object (lebih lengkap)**
```ts
export default {
  id: "nama-plugin",
  name: "Nama Plugin",
  register(api: any) {
    api.registerTool({ name: "my_tool", ... });
  }
};
```

### Tool Registration

```ts
api.registerTool({
  name: "tool_name",
  description: "Deskripsi tool — AI baca ini untuk tahu kapan pakai",
  parameters: {
    type: "object",
    required: ["param1"],
    properties: {
      param1: { type: "string", description: "..." },
      param2: { type: "number", description: "..." },
    },
  },
  async execute(params: Record<string, any>) {
    // ... logic ...
    return { content: [{ type: "text", text: "result" }] };
  },
});
```

---

## Langkah Install Plugin

### 1. Copy plugin ke server

```bash
# Dari local ke my-vps (exclude node_modules!)
rsync -avz --exclude='node_modules' --exclude='.git' \
  /path/to/my-plugin/ \
  my-vps:~/.openclaw/extensions/my-plugin/
```

### 2. Install dependencies di dalam Docker

```bash
ssh my-vps "docker exec openclaw-openclaw-gateway-1 sh -c \
  'cd /home/node/.openclaw/extensions/my-plugin && npm install --production'"
```

**PENTING:** Install di DALAM Docker, bukan di host! Karena:
- Docker pakai Linux x64, host mungkin beda arch
- Native modules (seperti `@resvg/resvg-js`) harus compile untuk platform Docker
- Path dependencies harus resolve di dalam container

### 3. Tambahkan ke `openclaw.json`

```bash
ssh my-vps "python3 -c \"
import json
with open('/home/ubuntu/.openclaw/openclaw.json') as f:
    cfg = json.load(f)

# Daftarkan plugin
cfg['plugins']['entries']['my-plugin'] = {
    'enabled': True,
    'config': {
        'API_KEY': 'xxx'  # sesuai configSchema
    }
}

# Tambahkan tools ke agent
for a in cfg['agents']['list']:
    if a['id'] == 'main':
        allow = a.get('tools', {}).get('allow', [])
        for tool in ['my_tool_1', 'my_tool_2']:
            if tool not in allow:
                allow.append(tool)
        a['tools']['allow'] = allow
        break

with open('/home/ubuntu/.openclaw/openclaw.json', 'w') as f:
    json.dump(cfg, f, indent=2)
print('Done!')
\""
```

### 4. Restart gateway

```bash
ssh my-vps "cd ~/openclaw && docker compose restart openclaw-gateway"
```

### 5. Verifikasi

```bash
ssh my-vps "cd ~/openclaw && docker compose logs --tail=15 openclaw-gateway" | grep -i "register\|error\|invalid"
```

Harus muncul:
```
[my-plugin] X tools registered
```

Kalau ada error: `Unrecognized key: "my-plugin"` di plugins, berarti plugin manifest-nya belum benar.

---

## Fonts untuk Image Rendering (Satori/Resvg)

Kalau plugin perlu generate gambar (PNG scorecard, chart, dll), perlu font:

### Font yang sudah tersedia di Docker

```
/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf        ✓
/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf    ✓
/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf     ✓
/usr/share/fonts/truetype/dejavu/DejaVuSerif.ttf        ✓
```

### Install font tambahan di Docker

```bash
# Install Noto Sans (bahasa Arab, CJK, dll)
ssh my-vps "docker exec openclaw-openclaw-gateway-1 sh -c \
  'apk add --no-cache font-noto'"

# Atau copy font manual
scp MyFont.ttf my-vps:~/.openclaw/fonts/
ssh my-vps "docker exec openclaw-openclaw-gateway-1 sh -c \
  'mkdir -p /usr/share/fonts/custom && \
   cp /home/node/.openclaw/fonts/MyFont.ttf /usr/share/fonts/custom/ && \
   fc-cache -fv'"
```

### Cek font tersedia

```bash
ssh my-vps "docker exec openclaw-openclaw-gateway-1 fc-list"
```

### Pattern di kode: fallback font paths

```ts
const FONT_PATHS = [
  "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf",  // Docker Alpine
  "/usr/share/fonts/TTF/DejaVuSans.ttf",               // Arch Linux
  "/System/Library/Fonts/Helvetica.ttc",                // macOS
];

function loadFont(): ArrayBuffer {
  for (const p of FONT_PATHS) {
    try { return readFileSync(p).buffer; } catch { /* next */ }
  }
  throw new Error("No font available");
}
```

---

## Config Structure Reference

```json
{
  "plugins": {
    "entries": {
      "nama-plugin": {
        "enabled": true,
        "config": { "KEY": "value" }
      }
    }
  },
  "agents": {
    "list": [
      {
        "id": "main",
        "tools": {
          "allow": ["tool_1", "tool_2"]
        }
      }
    ]
  }
}
```

**GOTCHA:**
- `plugins` punya sub-key `entries` (object) — BUKAN langsung `plugins.nama-plugin`
- `agents` punya sub-key `list` (array) — BUKAN `entries`
- `package.json` name harus cocok dengan folder name / manifest id, kalau tidak akan ada warning "plugin id mismatch"

---

## Contoh Plugin yang Sudah Berjalan (my-vps)

| Plugin | Tools | Fungsi |
|--------|-------|--------|
| `pollinations` | `generate_image` | Generate gambar AI dari teks |
| `turath` | 7 tools (`turath_search`, dll) | Riset kitab Islam klasik |
| `cv-advisor` | `cv_review`, `cv_scorecard` | Review CV + scorecard PNG |
| `fizzy` | 4 tools (`fizzy_*`) | Project management reports |
| `founderplus` | 14 tools (`fp_*`) | Commerce/checkout |

---

## Troubleshooting

### Plugin tidak ter-load (tidak muncul di log)

1. Cek folder ada: `docker exec ... ls /home/node/.openclaw/extensions/my-plugin/`
2. Cek `openclaw.plugin.json` valid JSON
3. Cek `index.ts` punya export (default function atau default object with register)

### "Unrecognized key" di startup

Plugin belum terdaftar di `plugins.entries`. Tambahkan:
```json
"plugins": { "entries": { "my-plugin": { "enabled": true } } }
```

### "plugin id mismatch" warning

`package.json` name berbeda dari folder name / manifest id. Samakan ketiganya.

### Tool tidak muncul untuk agent

Cek `agents.list[].tools.allow` — tool harus explicitly di-allow untuk setiap agent.

### npm install gagal (native module)

Pastikan install DI DALAM Docker:
```bash
docker exec ... sh -c 'cd /home/node/.openclaw/extensions/my-plugin && npm install'
```
BUKAN di host lalu copy node_modules.
