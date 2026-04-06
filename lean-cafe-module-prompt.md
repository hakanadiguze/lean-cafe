# Lean Café Retrospective — Module Specification

## Context

You are building a **Lean Café Retrospective** module as part of a larger **Agile Platform** that groups multiple agile tools under a single product. This module is one tool within that platform. Build it as a self-contained component that can later be embedded into the platform shell.

The platform's design language is already defined in `platform/design-tokens.css` and `platform/components/`. **If those files do not exist yet, define your own design system inline and document it so the platform can adopt it.** The aesthetic direction is: warm, editorial, paper-like — not a typical SaaS dashboard.

---

## What is Lean Café?

Lean Café is a structured, democratic retrospective technique for large groups (10–100+ people). It replaces the traditional facilitator-led retro with a self-organizing, vote-driven flow:

1. Everyone writes topics they want to discuss (free format, one per card)
2. Everyone votes on the most important topics with a fixed number of votes
3. Top-voted topics become discussion rooms
4. Participants self-select into rooms and discuss in parallel (e.g. in Teams breakout rooms)
5. Each room captures a root cause and action items
6. A PDF report is generated at the end

---

## Technical Architecture

### Stack
- **Pure HTML + CSS + Vanilla JS** — no framework, no build step, no npm
- **Single file**: `lean-cafe/index.html`
- **Storage**: `localStorage` with key `lc5_sessions` — all session data lives there
- **Real-time sync**: `BroadcastChannel('lean_cafe_v5')` for same-browser multi-tab sync + `setInterval` polling at 1200ms for cross-browser/device simulation
- **PDF export**: `jsPDF 2.5.1` from `https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js`
- **Fonts**: Google Fonts — `Fraunces` (serif, display), `DM Sans` (UI body), `DM Mono` (monospace/numbers)
- **Analytics**: Google Analytics 4 via gtag, tag ID `G-XD6NXRB6KB`
- **Favicon**: Inline SVG emoji `☕`

### Session Data Shape

```js
{
  id: "RETRO-A1B2",          // generated: 'RETRO-' + 4 random uppercase alphanum
  title: "Q2 Sprint 8 Retro",
  team: "Platform Team",
  votesPerPerson: 5,
  adminName: "Hakan",
  adminUid: "uid_string",
  phase: 1,                   // 1..5
  cards: [
    {
      id: "unique_id",
      text: "Our standups take too long",
      author: "Hakan",        // display name
      uid: "uid_string",      // unique per browser session
      votes: 3,
      voters: ["uid1","uid2","uid3"]  // uids who voted
    }
  ],
  participants: [
    { name: "Hakan", uid: "uid_string", isAdmin: true, votesUsed: 2 }
  ],
  rooms: [
    {
      id: "room0",
      cardId: "card_id",
      title: "Our standups take too long",
      votes: 7,
      participants: ["uid1","uid2"]   // uids of room members
    }
  ],
  roomData: {
    "room0": {
      rootCause: "We don't have a clear agenda...",
      actions: [
        { text: "Create standup template", assignee: "John", dueDate: "2025-05-01" }
      ]
    }
  },
  timers: {
    1: { running: true, end: 1714000000000, total: 600 },  // per-phase timers
    2: { running: false, end: 0, total: 0 }
  },
  createdAt: "2025-04-06T10:00:00.000Z"
}
```

### Identity Model
- **Every join creates a new `uid`** (`Date.now().toString(36) + random`). This means two people with the same name are tracked independently — they have separate vote counts, separate room memberships, separate card ownership.
- Votes, room membership, and card deletion are all keyed on `uid`, never on `name`.
- The admin's `uid` is stored in `adminUid` on the session.

### Sync Strategy
- **Phase changes only trigger a full re-render.** Tracking the last-seen phase in `_lastPhase` variable.
- **Within a phase, never re-render the full DOM** — this would clear text inputs mid-typing. Only the timer display updates between renders.
- Timer display updates via `setInterval` at 300ms — updating only `textContent` and CSS classes on existing elements.
- `BroadcastChannel` fires on every `persist()` call. Receivers check phase change only.

---

## Screens / Views

The app has these top-level screens (CSS class `.scr`, active = `.scr.on`):

| ID | Description |
|----|-------------|
| `s-home` | Landing page — facilitator entry point |
| `s-setup` | Create session form |
| `s-join` | Join session form (participants) |
| `s-link` | Link share screen (after session creation) |
| `s-session` | Main session view — phases 1–5 rendered here |

Screen switching: `document.querySelectorAll('.scr').forEach(s => s.classList.remove('on'))` then `document.getElementById(id).classList.add('on')`.

