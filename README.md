# PromptLab Studios — Meme Prompt Engine

**Version 13** · single-file web app · `PromptLab_Instagram_Updated_13.html`

Pulls live regional news with Google Search grounding, picks the 4 sharpest stories, and writes a **3-frame continuation sequence** of image prompts for each one — designed to be posted in order as a single Instagram / Facebook Reel.

Everything runs in the browser. No server, no build step, no backend. Your Gemini API key never leaves `localStorage`.

---

## What changed in v13

### 1. Prompt engine: linked set → continuation sequence

**Before:** the 3 prompts per story were 3 *camera angles* of one moment (close-up / exaggerated / wide).

**Now:** the 3 prompts are 3 *consecutive frames* of one short story, meant to be posted in order:

| Frame | Prompt field | Story beat |
|---|---|---|
| **1** | `hook_prompt` | Opening beat — the dramatic moment that stops the scroll |
| **2** | `meme_prompt` | What happens **next** in that same scene — the satirical/comedic reaction to frame 1 |
| **3** | `summary_prompt` | What happens **after** frame 2 — the calm aftermath that explains the news and closes the loop |

The camera treatment is now explicitly held **constant** across the three frames so they cut together smoothly. Only the subject's action, expression, and on-image text advance.

### 2. New field: `visual_anchor`

Each story now produces a 25–45 word "continuity anchor" locking the subject's identity, outfit, location, props, art style, palette, and lighting. That exact sentence is repeated **verbatim inside all 3 prompts**, which is what stops the image model from drifting between frames.

It renders as its own copyable block above the three prompts.

### 3. New UI

- Sequence flow header: `1 Hook → 2 Meme → 3 Summary · post in this order`
- Numbered badges (1 / 2 / 3) on each prompt block
- **Copy all 3 frames in order** button — copies the anchor plus all three prompts, labelled and separated, ready to paste into an image tool one after another

### 4. Dead code removed

| Removed | Why |
|---|---|
| `--orange` CSS variable | Declared, never referenced |
| `.prompt-group-label` rule | No element ever carried the class |
| `.btn-ghost .spin` rule | `.spin` is only ever injected into the primary button |
| `#gemDot` id | Never read by JS; CSS targets `.dot` |
| `<div id="panel-memes">` wrapper | No styling, no JS reference — leftover from the multi-panel build |
| `busyFlag()`, `stopBtnId()` | Single-branch indirection over one boolean / one constant |
| `kind` param on `laneEl` / `countEl` / `setCount` / `renderSkeletons` / `renderError` | Ignored — always resolved to the same element |
| `tabName` param on `clearWorkspace()` | Ignored |
| `KIND_META.emptyIcon` | Byte-identical duplicate of `KIND_META.icon` |
| "Output must match the provided response schema" | Referred to a `responseSchema` that was never sent to the API |

### 5. UI-flow bugs fixed

| # | Issue | Fix |
|---|---|---|
| 1 | The News-window toggle mutated live config instantly while every other field only applied on **Save** | All drawer edits are now pending; **Save** commits, closing discards |
| 2 | Closing the drawer left unsaved text in the inputs, so reopening showed values that didn't match saved config | `syncDrawerFromCfg()` reverts the drawer on close |
| 3 | After a model-list refresh, if your saved model wasn't on your key, the dropdown silently switched but runs still fired at the old model | `buildModelSelect()` now syncs `cfg.model` + header chip and logs the switch |
| 4 | Refreshing the model list wiped the RPM / tier descriptions from the dropdown | Descriptions restored via a `MODEL_NOTES` map |
| 5 | **Erase** reset the config but left the dropdown on the old model — chip and dropdown disagreed | Dropdown and chip explicitly re-synced |
| 6 | Clear's tooltip promised to reset "counters", but the API-call counter never reset | `apiCalls` and `#callCount` now reset on Clear and Erase |
| 7 | A failed re-run destroyed the good results from the previous run | Results are snapshotted and restored on error or on Stop |
| 8 | Stop logged "finishing current step then halting" — but abort is immediate | Copy corrected to "Stopping the run…" |
| 9 | Prompt blocks toggled via a `<label>` click handler — unreachable by keyboard, invisible to screen readers | Now `role="button"`, `tabindex="0"`, `aria-expanded`, Enter/Space support |
| 10 | Mobile collapse state was decided once at render time; rotating the phone did nothing | `matchMedia` listener re-applies on breakpoint change, and never overrides a block you toggled yourself |
| 11 | Under `file://`, Run was disabled but **Refresh models** wasn't — it failed with a confusing toast | Both disabled, plus a visible inline banner instead of a tooltip-only hint |
| 12 | Copy button could get stuck showing "Copied" if clicked twice quickly | Timer cleared, original markup stored on the element |
| 13 | Copying an empty prompt silently "succeeded" | Now warns "Nothing to copy yet" |
| 14 | `structure` token budget of 3400 was too tight for 4 stories × 3 prompts, triggering a `MAX_TOKENS` retry (a wasted call + 6s wait) | Raised to 6000; the schema also grew by `visual_anchor` |
| 15 | Fewer than 4 usable stories, or stories missing a frame, passed silently | Both now logged as warnings |
| 16 | `window.matchMedia` unguarded at module top level | Guarded — a missing API can't take down the whole app |

