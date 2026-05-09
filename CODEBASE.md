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
admin/index.html         — Admin dashboard (application review, member management)
apply/index.html         — Application/waitlist form (5 questions + sliders)
auth/confirm/index.html  — Email verification landing page
legal/privacy/index.html — Privacy policy (CCPA/CPRA compliant)
legal/terms/index.html   — Terms of service (California governing law)
logo.png                 — Wordmark asset
.gitignore               — Blocks INTERNAL.md, .docx, .env, editor files
INTERNAL.md              — Local-only operational reference (gitignored)
```

---

## index.html Sections

### CSS Design System

CSS custom properties at `:root` define the entire color system. All UI components pull from these variables — nothing is hardcoded outside this block. This makes theme changes a single-point edit.

Key variables:
- `--bg` / `--bg2` / `--bg3` — layered dark backgrounds
- `--border` / `--border2` — border levels for depth
- `--text` / `--text2` / `--text3` — text hierarchy
- `--accent` / `--accent2` / `--accent-dim` — purple accent system

Two typefaces: `Syne` (headers, labels, branded elements) and `DM Sans` (body copy, inputs). Both loaded from Google Fonts.

Responsive layout: desktop (769px+) shows the journal panel and chat panel side by side, full viewport width. Mobile uses tabs to switch between them.

### HTML Structure

Screens that toggle via the `.hidden` class:

- `#auth-screen` — sign in / create account, tab-switched
- `#app-screen` — full app with topbar, app-body, footer
- `#pending-screen` — shown when status is pending
- `#suspended-screen` — shown when status is suspended

The app-body inside `#app-screen` contains:
- `#reflect-tab` — chat interface (right panel on desktop, default tab on mobile)
- `#journal-tab` — filterable entry list (left panel on desktop, second tab on mobile)

On desktop the tab buttons are hidden via CSS and both panels are always visible via `display: flex !important`. On mobile the tab buttons control visibility normally.

### Supabase Client

The anon key (`sb_publishable_...`) is intentionally in the browser. Supabase's anon key is designed to be public — it grants access only within the boundaries of the Row Level Security policies on each table. Data is protected by RLS, not by hiding the key.

### System Prompt — `SYSTEM_PROMPT`

The core behavioral contract sent to the AI on every request. It defines:

- **Voice rules** — no emojis, no therapy clichés, no advice unless asked. Match the user's energy and pace.
- **Detection zones** — 6 internal categories (Emotional, Pattern, Organizing, Identity, Commitment, Completion) the AI classifies each message into before responding. The user never sees this classification.
- **Auto-log rules** — Commitment and Completion types generate a structured log block the app parses and saves. Emotional/Pattern/Organizing/Identity types never auto-log; they offer a save banner on wind-down signals only.
- **Privacy absolute** — the AI is never allowed to reveal that logging, routing, schemas, or backend systems exist.
- **Memory framing** — the AI never says it has no memory or starts fresh. If pattern context is available, it uses it naturally. If not, it redirects to the present moment.

### Tier Prompt Layers

Three additive layers appended to the base system prompt based on `currentUserTier`:

- **Tier 1 — Witness**: Active mirroring and clarifying. Reflects what it hears back to the user, names patterns it notices, and asks one question per response that goes one layer deeper into what the user is already expressing. No action questions, no pressure, no auto-logging without explicit command. Designed to build self-awareness and create appetite for Tier 2 structure.
- **Tier 2 — Calibrated**: Accountability and pattern naming are fully active. Holds users to stated commitments, names avoidance, sharpens vague intentions into specifics. Auto-logs commitments and completions. Pattern context connects current behavior to past patterns directly.
- **Tier 3 — Cold Mirror**: Built for high performers with strong self-awareness and pattern recognition. No warmth buffer. Reflects what is actually underneath the framing — not the surface story. Amplifies signal, strips noise. Will not validate a weak read. The question it asks cuts to what is being protected or avoided. Invitation-only access.

The base prompt handles all emotional attunement and privacy rules. Tier layers only modify execution behavior and pressure level.

### Entry Colors — `ENTRY_COLORS`

Maps the 6 entry types to their visual identity (background, border, dot color, display label). Centralized here so both the chat log tag and the journal card pull from the same source.

### Global State