**Important:** Never use `onclick` attributes on HTML elements. All event listeners must be bound via `addEventListener` in `DOMContentLoaded` or immediately after `setArea()` injects HTML.

---

## Home Screen (`s-home`)

- Single CTA: **"☕ Start a Session as Facilitator"** button — leads to `s-setup`
- No participant join button (participants arrive via URL with `?session=CODE`)
- **"How it works"** section: 5 numbered steps with icon, title, and description explaining the full flow
- Footer: `Built by Hakan Adıgüzel · hakanadiguzel.com`

---

## Session Phases

### Phase 1 — Write Cards

**Instruction box (blue):** Explains what to do step by step. Shown at top of every phase — users coming from Teams breakout rooms may have no context.

**Admin controls:**
- Banner: "Facilitator action required: when everyone has written their cards, click Open Voting"
- Button: "Open Voting →" (`.btn-rust`) — requires ≥2 cards, sets `phase = 2`

**Participant view:**
- "Waiting for facilitator to open voting" message

**Card input:**
- `<textarea>` for new card text — `Enter` (without Shift) triggers add
- "Add Card ↵" button
- **Do NOT re-render on poll** — only re-render when user submits a card or phase changes

**My Cards panel:**
- Shows only cards where `card.uid === me.uid`
- Delete button — only own cards deletable (admin can delete any)

**All cards grid:**
- `display: grid; grid-template-columns: repeat(auto-fill, minmax(210px, 1fr))`
- Sticky note style: left border 5px colored, slight hover lift + rotate
- 6 border color classes cycling: `--rust`, `--sage`, `--gold`, `--sky`, `--lav`, warm brown
- Shows author name in monospace at bottom

**Timer widget** (see Timer section below) — default duration: 10 min

---

### Phase 2 — Vote

**Instruction box:** Explains voting rules.

**Admin controls:** "Close Voting & Create Rooms →" — opens a `prompt()` asking how many rooms. Creates `session.rooms` from top-N voted cards, sets `phase = 3`.

**Vote status bar:**
- Large monospace display: remaining votes / total (e.g. "3 / 5")
- Color: `--rust` when votes remain, `--ink4` when used up
- Row of token circles: filled = used, empty = available

**Card grid:**
- Same sticky note layout as Phase 1
- Each card has: vote count (large monospace) + vote button
- Vote button states: default → "＋ Vote", after voting → "✓ Voted" (green style)
- Vote button disabled when no votes left AND not already voted
- Clicking "✓ Voted" removes the vote (toggle)
- Voting is uid-based: `card.voters` array contains uids

**Timer widget** — default duration: 8 min

---

### Phase 3 — Choose Room

**Teams integration box (purple/lavender):**
```
💬 Microsoft Teams — Breakout Rooms
Join the Teams breakout room with the same number as the room you pick here.
Example: pick Room 2 here → join Breakout Room 2 in Teams.
```

**Instruction box:** Step-by-step guide including Teams instruction.

**Admin controls:** "Start Discussion →" — sets `phase = 4`.

**Room cards grid:**
- Each card shows: room number, topic title, vote count badge, Teams room reference
- Participant list (resolved from uid to display name)
- Click joins the room: removes uid from all other rooms, adds to this room
- Joined room shows `mine` class (thicker rust border)
- Shows "📍 You are here" indicator
- Toast message on join: "Joined Room 2 → go to Teams Breakout Room 2"

**Timer widget** — default duration: 5 min

---

### Phase 4 — Discuss

**Teams box:** "You should be in your Teams Breakout Room right now"

**Instruction box:** Emphasizes Scribe role and action item quality.

**Admin controls:** "Finish Session & Generate Report →" — sets `phase = 5`.

**Room cards (clickable, opens modal):**
- Show: room number + votes, topic title, participant tags, root cause preview (truncated 120 chars), action item previews
- Status badges: "✓ Root cause" (sage) or "⚠ No root cause" (gray), action count (rust) or "No actions" (gray)
- Action preview shows: assignee tag in gold, due date in gray, "⚠ no assignee" / "no date" when missing
- Bottom prompt: "👆 Scribe: click here to enter root cause & actions"

**Timer widget** — default duration: 20 min

---

### Phase 5 — Report

**Stats row (5 cards):** Participants, Cards, Votes, Rooms, Actions — each with large serif number and color.

**All cards list:** Sorted by votes descending. Highlighted rows for cards that became rooms (rust left border + slightly different background).

**Room sections:** For each room:
- Room header with number and vote count
- Participant name tags
- Root cause in sage-green box (or warning if missing)
- Action items in rust-light rows: number, description, `@assignee` gold tag, date gray tag

