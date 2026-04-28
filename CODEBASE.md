# VIONYX — Codebase Guide

Internal reference for anyone working on this codebase. Explains what each section does and why it was built the way it was.

---

## Architecture Overview

Single-page application. No framework, no build tool, no bundler. Everything lives in `index.html` — HTML, CSS, and JavaScript in one file. This was intentional: the app has one screen with two states (auth + app). A framework would add deploy complexity and dependency surface for no real gain at this scale.

**Stack:**
- Frontend: Vanilla HTML/CSS/JS, hosted on Vercel
- Auth + database: Supabase (email/password, RLS-protected tables)
- AI proxy: Cloudflare Worker (routes browser requests to Anthropic API)
- AI model: claude-sonnet-4 (via the proxy, never called directly from the browser)

The browser never holds the Anthropic API key. All AI calls go through the Cloudflare Worker, which injects the key server-side.

---

## File Structure

```
index.html               — Main app (auth screen + full app, all logic)
auth/confirm/index.html  — Email verification landing page
legal/privacy/index.html — Privacy policy (CCPA/CPRA compliant)
legal/terms/index.html   — Terms of service (California governing law)
logo.png                 — Wordmark asset
.gitignore               — Blocks INTERNAL.md, .docx, .env, editor files
INTERNAL.md              — Local-only operational reference (gitignored)
```

---

## index.html Sections

### CSS Design System (lines 10–96)

CSS custom properties at `:root` define the entire color system. All UI components pull from these variables — nothing is hardcoded outside this block. This makes theme changes a single-point edit.

Key variables:
- `--bg` / `--bg2` / `--bg3` — layered dark backgrounds
- `--border` / `--border2` — border levels for depth
- `--text` / `--text2` / `--text3` — text hierarchy
- `--accent` / `--accent2` / `--accent-dim` — purple accent system

Two typefaces: `Syne` (headers, labels, branded elements — geometric, high letter-spacing) and `DM Sans` (body copy, inputs — clean, readable at small sizes). Both loaded from Google Fonts.

No responsive breakpoints. The app targets desktop. The auth card is fixed-width (380px). The chat layout uses percentage-based widths for message bubbles. A max-width cap on `#root` prevents the layout from stretching on wide monitors.

### HTML Structure (lines 98–174)

Two root-level screens that toggle via the `.hidden` class:

- `#auth-screen` — sign in / create account, tab-switched
- `#app-screen` — full app with topbar, two tabs (Reflect + Journal), footer

The app screen contains:
- `#reflect-tab` — chat interface with message area, save banner, and input row
- `#journal-tab` — filterable entry list

### Supabase Client (lines 177–180)

The anon key (`sb_publishable_...`) is intentionally in the browser. Supabase's anon key is designed to be public — it grants access only within the boundaries of the Row Level Security policies on each table. Data is protected by RLS, not by hiding the key.

### System Prompt — `SYSTEM_PROMPT` (lines 182–243)

The core behavioral contract sent to the AI on every request. It defines:

- **Voice rules** — no emojis, no therapy clichés, no advice unless asked. Match the user's energy and pace.
- **Detection zones** — 6 internal categories (Emotional, Pattern, Organizing, Identity, Commitment, Completion) the AI classifies each message into before responding. The user never sees this classification.
- **Auto-log rules** — Commitment and Completion types generate a structured log block the app parses and saves. Emotional/Pattern/Organizing/Identity types never auto-log; they offer a save banner on wind-down signals only.
- **Privacy absolute** — the AI is never allowed to reveal that logging, routing, schemas, or backend systems exist.
- **Memory framing** — the AI never says it has no memory or starts fresh. If pattern context is available, it uses it naturally. If not, it redirects to the present moment.

The prompt is written to produce behavior that feels like a grounded human presence, not a chatbot. Every rule exists to prevent a specific failure mode (over-comforting, advice-giving, breaking the fourth wall, pressuring the user).

### Entry Colors — `ENTRY_COLORS` (lines 245–252)

Maps the 6 entry types to their visual identity (background, border, dot color, display label). Centralized here so both the chat log tag and the journal card pull from the same source. Changing a color is a single-line edit.

### Global State (lines 254–262)

