# English Coach — Technical Implementation Plan

## Overview

Build a local web app that reads English learning data written by Claude Code hooks
and provides a dashboard for daily review, vocabulary, article generation, and progress tracking.

The Claude Code hooks remain untouched — they are the data source. The app only reads
and extends that data.

---

## Data Sources (written by hooks, read by the app)

All files live under `~/.claude/english-learning/`.

### Daily session logs

**Path:** `~/.claude/english-learning/YYYY-MM-DD.md`  
**Written by:** `record-english.sh` (async Claude Code hook on every message)

```
# English Learning Log — 2026-05-23

- 20:04 [EN] add the history of questions without response
- 20:10 [ZH] 现在看看这个机制是什么样的
- 20:11 [ERR] test english input
```

Tags:
- `[EN]` — correct English input
- `[ZH]` — Chinese input (needs translation practice)
- `[ERR]` — English with detected errors

### Vocabulary notebook

**Path:** `~/.claude/english-learning/vocabulary.md`  
**Written by:** `english-coach-hook.sh` (appends one line per coaching session message)

```
# Vocabulary & Grammar Notes

- 2026-05-23 20:04 — Use "add" not "adding" after modal verbs like "should"
- 2026-05-23 20:11 — "mechanism" vs "system": mechanism implies moving parts/process
```

### Articles

**Path:** `~/.claude/english-learning/articles/YYYY-MM-DD.md`  
**Written by:** the app (article generator feature)

```markdown
# English Learning — 2026-05-23

[Generated article body...]

## Today's Writing Prompt
Write 2-3 sentences about ...
```

### Weekly summaries

**Path:** `~/.claude/english-learning/articles/weekly-YYYY-MM-DD.md`  
**Written by:** the app (weekly generator feature, Fridays)

### Audio files

**Path:** `~/.claude/english-learning/audio/YYYY-MM-DD.aiff`  
**Text transcript:** `~/.claude/english-learning/audio/YYYY-MM-DD.txt`  
**Written by:** macOS `say` command via launchd at 17:45

### Progress metrics

**Path:** `~/.claude/english-learning/progress.md`  
**Written by:** the app (metrics feature)

---

## Tech Stack

| Layer | Choice |
|-------|--------|
| Framework | Next.js 14 (App Router) |
| Language | TypeScript |
| UI | shadcn/ui + Tailwind CSS |
| Charts | recharts |
| File access | Node.js `fs` (server components / API routes) |
| AI API | Alibaba Bailian / DashScope (DeepSeek) |

The app runs locally (`npm run dev`, port 3000). No database — all data is flat files.

---

## App Structure

```
english-coach-app/
├── app/
│   ├── page.tsx                  # Dashboard
│   ├── log/
│   │   └── [date]/page.tsx       # Daily log viewer
│   ├── vocabulary/
│   │   └── page.tsx              # Vocabulary notebook
│   ├── articles/
│   │   └── page.tsx              # Article list + viewer
│   ├── shadowing/
│   │   └── [date]/page.tsx       # Audio + text for shadowing
│   ├── progress/
│   │   └── page.tsx              # Progress chart
│   └── api/
│       ├── generate-article/route.ts   # POST: call DeepSeek, save article
│       ├── generate-weekly/route.ts    # POST: call DeepSeek, save weekly
│       └── metrics/route.ts           # POST: recalculate progress.md
├── lib/
│   ├── files.ts       # fs helpers: read log, read vocab, list articles
│   ├── parser.ts      # parse [ZH]/[ERR]/[EN] lines from log files
│   └── deepseek.ts    # DeepSeek API client
└── .env.local
    LEARNING_DIR=/Users/youxun/.claude/english-learning
    BAILIAN_API_KEY=sk-bf354a15ae084ad99197ec25e8fbcb9c
    BAILIAN_MODEL=deepseek-v4-flash
    BAILIAN_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
```

---

## Feature Specifications

### 1. Dashboard (`/`)

Shows today's summary at a glance.

**Display:**
- Today's date and session stats: `ZH: 22 | ERR: 1 | EN: 42 | EN%: 65%`
- Latest 5 vocabulary tips (from `vocabulary.md`)
- Link to today's article if it exists, or a "Generate Article" button
- 7-day EN% trend sparkline