**"⬇ Download PDF Report" button** — triggers `downloadPDF()`

---

## Room Modal

Opens on click of any room card in Phase 4.

**Header:** Room number + votes, topic title, close button (✕)

**Scribe reminder box (rust-light):**
```
📝 Scribe reminder: Before filling this in, agree with your group on who the Scribe is.
The Scribe types on behalf of the whole group while everyone else discusses in Teams.
```
No dropdown for scribe selection — just this text reminder.

**Root cause textarea:**
- Label: "Root Cause — Why does this problem exist?"
- Helper text: "Try asking 'Why?' 5 times to find the real cause."

**Action Items section:**
- "+ Add Action" button
- Each action item card contains:
  - "What needs to be done?" — text input
  - "Assignee — who is responsible?" — text input, **maxlength="20"**, helper: "⚠ Confirm with the person before assigning"
  - "Due Date" — `type="date"` input, helper: "Agree the date with the assignee"
  - Warning text below if assignee or due date missing: "⚠ No assignee set · No due date set" in gold color
  - Action card border turns amber/gold when incomplete
  - "Remove" button

**Save behavior:**
- On "Save & Close": check if any actions have text but missing assignee or due date
- If yes: show inline warning box with "⚠ N actions missing assignee or due date. This is fine — but follow-through is much harder without them."
- Offer two buttons: "Save anyway" (rust) and "Go back and fix" (ghost)
- If no warnings: save immediately and close

**Data persistence:** All field changes (`input` + `change` events) immediately call `persist(s)` — no draft state.

---

## Timer System

Each phase has its own **independent** timer stored in `session.timers[phaseNum]`.

### Timer data shape
```js
{ running: true, end: 1714000000000, total: 600 }
// end = epoch ms when timer hits zero
// total = original duration in seconds (for percentage calculation)
```

### Timer widget HTML structure
```html
<div class="timer-wrap" id="timer-wrap-{phase}">
  <div>
    <div class="timer-display" id="timer-disp-{phase}">--:--</div>
    <div class="timer-label" id="timer-lbl-{phase}">Timer not started</div>
  </div>
  <!-- admin only: -->
  <div class="timer-controls">
    <input type="number" id="timer-mins-{phase}" value="{default}" min="1" max="90">
    <span>min</span>
    <button id="btn-timer-start-{phase}">▶ Start</button>
    <button id="btn-timer-stop-{phase}">■ Stop</button>
  </div>
  <!-- participant only: -->
  <div>Set and started by facilitator</div>
</div>
```

### Default durations by phase
| Phase | Default |
|-------|---------|
| 1 — Write Cards | 10 min |
| 2 — Vote | 8 min |
| 3 — Choose Room | 5 min |
| 4 — Discuss | 20 min |

### Timer visual states

| Time remaining | `timer-display` class | `timer-wrap` class | Font | Color |
|---|---|---|---|---|
| > 50% | (none) | (none) | 36px regular | `--ink` |
| 25%–50% | `warn` | `warn` | 42px medium | `--gold` |
| < 25% | `danger` | `danger` | 50px bold | `--rust` |
| 0 (expired) | `danger` | `danger` | 50px bold | `--rust` + pulse animation |

Pulse animation: `@keyframes tpulse { 0%,100% { opacity:1 } 50% { opacity:.55 } }` — 1s loop.

When timer wraps change state, update `background` and `border-color` of the wrap container (gold-light / rust-light backgrounds).

### Timer tick behavior
- Timer display updates every 300ms via `setInterval`
- Timer intervals are stored per-phase: `_timerIntervals[phase] = intervalId`
- On phase change: clear all timer intervals except current phase
- On polling cycle (1200ms): call `_updateTimerDisplay(phase, freshSession)` — this only touches timer element `textContent` and `className`, never the full DOM

---

## Design System

### Color Tokens
```css
--cream: #f5f0e8      /* page background */
--cream2: #ece5d8     /* subtle surface, inputs bg hover */
--cream3: #e0d8c8     /* stronger border alt */
--ink: #1a1714        /* primary text, dark buttons */
--ink2: #3d3830       /* secondary text */
--ink3: #6b6358       /* muted text, labels */
--ink4: #9c9088       /* very muted, placeholders */
--rust: #c4522a       /* primary action, admin buttons, accent */
--rust-l: #f0ddd5     /* rust light background */
--sage: #4a7c59       /* success, done states, vote confirmed */
--sage-l: #d8eadf
--gold: #c49a3c       /* warnings, incomplete states */
--gold-l: #f5ead0
--sky: #3a6b8a        /* info boxes, participant tags */
--sky-l: #d0e4f0
--lav: #6b5b8a        /* Teams integration highlights */
--lav-l: #e4dcf0
--bdr: rgba(26,23,20,.12)   /* default border */
--bdr2: rgba(26,23,20,.22)  /* stronger border, inputs */
--sh: 0 2px 12px rgba(26,23,20,.08)
--sh-lg: 0 8px 32px rgba(26,23,20,.14)
--r: 10px
--r-lg: 16px
```

