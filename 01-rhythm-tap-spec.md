# Spec 1 of 3 · Rhythm Tap (เกมตบจังหวะ)

You are building a complete single-file web game for first-year music performance students (Chulalongkorn University, Faculty of Fine and Applied Arts, Western music). Read the whole spec, then build it. Where this spec is silent, choose the simplest behavior that satisfies the constraints.

**One-line summary.** The game plays a one-bar rhythm (4 beats) as sound, then the player taps it back. Each tap is scored by how many milliseconds it is from the exact moment. 10 rounds per game, 3 difficulty levels.

**Page `<title>`:** `เกมตบจังหวะ`
**Primary job:** train rhythmic accuracy and listening, and show the player precisely how early or late they are.

## Hard constraints
These are hard requirements. If a requirement conflicts with a convenience, the requirement wins.

**Delivery**
- Deliver ONE file: a complete, self-contained `index.html` (HTML + inline `<style>` + inline `<script>`). No other files, no build step, no framework, no modules. Vanilla JavaScript (ES2019 or older syntax is safest), no TypeScript.
- Return it as a single code block, followed by a short self-check list (see "Acceptance tests": mark each PASS or FAIL honestly). Do not return partial files or "rest of the code stays the same".

**Environment this must run in** (all three, with no code changes)
1. Double-clicked from disk (`file://`) on a school Windows lab PC, Chrome or Edge.
2. Served from a static host: GitHub Pages at `https://user.github.io/repo-name/` (a subpath), Netlify, and Vercel.
3. A phone browser (iOS Safari, Android Chrome) at 360–430 px width, portrait.

**Forbidden**
- Any network request at runtime: no CDN scripts, no Google Fonts, no external images, no analytics, no `fetch`, no `XMLHttpRequest`, no WebSocket, no service worker, no `<link rel=stylesheet>` to another file.
- Calling any AI, LLM or API at runtime.
- Microphone, camera, geolocation, notifications.
- `alert()`, `confirm()`, `prompt()`, `window.print()`. Any confirmation must be built into the page.
- Infinite or continuous CSS animation (`animation-iteration-count: infinite` is banned). Transitions are fine and must be 200 ms or less. A `requestAnimationFrame` loop is allowed only while a game or visualization is actively running and must stop when it ends or the tab is hidden. Respect `prefers-reduced-motion`.
- Absolute-root URLs such as `/style.css` or `/img/x.png` (they break on GitHub Pages subpaths). Use no external URLs at all.
- Collecting or displaying personal data: there is no name, email, account or login field anywhere.
- Any code that deletes files (not applicable in a browser, listed for completeness).

**Language**
- Every string the user sees is Thai, taken from the "Thai copy" table of this spec. Code comments are Thai. Identifiers, CSS classes and technical terms stay English.
- `<html lang="th">`.

**Storage**
- If the app stores anything, use `localStorage` wrapped in `try/catch` on every read and write, with an in-memory fallback so the app still works when storage is blocked (private window, locked-down school browser). When storage is unavailable, show one small Thai notice (the exact text is in the Thai copy table) and keep working.

**Audio** (when the app makes sound)
- Web Audio API only, sound synthesized with oscillators. No audio files, no `<audio>` elements.
- Create the `AudioContext` lazily inside the first user gesture handler and call `resume()` on it. Never autoplay.
- Master gain starts low (see each spec). Every note has a short attack and release ramp (5–20 ms) so there are no clicks or pops.
- Use `ctx.currentTime` scheduling for anything timing-sensitive; never use `setTimeout` to trigger a sound at an exact moment.
- If the tab becomes hidden (`visibilitychange`), stop any running game or timing round safely.

**Quality bar**
- Works at 320 px width without horizontal page scroll. Landscape phone is usable.
- All interactive targets are at least 44 × 44 px. Tap targets do not trigger text selection or double-tap zoom (use `touch-action: manipulation` on controls).
- Total file size under 150 KB.
- No console errors or warnings on load or during any normal use.

## 1. Screen layout (single screen, portrait first)

