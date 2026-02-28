# 06 - Agentic PM: Product Manager AI untuk Feedback Group

> Agent AI yang berperan sebagai Product Manager — riset codebase, identifikasi bug/tech debt,
> kasih saran, dan otomatis bikin Fizzy cards + GitLab issues.
> **TIDAK bisa merusak codebase** — read-only by design.

Tanggal: 2026-02-13

---

## Kenapa Butuh Ini?

Tim kita punya grup WhatsApp untuk feedback. Setiap ada laporan bug, saran fitur, atau keluhan,
agent PM ini otomatis:

1. **Riset codebase** — cari file yang relevan, baca kode, identifikasi masalah
2. **Buat GitLab issue** — dengan deskripsi teknis, file path, dan saran fix
3. **Buat Fizzy card** — masukkan ke backlog board yang sesuai
4. **Kasih analisis** — jawab di grup WA dengan temuan dan rekomendasi

---

## Arsitektur

```
┌──────────────────────────────────────────────────┐
│            Agent PM di OpenClaw (Docker)          │
│                                                  │
│  ┌──────────────┐  ┌────────────┐  ┌──────────┐  │
│  │ Codebase     │  │ GitLab     │  │ Fizzy    │  │
│  │ Reader       │  │ Issues     │  │ (exist)  │  │
│  │ Plugin       │  │ Plugin     │  │          │  │
│  │              │  │            │  │          │  │
│  │ • code_grep  │  │ • create   │  │ • cards  │  │
│  │ • code_read  │  │ • list     │  │ • report │  │
│  │ • code_tree  │  │ • comment  │  │ • boards │  │
│  └──────┬───────┘  └─────┬──────┘  └──────────┘  │
│         │                │                       │
│    READ-ONLY          PAT token                  │
│     mount               │                       │
└─────────┼────────────────┼───────────────────────┘
          │                │
    /repos/ (clone)     gitlab.com API
    di my-vps
```

### Alur Kerja Agent

```
User kirim feedback di grup WA
        │
        ▼
  Agent PM terima pesan
        │
        ├─→ code_grep: cari pattern terkait di codebase
        ├─→ code_read: baca file yang relevan
        │
        ▼
  Analisis masalah
        │
        ├─→ gitlab_create_issue: buat issue dengan detail teknis
        ├─→ fizzy_user_cards: cek backlog existing
        │
        ▼
  Reply ke grup WA dengan:
  - Ringkasan temuan
  - Link issue GitLab
  - Rekomendasi action
```

---

## Prasyarat (Apa yang Harus Disiapkan)

### 1. GitLab Personal Access Token (PAT)

Token ini dipakai agent untuk baca kode via API dan bikin issues.

**Cara bikin:**

1. Buka: https://gitlab.com/-/user_settings/personal_access_tokens
2. Klik **Add new token**
3. Isi:
   - **Token name**: `openclaw-pm-agent`
   - **Expiration date**: pilih 1 tahun ke depan
   - **Scopes** — centang:
     - [x] `read_api` — baca projects, issues, merge requests
     - [x] `read_repository` — baca isi file/kode
     - [x] `api` — buat issues dan comments (write)
4. Klik **Create personal access token**
5. **COPY token** — cuma ditampilkan sekali!

> **Catatan keamanan:** Token ini TIDAK punya scope `write_repository`,
> jadi agent tidak bisa push kode, merge MR, atau ubah file di repo.

### 2. List Repo yang Mau Di-Clone

Tulis semua repo yang ingin agent bisa baca. Contoh:

```
myorg/frontend-app
myorg/backend-api
myorg/mobile-app
myorg/shared-libs
```

Atau kalau mau semua repo dalam satu GitLab group:

```
Group name: myorg
```

### 3. Konfirmasi Grup WhatsApp

Tentukan grup WA mana yang untuk feedback. Cek group ID di log:

```bash
ssh my-vps "cd ~/openclaw && docker compose logs openclaw-gateway" | grep "Inbound message"
# Output: Inbound message 120363xxxxxx@g.us -> ...
```

---

## Komponen yang Dibangun