**Data:** Read `YYYY-MM-DD.md` for today, last 7 lines of `vocabulary.md`, check `articles/YYYY-MM-DD.md`

---

### 2. Daily Log Viewer (`/log/[date]`)

**Display:**
- Date picker to navigate between days
- Messages grouped by tag, color-coded:
  - `[EN]` → green
  - `[ERR]` → red
  - `[ZH]` → yellow
- Raw time + content, no editing

**Data:** Read and parse `YYYY-MM-DD.md`

**Parser logic:**
```typescript
type LogEntry = {
  time: string;      // "20:04"
  tag: "EN" | "ZH" | "ERR";
  message: string;
};

function parseLog(content: string): LogEntry[] {
  const re = /- (\d{2}:\d{2}) \[(EN|ZH|ERR)\] (.+)/g;
  // return matches
}
```

---

### 3. Vocabulary Notebook (`/vocabulary`)

**Display:**
- List of all tips in reverse chronological order
- Each item: `2026-05-23 20:04 — [tip text]`
- Search/filter box

**Data:** Read `vocabulary.md`, skip the header line, parse entries

**Parser logic:**
```typescript
// Line format: "- 2026-05-23 20:04 — tip text"
const re = /^- (\d{4}-\d{2}-\d{2} \d{2}:\d{2}) — (.+)$/;
```

---

### 4. Article Generator & Viewer (`/articles`)

**Display:**
- List of generated articles (date + title)
- Click to read full article (render markdown)
- "Generate Today's Article" button (calls `/api/generate-article`)
- "Generate Weekly Summary" button (visible on Fridays, calls `/api/generate-weekly`)

**API route: `POST /api/generate-article`**

```typescript
// 1. Read today's log from LEARNING_DIR/YYYY-MM-DD.md
// 2. Extract [EN], [ZH], [ERR] messages (max 40)
// 3. Call DeepSeek:
const system = `You are an English learning content creator for a Chinese native speaker.
Based on the session messages below, write a short English learning article (200-300 words)
covering vocabulary and grammar topics from the conversation. End with:

