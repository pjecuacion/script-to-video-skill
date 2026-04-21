---
name: script-to-video
description: Full pipeline — topic/script/text file → OmiVoice TTS (Prince's voice) → Whisper word-level timestamps → HyperFrames synced video composition. Use when asked to "make a video about", "create a voiced video", "script to video", "voice and animate", "narrate and compose", or "turn this into a video with audio". Also handles "catalog showcase" (transcribe existing video or audio file → apply all scene types) and "talking-cut" (talking head video + graphic cutaway scenes).
---

# Script-to-Video Pipeline

Orchestrates four skills end-to-end:

1. **Script** — prepare the voiceover text
2. **Synthesize** — OmiVoice TTS → WAV
3. **Transcribe** — word-level timestamps via HyperFrames CLI
4. **Storyboard** — assign a unique scene type to every sentence
5. **Compose** — HyperFrames full video, one richly designed scene per sentence

Default output: **1920×1080 landscape**, dark premium style, audio voiced as Prince.

**The goal:** every sentence gets a visually distinct treatment. No two consecutive scenes look alike. Use all of HyperFrames' scene types — bar charts, flow steps, stat reveals, counter-ups, list reveals, progress rings, callouts, kinetic text, and more. This is a showcase of what the platform can do.

**Public repo note:** this version uses placeholder local paths and environment variables for anything machine-specific or secret. Replace those values with paths that exist on your system before running the pipeline.

---

## Catalog Showcase Mode

**Trigger phrases:** "catalog showcase", "showcase all scene types", "show me all the scenes"

This mode skips TTS synthesis. It takes an **existing video or audio file** as the source, transcribes it, and builds a HyperFrames composition that uses as many unique scene types as possible — one per sentence, no repeats.

**How to invoke:**
> "catalog showcase from `<path-to-video-or-audio>`"

**Source type detection:**
- **Video file** (`.mp4`, `.mov`, `.mkv`, etc.) → extract audio with ffmpeg first, then transcribe
- **Audio file** (`.wav`, `.mp3`, `.m4a`, `.aac`, etc.) → transcribe directly, skip ffmpeg extraction

**What I'll do (video source):**
0. Detect source FPS with ffprobe — use it for the render command
1. Extract mono 16kHz WAV from the source video with ffmpeg
2. Transcribe with Whisper (`small.en` model)
3. Map each sentence to a unique scene type from the catalog (no two consecutive types the same)
4. Build the full composition in `C:\openclaw_files\projects\<slug>\`
5. Lint → Render → deliver MP4 (`npx hyperframes render --fps <mapped_fps> --output <slug>.mp4`, where mapped_fps is 24/30/60 — the nearest HyperFrames-supported value)

**What I'll do (audio source):**
0. Skip ffmpeg extraction — use the audio file directly
1. Transcribe with Whisper (`small.en` model)
2. Map each sentence to a unique scene type from the catalog (no two consecutive types the same)
3. Copy audio into the project's `audio/` folder
4. Build the full composition in `C:\openclaw_files\projects\<slug>\`
5. Lint → Render → deliver MP4 at `--fps 60` (default, since there is no source video FPS to detect)

**Detect source FPS before step 1 (video source only):**
```bash
ffprobe -v quiet -select_streams v:0 \
  -show_entries stream=avg_frame_rate -of csv=p=0 source.mp4
# → outputs e.g. "30/1" or "60000/1001". Round to nearest int → SRC_FPS (e.g. 30, 60, 25, 50).
# HyperFrames render only accepts 24, 30, or 60. Map SRC_FPS to nearest supported value:
#   SRC_FPS ≤ 26  → --fps 24
#   SRC_FPS 27-45 → --fps 30
#   SRC_FPS ≥ 46  → --fps 60
# Pass this to the render command: npx hyperframes render --fps <mapped_fps> --output <slug>.mp4
# For audio-only sources: always use --fps 60
```

**Reference project:** `C:\openclaw_files\projects\yt-create-app\` — 15 scenes, 15 unique types, 51.5s, Shadow Cut theme.

**Scene type priority order for showcase builds** (use as many as possible before repeating):
title-card → kinetic-impact → kinetic-slam → kinetic-text → quote-card → callout → comparison → bar-chart → stat-reveal → counter-up → split-layout → doodle-split → flow-steps-text → progress-ring → list-reveal-words → outro-card → list-reveal → flow-steps → icon-grid → comparison-verdict → cta-callout

---

## Talking-Cut Mode

**Trigger phrases:** "talking-cut from `<path>`", "mix talking head with graphics", "cutaway style"

Takes a **talking head video** and alternates between showing the raw footage and HyperFrames graphic cutaways. The VO audio continues during graphic scenes — same as how YouTube creators cut to a chart or kinetic text while still talking.

**How to invoke:**
> "talking-cut from `<path-to-video>`"

**Architecture (critical rules):**
- The full source video sits in a **plain non-timed wrapper div** (NOT a `.clip`) — `z-index:1`
- Graphic cutaway scenes are `.clip` divs at `z-index:2` — they cover the video during their window
- When no clip is active ("face cam" moments), the video shows through naturally
- **No pre-cutting of segments needed** — video plays continuously; clips overlay on top
- Source video MUST be re-encoded with dense keyframes before use (see Sync-Safe Encode below)
- Audio is extracted separately as WAV and used as the HyperFrames audio track (video element is `muted`)
- **Minimum gap between cutaway clips: ≥3 seconds** — gaps shorter than 3s feel like jarring micro-flashes of face-cam. If two storyboard scenes are close together, push the later one out until the gap is ≥3s and reduce its duration to compensate.

### ⚠️ Sync-Safe Encode — CRITICAL for lip sync (confirmed fix)

**The #1 cause of lip-sync drift in talking-cut:** extracting audio from a VFR (variable frame-rate) source while re-encoding video to a different CFR. VFR→CFR conversion shifts video timestamps; the WAV retains original timing → progressive drift accumulates throughout the composition.

**The fix — detect source FPS first, then extract audio from the SAME CFR encode as the video:**

```bash
# Step 0: Detect source FPS — always run this before encoding
ffprobe -v quiet -select_streams v:0 \
  -show_entries stream=avg_frame_rate -of csv=p=0 source.mp4
# → outputs a fraction like "30/1" or "60000/1001"
# Round to nearest integer → SRC_FPS (e.g. 30, 60, 25, 24)
# Substitute SRC_FPS into all -r / -g / -keyint_min flags below.

# Step 1: Re-encode source to SRC_FPS CFR WITH audio preserved
ffmpeg -y -i source.mp4 -c:v libx264 -r SRC_FPS -g SRC_FPS -keyint_min SRC_FPS \
  -movflags +faststart -c:a copy assets/source-kf-audio.mp4

# Step 2: Extract WAV from THAT same encode (not from the original!)
ffmpeg -y -i assets/source-kf-audio.mp4 \
  -vn -ac 1 -ar 44100 -c:a pcm_s16le audio/<slug>.wav

# Step 3: Strip audio for the muted face-cam video (same timing baseline)
ffmpeg -y -i assets/source-kf-audio.mp4 -c:v copy -an assets/source-kf.mp4
```

**Why this works:** All three outputs come from the same CFR encode — video timestamps and audio timestamps share the same baseline. No drift possible. Using the source FPS (instead of a fixed 30fps) also preserves the original motion cadence and avoids unnecessary frame interpolation or drop.

**data-duration — use the actual video file duration, NOT Whisper's estimate:**
```bash
# Get correct duration
ffprobe -v quiet -show_entries format=duration -of csv=p=0 assets/source-kf.mp4
```
Use this value for ALL `data-duration` attributes (root div, audio, video, s10 outro). Whisper's internal duration estimate is often off by 2–4 seconds and causes over-run at the end.

**Track index rules for talking-cut:**
- Face-cam `<video>` → `data-track-index="0"`
- `<audio data-main-audio>` → **no `data-track-index` attribute** (omit it entirely — adding it causes `overlapping_clips_same_track` lint error when video is also on track 0)
- Cutaway `.clip` divs → `data-track-index="1"`

**Typical scene pattern for a 10–15s video:**
```
0–4s    → face cam (no clip active)
4–7.5s  → graphic cutaway: kinetic-text, stat-reveal, bar-chart, etc.
7.5–10.5s → face cam
10.5–end  → graphic cutaway: callout or outro-card
```

**Reference project:** `C:\openclaw_files\projects\talking-cut-sample\` — 12.89s, 2 cutaways (kinetic-text + callout), Shadow Cut theme.

**Video wrapper pattern (copy exactly):**
```html
<!-- Plain non-timed wrapper — NOT a .clip div -->
<!-- No autoplay — HyperFrames owns playback via data-track-index -->
<div id="v-wrap" style="position:absolute;inset:0;z-index:1;">
  <video id="bg-video" src="assets/source-kf.mp4" muted playsinline
         data-start="0" data-duration="TOTAL_DURATION" data-track-index="0"
         style="width:100%;height:100%;object-fit:cover;"></video>
</div>
```

**Audio element pattern (omit data-track-index):**
```html
<!-- data-main-audio identifies this as the main audio track -->
<!-- Do NOT add data-track-index — it will conflict with the video on track 0 -->
<audio id="main-audio" data-start="0" data-duration="TOTAL_DURATION"
       src="audio/<slug>.wav" data-main-audio></audio>
```

**Graphic cutaway clip pattern:**
```html
<!-- z-index:2 covers the video when this clip is active -->
<div id="sN" class="clip" data-start="4.0" data-duration="3.5" data-track-index="1"
     style="z-index:2;">
  <!-- any scene type from the catalog -->
</div>
```

---

## Step 0: Derive a Slug

## Step 1: Script Preparation

Three input modes:

### Topic only
Write a voiceover script. Target ≤250 words (~90 seconds at natural speech pace).
Structure: hook (1–2 sentences) → body (3–4 paragraphs) → close (1 sentence).
Write for the ear: short sentences, no jargon, natural rhythm.

### Script text provided
Use as-is. Clean up punctuation if needed for natural TTS delivery.

### File path provided
Read the file. Treat contents as the script.

---

## Step 2: OmiVoice Synthesis

### Check server
```python
import urllib.request
try:
    urllib.request.urlopen("http://127.0.0.1:8261", timeout=3)
    print("Server up")
except:
    print("Server offline — start it")
```

If offline, run your local OmiVoice start script, for example `path\to\OmniVoice\start.bat`
Wait ~10 seconds before proceeding.

### Synthesize
```python
import os
os.environ["PYTHONIOENCODING"] = "utf-8"

from gradio_client import Client, handle_file

OUTPUT_DIR = r"C:\\openclaw_files\outputs"
os.makedirs(OUTPUT_DIR, exist_ok=True)

client = Client("http://127.0.0.1:8261")
result = client.predict(
    text=SCRIPT_TEXT,
    lang="Auto",
  ref_aud=handle_file(r"<path-to-reference-voice-sample>.mp3"),
    ref_text="",
    instruct="",
    ns=32,
    gs=2.0,
    dn=True,
    sp=1.0,
    du=0,
    pp=True,
    po=True,
    api_name="/_clone_fn",
)

# result is a filepath, list, or dict
if isinstance(result, (list, tuple)):
    src = result[0]
elif isinstance(result, dict) and "value" in result:
    src = result["value"]
else:
    src = result

import shutil
wav_path = os.path.join(OUTPUT_DIR, f"{SLUG}.wav")
shutil.copy(str(src), wav_path)
print(f"Audio: {wav_path}")
```

Output WAV: `C:\\openclaw_files\outputs\<slug>.wav`

---

## Step 3: Transcription

```bash
npx hyperframes transcribe "C:\\openclaw_files\outputs\<slug>.wav" --model small.en
```

This produces `transcript.json` in the current directory. Move/copy it into the project folder after scaffolding (Step 4).

> **Why `small.en`:** TTS audio is clean English speech — no background noise, no music. `small.en` is fast and accurate for this use case.

### Quality Check (mandatory)

After transcription, load the JSON and run:

```js
var raw = JSON.parse(transcriptJson);
var words = raw.filter(function (w) {
  if (!w.text || w.text.trim().length === 0) return false;
  if (/^[♪♫♬♭♮♯\u{1F3B5}\u{1F3B6}]+$/u.test(w.text)) return false;
  if (/^(huh|uh|um|ah|oh)$/i.test(w.text) && w.end - w.start < 0.1) return false;
  return true;
});
```

If >20% entries are music tokens or garbled, retry with `--model medium.en`.

---

## Step 3.5: Storyboard

**Do this before touching index.html.** Split the script into individual sentences. For each sentence, find its start/end time from the transcript (match the first and last word). Assign a scene type using the rules in Step 4c. Produce a storyboard JSON.

```python
import json, re

SCRIPT = """<your script here>"""
WORDS = json.load(open("transcript.json"))  # [{text, start, end}, ...]

# Split script into sentences
sentences = [s.strip() for s in re.split(r'(?<=[.!?])\s+', SCRIPT) if s.strip()]

def find_time(word_text, words, from_index=0, last=False):
    """Find start/end time of a word in the transcript."""
    matches = [w for i, w in enumerate(words) if i >= from_index
               and w["text"].strip(".,!?").lower() == word_text.strip(".,!?").lower()]
    if not matches: return None
    return matches[-1] if last else matches[0]

storyboard = []
search_from = 0
for i, sentence in enumerate(sentences):
    words_in_sentence = sentence.split()
    first_word = words_in_sentence[0] if words_in_sentence else ""
    last_word = words_in_sentence[-1] if words_in_sentence else ""
    start_w = find_time(first_word, WORDS, from_index=search_from)
    end_w = find_time(last_word, WORDS, from_index=search_from, last=True)
    storyboard.append({
        "id": i + 1,
        "sentence": sentence,
        "type": "TBD",  # fill in from type assignment rules in Step 4c
        "startTime": start_w["start"] if start_w else None,
        "endTime": end_w["end"] if end_w else None,
    })
    if start_w:
        search_from = WORDS.index(start_w)

print(json.dumps(storyboard, indent=2))
```

After generating the storyboard, **manually assign scene types** from Step 4c rules. Print the final storyboard to confirm no two consecutive scenes share a type before proceeding.

---

## Step 3.6: Fetch Pexels Assets

> **Only run this step if any scene has type `pexels-hero`.**
> Run it after scaffolding the project (Step 4a) so the `assets/` folder exists.

### Storyboard extension for pexels-hero scenes

Add a `pexels` object to any storyboard entry that uses this type:

```json
{
  "id": 3,
  "sentence": "Behind every great product is a human struggling with a real problem.",
  "type": "pexels-hero",
  "pexels": {
    "query": "developer coding laptop focus",
    "media": "video",
    "file": "assets/pexels-s3.mp4"
  },
  "startTime": 8.1,
  "endTime": 12.6
}
```

- `query` — plain English Pexels search. Be specific: prefer `"developer typing code dark office"` over `"coding"`.
- `media` — `"photo"` (static) or `"video"` (looping). Prefer video for emotional impact; photo for crisp product shots.
- `file` — output path (filled in by the fetch script below; do not invent — run the script).

### Python fetch script

Run once per project, after `npx hyperframes init`:

```python
import os, requests, shutil

PEXELS_API_KEY = os.environ.get("PEXELS_API_KEY")
if not PEXELS_API_KEY:
  raise RuntimeError("Set PEXELS_API_KEY before running the fetch script.")
HEADERS = {"Authorization": PEXELS_API_KEY}
ASSETS_DIR = r"C:\openclaw_files\projects\<slug>\assets"
os.makedirs(ASSETS_DIR, exist_ok=True)

def fetch_photo(query, scene_id):
    r = requests.get("https://api.pexels.com/v1/search",
                     params={"query": query, "per_page": 3, "orientation": "landscape"},
                     headers=HEADERS)
    photos = r.json().get("photos", [])
    if not photos:
        print(f"  [!] No photo for: {query}"); return None
    url = photos[0]["src"]["large2x"]   # 2560px wide, landscape
    out = os.path.join(ASSETS_DIR, f"pexels-s{scene_id}.jpg")
    with requests.get(url, stream=True) as resp:
        with open(out, "wb") as f:
            shutil.copyfileobj(resp.raw, f)
    print(f"  [photo] s{scene_id} → {out}")
    return out

def fetch_video(query, scene_id):
    r = requests.get("https://api.pexels.com/videos/search",
                     params={"query": query, "per_page": 3, "orientation": "landscape"},
                     headers=HEADERS)
    videos = r.json().get("videos", [])
    if not videos:
        print(f"  [!] No video for: {query}"); return None
    # Prefer HD (720p) — FHD may be too large for short clips
    files = videos[0]["video_files"]
    hd = next((f for f in files if f.get("quality") == "hd"), None) or files[0]
    out = os.path.join(ASSETS_DIR, f"pexels-s{scene_id}.mp4")
    with requests.get(hd["link"], stream=True) as resp:
        with open(out, "wb") as f:
            shutil.copyfileobj(resp.raw, f)
    print(f"  [video] s{scene_id} → {out}")
    return out

# ── Edit this list to match your storyboard pexels-hero scenes ──────────────
PEXELS_SCENES = [
    # (scene_id, query, media_type)
    (3,  "developer coding laptop dark",  "video"),
    (7,  "abstract technology neon data", "photo"),
]

print("Fetching Pexels assets...")
for scene_id, query, media in PEXELS_SCENES:
    if media == "video":
        fetch_video(query, scene_id)
    else:
        fetch_photo(query, scene_id)
print("Done.")
```

**Attribution note:** Pexels requires attribution when displaying images/videos publicly. Add a small credit in your project README or in the video description: *"Photos/videos provided by Pexels (pexels.com)"*.

---

## Step 4: HyperFrames Composition

### 4a. Scaffold the project

```bash
cd C:\\openclaw_files\projects
npx hyperframes init <slug> --audio "C:\\openclaw_files\outputs\<slug>.wav" --non-interactive
cd <slug>
```

Copy `transcript.json` into the project root.

### 4b. Visual Identity

Themes are defined in `themes/` next to this skill file (one JSON file per theme).
**Default theme: Shadow Cut** (`themes/shadow-cut.json`).

**If the user specified a style or mood** → use the closest theme:

| User says | Theme file |
|-----------|-----------|
| dark / cinematic / dramatic | `shadow-cut` |
| light / white / handwriting / casual | `open-page` |
| futuristic / AI / cyberpunk / neon / tech | `neon-tokyo` |
| hacker / dev / coding / terminal / CLI | `terminal-green` |
| editorial / journalism / newspaper | `broadsheet` |
| technical / engineering / process / spec | `blueprint` |
| luxury / premium / finance / gold | `velvet-standard` |
| creator / lifestyle / sunset / personal brand | `dusk-gradient` |
| SaaS / product / clean / modern / corporate | `frost` |
| bold / raw / punchy / hot take / viral | `brutalist` |

**If the user did not specify a style** → use **Shadow Cut** (dark canvas, high contrast, cinematic — works universally).

**All saved themes** (read JSON for full CSS values):

| Theme | Font | BG | Accent |
|-------|------|----|--------|
| `shadow-cut` | Outfit 900 | `#0a0a0a` | `#c0392b` deep red |
| `open-page` | Patrick Hand 400 | `#ffffff` | `#e63946` warm red |
| `neon-tokyo` | JetBrains Mono 800 | `#06061a` | `#00f5ff` cyan / `#ff2d78` pink |
| `terminal-green` | JetBrains Mono 700 | `#0d0d0d` | `#00ff41` phosphor green |
| `broadsheet` | Playfair Display 900 | `#f5f0e8` | `#1c1c1c` ink (no colour) |
| `blueprint` | IBM Plex Mono 700 | `#0a1628` | `#4da6ff` electric blue | **Signature:** dashed-border spec-box (`border:2px dashed rgba(77,106,153,0.4); background:#0d1f3c`) wrapping content + `// ANNOTATION` label above. Grid bg on every scene. REQUIRED on title-card. |
| `velvet-standard` | Cormorant Garamond 300 | `#0a0a0a` | `#c9a84c` gold |
| `dusk-gradient` | Syne 800 | `#1a0533` gradient | `#ff6b35` orange |
| `frost` | Inter 700 | `#ffffff` | `#1e3a5f` deep blue |
| `brutalist` | Bebas Neue 400 | `#ffffff` | `#ff0000` pure red |

Write a minimal `DESIGN.md` in the project root before touching `index.html`.

### 4c. Scene Structure — Sentence-Level Variety

**One sentence = one scene.** Parse the script into individual sentences. For each sentence, pick a scene type from the Scene Type Catalog (see Appendix). No two consecutive scenes may share the same type. Vary visual weight: heavy (full-bleed, kinetic, bar chart) → lighter (quote, stat) → heavy.

**Mandatory scene map (produce this JSON before writing any HTML):**

```json
{
  "slug": "<slug>",
  "totalDuration": 0,
  "scenes": [
    {
      "id": 1,
      "sentence": "The opening hook sentence.",
      "type": "title-card",
      "startWord": "<first word from transcript>",
      "endWord": "<last word from transcript>",
      "startTime": 0.0,
      "endTime": 4.2,
      "note": "Why this type was chosen"
    }
  ]
}
```

**Type assignment rules — apply in this order:**

| Content signal | Assign type |
|---|---|
| First sentence / topic intro | `title-card` |
| Last sentence / CTA / farewell | `outro-card` |
| Contains a number (%, count, stat) alone | `stat-reveal` |
| Contains a number + counting up context | `counter-up` |
| Lists 3+ items ("X, Y, and Z" / "first…second…third") | `list-reveal` |
| Describes a process / steps / sequence | `flow-steps` |
| Contains "vs" / "compared to" / two contrasting sides | `comparison` |
| Strong single claim / punchline / declaration | `callout` |
| Quote, attribution, or key phrase worth reading slowly | `quote-card` |
| Definition, explanation, "what is" | `kinetic-text` |
| Percentage or proportion to illustrate | `progress-ring` |
| Multiple concepts that benefit from a visual grid | `icon-grid` |
| Has a clear text + supporting visual split | `split-layout` |
| Concrete noun, action, emotion, or metaphor that maps to a doodle illustration | `doodle-split` |
| Two short paired concepts with equal visual weight (need → answer, problem → solution) | `split-screen` |
| Data with relative values across categories | `bar-chart` |
| Part-of-whole breakdown, market share, budget split | `donut-chart` |
| Trend over time, growth curve, before/after over a timeline | `line-chart` |
| Compares two named products, plans, tools, or pricing tiers with multiple attributes | `product-comparison` |
| Sentence describing a 3D concept, tech, or "wow" moment worth a visual statement | `threejs-object` |
| Atmospheric, contextual, or emotional sentence where real-world imagery adds impact | `pexels-hero` |
| Presenter is on screen talking, with related text or data shown on the other side | `presenter-aside` |
| Multi-element dashboard, HUD, terminal output, or floating card UI | `hud-overlay` |
| Fallback | `kinetic-text` |

**No-repeat rule:** if the rule above would repeat the previous scene type, move to the next best fit. E.g., two consecutive stat sentences → first gets `stat-reveal`, second gets `counter-up` or `bar-chart`.

**Always include:**
- **Title scene** at t=0 (3s): `title-card` with the topic name
- **Caption overlay**: runs the full audio duration, sentence-level — uses **original script sentences** (NOT transcribed tokens)
- Scene durations from transcript timestamps (not estimated)

### 4d. index.html Structure

Each scene uses its own `data-composition-id` and maps to exactly one storyboard entry. The scene type determines the inner HTML and GSAP animation — see the **Scene Type Catalog** appendix for full patterns.

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=1920, height=1080">
  <script src="https://cdn.jsdelivr.net/npm/gsap@3.14.2/dist/gsap.min.js"></script>
  <style>
    /* ── Viewport fill — REQUIRED for full-screen render ── */
    /* Use fixed 1920x1080 pixels — NOT percentages. The renderer sets the
       viewport to match, and clip divs need position:absolute to fill it. */
    html, body { width: 1920px; height: 1080px; margin: 0; padding: 0; overflow: hidden; }
    /* ── Shared scene shell ─────────────────────────────── */
    [data-composition-id] {
      position: absolute; inset: 0; overflow: hidden;
    }
    /* ⚠️ CRITICAL — NEVER OMIT THIS RULE IN ANY PROJECT (main or sample):
       Without it, clip divs have auto height → scene-content height:100% collapses
       → all content renders in the top-left corner, not full-screen.
       Verified broken: product-comparison-sample (missing) → fixed by adding this. */
    .clip { position: absolute; inset: 0; }
    .scene-bg { position: absolute; inset: 0; background: #0a0a0a; }
    .scene-content {
      position: relative; width: 100%; height: 100%;
      padding: 120px 160px;
      display: flex; flex-direction: column; justify-content: center;
      gap: 24px; box-sizing: border-box;
    }
    /* ── Typography ─────────────────────────────────────── */
    .hed  { font-family: Outfit, sans-serif; font-weight: 900; font-size: 120px; color: #e2e2e2; line-height: 1.05; }
    .sub  { font-family: Outfit, sans-serif; font-weight: 400; font-size: 52px;  color: #8899aa; }
    .accent { color: #c0392b; }
    /* ── Captions ───────────────────────────────────────── */
    #captions { position:absolute; inset:0; pointer-events:none; z-index:50; }
    #cap-text  { position:absolute; bottom:56px; left:0; right:0;
                 text-align:center; font-family:'Outfit',sans-serif; font-size:34px; font-weight:500;
                 color:#fff; text-shadow:0 2px 14px rgba(0,0,0,.95);
                 padding:0 140px; opacity:0; line-height:1.4; }
    /* Subtitles off by default — HyperFrames overrides visibility:hidden on composition divs,
       so use a CSS class with !important to suppress captions when SUBTITLES_ON = false */
    #captions.subs-off #cap-text { opacity: 0 !important; }
  </style>
</head>
<body>

<!-- ROOT COMPOSITION -->
<div data-composition-id="<slug>" data-start="0" data-duration="TOTAL_DURATION"
     data-width="1920" data-height="1080">

  <!-- Audio track — id is REQUIRED; omitting it causes media_missing_id lint error -->
  <audio id="main-audio" data-start="0" data-duration="TOTAL_DURATION" data-track-index="0"
         src="audio/<slug>.wav" data-main-audio></audio>

  <!-- ── SCENE 1 (title-card) ──────────────────────────── -->
  <!-- IMPORTANT: scene divs use class="clip" — NOT data-composition-id -->
  <div id="s1" class="clip" data-start="0" data-duration="SCENE_1_DUR"
       data-track-index="1">
    <!-- see Scene Type Catalog: title-card -->
  </div>

  <!-- ── SCENE 2 (kinetic-text) ───────────────────────── -->
  <div id="s2" class="clip" data-start="SCENE_2_START" data-duration="SCENE_2_DUR"
       data-track-index="1">
    <!-- see Scene Type Catalog: kinetic-text -->
  </div>

  <!-- ... repeat for every storyboard entry ... -->

  <!-- ── Caption overlay (full duration) — has its own composition-id ── -->
  <!-- Do NOT use style="visibility:hidden" — HyperFrames overrides it on composition divs.
       Use class="subs-off" + CSS !important instead (see .subs-off rule above). -->
  <div id="captions" data-composition-id="captions-overlay"
       data-track-index="10" data-duration="TOTAL_DURATION" class="subs-off">
    <p id="cap-text"></p>
  </div>

  <script>
    window.__timelines = window.__timelines || {};

    // ── MAIN timeline: all scene entrance animations ──────
    var tl = gsap.timeline({ paused: true });

    // Scene 1 entrances (example title-card)
    tl.from("#s1-title",    { y: 60, opacity: 0, duration: 0.7, ease: "power3.out" }, 0.2);
    tl.from("#s1-subtitle", { y: 40, opacity: 0, duration: 0.5, ease: "power2.out" }, 0.5);
    // ... add entrances per scene type (see Catalog) ...

    window.__timelines["<slug>"] = tl;

    // ── CAPTIONS timeline ─────────────────────────────────
    // Always use the ORIGINAL script sentences — never transcribed tokens.
    // Transcription models (especially small.en) mis-spell proper nouns
    // (e.g. "Qwen" → "QUEN"). Sentence start/end times come from the
    // storyboard, not the transcript text.
    // Press S in preview to toggle subtitles on/off.
    var SUBTITLES_ON = false;
    document.getElementById('captions').classList.add('subs-off');
    document.addEventListener('keydown', function(e) {
      if (e.key === 's' || e.key === 'S') {
        SUBTITLES_ON = !SUBTITLES_ON;
        document.getElementById('captions').classList.toggle('subs-off', !SUBTITLES_ON);
      }
    });

    var SENTENCES = [
      // { s: startTime, e: endTime, t: "Original script sentence." },
      // ... one entry per sentence, times from storyboard ...
    ];

    var capEl = document.getElementById("cap-text");
    var capTl = gsap.timeline({ paused: true });
    SENTENCES.forEach(function(sen) {
      capTl.call(function(txt){ capEl.textContent = txt; }, [sen.t], sen.s);
      capTl.set(capEl, { opacity: 1 }, sen.s);
      capTl.set(capEl, { opacity: 0 }, sen.e - 0.05);
    });
    window.__timelines["captions-overlay"] = capTl;
  </script>
</div>
</body>
</html>
```

### 4e. Layout Rules

- **Every timed scene div MUST have `class="clip"`** — HyperFrames uses this to schedule visibility. Without it, all scenes are visible at once.
- Only two elements should have `data-composition-id`: the root composition div and the captions overlay div. Individual scene divs do NOT get `data-composition-id`.
- **Use literal font names** in CSS (e.g., `font-family: 'Outfit', sans-serif`) — do NOT use CSS variables for font-family in the root composition. The renderer cannot resolve `var(--f)` and will fall back to a system font.
- `scene-content`: `width:100%; height:100%; padding:120px 160px; display:flex; flex-direction:column; justify-content:center; gap:24px; box-sizing:border-box`
- Headlines: 80–120px, weight 700–900
- Body text: 42–56px, weight 400–500
- Decorative bg element in non-title scenes (ghost text, hairline, grain, radial glow). **Title cards use a clean pure-dark background — no radial glow circle.** — see house-style.md
- Enter with `gsap.from()`, NO exit tweens except on final scene
- **Use Unicode glyphs, never emoji, in all scene HTML.** Emoji render as full-colour cartoons in Chromium (which HyperFrames uses for frame capture) and look out of place in a dark cinematic style. Unicode glyphs can be coloured via CSS and scale cleanly at any size.
  - **Choose glyphs that fit the semantic context of the scene** — do not default to a fixed set. A scalability scene might use `∞`, a contrast scene `≠`, a highlight `✸`, a warning `☠`, a direction `→`.
  - **Stick to these Unicode ranges** — they render as reliable monochrome text glyphs in Chromium, not emoji:
    - Geometric Shapes `U+25A0–U+25FF`: `◆ ▲ ▶ ■ ◉ ◈ ◎ ▣ ⬡`
    - Dingbats `U+2700–U+27BF`: `✦ ✧ ✕ ✗ ✓ ✸ ✶`
    - Arrows `U+2190–U+21FF`: `→ ← ↑ ↓ ⇒ ⇄`
    - Math Operators `U+2200–U+22FF`: `∞ ∑ ≠ ≈ ∆`
    - Misc Symbols `U+2600–U+26FF` (selective, avoid known emoji triggers like ⚡ U+26A1): `☠ ★ ☆ ♦ ♠`
  - **Avoid emoji codepoints** (`U+1F300–U+1FFFF`) entirely — they always render as colored emoji in Chromium regardless of CSS.
- Transitions between scenes: use a CSS transition class from `references/transitions/catalog.md`

### 4f. Lint and Preview

```bash
npx hyperframes lint        # must pass — fix any errors before handing off
npx hyperframes preview     # open studio to verify
```

---

## Step 5: Output Summary

Tell the user:

```
✅ Script-to-Video complete

📝 Script:      <word count> words (~<duration>s)
🎙️  Audio:       C:\\openclaw_files\outputs\<slug>.wav
📄 Transcript:  C:\\openclaw_files\projects\<slug>\transcript.json
🎬 Project:     C:\\openclaw_files\projects\<slug>\
🗂️  Scenes:      <N> scenes — <list scene types used>
🔍 Preview:     npx hyperframes preview  (in project dir)
📦 Render:      npx hyperframes render --fps 60 --output <slug>.mp4
```

---

## Error Handling

| Problem | Action |
|---------|--------|
| OmiVoice server offline | Run your local OmiVoice start script, wait 10s, retry |
| `cp1252` encoding error in Python | Set `PYTHONIOENCODING=utf-8` before running |
| Transcript >20% music tokens | Retry with `--model medium.en` |
| HyperFrames lint errors | Fix reported issues before preview |
| `gradio_client` not installed | `pip install gradio_client` |
| Two consecutive scenes same type | Re-assign second to next-best type from catalog |
| `timed_element_missing_clip_class` lint error | Add `class="clip"` to every scene div — this is required for HyperFrames to show/hide scenes at the right times |
| All scenes visible at once | Same fix: `class="clip"` missing from scene divs |
| Font renders as system fallback | Use `font-family:'Outfit',sans-serif` literally — do NOT use a CSS variable like `var(--f)` for font-family |
| Overlapping clips lint error | Two consecutive scenes share the same end/start time. Trim the earlier scene duration by 0.04s |
| `gsap_animates_clip_element` lint error | You called `tl.set('#sN', { display:... })` on a `.clip` div. HyperFrames owns `display` on clip elements. Remove the `tl.set()` calls — `data-start`/`data-duration` controls clip scheduling |
| `pexels-hero` video background bleeds through all clips in render | Wrong pattern: `<video autoplay>` inside a `.clip` div with no `data-track-index`. Correct pattern: `<video data-track-index="0">` in a plain non-timed wrapper div, outside all `.clip` divs |
| Sparse keyframe warning during render (`max interval: Xs`) | Raw Pexels videos have infrequent keyframes. Detect source FPS first: `ffprobe -v quiet -select_streams v:0 -show_entries stream=avg_frame_rate -of csv=p=0 in.mp4` (round fraction to int → SRC_FPS), then re-encode: `ffmpeg -y -i in.mp4 -c:v libx264 -r SRC_FPS -g SRC_FPS -keyint_min SRC_FPS -movflags +faststart -an out.mp4` |
| Captions show garbled words (e.g. "QUEN" for "Qwen") | Never use transcribed token text for captions. Use the original script sentences in a `SENTENCES` array with storyboard start/end times |
| Chinese model names mis-transcribed | `small.en` splits "Qwen" into "QU"+"EN" tokens. Use `medium.en` for scripts containing Chinese product names, or always use original script for captions |
| Composition not full-screen in preview/render (black bars, content in corner) | Three fixes required: (1) `<meta name="viewport" content="width=1920, height=1080">` in `<head>`, (2) `html, body { width:1920px; height:1080px; overflow:hidden }` — fixed pixels not percentages, (3) `.clip { position:absolute; inset:0; }` — without this, clip divs have auto height and content renders in a fraction of the canvas |
| `presenter-aside` video wrapper bleeds through earlier title card / scenes | The plain wrapper div is always in the DOM and renders even before `data-start`. Fix: add `display:none` to the wrapper `<div id="v-wrap">` and add `tl.set('#v-wrap', { display:'block' }, VIDEO_START_TIME)` in the GSAP timeline |
| Captions visible even when `SUBTITLES_ON = false` | HyperFrames overrides `style="visibility:hidden"` on `data-composition-id` elements. Use a CSS class instead: add `.subs-off #cap-text { opacity:0 !important; }` and toggle `classList.toggle('subs-off', !SUBTITLES_ON)` on the captions div |
| Title card / first scene is completely blank (elements invisible) | Using `tl.from('#el', { opacity:0 })` on an element that already has `opacity:0` inline → GSAP animates 0→0, nothing appears. **Fix:** remove inline `opacity:0` from the element, OR switch to `tl.fromTo('#el', { opacity:0, y:60 }, { opacity:1, y:0, ... })` — always use `fromTo` when you need an explicit destination value |
| `media_missing_id` lint error on `<video>` | Every `<video>` with `data-start` must also have a unique `id` attribute so the renderer can discover it. Add `id="bg-video"` (or any unique id) |
| `media_missing_id` lint error on `<audio>` | Same rule applies to `<audio>` elements. Always add `id="main-audio"` to the audio element in the root composition template |
| Serif font visible in `quote-card` scene | The decorative `"` mark defaulted to `font-family:Georgia,serif` in older versions of this skill. This is inconsistent with the Shadow Cut theme. Use `font-family:'Outfit',sans-serif` for ALL elements including decorative characters |
| `hud-overlay` cards float on dark background instead of over a video | HUD overlay requires a background video. Add a `<div id="v-wrap" style="display:none">` outside all `.clip` divs containing the `<video id="bg-video" ...>`, then `tl.set('#v-wrap', { display:'block' }, HUD_START_TIME)` to reveal it. Add a semi-transparent dark overlay inside the wrapper (`rgba(8,8,16,0.55)`) so the HUD cards remain readable |
| **Talking-cut:** lip sync drifts progressively — worse toward the end | Root cause: audio extracted from VFR source while video was re-encoded to CFR. Fix: always use the **sync-safe two-step encode** — re-encode with audio first, extract WAV from that encode, strip audio for face-cam. All three share identical timing baseline. |
| **Talking-cut:** `overlapping_clips_same_track` lint error | `<audio data-main-audio>` has `data-track-index="0"` and the face-cam `<video>` is also on track 0. Fix: **remove `data-track-index` entirely from the `<audio>` element**. The `data-main-audio` attribute is sufficient; HyperFrames manages audio routing automatically. |
| **Talking-cut:** `data-duration` causes 2–4s over-run at end | Whisper's internal duration estimate is not accurate — it may report 202s for a 198.8s video. Always run `ffprobe -v quiet -show_entries format=duration -of csv=p=0 assets/source-kf.mp4` and use that exact value for all `data-duration` attributes. |
| **Talking-cut:** micro-flash of face-cam between two consecutive cutaway clips | Gap between clips was <3s (e.g. s3 ends at 27.9s, s4 starts at 28.0s → 0.1s flash). Enforce ≥3s minimum gap between consecutive cutaway clips. Move the later clip's `data-start` out and reduce its `data-duration` by the same amount to keep it in sync with the audio. |
| **Talking-cut:** render stalls at ~60-70% during frame capture | Likely memory pressure or browser worker crash on long compositions (>3 min). Try `--workers 4` to reduce parallel load: `npx hyperframes render --workers 4 --output output.mp4` |

---

## File Layout

```
C:\\openclaw_files\
  outputs\
    <slug>.wav               ← synthesized audio
  projects\
    <slug>\
      DESIGN.md              ← visual identity (write first!)
      storyboard.json        ← sentence → scene type map
      index.html             ← main composition
      transcript.json        ← word-level timestamps
      audio\
        <slug>.wav           ← copy of audio for HyperFrames
      assets\                ← talking-cut mode only
        source-kf-audio.mp4  ← Step 1: CFR re-encode WITH audio (keep for re-extraction)
        source-kf.mp4        ← Step 3: muted face-cam (stripped from source-kf-audio.mp4)
```

---

## Standalone Sample Boilerplate

> **Use this shell for EVERY standalone sample project** (chart demos, scene type showcases, catalog entries). Never start from scratch — copy this exactly, then fill in the scene content.
>
> The #1 cause of "content not full-screen" bugs is missing `.clip { position:absolute; inset:0; }`. It is included here. Do NOT remove or abbreviate the CSS block.

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=1920, height=1080">
  <script src="https://cdn.jsdelivr.net/npm/gsap@3.14.2/dist/gsap.min.js"></script>
  <style>
    /* ── Full-screen foundation — ALL THREE RULES are required ── */
    * { margin: 0; padding: 0; box-sizing: border-box; }
    html, body { width: 1920px; height: 1080px; overflow: hidden; background: #000; }
    [data-composition-id] { position: absolute; inset: 0; overflow: hidden; }
    /* ⚠️ NEVER OMIT: without this, clips have auto-height → content renders in a corner */
    .clip { position: absolute; inset: 0; }
    /* ── Scene shell ── */
    .scene-bg { position: absolute; inset: 0; background: #0a0a0a; }
    .scene-content {
      position: relative; width: 100%; height: 100%;
      padding: 80px 120px;
      display: flex; flex-direction: column; justify-content: center;
      gap: 24px; box-sizing: border-box;
    }
  </style>
</head>
<body>

<div data-composition-id="<slug>" data-start="0" data-duration="TOTAL_DURATION"
     data-width="1920" data-height="1080">

  <!-- Scene 1 -->
  <div id="s1" class="clip" data-start="0" data-duration="SCENE_DUR" data-track-index="1">
    <div class="scene-bg"></div>
    <div class="scene-content">
      <!-- scene HTML here -->
    </div>
  </div>

  <!-- Add more scenes as needed -->

  <script>
    window.__timelines = window.__timelines || {};
    var tl = gsap.timeline({ paused: true });

    // GSAP tweens here

    window.__timelines['<slug>'] = tl;
  </script>
</div>
</body>
</html>
```

**Checklist before rendering any standalone sample:**
- [ ] `<meta name="viewport" content="width=1920, height=1080">` present
- [ ] `html, body { width:1920px; height:1080px; overflow:hidden }` present (pixels, not %)
- [ ] `.clip { position:absolute; inset:0; }` present in `<style>` block
- [ ] Every scene div has `class="clip"`
- [ ] `data-composition-id` on root div matches `window.__timelines['<slug>']` key
- [ ] Run `npx hyperframes lint` before render (catches clip class errors)

---

## Appendix: Scene Type Catalog

All patterns use `#0a0a0a` bg / `#e2e2e2` text / `#c0392b` accent (Shadow Cut). Adjust to DESIGN.md palette.
Every scene inner HTML goes inside `<div class="scene-bg"></div><div class="scene-content">...</div>`.
All GSAP tweens go inside the main `tl` using the scene's `data-start` time as the base offset.

---

### Shared Atoms

Reusable elements from the user-stories canonical reference. Use these inside any scene type.

**`eyebrow`** — Small uppercase accent label above a title. Drops in before the headline.

```html
<div id="sN-eye" style="font-size:22px;font-weight:700;letter-spacing:6px;
  color:#c0392b;text-transform:uppercase;opacity:0;">USER STORIES 101</div>
```

```js
tl.fromTo('#sN-eye', { opacity:0, y:-20 }, { opacity:1, y:0, duration:0.3, ease:'power2.out' }, offset + 0.1);
```

**`rule` bar** — 200px accent-colored horizontal rule that scales in from the left. Underlines a title.

```html
<div id="sN-bar" style="width:200px;height:6px;background:#c0392b;
  transform-origin:left;transform:scaleX(0);margin:28px auto 0;"></div>
```

```js
tl.fromTo('#sN-bar', { scaleX:0 }, { scaleX:1, duration:0.6, ease:'power2.inOut', transformOrigin:'left' }, offset + 0.45);
```

**`sec-label`** — Same visual as eyebrow but used as a category header inside a scene (list-reveal, flow-steps). Slides down from above.

```html
<div id="sN-lbl" style="font-size:22px;font-weight:700;letter-spacing:6px;
  color:#c0392b;text-transform:uppercase;margin-bottom:36px;opacity:0;">THE GPS OF PRODUCT DEV</div>
```

```js
tl.fromTo('#sN-lbl', { opacity:0, y:-20 }, { opacity:1, y:0, duration:0.3 }, offset + 0.1);
```

> **Reference:** All three atoms appear in S1–S2 of `C:\\openclaw_files\projects\user-stories\index.html`.

---

### 1. `title-card`

Large centered title + subtitle. Used for opening and major transitions.

```html
<div class="scene-bg" style="background:#0a0a0a;"></div>
<div class="scene-content" style="align-items:center;text-align:center;">
  <h1 id="s1-title" class="hed" style="font-size:130px;">YOUR TOPIC</h1>
  <p  id="s1-sub"   class="sub">the sentence text here</p>
</div>
```

```js
// offset = scene startTime
tl.from("#s1-title", { y:60, opacity:0, duration:0.7, ease:"power3.out" }, offset + 0.2);
tl.from("#s1-sub",   { y:40, opacity:0, duration:0.5, ease:"power2.out" }, offset + 0.5);
```

---

### 2. `kinetic-text`

Definition / explanation sentence — each word flies in sequentially, full-bleed.

```html
<div class="scene-bg"></div>
<div class="scene-content" style="align-items:flex-start;">
  <div id="sN-kinetic" style="font-family:Outfit,sans-serif;font-weight:900;
    font-size:96px;color:#e2e2e2;line-height:1.1;max-width:1500px;">
    <!-- split sentence into spans: one per word -->
    <span class="kw">Each</span> <span class="kw">word</span> <span class="kw">flies</span> <span class="kw">in.</span>
  </div>
</div>
```

```js
// Generate spans dynamically if preferred:
// sentence.split(" ").forEach((w, i) => { el.innerHTML += `<span class="kw" id="kw-${sceneId}-${i}">${w} </span>`; });
tl.from("#sN-kinetic .kw", {
  y:80, opacity:0, duration:0.45, ease:"power3.out",
  stagger:{ each:0.07, from:"start" }
}, offset + 0.15);
```

---

### 2b. `kinetic-impact`

Three-tier text punch: muted setup line (small/faded) → large white emphasis word → XL accent slam word. Used when a sentence has a setup + payoff structure ("but when they get them right? → EVERYTHING → FLOWS."). Derived from the canonical user-stories project.

```html
<div class="scene-bg"></div>
<div class="scene-content" style="align-items:flex-start;justify-content:center;gap:24px;">
  <!-- Tier 1: muted setup — small, faded, sets context -->
  <div id="sN-ki1" style="font-family:Outfit,sans-serif;font-weight:400;
    font-size:40px;color:#8899aa;letter-spacing:0.02em;">but when they get them right?</div>
  <!-- Tier 2: large white punch word -->
  <div id="sN-ki2" style="font-family:Outfit,sans-serif;font-weight:900;
    font-size:120px;color:#e2e2e2;line-height:0.95;letter-spacing:-2px;">EVERYTHING</div>
  <!-- Tier 3: XL accent slam — biggest, colored -->
  <div id="sN-ki3" style="font-family:Outfit,sans-serif;font-weight:900;
    font-size:180px;color:#c0392b;line-height:0.9;letter-spacing:-4px;">FLOWS.</div>
</div>
```

```js
// Stagger ~0.7s between tiers — setup lands first, punch follows, slam closes
tl.from("#sN-ki1", { y:40, opacity:0, duration:0.5, ease:"power2.out" }, offset + 0.15);
tl.from("#sN-ki2", { y:80, opacity:0, duration:0.6, ease:"back.out(2)" }, offset + 0.85);
tl.from("#sN-ki3", { y:100, opacity:0, scale:0.85, duration:0.55, ease:"back.out(2.5)" }, offset + 1.55);
```

> **Reference:** Scene S4 and S10 of `C:\\openclaw_files\projects\user-stories\index.html`. CSS classes: `.kt-sm` (40px/400/muted), `.kt-lg` (120px/900/text), `.kt-xl` (180px/900/accent).

---

### 2c. `kinetic-slam`

Single massive word that punches in as a one-word scene — used as a beat between heavier scenes. Ultra-short duration (0.6–1.0s). No setup, no sub-text, just the word.

```html
<div class="scene-bg"></div>
<div class="scene-content" style="align-items:center;justify-content:center;">
  <div id="sN-slam" style="font-family:Outfit,sans-serif;font-weight:900;
    font-size:240px;color:#e2e2e2;letter-spacing:-6px;text-align:center;opacity:0;">
    FLIP IT.
  </div>
</div>
```

```js
// Ultra-fast entrance — back.out(3) gives maximum punch
tl.fromTo('#sN-slam', { opacity:0, scale:0.7 }, { opacity:1, scale:1, duration:0.2, ease:'back.out(3)' }, offset + 0.02);
```

> **When to use:** One-word pivots ("FLIP IT.", "WRONG.", "STOP."), dramatic pauses, punctuation beats between comparison and flow scenes. Duration should match the word's audio timing — typically 0.5–0.8s. Reference: Scene S10 of `C:\\openclaw_files\projects\user-stories\index.html`, CSS class `.kt-slam`.

---

### 3. `stat-reveal`

Big isolated number/stat with a label and fill bar beneath.

```html
<div class="scene-bg"></div>
<div id="sN-bar-bg" style="position:absolute;bottom:0;left:0;width:100%;height:6px;background:#1a1a1a;"></div>
<div id="sN-bar"    style="position:absolute;bottom:0;left:0;width:0%;height:6px;background:#c0392b;transform-origin:left;"></div>
<div class="scene-content" style="align-items:center;text-align:center;">
  <div id="sN-num"   class="hed" style="font-size:200px;color:#e2e2e2;font-variant-numeric:tabular-nums;">47</div>
  <div id="sN-label" class="sub" style="font-size:56px;letter-spacing:0.08em;">THINGS SHIPPED</div>
</div>
```

```js
tl.from("#sN-num",   { scale:0.6, opacity:0, duration:0.6, ease:"back.out(2)" }, offset + 0.15);
tl.from("#sN-label", { y:30, opacity:0, duration:0.4, ease:"power2.out" }, offset + 0.55);
tl.to("#sN-bar",     { width:"100%", duration:0.8, ease:"power2.out" }, offset + 0.4);
```

---

### 4. `counter-up`

Number counts up from 0 to target. Good for counts, years, totals.

```html
<div class="scene-bg"></div>
<div class="scene-content" style="align-items:center;text-align:center;">
  <div id="sN-counter" class="hed" style="font-size:180px;font-variant-numeric:tabular-nums;">0</div>
  <div id="sN-clabel"  class="sub">LINES OF CODE GENERATED</div>
</div>
```

```js
var counterObj = { val: 0 };
var TARGET = 1000000;
var sceneDur = endTime - startTime;
tl.to(counterObj, {
  val: TARGET, duration: sceneDur * 0.75, ease: "power1.inOut",
  onUpdate: function() {
    var el = document.getElementById("sN-counter");
    if (el) el.textContent = Math.round(counterObj.val).toLocaleString();
  }
}, offset + 0.2);
tl.from("#sN-clabel", { y:30, opacity:0, duration:0.4, ease:"power2.out" }, offset + 0.3);
```

> **Note:** `onUpdate` callbacks are allowed — they only update textContent, not GSAP visual props.

---

### 5. `bar-chart`

Horizontal CSS bars that grow from left. Max 5 bars. No axes, no gridlines, no legends.

```html
<div class="scene-bg"></div>
<div class="scene-content">
  <div id="sN-chart-title" class="sub" style="font-size:48px;margin-bottom:32px;">Top 5 Languages</div>
  <div style="display:flex;flex-direction:column;gap:28px;width:100%;">
    <!-- Repeat for each bar -->
    <div style="display:flex;align-items:center;gap:24px;">
      <span style="font-family:Outfit;font-weight:700;font-size:40px;color:#e2e2e2;width:220px;text-align:right;">Python</span>
      <div style="flex:1;background:#1a1a1a;border-radius:4px;height:44px;overflow:hidden;">
        <div id="sN-bar1" style="width:0%;height:100%;background:#c0392b;border-radius:4px;transform-origin:left;"></div>
      </div>
      <span id="sN-pct1" style="font-family:Outfit;font-weight:900;font-size:44px;color:#e2e2e2;opacity:0;">78%</span>
    </div>
    <!-- ... repeat for bars 2-5 ... -->
  </div>
</div>
```

```js
tl.from("#sN-chart-title", { x:-40, opacity:0, duration:0.5, ease:"power2.out" }, offset + 0.15);
[1,2,3,4,5].forEach(function(i) {
  var pct = [78, 65, 54, 48, 32][i-1];
  tl.to("#sN-bar"  + i, { width: pct+"%", duration:0.6, ease:"power2.out" }, offset + 0.3 + i*0.12);
  tl.to("#sN-pct"  + i, { opacity:1, duration:0.3, ease:"power1.out"      }, offset + 0.7 + i*0.12);
});
```

---

### 6. `list-reveal`

3–5 items stagger in one by one. Use for capability/feature lists.

```html
<div class="scene-bg"></div>
<div class="scene-content">
  <div id="sN-lhead" class="hed" style="font-size:80px;">What it can do</div>
  <ul style="list-style:none;padding:0;margin:0;display:flex;flex-direction:column;gap:20px;">
    <li id="sN-li1" style="font-family:Outfit;font-weight:500;font-size:56px;color:#e2e2e2;display:flex;align-items:center;gap:24px;">
      <span style="width:12px;height:12px;border-radius:50%;background:#c0392b;flex-shrink:0;"></span>Write code from natural language
    </li>
    <li id="sN-li2" style="font-family:Outfit;font-weight:500;font-size:56px;color:#e2e2e2;display:flex;align-items:center;gap:24px;">
      <span style="width:12px;height:12px;border-radius:50%;background:#c0392b;flex-shrink:0;"></span>Run tests and fix failures
    </li>
    <li id="sN-li3" style="font-family:Outfit;font-weight:500;font-size:56px;color:#e2e2e2;display:flex;align-items:center;gap:24px;">
      <span style="width:12px;height:12px;border-radius:50%;background:#c0392b;flex-shrink:0;"></span>Search and explore codebases
    </li>
  </ul>
</div>
```

```js
tl.from("#sN-lhead",         { x:-50, opacity:0, duration:0.6, ease:"expo.out" }, offset + 0.1);
tl.from("#sN-li1, #sN-li2, #sN-li3", {
  x:-60, opacity:0, duration:0.45, ease:"power2.out",
  stagger: { each:0.18, from:"start" }
}, offset + 0.4);
```

---

### 6b. `list-reveal-words` (compact word-stack variant)

No numbers, no bullets — just stacked giant words, each sliding in from the left. Use for 2–4 item lists where each item is a single phrase you want to hit like a drumbeat ("THE HUMAN / THE PAIN / THE DESIRE"). Faster timing, bigger type, no header needed.

```html
<div class="scene-bg"></div>
<div class="scene-content" style="align-items:flex-start;justify-content:center;">
  <div id="sN-wlbl" style="font-size:22px;font-weight:700;letter-spacing:6px;
    color:#c0392b;text-transform:uppercase;margin-bottom:24px;opacity:0;">START WITH</div>
  <div style="display:flex;flex-direction:column;gap:8px;">
    <div id="sN-w1" style="font-family:Outfit,sans-serif;font-weight:900;
      font-size:100px;color:#e2e2e2;opacity:0;">THE HUMAN</div>
    <div id="sN-w2" style="font-family:Outfit,sans-serif;font-weight:900;
      font-size:100px;color:#e2e2e2;opacity:0;">THE PAIN</div>
    <div id="sN-w3" style="font-family:Outfit,sans-serif;font-weight:900;
      font-size:100px;color:#e2e2e2;opacity:0;">THE DESIRE</div>
  </div>
</div>
```

```js
// Each word individually timed — NOT stagger — so they can sync to audio words
tl.fromTo('#sN-wlbl', { opacity:0 }, { opacity:1, duration:0.2 }, offset + 0.06);
tl.fromTo('#sN-w1', { opacity:0, x:-50 }, { opacity:1, x:0, duration:0.25, ease:'power2.out' }, offset + 0.26);
tl.fromTo('#sN-w2', { opacity:0, x:-50 }, { opacity:1, x:0, duration:0.25, ease:'power2.out' }, offset + 0.66);
tl.fromTo('#sN-w3', { opacity:0, x:-50 }, { opacity:1, x:0, duration:0.25, ease:'power2.out' }, offset + 1.09);
```

> **Reference:** Scene S11 of `C:\\openclaw_files\projects\user-stories\index.html`. CSS class `.lr-big` (100px/900/text). Each item timed individually to match audio word timestamps, not auto-staggered.

---

### 7. `flow-steps`

Numbered steps connected by an animated SVG arrow. Ideal for process sentences.

> **⚠️ CRITICAL — Arrow arrowhead bug:** The `<polygon>` inside the SVG arrowhead is always visible unless the parent `<svg>` starts with `opacity:0` in its `style` attribute. Do NOT rely on GSAP alone — the `tl.set(…{opacity:1}…)` is NOT enough if the SVG element doesn't start hidden. Every SVG arrow MUST have `style="flex-shrink:0;opacity:0;"` (note `opacity:0`). The GSAP `tl.set("#sN-arr1",{opacity:1},…)` then reveals it at the right moment. Without this, the arrowhead triangle appears immediately at scene start before the line draws in.

```html
<div class="scene-bg"></div>
<div class="scene-content" style="align-items:center;">
  <div id="sN-fhead" class="sub" style="margin-bottom:48px;">How it works</div>
  <div style="display:flex;align-items:center;gap:0;width:100%;">

    <!-- Step 1 -->
    <div id="sN-st1" style="flex:1;text-align:center;">
      <div style="width:100px;height:100px;border-radius:50%;border:4px solid #c0392b;
        display:flex;align-items:center;justify-content:center;margin:0 auto 20px;
        font-family:Outfit;font-weight:900;font-size:52px;color:#c0392b;">1</div>
      <div style="font-family:Outfit;font-weight:700;font-size:44px;color:#e2e2e2;">Write</div>
    </div>

    <!-- Arrow: opacity:0 on the SVG is REQUIRED — polygon is visible by default -->
    <svg id="sN-arr1" width="120" height="40" viewBox="0 0 120 40" style="flex-shrink:0;opacity:0;">
      <line x1="0" y1="20" x2="100" y2="20" stroke="#c0392b" stroke-width="3" stroke-dasharray="100" stroke-dashoffset="100"/>
      <polygon points="100,10 120,20 100,30" fill="#c0392b"/>
    </svg>

    <!-- Step 2 -->
    <div id="sN-st2" style="flex:1;text-align:center;">
      <div style="width:100px;height:100px;border-radius:50%;border:4px solid #c0392b;
        display:flex;align-items:center;justify-content:center;margin:0 auto 20px;
        font-family:Outfit;font-weight:900;font-size:52px;color:#c0392b;">2</div>
      <div style="font-family:Outfit;font-weight:700;font-size:44px;color:#e2e2e2;">Run</div>
    </div>

    <!-- Arrow: opacity:0 on the SVG is REQUIRED — polygon is visible by default -->
    <svg id="sN-arr2" width="120" height="40" viewBox="0 0 120 40" style="flex-shrink:0;opacity:0;">
      <line x1="0" y1="20" x2="100" y2="20" stroke="#c0392b" stroke-width="3" stroke-dasharray="100" stroke-dashoffset="100"/>
      <polygon points="100,10 120,20 100,30" fill="#c0392b"/>
    </svg>

    <!-- Step 3 -->
    <div id="sN-st3" style="flex:1;text-align:center;">
      <div style="width:100px;height:100px;border-radius:50%;border:4px solid #c0392b;
        display:flex;align-items:center;justify-content:center;margin:0 auto 20px;
        font-family:Outfit;font-weight:900;font-size:52px;color:#c0392b;">3</div>
      <div style="font-family:Outfit;font-weight:700;font-size:44px;color:#e2e2e2;">Ship</div>
    </div>
  </div>
</div>
```

```js
tl.from("#sN-fhead", { y:-30, opacity:0, duration:0.5, ease:"power2.out" }, offset + 0.1);
tl.from("#sN-st1",   { scale:0.7, opacity:0, duration:0.5, ease:"back.out(2)" }, offset + 0.3);
// tl.set reveals SVG (which starts opacity:0), then the line draws in
tl.set("#sN-arr1", { opacity:1 }, offset + 0.65);
tl.to("#sN-arr1 line", { strokeDashoffset:0, duration:0.35, ease:"power2.out" }, offset + 0.65);
tl.from("#sN-st2",   { scale:0.7, opacity:0, duration:0.5, ease:"back.out(2)" }, offset + 0.85);
tl.set("#sN-arr2", { opacity:1 }, offset + 1.2);
tl.to("#sN-arr2 line", { strokeDashoffset:0, duration:0.35, ease:"power2.out" }, offset + 1.2);
tl.from("#sN-st3",   { scale:0.7, opacity:0, duration:0.5, ease:"back.out(2)" }, offset + 1.4);
```

---

### 7b. `flow-steps-text` (text-arrow variant)

Simpler flow-steps using plain text `→` characters as arrows instead of SVG. No arrowhead bug possible. Use this when steps are card-based (flat background), step timing is tight, or you prefer cleaner markup. Derived from the canonical user-stories project.

```html
<div class="scene-bg"></div>
<div class="scene-content" style="align-items:center;flex-direction:column;">
  <!-- sec-label drops in first -->
  <div id="sN-fttl" style="font-size:24px;font-weight:700;letter-spacing:6px;
    color:#c0392b;text-transform:uppercase;margin-bottom:56px;opacity:0;">THE CLARITY FORMULA</div>

  <div style="display:flex;align-items:center;justify-content:center;width:100%;">
    <!-- Step 1 -->
    <div id="sN-fs1" style="background:#111111;padding:48px 52px;flex:1;text-align:center;opacity:0;">
      <div style="font-size:52px;font-weight:900;color:#c0392b;margin-bottom:12px;">1</div>
      <div style="font-size:34px;font-weight:700;color:#e2e2e2;line-height:1.3;">Know Your<br>Audience</div>
    </div>

    <!-- Text arrow — no SVG, no polygon bug -->
    <div id="sN-fa1" style="font-size:60px;color:#6b7280;padding:0 16px;opacity:0;">→</div>

    <!-- Step 2 -->
    <div id="sN-fs2" style="background:#111111;padding:48px 52px;flex:1;text-align:center;opacity:0;">
      <div style="font-size:52px;font-weight:900;color:#c0392b;margin-bottom:12px;">2</div>
      <div style="font-size:34px;font-weight:700;color:#e2e2e2;line-height:1.3;">Speak Their<br>Language</div>
    </div>

    <!-- Text arrow -->
    <div id="sN-fa2" style="font-size:60px;color:#6b7280;padding:0 16px;opacity:0;">→</div>

    <!-- Step 3 -->
    <div id="sN-fs3" style="background:#111111;padding:48px 52px;flex:1;text-align:center;opacity:0;">
      <div style="font-size:52px;font-weight:900;color:#c0392b;margin-bottom:12px;">3</div>
      <div style="font-size:34px;font-weight:700;color:#e2e2e2;line-height:1.3;">Stop Over-<br>complicating</div>
    </div>
  </div>

  <!-- Tagline fades in after all steps -->
  <div id="sN-fsub" style="font-size:28px;font-weight:400;color:#6b7280;margin-top:36px;opacity:0;">
    The kind of clarity that works.
  </div>
</div>
```

```js
// Title drops, steps reveal, text arrows slide in between each step
tl.fromTo('#sN-fttl', { opacity:0, y:-20 }, { opacity:1, y:0, duration:0.4 }, offset + 0.1);
tl.fromTo('#sN-fs1',  { opacity:0, y:30  }, { opacity:1, y:0, duration:0.4, ease:'power2.out' }, offset + 0.56);
tl.fromTo('#sN-fa1',  { opacity:0, x:-20 }, { opacity:1, x:0, duration:0.2 }, offset + 1.11);
tl.fromTo('#sN-fs2',  { opacity:0, y:30  }, { opacity:1, y:0, duration:0.4, ease:'power2.out' }, offset + 2.16);
tl.fromTo('#sN-fa2',  { opacity:0, x:-20 }, { opacity:1, x:0, duration:0.2 }, offset + 2.66);
tl.fromTo('#sN-fs3',  { opacity:0, y:30  }, { opacity:1, y:0, duration:0.4, ease:'power2.out' }, offset + 4.16);
tl.fromTo('#sN-fsub', { opacity:0 },         { opacity:1, duration:0.4 }, offset + 4.96);
```

> **Reference:** Scene S7 of `C:\\openclaw_files\projects\user-stories\index.html`. CSS classes: `.fs-title`, `.fs-row`, `.fs-step`, `.fs-num`, `.fs-txt`, `.fs-arr` (the text arrow), `.fs-sub`. Each step/arrow is individually timed to audio, not auto-staggered.

---

### 8. `quote-card`

Pull quote — dark card with bold left border, 180px accent quote mark, and optional inline word highlights. For key phrases worth reading slowly.

**Structure:** muted uppercase intro label above → dark card with left red border → red `"` inside the card → quote text (use `<em>` for accent word highlights, no extra CSS needed) → optional attribution line.

```html
<div class="scene-bg"></div>
<div class="scene-content" style="align-items:flex-start;justify-content:center;">
  <div style="width:1400px;">

    <!-- Intro label — muted uppercase above the card -->
    <div id="sN-intro" style="font-family:'Outfit',sans-serif;font-size:26px;font-weight:500;
      color:#8899aa;letter-spacing:3px;text-transform:uppercase;margin-bottom:28px;opacity:0;">
      A good user story sounds like this:
    </div>

    <!-- Card: dark bg + thick left border -->
    <div id="sN-card" style="background:#111111;padding:60px 80px;
      border-left:8px solid #c0392b;position:relative;opacity:0;">

      <!-- 180px red quote mark — positioned inside card, hangs slightly above content -->
      <div id="sN-mark" style="position:absolute;top:-30px;left:56px;
        font-family:'Outfit',sans-serif;font-size:180px;color:#c0392b;line-height:1;opacity:0;">"</div>

      <!-- Quote text — use <em> for accent word highlights (no extra style needed) -->
      <blockquote id="sN-qt" style="font-family:'Outfit',sans-serif;font-weight:400;font-size:50px;
        color:#e2e2e2;line-height:1.5;padding-top:36px;margin:0;opacity:0;">
        As a <em style="color:#c0392b;font-style:normal;font-weight:700;">user</em>,
        I want to save my progress so I don't lose my work.
      </blockquote>

      <!-- Attribution — optional, omit if not needed -->
      <div id="sN-attr" style="font-family:'Outfit',sans-serif;font-size:24px;font-weight:500;
        color:#8899aa;margin-top:20px;letter-spacing:2px;opacity:0;">— Clean. Human. Clear.</div>

    </div>
  </div>
</div>
```

```js
// offset = scene startTime
tl.fromTo('#sN-intro', { opacity:0, y:-20 }, { opacity:1, y:0, duration:0.4 },                  offset + 0.12);
tl.fromTo('#sN-card',  { opacity:0, y:30  }, { opacity:1, y:0, duration:0.5, ease:'power2.out' }, offset + 0.5);
tl.fromTo('#sN-mark',  { opacity:0, scale:2 }, { opacity:1, scale:1, duration:0.35 },             offset + 0.5);
tl.fromTo('#sN-qt',    { opacity:0 },           { opacity:1, duration:0.5 },                      offset + 1.3);
tl.fromTo('#sN-attr',  { opacity:0 },           { opacity:1, duration:0.3 },                      offset + 3.5);
```

> **Reference:** Scene S5 of `C:\openclaw_files\projects\user-stories\index.html`. CSS classes: `.qc-wrap` (width:1400px), `.qc-intro` (26px/500/muted/uppercase), `.qc-card` (background:#111111, border-left:8px), `.qc-mark` (180px/accent, inside card at top:-30px), `.qc-text` (50px/400, `em` for accent words), `.qc-attr` (24px/muted).

---

### 9. `split-layout`

Left column: sentence text. Right column: supporting visual (icon, shape, number).

```html
<div class="scene-bg"></div>
<div class="scene-content" style="flex-direction:row;align-items:center;gap:80px;">
  <!-- Left: text -->
  <div id="sN-left" style="flex:1;display:flex;flex-direction:column;gap:20px;">
    <div class="hed" style="font-size:80px;">Key Point</div>
    <div class="sub">The sentence text goes here, describing the concept clearly.</div>
  </div>
  <!-- Divider -->
  <div id="sN-div" style="width:4px;height:400px;background:#c0392b;border-radius:2px;flex-shrink:0;opacity:0;"></div>
  <!-- Right: visual -->
  <div id="sN-right" style="flex:1;display:flex;align-items:center;justify-content:center;">
    <!-- e.g. a large icon, number, shape, or decorative element -->
    <div style="font-size:200px;line-height:1;color:#c0392b;">◆</div>
  </div>
</div>
```

```js
tl.from("#sN-left",  { x:-60, opacity:0, duration:0.65, ease:"expo.out" }, offset + 0.2);
tl.to("#sN-div",     { opacity:1, scaleY:0, transformOrigin:"top", duration:0 }, offset + 0.5);
tl.from("#sN-div",   { scaleY:0, transformOrigin:"top", duration:0.5, ease:"power2.out" }, offset + 0.5);
tl.set("#sN-div",    { opacity:1 }, offset + 0.5);
tl.from("#sN-right", { x:60, opacity:0, duration:0.65, ease:"expo.out" }, offset + 0.6);
```

---

### 9b. `split-screen`

Full-bleed 50/50 vertical split — each half fills its side of the canvas, text centered vertically within it. Left half uses the card background; right half uses the primary dark background. A large `→` floats absolutely at the center. No borders or card boxes — clean negative space carries the weight.

**When to use:** Sentence that maps two paired concepts cleanly — need → answer, problem → solution, input → output. Ideal for short punchy contrasts where both sides need equal visual weight. NOT the same as `split-layout` (which is text + visual in a horizontal content row).

**Type signal:** Sentence pairs two short concepts with a clear mapping ("Then build the feature as the answer", "Human pain → Build the feature").

```html
<div class="scene-bg"></div>
<!-- Full-bleed flex row — both halves stretch to full height -->
<div style="position:absolute;inset:0;display:flex;flex-direction:row;align-items:stretch;">

  <!-- Left half: slightly darker card bg, thin right border -->
  <div id="sN-sl" style="width:50%;height:100%;display:flex;flex-direction:column;
    align-items:center;justify-content:center;
    background:#111111;border-right:2px solid #222;opacity:0;">
    <div style="font-family:'Outfit',sans-serif;font-size:26px;font-weight:700;
      letter-spacing:5px;color:#8899aa;text-transform:uppercase;margin-bottom:16px;">THE NEED</div>
    <div style="font-family:'Outfit',sans-serif;font-size:64px;font-weight:900;
      color:#e2e2e2;text-align:center;line-height:1.2;">Human pain<br>+ desire</div>
  </div>

  <!-- Right half: primary dark bg -->
  <div id="sN-sr" style="width:50%;height:100%;display:flex;flex-direction:column;
    align-items:center;justify-content:center;
    background:#0a0a0a;opacity:0;">
    <div style="font-family:'Outfit',sans-serif;font-size:26px;font-weight:700;
      letter-spacing:5px;color:#c0392b;text-transform:uppercase;margin-bottom:16px;">THE ANSWER</div>
    <div style="font-family:'Outfit',sans-serif;font-size:64px;font-weight:900;
      color:#e2e2e2;text-align:center;line-height:1.2;">Build the<br>feature</div>
  </div>

</div>

<!-- Arrow floats absolutely at the vertical center seam -->
<div id="sN-sarr" style="position:absolute;top:50%;left:50%;transform:translate(-50%,-50%);
  font-family:'Outfit',sans-serif;font-size:120px;color:#c0392b;line-height:1;opacity:0;">→</div>
```

```js
// offset = scene startTime
// Left flies in from left, arrow pops, right flies in from right
tl.fromTo('#sN-sl',   { opacity:0, x:-40 }, { opacity:1, x:0, duration:0.3, ease:'power2.out' }, offset + 0.07);
tl.fromTo('#sN-sarr', { opacity:0, scale:0 }, { opacity:1, scale:1, duration:0.2 },               offset + 0.5);
tl.fromTo('#sN-sr',   { opacity:0, x:40  }, { opacity:1, x:0, duration:0.3, ease:'power2.out' }, offset + 0.67);
```

> **Reference:** Scene S12 of `C:\openclaw_files\projects\user-stories\index.html`. CSS classes: `.sl` (flex-direction:row on scene-content), `.sl-half` (50% width, full height, centered flex), `.sl-left` (card bg + right border), `.sl-right` (#0a0a0a), `.sl-lbl` (eyebrow, muted), `.sl-lbl.acc` (eyebrow, accent), `.sl-body` (64px/900), `.sl-arrow` (absolute center, 120px accent). Note: this scene can be very short (1–2s) since both sides land fast.

---

### 9b. `doodle-split`

Split layout — text on the left, a hand-drawn doodle illustration on the right. Uses the local doodle library at `E:\Doodle_Library`. Great for sentences with a concrete noun, action, emotion, or visual metaphor that can be illustrated.

**Doodle selection (do this before writing HTML):**

1. Identify the key concept in the sentence (e.g., "collaboration", "rocket", "thinking").
2. Search `<path-to-doodle-library>\_ALL\` for a filename that matches:
   ```powershell
  Get-ChildItem "<path-to-doodle-library>\_ALL" -File | Where-Object { $_.Name -match "keyword" }
   ```
3. Copy the best match into the project's `assets/` folder:
   ```powershell
  Copy-Item "<path-to-doodle-library>\_ALL\rocket.png" "C:\openclaw_files\projects\<slug>\assets\doodle-sN.png"
   ```
4. Use `assets/doodle-sN.png` as the `src` in the HTML below.

**Format preference:** SVG > PNG (SVGs scale perfectly at any size). Both work as `<img>` tags. PNG is fine if no SVG match exists.

**Categories available for keyword ideas:** Active People, animals, Animated, casual, emotions, faces, Food and Drink, Mythical Beings, Objects, places, plants, Science and Technology, Shapes and Arrows, Special Characters, Transportation.

**⚠️ Ink colour + theme rule:** Doodle Library assets are **black ink on transparent background**. Black ink is invisible on dark themes (`#0a0a0a`). Apply `filter: invert(1)` to the `<img>` when using a dark theme. On light themes (`#ffffff`, `open-page`, `broadsheet`) use no filter — black ink shows naturally.

| Theme | `<img>` style |
|---|---|
| Dark (Shadow Cut, Neon Tokyo, Terminal Green, Blueprint, Velvet Standard, Dusk Gradient) | `filter:invert(1)` |
| Light (Open Page, Broadsheet, Frost, Brutalist) | no filter |

```html
<div class="scene-bg"></div>
<div class="scene-content" style="flex-direction:row;align-items:center;gap:80px;padding:80px 120px;">

  <!-- Left: text block -->
  <div id="sN-dtxt" style="flex:1;display:flex;flex-direction:column;gap:24px;">
    <div id="sN-deye" style="font-size:22px;font-weight:700;letter-spacing:6px;
      color:#c0392b;text-transform:uppercase;opacity:0;">CONCEPT</div>
    <div id="sN-dhed" style="font-family:Outfit,sans-serif;font-weight:900;
      font-size:96px;color:#e2e2e2;line-height:1.05;opacity:0;">
      The headline.
    </div>
    <div id="sN-dsub" style="font-family:Outfit,sans-serif;font-weight:400;
      font-size:44px;color:#8899aa;line-height:1.4;opacity:0;">
      The sentence text goes here.
    </div>
  </div>

  <!-- Vertical divider -->
  <div id="sN-ddiv" style="width:3px;height:0px;background:#c0392b;border-radius:2px;flex-shrink:0;align-self:center;"></div>

  <!-- Right: doodle — filter:invert(1) for dark themes (black ink → white); remove for light themes -->
  <div id="sN-dimg" style="flex-shrink:0;width:480px;display:flex;align-items:center;justify-content:center;opacity:0;">
    <img src="assets/doodle-sN.png" alt="doodle"
         style="width:480px;height:480px;object-fit:contain;display:block;filter:invert(1);">
  </div>
</div>
```

```js
// offset = scene startTime
tl.fromTo('#sN-deye',  { opacity:0, y:-20 }, { opacity:1, y:0, duration:0.3, ease:'power2.out' },     offset + 0.15);
tl.fromTo('#sN-dhed',  { opacity:0, x:-50 }, { opacity:1, x:0, duration:0.6, ease:'expo.out' },       offset + 0.3);
tl.fromTo('#sN-dsub',  { opacity:0, x:-40 }, { opacity:1, x:0, duration:0.5, ease:'power2.out' },     offset + 0.65);
tl.to('#sN-ddiv', { height:'320px', duration:0.5, ease:'power2.inOut' },                              offset + 0.5);
tl.fromTo('#sN-dimg',  { opacity:0, scale:0.75, x:40 }, { opacity:1, scale:1, x:0, duration:0.65, ease:'back.out(1.8)' }, offset + 0.6);
```

> **No-match fallback:** If no doodle closely matches the sentence, skip this type and use `split-layout` or `kinetic-text` instead. Don't force a poor match.

**Reference projects (rendered, lint-clean, 12s, 60fps):**

| Theme | Project path | Notes |
|---|---|---|
| Dark (Shadow Cut) | `C:\openclaw_files\projects\doodle-split-dark\` | `filter:invert(1)` on images. laptop + brain doodles. Accent `#c0392b`. |
| Light (Open Page) | `C:\openclaw_files\projects\doodle-split-light\` | No filter on images. coffee + book doodles. Accent `#e63946`, bg `#ffffff`, text `#1a1a1a`. |

Both outputs are in the project root as `doodle-split-dark.mp4` and `doodle-split-light.mp4`. Use these as copy-paste starting points for new `doodle-split` scenes.

---

### 10. `progress-ring`

SVG circle that fills to a percentage. For proportions and completion metrics.

```html
<div class="scene-bg"></div>
<div class="scene-content" style="align-items:center;text-align:center;">
  <div id="sN-prhead" class="sub" style="font-size:52px;margin-bottom:40px;">Developer Adoption</div>
  <div style="position:relative;width:400px;height:400px;margin:0 auto;">
    <!-- Track circle -->
    <svg width="400" height="400" style="position:absolute;top:0;left:0;" viewBox="0 0 400 400">
      <circle cx="200" cy="200" r="160" fill="none" stroke="#1a1a1a" stroke-width="20"/>
      <!-- Animated fill — circumference = 2π×160 ≈ 1005 -->
      <circle id="sN-ring" cx="200" cy="200" r="160" fill="none" stroke="#c0392b" stroke-width="20"
        stroke-dasharray="1005" stroke-dashoffset="1005" stroke-linecap="round"
        transform="rotate(-90 200 200)"/>
    </svg>
    <!-- Center number -->
    <div id="sN-rpct" style="position:absolute;top:50%;left:50%;transform:translate(-50%,-50%);
      font-family:Outfit;font-weight:900;font-size:100px;color:#e2e2e2;">0%</div>
  </div>
  <div id="sN-rsub" class="sub" style="margin-top:32px;">of teams using AI tools daily</div>
</div>
```

```js
var TARGET_PCT = 72; // e.g. 72%
var circumference = 1005;
var targetOffset = circumference * (1 - TARGET_PCT / 100);
var ringObj = { pct: 0 };
tl.from("#sN-prhead", { y:-30, opacity:0, duration:0.5, ease:"power2.out" }, offset + 0.15);
tl.to(ringObj, {
  pct: TARGET_PCT, duration:1.0, ease:"power2.inOut",
  onUpdate: function() {
    var el = document.getElementById("sN-rpct");
    var ring = document.getElementById("sN-ring");
    if (el)   el.textContent = Math.round(ringObj.pct) + "%";
    if (ring) ring.setAttribute("stroke-dashoffset", String(circumference * (1 - ringObj.pct/100)));
  }
}, offset + 0.3);
tl.from("#sN-rsub", { y:20, opacity:0, duration:0.4, ease:"power1.out" }, offset + 0.5);
```

---

### 11. `comparison`

Two columns side by side — contrast two things. Highlight the "winner" with accent color.

```html
<div class="scene-bg"></div>
<div class="scene-content" style="flex-direction:row;gap:60px;align-items:stretch;">
  <!-- Left (loser) — centered content, muted glyph -->
  <div id="sN-cola" style="flex:1;background:#111;border-radius:16px;padding:60px 48px;
    display:flex;flex-direction:column;gap:20px;border:2px solid #222;
    align-items:center;justify-content:center;text-align:center;">
    <div style="font-family:Outfit,sans-serif;font-size:88px;color:#444;line-height:1;">✗</div>
    <div style="font-family:Outfit;font-weight:900;font-size:56px;color:#8899aa;">BEFORE</div>
    <div style="font-family:Outfit;font-weight:400;font-size:40px;color:#8899aa;line-height:1.4;">Manual, slow, error-prone</div>
  </div>
  <!-- VS badge -->
  <div id="sN-vs" style="flex-shrink:0;align-self:center;font-family:Outfit;font-weight:900;
    font-size:52px;color:#c0392b;">VS</div>
  <!-- Right (winner) — centered content, accent glyph -->
  <div id="sN-colb" style="flex:1;background:#150909;border-radius:16px;padding:60px 48px;
    display:flex;flex-direction:column;gap:20px;border:2px solid #c0392b;
    align-items:center;justify-content:center;text-align:center;">
    <div style="font-family:Outfit,sans-serif;font-size:88px;color:#c0392b;line-height:1;">◆</div>
    <div style="font-family:Outfit;font-weight:900;font-size:56px;color:#c0392b;">AFTER</div>
    <div style="font-family:Outfit;font-weight:400;font-size:40px;color:#e2e2e2;line-height:1.4;">Automated, instant, reliable</div>
  </div>
</div>
```

```js
tl.from("#sN-cola", { x:-80, opacity:0, duration:0.6, ease:"expo.out" }, offset + 0.2);
tl.from("#sN-vs",   { scale:0.5, opacity:0, duration:0.4, ease:"back.out(3)" }, offset + 0.6);
tl.from("#sN-colb", { x:80, opacity:0, duration:0.6, ease:"expo.out" }, offset + 0.8);
```

---

### 11b. `comparison-verdict`

Variant of comparison with a "SUDDENLY." (or similar impact word) header above the two cards. Cards use colored top borders (not full borders), an icon, label, descriptor, and an arrow connector (→) instead of VS. Ideal when the sentence has a reveal structure ("And SUDDENLY, bad vs good becomes obvious."). Derived from the canonical user-stories project.

```html
<div class="scene-bg"></div>
<div class="scene-content" style="align-items:center;gap:32px;">

  <!-- Impact header — reveals before cards -->
  <div id="sN-suddenly" style="font-family:Outfit,sans-serif;font-weight:900;
    font-size:96px;color:#e2e2e2;letter-spacing:-2px;align-self:flex-start;">SUDDENLY.</div>

  <!-- Card row -->
  <div style="display:flex;align-items:stretch;gap:0;width:100%;">

    <!-- Bad card: red top border -->
    <div id="sN-cbad" style="flex:1;background:#111;border-radius:16px;padding:48px 40px;
      border-top:6px solid #c0392b;display:flex;flex-direction:column;align-items:center;gap:16px;text-align:center;">
      <div style="font-size:72px;">☠</div>
      <div style="font-family:Outfit;font-weight:900;font-size:44px;color:#e2e2e2;">RANSOM NOTE</div>
      <div style="font-family:Outfit;font-weight:400;font-size:28px;color:#8899aa;line-height:1.4;">
        Vague, uncheckable, nobody owns it
      </div>
    </div>

    <!-- Arrow connector -->
    <div id="sN-cvs" style="flex-shrink:0;align-self:center;padding:0 32px;
      font-family:Outfit;font-weight:900;font-size:52px;color:#8899aa;">→</div>

    <!-- Good card: green top border -->
    <div id="sN-cgood" style="flex:1;background:#0a160a;border-radius:16px;padding:48px 40px;
      border-top:6px solid #27ae60;display:flex;flex-direction:column;align-items:center;gap:16px;text-align:center;">
      <div style="font-size:72px;">◆</div>
      <div style="font-family:Outfit;font-weight:900;font-size:44px;color:#27ae60;">STRATEGY</div>
      <div style="font-family:Outfit;font-weight:400;font-size:28px;color:#8899aa;line-height:1.4;">
        Clear outcome, testable, owned
      </div>
    </div>

  </div>
</div>
```

```js
// Header drops first, then bad card slides left, connector pops, good card slides right
tl.from("#sN-suddenly", { y:-50, opacity:0, duration:0.55, ease:"expo.out" }, offset + 0.1);
tl.from("#sN-cbad",   { x:-80, opacity:0, duration:0.55, ease:"expo.out" }, offset + 0.55);
tl.from("#sN-cvs",    { scale:0.4, opacity:0, duration:0.4, ease:"back.out(3)" }, offset + 0.9);
tl.from("#sN-cgood",  { x:80, opacity:0, duration:0.55, ease:"expo.out" }, offset + 1.05);
```

> **Reference:** Scene S13 of `C:\\openclaw_files\projects\user-stories\index.html` (lines 357–374 HTML, 544–550 GSAP). CSS classes: `.cmp-bad` (red top border), `.cmp-good` (green top border), `.cmp-ico` (72px emoji), `.cmp-lbl` (44px/900), `.cmp-desc` (28px/muted), `.cmp-vs` (arrow connector), `.suddenly` (impact header above row).

---

### 11c. `product-comparison`

Two product cards side-by-side separated by a VS divider. Left card: standard/challenger (muted, dark border). Right card: winner/recommended (accent border, floating badge). Feature rows stagger in row-by-row across both cards simultaneously, then the winner badge pops last.

**Type signal:** sentence compares two named products, plans, tools, or pricing tiers with multiple attributes (e.g., "Pro has everything Starter does, plus unlimited seats and 100GB storage at a lower price").

**Feature icon conventions:**

| Indicator | HTML entity | Color |
|---|---|---|
| Not supported | `&#10007;` (✗) | `#444` (muted) |
| Supported | `&#10003;` (✓) | `#27ae60` (green) |
| Unlimited | `&#8734;` (∞) | `#c0392b` (accent, winner card) |
| Numeric value | text `<span>` | `#8899aa` dim (loser) / `#c0392b` accent (winner) |

```html
<div class="scene-bg"></div>
<!-- Accent glow behind winner card -->
<div style="position:absolute;top:50%;right:200px;width:600px;height:600px;
  border-radius:50%;background:radial-gradient(circle,rgba(192,57,43,0.12) 0%,transparent 70%);
  transform:translateY(-50%);pointer-events:none;"></div>

<div class="scene-content" style="align-items:center;gap:24px;padding:60px 120px;">

  <!-- Eyebrow -->
  <div id="sN-pclbl" style="font-family:Outfit,sans-serif;font-size:20px;font-weight:700;
    letter-spacing:8px;color:#c0392b;text-transform:uppercase;opacity:0;">HEAD TO HEAD</div>

  <!-- Card row -->
  <div style="display:flex;align-items:stretch;gap:0;width:100%;flex:1;min-height:0;">

    <!-- Card A: standard/loser — muted colors -->
    <div id="sN-pca" style="flex:1;background:#111111;border-radius:20px 0 0 20px;
      border:2px solid #222;border-right:none;padding:44px 48px;
      display:flex;flex-direction:column;opacity:0;">

      <div style="margin-bottom:28px;padding-bottom:24px;border-bottom:1px solid #1e1e1e;">
        <div style="font-family:Outfit,sans-serif;font-size:50px;font-weight:900;color:#8899aa;">Starter</div>
        <div style="font-family:Outfit,sans-serif;font-size:22px;color:#444;margin:6px 0 14px;">For solo developers</div>
        <div style="font-family:Outfit,sans-serif;font-size:60px;font-weight:900;color:#8899aa;line-height:1;">
          $29<span style="font-size:24px;font-weight:400;color:#555;">/mo</span>
        </div>
      </div>

      <div style="display:flex;flex-direction:column;gap:18px;">
        <!-- Feature row pattern: icon + label. Repeat for each feature. -->
        <div id="sN-pca-f1" style="display:flex;align-items:center;gap:16px;opacity:0;">
          <span style="font-size:28px;width:36px;text-align:center;color:#444;">&#10007;</span>
          <span style="font-family:Outfit,sans-serif;font-size:30px;font-weight:600;color:#555;">API Access</span>
        </div>
        <!-- sN-pca-f2, f3, f4 follow same pattern -->
      </div>
    </div>

    <!-- VS divider -->
    <div style="width:72px;background:#161616;display:flex;align-items:center;justify-content:center;flex-shrink:0;">
      <div id="sN-pcvs" style="font-family:Outfit,sans-serif;font-size:22px;font-weight:900;
        color:#c0392b;letter-spacing:6px;opacity:0;">VS</div>
    </div>

    <!-- Card B: winner/recommended — accent border, red tint bg -->
    <div id="sN-pcb" style="flex:1;background:#150909;border-radius:0 20px 20px 0;
      border:2px solid #c0392b;border-left:none;padding:44px 48px;
      display:flex;flex-direction:column;position:relative;opacity:0;">

      <!-- Winner badge — floats above card top edge; starts hidden, pops in last -->
      <div id="sN-badge" style="position:absolute;top:-18px;left:50%;transform:translateX(-50%);
        background:#c0392b;color:#fff;font-family:Outfit,sans-serif;font-size:16px;
        font-weight:900;letter-spacing:5px;padding:6px 22px;border-radius:20px;
        white-space:nowrap;opacity:0;">&#9733; RECOMMENDED</div>

      <div style="margin-bottom:28px;padding-bottom:24px;border-bottom:1px solid #2a1010;">
        <div style="font-family:Outfit,sans-serif;font-size:50px;font-weight:900;color:#c0392b;">Pro</div>
        <div style="font-family:Outfit,sans-serif;font-size:22px;color:#555;margin:6px 0 14px;">For growing teams</div>
        <div style="font-family:Outfit,sans-serif;font-size:60px;font-weight:900;color:#e2e2e2;line-height:1;">
          $19<span style="font-size:24px;font-weight:400;color:#666;">/mo</span>
        </div>
      </div>

      <div style="display:flex;flex-direction:column;gap:18px;">
        <!-- Feature row pattern: icon + label. Repeat for each feature. -->
        <div id="sN-pcb-f1" style="display:flex;align-items:center;gap:16px;opacity:0;">
          <span style="font-size:28px;width:36px;text-align:center;color:#27ae60;">&#10003;</span>
          <span style="font-family:Outfit,sans-serif;font-size:30px;font-weight:600;color:#e2e2e2;">API Access</span>
        </div>
        <!-- sN-pcb-f2, f3, f4 follow same pattern -->
      </div>
    </div>

  </div>
</div>
```

```js
var offset = sceneStartTime;

tl.fromTo('#sN-pclbl', { opacity:0, y:-20 }, { opacity:1, y:0, duration:0.4, ease:'power2.out' }, offset + 0.1);

// Both cards fly in simultaneously from opposite sides
tl.fromTo('#sN-pca', { opacity:0, x:-60 }, { opacity:1, x:0, duration:0.55, ease:'expo.out' }, offset + 0.3);
tl.fromTo('#sN-pcb', { opacity:0, x:60  }, { opacity:1, x:0, duration:0.55, ease:'expo.out' }, offset + 0.3);

// VS badge scales in between the two cards
tl.fromTo('#sN-pcvs', { opacity:0, scale:0.5 }, { opacity:1, scale:1, duration:0.4, ease:'back.out(2)' }, offset + 0.65);

// Feature rows stagger — both cards reveal row-by-row in sync (mirrored entrance)
[1, 2, 3, 4].forEach(function(i) {
  var t = offset + 0.9 + (i - 1) * 0.22;
  tl.fromTo('#sN-pca-f' + i, { opacity:0, x:-20 }, { opacity:1, x:0, duration:0.3, ease:'power2.out' }, t);
  tl.fromTo('#sN-pcb-f' + i, { opacity:0, x:20  }, { opacity:1, x:0, duration:0.3, ease:'power2.out' }, t);
});

// Winner badge pops last — final emphasis on the recommended choice
tl.fromTo('#sN-badge', { opacity:0, scale:0.5, y:-8 }, { opacity:1, scale:1, y:0, duration:0.45, ease:'back.out(3)' }, offset + 1.85);
```

> **Reference:** `C:\\openclaw_files\projects\product-comparison-sample\index.html`. Use 3–5 feature rows. Always keep feature row `opacity:0` as the default inline style — card entrance and row reveals are separate tweens. Scale down to 3 rows for short scenes (≤5s), use 5 rows for longer scenes (≥8s).

---

Full-frame dark background + single bold punchline. Maximum impact. No radial glow — typography carries the weight.

```html
<div class="scene-bg"></div>
<div class="scene-content" style="align-items:center;text-align:center;">
  <div id="sN-ctext" class="hed" style="font-size:110px;color:#e2e2e2;max-width:1400px;line-height:1.1;">
    This is the one thing that matters.
  </div>
</div>
```

```js
tl.fromTo("#sN-ctext", { scale:0.85, opacity:0 }, { scale:1, opacity:1, duration:0.65, ease:"back.out(1.5)" }, offset + 0.1);
```

**`callout` with `rotationZ` entrance (kinetic variant):** When the main word should feel like it slams into place — use `scale:1.3, rotationZ:-2` on the entrance. Most effective for single-word callouts like "BACKWARDS." or "WRONG." Derived from S8 of user-stories.

```html
<!-- Same HTML as above, just change the GSAP entrance: -->
```

```js
tl.from("#sN-pre",  { opacity:0, y:20 }, { duration:0.4 }, offset + 0.26);
tl.fromTo("#sN-main", { opacity:0, scale:1.3, rotationZ:-2 },
  { opacity:1, scale:1, rotationZ:0, duration:0.5, ease:'power3.out' }, offset + 1.16);
tl.from("#sN-sub",  { opacity:0 }, { duration:0.3 }, offset + 2.56);
```

> **Reference:** Scene S8 of `C:\\openclaw_files\projects\user-stories\index.html` (line 513 GSAP). Use for single-word reveal sentences where the word itself carries maximum weight.

---

### 12b. `cta-callout` (CTA box with bounce arrow)

Callout variant with a framed CTA box and a bouncing directional arrow. Use for "click the link" or action-driving sentences at the end of a video. Pre-text + main sentence + box with arrow + hint text.

```html
<div class="scene-bg"></div>
<div class="scene-content" style="align-items:center;text-align:center;">
  <div id="sN-ctapre" style="font-size:28px;font-weight:700;letter-spacing:4px;
    color:#6b7280;text-transform:uppercase;margin-bottom:16px;opacity:0;">WANT THE QUICK HACK?</div>
  <div id="sN-ctamain" style="font-family:Outfit,sans-serif;font-weight:900;font-size:68px;
    color:#e2e2e2;max-width:1300px;line-height:1.2;opacity:0;">
    Makes user stories practically write themselves.
  </div>
  <!-- Bordered CTA box with bouncing arrow -->
  <div id="sN-ctabox" style="display:inline-flex;align-items:center;gap:24px;
    margin-top:40px;border:3px solid #c0392b;padding:20px 48px;opacity:0;">
    <span id="sN-ctaarr" style="font-size:60px;color:#c0392b;display:inline-block;">↓</span>
    <div>
      <div style="font-family:Outfit;font-weight:900;font-size:42px;color:#e2e2e2;letter-spacing:3px;">CLICK THE LINK</div>
      <div style="font-size:22px;color:#6b7280;">lower left</div>
    </div>
  </div>
</div>
```

```js
tl.fromTo('#sN-ctapre',  { opacity:0, y:-20 }, { opacity:1, y:0, duration:0.4 }, offset + 0.23);
tl.fromTo('#sN-ctamain', { opacity:0, y:30  }, { opacity:1, y:0, duration:0.5, ease:'power2.out' }, offset + 0.68);
tl.fromTo('#sN-ctabox',  { opacity:0, scale:0.85 }, { opacity:1, scale:1, duration:0.4, ease:'back.out(1.5)' }, offset + 3.18);
// Bouncing arrow — yoyo, finite repeats (calculate from remaining scene duration)
tl.to('#sN-ctaarr', { y:12, duration:0.35, ease:'sine.inOut', yoyo:true, repeat:5 }, offset + 3.68);
```

> **Reference:** Scene S14 of `C:\\openclaw_files\projects\user-stories\index.html` (lines 379–393 HTML, 552–558 GSAP). CSS classes: `.cta-box`, `.cta-arrow`, `.cta-label`, `.cta-hint`. `repeat` value must be finite — calculate from remaining scene time.

---

### 13. `icon-grid`

3–6 icons/emoji with labels, staggered reveal. For multi-concept sentences.

```html
<div class="scene-bg"></div>
<div class="scene-content">
  <div id="sN-ighead" class="hed" style="font-size:80px;margin-bottom:48px;">All in one place</div>
  <div style="display:grid;grid-template-columns:repeat(3,1fr);gap:40px;">
    <!-- Repeat for each icon -->
    <div id="sN-ic1" style="display:flex;flex-direction:column;align-items:center;gap:16px;
      background:#111;border-radius:16px;padding:40px 24px;">
      <div style="font-size:80px;line-height:1;color:#c0392b;">◈</div>
      <div style="font-family:Outfit;font-weight:700;font-size:40px;color:#e2e2e2;text-align:center;">Fix</div>
    </div>
    <div id="sN-ic2" style="display:flex;flex-direction:column;align-items:center;gap:16px;
      background:#111;border-radius:16px;padding:40px 24px;">
      <div style="font-size:80px;line-height:1;color:#c0392b;">✦</div>
      <div style="font-family:Outfit;font-weight:700;font-size:40px;color:#e2e2e2;text-align:center;">Write</div>
    </div>
    <div id="sN-ic3" style="display:flex;flex-direction:column;align-items:center;gap:16px;
      background:#111;border-radius:16px;padding:40px 24px;">
      <div style="font-size:80px;line-height:1;color:#c0392b;">▲</div>
      <div style="font-family:Outfit;font-weight:700;font-size:40px;color:#e2e2e2;text-align:center;">Ship</div>
    </div>
  </div>
</div>
```

```js
tl.from("#sN-ighead",         { y:-40, opacity:0, duration:0.55, ease:"power3.out" }, offset + 0.1);
tl.from("#sN-ic1, #sN-ic2, #sN-ic3", {
  y:60, opacity:0, scale:0.9, duration:0.5, ease:"back.out(1.8)",
  stagger: { each:0.15, from:"start" }
}, offset + 0.4);
```

---

### 14. `outro-card`

Closing scene — eyebrow label drops first, big title slams up, sub line fades, accent rule bar grows, then everything stagger-fades out.

**Three-beat structure:** eyebrow → title + sub → rule bar → hold → fade out.

```html
<div class="scene-bg"></div>
<!-- Optional: theme decoratives here (grid, corner brackets, etc.) -->
<div class="scene-content" style="align-items:center;text-align:center;gap:28px;">
  <!-- 1. Eyebrow label — drops in before the headline -->
  <div id="sN-olbl" style="font-size:22px;font-weight:700;letter-spacing:8px;
    color:#c0392b;text-transform:uppercase;opacity:0;">YOUR CLOSING LABEL</div>

  <!-- 2. Main title — the action/takeaway (big, high contrast) -->
  <div id="sN-otitle" class="hed" style="font-size:130px;opacity:0;">CLOSING WORDS.</div>

  <!-- 3. Sub line — supporting sentence -->
  <div id="sN-osub" class="sub" style="font-size:38px;opacity:0;">
    One short motivating or contextual line.
  </div>

  <!-- 4. Rule bar — grows from left, accent colored -->
  <div id="sN-orule" style="width:0px;height:3px;background:#c0392b;"></div>
</div>
```

```js
var sceneDur = endTime - startTime;
tl.fromTo('#sN-olbl',   { opacity:0, y:-20 }, { opacity:1, y:0, duration:0.4, ease:'power2.out' }, offset + 0.25);
tl.fromTo('#sN-otitle', { opacity:0, y:60  }, { opacity:1, y:0, duration:0.7, ease:'power3.out' }, offset + 0.55);
tl.fromTo('#sN-osub',   { opacity:0, y:30  }, { opacity:1, y:0, duration:0.5, ease:'power2.out' }, offset + 0.9);
tl.to('#sN-orule', { width:'280px', duration:0.8, ease:'power2.inOut' }, offset + 1.2);
// Final fade-out — only outro is allowed exit tweens
tl.to('#sN-otitle, #sN-olbl, #sN-osub, #sN-orule', {
  opacity:0, y:-30, duration:0.6, ease:'power2.in', stagger:0.08
}, offset + sceneDur - 1.4);
```

> **Reference:** Scene S14 of `C:\openclaw_files\projects\agile-vs-waterfall\index.html`. The eyebrow and rule bar are Shared Atoms (see above). Adapt colors to theme accent. Rule bar width (280px) is intentionally narrow — it's a punctuation mark, not a full-width divider.

---

### 15. `threejs-object`

A full 3D scene rendered with Three.js inside a HyperFrames clip. The Three.js render loop runs via `requestAnimationFrame` — GSAP tweens a plain `state` object, and rAF reads that state to update the mesh and call `renderer.render()` each frame.

**Why rAF, not `onUpdate`:** HyperFrames seeks the GSAP timeline with `suppressEvents=true` during frame capture, so `onUpdate` callbacks never fire during rendering. Use a `requestAnimationFrame` loop instead — HyperFrames captures the frame after rAF fires, so Three.js always has a fresh render ready.

**Key rules:**
- `<canvas>` fills the clip (`position:absolute;inset:0;display:block`)
- `renderer.setSize(1920, 1080)` + `renderer.setPixelRatio(1)` — fixed size, no auto-resize
- ALL 3D state driven via a plain `state` object that GSAP tweens
- Start an `(function animate(){ requestAnimationFrame(animate); ... renderer.render(); })()` loop — no `onUpdate` on `tl`

```html
<!-- Canvas fills the clip as the scene background -->
<canvas id="sN-canvas" style="position:absolute;inset:0;display:block;"></canvas>

<!-- Optional text overlay on the left, with gradient fade -->
<div style="position:absolute;inset:0;pointer-events:none;
  background:linear-gradient(90deg,rgba(10,22,40,0.92) 40%,transparent 72%);">
  <div style="display:flex;flex-direction:column;justify-content:center;height:100%;padding:120px 160px;max-width:900px;">
    <div id="sN-label" style="font-size:22px;font-family:'IBM Plex Mono',monospace;color:#4d6a99;letter-spacing:5px;opacity:0;">// 3D SCENE</div>
    <div id="sN-box" style="border:2px dashed rgba(77,106,153,0.4);background:#0d1f3c;padding:40px 48px;margin-top:20px;opacity:0;">
      <h1 style="font-family:'IBM Plex Mono',monospace;font-weight:700;font-size:88px;color:#e8f0ff;line-height:1.05;">Your<br><span style="color:#4da6ff;">headline.</span></h1>
      <p style="font-family:'IBM Plex Mono',monospace;font-size:38px;color:#4d6a99;margin-top:20px;">Supporting sentence here.</p>
    </div>
    <div id="sN-rule" style="width:0px;height:3px;background:#4da6ff;margin-top:32px;"></div>
  </div>
</div>
```

```js
// ── Three.js setup (runs once at load) ──────────────────────────────────
var canvas = document.getElementById('sN-canvas');
var renderer = new THREE.WebGLRenderer({ canvas: canvas, antialias: true });
renderer.setSize(1920, 1080);
renderer.setPixelRatio(1);
renderer.setClearColor(0x0a1628, 1);  // match theme BG

var scene = new THREE.Scene();
var camera = new THREE.PerspectiveCamera(55, 1920/1080, 0.1, 1000);
camera.position.set(4, 0, 7);
camera.lookAt(4, 0, 0);  // object offset right, camera looks at it

// Geometry — torus knot reads as complex/technical, great for Blueprint
var geo = new THREE.TorusKnotGeometry(1.5, 0.42, 160, 32);

var solidMesh = new THREE.Mesh(geo, new THREE.MeshStandardMaterial({
  color: 0x0d2a4a, emissive: 0x1a3d6b, roughness: 0.25, metalness: 0.85
}));
solidMesh.position.set(4, 0, 0);
scene.add(solidMesh);

var wireMesh = new THREE.Mesh(geo, new THREE.MeshBasicMaterial({
  color: 0x4da6ff, wireframe: true, opacity: 0.55, transparent: true
}));
wireMesh.position.set(4, 0, 0);
scene.add(wireMesh);

scene.add(new THREE.AmbientLight(0x4da6ff, 0.3));
var key = new THREE.PointLight(0x4da6ff, 4, 30);
key.position.set(6, 4, 6);
scene.add(key);

// State object — GSAP tweens these values, rAF reads them
var state = { rotX: 0, rotY: 0, scale: 0.01 };

// ── requestAnimationFrame render loop ────────────────────────────────────
// HyperFrames captures frames after rAF fires — Three.js must render here.
// GSAP sets state values when it seeks the timeline; rAF picks them up.
(function animate() {
  requestAnimationFrame(animate);
  solidMesh.rotation.x = wireMesh.rotation.x = state.rotX;
  solidMesh.rotation.y = wireMesh.rotation.y = state.rotY;
  solidMesh.scale.setScalar(state.scale);
  wireMesh.scale.setScalar(state.scale);
  renderer.render(scene, camera);
})();

// ── GSAP Timeline — no onUpdate needed ───────────────────────────────────
var tl = gsap.timeline({ paused: true });

tl.to(state, { scale: 1, duration: 1.2, ease: 'back.out(1.5)' }, offset + 0.1);
tl.to(state, { rotY: Math.PI * 2, rotX: Math.PI * 0.6, duration: sceneDur, ease: 'none' }, offset);

// Text entrances
tl.fromTo('#sN-label', { opacity:0, y:-20 }, { opacity:1, y:0, duration:0.4, ease:'power2.out' }, offset + 0.9);
tl.fromTo('#sN-box',   { opacity:0, y:40  }, { opacity:1, y:0, duration:0.7, ease:'power3.out' }, offset + 1.2);
tl.to('#sN-rule', { width:'240px', duration:0.8, ease:'power2.inOut' }, offset + 1.8);
```

**Geometry options** (swap `TorusKnotGeometry` for any of these):

| Geometry | Constructor | Character |
|---|---|---|
| Torus knot | `TorusKnotGeometry(1.5, 0.42, 160, 32)` | Complex, technical, infinite |
| Icosahedron | `IcosahedronGeometry(2, 1)` | Geometric, clean, crystalline |
| Sphere | `SphereGeometry(2, 64, 64)` | Smooth, planetary, AI |
| Box | `BoxGeometry(2.5, 2.5, 2.5)` | Corporate, SaaS, structured |
| Torus | `TorusGeometry(1.8, 0.5, 32, 100)` | Loop, cycle, infinity |
| Octahedron | `OctahedronGeometry(2, 0)` | Sharp, crystalline, abstract |

**More Three.js effects** — all work with the same rAF + state pattern:

| Effect | How | Use when |
|---|---|---|
| **Particle system** | `THREE.Points` with `BufferGeometry` + `PointsMaterial`; tween `state.spread` → update `positions` array in rAF | Galaxy, code rain, data scatter |
| **Custom shader** | `ShaderMaterial` with GLSL `vertexShader`/`fragmentShader`; tween a `uniforms.uTime` value → glowing plasma, noise waves | Futuristic BG, neon glow |
| **Multi-object scene** | Add multiple meshes; tween separate `state.rot1`, `state.rot2` etc. | DNA helix, solar system, logo orbit |
| **Line/wireframe only** | `EdgesGeometry` + `LineSegments` (no solid mesh) | Minimal/editorial, Blueprint spec diagrams |
| **Post-processing** | Not recommended — requires `EffectComposer` which needs ES module imports; stick to material tricks instead | — |
| **Explosion / morph** | Tween `geometry.attributes.position` per-vertex in rAF | Data transform, reveal animations |

**Spec-box sizing rule for `threejs-object`:** The text overlay column is `max-width:860px` with `padding:80px 120px`. Headline font size should be `≤68px` to fit inside the dashed spec-box without overflow. If the headline is short (1–2 words), 80px is safe.

> **Reference:** `C:\openclaw_files\projects\threejs-test\index.html`. 
> 
> **Three.js must be bundled locally — do NOT use CDN.** HyperFrames tries to inline CDN scripts at compile time, and Three.js CDN may return 404 or be unavailable offline. Download once:
> ```bash
> Invoke-WebRequest -Uri "https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.min.js" -OutFile "three.min.js"
> ```
> Then reference it as `<script src="three.min.js"></script>` (relative path in project root).
>
> **Confirmed working version:** `three@0.160.0`. Note: HyperFrames detects `requestAnimationFrame` and automatically switches to screenshot-capture mode with virtual time — this is the correct behavior and enables proper frame capture of Three.js scenes.

---

### 15b. `canvas2d-scene` (Canvas 2D / p5.js / PixiJS)

The same rAF pattern works for **any canvas-based JS library**. The pattern is always:
1. Get a `<canvas>` element
2. Start a `requestAnimationFrame` loop
3. Let GSAP tween a `state` object
4. In rAF, read `state` and draw to canvas

**Supported libraries (all confirmed compatible with HyperFrames rAF mode):**

| Library | Script | Best for |
|---|---|---|
| Canvas 2D API | Built-in (`canvas.getContext('2d')`) | Custom shapes, gradients, text effects |
| p5.js | `https://cdn.jsdelivr.net/npm/p5@1.9.0/lib/p5.min.js` | Generative art, flow fields, noise |
| PixiJS | `https://cdn.jsdelivr.net/npm/pixi.js@8/dist/pixi.min.js` | 2D sprites, particle showers, filters |

**Canvas 2D example (no external lib needed):**

```html
<canvas id="sN-c2d" style="position:absolute;inset:0;display:block;"></canvas>
```

```js
var cv = document.getElementById('sN-c2d');
cv.width = 1920; cv.height = 1080;
var ctx = cv.getContext('2d');
var state = { progress: 0, glow: 0 };

(function loop() {
  requestAnimationFrame(loop);
  ctx.clearRect(0, 0, 1920, 1080);
  // Draw using state.progress (0→1) driven by GSAP
  var r = state.progress * 960;
  ctx.beginPath();
  ctx.arc(960, 540, r, 0, Math.PI * 2);
  ctx.strokeStyle = 'rgba(77,166,255,' + state.glow + ')';
  ctx.lineWidth = 4;
  ctx.stroke();
})();

var tl = gsap.timeline({ paused: true });
tl.to(state, { progress: 1, glow: 1, duration: 2, ease: 'power2.out' }, offset);
```

> **Bundle p5.js locally** (same reason as Three.js — CDN may 404 at render time):
> ```bash
> Invoke-WebRequest -Uri "https://cdn.jsdelivr.net/npm/p5@1.9.0/lib/p5.min.js" -OutFile "p5.min.js"
> ```

---

### 16. `donut-chart`

Multi-segment SVG donut chart. Each slice draws on sequentially. Best for part-of-whole data: market share, budget splits, survey results. Max 5 segments.

**How it works:** Each segment is an SVG `<circle>` with `stroke-dasharray="0 1005"` (1005 ≈ circumference of r=160). GSAP animates `stroke-dasharray` to `"segment_px rest_px"`. Segments are offset using `stroke-dashoffset` to position them around the ring. Rotate the whole SVG −90° so segment 1 starts at 12 o'clock.

```html
<div class="scene-bg"></div>
<div class="scene-content" style="flex-direction:row;align-items:center;gap:80px;">

  <!-- Donut SVG (right side) -->
  <div style="position:relative;flex-shrink:0;width:480px;height:480px;">
    <svg width="480" height="480" viewBox="0 0 400 400"
         style="transform:rotate(-90deg);display:block;">
      <!-- Track -->
      <circle cx="200" cy="200" r="160" fill="none" stroke="#1a1a1a" stroke-width="60"/>
      <!-- Segment 1: e.g. 40% → 402px of 1005 circumference -->
      <circle id="sN-seg1" cx="200" cy="200" r="160" fill="none"
              stroke="#c0392b" stroke-width="60"
              stroke-dasharray="0 1005" stroke-dashoffset="0"/>
      <!-- Segment 2: 30% → 302px; offset = -(segment1_px) = -402 -->
      <circle id="sN-seg2" cx="200" cy="200" r="160" fill="none"
              stroke="#4da6ff" stroke-width="60"
              stroke-dasharray="0 1005" stroke-dashoffset="-402"/>
      <!-- Segment 3: 30% → 301px; offset = -(402+302) = -704 -->
      <circle id="sN-seg3" cx="200" cy="200" r="160" fill="none"
              stroke="#27ae60" stroke-width="60"
              stroke-dasharray="0 1005" stroke-dashoffset="-704"/>
    </svg>
    <!-- Centre label -->
    <div id="sN-clbl" style="position:absolute;top:50%;left:50%;transform:translate(-50%,-50%);
      font-family:Outfit,sans-serif;font-weight:900;font-size:72px;color:#e2e2e2;text-align:center;opacity:0;">
      100%
    </div>
  </div>

  <!-- Legend (left side) -->
  <div style="flex:1;display:flex;flex-direction:column;gap:28px;">
    <div id="sN-dhead" class="sub" style="font-size:52px;margin-bottom:8px;">Where it goes</div>
    <div id="sN-ll1" style="display:flex;align-items:center;gap:20px;opacity:0;">
      <div style="width:20px;height:20px;border-radius:4px;background:#c0392b;flex-shrink:0;"></div>
      <span style="font-family:Outfit;font-weight:600;font-size:44px;color:#e2e2e2;">Category A</span>
      <span style="font-family:Outfit;font-weight:900;font-size:44px;color:#c0392b;margin-left:auto;">40%</span>
    </div>
    <div id="sN-ll2" style="display:flex;align-items:center;gap:20px;opacity:0;">
      <div style="width:20px;height:20px;border-radius:4px;background:#4da6ff;flex-shrink:0;"></div>
      <span style="font-family:Outfit;font-weight:600;font-size:44px;color:#e2e2e2;">Category B</span>
      <span style="font-family:Outfit;font-weight:900;font-size:44px;color:#4da6ff;margin-left:auto;">30%</span>
    </div>
    <div id="sN-ll3" style="display:flex;align-items:center;gap:20px;opacity:0;">
      <div style="width:20px;height:20px;border-radius:4px;background:#27ae60;flex-shrink:0;"></div>
      <span style="font-family:Outfit;font-weight:600;font-size:44px;color:#e2e2e2;">Category C</span>
      <span style="font-family:Outfit;font-weight:900;font-size:44px;color:#27ae60;margin-left:auto;">30%</span>
    </div>
  </div>
</div>
```

```js
// Circumference = 2π × 160 ≈ 1005
// Segment px = percentage * 1005 / 100
// Segment dashoffset = -(sum of all previous segment px values)
var C = 1005;
var segs = [
  { id: '#sN-seg1', pct: 40 },  // 402px, offset 0
  { id: '#sN-seg2', pct: 30 },  // 302px, offset -402
  { id: '#sN-seg3', pct: 30 },  // 301px, offset -704
];

tl.from('#sN-dhead', { x:-40, opacity:0, duration:0.5, ease:'power2.out' }, offset + 0.1);

segs.forEach(function(seg, i) {
  var px = seg.pct * C / 100;
  tl.to(seg.id, {
    strokeDasharray: px + ' ' + (C - px),
    duration: 0.7, ease: 'power2.out'
  }, offset + 0.4 + i * 0.25);
  tl.fromTo('#sN-ll' + (i+1), { opacity:0, x:30 }, { opacity:1, x:0, duration:0.4 }, offset + 0.9 + i * 0.25);
});

tl.fromTo('#sN-clbl', { opacity:0, scale:0.6 }, { opacity:1, scale:1, duration:0.5, ease:'back.out(2)' }, offset + 1.4);
```

**Segment colours per theme:**

| Theme | Seg 1 | Seg 2 | Seg 3 |
|---|---|---|---|
| shadow-cut | `#c0392b` | `#e67e22` | `#8899aa` |
| blueprint | `#4da6ff` | `#27ae60` | `#c0392b` |
| neon-tokyo | `#00f5ff` | `#ff2d78` | `#a855f7` |
| velvet-standard | `#c9a84c` | `#8899aa` | `#555` |

---

### 17. `line-chart`

Animated SVG line chart — the polyline draws on from left to right. Best for trends over time, growth curves, before/after timelines. X-axis labels and data point dots pop in after the line draws.

```html
<div class="scene-bg"></div>
<div class="scene-content">
  <div id="sN-lchead" class="sub" style="font-size:52px;margin-bottom:40px;">Growth over time</div>

  <div style="position:relative;width:100%;height:520px;">
    <svg id="sN-svg" width="100%" height="520" viewBox="0 0 1580 520" preserveAspectRatio="none"
         style="position:absolute;top:0;left:0;overflow:visible;">

      <!-- Grid lines (horizontal, faint) -->
      <line x1="0"    y1="130" x2="1580" y2="130" stroke="#1a1a1a" stroke-width="1"/>
      <line x1="0"    y1="260" x2="1580" y2="260" stroke="#1a1a1a" stroke-width="1"/>
      <line x1="0"    y1="390" x2="1580" y2="390" stroke="#1a1a1a" stroke-width="1"/>
      <line x1="0"    y1="520" x2="1580" y2="520" stroke="#333"    stroke-width="2"/>

      <!-- Area fill under line (optional gradient) -->
      <defs>
        <linearGradient id="sN-grad" x1="0" y1="0" x2="0" y2="1">
          <stop offset="0%"   stop-color="#c0392b" stop-opacity="0.18"/>
          <stop offset="100%" stop-color="#c0392b" stop-opacity="0"/>
        </linearGradient>
      </defs>
      <!-- Fill polygon — same x/y as polyline but closes at bottom -->
      <polygon id="sN-fill"
        points="0,520 0,440 263,380 527,280 790,200 1053,140 1317,100 1580,60 1580,520"
        fill="url(#sN-grad)" opacity="0"/>

      <!-- The data line — stroke-dasharray trick for draw-on -->
      <!-- Measure total length: for 7 points across 1580px, ~2100 is a safe overestimate -->
      <polyline id="sN-line"
        points="0,440 263,380 527,280 790,200 1053,140 1317,100 1580,60"
        fill="none" stroke="#c0392b" stroke-width="4"
        stroke-dasharray="2100" stroke-dashoffset="2100"/>

      <!-- Data point dots — start hidden -->
      <circle id="sN-p1" cx="0"    cy="440" r="8" fill="#c0392b" opacity="0"/>
      <circle id="sN-p2" cx="263"  cy="380" r="8" fill="#c0392b" opacity="0"/>
      <circle id="sN-p3" cx="527"  cy="280" r="8" fill="#c0392b" opacity="0"/>
      <circle id="sN-p4" cx="790"  cy="200" r="8" fill="#c0392b" opacity="0"/>
      <circle id="sN-p5" cx="1053" cy="140" r="8" fill="#c0392b" opacity="0"/>
      <circle id="sN-p6" cx="1317" cy="100" r="8" fill="#c0392b" opacity="0"/>
      <circle id="sN-p7" cx="1580" cy="60"  r="8" fill="#c0392b" opacity="0"/>
    </svg>

    <!-- X-axis labels (positioned under each data point) -->
    <div style="position:absolute;bottom:-48px;left:0;right:0;display:flex;justify-content:space-between;">
      <span id="sN-xl1" style="font-family:Outfit;font-size:32px;color:#8899aa;opacity:0;">Q1</span>
      <span id="sN-xl2" style="font-family:Outfit;font-size:32px;color:#8899aa;opacity:0;">Q2</span>
      <span id="sN-xl3" style="font-family:Outfit;font-size:32px;color:#8899aa;opacity:0;">Q3</span>
      <span id="sN-xl4" style="font-family:Outfit;font-size:32px;color:#8899aa;opacity:0;">Q4</span>
      <span id="sN-xl5" style="font-family:Outfit;font-size:32px;color:#8899aa;opacity:0;">Q5</span>
      <span id="sN-xl6" style="font-family:Outfit;font-size:32px;color:#8899aa;opacity:0;">Q6</span>
      <span id="sN-xl7" style="font-family:Outfit;font-size:32px;color:#8899aa;opacity:0;">Q7</span>
    </div>
  </div>
</div>
```

```js
var LINE_DUR = 1.2; // time for line to fully draw

tl.from('#sN-lchead', { x:-40, opacity:0, duration:0.5, ease:'power2.out' }, offset + 0.1);

// Line draws on left→right
tl.to('#sN-line', { strokeDashoffset: 0, duration: LINE_DUR, ease: 'power2.inOut' }, offset + 0.4);

// Area fill fades in as line draws
tl.to('#sN-fill', { opacity: 1, duration: LINE_DUR * 0.8, ease: 'power1.out' }, offset + 0.6);

// Dots pop in staggered along the line draw timeline
var dotInterval = LINE_DUR / 7;
[1,2,3,4,5,6,7].forEach(function(i) {
  var t = offset + 0.4 + dotInterval * (i - 1);
  tl.fromTo('#sN-p' + i, { opacity:0, scale:0, transformOrigin: 'center' },
    { opacity:1, scale:1, duration:0.2, ease:'back.out(3)' }, t);
  tl.fromTo('#sN-xl' + i, { opacity:0 }, { opacity:1, duration:0.2 }, t + 0.05);
});
```

**To adapt to your data:**
1. Calculate each point's `cy` value: `cy = 520 - (value / maxValue) * 520` (SVG origin is top-left)
2. Space `cx` values evenly: `cx = (index / (numPoints-1)) * 1580`
3. Update `<polyline points="...">` and `<polygon points="0,520 [all cx/cy] [last_cx],520">` to match
4. Set `stroke-dasharray` to a safe overestimate of the polyline length (use 1.2× the width as a rough estimate)

---

### 18. `pexels-hero`

Full-bleed Pexels image or looping video as the scene background, with a dark gradient scrim and text overlay. Cinematic and documentary in feel.

**When to use:** Atmospheric, contextual, or emotional sentences that benefit from real-world imagery — product environments, people working, cityscapes, abstract tech visuals. Ideal for opening hooks, emotional peaks, or any sentence that would feel flat with a plain dark background.

**Pre-requisite:** Run Step 3.6 (Pexels asset fetch) before writing index.html. Files land in `assets/pexels-sN.jpg` or `assets/pexels-sN.mp4`.

#### Photo variant

```html
<div class="scene-bg"></div>

<!-- Full-bleed photo -->
<img id="sN-img" src="assets/pexels-s3.jpg"
     style="position:absolute;inset:0;width:100%;height:100%;object-fit:cover;opacity:0;">

<!-- Dark gradient scrim — left side stays readable, right side bleeds to photo -->
<div style="position:absolute;inset:0;
  background:linear-gradient(90deg,rgba(10,10,10,0.88) 38%,rgba(10,10,10,0.30) 100%);
  pointer-events:none;"></div>

<div class="scene-content" style="align-items:flex-start;max-width:900px;">
  <div id="sN-eye" style="font-size:20px;font-weight:700;letter-spacing:6px;
    color:#c0392b;text-transform:uppercase;opacity:0;">THE REALITY</div>
  <h2 id="sN-hed" style="font-family:Outfit,sans-serif;font-weight:900;
    font-size:96px;color:#e2e2e2;line-height:1.05;margin-top:16px;opacity:0;">
    The sentence text goes here.
  </h2>
</div>
```

```js
// Photo fades in first, text follows
tl.to('#sN-img',  { opacity:1, duration:0.9, ease:'power1.inOut' }, offset + 0.05);
tl.fromTo('#sN-eye', { opacity:0, y:-16 }, { opacity:1, y:0, duration:0.35, ease:'power2.out' }, offset + 0.55);
tl.fromTo('#sN-hed', { opacity:0, y:40  }, { opacity:1, y:0, duration:0.65, ease:'power3.out' }, offset + 0.75);
```

#### Ken Burns effect (recommended for photos)

Add subtle camera motion to photos to give them life. Apply to the `<img>` element via GSAP on the main timeline:

```js
// Scale up gently over the scene duration — transformOrigin varies per scene for variety
tl.fromTo('#sN-img', { scale:1.0 }, { scale:1.06, duration: sceneDur, ease:'none' }, offset);
// Optionally drift vertically for a tilt-down feel:
tl.fromTo('#sN-img', { scale:1.0, y:0 }, { scale:1.05, y:-18, duration: sceneDur, ease:'none' }, offset);
```

Vary `transformOrigin` across scenes: `'center center'` (default) → `'right center'` → `'left bottom'`. This prevents adjacent scenes from feeling like the same motion.

#### ✅ Video backgrounds work in rendered output — with the right pattern

The earlier "autoplay" approach failed because the video element was placed inside a `.clip` div with no `data-track-index`. HyperFrames only manages videos it knows about.

**The correct pattern (fully supported in render):**
- `<video>` gets `data-start`, `data-duration`, `data-track-index` attributes
- `<video>` sits in a **plain non-timed wrapper div** — NOT inside a `.clip` div
- No `autoplay` attribute — HyperFrames owns media playback
- Videos must be re-encoded with dense keyframes before composing (see Video variant section below)

**The broken anti-pattern (avoid):**
```html
<!-- ❌ WRONG — autoplay + inside .clip + no data-track-index -->
<div id="s1" class="clip" data-start="0" data-duration="5" data-track-index="1">
  <video autoplay muted playsinline loop src="..."></video>
</div>

<!-- ✅ CORRECT — outside .clip, framework owns timing -->
<div style="position:absolute;inset:0;">
  <video id="v1" data-start="0" data-duration="5" data-track-index="0" muted playsinline src="..."></video>
</div>
<div id="s1" class="clip" data-start="0" data-duration="5" data-track-index="1">
  <!-- scrim + text only -->
</div>
```

#### ⚠️ Do NOT manually control `.clip` display

HyperFrames owns `display` on elements with `class="clip"`. It manages show/hide via `data-start`/`data-duration`.

**Never do this:**
```js
tl.set('#s2', { display: 'none' }, 0);  // ← gsap_animates_clip_element lint error
```

The lint rule `gsap_animates_clip_element` will catch this and fail. Trust `data-start`/`data-duration` to handle clip scheduling.

#### Video variant — rendered output ✅

> **This works in rendered output.** HyperFrames recognises `<video>` elements with `data-track-index` and seeks `video.currentTime` per frame during render (`videoCount` in compiled metadata will be > 0).

**Two critical rules for video elements:**
1. **NEVER nest `<video>` inside a `.clip` div** (a timed element). Use a plain non-timed wrapper `<div>`.
2. **Re-encode videos with dense keyframes** before composing — raw Pexels downloads have sparse keyframes that cause freeze artifacts. First detect source FPS, then re-encode every video:

```bash
# Detect source FPS (round fraction to int → SRC_FPS)
ffprobe -v quiet -select_streams v:0 -show_entries stream=avg_frame_rate -of csv=p=0 pexels-vN.mp4

ffmpeg -y -i pexels-vN.mp4 -c:v libx264 -r SRC_FPS -g SRC_FPS -keyint_min SRC_FPS -movflags +faststart -an pexels-vN-enc.mp4
```

**Structure — one video per scene, each with its own timing:**

```html
<!--
  VIDEO BACKGROUNDS — sit directly under the root composition, NOT inside .clip divs.
  HyperFrames recognises data-track-index and seeks video.currentTime during render.
  Wrap each <video> in a plain (non-timed) <div> for positioning only.
-->

<!-- Scene 1 video: 0-5s -->
<div style="position:absolute;inset:0;pointer-events:none;">
  <video id="v1" data-start="0" data-duration="5" data-track-index="0"
         muted playsinline src="assets/pexels-v1.mp4"
         style="position:absolute;inset:0;width:100%;height:100%;object-fit:cover;"></video>
</div>

<!-- Scene 2 video: 5-10s -->
<div style="position:absolute;inset:0;pointer-events:none;">
  <video id="v2" data-start="5" data-duration="5" data-track-index="0"
         muted playsinline src="assets/pexels-v2.mp4"
         style="position:absolute;inset:0;width:100%;height:100%;object-fit:cover;"></video>
</div>

<!-- Scene 3 video: 10-15s -->
<div style="position:absolute;inset:0;pointer-events:none;">
  <video id="v3" data-start="10" data-duration="5" data-track-index="0"
         muted playsinline src="assets/pexels-v3.mp4"
         style="position:absolute;inset:0;width:100%;height:100%;object-fit:cover;"></video>
</div>

<!-- Scrim + text overlays go in .clip divs ABOVE the videos -->
<div id="s1" class="clip" data-start="0" data-duration="5" data-track-index="1">
  <div style="position:absolute;inset:0;background:linear-gradient(90deg,rgba(5,5,5,0.92) 36%,rgba(5,5,5,0.08) 100%);"></div>
  <div class="scene-content" style="align-items:flex-start;max-width:900px;">
    <div id="s1-eye" style="font-size:20px;font-weight:700;letter-spacing:6px;color:#c0392b;text-transform:uppercase;opacity:0;">LABEL</div>
    <h2 id="s1-hed" style="font-family:Outfit,sans-serif;font-weight:900;font-size:96px;color:#e2e2e2;line-height:1.05;opacity:0;">
      The sentence text goes here.
    </h2>
  </div>
</div>
```

```js
// No need to fade-in videos — HyperFrames shows them at data-start.
// Just animate the text overlays inside .clip divs:
tl.fromTo('#s1-eye', { opacity:0, y:-16 }, { opacity:1, y:0, duration:0.35, ease:'power2.out' }, offset + 0.6);
tl.fromTo('#s1-hed', { opacity:0, y:40  }, { opacity:1, y:0, duration:0.65, ease:'power3.out' }, offset + 0.8);
```

**What happens in render:**
- HyperFrames extracts frames from each video via ffmpeg before capture (`videoCount:N` in metadata)
- It seeks each video to the correct time per captured frame
- Dense keyframes (GOP = SRC_FPS frames = keyframe every 1s at source FPS) are required for accurate seeking

#### Scrim direction guide

| Content position | Gradient direction |
|---|---|
| Text on left, photo bleeds right (default) | `90deg` — dark left, light right |
| Text on right, photo bleeds left | `270deg` — dark right, light left |
| Text at bottom, photo fills top | `0deg` (top→bottom) — transparent top, dark bottom |
| Text centered, full-frame glow | `radial-gradient(ellipse at 40% 50%, rgba(0,0,0,0.6) 0%, transparent 70%)` |

#### Query writing tips

| Content | Good query | Avoid |
|---|---|---|
| AI / tech | `"developer typing code dark monitor"` | `"AI"` (abstract) |
| Speed / performance | `"race car highway motion blur"` | `"fast"` |
| Collaboration | `"team meeting whiteboard office"` | `"people"` |
| Growth / scale | `"cityscape aerial drone night"` | `"growth"` |
| Precision / quality | `"watchmaker craft hands close"` | `"quality"` |

> **Pexels API key** should come from the `PEXELS_API_KEY` environment variable — never hard-code it in this skill or embed it in `index.html`.
> **Attribution:** Add *"Media provided by Pexels (pexels.com)"* to your video description or project README.

---

### 19. `presenter-aside`

Webcam / talking-head video pushed to the right half of the canvas. Text (eyebrow + headline + subtitle) fills the left half. Inspired by tutorial-style YouTube content where the presenter is visible but the frame is also doing work.

> **Pre-requisite:** You must have a local webcam recording (e.g. `assets/webcam.mp4`). Detect its FPS first (`ffprobe -v quiet -select_streams v:0 -show_entries stream=avg_frame_rate -of csv=p=0 webcam.mp4` — round fraction to int → SRC_FPS), then re-encode with dense keyframes: `ffmpeg -y -i webcam.mp4 -c:v libx264 -r SRC_FPS -g SRC_FPS -keyint_min SRC_FPS -movflags +faststart -an webcam-enc.mp4`

**Critical layout rules (same as pexels video variant):**
- The `<video>` element sits in a **plain non-timed wrapper div** — NEVER inside a `.clip` div.
- The `.clip` div covers the full canvas (`position:absolute;inset:0`) and only contains the left-side text.
- A gradient scrim on the left edge of the video blends the two halves.
- **⚠️ If there is a title card or any scene before the presenter video starts, add `display:none` to the wrapper div and reveal it with `tl.set('#v-wrap', { display:'block' }, VIDEO_START_TIME)`. Without this, the video wrapper bleeds through earlier scenes.**

```html
<!--
  PRESENTER VIDEO — right half, plain non-timed wrapper.
  id="v-wrap" starts hidden (display:none) to prevent bleed during any earlier scenes.
  Revealed at VIDEO_START_TIME via tl.set() in the GSAP timeline.
  No autoplay. No clip class.
-->
<div id="v-wrap" style="position:absolute;top:0;right:0;width:55%;height:100%;pointer-events:none;overflow:hidden;display:none;">
  <video id="vN" data-start="SCENE_START" data-duration="SCENE_DUR" data-track-index="0"
         muted playsinline src="assets/webcam-enc.mp4"
         style="width:100%;height:100%;object-fit:cover;object-position:center top;"></video>
  <!-- Left-edge scrim blends presenter into text area -->
  <div style="position:absolute;inset:0;
    background:linear-gradient(90deg,rgba(10,10,10,1) 0%,rgba(10,10,10,0.6) 20%,transparent 45%);"></div>
</div>

<!-- Full-canvas clip — text content on the left side only -->
<div id="sN" class="clip" data-start="SCENE_START" data-duration="SCENE_DUR" data-track-index="1">
  <div class="scene-content" style="align-items:flex-start;max-width:860px;">
    <div id="sN-eye" style="font-size:22px;font-weight:700;letter-spacing:6px;
      color:#c0392b;text-transform:uppercase;opacity:0;">INPUT</div>
    <h2 id="sN-hed" style="font-family:Outfit,sans-serif;font-weight:900;font-size:100px;
      color:#e2e2e2;line-height:1.05;margin:0;opacity:0;">
      Natural<br>language.
    </h2>
    <p id="sN-sub" style="font-size:44px;font-weight:400;color:#8899aa;margin:0;opacity:0;">
      Describe it. Claude builds it.
    </p>
  </div>
</div>
```

```js
// offset = scene startTime (VIDEO_START_TIME = same value)
// MUST reveal the video wrapper at the same time the presenter scene starts:
tl.set('#v-wrap', { display:'block' }, offset);
tl.fromTo('#sN-eye', { opacity:0, y:-20 }, { opacity:1, y:0, duration:0.3, ease:'power2.out' }, offset + 0.3);
tl.fromTo('#sN-hed', { opacity:0, y:50  }, { opacity:1, y:0, duration:0.65, ease:'power3.out' }, offset + 0.55);
tl.fromTo('#sN-sub', { opacity:0, y:30  }, { opacity:1, y:0, duration:0.4, ease:'power2.out' }, offset + 1.0);
```

**Layout variants:**

| Presenter position | Video wrapper | Scrim direction | Text max-width |
|---|---|---|---|
| Right (default) | `right:0; width:55%` | `90deg` dark-left | `max-width:860px` |
| Left | `left:0; width:55%` | `270deg` dark-right | align `flex-end`, `max-width:860px` pushed right |
| Right, smaller (more text room) | `right:0; width:40%` | `90deg` dark-left | `max-width:1050px` |

---

### 20. `hud-overlay`

Floating glassmorphism UI cards overlaid on a **background video** — mimics a live dashboard or editor HUD. Typically two cards: a **terminal/code card** (upper-left) and a **stat card** (upper-right). Optionally a central text block at the bottom. Great for "tool in action" or "what this does under the hood" sentences.

> **⚠️ CRITICAL — Background video is required.** HUD cards floating on a plain dark background look like a flat slide, not an overlay. Always add a real background video. Use the same `display:none` / `tl.set` reveal pattern as `presenter-aside`.

> **⚠️ CRITICAL — `tl.fromTo()` required for elements without inline `opacity:0`.** If you remove `opacity:0` from element inline styles (recommended), you MUST use `tl.fromTo('#el', { opacity:0 }, { opacity:1 })`. Using `tl.from('#el', { opacity:0 })` on an element that already has no inline opacity will animate 0→current (which is 1), so it works. But if an element has inline `opacity:0` and you use `tl.from()`, GSAP targets the inline value and animates 0→0 — the element stays invisible. **Rule: use `fromTo` whenever you specify an explicit destination.** 

> **⚠️ CRITICAL — Mini bars must start at `height:0%` in HTML.** GSAP cannot animate `height` from 0 to X if the element already has `height:30%` in its inline style. Set all bar heights to `height:0%` in HTML and animate to target with GSAP.

**Card anatomy:**
- Dark semi-transparent background (`rgba(10,10,18,0.90)`)
- Subtle border (`rgba(255,255,255,0.13)`)
- `backdrop-filter:blur(12px)` for glass effect
- Mono font for code lines; `#4da6ff` for command arrows; `#27ae60` green for "ok"

```html
<!-- ── BACKGROUND VIDEO (hidden until hud scene starts) ── -->
<!-- Place OUTSIDE all .clip divs, before scene divs -->
<div id="sN-vwrap" style="position:absolute;inset:0;display:none;z-index:0;">
  <video id="sN-bgvid" data-start="SCENE_START" data-duration="SCENE_DUR"
         data-track-index="0" muted playsinline
         style="width:100%;height:100%;object-fit:cover;"
         src="assets/bg.mp4"></video>
  <!-- Dark overlay — keeps cards readable over video -->
  <div style="position:absolute;inset:0;background:rgba(8,8,16,0.55);"></div>
</div>

<!-- ── HUD SCENE (no background div — video shows through) ── -->
<div id="sN" class="clip" data-start="SCENE_START" data-duration="SCENE_DUR" data-track-index="1">

  <!-- ── LEFT HUD CARD: terminal / code ── -->
  <div id="sN-left" style="position:absolute;top:80px;left:100px;width:520px;
    background:rgba(10,10,18,0.90);border:1px solid rgba(255,255,255,0.13);border-radius:14px;
    padding:28px 32px;backdrop-filter:blur(12px);">
    <!-- Title bar with traffic light dots -->
    <div style="display:flex;align-items:center;gap:10px;margin-bottom:18px;
      padding-bottom:14px;border-bottom:1px solid rgba(255,255,255,0.08);">
      <div style="width:10px;height:10px;border-radius:50%;background:#c0392b;"></div>
      <div style="width:10px;height:10px;border-radius:50%;background:#f39c12;"></div>
      <div style="width:10px;height:10px;border-radius:50%;background:#27ae60;"></div>
      <div style="font-family:'JetBrains Mono',monospace;font-size:15px;color:#4da6ff;
        font-weight:700;margin-left:8px;">editing.loop()</div>
    </div>
    <!-- Code lines (each individually animated) -->
    <div style="display:flex;flex-direction:column;gap:12px;">
      <div id="sN-l1" style="font-family:'JetBrains Mono',monospace;font-size:15px;color:#8899aa;
        display:flex;justify-content:space-between;">
        <span><span style="color:#4da6ff;">▸</span> analyze_clip <span style="color:#e2e2e2;">"intro.mp4"</span></span>
        <span style="color:#27ae60;font-size:12px;">ok</span>
      </div>
      <div id="sN-l2" style="font-family:'JetBrains Mono',monospace;font-size:15px;color:#8899aa;
        display:flex;justify-content:space-between;">
        <span><span style="color:#4da6ff;">▸</span> sync_captions <span style="color:#e2e2e2;">114 words</span></span>
        <span style="color:#27ae60;font-size:12px;">ok</span>
      </div>
      <div id="sN-l3" style="font-family:'JetBrains Mono',monospace;font-size:15px;color:#8899aa;
        display:flex;justify-content:space-between;">
        <span><span style="color:#4da6ff;">▸</span> add_motion_gfx <span style="color:#e2e2e2;">5 scenes</span></span>
        <span style="color:#27ae60;font-size:12px;">ok</span>
      </div>
      <div id="sN-l4" style="font-family:'JetBrains Mono',monospace;font-size:15px;color:#e2e2e2;
        display:flex;justify-content:space-between;">
        <span><span style="color:#4da6ff;">▸</span> render_frame <span style="color:#e2e2e2;">1920×1080</span></span>
        <span id="sN-cursor" style="color:#4da6ff;font-size:13px;">▌</span>
      </div>
    </div>
  </div>

  <!-- ── RIGHT HUD CARD: stat + animated bar chart ── -->
  <div id="sN-right" style="position:absolute;top:80px;right:100px;width:440px;
    background:rgba(10,10,18,0.90);border:1px solid rgba(255,255,255,0.13);border-radius:14px;
    padding:32px 36px;backdrop-filter:blur(12px);">
    <div style="font-family:Outfit,sans-serif;font-size:16px;font-weight:700;letter-spacing:4px;
      color:#8899aa;text-transform:uppercase;margin-bottom:8px;">monthly creator views</div>
    <div id="sN-stat" style="font-family:Outfit,sans-serif;font-weight:900;font-size:88px;
      color:#e2e2e2;line-height:1;margin-bottom:4px;">+0%</div>
    <!-- Mini bar chart — all bars START at height:0% (GSAP animates them up) -->
    <div style="display:flex;align-items:flex-end;gap:10px;height:80px;margin-top:20px;">
      <div id="sN-b1" style="flex:1;background:#c0392b;opacity:0.4;border-radius:3px 3px 0 0;height:0%;"></div>
      <div id="sN-b2" style="flex:1;background:#c0392b;opacity:0.5;border-radius:3px 3px 0 0;height:0%;"></div>
      <div id="sN-b3" style="flex:1;background:#c0392b;opacity:0.6;border-radius:3px 3px 0 0;height:0%;"></div>
      <div id="sN-b4" style="flex:1;background:#c0392b;opacity:0.75;border-radius:3px 3px 0 0;height:0%;"></div>
      <div id="sN-b5" style="flex:1;background:#c0392b;border-radius:3px 3px 0 0;height:0%;"></div>
    </div>
    <!-- Trend arrow — starts opacity:0 to hide polygon -->
    <svg id="sN-trend" width="100%" height="30" viewBox="0 0 360 30"
         style="margin-top:8px;opacity:0;" preserveAspectRatio="none">
      <polyline points="0,28 80,22 160,16 240,10 320,4"
        fill="none" stroke="#c0392b" stroke-width="2"
        stroke-dasharray="400" stroke-dashoffset="400"/>
      <polygon points="315,0 330,8 315,16" fill="#c0392b"/>
    </svg>
  </div>

  <!-- ── OPTIONAL: Central sentence text ── -->
  <div class="scene-content" style="align-items:center;text-align:center;
    justify-content:flex-end;padding-bottom:100px;">
    <div id="sN-hed" style="font-family:Outfit,sans-serif;font-weight:900;font-size:76px;
      color:#e2e2e2;max-width:1200px;line-height:1.2;">
      The sentence text goes here.
    </div>
    <div id="sN-sub" style="font-family:Outfit,sans-serif;font-weight:400;font-size:40px;
      color:#8899aa;">Supporting detail line.</div>
  </div>

</div>
```

```js
// offset = scene startTime (same value as SCENE_START above)

// Reveal background video at scene start
tl.set('#sN-vwrap', { display:'block' }, offset);

// Cards drop in from above (use fromTo — explicit opacity:1 destination)
tl.fromTo('#sN-left',  { opacity:0, y:-30 }, { opacity:1, y:0, duration:0.55, ease:'expo.out' }, offset + 0.2);
tl.fromTo('#sN-right', { opacity:0, y:-30 }, { opacity:1, y:0, duration:0.55, ease:'expo.out' }, offset + 0.4);

// Code lines appear one by one (typewriter feel)
tl.fromTo('#sN-l1', { opacity:0 }, { opacity:1, duration:0.2 }, offset + 0.7);
tl.fromTo('#sN-l2', { opacity:0 }, { opacity:1, duration:0.2 }, offset + 1.2);
tl.fromTo('#sN-l3', { opacity:0 }, { opacity:1, duration:0.2 }, offset + 1.7);
tl.fromTo('#sN-l4', { opacity:0 }, { opacity:1, duration:0.2 }, offset + 2.2);

// Blinking cursor after last line (finite repeat — calculate from remaining scene time)
tl.to('#sN-cursor', { opacity:0, duration:0.35, repeat:8, yoyo:true, ease:'none' }, offset + 2.4);

// Stat number counts up
var statObj = { val: 0 };
tl.to(statObj, {
  val: 248, duration: 1.0, ease: 'power2.out',
  onUpdate: function() {
    var el = document.getElementById('sN-stat');
    if (el) el.textContent = '+' + Math.round(statObj.val) + '%';
  }
}, offset + 0.6);

// Mini bars grow from 0% (height:0% already set in HTML)
var barTargets = ['30%','45%','58%','75%','100%'];
['sN-b1','sN-b2','sN-b3','sN-b4','sN-b5'].forEach(function(id, i) {
  tl.to('#' + id, { height: barTargets[i], duration:0.4, ease:'power2.out' }, offset + 0.6 + i * 0.08);
});

// Trend arrow draws in
tl.set('#sN-trend', { opacity:1 }, offset + 1.2);
tl.to('#sN-trend polyline', { strokeDashoffset:0, duration:0.7, ease:'power2.out' }, offset + 1.2);

// Central text fades in last
tl.fromTo('#sN-hed', { opacity:0, y:30 }, { opacity:1, y:0, duration:0.5, ease:'power2.out' }, offset + 3.0);
tl.fromTo('#sN-sub', { opacity:0, y:20 }, { opacity:1, y:0, duration:0.4, ease:'power2.out' }, offset + 3.5);
```

**Customisation notes:**
- **Background video** — detect source FPS first (`ffprobe -v quiet -select_streams v:0 -show_entries stream=avg_frame_rate -of csv=p=0 in.mp4`), then re-encode: `ffmpeg -y -i in.mp4 -c:v libx264 -r SRC_FPS -g SRC_FPS -keyint_min SRC_FPS -movflags +faststart -an assets/bg.mp4`
- **Left card** — swap `editing.loop()` for any process name. Add/remove `<div id="sN-lN">` lines to match scene duration.
- **Right card** — change stat number, eyebrow label, and `barTargets` array to match actual data.
- **Trend arrow** — starts `opacity:0` to hide polygon (same rule as `flow-steps` arrows).
- **Overlay darkness** — adjust `rgba(8,8,16,0.55)` on the dark overlay div. Higher alpha = darker = more readable cards.

**Reference:** `C:\openclaw_files\projects\hud-overlay-sample\index.html`
