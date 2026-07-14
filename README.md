# PromptLab Studios — Project Handover

**Last updated:** July 14, 2026  
**File:** `PromptLab_Instagram_Updated.html`  
**Type:** Single-file HTML/JS app — no build step, no dependencies, runs entirely in the browser  
**Target:** Karnataka-focused Instagram content generation (meme prompts + Vedic devotional prompts)

---

## 1. Project Overview

PromptLab generates image and video prompts for Instagram content across two tabs:

| Tab | Purpose |
|---|---|
| **Memes** | 8 meme image prompts across 4 categories — live Karnataka news via Gemini Google Search grounding |
| **Vedic Engine** | 2 image + 2 video prompts tied to today's top Hindu calendar event |

All outputs are prompts only — users copy them into their image/video generation tool of choice (Midjourney, Kling, etc.). Nothing is generated inside the app.

---

## 2. Architecture

### Tech Stack
- Pure HTML + vanilla JS (ES module, `<script type="module">`)
- No npm, no bundler, no framework
- Google Fonts: Bricolage Grotesque, IBM Plex Mono, IBM Plex Sans
- All state in memory + `localStorage`

### API
- **Primary**: Google Gemini API (free tier)  
  Endpoint: `https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent?key={key}`
- **No secondary provider** — Groq was evaluated but dropped (no Google Search grounding)
- **No caching** — removed by design, all runs are live

### File Structure (single file)
```
<head>   — meta, fonts, CSS (~430 lines)
<body>   — HTML structure
  <header>  topbar (brand, model chip, Settings button)
  <section> tab switcher (Memes / Vedic Engine)
  <div>     Memes panel (deck + press lane)
  <div>     Vedic panel (deck + divine/video lanes)
  <footer>
  <aside>   Settings drawer
  <div>     Toast, File-warning modal
<script type="module">  — all JS (~1100 lines)
```

---

## 3. Settings Schema

Stored in `localStorage` key `doota_settings_v1` as JSON.

| Field | Type | Default | Description |
|---|---|---|---|
| `gem` | string | `""` | Gemini API key |
| `model` | string | `"gemini-2.5-flash"` | Selected Gemini model ID |
| `window` | number | `24` | Memes news lookback window in hours (24 or 48) |
| `useSearch` | number | `1` | Live Google Search grounding (1=on, 0=off) |
| `region` | string | `"Karnataka"` | Region for Memes news search |
| `lookahead` | number | `72` | Vedic event search window in hours (72 or 168) |

### Other localStorage Keys

| Key | Purpose |
|---|---|
| `promptlab_active_tab` | Last active tab (memes / divine) |
| `promptlab_model_cache` | `{ models:[{id,tier,group}], fetchedAt }` — 7-day TTL live model list |
| `promptlab_console_collapsed_v2` | `{ memes:bool, divine:bool }` — console collapse state |
| `promptlab_lane_collapsed` | `{ press:bool, divine:bool, video:bool }` — lane collapse state |
| `promptlab_settings_groups` | `{ gemini:bool, memestyle:bool, … }` — settings group collapse state |

---

## 4. Gemini API Call Architecture

### Model Selection
The dropdown is dynamically populated in two ways:
1. **Built-in fallback** (`FALLBACK_MODELS` constant) — 7 curated models shown when no live fetch available
2. **Live fetch** — `GET /v1beta/models?key=...&pageSize=100` → filtered to intersection with whitelist → cached 7 days

### Curated Model Whitelist (June 2026)
```js
const FALLBACK_MODELS = [
  { id:"gemini-3.1-flash-lite",  tier:"free", group:"gen3-free" }, // GA · 30 RPM
  { id:"gemini-3.5-flash",       tier:"free", group:"gen3-free" }, // GA · 15 RPM
  { id:"gemini-3-flash-preview", tier:"free", group:"gen3-free" }, // Preview · 10 RPM
  { id:"gemini-2.5-flash",       tier:"free", group:"gen2-free" }, // Stable · default ✓
  { id:"gemini-2.5-flash-lite",  tier:"free", group:"gen2-free" }, // Stable · 30 RPM
  { id:"gemini-3.1-pro-preview", tier:"paid", group:"paid"      }, // Paid only
  { id:"gemini-2.5-pro",         tier:"paid", group:"paid"      }, // Paid only
];
```

