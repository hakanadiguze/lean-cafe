# Lean Café Retrospective — Firebase Edition Module Specification

## Context

You are building a **Lean Café Retrospective** module as part of a larger **Agile Platform**. This module is one tool within that platform. Build it as a self-contained component that can be embedded into the platform shell later.

---

## What is Lean Café?

Lean Café is a structured, democratic retrospective technique for large groups (10–100+ people):

1. Everyone writes topics they want to discuss (one per card)
2. Everyone votes on the most important topics with a fixed number of votes
3. Top-voted topics become discussion rooms
4. Participants self-select into rooms and discuss in parallel (in MS Teams breakout rooms)
5. Each room captures a root cause and action items (text, assignee, due date)
6. A PDF report is generated at the end

---

## Tech Stack

- **Pure HTML + CSS + Vanilla JS** — no framework, no build step
- **Single file**: `public/index.html`
- **Firebase Realtime Database** — live sync via `onValue()`
- **Firebase Authentication** — Google Sign-In for facilitators only; participants are anonymous
- **Firebase SDK**: v10.12.0 via ES module imports from `gstatic.com`
- **PDF**: `jsPDF 2.5.1` from cdnjs (classic `<script>` tag, not module)
- **Fonts**: Google Fonts — Fraunces (serif), DM Sans (body), DM Mono (mono)
- **Analytics**: Google Analytics 4, tag `G-XD6NXRB6KB`
- **Favicon**: Inline SVG `☕`

---

## Firebase Configuration

```js
// In public/index.html — replace all REPLACE_WITH_YOUR_* values
const FIREBASE_CONFIG = {
  apiKey:            "REPLACE_WITH_YOUR_API_KEY",
  authDomain:        "REPLACE_WITH_YOUR_AUTH_DOMAIN",
  databaseURL:       "REPLACE_WITH_YOUR_DATABASE_URL",
  projectId:         "REPLACE_WITH_YOUR_PROJECT_ID",
  storageBucket:     "REPLACE_WITH_YOUR_STORAGE_BUCKET",
  messagingSenderId: "REPLACE_WITH_YOUR_MESSAGING_SENDER_ID",
  appId:             "REPLACE_WITH_YOUR_APP_ID"
};
```

Firebase imports (ES module, inside `<script type="module">`):
```js
import { initializeApp } from 'https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js';
import { getAuth, signInWithPopup, signOut, GoogleAuthProvider, onAuthStateChanged }
  from 'https://www.gstatic.com/firebasejs/10.12.0/firebase-auth.js';
import { getDatabase, ref, set, get, update, onValue, off }
  from 'https://www.gstatic.com/firebasejs/10.12.0/firebase-database.js';
```

---

## Database Structure

```
lean-cafe/
└── sessions/
    └── {SESSION_ID}/              e.g. RETRO-A1B2
        ├── id: "RETRO-A1B2"
        ├── title: "Q2 Sprint 8 Retrospective"
        ├── team: "Platform Team"
        ├── votesPerPerson: 5
        ├── adminName: "Hakan"
        ├── adminUid: "firebase_auth_uid"
        ├── adminEmail: "hakan@..."
        ├── phase: 1                        // 1..5
        ├── createdAt: "2025-04-06T10:00Z"
        ├── cards/
        │   └── {cardId}/
        │       ├── id, text, author, uid
        │       ├── votes: 3
        │       └── voters/
        │           └── {uid}: true         // uid-keyed boolean map
        ├── participants/
        │   └── {uid}/
        │       ├── name, uid
        │       ├── isAdmin: false
        │       └── votesUsed: 2
        ├── rooms/
        │   └── room0/
        │       ├── id: "room0"
        │       ├── cardId, title
        │       ├── votes: 7
        │       └── participants/
        │           └── {uid}: true
        ├── roomData/
        │   └── room0/
        │       ├── rootCause: "..."
        │       └── actions/
        │           └── act0/
        │               ├── id, text
        │               ├── assignee: "John"
        │               └── dueDate: "2025-05-01"
        └── timers/
            └── {phase}/                    // e.g. "1", "2", "3", "4"
                ├── running: true
                ├── end: 1714000000000      // epoch ms
                └── total: 600             // original duration in seconds
```

**Key rule:** Firebase Realtime Database does not support arrays. All collections are **keyed objects**. Always convert with `Object.values(obj || {})` before iterating.

---

## Identity & Auth Model

