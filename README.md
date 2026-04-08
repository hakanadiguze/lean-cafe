# ☕ Lean Café Retrospective — Firebase Edition

## Setup (10 minutes)

### 1. Create Firebase Project

1. Go to [console.firebase.google.com](https://console.firebase.google.com)
2. Click **Add project** → name it (e.g. `agile-platform`)
3. Enable Google Analytics if you want (optional)

### 2. Enable Authentication

1. Build → **Authentication** → Get started
2. Sign-in method → **Google** → Enable → Save
3. Add your domain to Authorized domains (add your Vercel domain after deploy)

### 3. Enable Realtime Database

1. Build → **Realtime Database** → Create database
2. Choose region (europe-west1 recommended for EU users)
3. Start in **test mode** (we'll add rules next)
4. Copy your database URL (looks like `https://your-project-default-rtdb.europe-west1.firebasedatabase.app`)

### 4. Deploy Security Rules

In Firebase Console → Realtime Database → Rules tab, paste the contents of `database.rules.json`.

Or via CLI:
```bash
npm install -g firebase-tools
firebase login
firebase init database
firebase deploy --only database
```

### 5. Get Your Config

Firebase Console → Project Settings (gear icon) → Your apps → Add app → Web

Copy the config object and replace the `REPLACE_WITH_YOUR_*` values in `public/index.html`:

```js
const FIREBASE_CONFIG = {
  apiKey:            "AIzaSy...",
  authDomain:        "your-project.firebaseapp.com",
  databaseURL:       "https://your-project-default-rtdb.europe-west1.firebasedatabase.app",
  projectId:         "your-project",
  storageBucket:     "your-project.appspot.com",
  messagingSenderId: "123456789",
  appId:             "1:123:web:abc123"
};
```

### 6. Deploy to Vercel

```bash
git add .
git commit -m "Add Firebase config"
git push origin main
```

Vercel auto-deploys on push if connected to GitHub.

After deploy, add your Vercel URL to Firebase Auth → Authorized domains.

---

## Architecture

```
Firebase Realtime Database
└── lean-cafe/
    └── sessions/
        └── RETRO-A1B2/
            ├── id, title, team, phase, createdAt, ...
            ├── cards/
            │   └── {cardId}: { text, author, uid, votes, voters{} }
            ├── participants/
            │   └── {uid}: { name, uid, isAdmin, votesUsed }
            ├── rooms/
            │   └── room0: { title, votes, participants{} }
            ├── roomData/
            │   └── room0: { rootCause, actions{} }
            └── timers/
                └── {phase}: { running, end, total }
```

## Auth Flow

- **Facilitator**: Must sign in with Google to create a session. Firebase Auth UID is used as their participant UID.
- **Participants**: No auth required. Join via link. A random UID is generated per browser session. Same display name = different UID = independent vote tracking.

## Real-time Sync

- `onValue()` listener on the full session path — any change anywhere triggers the callback
- Phase changes → full re-render
- Same phase updates (votes, cards, room joins, timer) → only update specific DOM elements by ID
- Timer display uses `requestAnimationFrame` for smooth countdown

---

Built by [Hakan Adıgüzel](https://hakanadiguzel.com) · Agile Leader & SAFe Coach