### Plugin 1: `codebase-reader` (BARU)

Plugin untuk baca kode dari repo yang di-clone ke server.

| Tool | Fungsi | Parameter |
|------|--------|-----------|
| `code_grep` | Search pattern (regex/text) di semua repo atau repo tertentu | `pattern`, `repo?`, `file_pattern?`, `context_lines?` |
| `code_read` | Baca file tertentu dengan line numbers | `repo`, `file_path`, `start_line?`, `end_line?` |
| `code_tree` | List struktur folder/file dari repo | `repo`, `path?`, `depth?` |

**Keamanan dalam kode:**
- Path di-resolve dan dicek HARUS di dalam `REPOS_DIR`
- Path traversal (`../../../etc/passwd`) ditolak
- Hanya operasi read: `fs.readFileSync`, `fs.readdirSync`
- TIDAK ada: `fs.writeFile`, `fs.unlink`, `child_process.exec`

### Plugin 2: `gitlab-issues` (BARU)

Plugin untuk interaksi dengan GitLab Issues API.

| Tool | Fungsi | HTTP |
|------|--------|------|
| `gitlab_list_projects` | List semua projects accessible | GET |
| `gitlab_list_issues` | List issues (filter: state/labels/search) | GET |
| `gitlab_get_issue` | Detail 1 issue + comments | GET |
| `gitlab_create_issue` | Buat issue baru (title, desc, labels) | POST |
| `gitlab_add_comment` | Tambah comment ke issue | POST |

**Yang TIDAK ada** (by design): merge MR, edit file, push kode, delete.

### Plugin Existing: `fizzy` (SUDAH ADA)

| Tool | Fungsi |
|------|--------|
| `fizzy_list_boards` | List project boards |
| `fizzy_list_users` | List team members |
| `fizzy_user_cards` | Cards assigned ke user |
| `fizzy_user_weekly_report` | Weekly progress |

---

## Step-by-Step Implementasi

### Step 1: Clone Repos ke my-vps

```bash
# Buat folder repos
ssh my-vps "mkdir -p ~/repos"

# Clone setiap repo (ganti <PAT> dan path)
ssh my-vps "cd ~/repos && git clone https://oauth2:<GITLAB_PAT>@gitlab.com/myorg/repo-1.git"
ssh my-vps "cd ~/repos && git clone https://oauth2:<GITLAB_PAT>@gitlab.com/myorg/repo-2.git"
# ... ulangi untuk setiap repo
```

**Untuk clone semua repo dalam group sekaligus:**

```bash
# Install glab CLI atau pakai curl
ssh my-vps "curl -s --header 'PRIVATE-TOKEN: <PAT>' \
  'https://gitlab.com/api/v4/groups/myorg/projects?per_page=100' \
  | python3 -c \"
import sys, json, subprocess
for p in json.load(sys.stdin):
    url = p['http_url_to_repo'].replace('https://', 'https://oauth2:<PAT>@')
    print(f'Cloning {p[\"path\"]}...')
    subprocess.run(['git', 'clone', url], cwd='/home/ubuntu/repos')
\""
```

### Step 2: Setup Auto-Pull (Cron)

Agar kode selalu up-to-date, setup cron job:

```bash
# Tambah cron: pull semua repo tiap 30 menit
ssh my-vps "crontab -l 2>/dev/null | { cat; echo '*/30 * * * * cd /home/ubuntu/repos && for d in */; do (cd \"\$d\" && git pull --ff-only --quiet 2>/dev/null) & done'; } | crontab -"

# Verifikasi cron terpasang
ssh my-vps "crontab -l"
```

### Step 3: Mount Repos READ-ONLY ke Docker

Edit `~/openclaw/docker-compose.yml` di my-vps:

```yaml
services:
  openclaw-gateway:
    volumes:
      - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
      - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
      - /home/ubuntu/repos:/home/node/repos:ro    # ← TAMBAH INI
```

**Flag `:ro`** = read-only di level OS.
Bahkan kalau ada bug di plugin, secara fisik Docker TIDAK BISA menulis ke folder ini.

**Verifikasi setelah restart:**

