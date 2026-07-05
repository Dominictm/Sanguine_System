---
name: run-sanguine-web
description: Launch and drive the Sanguine System web app (web/server.js, Express + vanilla JS) in a real browser to verify a frontend or API change. Use whenever asked to run, screenshot, or confirm a fix works in this project's web UI — covers restarting the already-running dev server to pick up code changes, headless Chrome via raw CDP (no Puppeteer in this repo), skipping the onboarding tour, and cleaning up any test data created along the way.
---

# Run / verify the Sanguine System web app

Captures the exact recipe already proven to work in this repo, so it doesn't
get rediscovered (and re-billed) from scratch every session.

## 1. The dev server is usually already running — restart, don't relaunch

`web/wrapper.js` supervises `web/server.js` and restarts it on exit code 75.
Assume a server is already up on `http://localhost:4295` (the user's own
session). To pick up code changes:

```bash
curl -s -X POST http://localhost:4295/api/restart   # wrapper relaunches server.js
sleep 2
```

Only start a fresh one (`node web/server.js` or `npm run dev` from `web/`) if
`curl http://localhost:4295/api/status` fails entirely. Never kill node
processes by name/PID guesswork — you can't tell the dev server apart from
unrelated node processes (MCP servers etc. also show up in the process list).

## 2. Headless Chrome via raw CDP — no Puppeteer/Playwright installed here

```bash
"/c/Program Files/Google/Chrome/Application/chrome.exe" \
  --headless=new --disable-gpu --remote-debugging-port=9333 \
  --user-data-dir="/tmp/cdp-profile-$$" \
  "http://localhost:4295/?city=balmont" > /tmp/chrome.log 2>&1 &
sleep 3
curl -s http://localhost:9333/json/version   # confirms it's up
```

- Use a **fresh, unique `--user-data-dir`** every launch — the previous
  instance is gone (see §4), a stale profile dir can hang.
- `?city=<slug>` (e.g. `balmont`, `paris`) sets the active city via query string.
- Node 22+ has native `WebSocket`/`fetch` — no `ws` package needed. Pull the
  page's `webSocketDebuggerUrl` from `GET /json`, send
  `{id, method, params}` JSON frames, await the matching `id` in the
  `message` handler. `Page.enable` + `Runtime.enable` first, then
  `Runtime.evaluate` to script the page, `Page.captureScreenshot` for visuals.

## 3. Skip the onboarding tour before doing anything else

Fresh profile → fresh tour overlay blocks every click. First evaluate:

```js
Array.from(document.querySelectorAll('button'))
  .filter(b => /пропустить/i.test(b.textContent))
  .forEach(b => b.click());
if (typeof navigate === 'function') navigate('characters'); // or 'locations', 'dashboard', etc.
```

then wait ~1s before interacting further.

## 4. Closing Chrome — CDP only, never `taskkill`

```js
ws.send(JSON.stringify({ id: ++id, method: 'Browser.close' }));
```

**Never** `taskkill /IM chrome.exe` or similar by-image-name kill — this
machine's real Chrome windows match the same image name and have been
closed by accident this way before. `Browser.close` over the CDP socket only
ever touches the one headless instance you opened.

Note: once `Browser.close` fires, that `--remote-debugging-port` is dead —
the next check needs a full relaunch (§2), not a reconnect.

## 5. Verify by measurement, not just a screenshot

`Page.captureScreenshot` is a sanity check, not the proof — headless render
timing means a screenshot can catch a half-painted frame. Prefer
`Runtime.evaluate` reading `getBoundingClientRect()` / `getComputedStyle()` /
`document.elementFromPoint()` for layout claims (e.g. "are these two badges
aligned", "is this button actually hit-testable here") — numbers don't lie
about timing the way a screenshot can.

For responsive/breakpoint bugs, sweep widths with
`Emulation.setDeviceMetricsOverride({width, height, deviceScaleFactor:1, mobile:false})`
rather than checking one viewport size — this repo has a 820px-breakpoint
class of bug (CSS cascade tie broken by source order, not media-query
"specificity") that only reproduces below a width threshold.

## 6. Clean up anything you created to test with

Any character/city/diary entry created through the API purely to exercise a
fix must not survive the session:

- `DELETE /api/characters/:slug` soft-deletes to `characters/_deleted/<slug>/`
  — that's not enough on its own; also remove that folder
  (`rm -rf cities/<city>/characters/_deleted/<slug>`) so it doesn't linger.
- After any test write, `git status --short` and `git diff` the touched
  files — confirm a revert left zero net diff before moving on. Watch for
  the BOM-strip side effect: `PUT /fields` strips a leading `﻿` from
  `<slug>.md` on its first edit ever, which shows up as a one-line diff with
  no content change — harmless, but restore the BOM if you want a truly
  clean `git diff` on someone else's file.
- Always `git checkout -- web/tests/report.html` after `npm test` — it's
  regenerated with a fresh timestamp on every run and isn't meaningful diff.

## Reference: known UI structure

- SPA, hash-free client-side routing via a global `navigate(page)` function
  (`'characters'`, `'locations'`, `'dashboard'`, `'chronicles'`, …).
- Character detail modal: `.char-card[data-name]` → click → `#char-detail-modal`.
  Tabs via `.cdet-tab[data-tab="..."]` (`info`, `bio`, `rels`, `diaries`, `sheet`, `desc`).
  Edit mode toggled by `#cdet-edit-btn`; fields are `.cdet-field-input[data-field="..."]`.
- V20 sheet tab loads `GET /api/characters/:slug/sheet-data` lazily on tab click.
