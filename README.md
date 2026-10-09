# WebDev Lab

An offline-first, installable PWA for learning **HTML, CSS, JavaScript and SQL/MySQL** by predicting, experimenting,
breaking, debugging and building real code. No account, no backend: everything runs in the browser and is stored on the device.

## Features

- 4 learning paths, 75+ lessons (13 HTML, 13 CSS, 32 JavaScript, 18 SQL) grouped into modules, each with a plain-English hook,
  worked examples, visualizers, quizzes and key concepts. Many open with a concrete problem.
- **Shared learning loop** (Understand, See, Predict, Experiment, Break, Debug, Practice, Challenge, Build, Explain, Review)
  used across all four paths through the same block types (see "Content architecture").
- Real code execution: sandboxed iframe for HTML/CSS/JS, Web Workers for pure JavaScript, SQLite-in-WebAssembly for SQL.
- Playground (multi-file, console, DOM inspector, debug panel), 66 challenges, 16 projects checked by behavioural tests.
- MySQL Lab: databases, tables, editable rows, schema and relationship views, query visualizer. **A simulator, not MySQL.**
- Progressive "I'm stuck" ladder (Concept, Direction, Example, Next step, then solution), explanation step, spaced review (1, 3, 7, 21, 60 days).
- Progress only records actions that really happened (Viewed, Understood, Predicted, Experimented, Practiced, Debugged, Challenged, Built, Explained, Reviewed).
- Search (offline), bookmarks, themes (dark/light/system), export/import (validated JSON), command palette (Ctrl/Cmd+K).

## Stack

React 19, TypeScript (strict), Vite, React Router (hash routing, deploys anywhere), CodeMirror 6, IndexedDB (`idb`),
`sql.js` (SQLite WASM), `vite-plugin-pwa` (Workbox), Vitest, ESLint, Prettier. No DOMPurify: lesson text is rendered as React
nodes (never as HTML), and learner HTML only ever runs inside the sandbox.

## Development

```bash
npm install
npm run dev        # dev server
npm run build      # type-check + production build (+ service worker)
npm run preview    # serve the production build (service worker only works in the production build)
npm run test       # ~300 unit and integration tests
npm run lint
```

## Folder structure

```
src/
  app/            providers (settings, toast), config (versions), router (app.tsx)
  components/     layout, editor (CodeMirror), playground (workspace, preview, console, inspector, debug),
                  lessons (blocks, quiz, lesson view), challenges (runner), visualizers, mysql, search, common
  pages/          Home, Learn, Lesson, Practice, Playground, Projects, MySQL, Review, Bookmarks, Search, Settings
  content/        lessons per subject (typed data), builders.ts (helpers), enrich.ts (progressive migration + modules), coverage.ts
  data/           challenges, projects
  engine/         execution (sandbox document, bridge, worker, error explanations), javascript (module transform, visualizer tracing),
                  sql (engine, seeds, explain), validation (challenge + project checkers)
  services/       storage (IndexedDB, import/export), progress (store, stats), search, offline (service worker)
  hooks/ utils/ types/ styles/
e2e/              optional real-browser tests (see e2e/README.md)
```

## Content architecture

Lessons are data (`src/content/<subject>/*.ts`) built with helpers from `content/builders.ts`:

| Block | Purpose |
|---|---|
| `p`, `h`, `tip`, `warn`, `note` | text (inline `[[code]]` and `**bold**`; rendered safely) |
| `code`, `show`, `live` | runnable / static / multi-file interactive examples |
| `visual` | boxmodel, flexbox, grid, variables/arrays/objects, call stack, async order, logic, loops, SQL pipeline, SQL join, DOM tree |
| `sqlRun` | editable SQL on a throwaway copy of a sample database |
| `predictJs/Html/Css/Sql` | predict before running, then compare with the real result |
| `experiment`, `breakIt` | live editor with a task and a "what happened" reveal |
| `challengeRef` | embeds any challenge (Debug / Practice / Challenge) inline |
| `explain` | explain-it step saved locally |
| `mistakes` | common mistakes |

`content/enrich.ts` adds problems, predictions and challenges to older lessons without rewriting them, and assigns modules.
`content/coverage.ts` maps the requested JavaScript topic list onto lessons; `content.test.ts` fails if a mapped phrase is missing.