### Facilitator
- Must sign in with **Google** (`signInWithPopup`) before creating a session
- Firebase Auth UID (`auth.currentUser.uid`) is used as their participant UID
- `adminUid` is stored on the session for security rules
- Auth state badge shown in topbar: avatar + name + sign-out button

### Participants
- **No auth required** — zero friction join
- On join: generate a random UID (`Date.now().toString(36) + randomStr`)
- Same display name + different browser = different UID = fully independent tracking
- Votes, room membership, and card ownership are all keyed on UID, never on name

---

## Firebase Helpers Pattern

```js
const sessPath = id => `lean-cafe/sessions/${id}`;

// Write full session (create only)
await set(ref(db, sessPath(id)), sessionObject);

// Partial update
await update(ref(db, sessPath(id)), { phase: 2 });

// Deep partial update (nested keys using slash notation)
await update(ref(db, sessPath(id)), {
  [`cards/${cardId}`]: cardObject,
  [`participants/${uid}/votesUsed`]: newCount
});

// Delete a node
await update(ref(db, sessPath(id)), { [`cards/${cardId}`]: null });

// Read once
const snap = await get(ref(db, sessPath(id)));
const s = snap.exists() ? snap.val() : null;

// Subscribe live
const sessionRef = ref(db, sessPath(id));
onValue(sessionRef, snap => {
  if (!snap.exists()) return;
  sess = snap.val();
  handleUpdate(sess);
});

// Unsubscribe
off(sessionRef);
```

---

## Sync Strategy — Critical Rules

1. **`onValue()` fires on every write anywhere in the session** — cards, votes, timer, phase change.

2. **Only re-render the full DOM on phase change.** Track `_lastPhase`. If `s.phase === _lastPhase`, do NOT call `renderPhase()` — this would clear text inputs the user is typing.

3. **Timer updates only touch timer element `textContent` and `className`** — never the full DOM. Call `_tickTimer(phase, s)` directly from the `onValue` callback.

4. **Timer countdown uses `requestAnimationFrame`** for smooth display between Firebase updates. Cancel all rAF IDs on phase change.

5. **After a user action that changes their own view** (e.g. adding a card), call `renderPhase()` once manually — the `onValue` callback won't trigger a re-render since the phase didn't change.

6. **All event listeners are bound via `addEventListener`** after `setArea()` injects HTML. Never use `onclick` attributes.

---

## Screens

| ID | Description |
|----|-------------|
| `s-home` | Landing — facilitator CTA + How it works + Built by |
| `s-setup` | Create session form (requires Google auth) |
| `s-join` | Join session form (no auth) |
| `s-link` | Link share (after creation) |
| `s-session` | Main session — phases 1–5 rendered here in `#phase-area` |

Screen switch: remove `.on` from all `.scr`, add `.on` to target.

---

## Authentication Flow

```js
// On home screen button click:
document.getElementById('go-setup').addEventListener('click', () => {
  if (!currentUser) { signInGoogle(); return; }
  show('s-setup');
});

async function signInGoogle() {
  await signInWithPopup(auth, new GoogleAuthProvider());
  // onAuthStateChanged fires → updates currentUser → show setup
}

onAuthStateChanged(auth, user => {
  currentUser = user;
  // Update auth badge in topbar
  if (user) {
    badge.innerHTML = `<img src="${user.photoURL}"> ${user.displayName} <button>Sign out</button>`;
  }
});
```

---

## Phase 1 — Write Cards

**Admin action:** Sets `phase: 2` via `update(ref(db, sessPath(id)), { phase: 2 })`

**Add card:**
```js
const cardId = genUid();
await update(ref(db, `${sessPath(sess.id)}/cards/${cardId}`), {
  id: cardId, text, author: me.name, uid: me.uid, votes: 0, voters: {}
});
```

**Delete card:** `update(ref(...), { [`cards/${cardId}`]: null })`

**Important:** After adding a card, call `renderPhase()` once — the `onValue` callback won't re-render since phase didn't change.

---

## Phase 2 — Vote

**Vote (add):**
```js
await update(ref(db, `${sessPath(sess.id)}/cards/${cid}`), {
  votes: (card.votes || 0) + 1,
  [`voters/${me.uid}`]: true
});
await update(ref(db, `${sessPath(sess.id)}/participants/${me.uid}`), {
  votesUsed: (part.votesUsed || 0) + 1
});
```

**Unvote (remove):**
```js
await update(ref(db, `${sessPath(sess.id)}/cards/${cid}`), {
  votes: Math.max(0, (card.votes || 0) - 1),
  [`voters/${me.uid}`]: null   // null deletes the key in RTDB
});
```

