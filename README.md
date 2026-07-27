# PromptLab Studios — Project Handover

**Last updated:** July 27, 2026
**File:** `PromptLab_Instagram_Updated.html`
**Type:** Single-file HTML/JS app — no build step, no dependencies, runs entirely in the browser
**Target:** Karnataka-focused Instagram/Facebook meme prompt generation

---

## 1. Project Overview — v10 (major rebuild)

PromptLab now generates **one thing**: meme image prompts, structured on the same "4×3" pipeline as the sibling `prompt-studio-kartanews` app.

| What | Detail |
|---|---|
| **Vedic Engine** | ❌ **Removed entirely** — tab, panel, CSS, JS engine, settings group, all gone |
| **Memes** | Rebuilt on kartanews logic: 1 pipeline (search → structure), 4 stories (2 Political Controversy + 2 Breaking News), 3 image prompts per story (Hook / Meme / Summary style) + 1 shared caption per story |

All outputs are prompts only — users copy them into their image/video generation tool of choice (Midjourney, Kling, etc.). Nothing is generated inside the app.

---

## 2. What changed in this rebuild

### Removed
- Vedic Engine (Image + Video agents), its tab, lane, settings group (`vedicopts`), and all `divine`/`video` JS/CSS
- Top-level tab switcher (only one panel now, so tabs were dropped)
- Kannada Film category (per decision: drop entirely, not folded into the 4 stories)
- Virality Agent (3rd API call, per-hashtag Live/Risky scoring) — dropped to match kartanews's simpler logic
- Old dual-call Memes design (Call A: Breaking+Controversy, Call B: Kannada Film), sub-groups, cross-call dedup

### Rebuilt — Memes engine ("4×3" logic, matches `prompt-studio-kartanews`)
- **Single pipeline**: Step 1 — grounded Google Search for raw headlines (plain prose). Step 2 — one structuring call (schema JSON, no search) that dedupes, grades (virality/severity, internal-only), and selects exactly 4 stories: 2 "Political Controversy" + 2 "Breaking News" (closest-fit fallback if a category is thin).
- **3 image prompts per story**, each starting with the exact required prefix:
  - `"Generate a 4k, Hook style Image (1:1 ratio), Text Language Kannada + English (Mix), ..."`
  - `"Generate a 4k, Memes style Image (1:1 ratio), Text Language Kannada + English (Mix), ..."`
  - `"Generate a 4k, Summary style Image (1:1 ratio), Text Language Kannada + English (Mix), ..."`
- **1 shared caption per story** (not per image, per explicit decision): tagline + summary (up to 4 lines) + 4 hashtags, assembled client-side from split JSON fields so hashtags can never land mid-caption.
- Card UI: each of the 4 cards shows 3 collapsible image-prompt fields + 1 caption field, each with its own Copy button.

### UI — mobile-first rebuild
- New app icon/logo (flask + quill "PromptLab" mark) used as the top-bar brand mark
- New gradient background image (pink → purple → blue → orange blob art) set as a fixed, dimmed body background behind all panels
- Single-column, mobile-first layout with a responsive breakpoint so it also works well on desktop (wider `--maxw`, restored padding, etc. already baked into the original responsive rules — carried forward)
- Accent color system switched from the old red/orange "press" + gold "divine" duo to a single pink→purple→blue gradient matching the new logo, since there's only one engine now

---

## 3. Settings Schema

Stored in `localStorage` key `promptlab_settings_v2` as JSON.

| Field | Type | Default | Description |
|---|---|---|---|
| `gem` | string | `""` | Gemini API key |
| `model` | string | `"gemini-2.5-flash"` | Selected Gemini model ID |
| `window` | number | `24` | News lookback window in hours (24 or 48) |
| `region` | string | `"Karnataka"` | Region for news search |

`useSearch` and `lookahead` were removed — Memes always forces search ON for Step 1 (as it did since v9.1), and there is no Vedic Engine left to need a lookahead window.

### Other localStorage Keys

| Key | Purpose |
|---|---|
| `promptlab_model_cache` | `{ models:[{id,tier,group}], fetchedAt }` — 7-day TTL live model list |
| `promptlab_console_collapsed_v3` | `{ memes:bool }` — console collapse state |
| `promptlab_lane_collapsed` | `{ press:bool }` — lane collapse state |
| `promptlab_settings_groups` | `{ gemini:bool, memestyle:bool, runopts:bool, danger:bool }` |

---

## 4. Gemini API Call Architecture

Unchanged from v9.1 in mechanics (retry/backoff/fallback-chain logic, thinking disabled on 2.5 models, JSON repair/extraction, model list refresh) — see prior version history below. Token budgets simplified to match the 2-call pipeline:

```js
const MAX_TOKENS = { search:1800, structure:3400 };
```

---

## 5. File Delivery Protocol

- **Output file:** `/mnt/user-data/outputs/PromptLab_Instagram_Updated.html`
- **Working copy:** `/home/claude/PromptLab_Instagram_Updated.html`
- **Validation:** Python extracts `<script type="module">` → `node --check extracted.mjs` before every delivery
- **Assets:** New logo + background are embedded as base64 data URIs (logo resized to 320×320 PNG, background resized to 900px-wide JPEG q78) to keep this a single portable file with no external image hosting dependency

---

## 6. Development Rules

1. No building until user says BUILD, Update, or Change
2. Strict "no guessing" on missing specs — ask clarifying questions first
3. Targeted edits over full rewrites where possible (this rebuild was a full-file rewrite by necessity — Vedic removal + engine restructuring touched the majority of the file)
4. Syntax check mandatory before every file delivery
5. Single output file — all changes go to `PromptLab_Instagram_Updated.html`
6. MD file updated on major feature additions (this update)

---

## 7. Feature History (Chronological) — abbreviated, see prior README versions for full v1–v9.1 history

| Feature | Status |
|---|---|
| ...all v1–v9.1 history (dual-call Memes, Vedic Engine, Virality Agent, etc.) | ✅ Shipped, superseded |
| **v10:** Vedic Engine removed entirely (tab, panel, engine, settings) | ✅ Shipped |
| **v10:** Kannada Film category dropped | ✅ Shipped |
| **v10:** Memes rebuilt on kartanews "4×3" single-pipeline logic (2 Political Controversy + 2 Breaking News, 3 image prompts/story) | ✅ Shipped |
| **v10:** Virality Agent (3rd call) removed | ✅ Shipped |
| **v10:** Caption (tagline + summary ≤4 lines + 4 hashtags) generated once per story, not per image | ✅ Shipped |
| **v10:** New logo + gradient background embedded as base64, mobile-first responsive rebuild | ✅ Shipped |