---

## Installation

The app is one self-contained HTML file. **It will not work when opened directly from your file system** (`file://`) — browsers block Gemini requests from local files. Pick one of the three options below.

### Prerequisite: get a Gemini API key

1. Go to <https://aistudio.google.com/app/apikey>
2. Create a key (free tier is fine)
3. Keep it handy — you'll paste it into Settings

### Option A — Local server (fastest, for testing on a computer)

```bash
# 1. Put the file in a folder
mkdir promptlab && cd promptlab
cp /path/to/PromptLab_Instagram_Updated_13.html index.html

# 2. Serve it
python3 -m http.server 8000

# 3. Open it
#    http://localhost:8000
```

To open it from your phone on the same Wi-Fi, find your computer's LAN IP and use `http://192.168.x.x:8000` instead.

```bash
# macOS / Linux
ipconfig getifaddr en0        # macOS
hostname -I | awk '{print $1}' # Linux
```

### Option B — GitHub Pages (free, permanent URL)

```bash
mkdir promptlab && cd promptlab
git init
cp /path/to/PromptLab_Instagram_Updated_13.html index.html
git add index.html
git commit -m "PromptLab v13"
git branch -M main
git remote add origin https://github.com/<your-username>/promptlab.git
git push -u origin main
```

Then in the repo: **Settings → Pages → Source: `main` / root → Save**.
Your app appears at `https://<your-username>.github.io/promptlab/` in about a minute.

> The page is public. Your API key is **not** — it lives only in your own browser's `localStorage`, never in the file.

### Option C — Netlify / Vercel drag-and-drop

1. Rename the file to `index.html`
2. Drop it on <https://app.netlify.com/drop> (or `vercel deploy` in the folder)
3. Use the URL it hands back

### Verify the install

Open the URL. You should see:

- Header pill reading **Gemini · 2.5 Flash** with a **red** dot (no key yet)
- **No** amber `file://` banner under the Run button

If you see the amber banner, you're still on `file://` — go back and use one of the options above.

---

## First-run setup

1. Tap the **Settings** gear (top right)
2. Open **Brain · Gemini** → paste your API key
3. *(Optional)* Tap **↻ Refresh** to pull the live model list for your key. Models your key can't reach are dropped from the dropdown.
4. Open **Region** → set your region (default `Karnataka`). This drives both the news search and the hashtags.
5. Open **News window** → choose **Last 24 hrs** or **Last 48 hrs**
6. Tap **Save**

The header dot turns **green** once a key is saved.

> Nothing in the drawer takes effect until you press **Save**. Closing the drawer with Esc, the X, or a tap outside discards your edits.

---

## Usage

### Basic run

1. Tap **🗞️ Run Meme Crew**
2. Watch the activity console: `Fetching fresh headlines…` → `News retrieved` → `Deduplicating, grading & drafting…`
3. In ~20–60 seconds you get 4 cards

Each card contains:

- Headline + category pill (**Political Controversy** in pink, **Breaking News** in blue)
- **Continuity anchor** — the locked subject/scene description
- **1 · Hook · opening beat** — 1:1 prompt
- **2 · Meme · reaction beat** — 1:1 prompt
- **3 · Summary · closing beat** — 1:1 prompt
- **🎬 Copy all 3 frames in order**
- **Caption** — tagline + up to 4 summary lines + exactly 4 hashtags

