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
| **Memes** | 6 meme image prompts across 3 categories (Controversy, Breaking News, Kannada Film) — live Karnataka news via Gemini Google Search grounding, with trend-aware captions |
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
const MAX_TOKENS = { pressA:3000, pressB:1800, divine:2500, video:2500, virality:2600 };
```
`pressA` = Breaking News + Controversy call (4 cards, bigger prompt). `pressB` = Kannada Film call (2 cards, lighter). Captions were folded into these calls (no separate caption call), which is what keeps the budgets tight despite each card now returning `tagline`/`summary_3line`/`hashtags` on top of the prompt fields.

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

### 5.1 Memes Crew (Press Lane) — v9

**Political category removed entirely.** 6 cards total across 3 categories, produced by **two PARALLEL API calls** (launched concurrently, each renders into its own sub-group(s) the moment it settles — genuinely progressive rendering, not a staged sequence):

#### Call A — Breaking News + Controversy, merged (4 cards)
- Prompt: `breakingControversyGenPrompt(useDirectWording)`
- Search: ON always (needs live news)
- Returns: 2 Breaking News + 2 Controversy in one call, each card carrying its own full caption fields (no separate caption call)
- Controversy angle uses **direct/raw wording** by default (`useDirectWording=true`) — the softened, filter-safe phrasing from the old dual-call design was dropped per spec
- **Safety-block fallback**: if the direct-wording attempt is blocked/fails, one automatic retry fires with `useDirectWording=false` (softer, filter-safe phrasing). If that also fails: warn log + toast + a **skipped-notice card** covering both Breaking News and Controversy sub-groups, with a **↻ Retry Breaking & Controversy** button (`retryCallA()`) that re-runs only Call A — Call B's (Film) results & quota are untouched
- Runs with `{failFast:true}` on both the direct and fallback attempts (a block is a block, not a network blip — no point sitting through the full backoff ladder)

#### Call B — Kannada Film (2 cards)
- Prompt: `kannadaFilmGenPrompt()`
- Search: ON always · normal retry ladder (less block-prone than Controversy)
- Returns: 2 Kannada Film cards with their own captions

**Skeleton sequence:**
- Run starts → all 3 sub-groups (Controversy, Breaking News, Kannada Film) show skeletons, both calls fire in parallel
- Whichever call settles first updates its sub-group(s) in place; the other sub-group keeps its skeletons until its call settles
- If Call A ultimately fails, its two sub-groups are replaced by one skipped-notice + retry; Kannada Film renders independently regardless

**Category normalization:** `normalizeCategory()` maps drifting model strings ("Breaking", "Film"…) onto the 3 canonical sub-groups; unrecognised categories render in a 📌 "Other" sub-group instead of vanishing silently.

**Sub-group structure (all start expanded, no persistence):**
```
🔥 Controversy  · 2 cards  ▾
📰 Breaking News· 2 cards  ▾
🎬 Kannada Film · 2 cards  ▾
```

**Image prompt formats** (Kanglish trailing sentence removed; replaced with an upfront language directive):
- Breaking News / Controversy: `"Generate 4k Meme's style Image (1:1 ratio), Text Language Kannada + English (Mix), " + punchline + ", " + summary`
- Kannada Film: `"Generate 4k Meme's style Image (1:1 ratio), Text Language Kannada + English (Mix), " + summary` (no punchline)

**Caption format (v9 — split JSON fields, assembled client-side):**
Each card returns `tagline`, `summary_3line`, and `hashtags` (array of exactly 4) as **separate JSON fields** instead of one pre-formatted caption string. `assembleCaption(tagline, summary3line, hashtags)` joins them client-side as: Tagline ⏎ 3-line summary ⏎ 4 hashtags. This was the fix for hashtags appearing mid-caption — the model can no longer misplace a hashtag inside free text because it never formats the caption as one block; the fields are joined in JS after the fact.
`CAPTION_FIELD_SPEC` (shared block injected into both prompts) also instructs the model to search current Instagram/Facebook trending caption styles and currently-active hashtags for the story, rather than writing generic evergreen tags.

**Uniqueness / dedup (v9 — cross-call only):**
- In-prompt uniqueness rule inside Call A forces Breaking News stories ≠ Controversy stories (single call, so this is a prompt instruction, not a client-side check)
- `dedupeAcrossCalls(callAItems, callBItems)` (word-overlap similarity ≥ 0.6 on headline or summary via `storiesSimilar`) drops any Kannada Film card that duplicates a Breaking/Controversy story, run after either call settles and again once both are in
- The old single-call "Call 1 vs Controversy" dedup + auto-retry-with-exclusion-list flow is obsolete and removed (there's no longer a within-category collision to guard against — Call A's uniqueness is now a prompt rule)

**Virality Agent (3rd call, unchanged from v8):**
- Non-blocking `runViralityAgent()` fires after both calls settle; own AbortController (`viralityCtrl`), aborted on new Memes runs
- `viralityGenPrompt(items)` — search-grounded, `failFast`, kind `"virality"` (2600 tokens)
- Returns per card: `viral_score` 0-100 + `viral_band` High/Medium/Low; per hashtag: score, band, status Live/Inactive/Risky
- `extractHashtags(item)` now reads the `hashtags` array field directly (falls back to regex-scanning `caption` only if the array is missing)
- UI: `.viral-wrap` under the caption — trend pill "🔥 High · ~78% viral" + hashtag chips with status dot, band/%, and IG↗/FB↗ verify links (`instagram.com/explore/tags/…`, `facebook.com/hashtag/…`)
- Cards matched via `card.dataset.h` (first 80 chars of headline); tags sanitised to `[A-Za-z0-9_]` before innerHTML
- Scores are Gemini estimates from live search — NOT official IG/FB metrics (no free API exposes those)
- On failure: pill becomes "⚠︎ click to retry" (re-runs the whole batch); Memes run now costs 3 API calls total (Call A + Call B + Virality)

**JSON schema per card:**
```json
{
  "category": "Breaking News|Controversy|Kannada Film",
  "headline": "string",
  "summary": "string (internal use — image prompt only, not shown in caption)",
  "punchline": "string (Breaking News/Controversy only)",
  "image_prompt": "string",
  "tagline": "string",
  "summary_3line": "string (exactly 3 lines, \\n-separated)",
  "hashtags": ["#tag1","#tag2","#tag3","#tag4"]
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

**Image prompts (v9):** the Kanglish trailing sentence was removed; each prefix now carries an upfront language directive instead —
- Image 1 (Cinematic): `"Generate 4k Cinematic style image (1:1) with warm ivory background, Text Language Kannada + English (Mix), " + event_name + ", " + god_name + ", " + god_significance`
- Image 2 (Vedic): `"Generate 4k Vedic style image (1:1) with warm ivory background, Text Language Kannada + English (Mix), " + event_name + ", " + day + " " + date + ", " + timings + ", " + key_rituals`

**Caption format (v9 — split JSON fields, assembled client-side, same pattern as Memes):** devotional `caption` (kept) + `summary_3line` (significance/timings in exactly 3 lines, was 2) + `hashtags` (array of 4) — joined client-side via `assembleCaption(caption, summary_3line, hashtags)`. Vedic Video Agent is untouched and still returns a single pre-formatted `caption` string.

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
  "caption": "string (devotional caption only, no hashtags)",
  "summary_3line": "string (exactly 3 lines, \\n-separated)",
  "hashtags": ["#SanatanDharma","#tag2","#tag3","#tag4"],
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
| Safety blocks | Controversy is merged into the same call as Breaking News (v9) — a safety block now risks both, mitigated by an automatic filter-safe-wording retry, then a manual ↻ Retry Breaking & Controversy button; Kannada Film (separate call) is unaffected either way |
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
| **v9:** Political category removed — Memes now 6 cards (Controversy, Breaking News, Kannada Film) | ✅ Shipped |
| **v9:** Memes restructured to 2 calls — Call A (Breaking News + Controversy, merged) + Call B (Kannada Film); old 3-call (Call 1/Call 2/Virality-only-separate) dual-call design retired | ✅ Shipped |
| **v9:** Progressive rendering — each sub-group updates the moment its call settles, independent of the other call | ✅ Shipped |
| **v9:** Controversy wording switched to direct/raw (softened filter-safe wording now only used as automatic fallback on safety block) | ✅ Shipped |
| **v9:** Kanglish trailing-sentence instruction replaced with upfront "Text Language Kannada + English (Mix)" directive (Memes + Vedic Image; Vedic Video untouched) | ✅ Shipped |
| **v9:** Captions restructured to split JSON fields (`tagline`/`caption` + `summary_3line` + `hashtags[4]`), assembled client-side — fixes hashtags appearing mid-caption | ✅ Shipped |
| **v9:** Caption generation folded into the main content calls (no separate caption-agent call) — captions are now trend-aware (search current IG/FB trending styles + live hashtags) at no extra API cost beyond the 2 content calls | ✅ Shipped |
| **v9:** Cross-call dedup (`dedupeAcrossCalls`) between Call A and Call B replaces the old single-call Call-1-vs-Controversy dedup/retry flow | ✅ Shipped |
| **v9:** Vedic Image caption summary widened from 2 lines to 3 lines | ✅ Shipped |
| **v9:** Token budgets re-split: `pressA:3000, pressB:1800` (was single `press:3500`) | ✅ Shipped |
