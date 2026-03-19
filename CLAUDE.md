# CLAUDE.md — Adi's Daily Schedule

## Project Goal

A lightweight personal productivity site to bring structure to an unemployed day across four core ambitions:

1. **LSAT Prep** — daily study blocks, weekly practice test on Saturday, review on Sunday
2. **Job Search** — active applications, networking, follow-ups, interview prep
3. **Substack Writing** — drafting, editing, publishing essays
4. **Claude Code** — building personal projects (Your Ballot 2026, this site, etc.)

Plus supporting categories: Running + Shower, Reading (wind-down), and Misc (catch-all for meals, one-off overrides, etc.).

Target audience: one user (Adi). No auth, no backend, no database. Static site.

---

## Running Locally

Open the HTML file directly — no server needed (no API calls, no CORS issues).

```bash
open "/Users/adibhalla/Downloads/Adi Schedule (unemployed)/index.html"
```

Or via HTTP if needed:
```bash
cd "/Users/adibhalla/Downloads/Adi Schedule (unemployed)"
python3 -m http.server 8080
# → http://localhost:8080
```

---

## Deployment

Hosted on **GitHub Pages** — repo: `https://github.com/aditya2001adi/adi-schedule`

Push to deploy:
```bash
cd "/Users/adibhalla/Downloads/Adi Schedule (unemployed)"
git add index.html
git commit -m "your message"
git push
```

Live URL: `https://aditya2001adi.github.io/adi-schedule/`

Auth: uses a GitHub Personal Access Token (PAT) stored in macOS Keychain — no password needed after first push.

---

## File Structure

```
Adi Schedule (unemployed)/
├── index.html    # Entire app — HTML, CSS, JS, data, all in one file
└── CLAUDE.md     # This file
```

Everything lives in `index.html`. No build step, no dependencies, no `node_modules`.

---

## Tech Stack

- **React 18** (via CDN, UMD build) + **Babel standalone** for JSX in-browser
- **No bundler** — single file, works offline
- **localStorage** for all persistence (checks, overrides, swaps, custom blocks, time mutations, history)
- **JSONBin.io** for cross-device sync — REST API, API key in HTML, free tier
- Fonts: Georgia (body), monospace (times/labels)

---

## What's Been Built

### 1. Weekly Schedule View
- 7-day tab navigation (Mon–Sun), auto-jumps to today on load
- Amber pip under today's tab so you always know which day is "now"
- Each day has a theme ("Deep Work Day", "Scholarship Day", etc.)
- Color-coded blocks by category (amber=LSAT, sky=jobs, violet=Substack, emerald=code, rose=reading, orange=run, pink=misc)

### 2. NOW Highlight
- Detects current time, highlights the active time block with a glow border
- Pulsing "NOW" badge on the active block
- Updates every 60 seconds via `setInterval`

### 3. Block Check-off
- Circle button on each non-break block — click to mark done
- Checked blocks dim + strikethrough
- Persisted in localStorage, keyed by ISO week → resets automatically each Monday
- Storage key: `schedule-v1-{weekKey}` (e.g. `schedule-v1-2026-W12`)
- Check keys use slot position: `{dayName}-{slotIndex}` for hardcoded, `{dayName}-cb_{id}` for custom blocks

### 4. Block Override UI
- **Click any block** (not the check button) → modal popup appears
- Modal shows 7 category buttons + "Misc / Other…"
- Picking a category instantly recolors the block to the new category
- "Misc / Other…" opens a text input — type what you actually did, press Enter
- Overridden blocks show `↺ was [original activity]` label
- "Reset to original" appears in the modal if an override exists
- Swapped hours count toward the **new** category in the scoreboard

Override storage: `schedule-overrides-v1` in localStorage
Override key format: `{ISO-date}-{slotIndex}` for hardcoded, `{ISO-date}-cb_{id}` for custom

### 5. Weekly Scoreboard
- Bottom of page — shows actual vs. target hours per category with a progress bar
- Hours accumulate from checked blocks; respects overrides and swaps
- "X / Y blocks done this week" counter
- Targets: LSAT 12h, Jobs 10h, Substack 4h, Code 5h, Reading 7h, Run 6h

### 6. Weekly History
- "Hist" tab shows archived past weeks with per-category hours
- Archived automatically on the first page load of a new week
- Storage key: `schedule-history-v1`

### 7. Cross-Device Sync (JSONBin.io)
- On page load: fetches remote state, merges into localStorage
- On any change: 2-second debounce push to JSONBin
- Sync status indicator in the tab bar (loading / syncing / synced / offline / error)
- Payload syncs: `checks`, `overrides`, `history`, `swaps`, `custom`, `timeMutations`
- localStorage is the offline/fallback cache; JSONBin is source of truth
- Credentials live in `index.html` lines ~237–238 (BIN_ID and API_KEY)

### 8. Drag-and-Drop Block Swap
- Drag any non-break hardcoded block and drop it onto another to swap their activities
- Time slots stay fixed; only the content (activity/category/detail) moves
- Swaps are date-specific — Monday this week's swap doesn't affect next Monday
- "Reset day order" button in the override modal undoes all swaps for that day
- Custom blocks are not draggable (they sort by time)
- Storage key: `schedule-swaps-v1`
- Key format: `{ISO-date}-{dayName}` → value: permutation array of original block indices