Tap any block header to collapse or expand it. On phones the blocks start collapsed.

### Making a Reel — the intended workflow

```
1. Tap "🎬 Copy all 3 frames in order"
2. Paste into your image tool (Nano Banana, Imagen, Midjourney, Firefly…)
3. Generate frame 1 → save
4. Generate frame 2 → save    ← same subject, same scene, story moves forward
5. Generate frame 3 → save
6. Drop all three into Reels / your editor, ~2s each, in order 1 → 2 → 3
7. Tap "Copy" on the Caption block → paste as the post caption
```

Because all three prompts embed the identical `visual_anchor` sentence, the three images share one character, one location, and one art style — so they read as one continuous story rather than three unrelated posts.

### Example output shape

**Continuity anchor**

> A middle-aged politician in a crisp white kurta with black-rimmed glasses, standing on the wide stone steps of a government building, bold pop-art illustration style, magenta and teal palette, hard afternoon light.

**Frame 1 — Hook**

> `Generate a 4k, Hook style Image (1:1 ratio), Text Language Kannada + English (Mix), Continuation frame 1 of 3 - A middle-aged politician in a crisp white kurta… ` *(anchor repeated verbatim)* `…mid-stride, mouth open, papers flying from his hands as reporters' mics crowd in. Bold Kannada headline text across the top.`

**Frame 2 — Meme**

> `Generate a 4k, Memes style Image (1:1 ratio), Text Language Kannada + English (Mix), Continuation frame 2 of 3, continuing directly from frame 1 - ` *(same anchor)* `…now frozen mid-panic, exaggerated cartoon sweat drops, the same scattered papers now settling around his feet. Punchline text in Kannada + English.`

**Frame 3 — Summary**

> `Generate a 4k, Summary style Image (1:1 ratio), Text Language Kannada + English (Mix), Continuation frame 3 of 3, continuing directly from frame 2 - ` *(same anchor)* `…standing calmly, papers stacked neatly, reporters gone, one explanatory caption block in the lower third.`

**Caption**

```
Big tagline naming the real event
Factual context line one.
Factual context line two.
#KarnatakaPolitics #BengaluruNews #TrendingNow #NewsUpdate
```

### Other controls

| Control | What it does |
|---|---|
| **🛑 Stop** | Aborts mid-run. Your previous results are restored. |
| **🧹 Clear** | Clears results, console log, and the API-call counter. Keeps your settings. |
| **Erase saved key & preferences** | Wipes key, settings, model cache, results, and log from this browser. Irreversible. |
| **Console header** | Tap to collapse/expand the activity log |
| **Lane header chevron** | Tap to collapse/expand the results lane |
| **Esc** | Closes the settings drawer, discarding unsaved edits |

---

## Configuration reference

| Setting | Default | Notes |
|---|---|---|
| Gemini API key | *(empty)* | Stored in `localStorage`, never transmitted anywhere except Google's API |
| Generation model | `gemini-2.5-flash` | Free-tier keys should stay on Flash or Flash Lite |
| Region | `Karnataka` | Drives the news search and hashtag flavour |
| News window | 24 hrs | Or 48 hrs |

### Models

| Model | Tier | RPM | Notes |
|---|---|---|---|
| `gemini-3.1-flash-lite` | Free | 30 | Best free throughput |
| `gemini-3.5-flash` | Free | 15 | Most intelligent free option |
| `gemini-3-flash-preview` | Free | 10 | Preview |
| `gemini-2.5-flash` | Free | 15 | **Default** |
| `gemini-2.5-flash-lite` | Free | 30 | Budget |
| `gemini-3.1-pro-preview` | Paid | — | Paid keys only |
| `gemini-2.5-pro` | Paid | — | Paid keys only |

On a rate limit, the app automatically retries on `gemini-2.5-flash-lite`, then `gemini-3.1-flash-lite`.

### `localStorage` keys

| Key | Contents |
|---|---|
| `promptlab_settings_v2` | API key, model, region, news window |
| `promptlab_model_cache` | Live model list + fetch timestamp (7-day TTL) |
| `promptlab_settings_groups` | Which settings sections are collapsed |
| `promptlab_console_collapsed_v3` | Console collapsed state |
| `promptlab_lane_collapsed` | Results lane collapsed state |
| `promptlab_file_warning_shown` | *(sessionStorage)* file:// warning dismissed |

