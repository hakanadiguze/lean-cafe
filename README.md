# ☕ Lean Café Retrospective

A democratic, facilitator-friendly retrospective tool for large groups — built as a single HTML file with zero backend dependencies.

## How it works

1. **Admin creates a session** → gets a shareable link
2. **Participants join** via the link and write cards (free format)
3. **Admin opens voting** → everyone votes with limited tokens
4. **Admin closes voting** → top topics become discussion rooms
5. **Everyone joins a room** → discusses root causes & action items
6. **Session ends** → full PDF report downloads automatically

## Deploy to Vercel

```bash
# 1. Clone / download this repo
git clone https://github.com/YOUR_USERNAME/lean-cafe-retro.git
cd lean-cafe-retro

# 2. Install Vercel CLI (optional for preview)
npm i -g vercel

# 3. Deploy
vercel --prod
```

Or connect this repo to Vercel dashboard → auto-deploys on every push.

## Deploy to GitHub Pages

1. Push to GitHub
2. Go to **Settings → Pages**
3. Set source to `main` branch, `/public` folder
4. Your site will be live at `https://USERNAME.github.io/lean-cafe-retro/`

## Local development

```bash
npx serve public
# open http://localhost:3000
```

## Tech stack

- Pure HTML + CSS + Vanilla JS (no framework, no build step)
- `localStorage` + `BroadcastChannel` for same-device multi-tab sync
- `jsPDF` (CDN) for PDF export
- Google Fonts (CDN)

## Notes

- Sessions are stored in `localStorage` — they persist in the same browser
- For real multi-device sync, replace `localStorage` with Firebase Realtime DB or Supabase (the storage API is isolated in two functions: `saveSess` / `loadSess`)
- No server, no database, no cost

---

Built for [hakanadiguzel.com](https://hakanadiguzel.com) · SAFe & Agile Coaching