```
┌────────────────────────────────────┐
│ เกมตบจังหวะ                 รอบ 3/10 │  header: title + round counter
│ [ง่าย][กลาง][ยาก]   [60][80][100]   │  level segmented + BPM segmented
├────────────────────────────────────┤
│  ┃ 1     ┃ 2     ┃ 3     ┃ 4     ┃  │  MEASURE stage (signature element)
│  ┃  ●    ┃  ●    ┃       ┃  ●    ┃  │  onset markers on a 4-beat bar
│  ┃       ┃       ┃       ┃       ┃  │
│  สถานะ: "ฟังให้ดี…"                  │  status line (aria-live)
├────────────────────────────────────┤
│            ┌──────────┐            │
│            │   ตบ     │            │  TAP PAD: large circle (min 168 px)
│            └──────────┘            │
│        กด Space หรือแตะวงกลม         │  hint
├────────────────────────────────────┤
│ ผลรอบนี้: 87 คะแนน                   │  result strip (after each round)
│ ●+12 ms ช้า  ●−48 ms เร็ว  ○ ไม่ได้ตบ │
├────────────────────────────────────┤
│ [ฟังอีกครั้ง]   [เริ่มใหม่]           │
│ ตั้งค่า ▾  (ระดับเสียง, ชดเชยความหน่วง)│
└────────────────────────────────────┘
```

All sections live on one scroll. Order top to bottom: header, stage card, tap pad, result strip, action row, settings (collapsed by default, a `<details>` element), footer.

## 2. The measure stage (signature element)

The signature element is a drawn **bar of music**: a rounded `--surface` card containing a 4-beat measure.
- Draw ONE horizontal timeline in the middle of the card with **barlines** (vertical 2 px `--line-strong` rules) at the left edge, the right edge, and a thin 1 px rule at each beat 2, 3, 4. Beat numbers 1 2 3 4 sit above their rules in `--muted`.
- Onset markers: filled circles (notehead style, 22 px) at their exact proportional x position (time within the bar). A beat position `b` (0 to 3.5) maps to `x = b / 4 × width`.
- Marker colors: expected onsets while listening are `--blue`; while it is the player's turn they are shown as **outlined** circles (2 px `--blue` border, transparent fill) so the player's taps can be compared.
- During playback of the pattern, the marker for the note being played gets class `.on` (fill `--action`, scale 1.2 via a 120 ms transition) at the exact scheduled moment (drive it with a `requestAnimationFrame` loop comparing `ctx.currentTime` to scheduled times, not with `setTimeout`).
- A thin vertical **playhead** (2 px, `--action`) moves across the bar during the pattern and during the player's turn. It is drawn by the same `requestAnimationFrame` loop that is running only while a round is active.
- After the player's turn, each tap is drawn as a filled circle on the same timeline at its actual position (clamped to the bar), colored by accuracy: `--good` for 100 points, `--warn` for 40–99 points, `--bad` below 40 points. Draw a short vertical connector from each tap to its matched expected marker. Label each matched tap beneath the line with signed milliseconds, e.g. `+12 ms` (late is positive). Taps that matched nothing are `--bad` with the label `เกิน`. Expected notes with no tap show an `--bad` outlined circle with a small `×` and the label `ขาด`.
- The stage must remain readable at 320 px width: marker size 18 px at narrow widths, labels 12.8 px.

**The one motion moment:** the playhead and the marker flash during playback. Nothing else animates.

## 3. Game rules

**Setup**
- Levels (`ง่าย`, `กลาง`, `ยาก`), default `ง่าย`. BPM options 60, 80, 100, default 80. Changing level or BPM during a game ends that game and returns to idle (no confirm dialog: the controls are disabled while a round is running, enabled between rounds and at idle).
- A game is 10 rounds. A round uses one pattern chosen at random from the level's pool, never the same pattern twice in a row.

**Patterns.** Beat positions are measured in beats from the start of the bar (beat 1 = 0). A beat lasts `60 / BPM` seconds.

| Level | Pool (each array is one pattern) |
|---|---|
| ง่าย | `[0,1,2,3]` · `[0,2]` · `[0,1,2]` · `[1,2,3]` · `[0,2,3]` |
| กลาง | `[0,0.5,1,2]` · `[0,1,2,2.5]` · `[0,1,1.5,3]` · `[0,2,2.5,3]` |
| ยาก | `[0,0.5,1.5,2,2.5,3.5]` · `[0.5,1,2,2.5,3]` · `[0,1.5,2,3,3.5]` · `[0,0.5,1,2.5,3.5]` |