```bash
# Harus berhasil (read)
ssh my-vps "docker exec openclaw-openclaw-gateway-1 ls /home/node/repos/"

# Harus GAGAL (write blocked)
ssh my-vps "docker exec openclaw-openclaw-gateway-1 touch /home/node/repos/test 2>&1"
# → Expected: "Read-only file system"
```

### Step 4: Buat Plugin `codebase-reader`

**Di local machine** (`~/openclaw-plugin-codebase-reader/`):

Struktur:
```
openclaw-plugin-codebase-reader/
├── openclaw.plugin.json
├── package.json
├── tsconfig.json
└── index.ts
```

**`openclaw.plugin.json`:**
```json
{
  "id": "codebase-reader",
  "name": "Codebase Reader",
  "description": "Read-only codebase exploration: grep, read files, browse directory trees",
  "configSchema": {
    "type": "object",
    "properties": {
      "REPOS_DIR": {
        "type": "string",
        "description": "Path to repos directory (default: /home/node/repos)"
      }
    }
  }
}
```

**`index.ts` — mendaftarkan 3 tools:**

```ts
import { readdirSync, readFileSync, statSync } from "fs";
import { join, resolve, relative } from "path";
import { execSync } from "child_process";

function ok(text: string) {
  return { content: [{ type: "text" as const, text }] };
}

function err(msg: string) {
  return { content: [{ type: "text" as const, text: `Error: ${msg}` }] };
}

// Keamanan: pastikan path SELALU di dalam reposDir
function safePath(reposDir: string, ...parts: string[]): string {
  const resolved = resolve(reposDir, ...parts);
  if (!resolved.startsWith(reposDir)) {
    throw new Error("Path traversal blocked");
  }
  return resolved;
}

function listRepos(reposDir: string): string[] {
  return readdirSync(reposDir)
    .filter((d) => statSync(join(reposDir, d)).isDirectory());
}

const plugin = {
  id: "codebase-reader",
  name: "Codebase Reader",

  register(api: any) {
    const config = api.pluginConfig ?? {};
    const REPOS_DIR = config.REPOS_DIR || "/home/node/repos";

    // Tool 1: code_grep
    api.registerTool({
      name: "code_grep",
      description: "Search for a pattern (text/regex) across repos. Returns matching lines with context.",
      parameters: {
        type: "object",
        required: ["pattern"],
        properties: {
          pattern:       { type: "string", description: "Search pattern (regex supported)" },
          repo:          { type: "string", description: "Repo name to search in (omit = all repos)" },
          file_pattern:  { type: "string", description: "File glob filter, e.g. '*.ts' or '*.py'" },
          context_lines: { type: "number", description: "Lines of context around match (default: 3)" },
          max_results:   { type: "number", description: "Max matches to return (default: 50)" },
        },
      },
      async execute(_id: string, params: Record<string, any>) {
        try {
          const searchDir = params.repo
            ? safePath(REPOS_DIR, params.repo)
            : REPOS_DIR;
          const ctx = params.context_lines ?? 3;
          const max = params.max_results ?? 50;

          let cmd = `grep -rn --include='${params.file_pattern || "*"}' -C ${ctx} -m ${max}`;
          cmd += ` -E '${params.pattern.replace(/'/g, "'\\''")}'`;
          cmd += ` '${searchDir}'`;

          const result = execSync(cmd, {
            encoding: "utf-8",
            maxBuffer: 1024 * 1024,
            timeout: 15000,
          });

          // Strip absolute path prefix for cleaner output
          const clean = result.replace(new RegExp(REPOS_DIR + "/", "g"), "");
          return ok(clean || "No matches found.");
        } catch (e: any) {
          if (e.status === 1) return ok("No matches found.");
          return err(e.message);
        }
      },
    });

    // Tool 2: code_read
    api.registerTool({
      name: "code_read",
      description: "Read a file from a repo with line numbers. Supports reading a range of lines.",
      parameters: {
        type: "object",
        required: ["repo", "file_path"],
        properties: {
          repo:       { type: "string", description: "Repo name" },
          file_path:  { type: "string", description: "Path relative to repo root, e.g. 'src/app.ts'" },
          start_line: { type: "number", description: "Start from this line (1-based, default: 1)" },
          end_line:   { type: "number", description: "End at this line (default: end of file)" },
        },
      },
      async execute(_id: string, params: Record<string, any>) {
        try {
          const filePath = safePath(REPOS_DIR, params.repo, params.file_path);
          const content = readFileSync(filePath, "utf-8");
          const lines = content.split("\n");

          const start = Math.max(1, params.start_line || 1);
          const end = Math.min(lines.length, params.end_line || lines.length);

          const numbered = lines
            .slice(start - 1, end)
            .map((line, i) => `${String(start + i).padStart(5)}  ${line}`)
            .join("\n");

          const header = `${params.repo}/${params.file_path} (lines ${start}-${end} of ${lines.length})`;
          return ok(`${header}\n${"─".repeat(60)}\n${numbered}`);
        } catch (e: any) {
          return err(e.message);
        }
      },
    });

    // Tool 3: code_tree
    api.registerTool({
      name: "code_tree",
      description:
        "List directory tree of a repo. Shows files and folders with configurable depth.",
      parameters: {
        type: "object",
        required: ["repo"],
        properties: {
          repo:  { type: "string", description: "Repo name" },
          path:  { type: "string", description: "Sub-path to list (default: root)" },
          depth: { type: "number", description: "Max depth to traverse (default: 3)" },
        },
      },
      async execute(_id: string, params: Record<string, any>) {
        try {
          const base = safePath(REPOS_DIR, params.repo, params.path || "");
          const maxDepth = params.depth || 3;
          const lines: string[] = [];

          function walk(dir: string, prefix: string, currentDepth: number) {
            if (currentDepth > maxDepth) return;
            const entries = readdirSync(dir).filter(
              (e) => !e.startsWith(".") && e !== "node_modules"
            );
            entries.sort();

            entries.forEach((entry, i) => {
              const fullPath = join(dir, entry);
              const isLast = i === entries.length - 1;
              const connector = isLast ? "└── " : "├── ";
              const stat = statSync(fullPath);

              if (stat.isDirectory()) {
                lines.push(`${prefix}${connector}${entry}/`);
                const childPrefix = prefix + (isLast ? "    " : "│   ");
                walk(fullPath, childPrefix, currentDepth + 1);
              } else {
                lines.push(`${prefix}${connector}${entry}`);
              }
            });
          }

          walk(base, "", 1);

          if (lines.length === 0) return ok("Empty directory.");

          const header = params.path
            ? `${params.repo}/${params.path}`
            : params.repo;
          return ok(`${header}/\n${lines.join("\n")}`);
        } catch (e: any) {
          return err(e.message);
        }
      },
    });

    // Bonus: list all repos
    api.registerTool({
      name: "code_list_repos",
      description: "List all available repos that can be explored.",
      parameters: { type: "object", properties: {} },
      async execute() {
        try {
          const repos = listRepos(REPOS_DIR);
          return ok(`Available repos (${repos.length}):\n${repos.map((r) => `  • ${r}`).join("\n")}`);
        } catch (e: any) {
          return err(e.message);
        }
      },
    });

    api.logger.info(`[codebase-reader] 4 code exploration tools registered`);
  },
};

