# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Web Development
```bash
yarn          # Install dependencies
yarn watch    # Start dev server (shadow-cljs + CSS watch) → http://localhost:3001
yarn release  # Production build → static/
```

### Desktop Development
```bash
yarn watch              # Must be running first
yarn dev-electron-app   # Launch Electron dev app
yarn release-electron   # Build packaged desktop app
```

### Testing & Linting
```bash
bb dev:lint-and-test                    # Run linters + all unit tests
bb dev:test -v <namespace/test-name>    # Run a single test (e.g. bb dev:test -v logseq.some-test/foo)
bb dev:e2e-basic-test                   # Run E2E tests
yarn cljs:lint                          # Lint with clj-kondo only
```

### Other Useful Commands
```bash
bb lint:worker-and-frontend-separate  # Validate architectural boundaries
bb lint:carve                         # Find unused code
bb dev:db-sync-start                  # Start local db-sync server
```

### REPL (with `yarn watch` running on port 8701)
- **VSCode/Calva**: Connect to nREPL → `shadow-cljs` → `:app`
- **Emacs/Cider**: `(shadow.user/worker-repl)` for web-worker context
- **IntelliJ/Cursive**: Remote nREPL → run `(shadow.cljs.devtools.api/repl :app)` in REPL

## Architecture Overview

Logseq is a multi-platform knowledge management app (web, desktop via Electron, mobile via Capacitor). The core pattern is an **event-driven GUI with an immutable in-memory database and reactive UI**.