## Today's Writing Prompt
Write 2-3 sentences about [a topic related to today's session].

Return only the article in markdown.`;
// 4. Save to LEARNING_DIR/articles/YYYY-MM-DD.md
// 5. Return { path, content }
```

**API route: `POST /api/generate-weekly`**

```typescript
// 1. Read last 7 days of logs
// 2. Calculate ZH/ERR/EN counts per day (metrics table)
// 3. Collect all [ERR] messages (max 30)
// 4. Read last 30 lines of vocabulary.md
// 5. Call DeepSeek with all context:
const system = `Write a weekly English learning summary with sections:
## This Week's Progress
## Your Most Common Mistakes
## Vocabulary & Tips Recap
## Writing Challenge for Next Week`;
// 6. Save to LEARNING_DIR/articles/weekly-YYYY-MM-DD.md
// 7. Also call metrics API to update progress.md
```

**DeepSeek client (`lib/deepseek.ts`):**

```typescript
const BASE = process.env.BAILIAN_BASE_URL!;
const KEY  = process.env.BAILIAN_API_KEY!;
const MODEL = process.env.BAILIAN_MODEL!;

export async function chat(system: string, user: string, maxTokens = 600) {
  const res = await fetch(`${BASE}/chat/completions`, {
    method: "POST",
    headers: { Authorization: `Bearer ${KEY}`, "Content-Type": "application/json" },
    body: JSON.stringify({
      model: MODEL,
      messages: [{ role: "system", content: system }, { role: "user", content: user }],
      max_tokens: maxTokens,
      temperature: 0.7,
    }),
  });
  const data = await res.json();
  return data.choices[0].message.content as string;
}
```

---

### 5. Audio Shadowing (`/shadowing/[date]`)

**Display:**
- Audio player for `audio/YYYY-MM-DD.aiff`
- Article text displayed below the player (from `audio/YYYY-MM-DD.txt`)
- "Click to reveal" toggle so the user can listen first, then check

**Data:** Serve audio file via Next.js static or API route; read `.txt` file

**Note:** Audio files are generated by macOS `say -v Samantha` via launchd at 17:45.
The app does not generate audio — it only plays what exists.

**API route: `GET /api/audio/[date]`**

```typescript
// Stream LEARNING_DIR/audio/YYYY-MM-DD.aiff
// Return 404 if not found
```

---

### 6. Progress Metrics (`/progress`)

**Display:**
- Line chart: EN% per day (last 30 days) using recharts
- Summary table: date | ZH | ERR | EN | Total | EN%
- Current 7-day average highlighted

**Data:** Parse `progress.md` OR recompute from log files on demand

**API route: `POST /api/metrics`**

```typescript
// 1. Scan all YYYY-MM-DD.md files in LEARNING_DIR
// 2. Count [ZH], [ERR], [EN] per file
// 3. Write progress.md
// 4. Return array of { date, zh, err, en, total, enPct }
```

**progress.md format written by the app:**

```markdown
# Progress Metrics

| Date | ZH | ERR | EN | Total | EN% |
|------|----|----|-----|-------|-----|
| 2026-05-23 | 22 | 1 | 42 | 65 | 65% |

**7-day average EN%: 65% →**
```

---

## DeepSeek API Reference

**Endpoint:** `https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions`  
**Auth:** `Authorization: Bearer sk-bf354a15ae084ad99197ec25e8fbcb9c`  
**Model:** `deepseek-v4-flash`  
**OpenAI-compatible:** Yes — use `openai` npm package with `baseURL` override, or raw `fetch`.

```typescript
// Using openai package:
import OpenAI from "openai";
const client = new OpenAI({
  apiKey: process.env.BAILIAN_API_KEY,
  baseURL: process.env.BAILIAN_BASE_URL,
});
```

---

## File Helper (`lib/files.ts`)

```typescript
import fs from "fs";
import path from "path";

const DIR = process.env.LEARNING_DIR!;

export function logPath(date: string) {
  return path.join(DIR, `${date}.md`);
}

export function articlePath(date: string, weekly = false) {
  const name = weekly ? `weekly-${date}` : date;
  return path.join(DIR, "articles", `${name}.md`);
}

export function audioPath(date: string, ext: "aiff" | "txt") {
  return path.join(DIR, "audio", `${date}.${ext}`);
}

export function vocabPath() {
  return path.join(DIR, "vocabulary.md");
}

export function progressPath() {
  return path.join(DIR, "progress.md");
}

export function listLogDates(): string[] {
  return fs.readdirSync(DIR)
    .filter(f => /^\d{4}-\d{2}-\d{2}\.md$/.test(f))
    .map(f => f.replace(".md", ""))
    .sort();
}

export function readFile(p: string): string | null {
  try { return fs.readFileSync(p, "utf8"); }
  catch { return null; }
}

export function writeFile(p: string, content: string) {
  fs.mkdirSync(path.dirname(p), { recursive: true });
  fs.writeFileSync(p, content, "utf8");
}
```

---

## Hooks That Remain (do not touch)

These Claude Code hooks continue to write data. The app only reads.

| Hook | Script | Writes to |
|------|--------|-----------|
| UserPromptSubmit | `english-coach-hook.sh` | `vocabulary.md` (tips), session history |
| UserPromptSubmit | `record-english.sh` | `YYYY-MM-DD.md` (message log) |
| Stop | `push-english-learning.sh` | Pushes `~/.claude/english-learning/` to GitHub |

---

## Implementation Order (for Cursor)

1. `npx create-next-app@latest english-coach-app --typescript --tailwind --app`
2. Install: `shadcn/ui`, `recharts`, `openai`, `react-markdown`
3. Create `.env.local` with the variables above
4. Implement `lib/files.ts`, `lib/parser.ts`, `lib/deepseek.ts`
5. Dashboard page (`/`)
6. Daily Log Viewer (`/log/[date]`)
7. Vocabulary page (`/vocabulary`)
8. Article API routes + viewer (`/articles`)
9. Shadowing page (`/shadowing/[date]`)
10. Progress metrics page + API (`/progress`)