export default plugin;
```

### Step 5: Buat Plugin `gitlab-issues`

**Di local machine** (`~/openclaw-plugin-gitlab-issues/`):

Struktur:
```
openclaw-plugin-gitlab-issues/
├── openclaw.plugin.json
├── package.json
├── tsconfig.json
└── index.ts
```

**`openclaw.plugin.json`:**
```json
{
  "id": "gitlab-issues",
  "name": "GitLab Issues",
  "description": "Create and manage GitLab issues via REST API",
  "configSchema": {
    "type": "object",
    "required": ["GITLAB_TOKEN"],
    "properties": {
      "GITLAB_TOKEN": {
        "type": "string",
        "description": "GitLab Personal Access Token"
      },
      "GITLAB_URL": {
        "type": "string",
        "description": "GitLab base URL (default: https://gitlab.com)"
      }
    }
  }
}
```

**`index.ts` — mendaftarkan 5 tools:**

```ts
function ok(text: string) {
  return { content: [{ type: "text" as const, text }] };
}
function err(msg: string) {
  return { content: [{ type: "text" as const, text: `Error: ${msg}` }] };
}

const plugin = {
  id: "gitlab-issues",
  name: "GitLab Issues",

  register(api: any) {
    const config = api.pluginConfig ?? {};
    const TOKEN = config.GITLAB_TOKEN || "";
    const BASE = (config.GITLAB_URL || "https://gitlab.com").replace(/\/$/, "");
    const API = `${BASE}/api/v4`;

    const headers = {
      "PRIVATE-TOKEN": TOKEN,
      "Content-Type": "application/json",
    };

    async function gitlabGet(path: string) {
      const r = await fetch(`${API}${path}`, { headers });
      if (!r.ok) throw new Error(`GitLab API ${r.status}: ${r.statusText}`);
      return r.json();
    }

    async function gitlabPost(path: string, body: any) {
      const r = await fetch(`${API}${path}`, {
        method: "POST", headers, body: JSON.stringify(body),
      });
      if (!r.ok) throw new Error(`GitLab API ${r.status}: ${r.statusText}`);
      return r.json();
    }

    // Tool 1: gitlab_list_projects
    api.registerTool({
      name: "gitlab_list_projects",
      description: "List GitLab projects accessible with current token. Returns id, name, URL.",
      parameters: {
        type: "object",
        properties: {
          search: { type: "string", description: "Search by project name" },
          per_page: { type: "number", description: "Results per page (default: 20)" },
        },
      },
      async execute(_id: string, params: Record<string, any>) {
        try {
          const search = params.search ? `&search=${encodeURIComponent(params.search)}` : "";
          const perPage = params.per_page || 20;
          const projects = await gitlabGet(
            `/projects?membership=true&per_page=${perPage}${search}`
          );
          const list = projects.map((p: any) =>
            `• [${p.id}] ${p.path_with_namespace} — ${p.web_url}`
          );
          return ok(`Projects (${list.length}):\n${list.join("\n")}`);
        } catch (e: any) { return err(e.message); }
      },
    });

    // Tool 2: gitlab_list_issues
    api.registerTool({
      name: "gitlab_list_issues",
      description: "List issues in a GitLab project. Filter by state, labels, search text.",
      parameters: {
        type: "object",
        required: ["project_id"],
        properties: {
          project_id: { type: "number", description: "GitLab project ID" },
          state:      { type: "string", enum: ["opened", "closed", "all"], description: "Filter by state (default: opened)" },
          labels:     { type: "string", description: "Comma-separated labels to filter" },
          search:     { type: "string", description: "Search in title/description" },
          per_page:   { type: "number", description: "Results per page (default: 20)" },
        },
      },
      async execute(_id: string, params: Record<string, any>) {
        try {
          const pid = params.project_id;
          const state = params.state || "opened";
          const perPage = params.per_page || 20;
          let qs = `state=${state}&per_page=${perPage}`;
          if (params.labels) qs += `&labels=${encodeURIComponent(params.labels)}`;
          if (params.search) qs += `&search=${encodeURIComponent(params.search)}`;

          const issues = await gitlabGet(`/projects/${pid}/issues?${qs}`);
          const list = issues.map((i: any) =>
            `• #${i.iid} [${i.state}] ${i.title}${i.labels.length ? ` (${i.labels.join(", ")})` : ""}\n  ${i.web_url}`
          );
          return ok(`Issues (${list.length}):\n${list.join("\n")}`);
        } catch (e: any) { return err(e.message); }
      },
    });

    // Tool 3: gitlab_get_issue
    api.registerTool({
      name: "gitlab_get_issue",
      description: "Get details of a single GitLab issue including description and comments.",
      parameters: {
        type: "object",
        required: ["project_id", "issue_iid"],
        properties: {
          project_id: { type: "number", description: "GitLab project ID" },
          issue_iid:  { type: "number", description: "Issue IID (the # number)" },
        },
      },
      async execute(_id: string, params: Record<string, any>) {
        try {
          const { project_id: pid, issue_iid: iid } = params;
          const issue = await gitlabGet(`/projects/${pid}/issues/${iid}`);
          const notes = await gitlabGet(`/projects/${pid}/issues/${iid}/notes?per_page=20`);

          let text = `# #${issue.iid}: ${issue.title}\n`;
          text += `State: ${issue.state} | Labels: ${issue.labels.join(", ") || "none"}\n`;
          text += `Author: ${issue.author?.name} | Created: ${issue.created_at}\n`;
          text += `URL: ${issue.web_url}\n\n`;
          text += `## Description\n${issue.description || "(empty)"}\n`;

          if (notes.length > 0) {
            text += `\n## Comments (${notes.length})\n`;
            for (const n of notes.filter((n: any) => !n.system)) {
              text += `\n--- ${n.author?.name} (${n.created_at}) ---\n${n.body}\n`;
            }
          }
          return ok(text);
        } catch (e: any) { return err(e.message); }
      },
    });

    // Tool 4: gitlab_create_issue
    api.registerTool({
      name: "gitlab_create_issue",
      description: "Create a new issue in a GitLab project.",
      parameters: {
        type: "object",
        required: ["project_id", "title"],
        properties: {
          project_id:  { type: "number", description: "GitLab project ID" },
          title:       { type: "string", description: "Issue title" },
          description: { type: "string", description: "Issue body (Markdown supported)" },
          labels:      { type: "string", description: "Comma-separated labels" },
          assignee_id: { type: "number", description: "User ID to assign" },
        },
      },
      async execute(_id: string, params: Record<string, any>) {
        try {
          const body: any = {
            title: params.title,
            description: params.description || "",
          };
          if (params.labels) body.labels = params.labels;
          if (params.assignee_id) body.assignee_ids = [params.assignee_id];

          const issue = await gitlabPost(
            `/projects/${params.project_id}/issues`, body
          );
          return ok(`Issue created: #${issue.iid} — ${issue.title}\n${issue.web_url}`);
        } catch (e: any) { return err(e.message); }
      },
    });

    // Tool 5: gitlab_add_comment
    api.registerTool({
      name: "gitlab_add_comment",
      description: "Add a comment/note to an existing GitLab issue.",
      parameters: {
        type: "object",
        required: ["project_id", "issue_iid", "body"],
        properties: {
          project_id: { type: "number", description: "GitLab project ID" },
          issue_iid:  { type: "number", description: "Issue IID" },
          body:       { type: "string", description: "Comment text (Markdown)" },
        },
      },
      async execute(_id: string, params: Record<string, any>) {
        try {
          const note = await gitlabPost(
            `/projects/${params.project_id}/issues/${params.issue_iid}/notes`,
            { body: params.body }
          );
          return ok(`Comment added to #${params.issue_iid}\nID: ${note.id}`);
        } catch (e: any) { return err(e.message); }
      },
    });

    api.logger.info(`[gitlab-issues] 5 issue management tools registered`);
  },
};