### Typography
- **Headings** (`h1`, `h2`, `h3`): Fraunces serif, weights 700/600
- **Body**: DM Sans, 15px, line-height 1.6
- **Monospace** (codes, numbers, timer): DM Mono
- **Labels** (`.lbl`): 11px, uppercase, letter-spacing 1px, `--ink3`

### Component Patterns

**Buttons:**
- `.btn-ink` — dark fill (primary, neutral actions)
- `.btn-rust` — rust fill (facilitator phase-advance actions)
- `.btn-sage` — sage fill (timer start)
- `.btn-ghost` — transparent with border
- `.btn-out` — white with border (secondary)
- `.btn-sm` — smaller padding
- `.btn-lg` — larger padding, 16px font

**Info box (blue):** `.infobox` — sky-light background, bulleted step list with `→` prefix, shown at top of each phase to explain what to do.

**Admin banner (rust):** Rust-light background strip above admin action button, text: "🎯 Facilitator action required: [instruction]"

**Participant waiting message:** Cream2 background, "⏳ Waiting for facilitator to [action]"

**Teams box (lavender):** `.teams-box` — shown in phases 3 and 4, explains Teams breakout room connection.

**Sticky note cards:**
```css
.sticky {
  padding: 14px;
  border-radius: 4px 12px 12px 4px;  /* flat left, rounded right */
  border-left: 5px solid [color];
  background: #fff;
  min-height: 90px;
  display: flex;
  flex-direction: column;
  gap: 7px;
  word-break: break-word;
}
.sticky:hover { transform: translateY(-3px); box-shadow: var(--sh-lg); }
```
6 colors cycling: rust, sage, gold, sky, lav, warm brown `#b0785a`

**Tags:** `.tag` — small pill badges, 6 color variants (t-rust, t-sage, t-gold, t-sky, t-gray)

**Toast:** Fixed bottom-right, dark background, 2800ms auto-dismiss, slide-up animation

**Progress steps bar:** Horizontal step indicator at top of session view. States: done (sage circle), active (dark circle), upcoming (empty circle). Connected by horizontal lines.

**Step circles:**
```css
.step-c { width: 26px; height: 26px; border-radius: 50%; }
.step.on .step-c { background: var(--ink); color: var(--cream); }
.step.done .step-c { background: var(--sage); color: #fff; }
```

---

## PDF Report Structure

Use `jsPDF` in portrait A4 format (210×297mm), 10mm margin, `unit: 'mm'`.

### Page 1 — Cover
- Black full-page background (`#1a1714`)
- Centered text in cream: "LEAN CAFÉ RETROSPECTIVE REPORT" (9px, uppercase)
- Session title (20px bold, wrapped to 160mm max width)
- Team name + date (11px, `#9c9088`)
- 5 stat boxes in a row (each 33mm wide, dark bg `#28251f`): Participants, Cards, Votes, Rooms, Actions
- Footer: "hakanadiguzel.com · Lean Café Retrospective" (8px, very muted)

### Page 2 — All Cards
- Title "All Cards" (15px bold)
- Rounded rect rows (11mm tall) for each card, sorted by votes descending
- Top-N cards (those that became rooms) have rust left accent bar (2.5mm wide)
- Top cards use bold font; others normal
- Vote count right-aligned

### Pages 3+ — One page per Room
- Rust horizontal rule + "ROOM N · X VOTES" label
- Room title (13px bold, wrapped)
- Participants list
- Root cause in sage-green rounded box with left accent bar and "ROOT CAUSE" label
- Action items in rust-light rounded boxes: number, description, `@assignee` (gold bold right-aligned), due date (gray bottom-right)

---

## Validation & Edge Cases