**Round flow** (state machine: `idle → countIn1 → listen → countIn2 → play → scored → (next round | gameOver)`)
1. `countIn1`: four soft clicks, one per beat (the "1 2 3 4" count). Status: `นับ 1-2-3-4`.
2. `listen`: the pattern is played with a bright tone (880 Hz sine, 70 ms). Status: `ฟังให้ดี…`. The measure markers are filled `--blue` and flash as they sound.
3. `countIn2`: four soft clicks again. Status: `เตรียมตบ…`.
4. `play`: the player taps. Status: `ตาคุณ! ตบตามเลย`. The tap window opens at the start of this bar and closes at the end of the bar plus 300 ms.
5. `scored`: show the result strip, the comparison on the stage, and one primary button. It reads `รอบถัดไป` after rounds 1–9 and `ดูคะแนนรวม` after round 10.
6. After round 10: `gameOver` panel (see section 5).

Click sounds: the count-in click is a 220 Hz sine, 40 ms, quiet. The accented first click of each count is 440 Hz.

**Taps.** Valid inputs: `pointerdown` on the tap pad (use `pointerdown`, not `click`, to avoid the 300 ms delay and capture the earliest timestamp), and `keydown` for the Space key (ignore `event.repeat`; call `preventDefault()` so the page does not scroll). The pad plays a short wood-block style feedback tone (1100 Hz triangle, 30 ms) on every tap so the player hears themselves; this feedback must not be heard during `listen` or when no round is in `play` (taps outside `play` do nothing and produce no sound).

**Timing (important).**
- All timing is based on the Web Audio clock. When a round starts, schedule every click and every pattern note with `ctx.currentTime` plus offsets. Record one anchor pair at that moment: `anchor = { audio: ctx.currentTime, perf: performance.now() }`.
- A tap's audio-time is `anchor.audio + (event.timeStamp − anchor.perf) / 1000`, then subtract the player's latency compensation (section 4). `event.timeStamp` for pointer and key events is on the same clock as `performance.now()`.
- Expected onset time for a pattern note at beat `b` is `playBarStart + b × (60 / BPM)`, where `playBarStart` is the audio time at which the `play` bar begins (right after `countIn2`).

**Scoring.**
- `window = min(0.25 s, 0.45 × smallestGapBetweenExpectedOnsets)` where the smallest gap is computed from the round's pattern (for a single-note pattern use 0.25 s). At 100 BPM with eighth notes the gap is 300 ms so the window is 135 ms.
- Match taps to expected onsets greedily by smallest absolute error, one tap per onset, only when the error is within `window`.
- For a matched pair with absolute error `e` (seconds): `points = e ≤ 0.035 ? 100 : round(100 × (1 − (e − 0.035) / (window − 0.035)))`, clamped to 0–100.
- A missed expected onset scores 0. A tap that matched nothing is an "extra".
- `roundScore = clamp(round(mean(points over all expected onsets)) − 10 × extras, 0, 100)`.
- Labels for each tap error: `|error| ≤ 35 ms` → `ตรงเป๊ะ`, otherwise `เร็วไป` (negative error) or `ช้าไป` (positive).
- `gameTotal` is the sum of the 10 round scores (0–1000). Store the best total per level and BPM in `localStorage` under the key `rhythm-tap-best-v1`: an object `{ "easy-80": 842, ... }`. Show it as `คะแนนสูงสุด` on the game-over panel.

## 4. Settings (inside a `<details>` titled `ตั้งค่า`)

- **ระดับเสียง** (volume): range 0–100, default 25. Master gain = value / 100 × 0.8.
- **ชดเชยความหน่วงของเสียง** (latency compensation): range −200 to +200 ms, step 10, default 0. Shown with the live value. Bluetooth headphones add delay, so this matters. Persist it in `localStorage` (`rhythm-tap-latency-v1`).
- **วัดความหน่วงให้อัตโนมัติ** (auto-calibrate, "Should have"): a button that plays 8 steady clicks at 80 BPM and asks the player to tap each one; the median of the 8 signed errors becomes the new compensation value (clamped to ±200). Status text while running: `แตะตามเสียงคลิก (ครั้งที่ n/8)`. After: `ตั้งค่าชดเชยเป็น … ms`.
- **ทดสอบเสียง** (sound test): plays one 880 Hz note so the player can check the volume.