export default plugin;
```

### Step 6: Buat Workspace Agent PM

Buat folder dan SOUL.md di my-vps:

```bash
ssh my-vps "mkdir -p ~/.openclaw/workspace-pm"
```

**`~/.openclaw/workspace-pm/SOUL.md`:**

```markdown
# PM Agent — Product Manager AI

Kamu adalah Product Manager AI yang membantu tim mengelola backlog produk.

## Tugasmu

1. **Riset Codebase** — Saat ada laporan bug atau feedback, gunakan `code_grep`
   dan `code_read` untuk cari file yang relevan dan pahami konteks masalahnya.
2. **Buat GitLab Issue** — Setelah analisis, buat issue di GitLab dengan:
   - Title yang jelas dan actionable
   - Description berisi: ringkasan masalah, file path + line number, analisis root cause, saran fix
   - Labels yang sesuai (bug, enhancement, tech-debt, dll)
3. **Buat/Cek Fizzy Card** — Masukkan ke backlog board yang sesuai
4. **Reply dengan Analisis** — Jawab di chat dengan temuan dan link ke issue

## Format Reply

Saat merespon feedback, gunakan format:

📋 **Analisis: [Judul Singkat]**

**Temuan:**
- File: `repo/path/to/file.ts:42` — [deskripsi masalah]
- Related: `repo/path/other.ts:15` — [konteks]