**Note:** `gemini-2.0-flash` was shut down June 1, 2026. Pro models are paid-only since April 2026.

### Rate Limit Configuration
```js
const BACKOFFS = [6000, 15000, 30000]; // ms — used for 500/503 + empty-response retries only
```

- `MIN_GAP_MS` throttle gate was removed entirely (was 4500ms dead wait)
- **RPD detection**: daily cap errors stop retrying immediately (no point waiting)
- **RPM 429**: NO in-model retries — throws immediately so the fallback chain switches models instantly (Flash-Lite has a separate RPM bucket)
- **Thinking disabled**: `thinkingConfig:{thinkingBudget:0}` sent for all `gemini-2.5*` models (thinking is ON by default and eats maxOutputTokens → latency + MAX_TOKENS loops). Strictly 2.5-only — 3.x models cannot disable thinking and may error on the field
- **failFast option**: `callGemini(..., {failFast:true})` limits retries to 1 — used by the Memes Controversy call

### Per-Engine Token Budgets
```js
const MAX_TOKENS = { press:3500, divine:2500, video:2500, virality:2600 };
```

### Fallback Chain (rate-limit triggered)
```
Primary model → gemini-2.5-flash-lite → gemini-3.1-flash-lite → fail
```
Skips models already in use. Does NOT retry on daily cap (`isDailyCapHit`).

### Search OFF Auto-routing
When `useSearch=0`, calls auto-route to `gemini-2.5-flash-lite` (30 RPM, fastest for pure JSON).

### AbortController (Stop Button)
- Each engine has its own `AbortController` stored in `abortCtrl[kind]`
- Signal is threaded through `callGemini → rawGemini → fetch()` and all `sleep()` calls
- `sleep(ms, signal)` is abort-aware — rejects immediately via event listener on signal, not polling
- Stop during Memes Call 1 → reset to empty; Stop during Call 2 → keep 6 cards from Call 1

---

## 5. Engines

### 5.1 Memes Crew (Press Lane)

**Two PARALLEL API calls per run** (launched concurrently; safety isolation preserved — a Call 2 block never affects Call 1):

#### Call 1 — Standard News (6 cards)
- Prompt: `pressGenPrompt()`
- Search: ON always (needs live news)
- Returns: 2 Political + 2 Breaking News + 2 Kannada Film
- On fail: entire run aborts

#### Call 2 — Controversy (2 cards)
- Prompt: `controversyGenPrompt()` — reworded to filter-safe meta-language ("sharp, pointed political satire", "most talked-about controversies") after the old wording ("explosive", "no topic off-limits") tripped Gemini's prompt filters
- Search: ON always · runs with `{failFast:true}` (1 retry max)
- On block/failure: warn log + toast + visible **skipped-notice card** in the Controversy sub-group with a **↻ Retry Controversy** button that re-runs only Call 2 (Call 1 results & quota untouched). Button shows only when Controversy came back empty

**Skeleton sequence:**
- Run starts → 6 skeletons, both calls fire in parallel
- Call 1 settles → 6 cards render in sub-groups, Controversy sub-group shows 2 skeletons
- Call 2 settles → Controversy sub-group updates in-place (cards, or skipped-notice + retry)

**Category normalization:** `normalizeCategory()` maps drifting model strings ("Politics", "Breaking", "Film"…) onto the 4 canonical sub-groups; unrecognised categories render in a 📌 "Other" sub-group instead of vanishing silently.

**Sub-group structure (all start expanded, no persistence):**
```
🗞 Political    · 2 cards  ▾
🔥 Controversy  · 2 cards  ▾
📰 Breaking News· 2 cards  ▾
🎬 Kannada Film · 2 cards  ▾
```

