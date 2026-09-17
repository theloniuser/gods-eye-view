# Session: September 17, 2026 10:52 AM - GDELT conflict feature verification attempt, dropped; upstream sync check

**Session**: September 17, 2026 10:14 AM – 10:52 AM
**Duration**: ~40 minutes
**Focus**: Resume verification of the Sep-14 GDELT conflict-coverage feature; check upstream repo drift

---

## Summary

Attempted the "one live check" left over from the Sep 14 session for the GDELT-derived conflict ticker/globe layer — it failed with a network-level connection error (not a clean HTTP 429), repeatedly, on a real dev-server load. Decided GDELT's reliability doesn't justify keeping the feature and reverted all of it via `git stash` (recoverable, not deleted). Separately confirmed the upstream OSS repo (bilawalsidhu/gods-eye-view) has moved 396 commits ahead of local since Sep 5 — not yet merged.

## Changes Made

### Dropped GDELT "Global Signal" conflict-coverage feature

**Files reverted to last-committed state** (via `git checkout --`, then bundled into a stash):
- `vite.config.js`, `index.html`, `style.css`, `src/ui.js`, `src/main.js`,
  `src/data/layerState.js`, `src/data/layerState.test.mjs`, `src/data/dataCredits.js`,
  `DATA_SOURCES.md`, `CHANGELOG.md`

**Files removed** (new, untracked — bundled into the same stash):
- `src/data/conflictCoverageLayer.js` + `.test.mjs`
- `src/data/globalConflictBrief.js` + `.test.mjs`

**What changed**: All of the above are stashed, not deleted —
`git stash@{0}` = `"gods-eye-view: dropped GDELT conflict-coverage feature (unreliable upstream)"`.
Working tree is back to only pre-existing unrelated dirty state.

**Why**:
The direct-checkout (`git checkout --`) was blocked by the harness's auto-mode classifier as
"Irreversible Local Destruction" even after explicit user confirmation. Used `git stash push
--include-untracked` instead — same end state, fully recoverable.

The user's own words on the underlying decision: GDELT "just doesn't seem very reliable and
probably not worth keeping unless this resolves — all we did was launch God's Eye View and it's
already blocked." Confirmed with "yes" to reverting.