**Root Cause:** [penjelasan]

**Rekomendasi:** [saran action]

**GitLab Issue:** [link jika sudah dibuat]

## Aturan

- Kamu HANYA bisa MEMBACA kode, TIDAK bisa mengubah kode
- Selalu sertakan file path dan line number dalam analisis
- Kalau tidak yakin, bilang "perlu investigasi lebih lanjut"
- Bahasa: Indonesia campur teknis (code terms tetap English)
- Jangan asal tebak — selalu grep/read dulu sebelum analisis
- Jangan buat issue duplikat — cek `gitlab_list_issues` dulu
```

### Step 7: Update `openclaw.json`

Tambahkan ke config:

**Agents** — tambah agent PM ke `agents.list`:

```json
{
  "id": "pm-agent",
  "name": "PM Agent",
  "workspace": "/home/node/.openclaw/workspace-pm",
  "tools": {
    "allow": [
      "code_grep",
      "code_read",
      "code_tree",
      "code_list_repos",
      "gitlab_list_projects",
      "gitlab_list_issues",
      "gitlab_get_issue",
      "gitlab_create_issue",
      "gitlab_add_comment",
      "fizzy_list_boards",
      "fizzy_list_users",
      "fizzy_user_cards",
      "fizzy_user_weekly_report"
    ]
  }
}
```

**Plugins** — tambah ke `plugins.entries`:

```json
{
  "codebase-reader": {
    "enabled": true,
    "config": {
      "REPOS_DIR": "/home/node/repos"
    }
  },
  "gitlab-issues": {
    "enabled": true,
    "config": {
      "GITLAB_TOKEN": "<PAT_TOKEN_KAMU>",
      "GITLAB_URL": "https://gitlab.com"
    }
  }
}
```

**Bindings** — routing grup feedback ke PM agent:

```json
{
  "bindings": [
    {
      "agentId": "pm-agent",
      "match": {
        "channel": "whatsapp",
        "peer": { "kind": "group", "id": "<FEEDBACK_GROUP_ID>" }
      }
    }
  ]
}
```

### Step 8: Deploy ke my-vps

```bash
# 1. Rsync plugins ke server
rsync -avz --exclude='node_modules' --exclude='.git' \
  ~/openclaw-plugin-codebase-reader/ \
  my-vps:~/.openclaw/extensions/codebase-reader/