### Adding a lesson
Add `lesson('javascript', 33, { title, level, time, hook, problem, objectives, summary, keywords, prereqs, blocks, takeaways, quiz })`
to the subject file, assign a module in `MODULES` (`enrich.ts`). Search, routing, prerequisites and progress pick it up automatically.
JavaScript `predict` answers are checked against real execution by `npm test`.

### Adding a challenge
Add an object to `src/data/challenges/extra.ts`. Code challenges need `files`, `editable`, a `test` script (helpers: `$`, `$$`, `click`, `type`, `css`,
`wait`, `logs`, `assert`, `assertEqual`), a `solution`, `hints` (up to 3) and an `explanation`. Use `runner: 'worker'` for pure logic (terminated after 3s)
and `'iframe'` for DOM/CSS. SQL challenges give `db`, `solutionSql` and optionally `verifySql` (compares resulting table state). `npm test`
verifies that every solution passes and every starter fails.

### Adding a project
Add to `src/data/projects/extra.ts` with `starterFiles`, `milestones` (`files`, `web` test script, or `sql` file + verify query + expected rows) and a
`solution`. Web milestones run in order inside one document, so later milestones can build on earlier state.

### Adding SQL examples / sample data
Edit `src/engine/sql/seed.ts` (MySQL-style DDL, translated automatically) and use `sqlRun('school', '...')` in lessons.

## SQL: what it is and is not

The lab runs SQLite compiled to WebAssembly and translates common MySQL syntax (`AUTO_INCREMENT`, `LIMIT a, b`, `ENGINE=`, `SHOW`, `DESCRIBE`,
`USE`, `CREATE DATABASE`, `CONCAT`, `NOW`, ...). The supported list is shown in the app. Users, privileges, stored procedures, triggers and full-text search
are not available. FULL JOIN is taught as a concept; MySQL builds it with `UNION`.

## Offline and PWA

`vite-plugin-pwa` generates a Workbox service worker that precaches the app shell, every lesson chunk, the editor and the SQL WASM binary.
Update flow is user-controlled (`registerType: 'prompt'`): a banner offers "Update now"; user data lives in IndexedDB and is untouched.
IndexedDB schema is versioned (`DB_VERSION` in `services/storage/db.ts`); migrations only add stores or transform records. Content has its own
`CONTENT_VERSION`. Only code you write that calls outside servers needs the Internet.

## Security notes

- Learner code runs in an iframe with `sandbox="allow-scripts allow-forms allow-modals allow-popups"` (no `allow-same-origin`), so it cannot reach app storage, cookies or the parent.
  Messages are accepted only from the iframe's window and a per-run random token.
- `localStorage`/`sessionStorage` inside the sandbox are in-memory shims synced back to the app, because a null-origin frame has no real storage.
- Pure JavaScript challenges and visualizers run in terminated Web Workers. DOM challenges run in a hidden sandboxed iframe with a 4s timeout.
- A beginner loop guard stops `while`/`for` loops after 2,000,000 iterations in the playground. It only patches simple loop headers; a pathological
  loop elsewhere can still block the tab.
- Imported JSON is validated field by field; malformed rows are dropped.

## Known limits

- ES modules are simulated (import/export rewritten to a small registry). No circular imports, no `import()` or `export ... from`.
- No real breakpoints: the Debug panel explains errors; the lesson points to browser DevTools for breakpoints.
- Layout-dependent checks need a real browser; they were verified in headless Chromium (`e2e/`), not in jsdom.
- Spaced review uses each lesson's quiz questions; it measures recall, not skill. Challenges and projects are the skill check.
- The "Explain it" steps are saved and compared against a reference explanation by the learner. They are not graded automatically.

## Troubleshooting

- **No install prompt / no offline:** use the production build over HTTPS or `localhost` (`npm run build && npm run preview`).
- **Old version stuck:** accept the update banner, or unregister the service worker in DevTools, Application tab.
- **Editor does not load:** the CodeMirror chunk is lazy; check the network panel for a blocked `.js` request.
- **Data missing after a browser "clear site data":** export regularly from Settings.
- **Tests fail on `jsdom` types:** run `npm i -D @types/jsdom`.
