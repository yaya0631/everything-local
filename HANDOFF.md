# everything-local: handoff (status, full context, plan)

> This file lets the project continue in a later session or with another AI. It is self-contained.
> **Delete this file in the final cleanup commit.** The finished repo's top level may contain only `apps/`, `platform/`, `README.md`, `LICENSE`, `package.json`, `package-lock.json` and `.gitignore`.

## 1. Links and identifiers

| What | Where | Role |
|---|---|---|
| Original apps (source of truth) | https://github.com/HeyPuter/everything at `b5d25841527e2b772eb34a80d4973c6bc1dc67c1` | MIT at the root. Some apps carry their own LICENSE (for example `notes` is AGPL-3.0); keep each one exactly. |
| Puter platform (behaviour reference only, AGPL-3.0) | https://github.com/HeyPuter/puter at `480fa7b85b81ec5a8034425b093c163f7f34e1bd` (v26.07) | Read `src/docs/src/**` (one page per method plus `Objects/*.md`) as the API spec. **Never copy Puter code.** |
| Owner's earlier apps fork (audited, not reused) | https://github.com/yaya0631/everything-apps (private), HEAD `b760b00a7b1da87c72a02f22629c7c52aca61fb9` | Messy earlier attempt. |
| Owner's earlier platform fork (audited, not reused) | https://github.com/yaya0631/everything-platform (private), HEAD `aaf425f2db7ed19763596ac43d1e38199124950a` | Messy earlier attempt. |
| **Deliverable** | **https://github.com/yaya0631/everything-local** (private) | Created empty. Push all work here. |

## 2. Status at handoff

Done:
- Phase 0 audit and verdict (§5), owner decisions (§4), and the approved plan (this document).
- The private repo `yaya0631/everything-local` exists. This file is its only content.
- Figma Make workspace (only relevant if you continue inside Figma Make):
  - The deliverable is meant to live in `./everything-local/`, which is already ignored by the workspace `.gitignore`.
  - It is also excluded from the Make preview's Vite watcher (`server.watch.ignored`) and dependency scan (`optimizeDeps.entries: ['index.html']`).
- A Make secret `GITHUB_PUSH_TOKEN` was requested so git can push from the sandbox. It should be a fine-grained PAT for this repo only, with Contents: Read and write. It is not confirmed yet.

Not started: every build step in §10. No app or platform code exists yet.

## 3. The brief (condensed; all requirements)

**Goal.** Make the open-source Everything apps (about 50 browser apps) run entirely on one PC.
- Replace Puter (storage, login, database, AI, serverless workers, email, hosting) with a local platform that the owner owns and that is written from scratch.
- Remove every other cloud or third-party runtime call: analytics, CDNs, hosted fonts and external APIs.
- The result must be as simple as the original.

**Input rules.**
- Do not copy code from the Puter repo. Read it only to learn behaviour, which keeps the platform free of AGPL obligations.
- Keep each app's own LICENSE file unchanged.
- Always diff against the pristine originals. They define what "simple" and "correct" mean.

**Target layout** (exactly this):

```
everything-local/
├── README.md        # what it is, how to run it, under 60 lines
├── LICENSE          # same as the original repo root
├── package.json     # npm workspaces; scripts: install (npm default), dev, build, start (justify a fifth)
├── .gitignore
├── apps/<name>/     # every app, same folder names as the original src/, original internal layout
└── platform/
    ├── package.json
    ├── server/      # index.js auth.js fs.js kv.js workers.js mail.js hosting.js ai.js (one small file per service; split only past ~400 lines)
    ├── sdk/         # client library loaded instead of js.puter.com: one source folder that builds to ONE static file
    └── data/        # runtime data (database, files), git-ignored
```

**Hard layout rules.**
- Only `apps/` and `platform/` as top-level code folders. No `tools/`, `scripts/`, `docs/`, `misc/`, `old/`, `backup/`, `tmp/`, `shared/` or `utils/`.
- App internals match the original.
- No generated or duplicate files in git: no `dist/`, `node_modules/`, per-folder lockfiles, `.bak`, `*-copy`, `v2`, or commented-out dead code.
- Delete anything that is not in the original and not required by the platform.

**API surface the brief lists** (its own count of call sites; the real surface is larger, see §6):
- **fs:** write, read, delete, readdir, stat, rename, getReadURL, revokeReadURL, space, share, unshare, listShared, getShares.
- **kv:** set, get, del, list, update, add, incr, plus the MAX constants.
- **auth:** signIn, signOut, isSignedIn, getUser, getProfilePicture, whoami.
- **workers:** create, exec, delete, list, get.
- **email:** send, sendTransactional, list.
- **hosting:** create, delete.
- **perms:** request, check.
- **ui:** showOpenFilePicker, showSaveFilePicker, launchApp, contextMenu, parentApp, onLaunchedWithItems, onItemsOpened, setWindowTitle, showWindow, hideWindow, requestPermission, onWindowClose, plus picture-in-picture request and exit.
- **ai:** chat, txt2img, txt2vid, speech2txt, txt2speech, img2txt.
- **net:** fetch, Socket, tls (only if an app breaks without them).
- **env** and **exit**.
- **Workers:** plain JS files using `router.get/post/put/delete/…`. They receive `me.puter` (the owner) and `user.puter` (the caller). Apps deploy them with `workers.create` and call them with `workers.exec`.

**Recommended architecture.**
- A drop-in SDK exposing `window.puter`, with the same method names and return shapes as Puter, talking to the local server. Apps then need only a mechanical change.
- One plain Node server with minimal dependencies:
  - Each app on its own origin, `http://<app>.localhost:<port>`, so per-app storage stays isolated.
  - The API for fs, kv, auth, workers, mail, hosting and ai.
  - Everything stored under `platform/data/`: one SQLite file for kv, accounts and metadata; ordinary files for `fs`.
  - Starts with one command.
- Local accounts with tokens. `signIn()` works offline. Single-user by default is fine, but the multi-user cases (teamchat, shared files, perms) must still work.
- Workers run in a worker thread or `vm` context with `router`, `me` and `user`, and keep their original source format.
- Email goes to a local mailbox store. SMTP is optional, not required.
- `hosting.create` publishes a folder at a local subdomain.
- AI: one small adapter interface in `ai.js` and one adapter for a local runtime (an Ollama-compatible chat endpoint). Image, video and speech methods return a clear "not available locally" error, and apps must degrade gracefully. Never call an external AI service.