---

## How it works

```
Run Meme Crew
      │
      ├─► Call 1: searchPrompt()      · Google Search grounding ON  · 1800 tokens
      │   "Find <region> news from the last <window> hours,
      │    2 categories, plain prose"
      │            │
      │            ▼
      │      raw news text
      │            │
      ├─► Call 2: structurePrompt()   · JSON mode ON · 6000 tokens
      │   dedupe → grade virality/severity → pick 4 (2+2)
      │   → write visual_anchor + 3 sequenced prompts + caption
      │            │
      │            ▼
      │      raw JSON array
      │            │
      ├─► extractArray() → repairJsonString() if needed
      ├─► validateCards() → drop objects with no usable fields
      └─► renderCards() → 4 cards
```

Two API calls per run (plus retries). Retries and fallback-model attempts each increment the counter shown in the console header.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Amber banner, Run greyed out | Opened as `file://` | Serve over http:// — see Installation |
| "Add your Gemini API key in Settings first" | No key saved | Settings → paste key → **Save** |
| "Invalid Gemini API key" | Wrong or revoked key | Regenerate at aistudio.google.com |
| "RPM limit — try Flash Lite" | Too many runs too fast | Wait ~60s, or switch to a 30-RPM Flash Lite model |
| "Daily quota hit — resets midnight PT" | Free daily cap reached | Switch model or wait |
| "Model output wasn't valid JSON even after repair" | Model drifted | Run again; if it repeats, switch to `gemini-2.5-flash` |
| Prompts don't share a subject | Model ignored the anchor | Run again — a re-roll usually fixes it. Check the Continuity anchor block is populated. |
| Settings changes don't stick | Drawer closed without saving | Press **Save**, not Esc/X |
| Console shows "X isn't available on this key" | Saved model isn't on your key | Already auto-switched — press **Save** to persist the new choice |

---

## Browser support

Chrome / Edge 90+, Safari 15+, Firefox 90+, and mobile equivalents. Requires `fetch`, `AbortController`, `matchMedia`, and ES modules. Clipboard writes fall back to `document.execCommand("copy")` on older Safari.

---

## Privacy

- The API key lives in your browser's `localStorage` and is sent only to `generativelanguage.googleapis.com`
- No analytics, no telemetry, no backend
- Nothing is stored server-side — hosting the file publicly does **not** expose your key
- **Erase saved key & preferences** removes everything from this browser

---

## Known limitations

- **No `responseSchema` is sent to the API.** Output structure is enforced by prompt instruction plus a JSON repair pass, not by Gemini's structured-output feature. Migrating to a real `responseSchema` would harden this further — see *Open questions* below.
- **The settings drawer has no focus trap.** Focus moves into the drawer on open and back to the gear on close, but Tab can still reach the page behind it.
- **Anchor consistency isn't verified.** The app doesn't check that the three prompts actually contain the anchor text — it relies on the model following instructions.
- **Card order isn't controlled.** Stories render in the order the model returns them, not sorted by virality or category.
- **On ≤480px screens the card title is visually reordered below the index badge** while DOM order is unchanged — a minor mismatch for screen readers.

---

## Open questions / assumptions made

These were inferred while implementing the continuation change. Say the word and any of them can be reversed:

1. **`visual_anchor` was added as a new field.** You asked for continuation rather than different angles; a repeated verbatim anchor is the most reliable way to hold subject and scene steady across three separate image generations. It costs a few hundred output tokens per run. Removable if you'd rather keep the payload lean.
2. **The prompt prefixes were kept exactly as they were** (`Generate a 4k, Hook style Image (1:1 ratio), Text Language Kannada + English (Mix), `). The frame marker was appended *after* them rather than replacing them.
3. **Aspect ratio is still 1:1.** Reels are usually 9:16 — if you want the sequence generated at 9:16, that's a one-line change in three places.
4. **The three style names are unchanged** (Hook / Memes / Summary). If your image tool responds better to a single consistent style keyword across all three frames, that's worth testing.
5. **`responseSchema` was not introduced.** The false "provided response schema" line was removed rather than made true, because a malformed schema would hard-fail every run and it can't be validated against the live API from here.
6. **Four stories, two per category, remains the fixed shape.**
