# Piano editoriale — node-tooling

Requisiti specifici dei capitoli di questa directory. Per le regole editoriali comuni e i criteri di accettazione, seguire [AGENTS.md](../../AGENTS.md).

## `08-node-npm.md` — Node.js e npm

**Da coprire:**
- Node.js vs JavaScript vs browser; V8, libuv, Node event loop e differenze con il browser.
- Event-driven/non-blocking I/O, worker pool, Worker Threads, Child Processes.
- `process`, environment variables, filesystem, EventEmitter, Buffer, Streams, backpressure.
- Node ESM vs CommonJS, package exports/imports (overview), error handling e logging.
- Backend basics: HTTP server, Express, middleware, routing, validation, REST, auth, CORS, rate limiting.
- npm vs npx, `package.json`, `package-lock.json`, `node_modules`, lifecycle e scripts.
- dependencies/devDependencies/peerDependencies/optionalDependencies.
- Semantic versioning (`^`, `~`, pin), `npm install` vs `npm ci`, npm audit, lockfile e sicurezza.
- npm vs pnpm vs Yarn, workspaces, LTS e versioni di runtime (verificare, non fissare informazioni obsolete).

**Confronti/snippet:** event loop Node/browser, Worker Threads vs Child Processes, `npm install` vs `npm ci`, dipendenze vs peer dependencies.

## `09-build-tools.md` — Bundler e Build Tools

**Da coprire:**
- Bundling, transpilation, compilation, minification, source maps; differenze fondamentali.
- Webpack: entry, output, loaders, plugins, code splitting, tree shaking.
- Vite, esbuild, Rollup, Babel; responsabilità, differenze e casi d'uso.
- ESM vs CJS nell'ecosistema degli strumenti.
- Dev server, Hot Module Replacement, development vs production builds.
- Static assets, environment configuration, performance budgets, caching, chunking.
- Framework CLI e bundler: separare la spiegazione generica dalla pipeline specifica di Angular.

**Confronti/snippet:** Webpack vs Vite vs esbuild, Babel vs TypeScript compiler, bundle vs transpile, dynamic import per lazy loading.