## 5. Game-over panel (in-page, not a dialog)

Replace the stage area content with: title `จบเกม`, the total score as a 64 px numeral with `/ 1000`, a one-line comment chosen by total: ≥ 900 `แม่นมาก ใกล้เมโทรนอมแล้ว`; ≥ 700 `ดีมาก ฟังและตบได้แม่น`; ≥ 450 `ใช้ได้ ลองลดความเร็วแล้วฝึกต่อ`; else `ฝึกต่ออีกนิด เริ่มจากระดับง่ายก่อน`. Show the best total, the average signed error across the whole game (e.g. `โดยเฉลี่ยตบช้ากว่าจังหวะ 14 ms`, or `เร็วกว่า`), and a primary button `เล่นอีกครั้ง`.

## 6. Thai copy (use these strings exactly)

| Key | Text |
|---|---|
| title | `เกมตบจังหวะ` |
| round | `รอบ {n}/10` |
| level labels | `ง่าย` · `กลาง` · `ยาก` |
| bpm label | `ความเร็ว` ; options `60` `80` `100` ; unit `BPM` |
| start button (idle) | `เริ่มเล่น` |
| status idle | `กดเริ่มเล่น แล้วฟังจังหวะให้ดี` |
| status countIn1 | `นับ 1-2-3-4` |
| status listen | `ฟังให้ดี…` |
| status countIn2 | `เตรียมตบ…` |
| status play | `ตาคุณ! ตบตามเลย` |
| pad label | `ตบ` |
| pad hint | `กด Space หรือแตะวงกลม` |
| replay | `ฟังอีกครั้ง` (allowed only in `scored`, replays the pattern without changing the score) |
| restart | `เริ่มใหม่` |
| next | `รอบถัดไป` · `ดูคะแนนรวม` |
| result title | `ผลรอบนี้: {n} คะแนน` |
| tap labels | `ตรงเป๊ะ` · `เร็วไป` · `ช้าไป` · `เกิน` · `ขาด` |
| milliseconds | `{+/−n} ms` |
| game over | `จบเกม` · `เล่นอีกครั้ง` · `คะแนนสูงสุด` |
| settings | `ตั้งค่า` · `ระดับเสียง` · `ชดเชยความหน่วงของเสียง` · `วัดความหน่วงให้อัตโนมัติ` · `ทดสอบเสียง` |
| storage notice | `เบราว์เซอร์นี้เก็บคะแนนสูงสุดไม่ได้ เกมยังเล่นได้ตามปกติ` |
| footer | `เกมตบจังหวะ · สร้างด้วย AI ช่วย · วิชา 3500230` |

## 7. Design for this app

Follow the shared design system below. App-specific choices:
- Accent for this app is `--action` (the vermilion). It marks the playhead, the pad and the primary button. `--blue` marks the expected rhythm. `--brass` is not used.
- The **tap pad** is the hero: a circle, 176 px on phones (up to 220 px on wide screens), `--action` fill, label `ตบ` at 28 px weight 700 in `--action-ink`. Pressed state: scale .97 and a 2 px inset ring in `--action-ink` at 40% opacity (60 ms transition). It sits centered with generous space (24 px) above and below, and it must be reachable by a thumb in portrait.
- The measure card has the barline rules and the beat numbers as its only ornament. Keep it flat. The result strip below uses small circles (10 px) as legend dots.
- Level and BPM controls are segmented controls; the selected state is clear without color alone (filled surface + outline).
- During `play`, the stage card gets a 2 px `--action` outline as the "your turn" cue (a transition, not an animation), in addition to the status text.

## Design system (shared)
**Direction.** The app belongs to a university music course (Faculty of Fine and Applied Arts), used by performance majors. It should feel like a well-made practice tool from a music publisher, not a generic web app: calm, precise, a little warm, with one signature visual element drawn from music notation. Avoid these generic looks: purple-to-blue gradient heroes, emoji used as icons or section markers, everything centered in a column of identical rounded cards, neon-on-black, a cream-and-terracotta "AI" palette, decorative blobs. Use inline SVG or plain text symbols instead of emoji.

