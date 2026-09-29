---
name: gamebox-game-prep
description: "Prepare or repair a self-contained HTML5/WebGL game for upload to kawka.app (formerly GameBox), including its ZIP package and optional player save data."
---

# kawka.app Game Prep

Use this skill when an agent needs to create, finish, package, or repair a browser game so that a creator can upload one ready-to-play ZIP to kawka.app.

The deliverable is the tested ZIP, not merely source code. Work on the game itself and its build output. Do not upload, publish, or alter the kawka.app platform unless the user separately asks for that.

## Public boundary

Describe user-visible packaging and runtime practices, including the documented save-bridge contract below. Do not expose or speculate about private archive validation, scanning, moderation, deployment, or authorization implementation. If something fails, explain the observable symptom and a general fix (for example, a missing relative asset or a nested entrypoint), not the hidden mechanism that detected it.

## Target runtime

Treat the game as a static web build that starts from `index.html` and runs in an isolated iframe.

- The game must boot without a server-side process, database, account, secret, or build step at play time.
- Prefer bundling all required JavaScript, CSS, fonts, images, audio, 3D models, shaders, and data files in the archive. External CDNs, remote fonts, APIs, analytics, and remote configuration are fragile; avoid them or make them optional with a useful offline fallback.
- Never put credentials, private tokens, or privileged API keys in client files.
- Do not depend on cookies, cross-window messaging, a particular parent page, or persistent browser storage for the first playable session. If a high score or setting is optional, wrap storage access in a fallback and keep the game playable in memory.
- Game code runs in an opaque-origin sandbox. Browser `localStorage`, `sessionStorage`, IndexedDB, and cookies are unavailable there. Never add `allow-same-origin` to make storage work.
- Do not require pop-ups, new tabs, top-level redirects, submitted forms, or access to the parent page. Fullscreen, pointer lock, gamepad, audio, and other enhanced browser features must have a graceful fallback.
- Start audio only after a user gesture and handle browsers that deny or interrupt playback.

## Build contract

The final archive must have this shape:

```text
my-game.zip
├── index.html
├── styles.css
├── game.js
└── assets/
    ├── image.webp
    └── sound.ogg
```

The important rule is that `index.html` is directly at the ZIP root. This is wrong:

```text
my-game.zip
└── my-game/
    └── index.html
```

If a framework is used, package the contents of its production output directory (often `dist/` or `build/`), not the project folder and not `node_modules/`. Include only files needed at runtime; leave tests, source design files, `.git`, caches, package-manager folders, and development-only configuration outside the upload archive.

The upload form currently allows a 100 MiB ZIP, up to 2,000 entries, and 500 MiB unpacked. Check the current form if preparing a larger game. Include only assets the creator has the right to distribute; keep the game and its metadata suitable for the site's work-safe catalog.

Use relative, case-correct URLs everywhere:

```html
<link rel="stylesheet" href="./styles.css">
<script type="module" src="./game.js"></script>
<img src="./assets/logo.webp" alt="">
```

Avoid root-relative paths such as `/assets/logo.webp`, filesystem paths, and assumptions about a particular host or subdirectory. Verify every referenced file, including dynamically loaded assets, with the same capitalization used on disk. Configure bundlers to emit relative production asset URLs when the game is meant to be portable.

Keep the entrypoint simple and deterministic:

- use a valid HTML document with a viewport declaration;
- load only the scripts and styles the game needs;
- make the first screen useful while larger assets load;
- show a clear loading or error state instead of leaving a blank canvas;
- avoid code that assumes `window`, `document`, or canvas dimensions are fixed at initial load.

## Player experience

The game should be understandable and playable within the first few seconds.

- Show the goal and the controls before or at the start of play.
- Support keyboard/mouse where appropriate and add visible touch controls when the game is marked as mobile-friendly.
- Provide a clear start, pause/resume, restart, and end-game path. Pause when the page loses visibility if real-time play would otherwise continue unfairly.
- Resize the game when the iframe or device orientation changes. Keep important UI inside safe areas and make controls large enough to use with a thumb.
- Use semantic buttons and labels, visible keyboard focus, readable contrast, and text alternatives for important non-decorative UI. Respect reduced-motion preferences when practical.
- Prevent accidental page scrolling only while the player is interacting with a game surface or control; do not make the whole surrounding page unusable.

For Canvas or WebGL games:

- use `requestAnimationFrame` and make simulation timing stable when frame rate changes;
- cap rendering pixel ratio and avoid allocating large buffers every frame;
- handle resize, context loss, missing assets, and unsupported features without crashing;
- release listeners, timers, audio nodes, and GPU resources when restarting or leaving a screen;
- compress images and audio, remove unused assets, and load expensive content only when needed.

## Optional player saves

Use the kawka.app save bridge only when progress, settings, or a high score should persist between visits. It stores small string values in a per-game save slot in this browser. It is not an account save, does not sync across devices, and may be cleared with site data. Saves survive replacing the game's build.