**Image prompt formats** (all end with a Kanglish instruction):
- Political / Controversy: `"Generate an 4K, Meme Style Image, 1:1 ratio, " + punchline + ", " + summary + ", on-image text in a natural Kannada-English (Kanglish) mix, Latin script only"`
- Breaking News / Kannada Film: `"Generate 4k Memes style Image (1:1 ratio), " + punchline + ", " + summary + ", on-image text in a natural Kannada-English (Kanglish) mix, Latin script only"`

**Caption format:** witty caption + (new line) neutral 2-line news summary + EXACTLY 4 hashtags on the final line.

**Uniqueness / dedup (July 2026):**
- Both prompts carry a distinct-stories rule (6 different stories in Call 1; 2 different controversies in Call 2)
- `controversyGenPrompt(excludeHeadlines)` accepts an exclusion list injected as "ALREADY COVERED — pick DIFFERENT controversies"
- Client-side `dedupeControversy()` (word-overlap similarity ≥ 0.6 on headline or summary via `storiesSimilar`) runs after both calls settle
- On collision: ONE automatic Controversy retry with Call 1 headlines excluded; falls back to the unique subset if the retry also collides/fails
- Manual ↻ Retry Controversy also passes the exclusion list and dedupes

**Virality Agent (3rd call, July 2026):**
- Non-blocking `runViralityAgent()` fires after cards render (and after Controversy retry); own AbortController (`viralityCtrl`), aborted on new Memes runs
- `viralityGenPrompt(items)` — search-grounded, `failFast`, kind `"virality"` (2600 tokens)
- Returns per card: `viral_score` 0-100 + `viral_band` High/Medium/Low; per hashtag: score, band, status Live/Inactive/Risky
- UI: `.viral-wrap` under the caption — trend pill "🔥 High · ~78% viral" + hashtag chips with status dot, band/%, and IG↗/FB↗ verify links (`instagram.com/explore/tags/…`, `facebook.com/hashtag/…`)
- Cards matched via `card.dataset.h` (first 80 chars of headline); tags sanitised to `[A-Za-z0-9_]` before innerHTML
- Scores are Gemini estimates from live search — NOT official IG/FB metrics (no free API exposes those)
- On failure: pill becomes "⚠︎ click to retry" (re-runs the whole batch); Memes run now costs 3 API calls

**JSON schema per card:**
```json
{
  "category": "Political|Breaking News|Kannada Film|Controversy",
  "headline": "string",
  "summary": "string",
  "punchline": "string",
  "image_prompt": "string",
  "caption": "string (ends with 4 hashtags)"
}
```

---

### 5.2 Vedic Image Agent (Divine Lane)

**Single API call, 2 cards:**

| Card | Style | Prompt prefix |
|---|---|---|
| 1 | Cinematic | `"Generate 4k Cinematic style image (1:1) with warm ivory background, "` |
| 2 | Vedic | `"Generate 4k Vedic style image (1:1) with warm ivory background, "` |