**Color tokens.** Define every color as a CSS custom property on `:root` (light values). Redefine only the tokens for dark mode under `@media (prefers-color-scheme: dark) { :root:not([data-theme="light"]) { ... color-scheme: dark } }`. Do not hard-code colors in components. Body has an explicit `background: var(--bg)`.

| Token | Light | Dark | Use |
|---|---|---|---|
| `--bg` | `#dfe9ef` | `#0f1a21` | page background (a pale blue-grey field) |
| `--surface` | `#ffffff` | `#172630` | cards |
| `--surface-2` | `#f3f8fa` | `#1c2f3b` | inset areas, inputs |
| `--line` | `#c6d6e0` | `#2c4150` | borders, rules |
| `--line-strong` | `#b5c8d4` | `#3a5263` | staff lines, strong rules |
| `--text` | `#17252f` | `#e6eef3` | primary text |
| `--text-soft` | `#33454f` | `#c3d3dc` | secondary text |
| `--muted` | `#5f7584` | `#8ea3b1` | captions (large or bold text only on `--bg`) |
| `--action` | `#c4421f` | `#f07a5a` | primary buttons, key accents |
| `--action-ink` | `#ffffff` | `#1b0e09` | text on `--action` |
| `--action-soft` | `#fdece7` | `#3a221b` | soft highlights |
| `--blue` | `#3f6f8d` | `#8cb8d3` | secondary accent, links |
| `--blue-soft` | `#e3eef5` | `#1d3442` | info areas |
| `--brass` | `#9a6a1f` | `#d9a94f` | third accent (used by one app only, see its spec) |
| `--good` | `#2e7d59` | `#7fd1a6` | success, accurate |
| `--warn` | `#9a6a1f` | `#e3b04b` | caution, close |
| `--bad` | `#b3261e` | `#ff8a80` | error, off |

Text and its background must reach 4.5:1 contrast (3:1 for text of 24 px and above). Measured ratios for the values above: every text token on `--surface` passes in both themes, and `--action-ink` on `--action` is 5.06:1 (light) and 6.86:1 (dark). Two limits in the light theme: `--muted` on `--bg` is only 3.9:1 and `--blue` on `--bg` is 4.4:1, so on the page background use them only for large or bold text, and set normal-size captions and links on `--surface` cards. `--brass` on `--surface` is 4.7:1 (light), fine for numerals and labels.

**Typography.** No web fonts. Use the system stack `"Sarabun", "Leelawadee UI", "Noto Sans Thai", Thonburi, Tahoma, system-ui, sans-serif` for everything. Numbers that change or align (timers, scores, milliseconds) use `font-variant-numeric: tabular-nums` and weight 700. Monospace stack `ui-monospace, "Cascadia Mono", Menlo, Consolas, monospace` only for small technical readouts. Scale: 12.8 / 14.4 / 16 / 18 / 22 / 28 / 40 / 64 px. Body line-height at least 1.6 for Thai (Thai needs more vertical room than Latin); large numerals 1.15. Never add `letter-spacing` or uppercase transforms to Thai text. Headings use `text-wrap: balance`.

**Space and shape.** Spacing scale 4 / 8 / 12 / 16 / 24 / 32 px; lay out with flex or grid and `gap`. Radii: 12 px (controls), 18 px (cards), 999 px (pills). Borders 1 px `--line`. Use shadow almost never (at most one very soft shadow on the single hero element). Page gutter at least 16 px at every width: `padding-inline: max(16px, env(safe-area-inset-left))` style. Max content width 560 px, centered on wide screens.

**Components.**
- Primary button: filled `--action`, text `--action-ink`, min-height 52 px, radius 12 px, weight 700. Pressed: `transform: scale(.98)`. Secondary button: `--surface-2` fill, 1 px `--line` border. Disabled: 50% opacity, no pointer cursor.
- Segmented control (for levels and options): a row of buttons in one `--surface-2` track, the selected one filled `--surface` with a 1 px `--blue` outline, built from real `<button>` elements with `aria-pressed`.
- Focus: `:focus-visible { outline: 3px solid var(--blue); outline-offset: 2px }` everywhere.
- Feedback colors always come with a text label or shape, never color alone.
- Empty, loading and error states are designed, with one plain Thai sentence that says what to do next.