### 9. Add Custom Block
- "+ Add block" button at the bottom of each day's list
- User enters start time, end time, activity name, category, optional note
- Any overlapping existing blocks are trimmed to make room (trim-to-fit model)
- Fully covered existing blocks are hidden for that date only
- Custom blocks are marked with a small `+ added` label
- "Remove this block" in the override modal removes the custom block and restores any trimmed blocks
- Storage key: `schedule-custom-v1`
- Custom block ID format: `cb_{timestamp}` — never shifts hardcoded block indices
- Key format: `{ISO-date}-{dayName}` → array of custom block objects

### 10. Edit Block Time
- "Edit time" button in the override modal (works for both hardcoded and custom blocks)
- Pre-filled with the block's current start and end times
- Adjacent blocks adjust automatically:
  - Move start **later** → previous block's end extends to match
  - Move start **earlier** → previous block's end trims to match
  - Move end **later** → next block's start trims to match
  - Move end **earlier** → next block's start extends to match
- If an adjacent block is squeezed to zero duration, it hides for that day
- Time mutations stored in `schedule-timemut-v1`
- Key format: `{ISO-date}-{dayName}-{slotIndex}` → `{newStart, newEnd}` or `{hidden: true, causedBy?}`

---

## Key Data Structures

### Schedule (hardcoded in JS)
```js
const schedule = {
  Monday: {
    theme: "Deep Work Day",
    blocks: [
      { time: "11:00am – 1:30pm", activity: "LSAT Prep", detail: "...", category: "lsat" },
      // ...
    ]
  },
  // Tuesday through Sunday
};
```

Categories: `lsat`, `jobs`, `substack`, `code`, `reading`, `run`, `misc`, `break`
Lunch and Dinner blocks use `category: "misc"` (not `"break"`) so they are checkable and editable.
Pure break blocks (Off/Flex, Friday Social, etc.) use `category: "break"` and are not interactive.

### localStorage Keys Summary
| Key | Contents |
|-----|----------|
| `schedule-v1-{weekKey}` | Check states for current week |
| `schedule-overrides-v1` | Category/activity overrides (date-specific) |
| `schedule-history-v1` | Archived weekly summaries |
| `schedule-swaps-v1` | DnD swap permutations (date+day-specific) |
| `schedule-custom-v1` | User-added blocks (date+day-specific) |
| `schedule-timemut-v1` | Time edits and overlap trims (date+day+slot-specific) |

### Checks
```js
// Key: "schedule-v1-2026-W12"
{ "Monday-0": true, "Tuesday-cb_1710000000000": true }
// Hardcoded: "{dayName}-{slotIndex}"
// Custom:    "{dayName}-cb_{id}"
```

### Overrides
```js
// Key: "schedule-overrides-v1"
{
  "2026-03-18-0": { category: "jobs", custom: "" },
  "2026-03-18-cb_1710000000000": { category: "misc", custom: "Dentist" }
}
```

### Swaps
```js
// Key: "schedule-swaps-v1"
{ "2026-03-18-Monday": [0, 2, 1, 3, 4, 5, 6, 7, 8] }
// Identity permutation = no swap. Entry only created when a swap occurs.
```

### Custom Blocks
```js
// Key: "schedule-custom-v1"
{
  "2026-03-18-Monday": [
    { id: "1710000000000", time: "2:00 – 2:30pm", startMinutes: 840, endMinutes: 870,
      activity: "Dentist", detail: "", category: "misc" }
  ]
}
```

### Time Mutations
```js
// Key: "schedule-timemut-v1"
{
  "2026-03-18-Monday-2": { newStart: 870, newEnd: 1110 },           // time edit or trim
  "2026-03-18-Monday-3": { hidden: true, causedBy: "1710000000000" } // covered by custom block
}
```

---

## Rendering Pipeline

The render loop builds a merged block list for each day via `getDisplayBlocks(dayName)`:
1. Take hardcoded blocks, apply `timeMutations` (skip hidden, use mutated time if present)
2. Append custom blocks
3. Sort chronologically by start time
4. For each block in the merged list:
   - Apply DnD swap (`getPermutedContent`) to get the content source (hardcoded only)
   - Apply override (`getEffectiveBlock`) on top of permuted content
   - Determine `isBreak`, `isNow`, `isChecked`, drag state

---

## Time Parsing

The schedule uses time strings like `"11:00am – 1:30pm"` and `"2:30 – 4:30pm"`. Parser rules:

- If meridiem (am/pm) is explicit: use it
- If no meridiem: hour `11` = 11am; all other hours = pm (schedule runs 11am–11pm)
- Duration in hours = `(end - start) / 60`
- `minutesToTimeString(start, end)` converts back to display format
- `fmtTime(mins)` formats a single minute value as `"2:30pm"` (used to pre-fill edit inputs)

---

## Design Tokens

| Token | Value |
|-------|-------|
| Background | `#09090b` (zinc-950) |
| Card background | `#18181b` (zinc-900) |
| LSAT | `#fbbf24` (amber-400) |
| Job Search | `#38bdf8` (sky-400) |
| Substack | `#a78bfa` (violet-400) |
| Claude Code | `#34d399` (emerald-400) |
| Reading | `#fb7185` (rose-400) |
| Run | `#fb923c` (orange-400) |
| Misc | `#f9a8d4` (pink-300) — light pink |
| Font | Georgia (body), monospace (times/data) |