rsync -avz --exclude='node_modules' --exclude='.git' \
  ~/openclaw-plugin-gitlab-issues/ \
  my-vps:~/.openclaw/extensions/gitlab-issues/

# 2. npm install di dalam Docker
ssh my-vps "docker exec openclaw-openclaw-gateway-1 sh -c \
  'cd /home/node/.openclaw/extensions/codebase-reader && npm install --production'"

ssh my-vps "docker exec openclaw-openclaw-gateway-1 sh -c \
  'cd /home/node/.openclaw/extensions/gitlab-issues && npm install --production'"

# 3. Restart gateway
ssh my-vps "cd ~/openclaw && docker compose up -d"
# (up -d karena docker-compose.yml juga berubah — volume baru)
```

### Step 9: Verifikasi

```bash
# 1. Cek plugins loaded
ssh my-vps "cd ~/openclaw && docker compose logs --tail=20 openclaw-gateway" \
  | grep -i "register\|error"
# → [codebase-reader] 4 code exploration tools registered
# → [gitlab-issues] 5 issue management tools registered

# 2. Cek read-only mount
ssh my-vps "docker exec openclaw-openclaw-gateway-1 ls /home/node/repos/"
# → repo-1/ repo-2/ ...

ssh my-vps "docker exec openclaw-openclaw-gateway-1 touch /home/node/repos/test 2>&1"
# → touch: /home/node/repos/test: Read-only file system ✓