**Check if voted:** `!!card.voters?.[me.uid]` (boolean map)

**Create rooms (admin):**
```js
const rooms = {};
topCards.slice(0, n).forEach((c, i) => {
  rooms[`room${i}`] = { id: `room${i}`, cardId: c.id, title: c.text, votes: c.votes||0, participants: {} };
});
await update(ref(db, sessPath(sess.id)), { rooms, phase: 3 });
```

---

## Phase 3 — Choose Room

**Join a room (remove from all, add to one):**
```js
const patch = {};
objToArr(sess.rooms).forEach(r => {
  patch[`rooms/${r.id}/participants/${me.uid}`] = null;
});
patch[`rooms/${rid}/participants/${me.uid}`] = true;
await update(ref(db, sessPath(sess.id)), patch);
```

**Teams integration box:** Show purple box explaining that the Teams breakout room number matches the room number here.

---

## Phase 4 — Discuss (Room Modal)

**Save room data:**
```js
const acts = objToArr(sess.roomData?.[rid]?.actions || {});
const newActions = {};
acts.forEach((a, i) => { newActions[`act${i}`] = a; });
await update(ref(db, `${sessPath(sess.id)}/roomData/${rid}`), { rootCause, actions: newActions });
```

**Action item fields:** `text` (required to show), `assignee` (max 20 chars, soft-required), `dueDate` (date input, soft-required)

**Soft warning on save:** If any action has `text` but missing `assignee` or `dueDate`, show inline warning with "Save anyway" and "Go back and fix" buttons. Do not block save.

**Scribe reminder:** Static text box — no dropdown. "Agree with your group on who the Scribe is before filling this in."

---

## Timer System (per phase, stored in Firebase)

**Start:**
```js
await update(ref(db, `${sessPath(sess.id)}/timers/${phase}`), {
  running: true,
  end: Date.now() + minutes * 60 * 1000,
  total: minutes * 60
});
```

**Stop:**
```js
await update(ref(db, `${sessPath(sess.id)}/timers/${phase}`), {
  running: false, end: 0, total: 0
});
```

**Display update** (called from `onValue` callback and `rAF` loop):
```js
function _tickTimer(phase, s) {
  const el   = document.getElementById(`timer-disp-${phase}`);
  const wrap = document.getElementById(`timer-wrap-${phase}`);
  if (!el) return;
  const t   = s?.timers?.[phase];
  if (!t?.running || !t.end) { /* show --:-- */ return; }
  const rem = Math.max(0, Math.ceil((t.end - Date.now()) / 1000));
  el.textContent = `${String(Math.floor(rem/60)).padStart(2,'0')}:${String(rem%60).padStart(2,'0')}`;
  const pct = t.total > 0 ? rem / t.total : 1;
  // Apply classes: danger (≤25%), warn (≤50%), normal (>50%)
  // Schedule next frame if rem > 0
  if (rem > 0) _timerRafs[phase] = requestAnimationFrame(() => _tickTimer(phase, sess));
}
```

**Default durations:** Phase 1 → 10min, Phase 2 → 8min, Phase 3 → 5min, Phase 4 → 20min

**Timer visual states:**

| Remaining | Classes | Font size | Color |
|-----------|---------|-----------|-------|
| > 50% | — | 36px regular | `--ink` |
| 25–50% | `warn` on wrap + display | 42px medium | `--gold` |
| ≤ 25% | `danger` on wrap + display | 50px bold + pulse | `--rust` |

Pulse: `@keyframes tpulse { 0%,100%{opacity:1} 50%{opacity:.55} }` 1s loop

---

## Design System

### Colors
```css
--cream:#f5f0e8  --cream2:#ece5d8  --ink:#1a1714  --ink2:#3d3830
--ink3:#6b6358   --ink4:#9c9088   --rust:#c4522a  --rust-l:#f0ddd5
--sage:#4a7c59   --sage-l:#d8eadf  --gold:#c49a3c  --gold-l:#f5ead0
--sky:#3a6b8a    --sky-l:#d0e4f0   --lav:#6b5b8a   --lav-l:#e4dcf0
--bdr:rgba(26,23,20,.12)  --bdr2:rgba(26,23,20,.22)
--sh:0 2px 12px rgba(26,23,20,.08)  --sh-lg:0 8px 32px rgba(26,23,20,.14)
--r:10px  --r-lg:16px
```

