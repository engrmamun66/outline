# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Coding conventions (TypeScript rules, JSDoc, class member order, security, testing rules) live in AGENTS.md and must be followed:

@AGENTS.md

## Commands

```bash
make up                          # start postgres + redis (docker), install local SSL, yarn install, run dev:watch
yarn dev:watch                   # backend (nodemon rebuild) + Vite frontend dev server
yarn build                       # full production build (vite + i18n + server via build.js)
yarn start --services=web,worker # run built server with selected services

yarn lint                        # oxlint --type-aware over app/server/shared/plugins
yarn format                      # oxfmt
yarn tsc                         # type check

yarn db:create-migration --name my-migration
yarn db:migrate                  # add `--env test` to migrate the test DB
yarn db:rollback
```

Tests (Vitest, `TZ=UTC`). Server tests need Postgres and a migrated test DB; `make test` creates it (drop/create/migrate with `NODE_ENV=test`) and then runs everything.

```bash
yarn test path/to/file.test.ts           # single file (preferred)
yarn test path/to/file.test.ts --watch
yarn test:server | test:app | test:shared # whole suites (avoid)
```

Vitest projects (see `vitest.config.ts`): `server-shared` runs server + plugin tests with `isolate: false` (shared module registry); files whose first 512 bytes contain `@vitest-isolate true` run in the isolated `server` project instead — required when using `vi.mock`/`vi.resetModules`. `app` and `shared-jsdom` use jsdom; `shared-node` uses node.

Dev URL is `https://local.outline.dev:3000` (`.env.development`). Logging: `DEBUG=http` or `DEBUG=*`, `LOG_LEVEL=debug|silly`.

Path aliases: `@server/*`, `@shared/*`, `~/*` (app), `plugins/*`.

## Architecture

See `docs/ARCHITECTURE.md` (directory layout) and `docs/SERVICES.md`.

### Server process and services

`server/index.ts` boots one process running the services selected by `--services` / `SERVICES` (`server/services/`): `web` (app + API), `websockets`, `worker` (queue processors), `collaboration` (Hocuspocus/Y.js realtime editing), `cron`, `admin` (queue UI at `/admin`, dev only). `yarn dev` runs all of them in one process from `build/server`, so backend changes require the build step (`dev:backend` handles this with nodemon).

### API request flow

RPC-style: every endpoint is `POST /api/<resource>.<action>` (e.g. `documents.info`). Each resource folder in `server/routes/api/<resource>/` has `<resource>.ts` (Koa router), `schema.ts` (zod schemas extending `BaseSchema`, exporting `XxxSchema` + `XxxReq` types), and `<resource>.test.ts`. Typical handler chain:

```ts
router.post("documents.info", auth(), validate(T.DocumentsInfoSchema), transaction(), async (ctx: APIContext<T.DocumentsInfoReq>) => {
  const { id } = ctx.input.body;         // validated input
  const { user } = ctx.state.auth;
  authorize(user, "read", document);     // cancan policy from server/policies
  ctx.body = { data: presentDocument(ctx, document), policies: presentPolicies(user, [document]) };
});
```

- Multi-model business logic goes in `server/commands/` (e.g. `documentCreator`, `documentMover`), not routes.
- Responses are shaped by `server/presenters/`; responses include a `policies` array that the frontend stores in `PoliciesStore` to drive UI permissions.
- Errors use factories from `server/errors.ts` (`NotFoundError`, `AuthorizationError`, …).

### Events and background work

Models extend `server/models/base/Model.ts`. Using `saveWithCtx` / `createWithCtx` / `destroyWithCtx` (passing the API ctx) automatically persists an `Event` row and publishes it to the global event queue (Redis + Bull, `server/queues`). Processors in `server/queues/processors/` extend `BaseProcessor`, declare `static applicableEvents`, and implement `perform(event)` in the worker. Arbitrary async jobs (not event-driven) are tasks in `server/queues/tasks/`. Side effects like notifications, search indexing, emails, websocket broadcasts, and webhooks all hang off this event bus — prefer emitting events over calling side effects inline.

### Collaboration

`server/collaboration/` contains Hocuspocus extensions (authentication, persistence to `Document.state`, connection limits, views, awareness). The editor itself is ProseMirror in `shared/editor/` (nodes, marks, extensions shared by client and server for Markdown/HTML conversion).

### Plugins

Each folder in `plugins/` has `plugin.json` (id, name, priority) plus optional `server/`, `client/`, `shared/`. Server side registers with `PluginManager.add([...])` from `@server/utils/PluginManager` using `Hook` types (`API`, `AuthProvider`, `Processor`, `Task`, `SearchProvider`, `UnfurlProvider`, `IssueProvider`, `EmailTemplate`, …), usually gated on its env vars being set. Client side uses `app/utils/PluginManager.ts`. Auth providers (google, slack, oidc, azure…) and integrations are all plugins.

### Frontend

React + MobX. `app/stores/RootStore.ts` wires all stores; stores extend `app/stores/base/Store.ts` which provides `fetch`/`fetchPage`/`save`/`delete` against the RPC API via `app/utils/ApiClient.ts` and hydrates models in `app/models/`. Pages live in `app/scenes/`, routes in `app/routes/` (lazy loaded). Reusable commands (menus, command bar, shortcuts) are defined in `app/actions/`.

### Tests (server)

Use factories in `server/test/factories.ts` (`buildUser`, `buildDocument`, `buildCollection`, …) and `getTestServer()` from `server/test/support.ts` for API tests (`server.post("/api/documents.info", user, { body: { id } })` — omit `user` for unauthenticated requests). External HTTP is mocked via MSW (`server/test/msw.ts`).

### i18n

Wrap UI strings with `t()` from react-i18next; `yarn build:i18n` extracts keys into `shared/i18n/locales/en_US/translation.json`. Do not edit translation files manually.