Minimal. Only what's needed across function calls:
- `currentUser` — Supabase auth user object
- `currentUserTier` — loaded from `profiles` table (1, 2, or 3)
- `currentUserStatus` — loaded from `profiles` table (pending, active, suspended)
- `entries` — full entry array loaded on login, updated on persist
- `history` — message history sent to the AI (Claude's conversation context)
- `sessionMessages` — parallel array for rendering; includes `log` metadata the AI sends back
- `pendingLog` — holds a log block awaiting user save confirmation
- `loading` / `authMode` — UI state flags

`history` and `sessionMessages` are separate because the AI only needs clean text (no log metadata), while the UI needs log metadata to render entry tags and save banners.

### Auth Flow

`window.addEventListener("load")` — checks for an existing Supabase session on page load. If one exists, skips the auth screen entirely and calls `showApp()`.

`handleAuth()` — handles both sign-up and sign-in from the same form.

`signOut()` — clears sessionStorage, calls Supabase sign-out, resets all state.

`showApp()` — calls `loadProfile()` first, then routes based on `currentUserStatus`: active users get the full app, pending users get the pending screen, suspended users get the suspended screen.

### Entry Management

`loadEntries()` — fetches all entries for the current user, ordered newest-first. Runs once on login.

`persistEntry()` — inserts a single entry to Supabase and prepends it to the local `entries` array.

### Pattern Context — `buildPatternContext()`

Compresses the user's entry history into a structured block injected into the system prompt. Weighted by signal value: Red/Black journals (emotional/pattern) carry the most weight. Commitments/completions carry the least. Capped at 3000 characters to prevent token overflow.

### AI Request — `sendMessage()`

1. Captures input, sets loading state, appends user bubble to chat
2. Sends `history + current message` to the Cloudflare Worker with the system prompt
3. Parses the response, extracts `LOG_BEGIN...LOG_END` block if present
4. Strips the log block from displayed text (user never sees it)
5. Routes based on entry type and tier: Tier 2+ non-journal types auto-persist; everything else shows a save banner
6. Saves session to `sessionStorage`

### Session Persistence

Stored in `sessionStorage` keyed by user ID. Clears on tab close — intentional. The session is working memory for the current sitting. Permanent entries live in Supabase.

### Profile Loading — `loadProfile()`

Queries `profiles` table by `id` (not `user_id` — the profiles table uses `id` as its primary key linked to `auth.users.id`). Falls back to `tier: 1, status: pending` on any error. Uses `|| 1` not `?? 1` because tier defaults to 0 in the DB and `??` does not catch 0 as falsy.

### Upgrade Panel

`openUpgradePanel()` — applies active/badge state to the tier card matching `currentUserTier`. Tier 3 shows "YOUR TIER" when active, nothing otherwise (it is live, not coming soon).

---

## admin/index.html

Standalone admin dashboard at `/admin/`. Auth-gated: checks Supabase session then queries `profiles.is_admin`. Non-admins see a blank wall. Never linked from the main app — accessed by URL directly.

Three sections:

**Applications** — loads all applications, cross-references profiles to filter pending-only. Each card shows full Q&A and 1-10 ratings. Accept sets status to active + tier 1. Reject sets status to suspended. Card removes from DOM after decision.

**Members** — all active profiles joined with application data for name/email. Each row has tier pills (1/2/3), an admin toggle (MAKE ADMIN / ADMIN), and a SUSPEND button. All hidden on the current user's own row to prevent self-demotion or self-suspension.

**Suspended** — all suspended profiles. Each row has a REINSTATE button that sets status back to active.

Multiple admins are supported — any profile with `is_admin = true` gets access. The admin toggle in the members panel grants or removes this without needing Supabase dashboard access.

---

## apply/index.html

Standalone application form at `/apply/`. Requires an active Supabase session. Redirects non-pending users back to `/`. Tracks submission via `localStorage` (`vx_applied_{userId}`) to show confirmation state on return visits without a SELECT query.

Five questions, each with a text area and a 1-10 slider. Text is optional but encouraged. The slider always has a value. Submission requires name, age, and gender — nothing else is mandatory. Data goes to the `applications` table.

Duplicate submission (error code 23505) is treated as success — sets localStorage flag and shows confirmation rather than erroring.

---

## auth/confirm/index.html

Email verification page. Extracts `token_hash` and `type` from the URL, calls `verifyOtp()`, shows confirmation, and auto-redirects to the main app after 3 seconds.

`type` param is allowlisted to prevent injection. `error_description` is capped at 200 characters before display.

---

## legal/

Both pages are standalone static documents. California governing law throughout (CCPA, CPRA, CMIA, Shine the Light, CalOPPA). The Terms include a mental health disclaimer. The Privacy Policy covers all 7 California consumer rights and classifies emotional/behavioral data as CPRA-sensitive personal information.

---

## What's Not In This Repo

- **Cloudflare Worker code** — deployed via Wrangler separately. Receives `{model, system, messages, max_tokens}`, adds the Anthropic API key from Cloudflare secrets, proxies the response.
- **Supabase migrations** — schema managed via Supabase dashboard. Tables: `entries`, `profiles`, `applications`, `pattern_flags`.
- **Stripe** — not integrated yet. Upgrade panel uses mailto as placeholder.