| Scenario | Handling |
|---|---|
| Same name, multiple users | Each join generates new uid — fully independent tracking |
| 0 cards when opening voting | `toast('Need at least 2 cards')` — block phase advance |
| Voting with 0 votes left | Vote button disabled; toast "No votes left" if somehow triggered |
| Unvoting | Click "✓ Voted" again — removes uid from `card.voters`, decrements both `card.votes` and `participant.votesUsed` |
| Room with 0 participants in Phase 4 | Still shows card, still clickable for Scribe |
| Action with no text | Skipped in PDF rendering (`if(!a.text) return`) |
| Missing assignee/due date | Soft warning on save — not blocked, but shown in report with "⚠ no assignee" / "no date" labels |
| Timer already running on page load | Detected by checking `timer.running && timer.end && Date.now() < timer.end` — restart tick interval |
| Timer expired during reload | `Math.max(0, ...)` on remaining seconds — shows 00:00 in danger state |
| Phase advance clears timer | Timer per-phase — advancing phase doesn't affect other phases' timers |

---

## URL Handling

- Participant join link: `https://[domain]/lean-cafe/?session=RETRO-CODE`
- On load: `new URLSearchParams(location.search).get('session')`
- If session code found in URL and session exists in localStorage: auto-navigate to `s-join` screen with code pre-filled

---

## File Structure

```
lean-cafe/
├── index.html          ← entire app (single file)
├── package.json        ← { "scripts": { "start": "npx serve public" } }
├── vercel.json         ← static routes pointing to public/
├── .gitignore
└── README.md
```

`vercel.json`:
```json
{
  "version": 2,
  "builds": [{ "src": "public/**", "use": "@vercel/static" }],
  "routes": [{ "src": "/(.*)", "dest": "/public/$1" }]
}
```

Wait — if integrating into the Agile Platform, the file lives at:
```
platform/
├── tools/
│   └── lean-cafe/
│       └── index.html   ← this file
├── shared/
│   ├── design-tokens.css
│   └── components/
└── index.html           ← platform shell
```

Adjust routing and asset paths accordingly.

---

## Implementation Notes for Claude Code

1. **Never use `onclick="..."` inline attributes.** All events must be bound with `addEventListener` after HTML is injected via `setArea()`. This is the most common source of "function not defined" errors in sandboxed environments.

2. **`setArea(html)` replaces the entire `#phase-area` innerHTML.** Always bind events immediately after calling it. Do not rely on previously bound events surviving a re-render.

3. **The poll interval must NOT trigger a full re-render during Phase 1 while user is typing.** Only re-render when `s.phase !== _lastPhase`. Otherwise, update only timer display elements by ID.

4. **Timer display elements are identified by phase number** (e.g. `timer-disp-1`, `timer-wrap-1`). After a re-render, the old timer interval keeps running and will find the new elements by ID — no need to restart the interval on re-render.

5. **`persist(s)` always updates both `localStorage` AND the module-level `sess` variable AND fires BroadcastChannel.** Never write to localStorage directly.

6. **`fresh()`** always re-reads from localStorage. Use it at the start of any event handler to get the latest state — never close over a stale `s` variable.

7. **PDF generation:** Call `Math.max(0, ...)` on all calculated positions to prevent negative Y coordinates. Always call `cy(n)` (check-Y) before placing any element that needs `n` mm of vertical space.

8. **Name field maxlength:** Facilitator and participant names are limited to **15 characters** with a live character counter. Assignee field in action items is limited to **20 characters**.

9. **Rooms store participant UIDs, not names.** Always resolve to display names via `s.participants.find(p => p.uid === uid)?.name` when rendering.

10. **Session ID format:** `'RETRO-' + Math.random().toString(36).substr(2,4).toUpperCase()` — always 10 chars total.

---

## Acceptance Criteria

The implementation is complete when:

- [ ] Facilitator can create a session and get a shareable URL
- [ ] Participants can join via URL or manual code entry — same name works for multiple independent participants
- [ ] Phase 1: Cards can be added, deleted, seen by all; adding a card does NOT clear other users' in-progress text
- [ ] Phase 2: Votes are uid-tracked; unvoting works; vote button disables correctly
- [ ] Phase 3: Room join works; Teams room number is shown clearly; same uid can switch rooms
- [ ] Phase 4: Room modal opens; root cause saves; actions with assignee (max 20 chars) + due date save; soft warning on missing fields works
- [ ] Phase 5: PDF downloads with cover page, all cards page, one page per room with root cause and action items
- [ ] Timer: per-phase; starts/stops independently; visual states (normal → gold warn → red danger → red pulse) work correctly; participants see timer without controls
- [ ] BroadcastChannel sync: phase changes propagate across tabs without clearing input fields
- [ ] "Built by Hakan Adıgüzel" with link to hakanadiguzel.com appears in footer
- [ ] Google Analytics tag `G-XD6NXRB6KB` is included
- [ ] No `onclick` attributes anywhere in the HTML — all events via `addEventListener`
