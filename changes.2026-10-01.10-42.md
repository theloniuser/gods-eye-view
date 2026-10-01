# Session 2026-10-01 10:42 — GEV upstream update

## What / why
- Checked upstream (`origin` = bilawalsidhu/gods-eye-view): 97 new commits (weather panel, Cyber HUD theme, rail cards, OSM/Overpass change, CCTV fixes).
- Merged `origin/main` into local `main` — clean, no conflicts (commit b817c4f).
- `npm install` (+5 packages, allowScripts pins still matched), `npm run build` OK, `npm test` 13/13 pass.
- Served dev on LAN (`vite --host`) for the couch Mac; stopped at session end.

## Issues + solutions
- Voice error "WebRTC microphone support unavailable" on the couch Mac: not a bug. `navigator.mediaDevices` is undefined on non-HTTPS, non-localhost origins (check at `src/voice/realtimeConnection.js:61`). Fix: SSH tunnel so the browser sees `localhost`:
  `ssh -N -L 4173:localhost:4173 jamescantwell@<this-mac-ip>` then open `http://localhost:4173`. `-N` prints nothing when connected.
  Alternative: chrome://flags/#unsafely-treat-insecure-origin-as-secure.
- Not verified: new UI features visually; voice on couch Mac via tunnel.
- `npm audit`: 1 low vulnerability, not investigated.

## Next session
- Confirm tunnel + mic works on couch Mac; eyeball weather panel / Cyber HUD / rail cards.
- Look at the low npm audit finding.
- Untracked, left alone: `.nvmrc`, `src/data/CLAUDE.md`.