### Fonts
- `--serif: 'Fraunces', Georgia, serif` — headings, logo, stat numbers
- `--sans: 'DM Sans', system-ui, sans-serif` — body, buttons
- `--mono: 'DM Mono', 'Courier New', monospace` — codes, timer, labels

### Button variants
`.btn-ink` (dark), `.btn-rust` (phase-advance, admin), `.btn-sage` (timer start), `.btn-ghost` (secondary), `.btn-out` (white+border), `.btn-sm`, `.btn-lg`

### Sticky note cards (phase 1 & 2)
```css
.sticky {
  border-radius: 4px 12px 12px 4px;  /* flat left, rounded right */
  border-left: 5px solid {color};
  min-height: 90px;
  word-break: break-word;
}
.sticky:hover { transform: translateY(-3px); }
```
6 colors: rust, sage, gold, sky, lav, `#b0785a` (cycling by index `i % 6`)

### Info box (blue) — shown at top of every phase
```html
<div class="infobox">
  <div class="infobox-title">✍️ Your turn: Write your cards</div>
  <ul class="infobox-steps">
    <li>Step one</li>  <!-- rendered with → prefix via CSS ::before -->
  </ul>
</div>
```

### Admin banner (rust)
```html
<div class="admin-banner">🎯 <strong>Facilitator action required:</strong> When everyone...</div>
```

### Participant waiting message
```html
<div class="waiting-msg">⏳ <strong>Waiting for facilitator</strong> to open voting.</div>
```

---

## Firebase Security Rules (`database.rules.json`)

```json
{
  "rules": {
    "lean-cafe": {
      "sessions": {
        "$sessionId": {
          ".read": true,
          ".write": "auth != null || data.exists()",
          "phase":   { ".write": "auth != null && auth.uid === data.parent().child('adminUid').val()" },
          "timers":  { ".write": "auth != null && auth.uid === data.parent().child('adminUid').val()" },
          "rooms":   { ".write": "auth != null && auth.uid === data.parent().child('adminUid').val()" },
          "cards":        { "$cardId": { ".write": true } },
          "participants": { "$uid":    { ".write": true } },
          "roomData":     { ".write": true }
        }
      }
    }
  }
}
```

---

## PDF Report (jsPDF)

**Load as classic script (not ES module):**
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
```
Access via `window.jspdf.jsPDF`.

**Structure:**
1. **Cover page** — black bg, title, team, date, 5 stat boxes in a row
2. **Cards page** — all cards sorted by votes; top-N (room-creating) cards have rust left accent
3. **One page per room** — participants, root cause (sage box), action items (rust boxes with @assignee + date)

**Page overflow check:** `const cy = n => { if (y + n > 278) { doc.addPage(); y = 15; } }` — call before every element.

**Footer on cover:** `hakanadiguzel.com · Lean Café Retrospective`

---

## File Structure

```
lean-cafe-firebase/
├── public/
│   └── index.html          ← entire app (single file)
├── database.rules.json     ← Firebase RTDB security rules
├── vercel.json             ← static deploy config
├── README.md               ← setup guide
└── .gitignore
```

`vercel.json`:
```json
{
  "version": 2,
  "builds": [{ "src": "public/**", "use": "@vercel/static" }],
  "routes": [{ "src": "/(.*)", "dest": "/public/$1" }]
}
```

---

## Implementation Checklist

- [ ] Firebase config placeholder values present (documented for replacement)
- [ ] Google Sign-In works; auth badge shown in topbar
- [ ] Participants can join without auth; random UID generated each time
- [ ] Session created in Firebase RTDB at `lean-cafe/sessions/{id}`
- [ ] `onValue()` listener active; only re-renders on phase change
- [ ] Timer: per-phase, stored in Firebase, visible to all, only admin can start/stop
- [ ] Phase 1: cards add/delete via Firebase; no input clearing on sync
- [ ] Phase 2: votes are uid-based boolean maps; unvote removes key; button disabled correctly
- [ ] Phase 3: room join via atomic `update()` patch (remove from all, add to one)
- [ ] Phase 4: room modal saves rootCause + actions; soft warning for missing assignee/date
- [ ] Phase 5: report renders; PDF downloads with cover + cards + room pages
- [ ] Assignee maxlength 20, name fields maxlength 15 with live counter
- [ ] No `onclick` attributes — all `addEventListener`
- [ ] Google Analytics tag included
- [ ] "Built by Hakan Adıgüzel" with hakanadiguzel.com link in home footer
- [ ] `database.rules.json` restricts phase/timer/room writes to admin only