**What to remove.**
- All references to `js.puter.com`, `api.puter.com`, `puter.com`, `*.puter.site`, `developer.puter.com` and `docs.puter.com`. Replace them with the local SDK or delete them.
- `<app>.puter.com` demo links in READMEs and `STORE_LISTING.md` (delete or make local).
- `plausible.io` analytics.
- Strict CSP headers that whitelist Puter hosts (chess, backgammon, checkers). Point them at the local origins instead.
- Hosted fonts and CDN scripts. Bundle them locally.
- Marketing links that act as runtime dependencies (for example cap-table's landing page).
- Cloud-only files: `amplify.yml`, `customHttp.yml` (root and per app), `dev/puter-localhost.mjs`, and the per-app `puter worker deploy` scripts.

**Verification required by the brief.**
- `grep -rE "puter\.(com|site)|js\.puter|plausible|https?://" apps platform --include=*.js --include=*.html --include=*.css` must show no runtime fetches to the internet.
- Browser dev tools, or a headless browser with the network disabled, must show zero external requests while the apps are used.

**Quality bar.**
- Each app builds and runs from the root with its original tooling (Vite and Vitest).
- Tests that passed on the originals still pass. Tests of Puter's cloud behaviour are updated to the local SDK, never deleted.
- The platform has its own small test suite, kept beside the code, that checks every listed method against the behaviour the apps expect.
- Readable code: no boilerplate, no giant files, no TODOs.
- The README covers what it is, requirements, `npm install`, `npm run dev`, where the data lives, backups, and enabling AI.

**Acceptance checklist** (report each line as pass or fail):
1. Fresh clone, then `npm install`, then `npm start`, serves every app locally. One command, no manual steps.
2. The top level contains only `apps/`, `platform/`, `README.md`, `LICENSE`, `package.json` and `.gitignore`, plus the lockfile.
3. `apps/` has the same set of apps as the original `src/`, each with its original internal layout and no extra files.
4. With the network disconnected, sign in, create and save data in at least:
   - notes (fs + kv)
   - spreadsheet (worker + email path)
   - teamchat (workers + multi-user)
   - video-studio (fs + AI degrade)
   - chess (auth only)

   The data survives a server restart.
5. Searching `apps/` and `platform/` for any Puter or analytics host returns nothing used at runtime.
6. Every listed method is implemented or explicitly documented as unsupported, and the apps handle the unsupported case without crashing.
7. Outside `platform/`, zero files exist that are not in the original, except the SDK `<script>` change.

**Working rules.**
- The audit and verdict come first (done).
- Then work in small, reviewable steps and tell the owner what changed after each one.
- Leave nothing half-finished. Say plainly in the README what is unsupported.
- Add no new features, frameworks, ORMs, build systems or folders, and take the simpler option.
- If a requirement conflicts with reality, say so and propose the smallest fix.

## 4. Decisions

**From the owner** (they picked the recommended option each time):
1. **Scope:** the listed surface plus everything the apps actually need. That means live updates (`events.onLocal`), KV shares and handles, `kv.expire`/`remove`/`decr`, `fs.mkdir`/`getShareLink`, token-read URLs, Puter-shaped tokens, `email.get`, the Email app's raw send, and the full `perms` call forms.
   **`puter.peer` is NOT implemented.** Every peer call rejects with `{code:'not_available_locally'}`.
2. **whiteboard:** include it, but drop its tracked `tmp/` folder. This is the only change to any app's layout.
3. **Per-app lockfiles:** delete all 51. The 12 `tools/check-licenses.mjs` files read the root `package-lock.json`, walking only their own app's dependency closure (see §7).
4. **Sign-in:** one-click local accounts with no passwords. The server listens on 127.0.0.1 only.
5. **Delivery:** the private repo yaya0631/everything-local. Fallback: `git bundle create everything-local.bundle --all`.

**Made in the approved plan:**
- **Fixed port 4400.**

  | Host | Purpose |
  |---|---|
  | `http://localhost:4400` | Launcher and `/signin` window |
  | `http://api.localhost:4400` | REST API |
  | `http://<app>.localhost:4400` | Each app |
  | `http://<name>.work.localhost:4400` | Workers; replaces `https://<name>.puter.work` |
  | `http://<sub>.site.localhost:4400` | Hosted sites; replaces `https://<sub>.puter.site` |

  The SDK is served at `/puter.js` on every app and site host. The port is one constant, because app code embeds it in worker and site URLs.
- **No runtime npm dependencies.** Only Node built-ins: `node:http`, `crypto`, `fs`, `worker_threads`, `vm` and `node:sqlite`. Vite, already in the tree, builds the SDK. `playwright-core` drives the browser tests using a system Chromium.
- **Live updates use Server-Sent Events read through `fetch`,** so there is no WebSocket server.
- **Mail never leaves the PC.**
  - Messages are delivered as RFC822 files into each local recipient's `~/.mail/objects/…`, which the Email app already reads.
  - Other recipients produce a bounce in the sender's inbox.
  - There is no SMTP.
- **AI uses one OpenAI-compatible `/chat/completions` adapter.**
  - `AI_URL` defaults to `http://127.0.0.1:11434/v1` (Ollama) and `AI_MODEL` to `llama3.2`.
  - Any host other than loopback is refused.
  - Every other AI modality returns `not_available_locally`.
- **The platform sends one local-only CSP on every app and site response.** It replaces the `customHttp.yml` CSPs.
- **Root scripts:** `dev`, `build`, `start`, plus `postinstall`. The fifth script is justified because presentation's `fflate` patch must be applied from the root once `fflate` is hoisted.
  - Tests run with `npm test --workspaces --if-present`, which works with no root `test` script.
- **`platform/server/events.js`** is added beyond the eight listed files. It is justified because live updates and KV handles span both fs and kv, and 7 apps, including teamchat, need them.
- **Requirements:** Node 22.18+ (presentation's `engines` field) and npm 10.9+. Browsers: Chrome, Edge and Firefox resolve `*.localhost` on their own; Safari is not promised.

## 5. Phase 0 verdict (evidence)

**Apps: re-derive from b5d2584.**
- The apps fork's base is upstream `0fc4d361`, which is 64 commits and 3,019 files behind b5d2584. 16 apps are stale (spreadsheet 854 files, 3d 766, documents 650, …).
- It has 15 fork-only Copilot commits: about 106 file changes, +20.8k lines and 26 new files.
  - Required for local operation: 0. The fork never replaced the `js.puter.com` tags; its platform rewrote apps at serve time.
  - Unnecessary: about 94 (AI assistants and model pickers in 9 apps, a +4,060-line CAD rework, test-timing tweaks).
  - Harmful: 10 (a generated 12,562-line teamchat worker bundle committed to git, and invented AssetDesk source in the `assets` placeholder).
  - Idea worth keeping: CRLF-safe hashing in the documents and help-desk license checks.
- 31 apps are byte-identical to b5d2584.

Fork-only commits:

| Commit | Files | Change | Class |
|---|---|---|---|
| 1d2588e8 | 27 | cad: AI tools, live collab, `local-cloud.js`, assistant (+4,060 lines) | unnecessary, harmful |
| e750f374 | 27 | Model pickers in 7 apps, worker paths for `/h/`, generated teamchat bundle (+13,093 lines) | unnecessary, harmful |
| 48b1d7d8, 852552d9 | 9 | Invented AssetDesk app | harmful |
| a84e06a8, c0b98f97, 0936adcc, b760b00a | 16 | photo-studio assistant work | unnecessary |
| 0e023619, 0af51c7f | 7 | documents assistant work | unnecessary |
| 522a5a6e, bba8dead | 10 | whiteboard assistant agent | unnecessary |
| 8189d758 | 8 | building-design assistant | unnecessary |
| 630ececf, 9b980048 | 2 | Windows CRLF license-hash fixes | idea kept |

**Platform: rebuild from scratch.**
- **Rule (b) triggers.** About 160 files, plus a `verification/` tree. Mapping them onto the target tree would move nearly every file.
  - 39 files at the top of `platform/`: 15 e2e and smoke scripts, 11 committed result JSONs and an empty `_smoke.txt`.
  - 39 `lib/` modules (16 named `hub-*`), an admin panel (11 files), 49+ tests, docs (8 files, about 264 KB of AI logs), vendored Pyodide and an engine installer.
  - Duplicates: two SDKs (`lib/shim/puter.js` 119 KB, `sdk/puter.js` 14 KB), two app registries and two hub entry points.
- **Rule (a) does not trigger.** A static read of the 58 listed entries found:
  - Stubs: `auth.signIn` (`notYet`), `ui.contextMenu` (logs only).
  - Missing: `auth.getProfilePicture`, `fs.revokeReadURL`, `ui.requestPermission`.
  - No-ops: `showWindow`, `hideWindow`.
  - The fork was not executed.
- **Rule (c) not observed.** Only 3 of 744 distinctive shim lines match Puter, all generic.
- **Harmful by design:**
  - Cloud AI providers (OpenRouter).
  - A public Tailscale funnel listener.
  - Opt-in NHTSA calls.
  - All apps on one origin under `/h/<app>/`.
  - 33 KB of regex rewriting of app code at serve time.
  - Monkeypatched `fetch`, `XMLHttpRequest`, `setAttribute` and link `href`.
  - Test hooks inside the SDK.
  - A dependency on `../upstream/src/cad`.

**Where the brief differs from reality** (each is resolved in §4):
- whiteboard exists.
- No app uses `@heyputer/puter.js`.
- The surface is wider than the brief lists.
- Workspaces conflict with npm 10, `patch-package`, pdf's paths and the license gates.
- Several runtime internet calls exist.
- presentation needs Node 22.18.

## 6. Measured facts about b5d2584 (inputs for the work)

**Apps.** There are 51 folders in `src/`:

3d, accounting, api-client, assets, audio-studio, backgammon, building-design, cad, calendar, cap-table, checkers, chess, childcare, circuits, contacts, contracts, crm, documents, email, expense, field-service, fleet, help-desk, hiring, inventory, invoicing, legal, meetings, music-studio, notes, parametric, pdf, photo-studio, pos, presentation, projects, property-management, records, shifts, signatures, solitaire, spreadsheet, sudoku, survey, tasks, teamchat, time-tracking, vector-studio, video-studio, whiteboard, wiki.

- The repo tracks 34,458 files.
- `assets` is a placeholder: its `index.html` points at a missing `/src/main.js`, and the original CI skips it.

**Sizes.**
- whiteboard: 576 MB, of which `tmp/` is 562 MB and 4,375 files. Nothing reads `tmp/`.
- documents: 381 MB, of which `tests/` is 332 MB.
- spreadsheet 61 MB, presentation 35 MB, 3d 32 MB.

**Tooling.**
- Every app has its own `package.json`, `package-lock.json` and `vite.config.(js|ts)`.
- Vite majors 6, 7 and 8 and Vitest majors 2 to 5 are all in use.
- whiteboard tests with `node --test`, using miniflare and `@napi-rs/canvas`.
- presentation declares `engines: node >=22.18` and runs `postinstall: patch-package --error-on-fail` with `patches/fflate+0.8.3.patch`.
  - circuits and documents also pin `fflate` 0.8.3.
  - The patch adds a size guard and also tightens `unzipSync` by default.
- 50 `vite.config` files import `../../dev/puter-localhost.mjs` and register `puterLocalhost('<name>')`.

**How apps load the SDK.**
- About 50 `<script src="https://js.puter.com/v2/">` tags in `index.html` files (contacts has 3, email 2).
- JS loaders:
  - `3d/src/assistant/puter.ts` (`SDK_URL`).
  - `backgammon|checkers|chess/src/net/online.js` (`PUTER_SRC`, loaded lazily for online play).
- Service-worker constants:
  - `accounting/vite.config.js` and `invoicing/vite.config.js` generate `sw.js` with `const SDK = 'https://js.puter.com/'`.
  - `calendar/public/sw.js` (`SDK_ORIGIN`), `expense/public/sw.js` (`isPuterSdk`), `fleet/public/sw.js` (`SDK`), `inventory/public/sw.js` (`SDK_URL`).
  - `contacts/public/sw.js` (comment only) and a regex in `contacts/src/main.js:24`.
- `@heyputer/puter.js` is not used anywhere.

**Analytics and cloud config.**
- **Plausible appears in 9 apps.** Among them are backgammon, calendar, checkers, chess, contacts and inventory; grep `plausible.io` to find the rest.
  - cap-table gets it through `site/analytics.html`, which `vite.config.js` injects at `<!-- site-analytics -->`.
  - The apps' `analytics.js` wrappers call `window.plausible?.()`.
- **`customHttp.yml`** exists at the root and in backgammon, checkers, chess, spreadsheet and teamchat.
  - It sets only `Cache-Control` (`no-cache` for HTML, `immutable` for `/assets/*`), wasm `Content-Type`, and 3 security sets including CSP.
  - There is no COOP/COEP.
- **`amplify.yml`** holds build settings only, with no rewrites.

**Developer-deployed workers.** These become platform "developer workers".

| Package script | Source file | Worker name |
|---|---|---|
| calendar `deploy:email-worker` | `src/workers/email-worker.js` | `calendar-email` |
| field-service `deploy:email-worker` | `src/workers/email-worker.js` | `field-service-email` |
| invoicing `deploy:email-worker` | `src/workers/email-worker.js` | `invoicing-email` |
| property-management `deploy:email-worker` | `src/workers/email-worker.js` | `property-management-email` |
| signatures `deploy:email-worker` | `src/workers/email-worker.js` | `signatures-email` |
| spreadsheet `deploy:email-worker` | `src/workers/email-worker.js` | `spreadsheet-email` |
| teamchat `deploy:email-worker` | `worker/email.js` | `teamchat-email` |
| meetings `deploy:worker` | `worker/rooms.js` | `puter-meetings-rooms` |

- calendar's `publish:mail-handler` script (`scripts/publish-mail-handler.mjs`) publishes an events handler, which is unsupported: delete it.
- `.github/workflows/deploy-meetings-worker.yml` also exists.

**Worker entry files** (router-based, no imports):
- calendar `src/workers/email-worker.js` and `rsvp-worker.js`.
- documents `src/collab/server/index.js`.
- field-service, invoicing, property-management, signatures and spreadsheet `src/workers/email-worker.js`.
- signatures `src/workers/sign-worker.js` and meetings `worker/rooms.js`.
- photo-studio `src/collab/server/index.ts` and spreadsheet `src/collab/server/index.js`.
- survey `src/worker/collector.js`.
- teamchat `worker/index.js` and `email.js`.
- vector-studio, video-studio, whiteboard and wiki `worker/index.js`.

Notes:
- Bundles come from each app's `vite.worker.config.*` and land in `public/worker/`.
  - photo-studio, spreadsheet, whiteboard and wiki commit theirs. Rebuild them after editing their sources.
- Workers built at runtime: `src/team.js` in childcare, field-service, fleet, legal and property-management, and help-desk `src/client/deploy.js`.
- Workers use `router`, `me.puter.{kv, email.send, email.sendTransactional, authToken, APIOrigin}`, `user.puter.{email…, authToken}`, `crypto.subtle`, `fetch` and `setTimeout`.

**`workers.create(name, path)` call sites:**
- calendar `src/js/rsvp-links.js`.
- `src/team.js` in childcare, field-service, fleet, legal and property-management.
- documents `src/collab/api.js`, photo-studio `src/collab/api.ts`, spreadsheet `src/collab/api.js`, vector-studio `src/features/collab/api.js`, whiteboard `src/collaboration/api.js`.
- help-desk `src/client/deploy.js`, signatures `src/lib/signlinks.js`, survey `src/data/collector-client.js`, teamchat `src/lib/deploy.js`, video-studio `src/review/api.js`.

The flow is always:
1. Fetch the app's own bundle.
2. `fs.write` it to the user's drive.
3. Call `workers.create`.
4. Poll the worker URL until it answers.

**Worker and site URLs in runtime code.**
- `puter.work` lines (comments excluded, about 116): calendar 10, childcare 5, documents 5, field-service 8, fleet 6, help-desk 3, invoicing 1, legal 6, meetings 1, photo-studio 5, property-management 7, signatures 4, spreadsheet 6, survey 6, teamchat 20, vector-studio 6, video-studio 5, whiteboard 7, wiki 5.
- `puter.site` (13 lines):
  - contracts `src/lib/calendar.js`.
  - documents `src/lib/publish.js` and `src/lib/puter.js`.
  - hiring `src/ui/publish.js`.
  - invoicing `src/lib/share.js` and `src/workers/email-worker.js`.
  - shifts `src/ui/share.js`.
  - spreadsheet `src/workers/email-worker.js`.
  - teamchat `worker/email.js`.
  - vector-studio `src/features/collab/protocol.js`.
  - whiteboard `src/organization/email-domains.js`.

**Direct REST calls on `APIOrigin`.**
- email `src/ui/compose.js`: `POST ${APIOrigin}/email/send` with `content-type: message/rfc822`.
- The collab `auth.js` in documents, photo-studio, spreadsheet, video-studio (`review/`), whiteboard and wiki check for `/token-read` (details below).
- `${APIOrigin}/events/kv-handles` is used in:
  - the photo-studio and spreadsheet collab `server/service.*`;
  - teamchat `worker/events.js`;
  - whiteboard `worker/service.js` and `retention.js`;
  - wiki `worker/service.js`.

**Other API use.**
- **peer:** backgammon, checkers, chess (online play), meetings (core), teamchat (huddles), and photo-studio, spreadsheet, vector-studio and whiteboard (presence). Methods: serve, connect, ensureTurnRelays, createGuestGrant.
- **`events.onLocal`:**
  - email (watches `.mail`), photo-studio, spreadsheet, teamchat, whiteboard and wiki.
  - calendar uses `events.list`.
- **KV extras:**
  - `expire` (calendar, help-desk, video-studio, whiteboard, wiki), `remove` (help-desk), `decr` (calendar).
  - tasks uses `share`, `unshare`, `listShared` and `listSharedByMe` in `src/lib/collab.js` and `store.js`.
  - `kv.set` takes a third `{disableSharing:true}` argument.
- **`perms.request` forms:**
  - `request('email')`.
  - `request('permission', {permission:'fs:/<user>/.mail:write', create:true})`.
  - Arrays of requests.
  - `request({resource:'permission', permission:'events:background'})`.
  - `request(perms.Resource.KV_SHARE, {prefix:'ss:', access:'watch'})`.
  - `check('permission', {permission})`.
- **email:** `list({folder:'sent', limit})` and `get({id, folder})` in calendar `mail-intake.js`. calendar's `mail-handler.js` is an events worker, which is unsupported.
- **`net.fetch`:** spreadsheet `src/lib/puter.js`, whiteboard `src/content/google.js` and `migration/host.js`, wiki `src/ui/app.js`. `net.Socket` and `tls` are used by api-client.
- **`puter.env`** is a string: apps check for `'app'`, `'gui'` or `'web'`. audio-studio calls `puter.env()` as a function; keep Puter parity with a string.

**Tokens and auth details the apps depend on** (see spreadsheet `src/collab/auth.js`).
- Tokens are JWT-shaped (`a.b.c`), with a base64url JSON payload:
  - `type` is `'access-token'`, `'app-under-user'` or `'session'`. The compressed form uses `t` with `'t'`, `'au'` or `'s'`.
  - `user_uid` (compressed: `uu`), `full_access` and `exp`.
- Workers read `app_uid` from `me.puter.authToken` (the `workerAppUid()` helper).
- `fs.getReadURL(path, '24h')` takes durations as strings. It must return `${APIOrigin}/token-read?token=<access-token>`; apps check both the origin and the path.
- `kv.set` and `fs.write` take `{overwrite, dedupeName, createMissingParents}` options.
- `workers.exec` sends a `puter-auth: <token>` header. documents sends `x-puter-no-auth: 1` on `/health`.

**KV-handle REST** (teamchat `worker/events.js`).
- Mint: `POST /events/kv-handles` with `Authorization: Bearer <me.puter.authToken>` and body `{granteeUid, prefix}`. Returns `{handle}`.
- Revoke: `DELETE /events/kv-handles/<handle>`. A 404 with code `subject_does_not_exist` counts as success.
- Error codes used: `events_kv_handle_not_delegated`, `events_kv_handle_owner_only`, `events_kv_handle_outside_namespace`, `events_kv_handle_limit_reached`, `too_many_requests`, `recipient_not_accepting_shares`, `invalid_kv_share_prefix`.

**Email app mailbox layout.**
- Incoming: `/<username>/.mail/objects/<YYYY-MM-DD>/<uuidv7>--<base64url subject>`, raw RFC822.
- Sent copies: `.mail/sent/<day>/<name>`, written by the app itself.
- Live updates come from `events.onLocal` on `.mail`.

**Internet use at runtime.**
- 3d `src/scripting/python.ts`: `PYODIDE_INDEX_URL` points at cdn.jsdelivr.net. `pyodide` is already a 3d devDependency.
- fleet `src/core/vin.js` and `recalls.js` call NHTSA.
- whiteboard:
  - `src/migration/miro-client.js` calls api.miro.com.
  - `src/content/google.js` imports Google content.
  - The organization admin has a Slack webhook.
  - Video and doc embeds.
- cap-table site analytics.
- Not fetched, only documenting provenance: presentation `src/assets/fonts/SOURCES.json` and spreadsheet `public/fonts/caladea/README.txt`.
- Plain outbound links (maps, search, docs, GitHub) may stay.

**Puter clutter to clean.**
- "Built on Puter" anchors to developer.puter.com in 31 apps.
- `canonical`, `og:*`, `twitter:*` and JSON-LD tags in `index.html` files.
- `robots.txt` `Sitemap:` lines and `public/sitemap.xml` (8 apps).
- `https://puter.com/app/calendar` (calendar `public/book.html`) and `<app>.puter.com` links.
- **Keep** documents' DOCX ink namespace `http://schemas.puter.com/documents/2026/ink`. It is an identifier that is never fetched, and changing it would break existing files.

**Tests that mention Puter hosts or Plausible** (number of files per app):
- pdf 225, whiteboard 183, presentation 98, teamchat 34, spreadsheet 22, photo-studio 20, documents 15, calendar 14, signatures 11, childcare 10, fleet 9.
- legal 5, invoicing 5, wiki 4, survey 4, video-studio 3, vector-studio 2, property-management 2, field-service 2, api-client 2.
- 1 each in sudoku, solitaire, meetings, inventory, hiring, help-desk, email, contracts, contacts, chess, checkers and cap-table.
- Most are Playwright `page.route('https://js.puter.com/**', …)` calls.

**Build steps that break under npm workspaces.**
- **12 license gates.** Their `prebuild` runs `npm run check:licenses`, which reads `<app>/package-lock.json`.
  - Apps: api-client, audio-studio, circuits, documents, help-desk, music-studio, notes, parametric, records, vector-studio, video-studio, wiki.
  - audio-studio has an exception keyed `node_modules/@breezystack/lamejs`.
- **pdf.** Its `licenses` script (`scripts/notices.mjs`) and `vite.config.js` read app-relative `node_modules/…` paths for tesseract.js, tesseract.js-core, @tesseract.js-data/eng, pdfjs-dist, @pdf-lib/fontkit and tslib. Under hoisting these break; resolve them with `createRequire`.

## 7. Environment notes and gotchas (verified)

**Figma Make sandbox.**
- Workspace: `/workspaces/default/code`, a Make React scaffold. Keep it.
- `/tmp` is wiped between turns, so work in `./everything-local/`.
- About 4.2 GB disk free, 4 GB RAM, 6 CPUs, Node v22.23.4, pnpm.
- **npm is not installed.** Run it through pnpm: `pnpm dlx npm@10 …` or `pnpm dlx npm@11 …`. Alternatively run `node <npm-cli.js>` directly; it lives under `$HOME/.cache/pnpm/dlx/**/npm@<v>/node_modules/npm/bin/`.
- System Chromium is at `/usr/bin/chromium`. Pass it to Playwright as `executablePath`.
- A Figma Make agent must end every turn with `figma make versions create --commit -m "…" -- <changed workspace paths>`.

**GitHub.**
- The Make GitHub connector is signed in as yaya0631. It can read the private forks, create repos, and push small sets of files.
- The `gh` CLI token is invalid, and anonymous git cannot read private repos.
- Plain git push needs `GITHUB_PUSH_TOKEN`. On a normal PC, clone and push as usual.

**npm.**
- npm 10.9.9 crashes on a resolve without a lockfile: `Cannot read properties of null (reading 'edgesOut')`. The minimal trigger is wiki's `package.json` on its own.
- npm 11.21.1 resolves the full workspace (`apps/*`, `!apps/assets`) in about 17 s: about 1,460 packages and an 860 KB lockfile.
- With that lockfile committed, npm 10 `install` and `ci` both succeed. Generate the lockfile with npm 11; never commit a `--legacy-peer-deps` tree.
- Negated workspace globs such as `"!apps/assets"` work.
- `npm test --workspaces --if-present` needs no root script.
- No installed package depends on any workspace name, including `platform`. Name the platform workspace `everything-local-platform` anyway.
- `fflate@0.8.3` hoists to the root `node_modules`. `patch-package` 8.0.1 then throws "not present at node_modules/fflate" when run from `apps/presentation`.
- `node:sqlite` works on Node 22.23, with an `ExperimentalWarning`.

**License-closure walk (tested).** It gives about 155 packages for notes, against 117 in its old lockfile; skipping optional peers narrows the gap.
1. Read the root lockfile and start at `packages['apps/<app>']`, taking its `dependencies` and `devDependencies`.
2. Resolve each name Node-style from the dependent's location upward: `<loc>/node_modules/<n>`, then each parent folder, up to the root.
3. Recurse into `dependencies`, `optionalDependencies` and non-optional `peerDependencies`. Skip peers that `peerDependenciesMeta` marks optional.
4. Every lockfile entry carries a `license` field.

## 8. Platform specification (to build)

**Files.** Each server file has its test beside it.
- `platform/server/`: `index.js`, `auth.js`, `fs.js`, `kv.js`, `events.js`, `workers.js`, `mail.js`, `hosting.js`, `ai.js`, each with a `<name>.test.js`.
  - `apps.test.js` holds the browser acceptance tests and the offline sweep.
- `platform/sdk/`: `index.js`, `http.js`, `auth.js`, `fs.js`, `kv.js`, `events.js`, `workers.js`, `email.js`, `hosting.js`, `perms.js`, `ui.js`, `dialogs.js`, `ai.js`, `net.js`, `unsupported.js`, `vite.config.js` (lib mode, IIFE, unminified) and `puter.test.js`. It builds to `sdk/dist/puter.js`, which git ignores.

**Root `package.json`:**

```json
{ "name": "everything-local", "private": true, "type": "module",
  "engines": { "node": ">=22.18" },
  "workspaces": ["apps/*", "!apps/assets", "platform"],
  "scripts": {
    "dev": "node --watch platform/server/index.js --dev",
    "build": "npm run build --workspaces --if-present",
    "start": "node platform/server/index.js",
    "postinstall": "patch-package --patch-dir apps/presentation/patches" } }
```

**`platform/package.json`.**
- Name `everything-local-platform`, `private`, `"type": "module"`.
- Scripts: `build` runs `vite build --config sdk/vite.config.js`; `test` runs `node --test`.
- `dependencies`: `vite` 8.3.4, matching the apps.
- `devDependencies`: `playwright-core`, matching the apps' Playwright 1.63/1.64.

**`.gitignore`:** the original root file plus `platform/data/`.

**`index.js`.**
- **Startup:**
  - Open the SQLite database and create the data folders.
  - Route requests by `Host`, listening on 127.0.0.1:4400.
  - Convert each Node request to a Web `Request` and its `Response` back. Handlers are `(Request, ctx) => Response`, so workers and server code can call the API in-process; the server never needs DNS for `*.localhost`.
- **Static serving:**
  - MIME map including `application/wasm`, and Range support.
  - `/assets/*` gets `public, max-age=31536000, immutable`; everything else gets `no-cache`.
  - A directory serves `index.html`. A miss without a file extension falls back to `index.html`; any other miss is a 404.
- **CSP** on every app and site response:
  `default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval' 'wasm-unsafe-eval' blob:; style-src 'self' 'unsafe-inline' blob:; img-src 'self' data: blob: http://*.localhost:4400; media-src 'self' data: blob: http://*.localhost:4400; font-src 'self' data: blob:; connect-src 'self' data: blob: http://localhost:4400 http://*.localhost:4400; worker-src 'self' blob:; frame-src 'self' blob: data: http://localhost:4400 http://*.localhost:4400; object-src 'none'; base-uri 'self'`
  - Dev mode adds `ws://localhost:* ws://*.localhost:*` for HMR.
- **At start:**
  - Build the SDK if it is missing.
  - Deploy the 8 developer workers.
  - Run the build queue: every app without `dist/` runs `npm run build -w apps/<x>` (spawned with `shell: true`, two at a time). Each app is served as soon as it is built; an app opened while waiting jumps the queue and shows a "Building…" page.
  - Build logs go to `data/logs/<app>.log`.
- **Dev mode (`--dev`):**
  - Each app is served lazily through its own Vite version, imported from the app's own folder (with `createRequire`), in `middlewareMode`.
  - HMR runs on a per-app port.
  - If the app has a `build:worker` script, run it first.
  - This replaces `dev/puter-localhost.mjs`.
- **Launcher at `/`:**
  - Shows an app grid using each built app's `<title>` and `icon-128.png`.
  - Shows the current account with a way to switch.
  - Links to the Email app as the mailbox, and shows the data path and AI status.
  - Styling: `system-ui` font, light and dark modes, clean and minimal.

**`auth.js`.**
- **Accounts:**
  - First start creates an owner account from `os.userInfo().username`, sanitised (fallback `owner`), with email `<user>@localhost`.
  - A `system` account that cannot sign in owns the developer workers.
- **Tokens:** JWT with HS256, signed with `data/secret.key`.
  - App token: `{type:'app-under-user', user_uid, app_uid, app, jti, iat}`.
  - Read token: `{type:'access-token', full_access:false, user_uid, uid, exp}`.
  - Session cookie on the launcher origin: `{type:'session', …}`.
  - The `tokens` table makes app and session tokens revocable.
- **Sign-in flow:**
  1. `signIn()` opens a popup at `http://localhost:4400/signin?app=<name>`.
  2. The window shows "Continue as <account>" and "Use another account / Create account".
  3. It mints an app token and `postMessage`s it to the verified app origin only (`^http://[a-z0-9-]+(\.site)?\.localhost:4400$`), then closes.
  4. The SDK stores the token in `localStorage`.
- The API checks that each request's `Origin` matches the token's app.
- Also implements `getUser`, `whoami` and an SVG-initials `getProfilePicture`.
- **perms** (single-machine policy):
  - Requests from the same user are auto-granted and recorded.
  - `request('email')` returns the account's email.
  - `check` reports recorded grants.
  - KV-share and appData requests follow Puter's docs.
  - The README states this trust model.

**`fs.js`.**
- Files on disk at `data/files/<user>/…`, with Puter-style paths. Relative paths resolve to `~/AppData/<app_uid>/`, `~` is the home folder, and absolute paths are `/<user>/…`.
- SQLite keeps uids, shares and read tokens.
- Path traversal is blocked, and other users' paths require a share.
- **Methods:**
  - `write` (`overwrite`, `dedupeName`, `createMissingParents`), `read` (returns a Blob), `mkdir`, `readdir`, `stat`, `delete`, `rename`, `move`, `copy`, `space`.
  - Sharing: `share`, `unshare`, `listShared`, `getShares`, `getShareLink`.
  - Read URLs: `getReadURL` returns `${APIOrigin}/token-read?token=…`; `revokeReadURL` revokes it.
- `FSItem` fields follow Puter's docs (`Objects/fsitem.md`).
- Every mutation emits an event.

**`kv.js`.**
- One namespace per (user, app), with a TTL column.
- **Methods:**
  - `set` (accepts an options argument), `get`, `del`, `list` (`prefix*` patterns, `returnValues`).
  - `incr` and `decr`, including the `{path: n}` form; `add` (appends to arrays); `update` (dotted-path merge); `remove`; `expire`; `expireAt`; `flush`.
  - `MAX_KEY_SIZE` and `MAX_VALUE_SIZE` follow the docs.
  - KV sharing: `share`, `unshare`, `listShared`, `listSharedByMe`.
- Every mutation emits an event.

**`events.js`.**
- **Subscriptions:** `onLocal` subjects `fs:<path>`, `kv:<key|prefix*>` and `kv:<handle>:<pattern>`, with optional `includeValue`.
- **Streams:** one SSE stream per client token (`GET /events/stream`, read through `fetch` with an `Authorization` header), with heartbeats.
  - Subscribe and unsubscribe are separate POSTs.
  - On reconnect the client gets `{op:'gap'}` and re-subscribes.
- **KV handles:** `POST`, `DELETE` and `GET /events/kv-handles` grant another user watch and read access to the owner's (user, app) prefix.
- `events.list()` returns `{items: []}`. `onPersistent` and handler workers reject as unsupported.

**`workers.js`.**
- **Deploy:**
  - `create(name, filePath)` reads the source from the user's fs and stores it in the `workers` table: name, owner, app, file and source hash.
  - Names are unique across users; redeploying the same owner's name replaces it.
  - The worker is ready before the call resolves, and returns `url: http://<name>.work.localhost:4400`.
  - `list`, `get` and `delete` follow `Objects/workerinfo.md`.
- **Isolation:** one `worker_threads` thread per worker. The source runs in a `vm` context whose only globals are:
  - `router`: `get`, `post`, `put`, `delete`, `patch`, `options`; `:param` and named `*wildcard` routes; automatic CORS `*` with OPTIONS answered. A returned object becomes JSON; a returned `Response` passes through.
  - `me`.
  - `fetch` (local URLs only, dispatched in-process), `Request`, `Response`, `Headers`, `URL`, `crypto`, `TextEncoder`/`TextDecoder`, `atob`/`btoa`, timers, `AbortController`, `structuredClone`, `console`.
- **Limits:** 30 s per request and a memory cap. A thread that exceeds them is restarted.
- **`user`** is built from a valid `puter-auth` or Bearer token.
- **`me.puter` and `user.puter`** are RPC objects that call the main thread under the right identity. They support kv, fs, email (`send`, `sendTransactional`, `list`, `get`), `auth.getUser`, `authToken` and `APIOrigin`. Anything else throws `not_available_locally`.
- **Developer workers:** the 8 listed in §6 are deployed at every start from `apps/<app>/<file>`, owned by `system` and running as their app.

**`mail.js`.**
- `send` and `sendTransactional` take `{to, cc, bcc, subject, html, text, replyTo, attachments}` and build RFC822.
- `POST /email/send` accepts raw `message/rfc822` from the Email app.
- **Delivery:** a recipient whose address matches a local account's email gets the message at `/<user>/.mail/objects/<day>/<uuidv7>--<b64url subject>`, which emits an fs event. Any other recipient produces a bounce in the sender's inbox.
- `list` and `get` read `objects/` and `sent/`, with shapes from `Objects/email*.md`.
- Developer workers send from `<app>@localhost`.

**`hosting.js`.**
- `create(subdomain, dirPath)`, `delete`, `list`, `get` and `update`. Names match `[a-z0-9-]{1,63}` and must be unique.
- Each site is served read-only at `<sub>.site.localhost:4400`, with the CSP and no cookies.

**`ai.js`.**
- **Interface:** `{available(), chat({messages, tools, model, stream, temperature, max_tokens})}`.
- **The OpenAI-compatible adapter:**
  - Streams by translating SSE.
  - Passes tools and `tool_calls` through.
  - Maps any model name to `AI_MODEL`.
  - Accepts only loopback `AI_URL` values.
- **Endpoints:** `POST /ai/chat` (JSON, or SSE when streaming). The other modalities return 501 `not_available_locally`; no reachable server returns `ai_unavailable`.

**SQLite tables:** accounts, tokens, apps (name→`app-<uuid>`), kv, kv_shares, kv_handles, fs_nodes, fs_shares, read_tokens, workers, sites, perms.

**SDK (`window.puter`, one IIFE file).**
- **Behaviour spec:** Puter's docs, completed by running each app against the SDK.
- **Properties:** `env='web'`, `APIOrigin`, `authToken`, `appID`, `args` (from the URL) and `print`.
- **auth:** `isSignedIn()` is synchronous.
- **fs:** all methods, plus the `FSItem` class with its methods.
- **kv**, **events**, **email** and **hosting:** as specified above.
- **workers:** `exec` is `fetch` plus a `puter-auth` header.
- **perms:** includes the `Resource` enum.
- **ai:** `chat` handles every documented call form, including streaming as an async iterable, tools and images.
- **net:** `fetch` works only for local URLs; `Socket` and `tls` throw.
- **peer:** every method rejects with `not_available_locally`.
- **ui** (rendered in a closed Shadow DOM):
  - Open, save and directory pickers over the user's files: breadcrumbs, "New folder", keyboard support, light and dark.
  - `alert`, `prompt`, and `contextMenu` at the pointer.
  - `launchApp`: a new tab, or a hidden iframe for background launches, returning an AppConnection over `postMessage`. `parentApp` is its other end.
  - `onLaunchedWithItems` and `onItemsOpened` read `?puter.items=` from the URL.
  - `setWindowTitle` sets `document.title`. `showWindow` and `hideWindow` resolve without doing anything.
  - `onWindowClose` triggers the browser's leave-page prompt.
  - Picture-in-picture uses the browser API. `requestPermission` and `exit` follow the docs.
- **Errors:** always `{code, message}`.

**Tests.** Node's built-in test runner, with each test beside its file. Browser tests use `playwright-core` and skip themselves when no Chromium is found.

## 9. App change rules (only these; everything else stays byte-identical to b5d2584)

1. **Import** `src/*` into `apps/*`, except `apps/whiteboard/tmp/`. Copy the root `LICENSE` and `.gitignore` as they are.
2. **Delete cloud-only files:**
   - `amplify.yml`, `customHttp.yml` (root and 5 apps), `dev/`, `.github/`, and the root `assets/` folder (README images).
   - The 8 `deploy:*` scripts, calendar's `publish:mail-handler` and its `.mjs` file.
   - `public/sitemap.xml` files and `Sitemap:` lines.
3. **Workspace fixes:**
   - Delete the 51 per-app lockfiles.
   - Point the 12 license checks at the root lockfile closure. Normalise CRLF before hashing in documents and help-desk.
   - pdf: resolve package paths with `createRequire`.
   - presentation: remove its `postinstall`.
   - 50 vite configs: remove the `puter-localhost` import and plugin.
4. **SDK URL switch:**
   - `<script src="https://js.puter.com/v2/">` becomes `<script src="/puter.js">`, keeping the other attributes.
   - JS loader constants become `'/puter.js'`.
   - The 7 service workers keep network-first for `new URL('/puter.js', self.location)`.
5. **Puter links, metadata and analytics:**
   - Remove the canonical, `og:url`, `og:image`, `twitter:image` and JSON-LD tags that point at Puter.
   - Remove "Built on Puter" anchors but keep the layout.
   - `puter.com/app/<x>` and `<x>.puter.com` links become `http://<x>.localhost:4400/`.
   - Remove the Plausible loader and stub. cap-table stops injecting analytics, and `site/analytics.html` is deleted. The `analytics.js` wrappers stay as inert no-ops.
   - In app READMEs and `STORE_LISTING.md`, make demo links local and drop Puter deploy instructions.
6. **Worker and site URLs** (22 apps, about 130 lines; hand edits with their tests):
   - `https://X.puter.work` becomes `http://X.work.localhost:4400`.
   - `https://X.puter.site` becomes `http://X.site.localhost:4400`.
   - Validators check the matching hostname suffix with the `http:` scheme.
   - Rebuild the committed `public/worker/*` bundles with each app's `build:worker`.
7. **External services:**
   - **3d:** a few lines in `vite.config.ts` copy the Pyodide core from `node_modules/pyodide` into `dist/pyodide/`, and `PYODIDE_INDEX_URL` becomes `./pyodide/`.
   - **fleet:** the NHTSA lookups reject with "not available offline".
   - **whiteboard:** Miro import rejects the same way. Google, Slack and embeds stay in the code but are blocked by `net.fetch`, the worker `fetch` and the CSP.
   - **cap-table and inventory:** bundle or remove anything loaded from external sites. Plain links stay.
8. **Tests:**
   - Playwright routes `https://js.puter.com/**` become `**/puter.js`.
   - Update assertions about hosts, Plausible and the SDK tag (cap-table public-content, fleet service worker, and the URL-shape tests in the 22 apps).
   - Never delete a test.

## 10. Steps (small commits pushed to everything-local; summarise each one to the owner)

1. **Environment.**
   - Done: the repo exists and the Make workspace is prepared.
   - If a push token is available, add the remote using it without echoing it. Otherwise produce a bundle at each step.
2. **Import the originals:**

   ```sh
   git init -q /tmp/up && cd /tmp/up && git remote add origin https://github.com/HeyPuter/everything.git
   git fetch -q --depth 1 origin b5d25841527e2b772eb34a80d4973c6bc1dc67c1 && git checkout -q FETCH_HEAD
   # in everything-local: copy src/* to apps/ excluding whiteboard/tmp/, then copy LICENSE and .gitignore
   ```

   Commit "Import apps from HeyPuter/everything@b5d2584". Then check that `git ls-tree HEAD apps/` matches `git -C /tmp/up ls-tree HEAD src/` for every app except whiteboard.
3. **Remove cloud-only files** (§9.2), in one commit.
4. **Set up the workspace** (§9.3).
   - Generate the lockfile with npm 11 (`--package-lock-only`), then verify `npm ci` with npm 10.
   - **Record a baseline:** build all 50 apps, then run every unit suite (Vitest or `node --test`), one app at a time. Record pass or fail per test file.
   - If an app fails, rerun it standalone with its original lockfile to tell pre-existing failures from ones the workspace caused. Fix only the workspace-caused ones, narrowly.
   - Clean the npm cache afterwards to save disk.
5. **Platform core**, one commit per piece with its tests: `index.js`, `auth.js`, `kv.js`, `fs.js`, `events.js`, then the SDK core and its build.
6. **SDK URL switch, Puter links and analytics** (§9.4–9.5) across all apps, with test updates.
   - Rebuild, rerun the suites against the baseline, and do a first offline sweep.
7. **Workers, developer workers, mail and hosting.** Then the §9.6 URL edits in per-app commits, rebuilding the worker bundles.
8. **UI pieces:** pickers, dialogs, `launchApp`/AppConnection, window functions, and the rest of perms.
9. **AI adapter and external services** (§9.7), plus the degrade checks.
10. **Verification, README and report:**
    - Run §11.
    - Write the README (§12 outline).
    - Fill in the report (§14).
    - Delete `HANDOFF.md`.

## 11. Verification

- **Checklist 1:** in a fresh clone, `npm ci` then `npm start` (npm 10.9, Node 22.18+). The log must show all 50 apps built and served.
- **Checklists 2, 3 and 7:**
  - The top-level listing must be exactly the allowed set.
  - A one-off command (not committed) compares the `apps/<x>` file lists with `b5d2584:src/<x>`. It must find zero added files, and the deletions are listed in the report.
- **Checklist 4**, in `platform/server/apps.test.js`:
  - Use Playwright with the system Chromium, `--host-resolver-rules` blocking all non-local DNS, and every non-local request recorded and aborted.
  - Flows:
    - notes (fs + kv).
    - spreadsheet: save, then share by email with a second account and see the mail in that account's Email app.
    - teamchat: account A creates a workspace, B joins from a second browser context, and live messages flow both ways.
    - video-studio: save a project, and the AI action shows its "not available" message.
    - chess: sign in, and online mode shows "not available".
  - Restart the server, then check the data persisted.
- **Checklist 5:**
  - The sweep opens every app and signs in. It must record zero external requests and no new page errors compared with the baseline.
  - Review the brief's grep. There must be zero Puter or analytics hosts, apart from the DOCX ink namespace. Classify the remaining `https://` lines as outbound links, URL recognizers, XML namespaces, comments or license text.
- **Checklist 6:**
  - `platform/sdk/puter.test.js` runs in a browser and calls every listed method plus the extras, checking result shapes and error codes.
  - `server/*.test.js` cover each service.
  - **Degrade checks:**
    - chess, backgammon and checkers online.
    - meetings join and the teamchat huddle.
    - AI generation in video-studio, documents and photo-studio.
    - fleet VIN lookup and 3d Python.
    - whiteboard Miro import and embeds.
    - spreadsheet and wiki web fetches.
- **Test suites:**
  - `npm test --workspaces --if-present` must equal the baseline.
  - The Playwright e2e suites (pdf, presentation, documents, 3d, email, parametric, building-design, photo-studio, api-client) are updated mechanically and spot-run. Their full runs take hours each and are manual in the original too. Report exactly which ones ran.

## 12. README outline (under 60 lines) and the "not supported locally" list

**README sections:**
1. What it is.
2. Requirements: Node 22.18+, npm 10.9+, Chrome, Edge or Firefox.
3. Running it:
   - `npm install`, then `npm start`. The first start builds every app once.
   - Open http://localhost:4400.
   - `npm run dev` for live editing.
   - `npm run build` to rebuild everything.
   - `npm test --workspaces --if-present` to test.
4. Data: `platform/data/` holds `platform.db`, `files/` and `logs/`.
5. Backups: stop the server and copy `platform/data/`.
6. AI: install Ollama and run `ollama pull llama3.2`, or set `AI_URL` and `AI_MODEL` (loopback only).
7. The port is fixed at 4400; explain why and how to change it.
8. Not supported locally (list below).
9. Licenses: MIT at the root; some apps carry their own, for example notes is AGPL-3.0.

**Not supported locally** (error code `not_available_locally`):
- `puter.peer`: online play in chess, backgammon and checkers; all of Meetings; teamchat huddles; live presence in spreadsheet, vector-studio, photo-studio and whiteboard.
- Persistent event handlers: calendar's mail-reply intake.
- `net.fetch` to the internet, `net.Socket` and `tls`.
- Mail to addresses outside this PC (bounced locally).
- AI image, video and speech generation, transcription, and image-to-text.
- NHTSA lookups (fleet); Miro, Slack, Google and web embeds (whiteboard); Pyodide packages outside its core distribution (3d).
- Access from other machines (`*.localhost` resolves only on this PC).
- The `assets` placeholder app.

## 13. Risks

- **Sandbox limits:** builds run one at a time, the npm cache is cleaned, and installs fall back to `npm ci -w` batches when needed.
- **npm 10 without a lockfile:** the committed lockfile avoids the crash. Tell contributors to use npm 11+ when changing dependencies.
- **Collab protocol details** only show up once the apps run against the SDK. Report any remaining gaps honestly.
- **The `fflate` patch** also affects circuits and documents, so both suites must stay green.
- **Windows:**
  - Use no shell scripts and spawn npm with `shell: true`.
  - CRLF-safe hashing covers the two known checks.
  - Testing here is Linux only.
- **The push token may never reach the sandbox:** fall back to `everything-local.bundle`.

## 14. Final report template (send to the owner)

- **Acceptance checklist 1–7:** pass or fail for each, with evidence (commands and outputs).
- **Test suites:** before and after per app; which e2e suites ran.
- **Grep review summary and offline sweep result.**
- **Every deviation from the brief, with its reason:** `events.js`, `postinstall`, deleted cloud files and lockfiles, whiteboard `tmp/`, license-check edits, pdf and presentation fixes, the Node 22.18 requirement, and the fixed port.
- **Unsupported list,** and anything that is known to be partial.