Send requests from the game iframe to `window.parent` with `postMessage`. Use the known portal origin as `targetOrigin` when available; a portable build can use `"*"` for this non-sensitive game state. The iframe's opaque origin requires `"*"` for the portal's replies to the game, not for every outgoing request. Match each response by a unique `requestId`, verify `event.source === window.parent` and `data.type === "gamebox:storage:result"`, and validate the response fields. If a portal origin is configured, also check `event.origin` against it. The portal returns:

```js
// Read a value; missing keys return value: null.
{ type: "gamebox:storage:get", requestId: "r1", key: "save" }
// Write or remove a string value.
{ type: "gamebox:storage:set", requestId: "r2", key: "save", value: "..." }
{ type: "gamebox:storage:remove", requestId: "r3", key: "save" }
// Response: { type: "gamebox:storage:result", requestId, ok: true, value: string | null }
// Failure:  { type: "gamebox:storage:result", requestId, ok: false, error: string }
```

Serialize structured game state with `JSON.stringify`; include a schema version, validate parsed data, and recover gracefully from an old or corrupt save. Limits are measured as JavaScript string lengths: at most 32,768 per value and 65,536 total across keys and values, with up to 16 keys. Keys must match `[A-Za-z0-9._-]{1,64}`; request IDs must match `[A-Za-z0-9._:-]{1,64}`. Successful writes and removals return `value: null`.

Register the response listener before sending, use a bounded timeout (for example 2 seconds), and clean up pending requests and listeners. Save at checkpoints or debounce changes rather than writing every frame. Handle `value_too_large`, `too_many_keys`, `quota_exceeded`, `storage_unavailable`, and timeouts by continuing with an in-memory game state and informing the player only if relevant. Saves are optional: a failed save must not block starting or playing.

For direct/local play outside kawka.app, use a best-effort fallback (for example localStorage when available, then memory). Never assume that fallback will work in the hosted iframe. If the game project already has a save helper, inspect and reuse it instead of adding a competing storage path.

## Package and verify

Always create the archive from the exact directory that was tested. From that directory, a typical command is:

```bash
cd path/to/game-output
zip -r ../my-game.zip . -x '*.DS_Store' -x '__MACOSX/*'
```

Before handing it off:

1. Inspect the archive listing. Confirm that `index.html` is at the root, there is no extra wrapper directory, and every runtime asset is present.
2. Serve the game output from a local static HTTP server. Testing only with `file://` can hide path and module problems.
3. Open the game in a real desktop browser and a narrow mobile-sized viewport. Check a fresh load, first interaction, restart, pause/resume, resize/orientation, keyboard/mouse controls, touch controls when applicable, audio after a gesture, and fullscreen as an enhancement.
4. Also test in an iframe with `sandbox="allow-scripts allow-pointer-lock"` and `allow="fullscreen; gamepad; autoplay"`, without `allow-same-origin`. For games with saves, check write/read after reload, missing or corrupt data, and an unavailable bridge. A local parent-page fixture can emulate the documented messages; report that as a local simulation, not a production test.
5. Check the browser console for errors and check that required assets load successfully. Test with the network unavailable if the game is intended to be self-contained.
6. Rebuild or re-export and recreate the ZIP after any source or asset change. Do not hand over an archive that predates the last tested output.

Useful inspection commands are:

```bash
unzip -l my-game.zip
node --check game.js
python3 -m http.server 4179 --bind 127.0.0.1 --directory path/to/game-output
```

Use the language/runtime's equivalent checks when the game has multiple scripts or a compiled bundle. Browser verification matters more than a syntax check alone.

## Common corrections

- `index.html` is nested: zip the contents of the output directory, not the directory itself.
- The screen is blank: fix a case-sensitive filename, relative URL, missing production asset, or runtime exception; test from HTTP.
- The game works locally but not after hosting: remove absolute URLs and local filesystem assumptions, then test the production output in a clean directory.
- A framework build works only with its dev server: export a static production build and make all runtime routes/assets resolvable from the entrypoint.
- A score or setting crashes the game: treat storage as optional and catch browser permission or quota errors.
- Sound never starts: wait for a click/tap/key gesture and expose a mute or sound toggle.
- Mobile play is awkward: add touch input, pause on visibility changes, responsive layout, and concise on-screen instructions.
- The upload is unnecessarily large: remove development files and unused assets, then optimize images, audio, fonts, and 3D resources.

## Handoff format

Return:

1. a clickable path to the final ZIP;
2. the archive's root entrypoint and a short runtime manifest;
3. the controls and device support the creator should enter in the upload form;
4. the checks actually performed, including viewport/browser coverage and any known limitation.

Never claim a browser, device, or feature was tested when it was not. If a user asks only for preparation, stop after producing and verifying the ZIP.