**Signature detail.** Each app has one signature element drawn from notation (stated in its spec). Draw it with CSS or inline SVG. Keep everything else quiet so that element carries the identity.

**Motion.** One purposeful moment only (stated in each spec). Everything else is static or a short transition. No bouncing, parallax or looping ambient motion.

## 8. Edge cases

- First tap before audio is unlocked: the Start button is the gesture that creates and resumes the AudioContext; there is no earlier tap.
- Tab hidden or window blurred during a round: stop sound, cancel the round, return to `idle` with status `หยุดชั่วคราว กดเริ่มเล่นเพื่อเริ่มใหม่`, keep the score of finished rounds out of the total (the game restarts).
- A tap arrives within the window of two expected onsets: assign it to the closer one.
- Double taps within 20 ms count as separate taps (touch devices can emit them); each is judged normally.
- `localStorage` blocked: show the storage notice once, keep the best score in memory for this session.
- Very small screens (320 px): the BPM and level segmented controls wrap to two rows instead of overflowing.
- Window resize during a round: the stage recomputes marker positions from percentages, no restart.

## 9. Acceptance tests

Mark each PASS or FAIL in your self-check.
1. Opens from `file://` and shows the idle screen with no console errors.
2. Pressing `เริ่มเล่น` plays the 4-click count-in, then the pattern, then a second count-in, with the status line changing as specified.
3. No sound plays before the first user gesture.
4. The pattern played for `ง่าย` is always one of the five listed patterns, and the same pattern never repeats twice in a row (verify by reading the code path that picks it).
5. A tap placed exactly on an expected onset (simulate: error 0) scores 100 points for that note; an error of 35 ms also scores 100; an error equal to `window` scores 0.
6. A tap outside every window counts as `เกิน` and subtracts 10 from the round score; a missing tap shows `ขาด` and scores 0 for that note.
7. Space and `pointerdown` on the pad both register taps; holding Space does not register repeats; taps outside the `play` state do nothing.
8. After round 10 the game-over panel shows the total out of 1000, the best score, the average signed error and `เล่นอีกครั้ง`.
9. The best score persists after reload when `localStorage` works, and the game works with `localStorage` throwing errors.
10. The latency compensation changes the computed error by the entered number of milliseconds.
11. Layout has no horizontal scroll at 320 px and the pad is at least 168 px wide.
12. Light and dark themes both render with correct contrast, and `prefers-reduced-motion` removes the marker scale transition and the stage outline transition.
13. The file makes no network requests and contains no `alert`, `confirm`, `prompt`, `infinite` animation, CDN URL or external font.

## Deployment readiness
The instructor will upload this single file to GitHub Pages, Netlify Drop and Vercel, then open the link on a phone. Make that frictionless:

- The file is named `index.html` and has no dependencies, so uploading the file alone (or a folder containing only it) is enough.
- `<head>` includes: `<meta charset="utf-8">`, `<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">`, a specific Thai `<title>` (given in the spec), `<meta name="description">` (one Thai sentence), `<meta name="theme-color">` for both schemes, `<meta name="color-scheme" content="light dark">`, and a favicon as an inline SVG data URI: `<link rel="icon" href="data:image/svg+xml,...">` (a simple musical motif in the app's accent color; percent-encode `#` as `%23`).
- Open Graph basics without an image: `og:title`, `og:description`, `og:type` = `website`, `og:locale` = `th_TH`.
- No `<base>` tag, no root-relative URLs, no hash-routing, no history API. The page is one screen (or sections on one scroll) so it behaves identically at `/` and `/repo-name/`.
- Never rely on features that fail on `file://`: no ES modules, no `fetch`, no service worker. (`localStorage` is allowed but guarded.)
- A small footer line in Thai: the app name and "สร้างด้วย AI ช่วย · วิชา 3500230" (exact string in the Thai copy table).

## Output format
Return:
1. One code block containing the complete `index.html`.
2. Under it, a "Self-check" list: every acceptance test from this spec, each marked PASS or FAIL with one short reason. If something cannot be satisfied, say so plainly instead of marking PASS.
3. Nothing else: no alternative versions, no explanations of the code, no suggestions for extra features.