**Event selection logic** (date-anchored since July 2026 — fixes outdated-event bug; the block previously never stated today's date, so Gemini anchored "next 72h" to its training-data present):
1. Prompt opens with `TODAY IS <live IST date+time>` and hard rules: event MUST be today or later, ignore previous-year/already-passed occurrences
2. Option 1: Search next `lookahead` hours (72h or 7d) for significant Hindu calendar events
3. Option 2 (fallback): Research today's Panchang (tithi/nakshatra/vrat), verified against today's actual date

**Image prompts** end with `", any on-image text in a respectful Kannada-English (Kanglish) mix, Latin script only"`.
**Caption format:** devotional caption + (new line) 2-line significance/timings summary + 4 hashtags.

**Karnataka note:** Uses Amanta lunar system (South Indian convention) over North Indian Purnimanta when dates differ.

**JSON schema per card:**
```json
{
  "event_name": "string",
  "day": "string",
  "date": "string",
  "significance": "string",
  "timings": "string (IST)",
  "key_rituals": "string",
  "god_name": "string",
  "god_significance": "string",
  "image_prompt": "string",
  "caption": "string (ends with 4 hashtags)",
  "best_posting_time": "string"
}
```

**Card UI:** Card 1 (Cinematic): Title + role pill + Image prompt + Caption. Card 2 (Vedic): Title + role pill + Image prompt — caption hidden by design (still generated in JSON). All metadata fields (significance, timings, etc.) are generated but not displayed.

---

### 5.3 Vedic Video Agent (Video Lane)

**Single API call, 2 cards:**

| Card | Style | Prompt prefix |
|---|---|---|
| 1 | Vedic | `"Generate Vedic style video (9:16), 10 Sec, "` |
| 2 | Cinematic | `"Generate Cinematic style video (9:16), 10 Sec, "` |

Same event selection logic as Image Agent. Camera angles are model-free (no fixed phrasing required).

**JSON schema per card:**
```json
{
  "event_name": "string",
  "day": "string",
  "date": "string",
  "significance": "string",
  "god_name": "string",
  "god_significance": "string",
  "video_prompt": "string",
  "caption": "string (ends with 4 hashtags)",
  "best_posting_time": "string"
}
```

**Card UI:** Title + role pill (Vedic/Cinematic) + Video prompt + Caption only.

---

## 6. JSON Repair Logic

The `repairJsonString(t)` function handles malformed model output:
- Walks text character-by-character tracking string boundaries
- For quotes inside strings: uses lookahead (next non-whitespace char) to decide if it's a real closer or embedded quote
- Escapes `\n`, `\r`, `\t` inside strings
- Drops control characters `\x00-\x08`, `\x0b`, `\x0c`, `\x0e-\x1f`

`extractArray(text)` → strips markdown fences → `JSON.parse` → if fails → `repairJsonString` → retry parse → throw descriptive error

---

## 7. UI Components

### Top Bar
- Brand logo + name
- Model chip (shows active model, green dot = key saved, updates live on dropdown change)
- Settings button

### Tab Bar
- Memes tab / Vedic Engine tab
- Active tab persisted to `localStorage`

### Activity Console (per tab)
- Collapsible (state persisted per tab)
- Shows timestamped log lines with icons and colour coding
- API call counter (shared across both tabs)
- Default: collapsed on mobile (≤640px)

### Lanes
- Press lane (Memes), Divine lane (Vedic Image), Video lane (Vedic Video)
- Each lane is collapsible (state persisted)
- Press lane has 4 collapsible sub-groups (Political/Controversy/Breaking/Film)
- Sub-group collapse: always starts expanded, not persisted

### Cards
- Animated entrance (`cardin` keyframe)
- Card top: index number + title + category/role pill
- Card body: prompt field + caption field (both collapsible, auto-collapsed on mobile)
- Copy button per field with ✓ confirmation

### Settings Drawer
- Slide-in from right, backdrop scrim
- Groups: Brain · Gemini, Memes · Region, Memes · News window, Vedic Engine · Search window, Danger zone
- All groups collapsible (state persisted)
- Model dropdown: dynamically rebuilt from live fetch / fallback
- ↻ Refresh button fetches live model list and shows active count + date

### Stop Buttons
- One per engine (🛑 Stop Memes, 🛑 Stop Image, 🛑 Stop Video)
- Hidden by default, shown only while that engine is running
- Uses `AbortController` — works during fetch AND sleep backoffs

---

## 8. Known Constraints & Limitations

| Constraint | Detail |
|---|---|
| Free tier RPM | Flash: 15 RPM, Flash-Lite: 30 RPM — Memes (2 calls) can hit this on rapid reruns |
| Free tier RPD | 1,500 requests/day (all engines share same key) |
| Pro models | Paid-only since April 2026 — free key users will get rate limit errors |
| Google Search grounding | Available only via Gemini — no alternative free provider |
| file:// protocol | Gemini API blocked from local file — must be hosted (localhost or web) |
| Safety blocks | Controversy cards may be blocked by Gemini's safety filters regardless of prompt wording |
| Gemini 2.0 models | Shut down June 1, 2026 — removed from all lists |
| Single Gemini key | No key rotation — separate Google Cloud projects needed for more quota |
| No caching | All runs are live — decided against caching for freshness |

---

## 9. File Delivery Protocol

- **Project file (read-only):** `/mnt/project/PromptLab_Instagram_with_Astro.html`
- **Output file:** `/mnt/user-data/outputs/PromptLab_Instagram_Updated.html`
- **Working copy:** `/home/claude/PromptLab_Instagram_Updated.html`
- **Validation:** Python extracts `<script type="module">` → `node --check extracted.mjs` before every delivery
- **Edit method:** `str_replace` for targeted edits; Python line-range replacement for large blocks

---

## 10. Development Rules

1. **No building until user says BUILD, Update, or Change**
2. **Strict "no guessing" on missing specs** — ask clarifying questions first
3. **Targeted `str_replace` edits** over full rewrites where possible
4. **Syntax check mandatory** before every file delivery
5. **Single output file** — all changes go to `PromptLab_Instagram_Updated.html`
6. **MD file updated** on major feature additions

---

## 11. Feature History (Chronological)

| Feature | Status |
|---|---|
| Memes Crew — 3 political cards | ✅ Shipped → replaced |
| Vedic Engine — 3 image + 3 video cards | ✅ Shipped → replaced |
| Memes restructured to 6 cards (3 categories × 2) | ✅ Shipped |
| Vedic Image: 3 parts → 2 (Vedic + Cinematic) | ✅ Shipped |
| Vedic Video: 3 parts → 2 (Vedic + Cinematic) | ✅ Shipped |
| Dynamic live model list (↻ Refresh, 7-day cache) | ✅ Shipped |
| Model chip live update on dropdown change | ✅ Shipped |
| Whitelist intersection on live fetch | ✅ Shipped |
| Stop buttons with AbortController | ✅ Shipped |
| Abort-aware sleep() — stops during backoff waits | ✅ Shipped |
| API optimisation: remove throttle gate, tighten backoffs | ✅ Shipped |
| RPD daily cap detection (no-retry) | ✅ Shipped |
| Right-sized maxOutputTokens per engine | ✅ Shipped |
| Flash-Lite auto-routing when search OFF | ✅ Shipped |
| Prompt compression ~30% | ✅ Shipped |
| Gemini 2.0 models removed (shut down Jun 2026) | ✅ Shipped |
| Vedic Image prompt order swapped (Cinematic first) | ✅ Shipped |
| Vedic cards stripped to Title + Prompt + Caption | ✅ Shipped |
| Memes dual-call (8 cards, 4 sub-groups) | ✅ Shipped |
| Controversy sub-group (2 cards, unfiltered satire) | ✅ Shipped |
| Groq fallback | ❌ Dropped — no Google Search grounding |
| Result caching | ❌ Dropped — decided against for freshness |
| Thinking disabled on 2.5 models (thinkingBudget:0) | ✅ Shipped |
| Memes dual-call parallelized (Promise-style concurrent) | ✅ Shipped |
| RPM 429 → immediate model fallback (no in-model backoff) | ✅ Shipped |
| Controversy prompt reworded filter-safe + failFast | ✅ Shipped |
| Controversy skipped-notice + ↻ Retry-only button | ✅ Shipped |
| Category normalization + "Other" catch-all sub-group | ✅ Shipped |
| Safety finishReasons handled (PROHIBITED_CONTENT/BLOCKLIST/SPII) | ✅ Shipped |
| API counter counts every HTTP request (retries/fallbacks) | ✅ Shipped |
| Default-model index bug fixed (was resolving to Flash-Lite) | ✅ Shipped |
| Dead code removed (classifyModel, spinOn dataset.txt) | ✅ Shipped |
| Memes uniqueness: distinct-story rules + client dedup + auto-retry w/ exclusion list | ✅ Shipped |
| Virality Agent — 3rd non-blocking call: story viral band+%, per-hashtag band/%/Live-Risky status, IG/FB verify links | ✅ Shipped |
| Kanglish (Kannada-English, Latin script) instruction appended to all meme + Vedic image prompts | ✅ Shipped |
| Captions restructured: caption + 2-line summary + 4 hashtags (Memes + Vedic Image) | ✅ Shipped |
| Vedic outdated-info fix: live IST date anchor + future-only event rule in eventSelectionBlock | ✅ Shipped |
| Vedic Image 2 caption removed from card UI (JSON unchanged) | ✅ Shipped |