Minimal. Only what's needed across function calls:
- `currentUser` — Supabase auth user object
- `currentUserTier` — loaded from `profiles` table (1, 2, or 3)
- `currentUserStatus` — loaded from `profiles` table (pending, active, suspended)
- `entries` — full entry array loaded on login, updated on persist
- `history` — message history sent to the AI (Claude's conversation context)
- `sessionMessages` — parallel array for rendering; includes `log` metadata the AI sends back
- `pendingLog` — holds a log block awaiting user save confirmation
- `loading` / `authMode` — UI state flags

`history` and `sessionMessages` are separate because the AI only needs clean text (no log metadata), while the UI needs log metadata to render entry tags and save banners. Conflating them would mean stripping log blocks on every API call instead of once.

### Auth Flow (lines 264–335)

`window.addEventListener("load")` — checks for an existing Supabase session on page load. If one exists, skips the auth screen entirely and calls `showApp()`.

`handleAuth()` — handles both sign-up and sign-in from the same form. Tab state (`authMode`) determines which Supabase method to call. Sign-up shows a confirmation message; sign-in calls `showApp()` directly.

`signOut()` — clears sessionStorage (wipes session memory), calls Supabase sign-out, resets all state, hides all screens, returns to auth.

`showApp()` — the entry point into the app. Calls `loadProfile()` first, then routes based on `currentUserStatus`: active users get the full app, pending users get the pending screen, suspended users get the suspended screen. Only active users trigger `loadEntries()` and session history restore.

### Entry Management (lines 360–383)

`loadEntries()` — fetches all entries for the current user, ordered newest-first. Runs once on login. Entries are kept in memory and updated locally on new persists to avoid re-fetching.

`persistEntry()` — inserts a single entry to Supabase and prepends it to the local `entries` array. Called either automatically (Tier 2+ commitments/completions) or after user confirmation (save banner).

### Pattern Context — `buildPatternContext()` (lines 385–437)

Compresses the user's entry history into a structured block injected into the system prompt. Weighted by signal value: Red/Black journals (emotional/pattern) carry the most weight and get the most detail. Commitments/completions carry the least.

Why weight by type: emotional and pattern entries contain the richest behavioral signal — recurring triggers, loops, identity statements. Commitments and completions are useful context but produce less insight per token. The weighting ensures the most meaningful signals survive the compression.

The context block instructs the AI to use history as present insight, not as memory recall. The user should feel understood, not tracked.

### Tier Prompt Layers (lines 244–253, after `SYSTEM_PROMPT`)

Three additive layers appended to the base system prompt based on `currentUserTier`:

- **Tier 1 — Witness**: Pure observation. No pressure, no auto-logging. Only logs on explicit user command. Designed for users building the habit of reflection.
- **Tier 2 — Calibrated**: Accountability and pattern naming are active. Auto-logs commitments and completions. Designed for users ready for structured execution tracking.
- **Tier 3 — Cold Mirror**: On hold. Prompt written, not deployed. Direct and unflinching — challenges assumptions and calls out avoidance. Invitation-only when it launches.

The base prompt handles all emotional attunement and privacy rules. Tier layers only modify execution behavior and pressure level. The privacy and tone rules are never overridden.

### AI Request — `sendMessage()` (lines 488–525)

1. Captures input, sets loading state, appends user bubble to chat
2. Sends `history + current message` to the Cloudflare Worker with the system prompt
3. Removes typing indicator, parses the response
4. Extracts `LOG_BEGIN...LOG_END` block if present (structured entry data the AI appends when it detects a loggable moment)
5. Strips the log block from the displayed text (user never sees it)
6. Routes based on entry type and tier:
   - Tier 2+, non-journal type → auto-persist
   - Tier 1, or any journal type → show save banner
7. Saves session to `sessionStorage`

The log block parsing (`parseLog()`) is tolerant — if the AI includes a malformed block, it returns null and the message renders as plain text with no log action.

### Session Persistence (lines 460–480)

Session history is stored in `sessionStorage` keyed by user ID (`vx_session_{userId}`). `sessionStorage` clears on tab close — intentional. The session is a working memory for the current sitting, not a permanent log. Permanent entries exist in Supabase; the session exists so the conversation doesn't reset on page refresh.

### Profile Loading — `loadProfile()` (after `loadSessionHistory`)

Queries `profiles` table for `tier` and `status`. Falls back to `tier: 1, status: pending` on any error or missing row — new users whose profile trigger hasn't fired yet land on the pending screen rather than breaking.

### UI Rendering (lines 527–609)

`appendMessage()` — creates message bubbles. For assistant messages with a log entry, appends a color-coded log tag below the bubble (from `ENTRY_COLORS`). The tag is suppressed when a save banner is showing (because the entry isn't saved yet — confirming or skipping changes the outcome).

`renderJournal()` — renders the full filtered entry list. Each card builds its fields conditionally based on entry type (journals show trigger + expression, commitments show commitment/plan/deadline, completions show task + summary).

`showSaveBanner()` / `confirmSave()` / `dismissSave()` — three-function save flow. Banner shows when a journal entry is detected. Confirm calls `persistEntry()`. Dismiss clears `pendingLog` without saving.

### Upgrade Panel (lines 568–595, after `dismissSave`)

`openUpgradePanel()` — applies active/badge state to the tier card matching `currentUserTier`, then shows the overlay. Tier 3 always shows "Coming Soon" regardless of user tier.

`updateTierButton()` — called from `showApp()` to sync the topbar tier badge text with the loaded tier number.

---

## auth/confirm/index.html

Standalone email verification page. Supabase sends the user here after signup. Extracts `token_hash` and `type` from the URL, calls `verifyOtp()`, shows confirmation text, and auto-redirects to the main app after 3 seconds.

Uses `Playfair Display` serif for the headline — a deliberate style departure from the main app to give the confirmation moment its own visual weight. First impression of the product.

---

## legal/

Both pages are standalone static documents. California governing law throughout (CCPA, CPRA, CMIA, Shine the Light, CalOPPA). The Terms include an explicit mental health disclaimer. The Privacy Policy covers all 7 California consumer rights and classifies emotional/behavioral data as CPRA-sensitive personal information.

---

## What's Not In This Repo

- **Cloudflare Worker code** — deployed via Wrangler separately. The worker receives `{model, system, messages, max_tokens}`, adds the Anthropic API key from Cloudflare secrets, and proxies the response.
- **Supabase migrations** — schema managed directly via Supabase dashboard. Tables: `entries`, `profiles`, `applications`, `pattern_flags`.
- **Stripe** — not integrated yet. Upgrade panel uses mailto as placeholder.