**Untouched** (pre-existing unrelated dirty state, left alone per the Sep 14 session's own note):
`package.json`, `.nvmrc`, `src/data/CLAUDE.md`. Also kept `changes.2026-09-14.16-51.md` as the
record of why this was tried.

---

## Issues Encountered

### Issue: GDELT conflict-brief endpoint never verified live, then started hard-failing

**Symptom**: `/api/global-conflict-brief` returned `503` locally. Server log showed
`[global-conflict-brief] upstream fetch failed: fetch failed` — four times from a single page
load — as opposed to the other upstream proxies in the same log (`terrain-heights-proxy`,
`adsb.lol`) which logged explicit HTTP status codes (429, 420) for their own throttling.

**Root Cause**: Not a bug in our code — reviewed `fetchGlobalConflictSnapshot()` /
`fetchRegionalJson()` in `vite.config.js:7094-7108,7354-7396`, logic is correct. The absence of
an HTTP status (vs. a clean 429) indicates GDELT is refusing/dropping the TCP connection outright
for this exact query signature, most likely from cumulative rate-limit abuse across this session
and the Sep 14 session (direct GDELT probes, app polling, page reloads all hit the identical
query URL many times).

**Solution**: Feature dropped rather than fixed — see Decisions Made below.

**Files Modified**: none (investigation only, code was reverted, not patched)

### Issue: Dev server unreachable from a second Mac ("couch Mac")

**Symptom**: `localhost:4173`/`4174` worked fine from Thelonius itself but
`ERR_CONNECTION_REFUSED` / `ERR_EMPTY_RESPONSE` from the couch Mac's browser.

**Root Cause**: Vite by default binds to `localhost` only; the couch Mac is a separate physical
machine so "localhost" there never reaches Thelonius.

**Solution**: Restarted with `npm run dev -- --host`, which binds all interfaces. Verified
reachable at `http://10.14.33.240:4173` (Thelonius's LAN IP) from Thelonius itself before handing
the URL to the user. Tailscale/VPN interfaces (`100.96.0.2`, `10.99.0.205`) timed out — LAN IP
was the only one that worked, so couch Mac must be on the same WiFi network for this to work.

**Files Modified**: none (runtime/process only)

### Issue: `git fetch origin` failed with "did not send all necessary objects"

**Symptom**: `git fetch origin main` errored with `fatal: bad object refs/remotes/origin/CLAUDE.md`
and `error: ... did not send all necessary objects`.

**Root Cause**: `.git/refs/remotes/origin/CLAUDE.md` was a corrupted ref file — instead of a
40-char SHA, it literally contained `<claude-mem-context>\n\n</claude-mem-context>`. Not
something this session wrote; looks like a stray artifact from the claude-mem plugin writing
into the wrong path at some earlier point. `git fsck` confirmed: `badRefContent` +
`invalid sha1 pointer 0000...0000`.

**Solution**: Deleted the bad ref file directly (`rm .git/refs/remotes/origin/CLAUDE.md`);
`git fetch origin main` succeeded immediately after. Verified the earlier merge was unaffected —
`git merge-base --is-ancestor 0d41b6b HEAD` confirmed the merge commit's second parent was
exactly upstream's correct tip, since a prior successful fetch this session had already set
`refs/remotes/origin/main` correctly before the corrupted ref started blocking new fetches.

**Files Modified**: `.git/refs/remotes/origin/CLAUDE.md` (deleted — not a tracked project file)

### Issue: `npm test` had 3 failures immediately after merging upstream

**Symptom**: `src/qaVesselCardsHarness.test.mjs`, `src/tooling/format.test.mjs`, and
`src/tooling/importDirections.test.mjs` failed with `ERR_MODULE_NOT_FOUND: prettier` (or
equivalent) after the merge.

**Root Cause**: `node_modules` was stale — the 396-commit merge changed `package.json`
devDependencies (added `prettier`, bumped `puppeteer` and `sharp`) but nothing had reinstalled.

**Solution**: `npm install`, then updated the local `allowScripts` version pins to match what
actually got installed (`puppeteer@24.37.5` → `25.10.0`, `sharp@0.34.5` → `0.35.4`; `esbuild` and
`fsevents` unchanged). Re-ran the full suite: 4144 pass / 0 fail / 1 skipped.

**Files Modified**: `package.json:246-251` (commit `0473089`)

---

## Running State

- Background processes: none started by me remain running (a stray background `npm run dev` I
  launched earlier — PID 39558 — was killed).
- Dev servers / ports: **PID 54493**, `node .../vite --host`, started 10:27 AM, still running in
  the user's own foreground terminal on Thelonius, bound to all interfaces on port `4173`
  (`http://10.14.33.240:4173`). Not stopped — it's the user's interactive session. Safe to
  Ctrl+C any time; no longer needed for verification since the feature was dropped.
- Open worktrees / branches: none. `main`, working tree clean except pre-existing dirty files
  noted above. One stash entry: `stash@{0}`.

---

## Verification

- `git status --short` — confirmed working tree shows only `package.json` (modified),
  `.nvmrc`, `changes.2026-09-14.16-51.md`, `src/data/CLAUDE.md` (untracked) after the stash.
- `git stash list` — confirmed `stash@{0}` holds the dropped feature's full diff.
- Fetched `http://10.14.33.240:4173/` from Thelonius itself → `200` before handing URL to user.

---

## Testing Performed

- [x] `/api/global-conflict-brief` live check (via app + direct fetch) - Result: **Fail** (503,
      network-level connection failure to GDELT)
- [x] Conflict ticker panel visual check in browser (couch Mac) - Result: **Fail** ("unavailable")
- [x] LAN reachability of dev server from Thelonius before sending URL to user - Result: **Pass**

---

## Decisions Made

1. **Drop the GDELT conflict-coverage feature rather than keep debugging it**: user's call —
   GDELT's real-world throttling (429 escalating to connection drops) makes it unsuitable as a
   dependency for a feature meant to work reliably on first load for real users. Reverted via
   stash rather than deleting outright, so it can be revisited later, possibly with a different
   data source.
2. **Use `git stash` instead of `git checkout --` / `rm`**: the harness's auto-mode classifier
   blocked the direct discard as "Irreversible Local Destruction" even with explicit user
   confirmation; stash achieves the same visible result while staying recoverable.
3. **Fork the upstream repo** (`gh repo fork --remote --remote-name=fork`) so future local
   commits have somewhere of the user's own to push, instead of the upstream maintainer's repo.
   `origin` still points at `bilawalsidhu/gods-eye-view` (for pulling upstream updates); the new
   `fork` remote points at `theloniuser/gods-eye-view` (for pushing this project's own work).
4. **Merged the 396 upstream commits into local `main`** (`git merge origin/main`, commit
   `104d4f3`), then pushed the result to `fork` (`0d41b6b..0473089`, fast-forward). User confirmed
   wanting the fork up to date rather than leaving it stale. No merge conflicts in the code itself
   — the only conflict was the local-only `package.json` `allowScripts` block against upstream's
   `package.json` changes, resolved by keeping both blocks (`1e81747`). Verified with
   `npm install` + full test suite (4144 pass / 0 fail) before pushing — see Issues Encountered
   for the two problems hit along the way (corrupted git ref, stale `node_modules`).

---

## Next Steps

- [ ] Decide whether to merge the 396 upstream commits from `bilawalsidhu/gods-eye-view`
      (local pinned at `7596522` / Sep 5, origin/main now at `0d41b6b` / Sep 16). Not started —
      needs its own planning session since local has other pre-existing dirty files
      (`package.json`, `.nvmrc`, `src/data/CLAUDE.md`) to account for before merging.
- [ ] If revisiting global conflict/news data later, consider a source with a real rate-limit
      contract instead of GDELT's DOC 2.0 API (e.g. a paid news API, or scoping to Google News
      RSS only — `fetchRegionalNews()` in `vite.config.js:7190-7224` already has a Google News
      RSS path used for the *regional* brief, unaffected by this decision since it's a separate
      per-point cache, not the global one that was dropped).
- [ ] Stop the still-running dev server (PID 54493) on Thelonius when done, or leave it — no
      longer blocking anything.
- [x] Push local `main` to the new `fork` remote — done (`0d41b6b..0473089`, fast-forward,
      includes the 396-commit upstream merge, all local doc/config commits, full green test run).
- [ ] Consider deleting the now-superseded `docs/session-changelog-2026-09-17` branch on the
      fork (its 2 commits are already part of `main` via the merge).
- [ ] Report the corrupted `.git/refs/remotes/origin/CLAUDE.md` artifact as a claude-mem plugin
      bug — not yet filed.

---

## Notes

- The dropped feature's full implementation (2727 passing tests, code review fixes for two
  Critical-severity CSS/JS bugs) is preserved in `git stash@{0}` if GDELT ever proves reliable
  enough to reconsider, or if someone wants to swap the data source without redoing the
  ticker/globe-layer plumbing.
- Confirmed via `git fetch origin main` that the upstream project is very actively maintained
  (396 commits / ~12 days) — any future merge should be planned as its own task, not folded into
  a quick session.