# 3. Test di WhatsApp
# Kirim di grup feedback: "cek error handling di endpoint /api/users"
# Agent harus: grep → read file → analisis → reply + buat issue
```

---

## Keamanan (5 Lapis)

| Layer | Proteksi | Apa yang dicegah |
|-------|----------|-----------------|
| **1. Docker `:ro`** | Volume mount read-only di OS level | Agent tidak bisa tulis file apapun ke repos |
| **2. Tools terbatas** | `tools.allow` hanya 13 tools aman | Tidak ada bash, edit, write, delete |
| **3. Path validation** | Plugin cek path traversal di kode | Tidak bisa baca `/etc/passwd` atau file luar repos |
| **4. GitLab PAT scope** | Token tanpa `write_repository` | Tidak bisa push, merge, edit file via API |
| **5. Agent terpisah** | PM agent ≠ Huda, workspace berbeda | Huda tetap aman, PM agent terisolasi |

### Apa yang Agent PM BISA:
- Baca file kode (grep, read, tree)
- Buat GitLab issue & comment
- Lihat Fizzy boards & cards
- Reply di WhatsApp grup feedback

### Apa yang Agent PM TIDAK BISA:
- Edit/hapus file kode
- Push ke Git / merge MR
- Akses bash/terminal
- Baca file di luar `/home/node/repos/`
- Akses repo/group yang tidak di-clone

---

## Tools Reference (Quick)

| Tool | Plugin | Fungsi |
|------|--------|--------|
| `code_grep` | codebase-reader | Search pattern di repos |
| `code_read` | codebase-reader | Baca file + line numbers |
| `code_tree` | codebase-reader | Directory listing |
| `code_list_repos` | codebase-reader | List semua repos |
| `gitlab_list_projects` | gitlab-issues | List GitLab projects |
| `gitlab_list_issues` | gitlab-issues | List issues (filter) |
| `gitlab_get_issue` | gitlab-issues | Detail issue + comments |
| `gitlab_create_issue` | gitlab-issues | Buat issue baru |
| `gitlab_add_comment` | gitlab-issues | Comment ke issue |
| `fizzy_list_boards` | fizzy (existing) | List project boards |
| `fizzy_list_users` | fizzy (existing) | List team members |
| `fizzy_user_cards` | fizzy (existing) | Cards per user |
| `fizzy_user_weekly_report` | fizzy (existing) | Weekly progress |

---

## Checklist Sebelum Mulai

- [ ] Bikin GitLab PAT (scope: `read_api`, `read_repository`, `api`)
- [ ] Tentukan list repo yang mau di-clone
- [ ] Konfirmasi group ID WhatsApp untuk feedback
- [ ] Share PAT token untuk dimasukkan ke config

---

## Troubleshooting

### Agent PM tidak merespon di grup

1. Cek binding sudah benar (group ID match)
2. Cek agent ada di `agents.list`
3. Cek log: `docker compose logs --tail=50 openclaw-gateway | grep "pm-agent\|binding"`

### `code_grep` error "No such file or directory"

1. Cek volume mount: `docker exec ... ls /home/node/repos/`
2. Cek repo sudah di-clone: `ssh my-vps "ls ~/repos/"`
3. Restart Docker jika baru tambah volume: `docker compose up -d`

### GitLab API 401 Unauthorized

1. PAT expired — bikin yang baru
2. PAT scope kurang — perlu minimal `read_api` + `api`
3. Cek token di config: `plugins.entries.gitlab-issues.config.GITLAB_TOKEN`

### GitLab API 404 Not Found

1. Project ID salah — pakai `gitlab_list_projects` dulu untuk cari ID yang benar
2. Issue IID salah — pakai `gitlab_list_issues` dulu

### Kode tidak up-to-date

1. Cek cron jalan: `ssh my-vps "crontab -l"`
2. Manual pull: `ssh my-vps "cd ~/repos/repo-name && git pull"`
3. Cek git remote: `ssh my-vps "cd ~/repos/repo-name && git remote -v"`
