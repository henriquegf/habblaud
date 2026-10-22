# Cursor adapter (observe-only) — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Show recent Cursor Agent chats from `~/.cursor/chats/` as agents in the Habblaud office (cwd → room, title, coarse status), without terminal, permissions, or messages.

**Architecture:** New `CursorSource` (`AgentSource`, `provider: 'cursor'`) polls `meta.json` (+ `store.db` mtime) with a 30‑minute presence window. Synthetic account `.cursor`. `Provider` gains `'cursor'`; client gets name/chip only. Folder layout leaves room for store.db (phase 2) and hooks (phase 3) later.

**Tech Stack:** Node 22+, TypeScript, Vitest, existing Office / SourceSet / AccountsService patterns (mirror Codex, smaller).

**Spec:** `docs/hrq/2026-10-15-cursor-adapter-design.md`  
**Branch:** `feat/cursor-adapter` from `main` (worktree recommended).

## Global Constraints

- MVP = observe only: no `transcriptPathOf`, no terminal parser, no `/api/cursor/events`, no hooks.
- Agent id: `.cursor:<chatId>` (directory name under `chats/<hash>/`).
- Presence: `updatedAtMs` OR `store.db` mtime within `CURSOR_RECENT_MS` (30 * 60_000).
- No `cwd` (missing/invalid) → do not add to Office.
- Subagents (`isSubagent: true`) → skip in MVP.
- `HABBLAUD_CURSOR=0` disables; without `chats/` directory, source does not start.
- Do not read `auth` / API keys; do not parse SQLite blob contents in MVP.
- Conventional Commits English; `npm test` + `npm run typecheck` before finish.
- Phases 2–3 out of scope (document in code comments only where a stub would invite wrong PRs — prefer no stubs).

## File map

| File | Responsibility |
| --- | --- |
| `shared/types.ts` | `Provider` includes `'cursor'` |
| `server/sources/cursor/files.ts` | Discover chats dirs; list/read `meta.json`; mtime |
| `server/sources/cursor/accounts.ts` | `discoverCursorDirs`, synthetic account |
| `server/sources/cursor/source.ts` | `CursorSource` |
| `server/sources/cursor/*.test.ts` | Unit + source tests with temp fixtures |
| `server/config.ts` | `cursor: boolean` |
| `server/index.ts` | Wire source + log line |
| `client/src/ui/provider.ts` (+ tests) | Name, chip, helpers |
| `client` other | Only compile fixes for exhaustive `Provider` switches |
| `README.md`, `server/README.md` | `HABBLAUD_CURSOR` |

---

### Task 1: Provider type + cursor file helpers

**Files:**
- Modify: `shared/types.ts` (`Provider`)
- Create: `server/sources/cursor/files.ts`
- Create: `server/sources/cursor/files.test.ts`

**Interfaces:**
- Produces:
  ```ts
  export type Provider = 'claude' | 'codex' | 'cursor';

  export const CURSOR_RECENT_MS = 30 * 60_000;
  /** Hot window for status `working` (vs idle while still present). */
  export const CURSOR_WORKING_MS = 5 * 60_000;

  export interface CursorChatMeta {
    chatId: string;
    /** Absolute path to chat directory (contains meta.json / store.db). */
    dir: string;
    title?: string;
    cwd?: string;
    updatedAtMs: number;
    isSubagent: boolean;
    storeMtimeMs?: number;
  }

  /** Root dirs that contain a `chats/` subdirectory (usually ~/.cursor). */
  export function discoverCursorRoots(env, home): string[];
  /** All chat metas under root/chats (skips unreadable). */
  export function listCursorChats(root: string, now?: number): CursorChatMeta[];
  export function isCursorPresent(meta: CursorChatMeta, now: number, recentMs?: number): boolean;
  export function isCursorWorking(meta: CursorChatMeta, now: number, workingMs?: number): boolean;
  ```
- `discoverCursorRoots`: `HABBLAUD_CURSOR_DIRS` / `HABBLAUD_CURSOR_DIR` override (comma list), else `join(home,'.cursor')` if it has `chats/` dir.
- Activity time for presence = `max(updatedAtMs, storeMtimeMs ?? 0)`.

- [ ] **Step 1: Failing tests** in `files.test.ts` with `tempDir()` from `server/test/fixtures.ts`:
  - root without chats → not discovered
  - meta with cwd + recent updatedAtMs → present + working
  - old updatedAtMs → not present
  - `isSubagent: true` still listed by `listCursorChats` (filter is source’s job) but fixture covered
  - missing cwd field preserved as undefined

- [ ] **Step 2:** `npx vitest run server/sources/cursor/files.test.ts` → FAIL

