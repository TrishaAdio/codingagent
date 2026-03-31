# Claude Code — Memory System: Deep-Dive Architecture

> **A complete technical reference for contributors, power-users, and the curious.**  
> Covers every layer of how Claude Code persists, recalls, consolidates, and protects memory across sessions.

---

## Table of Contents

1. [Overview](#1-overview)
2. [The Three-Layer Storage Model](#2-the-three-layer-storage-model)
3. [Write Paths](#3-write-paths)
   - 3.1 [Manual Write](#31-manual-write)
   - 3.2 [extractMemories — Per-Turn Capture](#32-extractmemories--per-turn-capture)
   - 3.3 [autoDream — Background Consolidation](#33-autodream--background-consolidation)
4. [Read Paths](#4-read-paths)
   - 4.1 [System Prompt Injection (Layer 1)](#41-system-prompt-injection-layer-1)
   - 4.2 [FileReadTool — On-Demand Loading (Layer 2)](#42-filereadtool--on-demand-loading-layer-2)
   - 4.3 [Targeted Grep — Session Transcripts (Layer 3)](#43-targeted-grep--session-transcripts-layer-3)
5. [Memory Types Taxonomy](#5-memory-types-taxonomy)
6. [File System Layout](#6-file-system-layout)
7. [Core Modules — Code Deep-Dive](#7-core-modules--code-deep-dive)
   - 7.1 [memdir/paths.ts — Path Resolution & Security](#71-memdirpathsts--path-resolution--security)
   - 7.2 [memdir/memdir.ts — Entry-Point & Truncation](#72-memdirmemirts--entry-point--truncation)
   - 7.3 [memdir/memoryTypes.ts — Type Taxonomy](#73-memdirmemorytypests--type-taxonomy)
   - 7.4 [memdir/memoryScan.ts — Directory Scanning](#74-memdirmemoryscansts--directory-scanning)
   - 7.5 [memdir/findRelevantMemories.ts — Recall Selector](#75-memdirfindrelevantmemorits--recall-selector)
   - 7.6 [services/extractMemories/](#76-servicesextractmemories)
   - 7.7 [services/autoDream/](#77-servicesautodream)
8. [autoDream — 4-Phase Consolidation Cycle](#8-autodream--4-phase-consolidation-cycle)
9. [ConsolidationLock — Race Protection](#9-consolidationlock--race-protection)
10. [Tool Permission Model](#10-tool-permission-model)
11. [Feature Flags & Configuration](#11-feature-flags--configuration)
12. [Team Memory (TEAMMEM)](#12-team-memory-teammem)
13. [KAIROS / Assistant Mode](#13-kairos--assistant-mode)
14. [Memory Lifecycle — End-to-End Flow](#14-memory-lifecycle--end-to-end-flow)
15. [Architecture Diagram Reference](#15-architecture-diagram-reference)
16. [BUDDY — A Tamagotchi-Style AI Pet](#16-buddy--a-tamagotchi-style-ai-pet)
17. [KAIROS — Persistent Assistant Mode (The "Always-On Claude")](#17-kairos--persistent-assistant-mode-the-always-on-claude)
18. [ULTRAPLAN — 30-Minute Remote Planning Sessions](#18-ultraplan--30-minute-remote-planning-sessions)
19. [Coordinator Mode — Multi-Agent Orchestrator](#19-coordinator-mode--multi-agent-orchestrator)
20. [References](#20-references)

---

## 1. Overview

Claude Code's memory system solves a fundamental problem with AI coding assistants: **context amnesia**. Without persistence, every new session starts from zero — you re-explain your stack, your preferences, your in-flight decisions.

The system is built around three key ideas:

| Idea | Implementation |
|------|---------------|
| **Layered storage** | Three tiers: always-loaded index → on-demand topic files → grepped transcripts |
| **Dual write paths** | Human-triggered (manual/`/remember`) + AI-triggered (background extraction & consolidation) |
| **Self-healing** | `autoDream` prunes stale entries, resolves contradictions, converts relative dates — without user action |

Everything is **local-first**: plain Markdown files in `~/.claude/projects/<project>/memory/`. No cloud dependency, no black-box embeddings, fully auditable with any text editor.

---

## 2. The Three-Layer Storage Model

```
~/.claude/projects/<git-root-slug>/memory/
│
├── MEMORY.md          ← Layer 1: Index — ALWAYS injected into system prompt
│                         (max 200 lines / ~25 KB; truncated with warning if exceeded)
│
├── user_profile.md    ← Layer 2: Topic files — loaded on-demand by FileReadTool
├── feedback_tests.md     when the model judges them relevant to the current query
├── project_auth.md
├── reference_linear.md
│
└── .jsonl transcripts  ← Layer 3: Session transcripts — NEVER fully read;
    (in project dir)      only grepped with narrow search terms as a last resort
```

### Why Three Layers?

- **Layer 1** needs to be fast and always present — the model must orient instantly. Keeping it ≤ 200 lines prevents token waste.
- **Layer 2** handles depth. Topic files can be long and detailed; they're only loaded when relevant.
- **Layer 3** is the "cold storage" for anything that didn't get promoted. Grepping is cheap; reading whole JSONL files is not.

---

## 3. Write Paths

There are three ways memories get written, each with a different trigger:

```
┌─────────────────────┐     ┌──────────────────────────────────────┐
│  Manual Write       │────▶│            MEMORY.md                 │
│  (user/slash cmd)   │     │     Layer 1 — always in context      │
└─────────────────────┘     └──────────────┬───────────────────────┘
                                            │
┌─────────────────────┐     ┌──────────────▼───────────────────────┐
│  extractMemories    │────▶│          Topic files (*.md)           │
│  (per-turn, async)  │     │     Layer 2 — loaded on-demand       │
└─────────────────────┘     └──────────────────────────────────────┘
         ╎ (async /
┌────────╎────────────┐     ┌──────────────────────────────────────┐
│  autoDream          │     │       Session transcripts (.jsonl)   │
│  (background, 24h+) │────▶│     Layer 3 — grep only             │
└─────────────────────┘     └──────────────────────────────────────┘
         │
         └── consolidationLock.ts (race protection)
```

---

### 3.1 Manual Write

**Trigger:** User explicitly asks Claude to remember something, or runs `/remember`.

**Behavior:**
- Claude writes immediately and synchronously (immediate write path, solid line in the diagram).
- Creates or updates a topic file in the memory directory.
- Updates `MEMORY.md` index pointer.

**Characteristics:**
- Highest priority — user intent is explicit.
- Claude is instructed: *"If the user explicitly asks you to remember something, save it immediately as whichever type fits best."*

---

### 3.2 `extractMemories` — Per-Turn Capture

**Source:** `services/extractMemories/extractMemories.ts`  
**Trigger:** End of each complete query loop (model produces final response with no tool calls).  
**Pattern:** Async / background (dashed line in diagram).

This is the **passive, always-on** memory capture system. After every assistant response, a forked sub-agent silently reviews the conversation and decides what's worth keeping.

#### How it works

1. **Gate check** — skips if: remote mode, auto-memory disabled, feature flag `tengu_passport_quail` is off, or if the main agent already wrote to memory files this turn (`hasMemoryWritesSince`).
2. **Cursor advancement** — tracks `lastMemoryMessageUuid` so each run only scans *new* messages since the last extraction (not the entire history).
3. **Manifest pre-injection** — scans existing memory files and injects their manifest into the prompt so the sub-agent doesn't waste a turn on `ls`.
4. **Forked agent** — runs `runForkedAgent` with a **5-turn cap** (read → write in 2–4 turns normally).
5. **Tool restrictions** — can only read files freely; can only write within the memory directory.
6. **Coalescing** — if an extraction is already in progress when a new one is triggered, the new context is "stashed" and runs as a trailing extraction after the current one finishes.

#### Throttle mechanism

A feature flag (`tengu_bramble_lintel`) can increase the minimum eligible turns between extractions. Default is 1 (every eligible turn). This is a GrowthBook-controlled knob for gradual rollout.

#### Mutual exclusion with main agent

```
Main agent writes memory → extractMemories detects this via hasMemoryWritesSince()
                         → skips the forked agent for that turn
                         → advances cursor past the main agent's writes
```

This prevents redundant double-writes where both the main agent and the extractor save the same information.

#### Turn budget strategy (from source comments)

> *"The efficient strategy is: turn 1 — issue all FileRead calls in parallel for every file you might update; turn 2 — issue all Write/Edit calls in parallel. Do not interleave reads and writes across multiple turns."*

---

### 3.3 `autoDream` — Background Consolidation

**Source:** `services/autoDream/autoDream.ts`  
**Trigger:** Fires automatically via `stopHooks` when **both** gates pass:
- ≥ 24 hours since last consolidation (`minHours`, configurable via GrowthBook)
- ≥ 5 new sessions accumulated since last consolidation (`minSessions`, configurable)

**Pattern:** Async / background (dashed line in diagram), runs as a **forked sub-agent**.

> **Neuroscience analogy:** Just like the brain consolidates memories during REM sleep — replaying, strengthening, and pruning — `autoDream` does the same for Claude's memory files while you're not actively using it.

#### Gate evaluation order (cheapest first)

```
1. Feature/mode gates (O(1))   → KAIROS off? remote mode? auto-memory disabled?
2. Time gate (1 stat call)     → hours since lastConsolidatedAt >= minHours?
3. Scan throttle (in-memory)   → has it been < 10 min since last scan?
4. Session gate (readdir)      → sessions since lastConsolidatedAt >= minSessions?
5. Lock acquire (write + read) → no other process mid-consolidation?
```

This ordering minimizes I/O: the cheapest checks run first, and expensive operations only happen when earlier gates pass.

---

## 4. Read Paths

### 4.1 System Prompt Injection (Layer 1)

`MEMORY.md` is **always** loaded into the system prompt at session start.

- Injected via `loadMemoryPrompt()` in `memdir/memdir.ts`.
- Hard limits: **200 lines** AND **~25 KB** — both caps are enforced sequentially (line cap first, then byte cap if still large).
- If truncated, a warning is appended naming which cap fired and by how much.
- Line truncation happens first (natural boundary), then byte truncation at the last newline before the cap.

```
WARNING: MEMORY.md is 247 lines (limit: 200). Only part of it was loaded.
Keep index entries to one line under ~200 chars; move detail into topic files.
```

**Why both a line cap AND a byte cap?**  
The line cap is the normal guard. The byte cap catches a pathological case: an index with lines that are individually ≤ 200 chars but have extreme total size (observed up to 197 KB in production under 200 lines at p100).

---

### 4.2 FileReadTool — On-Demand Loading (Layer 2)

Topic files (any `*.md` other than `MEMORY.md`) are **never** pre-loaded. Instead:

1. `findRelevantMemories()` is called with the current user query.
2. It scans the memory directory for all `.md` files and reads their frontmatter (first 30 lines max).
3. A **Sonnet side-query** (`sideQuery`) evaluates the manifest and returns the ≤ 5 most relevant filenames.
4. Those files are injected into the conversation via `FileReadTool`.

**Selector system prompt:**
> *"Only include memories that you are certain will be helpful based on their name and description. Be selective and discerning. If you are unsure, do not include it."*

**Tool-noise suppression:** If the model is currently exercising a tool (e.g., `mcp__X__spawn`), reference documentation for that tool is filtered from candidates — it's noise when the model is already using it successfully. However, *warnings and gotchas* about that tool are still included ("active use is exactly when those matter").

`alreadySurfaced` set prevents re-selecting files shown in prior turns, ensuring the 5-slot budget targets fresh candidates.

---

### 4.3 Targeted Grep — Session Transcripts (Layer 3)

Session transcripts (`.jsonl` files in the project directory) are a **last resort** — they're large, slow to read, and contain everything including noise.

**Access pattern:** grep only, with narrow search terms:
```bash
grep -rn "<narrow term>" ~/.claude/projects/<slug>/ --include="*.jsonl" | tail -50
```

**Use cases:**
- Looking for a specific error message from a past session
- Reconstructing context for a decision that wasn't captured in memory files
- Archaeology on "what was the exact command that fixed X?"

The system prompt explicitly discourages exhaustive transcript reads:
> *"Use narrow search terms (error messages, file paths, function names) rather than broad keywords."*

---

## 5. Memory Types Taxonomy

The system enforces a **closed four-type taxonomy**. Every memory file must declare one of these types in its frontmatter.

| Type | Scope | What to store | When to save |
|------|-------|---------------|--------------|
| `user` | Always private | Role, skills, goals, preferences, knowledge level | When you learn details about who the user is |
| `feedback` | Private (default); team if project-wide convention | Corrections AND confirmations — both what to avoid and what to keep doing | When user corrects approach OR confirms a non-obvious approach worked |
| `project` | Team (strongly biased) | Ongoing work, goals, initiatives, bugs, deadlines, decisions | When you learn who is doing what, why, or by when |
| `reference` | Usually team | Pointers to external systems (Linear, Grafana, Slack, dashboards) | When you learn about external resources and their purpose |

### Why this taxonomy?

The types map to **volatility** and **scope**:
- `user` memories are personal and stable (preferences change slowly).
- `feedback` memories are the most operationally important — they prevent repeated mistakes.
- `project` memories decay fast (deadlines pass, bugs get fixed) — the `why` field is critical to judge staleness.
- `reference` memories are durable pointers — external systems don't often move.

### Memory File Format

Every topic file uses YAML frontmatter:

```markdown
---
name: Integration Tests Must Use Real DB
description: Test policy requiring real database connections — not mocks
type: feedback
---

Integration tests must hit a real database, not mocks.

**Why:** Prior incident where mock/prod divergence masked a broken migration — mocked tests passed but the prod migration failed.

**How to apply:** Any time writing or reviewing test code that touches the database layer; block any PR that introduces mock DB in integration tests.
```

**Body structure for `feedback` and `project` types:**
- Lead with the rule/fact itself
- `**Why:**` — the reason the user gave (incident, constraint, preference)
- `**How to apply:**` — when/where this guidance kicks in

Knowing *why* lets the model judge edge cases instead of blindly applying rules.

---

## 6. File System Layout

```
~/.claude/
├── CLAUDE.md                          # Global instructions (every session)
│
└── projects/
    └── <sanitized-git-root>/          # e.g., "home-user-projects-myapp"
        │                              # (canonical git root, so all worktrees share one dir)
        │
        ├── *.jsonl                    # Session transcripts (Layer 3)
        │
        └── memory/                    # Auto-memory directory (Layer 1 + 2)
            ├── .consolidate-lock      # Lock file: mtime = lastConsolidatedAt, body = PID
            ├── MEMORY.md              # Index (Layer 1 — always loaded)
            │
            ├── user_profile.md        # Topic files (Layer 2 — on-demand)
            ├── feedback_testing.md
            ├── project_auth_rewrite.md
            ├── reference_linear.md
            │
            └── logs/                  # KAIROS / assistant mode only
                └── YYYY/
                    └── MM/
                        └── YYYY-MM-DD.md   # Append-only daily logs
```

### Path Resolution

`getAutoMemPath()` resolves in this priority order:

1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` env var (Cowork/SDK explicit override)
2. `autoMemoryDirectory` in `settings.json` (trusted sources: policy / local / user — **not** projectSettings for security)
3. Default: `<memoryBase>/projects/<sanitized-git-root>/memory/`

**Why git root for the slug?**  
`findCanonicalGitRoot()` ensures all worktrees of the same repo share one memory directory. Without this, creating a worktree at a different path would produce a different project slug and lose all context.

**Security constraints on path overrides:**
- `projectSettings` (committed `.claude/settings.json`) is explicitly excluded — a malicious repo could otherwise redirect writes to `~/.ssh` via the filesystem write carve-out.
- Paths are validated against: not relative, not root/near-root, not Windows drive-root, not UNC, no null bytes.

---

## 7. Core Modules — Code Deep-Dive

### 7.1 `memdir/paths.ts` — Path Resolution & Security

**Responsibilities:**
- `isAutoMemoryEnabled()` — master kill-switch with priority chain
- `getAutoMemPath()` — memoized path resolution (keyed on project root)
- `isAutoMemPath()` — security check for write permission gating
- `getAutoMemDailyLogPath()` — KAIROS daily log path

**Enable/disable priority chain (first wins):**
```
CLAUDE_CODE_DISABLE_AUTO_MEMORY env var
  → CLAUDE_CODE_SIMPLE (--bare mode) 
    → CCR without CLAUDE_CODE_REMOTE_MEMORY_DIR
      → settings.json autoMemoryEnabled
        → Default: enabled
```

**Memoization rationale:** `getAutoMemPath()` is called on every tool-use message render in the UI (`collapseReadSearchGroups → isAutoManagedMemoryFile`). Without memoization, each call would hit `getSettingsForSource × 4 → parseSettingsFile (realpathSync + readFileSync)`.

---

### 7.2 `memdir/memdir.ts` — Entry-Point & Truncation

**Key exports:**
- `ENTRYPOINT_NAME = 'MEMORY.md'`
- `MAX_ENTRYPOINT_LINES = 200`
- `MAX_ENTRYPOINT_BYTES = 25_000`
- `truncateEntrypointContent()` — dual-cap truncation with human-readable warning
- `buildMemoryLines()` — constructs behavioral instructions for the system prompt
- `buildMemoryPrompt()` — lines + current `MEMORY.md` content (used by agent memory)
- `loadMemoryPrompt()` — top-level dispatcher: routes to auto/team/KAIROS variants
- `ensureMemoryDirExists()` — idempotent mkdir, called once per session

**`truncateEntrypointContent()` — how dual-cap works:**
```
1. Split content into lines
2. Check line count (wasLineTruncated = count > 200)
3. Check byte count against ORIGINAL content (wasByteTruncated = bytes > 25_000)
   └── Uses original, not post-line-truncation, to correctly warn about long-line attacks
4. If line-truncated: slice to 200 lines first
5. If still over byte cap: cut at last newline before 25_000
6. Append warning naming which cap(s) fired
```

---

### 7.3 `memdir/memoryTypes.ts` — Type Taxonomy

Defines the four memory types and exports three key prompt sections:
- `TYPES_SECTION_INDIVIDUAL` — for solo auto-memory (no team)
- `TYPES_SECTION_COMBINED` — for auto + team memory (includes `<scope>` tags)
- `WHAT_NOT_TO_SAVE_SECTION` — explicit exclusions (see §17)
- `WHEN_TO_ACCESS_SECTION` — recall trigger rules + drift caveat
- `TRUSTING_RECALL_SECTION` — "before recommending from memory, verify existence"
- `MEMORY_FRONTMATTER_EXAMPLE` — standard frontmatter template

**"Before recommending from memory" section** exists because of a specific failure mode: a memory naming a function, file, or flag is a claim it existed *when written* — it may have been renamed, removed, or never merged. Claude is instructed to verify before recommending.

---

### 7.4 `memdir/memoryScan.ts` — Directory Scanning

**`scanMemoryFiles(memoryDir, signal)`**
- Recursively reads the memory directory for `.md` files (excludes `MEMORY.md`).
- Reads only the first **30 lines** of each file (frontmatter region).
- Parses frontmatter for `name`, `description`, `type`.
- Returns headers sorted newest-first, capped at **200 files**.
- Uses `Promise.allSettled` so a single unreadable file doesn't abort the scan.

**Single-pass design:** `readFileInRange` internally stats the file (returns `mtimeMs`), so the scan does one syscall per file instead of stat-then-read. For N ≤ 200 this halves syscall count vs a separate stat round.

**`formatMemoryManifest(memories)`**  
Produces one-line summaries for the recall selector prompt:
```
- [user] user_profile.md (2026-03-15T10:22:00.000Z): senior backend dev, Go expert, React beginner
- [feedback] feedback_testing.md (2026-03-14T08:11:00.000Z): no DB mocks in integration tests
```

---

### 7.5 `memdir/findRelevantMemories.ts` — Recall Selector

**`findRelevantMemories(query, memoryDir, signal, recentTools, alreadySurfaced)`**

1. Calls `scanMemoryFiles` to build the manifest.
2. Filters already-surfaced files (prevents re-selecting what the model already has).
3. Sends a **Sonnet side-query** with the manifest + current query.
4. Returns up to 5 relevant file paths with their `mtimeMs` (for freshness signaling).

**Output format (JSON schema enforced):**
```json
{ "selected_memories": ["user_profile.md", "feedback_testing.md"] }
```

**Why Sonnet for selection?**  
The selection query is a semantic relevance task — keyword matching alone fails for "explain the auth flow" → `project_auth_rewrite.md`. Using a model allows natural-language description matching. Sonnet is used (not Opus) since this is a fast, cheap routing decision.

---

### 7.6 `services/extractMemories/`

#### `extractMemories.ts`

The per-turn memory extraction engine. Key internal state (closure-scoped, reset per `initExtractMemories()` call):

| State | Purpose |
|-------|---------|
| `lastMemoryMessageUuid` | Cursor — only process messages after this point |
| `inProgress` | Mutex — prevents overlapping extraction runs |
| `pendingContext` | Stash — holds the latest context while a run is in progress |
| `turnsSinceLastExtraction` | Throttle counter — configurable via `tengu_bramble_lintel` |
| `inFlightExtractions` | Set of live promises — for `drainPendingExtraction()` |

**Why closure-scoped state?**  
Tests call `initExtractMemories()` in `beforeEach` for a fresh closure, eliminating shared-state pollution between tests. This pattern (also used by `confidenceRating.ts`) makes the module trivially testable without mocking module-level variables.

**`drainPendingExtraction(timeoutMs?)`**  
Called by `print.ts` after the response is flushed but before graceful shutdown. Ensures in-flight forked agents complete within the 5-second shutdown failsafe. Default timeout: 60 seconds.

#### `prompts.ts`

Builds the extraction prompt injected into the forked sub-agent. Two variants:
- `buildExtractAutoOnlyPrompt` — individual memory, no scope guidance
- `buildExtractCombinedPrompt` — with `<scope>` tags for private/team routing

**Efficiency instruction baked into prompt:**
> *"Turn 1 — issue all FileRead calls in parallel. Turn 2 — issue all Write/Edit calls in parallel. Do not interleave reads and writes."*

This cuts extraction from potential 6–8 turns to 2 turns for the common case.

---

### 7.7 `services/autoDream/`

#### `autoDream.ts`

The background consolidation engine. Key functions:

| Function | Role |
|----------|------|
| `initAutoDream()` | Creates the closure; call at startup or in test `beforeEach` |
| `executeAutoDream(context, appendSystemMessage?)` | Entry point from `stopHooks`; no-op until init |
| `makeDreamProgressWatcher(taskId, setAppState)` | Watches forked agent messages; extracts text + touched file paths for the task UI |

**`makeDreamProgressWatcher` — what it captures:**
- Text blocks from assistant messages → displayed in the background task panel
- `FileEdit`/`FileWrite` tool_use blocks → collects `file_path` for phase-flip and the inline completion message ("Improved N memory files")

#### `consolidationPrompt.ts`

Builds the 4-phase Dream prompt. Intentionally extracted from `dream.ts` (which is behind a `feature('KAIROS')` gate) so that `autoDream` can ship independently without those feature flag dependencies.

#### `consolidationLock.ts`

See §9 for the full lock mechanism description.

#### `config.ts`

Minimal-import leaf module. Intentionally kept with few imports so UI components can read `isAutoDreamEnabled()` without pulling in the forked agent / task registry / message builder chain.

```typescript
export function isAutoDreamEnabled(): boolean {
  const setting = getInitialSettings().autoDreamEnabled
  if (setting !== undefined) return setting
  const gb = getFeatureValue_CACHED_MAY_BE_STALE('tengu_onyx_plover', null)
  return gb?.enabled === true
}
```

---

## 8. autoDream — 4-Phase Consolidation Cycle

The Dream prompt instructs the consolidation sub-agent to follow four phases, explicitly modeled after biological sleep-based memory consolidation:

```
┌─────────────────────────────────────────────────────────────┐
│  Phase 1: Orient                                            │
│  • ls the memory directory                                  │
│  • Read MEMORY.md — understand the current index           │
│  • Skim existing topic files (avoid creating duplicates)   │
│  • If logs/ exists (KAIROS mode): review recent entries    │
└─────────────────────────────┬───────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────┐
│  Phase 2: Gather Signal                                     │
│  Sources in priority order:                                 │
│  1. Daily logs (logs/YYYY/MM/YYYY-MM-DD.md)                │
│  2. Existing memories that drifted (contradicted by code)  │
│  3. Transcript search (grep JSONL, narrow terms only)      │
│                                                             │
│  "Don't exhaustively read transcripts. Look only for       │
│   things you already suspect matter."                       │
└─────────────────────────────┬───────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────┐
│  Phase 3: Consolidate                                       │
│  • Merge new signal into existing topic files               │
│  • Convert relative dates → absolute dates                  │
│    ("yesterday" → "2026-03-15")                            │
│  • Delete contradicted facts — fix at the source           │
│  • Do NOT create near-duplicate files                       │
└─────────────────────────────┬───────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────┐
│  Phase 4: Prune & Index                                     │
│  • Update MEMORY.md — keep under 200 lines AND ~25 KB      │
│  • Remove pointers to stale/superseded memories            │
│  • Shorten verbose index entries (content → topic file)    │
│  • Add pointers to newly important memories                │
│  • Resolve contradictions between files                    │
└─────────────────────────────────────────────────────────────┘
```

**Tool constraints during autoDream:**  
Bash is **read-only** (`ls`, `find`, `grep`, `cat`, `stat`, `wc`, `head`, `tail` only). Any write-capable shell command is denied. This is enforced via `createAutoMemCanUseTool()` and also stated explicitly in the prompt's `extra` field.

**Scheduling configuration (from `tengu_onyx_plover` GrowthBook flag):**

| Knob | Default | Effect |
|------|---------|--------|
| `minHours` | 24 | Minimum hours since last consolidation |
| `minSessions` | 5 | Minimum new sessions since last consolidation |

---

## 9. ConsolidationLock — Race Protection

**Source:** `services/autoDream/consolidationLock.ts`

The lock file serves double duty: it's both a mutex AND the `lastConsolidatedAt` timestamp.

```
~/.claude/projects/<slug>/memory/.consolidate-lock
  body = PID of the process currently consolidating
  mtime = timestamp of last completed consolidation
```

### Lock Protocol

```
Acquire:
  1. Read existing lock: mtime (= lastConsolidatedAt), body (= holder PID)
  2. If lock is recent (< 1 hour) AND holder PID is alive → blocked, return null
  3. If lock is stale OR holder is dead → reclaim
  4. Write own PID to file (mtime = now)
  5. Re-read to verify: if PID != own → lost a race, return null
  6. Return prior mtime (for rollback on failure)

On success:
  → Run the dream sub-agent

On failure / crash:
  → rollbackConsolidationLock(priorMtime): rewind mtime, clear PID body
  → Next process sees stale mtime → time-gate passes → retries

On user kill (abort signal):
  → DreamTask.kill() handles abort + rollback
  → autoDream catches abortController.signal.aborted → no double-rollback
```

### Why mtime IS the timestamp

Using the lock file's mtime as `lastConsolidatedAt` means:
- One file serves two purposes (no separate timestamp file to keep in sync)
- The lock's lifetime implicitly records when consolidation happened
- `readLastConsolidatedAt()` costs exactly **one `stat` call** per turn

### Stale lock detection

A lock is considered stale if it's older than **1 hour** even if the PID is still alive — this guards against PID reuse (a new process with the same PID that isn't actually the consolidation process).

---

## 10. Tool Permission Model

Both `extractMemories` and `autoDream` sub-agents use `createAutoMemCanUseTool()` from `extractMemories.ts` — a shared permission factory:

```
Tool                  | Permission
─────────────────────────────────────────────────────────────────
FileReadTool          | Always allowed (inherently read-only)
GrepTool              | Always allowed (inherently read-only)
GlobTool              | Always allowed (inherently read-only)
REPLTool              | Always allowed (inner calls re-checked)
BashTool (read-only)  | Allowed if isReadOnly() passes
                      | (ls, find, grep, cat, stat, wc, head, tail)
BashTool (write)      | DENIED
FileEditTool          | Allowed only if file_path is within memory dir
FileWriteTool         | Allowed only if file_path is within memory dir
All other tools       | DENIED (MCP, Agent, etc.)
```

**REPL mode handling:**  
When REPL mode is enabled (ant-default builds), primitive tools are hidden from the tool list and the model invokes them through REPL scripts. The permission check still fires for each inner primitive via `toolWrappers.ts createToolWrapper`, so security is preserved. Giving the fork a different tool list would break prompt cache sharing (tools are part of the cache key).

---

## 11. Feature Flags & Configuration

The memory system is controlled by a combination of GrowthBook feature flags (gradual rollout) and environment variables / settings.

### GrowthBook Flags

| Flag | Controls |
|------|---------|
| `tengu_passport_quail` | Master gate for `extractMemories` background extraction |
| `tengu_onyx_plover` | `autoDream` enable + `{minHours, minSessions}` scheduling config |
| `tengu_herring_clock` | Whether user is in the team memory cohort (for telemetry) |
| `tengu_coral_fern` | "Searching past context" section in memory prompt |
| `tengu_moth_copse` | `skipIndex` — skip `MEMORY.md` two-step save (write file only) |
| `tengu_bramble_lintel` | Minimum turns between extraction runs (throttle) |
| `tengu_slate_thimble` | Run extraction in non-interactive sessions |
| `MEMORY_SHAPE_TELEMETRY` | Log memory recall shape telemetry |

### Environment Variables

| Variable | Effect |
|----------|--------|
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | `1`/`true` → disable all memory; `0`/`false` → force enable |
| `CLAUDE_CODE_SIMPLE` | Disables memory (bare mode) |
| `CLAUDE_CODE_REMOTE` + no `CLAUDE_CODE_REMOTE_MEMORY_DIR` | Disables memory |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | Redirects memory to a custom path (CCR) |
| `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` | Full path override (Cowork/SDK) |
| `CLAUDE_COWORK_MEMORY_EXTRA_GUIDELINES` | Injects extra guidelines into memory prompt |

### `settings.json` Keys

| Key | Type | Effect |
|-----|------|--------|
| `autoMemoryEnabled` | boolean | Override auto-memory enable/disable |
| `autoDreamEnabled` | boolean | Override autoDream enable/disable |
| `autoMemoryDirectory` | string | Custom memory directory path (supports `~/`) |

**Security note on `autoMemoryDirectory`:** Only accepted from `policySettings`, `flagSettings`, `localSettings`, and `userSettings`. **Explicitly excluded from `projectSettings`** (committed `.claude/settings.json`) to prevent a malicious repo from redirecting writes to sensitive directories.

---

## 12. Team Memory (TEAMMEM)

When the `TEAMMEM` feature flag is enabled, Claude Code supports a **shared team memory directory** alongside the private individual memory directory.

### Structure with Team Memory

```
~/.claude/projects/<slug>/memory/
├── MEMORY.md              ← Private index (Layer 1 — individual)
├── user_profile.md        ← Private topic files
├── feedback_style.md
│
└── team/                  ← Shared team memory directory
    ├── MEMORY.md          ← Team index (also Layer 1)
    ├── feedback_testing.md ← Project-wide conventions
    ├── project_auth.md    ← Team project context
    └── reference_linear.md ← Shared external pointers
```

### Type Scope Routing (TEAMMEM mode)

```
user      → always private
feedback  → private by default; team if project-wide convention
project   → strongly bias toward team
reference → usually team
```

### Combined Prompt

`buildCombinedMemoryPrompt()` (in `teamMemPrompts.ts`) generates a single prompt section covering both directories. Each `<type>` block includes a `<scope>` tag with explicit routing guidance. Both `MEMORY.md` files are injected (both are Layer 1).

### Precondition

Team memory requires auto memory to be enabled — `isTeamMemoryEnabled()` gates on `isAutoMemoryEnabled()` first.

---

## 13. KAIROS / Assistant Mode

When `feature('KAIROS')` is enabled and `getKairosActive()` returns true, Claude Code is running in **assistant / perpetual session mode** (long-lived sessions rather than discrete conversations).

The memory model changes significantly:

### KAIROS Memory Model vs Standard

| Aspect | Standard Mode | KAIROS Mode |
|--------|--------------|-------------|
| Write target | `MEMORY.md` (live index) | `logs/YYYY/MM/YYYY-MM-DD.md` (append-only) |
| Write style | Update/replace | Append-only timestamped bullets |
| Index maintenance | Every write updates `MEMORY.md` | Nightly `/dream` skill distills logs → `MEMORY.md` |
| autoDream | Fires on time+session gates | Skipped (uses disk-skill dream instead) |
| Log path | N/A | `<autoMemPath>/logs/YYYY/MM/YYYY-MM-DD.md` |

### Why append-only for KAIROS?

Long-lived sessions have a different failure mode: concurrent writes from parallel sub-agents. Append-only eliminates write contention — any agent can safely append a bullet without reading the existing content first.

The nightly `/dream` skill (a bundled skill, not `autoDream`) then distills the accumulated daily logs into the structured topic file + `MEMORY.md` format.

**Prompt cache note:** The KAIROS daily-log prompt is cached and NOT invalidated on date rollover mid-session. The model derives the current date from a `date_change` attachment (appended at midnight) rather than the user-context message — this preserves the prompt cache prefix across midnight.

---

## 14. Memory Lifecycle — End-to-End Flow

Here is the complete flow from "user says something interesting" to "it's persisted and recalled in a future session":

```
Session N
─────────────────────────────────────────────────────────────
User: "don't mock the DB in integration tests — we got burned last quarter"
                    │
                    ▼
         [Main agent receives turn]
                    │
                    ├── Main agent may write memory itself
                    │   (has full save instructions in system prompt)
                    │
                    ▼
         [End of query loop — stopHooks fires]
                    │
                    ▼
         executeExtractMemories()
                    │
         ┌──────────┴──────────┐
         │ hasMemoryWritesSince? │
         └──────────┬──────────┘
             YES ──▶ skip, advance cursor
             NO  ──▶ run forked sub-agent
                    │
                    ▼
         [Forked sub-agent, max 5 turns]
          Turn 1: FileRead existing feedback_*.md files
          Turn 2: FileWrite feedback_testing.md (new/updated)
                  FileEdit MEMORY.md (add pointer)
                    │
                    ▼
         "Saved 1 memory" appears in conversation
         [cursor advances to latest message UUID]

─────────────────────────────────────────────────────────────
24+ hours and 5+ sessions later...
─────────────────────────────────────────────────────────────
[Session N+7, end of a turn]
         executeAutoDream()
                    │
         Time gate: ✓ (31 hours since last)
         Session gate: ✓ (8 new sessions)
         Lock: ✓ acquired
                    │
                    ▼
         [Dream sub-agent, 4 phases]
          Phase 1: ls + read MEMORY.md + skim topic files
          Phase 2: grep transcripts for recent context
          Phase 3: merge duplicates, fix relative dates,
                   resolve contradictions
          Phase 4: prune MEMORY.md, update index
                    │
                    ▼
         "Improved 3 memory files" appears
         Lock mtime updated = new lastConsolidatedAt

─────────────────────────────────────────────────────────────
Session N+8 (new session)
─────────────────────────────────────────────────────────────
[System prompt built]
         loadMemoryPrompt()
                    │
                    ▼
         MEMORY.md injected (Layer 1, always)
         "- [feedback] feedback_testing.md — no DB mocks in integration tests"
                    │
User: "can you write some tests for the auth service?"
                    │
         findRelevantMemories("write tests for auth service")
                    │
         Sonnet selector: picks feedback_testing.md, project_auth_rewrite.md
                    │
         Both files loaded via FileReadTool (Layer 2)
                    │
         Claude responds with test plan that:
         ✓ Uses real database (not mocks)
         ✓ Understands auth rewrite context
```

---

## 15. Architecture Diagram Reference

The image referenced in this project's context shows the complete write/read path architecture:

```
                    Write paths                        Read path
                    ───────────                        ─────────

┌──────────────┐                              ┌───────────────────┐
│ Manual write │──────────────────────────────│  System prompt    │
│ On user req  │      MEMORY.md (L1)  always──│  MEMORY.md inject │
└──────────────┘  ┌─────────────────────┐     └───────────────────┘
                  │  ~/.claude/projects │
┌──────────────┐  │  .../memory/        │     ┌───────────────────┐
│  autoDream   │──│                     │─────│  FileReadTool     │
│  Background  │  │  Topic files (*.md) │  if │  Topic files      │
│  consolidat. │  │  (L2 — on-demand)  │relev│  on-demand        │
└──────────────┘  │                     │     └───────────────────┘
       ╎╎         │  Session transcripts│
┌──────────────┐  │  (.jsonl) (L3)      │     ┌───────────────────┐
│ extractMems  │  │  grep only          │─────│  Targeted grep    │
│ Per-turn cap │  │                     │     │  Narrow terms only│
└──────────────┘  │  consolidationLock  │     └───────────────────┘
                  │  — race protection  │
                  └─────────────────────┘

── Immediate write (solid)
╎╎ Async / background (dashed)

autoDream phases — fires after >24h + >5 sessions, via forked subagent:
┌─────────┐   ┌──────────────┐   ┌─────────────┐   ┌──────────────┐
│ Phase 1 │──▶│   Phase 2    │──▶│   Phase 3   │──▶│   Phase 4   │
│ Orient  │   │ Gather signal│   │ Consolidate │   │ Prune+index │
└─────────┘   └──────────────┘   └─────────────┘   └──────────────┘
```

---

---

#  New & Hidden Features of Claude Code

> The following sections (16–19) document **undiscovered and unreleased features** found by reverse-engineering the Claude Code source tree under `claude/src`. These features range from gradual rollouts gated behind GrowthBook flags, to fully shipped features that were never publicly announced, to experimental modes accessible only via environment variables or internal flags.
>
> **Sources:** Discovered by analyzing `buddy/`, `coordinator/`, `services/autoDream/`, `memdir/`, `voice/`, `skills/bundled/`.

| # | Feature | Status | Access |
|---|---------|--------|--------|
| 16 | **BUDDY** — Tamagotchi-style AI pet | Shipped (May 2026) | All users |
| 17 | **KAIROS** — Always-on persistent assistant | Internal / rolling out | Anthropic internal + flag |
| 18 | **ULTRAPLAN** — Remote 30-min planning sessions | Behind GrowthBook flag | `tengu_ultraplan_model` |
| 19 | **Coordinator Mode** — Multi-agent orchestrator | Partial access | `CLAUDE_CODE_COORDINATOR_MODE=1` |

---

## 16. BUDDY — A Tamagotchi-Style AI Pet

> **Origin:** Started as an April Fools' joke (teaser April 1–7, 2026), then shipped as a real feature in May 2026. Always enabled for Anthropic employees.

Every Claude Code user gets a **deterministic generative companion** — a small animated sprite that lives beside the input box in a speech bubble. No two users with different accounts ever get the same buddy, because the entire creature is derived from a hash of your `userId`.

---

### 16.1 How Your Buddy Is Born — Deterministic Generation

The generation pipeline uses a **Mulberry32 seeded PRNG** seeded from your `userId` hash. Because the seed is deterministic, your buddy is always the same across devices and reinstalls — it is your buddy, not a random one per session.

**Step-by-step generation:**

```
userId  ──hash──▶  Mulberry32 seed
                        │
          ┌─────────────┼─────────────────────┐
          ▼             ▼                      ▼
       species        rarity               cosmetics
    (1 of 18)       roll (0–1)        eyes + hat + shiny
```

---

### 16.2 Species

18 possible species, each with distinct sprite art:

| Group | Species |
|-------|---------|
| Birds | duck, goose, owl, penguin |
| Mythical / Sci-Fi | dragon, ghost, robot |
| Animals | cat, turtle, snail, axolotl, capybara, rabbit |
| Abstract / Cute | blob, octopus, cactus, mushroom, chonk |

---

### 16.3 Rarity Tiers

The PRNG roll determines rarity, which gates cosmetic variety and stat floors:

| Tier | Probability | Effect |
|------|------------|--------|
| **Common** | 60% | Base stat floors |
| **Uncommon** | 25% | Slightly higher stat floors |
| **Rare** | 10% | Noticeably higher stats |
| **Epic** | 4% | High stat floors, rarer hats |
| **Legendary** | 1% | Maximum stat floors |
| ✨ **Shiny** | 1% (any tier) | Special visual treatment |

Rarity is a multiplier on stat floors, not a separate roll — a legendary buddy always has higher base stats than a common one of the same species.

---

### 16.4 Cosmetics

**6 eye styles:** `·` `+` `×` `●` `@` `°`

**8 hat options:** crown, tophat, propeller, halo, wizard, beanie, tinyduck, none

Both are drawn from the PRNG roll after species and rarity are settled. Higher rarity tiers unlock more cosmetic combinations.

---

### 16.5 Stats

Every buddy has 5 named stats whose values scale with rarity (higher rarity = higher floor):

| Stat | Flavour |
|------|---------|
| `DEBUGGING` | How good your buddy is at finding bugs |
| `PATIENCE` | How long your buddy waits before fidgeting |
| `CHAOS` | Unpredictability of idle behaviour |
| `WISDOM` | Quality of unsolicited advice |
| `SNARK` | Sharpness of speech-bubble commentary |

Stats are cosmetic / narrative — they influence animation behaviour and speech-bubble flavour text, not Claude's actual outputs.

---

### 16.6 Soul Generation — Name & Personality

On a user's **first hatch** (first time the buddy renders), Claude runs a one-shot generation to produce:
- A **name** for the buddy
- A **personality** description

Both are stored permanently and **never regenerated**. Even if Claude Code is reinstalled, the soul persists. This gives the companion genuine continuity — it is the same creature with the same name across all future sessions.

---

### 16.7 Animations & Interactions

| Animation | Details |
|-----------|---------|
| Sprite tick | 500ms frame interval |
| Idle fidgets | Random small movements when Claude is not busy |
| Rare blinks | Low-probability eye-close frames |
| Speech bubbles | 10s display duration, 3s fade-out |
| `/buddy pet` command | Triggers a floating ❤️ heart animation lasting 2.5s |

Speech bubbles contain personality-driven commentary from the buddy — tone and content are shaped by its generated personality and SNARK stat.

---

### 16.8 Release Timeline

| Window | Status |
|--------|--------|
| April 1–7, 2026 | Teaser (rolling 24h wave across timezones — not everyone saw it simultaneously) |
| May 2026 | Full public launch |
| Always | Enabled for all Anthropic employees regardless of rollout gate |

---

## 17. KAIROS — Persistent Assistant Mode (The "Always-On Claude")

> **What it is:** A complete alternate UX mode where Claude stops being a turn-based assistant and becomes a **long-lived autonomous agent** that persists across sessions, self-manages its memory, and can act proactively without being prompted.

KAIROS (referenced throughout the codebase as `feature('KAIROS')` and `getKairosActive()`) fundamentally changes how Claude Code operates. Rather than discrete conversations, you get a single ambient assistant that is always running, always accumulating context, and capable of working in the background.

> Note: Section 13 of this document covers KAIROS from a memory-architecture perspective. This section covers the **full UX and operational model**.

---

### 17.1 The Core Shift — From Turns to Persistence

| Dimension | Standard Mode | KAIROS Mode |
|-----------|--------------|-------------|
| Session model | Discrete conversations | Single long-lived session |
| Memory writes | Topic files via two-step index | Append-only daily logs |
| Memory distillation | autoDream (time/session gates) | Nightly `/dream` skill (scheduled) |
| Agent posture | Reactive (responds to user) | Reactive + Proactive (can initiate) |
| Output format | Raw console text | Structured via `SendUserFile` tool |
| Long commands | Block until complete | Auto-backgrounded after 15s |
| Status | N/A | `normal` (replying) or `proactive` (initiating) |

---

### 17.2 Append-Only Daily Log Memory

Instead of maintaining `MEMORY.md` as a live index that gets updated every turn, KAIROS uses **append-only daily log files**:

```
~/.claude/projects/<slug>/memory/logs/
└── YYYY/
    └── MM/
        └── YYYY-MM-DD.md    ← append-only, timestamped bullets
```

**Why append-only?**
Long-lived sessions may have parallel sub-agents writing simultaneously. Append-only eliminates all write contention — any agent can safely add a bullet without reading the existing content. There is no risk of two agents overwriting each other's entries.

**What gets logged:**
- User corrections ("use bun, not npm")
- Decisions and their rationale
- Project context not in the codebase
- Pointers to external systems
- Anything the user explicitly asks to remember

Each entry is a short timestamped bullet written with the current date derived from `currentDate` in context.

---

### 17.3 Nightly Dreaming — Distillation via Forked Sub-Agent

The accumulated daily logs are distilled nightly by a **forked sub-agent** running the full 4-phase Dream cycle:

```
Orient ──▶ Gather Signal ──▶ Consolidate ──▶ Prune & Index
```

- Reads all recent log entries
- Merges them into structured topic files (same format as standard mode)
- Updates `MEMORY.md` as a clean index
- Uses **read-only Bash** during the dream pass — no write-capable shell commands are permitted, eliminating any risk of log corruption during consolidation

This means `MEMORY.md` in KAIROS is a *distilled product* of the logs, not a live index. The model reads logs to orient itself; it reads `MEMORY.md` only to understand what has already been distilled.

---

### 17.4 The 15-Second Blocking Budget

Any shell command or tool call that runs longer than **15 seconds** is automatically **backgrounded** as a sub-agent task. This keeps the main assistant thread responsive:

```
User sends message
      │
Claude executes tool/command
      │
      ├── Completes < 15s ──▶ Normal inline result
      │
      └── Exceeds 15s ──────▶ Auto-backgrounded to sub-agent
                              Main thread returns immediately
                              Sub-agent reports back when done
```

This is critical for KAIROS because the session never ends — blocking the main thread for a long build or test run would make the assistant unresponsive to new messages.

---

### 17.5 Proactive Mode — `<tick>` Prompts

KAIROS introduces a **proactive posture** where Claude doesn't just respond — it periodically decides what to do next on its own.

This is implemented via periodic `<tick>` prompts injected into the session. On each tick, Claude must choose one of two responses:
- **Do something** — take an action, run a command, write a file, update a memory
- **Call `SleepTool`** — explicitly signal that there is nothing to do right now and the assistant should wait

This creates a genuine ambient agent loop: the assistant is always "awake," periodically checking in, and capable of self-initiated work (monitoring a CI pipeline, checking for new PRs, updating memory based on code changes it detects).

---

### 17.6 Brief Mode — Structured Output via `SendUserFile`

In KAIROS, **all output goes through the `SendUserFile` tool** rather than being written as raw console text. The tool wraps output in a structured format:

```
SendUserFile({
  content: "...",         // structured markdown
  attachments: [...],     // optional file references
  status: "normal"        // or "proactive"
})
```

**Status field semantics:**
- `normal` — Claude is replying to something the user said
- `proactive` — Claude is initiating this message on its own (tick-driven)

This distinction lets the UI render proactive messages differently (e.g., a notification rather than a chat bubble) and lets downstream tooling filter by who initiated the exchange.

---

### 17.7 Exclusive KAIROS Tools

These tools are only available in KAIROS mode — they don't exist in the standard tool list:

| Tool | Purpose |
|------|---------|
| `SendUserFile` | Structured output delivery (replaces raw console text) |
| `PushNotification` | Sends a push notification to the user's device |
| `SubscribePR` | Subscribes to GitHub webhook events for a specific PR |
| `SleepTool` | Explicitly signals "nothing to do, go idle" during a tick |

`SubscribePR` is particularly powerful — it allows the always-on assistant to watch a pull request and automatically act when it receives new comments, CI results, or review requests without the user needing to check manually.

---

### 17.8 Midnight Boundary Handling

Because KAIROS sessions persist across days, the system must handle date rollover correctly:

- At midnight, **yesterday's transcript is flushed** to disk so the nightly dream process can find and process it
- The daily log file path rolls over to the new date (`YYYY-MM-DD.md`)
- The prompt cache prefix is intentionally **not** invalidated on date change — the model derives the current date from a `date_change` attachment appended at midnight, preserving the cache prefix across the rollover

This is a deliberate tradeoff: losing the cache prefix at midnight would significantly increase API costs for a session that may have been running for weeks.

---

## 18. ULTRAPLAN — 30-Minute Remote Planning Sessions

> **What it is:** An interactive planning system that **farms out complex exploration to a remote Claude Code instance (CCR)**, lets it plan autonomously, then presents the plan for user approval before executing — either in the cloud or teleported back to the local terminal.

ULTRAPLAN separates *planning* from *execution*, running the planning phase on a remote compute instance so that local sessions stay responsive and the full planning budget is available without affecting the user's active session.

---

### 18.1 High-Level Architecture

```
Local Claude Code (user's terminal)
         │
         │  spins up
         ▼
Remote Claude Code Instance (CCR)
         │  explores codebase, calls tools, builds plan
         │  (up to 30 minutes)
         │
         ▼  calls ExitPlanMode when ready
Local polls every 3s ──▶ detects ExitPlanMode
         │
         ▼
User reviews plan in browser (claude.ai)
         │
    ┌────┴─────────────────────────┐
    │                              │
    ▼                              ▼
┌──────────┐                 ┌──────────┐
│  Approve │                 │  Reject  │
└────┬─────┘                 └────┬─────┘
     │                            │
     │                            └──▶ loops back to Remote
     │                                 CCR for iteration
     ▼
┌─────────────────────┐
│  Two execution paths│
└──────────┬──────────┘
           │
     ┌─────┴──────────────────────────────────────────────┐
     │ "remote"              │ "teleport to terminal"      │
     │                       │                             │
     │ Execute in the cloud  │ Archive remote session,     │
     │ instance directly     │ then execute locally via    │
     │                       │ __ULTRAPLAN_TELEPORT_LOCAL__│
     │                       │ sentinel                    │
     └───────────────────────┴─────────────────────────────┘
```

---

### 18.2 The Planning Phase — What the Remote Instance Does

The remote Claude Code instance has full access to tools (read the codebase, run commands, search files) and runs autonomously for up to **30 minutes**. It is not given an execution mandate — its only job is to:

1. Explore the codebase and understand the problem
2. Research solutions, investigate edge cases
3. Produce a concrete, detailed plan
4. Call `ExitPlanMode` to signal it is ready for review

**Local polling:** The local instance polls the remote session every **3 seconds** to detect when `ExitPlanMode` has been called. This is a lightweight status check, not a data transfer — the full plan is fetched only when the remote signals readiness.

---

### 18.3 Plan Review — User Approval in the Browser

The plan is surfaced in **claude.ai** (the browser UI) for review. This is intentional: browser rendering gives the plan full markdown formatting, making long structured plans much easier to read than terminal output.

**Approval flow:**
- **Approve** → proceed to execution (local or remote)
- **Reject** → the remote instance receives the rejection (optionally with feedback) and loops back to revise and re-plan. This can iterate multiple times until the user is satisfied.

The loop means ULTRAPLAN is genuinely interactive even in the planning phase — it's not a one-shot proposal, it's a negotiation.

---

### 18.4 Execution Paths

Once the plan is approved, there are two execution paths:

#### Path A — Remote Execution
The remote CCR instance executes the approved plan in the cloud. Useful when:
- The task requires cloud-scale compute
- The user wants execution to continue even if they close their local terminal
- The codebase is already available in the remote environment

#### Path B — Teleport to Terminal
The remote session is **archived** (its full context and plan are serialized), then execution happens locally via the `__ULTRAPLAN_TELEPORT_LOCAL__` sentinel.

"Teleport" means: the local Claude Code instance picks up exactly where the remote one left off — same plan, same context — and executes it in the local terminal environment. This gives users the benefit of cloud-scale planning with local execution control.

---

### 18.5 Model & Configuration

| Setting | Value |
|---------|-------|
| Default model | **Opus 4.6** |
| Configurable via | `tengu_ultraplan_model` GrowthBook flag |
| Session duration limit | **30 minutes** |
| Poll interval | **3 seconds** |
| Plan review surface | **claude.ai** (browser) |

Opus 4.6 is chosen for ULTRAPLAN because planning-phase quality matters more than cost — the remote instance only runs once per plan, and its output gates everything that follows.

---

### 18.6 Why Remote Planning?

| Problem | ULTRAPLAN Solution |
|---------|--------------------|
| Complex planning blocks the local session | Planning runs on a remote instance — local stays responsive |
| Plans need deep exploration (30+ minutes of tool calls) | Remote instance has its own full budget |
| User can't review a wall of terminal output | Plan surfaces in a clean browser UI |
| Bad plans waste execution time | Rejection loop ensures the plan is right before any execution |
| Cloud vs local execution tradeoff | Teleport sentinel gives the user the choice at approval time |

---

## 19. Coordinator Mode — Multi-Agent Orchestrator

> **Access:** Already partially enabled via `CLAUDE_CODE_COORDINATOR_MODE=1` environment variable. Full feature is behind gradual rollout.

Coordinator Mode transforms Claude Code from a single-agent assistant into a **multi-agent orchestration layer**. A master Claude instance acts as an intelligent coordinator that spawns, directs, and synthesizes work from multiple parallel worker agents.

---

### 19.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                  COORDINATOR (Master Claude)             │
│                                                         │
│  - Plans and decomposes tasks                           │
│  - Spawns workers via AgentTool                         │
│  - Synthesizes results (never lazy-delegates)           │
│  - Receives push notifications from workers             │
└──────────────┬──────────────────────────────────────────┘
               │  spawns (multiple in parallel)
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐
│Worker 1│ │Worker 2│ │Worker 3│   ... (N workers)
│        │ │        │ │        │
│isolated│ │isolated│ │isolated│
│scratch │ │scratch │ │scratch │
│ dir    │ │ dir    │ │ dir    │
└────┬───┘ └────┬───┘ └────┬───┘
     │          │          │
     └──────────┴──────────┘
                │  push <task-notification> XML
                ▼
         Coordinator receives
         (never polls — push only)
```

---

### 19.2 How Workers Are Spawned — `AgentTool`

The coordinator spawns workers using `AgentTool`. Multiple workers can be spawned in a single coordinator turn, enabling genuine parallelism:

```
Coordinator turn:
  AgentTool(task="implement auth middleware", id="worker-1")
  AgentTool(task="write auth middleware tests", id="worker-2")
  AgentTool(task="update API documentation", id="worker-3")
                    ↓
  All three run simultaneously
```

Workers are full Claude instances with their own tool access, context, and execution budget. The coordinator does not micromanage them — it gives a task and waits.

---

### 19.3 Worker Reporting — XML `<task-notification>`

When a worker completes (or reaches a checkpoint), it reports back to the coordinator via a structured **XML notification message**:

```xml
<task-notification>
  <agent-id>worker-1</agent-id>
  <status>complete</status>
  <summary>Implemented JWT middleware with refresh token support</summary>
  <token-usage>12847</token-usage>
  <duration-ms>34200</duration-ms>
</task-notification>
```

**Fields:**

| Field | Content |
|-------|---------|
| `agent-id` | The worker's unique identifier |
| `status` | `complete`, `failed`, `blocked`, `checkpoint` |
| `summary` | Human-readable summary of what was done |
| `token-usage` | Tokens consumed by this worker (for budget tracking) |
| `duration-ms` | Wall-clock time the worker ran |

The coordinator receives these notifications and uses them to decide next steps: synthesize results, spawn follow-up workers, or report back to the user.

---

### 19.4 Push Model — Coordinator Never Polls

A key design principle: **the coordinator never polls workers**. Workers push their `<task-notification>` when ready.

**Why this matters:**
- Polling would require the coordinator to repeatedly check each worker's status, wasting tokens on no-op turns
- Push means the coordinator only wakes up when there is actual new information
- Multiple workers completing at different times each trigger their own notification — the coordinator processes them as they arrive
- The coordinator's context is not cluttered with "still running..." status checks

---

### 19.5 Continuing a Specific Worker — `SendMessage`

After a worker reports a `blocked` or `checkpoint` status, the coordinator can **continue that specific worker** using `SendMessage` with the worker's agent ID:

```
SendMessage(agent_id="worker-2", content="The auth tests should use the real DB, not mocks")
```

This allows the coordinator to:
- Unblock a worker that needs clarification
- Inject new requirements mid-task
- Redirect a worker that went off-course

Workers that receive a `SendMessage` resume from their current state — they don't restart from the beginning.

---

### 19.6 Isolated Scratch Directories

Each worker gets its own **isolated scratch directory** (gated behind `tengu_scratch` feature flag). This prevents workers from interfering with each other's intermediate work:

```
~/.claude/scratch/
├── worker-1-<uuid>/    ← Worker 1's private workspace
├── worker-2-<uuid>/    ← Worker 2's private workspace
└── worker-3-<uuid>/    ← Worker 3's private workspace
```

Workers can freely create, modify, and delete files in their scratch directory without affecting other workers or the main project. The coordinator decides what to promote to the actual project after reviewing worker outputs.

---

### 19.7 The Anti-Lazy-Delegation System Prompt

Coordinator Mode enforces a specific behavioral constraint via the system prompt:

> *"Never use lazy phrases like 'based on your findings' — synthesize before delegating."*

This prevents a failure mode where the coordinator becomes a pass-through that:
1. Delegates everything to workers
2. Returns their raw output to the user without adding value
3. Uses phrases like "based on the worker's findings, it seems that..."

The coordinator is expected to **genuinely synthesize** — combine outputs from multiple workers into a coherent, deduplicated, higher-quality result. It should understand what each worker produced and form its own conclusions, not just relay them.

---

### 19.8 Enabling Coordinator Mode

**Current access:**

```bash
export CLAUDE_CODE_COORDINATOR_MODE=1
claude
```

This enables the partial implementation available now. Full coordinator mode with all features is behind a gradual GrowthBook rollout.

**What `COORDINATOR_MODE=1` enables today:**
- Master Claude uses coordinator system prompt
- Workers are spawned via AgentTool
- XML task-notification parsing is active
- SendMessage routing by agent ID is active

**What is still gated:**
- Isolated scratch directories (`tengu_scratch` flag)
- Full parallel worker scheduling optimisations

---

### 19.9 Coordinator vs Standard Agent — Summary

| Dimension | Standard Agent | Coordinator Mode |
|-----------|---------------|-----------------|
| Agents | 1 | 1 master + N workers |
| Parallelism | Sequential tool calls | True parallel sub-agents |
| Worker communication | N/A | XML push notifications |
| Workspace isolation | Shared project dir | Per-worker scratch dirs |
| Synthesis obligation | N/A | Must synthesize, never relay |
| Best for | Focused single tasks | Complex multi-part tasks |

---

## 20. References

The following resources were used in the research and writing of this document. They are recommended for further reading and exploration of the Claude Code memory system.

| Resource | Description |
|----------|-------------|
| [Claude Code — Official Mintlify Docs](https://www.mintlify.com/VineeTagarwaL-code/claude-code) | The official generated documentation for the `VineeTagarwaL-code/claude-code` repository. Covers quickstart, installation, core concepts (memory, permissions, tools), guides (MCP servers, hooks, skills, multi-agent workflows), and CLI reference. Hosted on Mintlify. |
| [Claude Files Info — Source Explorer](https://t.co/sHCbfbN9pu) | A file-by-file explorer of every source file under `claude/src`. Provides summaries, exports, dependencies, and live source previews in a clean monospaced layout. Useful for navigating the codebase without cloning the repo. |

---