### Technology Stack
- **ClojureScript** (primary language) compiled via **shadow-cljs**
- **React 18 + Rum** for reactive UI components
- **DataScript** (Logseq's fork) — in-memory Datalog database for all graph state (pages, blocks, properties)
- **SQLite** (via WASM / better-sqlite3) — persistent on-disk storage
- **TailwindCSS + Shadcn** — styling
- **Babashka** — build/task scripts via `bb.edn`

### shadow-cljs Build Targets
- `:app` — main browser/web app
- `:db-worker` — database operations (background web worker)
- `:inference-worker` — AI/HuggingFace transformers (background web worker)
- `:electron` — Electron main process
- `:mobile` — Capacitor mobile app
- `:publishing` — static site export

### Key Source Directories
| Path | Purpose |
|------|---------|
| `src/main/frontend/` | Core app: components, handlers, DB queries |
| `src/main/frontend/components/` | React/Rum UI components (~92 dirs) |
| `src/main/frontend/handler/` | Event handlers (editor, DB, export, navigation) |
| `src/main/frontend/db/` | DataScript queries & schema |
| `src/main/frontend/worker/` | DB worker + RTC (real-time collaboration) |
| `src/main/frontend/worker/rtc/` | Real-time collaboration protocol implementation |
| `src/main/frontend/extensions/` | PDF, video, graph viz, spaced repetition |
| `src/main/mobile/` | Mobile-specific code |
| `src/electron/` | Electron main process code |
| `src/test/` | ClojureScript unit tests |
| `deps/` | Internal libraries (see below) |
| `clj-e2e/` | End-to-end Playwright tests |

### Internal Libraries (`deps/`)
- **`common`** — shared utilities across platforms
- **`db`** — DataScript schema, DB property definitions
- **`db-sync`** — real-time collaboration server & client
- **`graph-parser`** — Markdown/Org-mode → DataScript DB
- **`outliner`** — block tree operations (insert, move, delete)
- **`publishing`** — static site generator
- **`shui`** — internal component library

### Data Flow
1. **User action** → handler in `frontend/handler/`
2. **Handler** issues DataScript transactions → `frontend/db/`
3. **DB worker** persists changes to SQLite and syncs via db-sync
4. **Rum reactive atoms** trigger re-renders in `frontend/components/`

### Properties System
- Built-in properties live in `logseq.db.frontend.property/built-in-properties`
- Add new properties there; avoid creating new classes or properties unless necessary
- ClojureScript keywords must be defined via `logseq.common.defkeywords/defkeyword`; reuse existing keywords

## Coding Conventions

- **Keyword naming**: Use `-` not `_` in map keys (`:user-id` not `:user_id`)
- **Shadow variables**: Avoid shadowing common names — e.g., name a bytes buffer `payload` not `bytes`
- **Commit style**: Short imperative subjects with optional scope prefix — `fix: download`, `enhance(rtc): ...`
- **PR title prefixes**: `chore:`, `dev:`, `enhance:`, `feat:`, `fix:`, `test:`

## Agent-Specific Notes

- Review guidelines before major changes: `prompts/review.md`
- DB-sync feature guide: `docs/agent-guide/db-sync/db-sync-guide.md`
- DB-sync protocol reference: `docs/agent-guide/db-sync/protocol.md`
- Architecture boundary linting enforces that worker and frontend code stay separate — run `bb lint:worker-and-frontend-separate` to validate

---

## Custom Features (this fork)

### Bullet Threading (native integration)

Ports `logseq-plugin-bullet-threading` into core as a default-on feature with a settings toggle.

**Files:**
| File | Role |
|------|------|
| `src/main/frontend/components/bullet-threading.css` | All visual CSS, scoped under `.is-bullet-threading` |
| `src/main/frontend/components/container.cljs` | Event listeners + root class binding |
| `src/main/frontend/state.cljs` | `:ui/bullet-threading?` state + `bullet-threading?` / `toggle-bullet-threading!` fns |
| `src/main/frontend/components/settings.cljs` | Toggle row in Editor settings panel |
| `src/resources/dicts/en.edn` | `"Bullet threading"` |
| `src/resources/dicts/zh-cn.edn` | `"子弹连接线"` |
| `src/resources/dicts/zh-hant.edn` | `"項目連接線"` |

**How it works:**

1. `container.cljs` binds class `.is-bullet-threading` to `#app-container-wrapper` when the feature is enabled.
2. On `focusin`, `update-bullet-threading!` walks up the DOM from the focused `.ls-block`, and for each ancestor `.block-children-container` sets:
   - `data-thread-active` attribute
   - `--thread-blue-height` CSS custom property = distance from container top to the arc start point of the focused block (`block-control-wrap.top − block-control-wrap.height / 2`)
3. `bullet-threading.css` renders the blue segment via `.block-children-container[data-thread-active]::before { height: var(--thread-blue-height) }`. This element lives inside the **parent** block's compositor layer, avoiding the `will-change: transform` clipping issue that affects child `.ls-block` pseudo-elements.
4. On `focusout` (only when `relatedTarget === null`, i.e. focus leaves the document), highlights are cleared.

**Key CSS design decisions:**
- CSS variables (`--ls-block-bullet-active-color`, `--ls-block-bullet-threading-width`) are scoped inside `.is-bullet-threading`, not `:root`
- The arc connecting each child block to the guide line uses `block-control-wrap::before` (blue only when `:focus-within`)
- The parent bullet's descending line uses `block-content-wrapper::before` (blue when `:focus-within`)
- Doc-mode and embed-mode disable threading via explicit `display: none` overrides

**Settings UI:** Settings → Editor → "Bullet threading" toggle (persisted to localStorage)

---

### Release Workflow

`.github/workflows/build-release.yml` — manually triggered (`workflow_dispatch`).

Builds: macOS arm64 (Apple Silicon), macOS x64 (Intel), Linux x64. No code-signing secrets required.

**To trigger:** GitHub → Actions → Build-Release → Run workflow, or:
```bash
gh workflow run build-release.yml --repo ericwangzq/logseq
```

Output: a rolling `latest` GitHub Release with `.dmg`, `.AppImage`, and `.zip` files.
Version format: `<base-version>+<YYYYMMDD>.<short-sha>`