- [ ] **Step 3: Implement** `files.ts` + extend `Provider` in `shared/types.ts`. Fix any exhaustive-switch compile errors that break `tsc` in the same commit only if trivial; otherwise Task 3/4.

- [ ] **Step 4:** tests PASS

- [ ] **Step 5: Commit** `feat(cursor): add chat meta helpers and provider type`

---

### Task 2: CursorSource + accounts + config wire

**Files:**
- Create: `server/sources/cursor/accounts.ts`, `accounts.test.ts`
- Create: `server/sources/cursor/source.ts`, `source.test.ts`
- Modify: `server/config.ts` (`cursor` flag like `codex`)
- Modify: `server/index.ts` (add to SourceSet, log)
- Modify: `server/http/terminal.test.ts` or config tests only if `loadConfig` tests need `cursor: true` default

**Interfaces:**
- Consumes: Task 1 helpers
- Produces:
  ```ts
  export function detectCursorAccount(root: string): DetectedAccount; // id '.cursor', provider 'cursor', short 'R', name 'Cursor', color from palette
  export class CursorSource implements AgentSource {
    readonly provider = 'cursor';
    constructor(opts: { accounts: AccountsService; office: Office; roots: string[]; now?: () => number; pollMs?: number });
    start(): void;
    stop(): void;
    sources(): SourceInfo[];
    transcriptPathOf(): undefined;
    accountEntries(): ... // if Codex exposes similar; else register via accounts.setProviderAccounts in start
  }
  ```
- On tick: for each present non-subagent chat with cwd → `office.addMain` / update; missing → remove like Codex close grace (~1.5s).
- Role string: `'Agente principal (Cursor)'`.
- `sessionId` = `chatId`.
- Account id always `.cursor` (one account for MVP even if multiple roots — first root wins for account path; chats from all roots scanned).

**source.test.ts** (temp chats tree, fake Office or real Office like codex tests):
1. Recent chat → agent present with provider cursor, room path = cwd, title set.
2. After advancing `now` past recent window → agent gone.
3. Subagent meta → never added.
4. `transcriptPathOf(id)` → undefined.

- [ ] **Step 1:** Write failing source/accounts/config tests  
- [ ] **Step 2:** Run → FAIL  
- [ ] **Step 3:** Implement source + `config.cursor` (`!env.HABBLAUD_CURSOR?.trim() || isTruthy`, same as codex) + index wiring  
- [ ] **Step 4:** `npx vitest run server/sources/cursor` + related config tests → PASS  
- [ ] **Step 5: Commit** `feat(cursor): observe open chats in the office`

---

### Task 3: Client provider labeling

**Files:**
- Modify: `client/src/ui/provider.ts`, `provider.test.ts`
- Fix compile breaks: any `Record<Provider, …>` or switch missing `cursor` (grep `Provider` / `'codex'` exhaustiveness). Prefer Cursor to look like a third provider: name “Cursor”, short “R”, optional tag “Cursor” where Codex shows “Codex” — **do not** reuse Codex hollow-chip art unless trivial; text chip is enough for MVP.
- Avoid changing permission/composer paths except so Cursor agents don’t claim Codex UI (`isCodex` stays false).

- [ ] **Step 1:** Tests for `PROVIDER_NAME.cursor`, `providerOf`, short letter R for `.cursor`  
- [ ] **Step 2–4:** Implement + `npx vitest run client/src/ui/provider.test.ts` + `npm run typecheck`  
- [ ] **Step 5: Commit** `feat(cursor): show Cursor provider in the client`

---

### Task 4: Docs + full verification

**Files:**
- Modify: `README.md`, `server/README.md` — `HABBLAUD_CURSOR`, observe-only caveat, roadmap one-liner (terminal/hooks later)
- Modify: `docs/hrq/CONTEXTO.md` on `hrq` if touching that branch; on `feat/cursor-adapter` skip if file absent

- [ ] **Step 1:** Docs  
- [ ] **Step 2:** `npm test && npm run typecheck`  
- [ ] **Step 3:** Smoke checklist in report: run Habblaud against real `~/.cursor/chats` on LXC; confirm agent appears for an active chat  
- [ ] **Step 4: Commit** `docs(cursor): document HABBLAUD_CURSOR observe mode`

---

## Self-review (plan vs spec)

| Spec | Task |
| --- | --- |
| meta.json + mtime presence | 1–2 |
| Provider `'cursor'` | 1, 3 |
| Account `.cursor` | 2 |
| No terminal/hooks | 2 (`transcriptPathOf` undefined) |
| Client chip/name | 3 |
| HABBLAUD_CURSOR=0 | 2, 4 |
| Roadmap phases 2–3 not built | all |
| Tests + criterion | 2, 4 |
