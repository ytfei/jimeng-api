# CODEBUDDY.md

This file provides guidance to CodeBuddy Code when working with code in this repository.

## What this is

A Koa + TypeScript API gateway that mimics the public Jimeng (即梦) / Dreamina web app to expose image and video generation through OpenAI-style endpoints. There is no local model: every request is translated into signed calls against the Jimeng/Dreamina backend, submitted as a draft, then polled until the media URL is available. Requests are **synchronous** — a video call can stay open for tens of minutes.

## Commands

```bash
npm install
npm run dev          # tsup watch build + runs dist/index.js with source maps
npm run build        # tsup bundle to dist/ (cjs + esm + d.ts)
npm start            # node dist/index.js (expects a prior build)
npm run type-check   # tsc --noEmit, the closest thing to a test gate
npm run format       # prettier over src/**/*.{ts,js,json}
```

- There is **no test suite**; `npm test` is a stub that exits 1. Verify changes with `npm run type-check`, a build, then a live request against `/ping` and the affected endpoint.
- Docker: `docker compose up -d --build`, logs via `docker logs jimeng-api`.
- Published images are built by `.github/workflows/docker-publish.yml` on `v*` tags (multi-arch amd64/arm64).

## Configuration

Runtime config is read at startup from `configs/${SERVER_ENV:-dev}/service.yml` and `system.yml`:

- `service.yml`: `name`, `host`, `port`, optional `urlPrefix`, `bindAddress`
- `system.yml`: `requestLog`, `debug`, `log_level` (`error|warning|info|debug`), `logDir`, `tmpDir`, body limits

Override with CLI args (`--env`, `--host`, `--port`, `--name`) or `SERVER_ENV` / `SERVER_HOST` / `SERVER_PORT` / `SERVER_NAME`. Paths are resolved from `process.cwd()`, so the process must start from the repo root (or `/app` in Docker) where both `configs/` and `package.json` exist. Defaults in code are port 5566, but `configs/dev/service.yml` and the Dockerfile set 5100.

## Request lifecycle

`src/index.ts` → `lib/server.ts` → `api/routes/index.ts`.

Routes are **plain objects**, not koa-router instances:

```ts
export default { prefix: '/v1/images', post: { '/generations': async (request) => ... } }
```

`Server.attachRoutes` walks each object's `get`/`post` maps and registers them on a single koa-router under `service.urlPrefix`. Handlers receive a `Request` wrapper (`src/lib/request/Request.ts`) with the chainable `request.validate('body.prompt', _.isString)` helper.

Handler return values:
- A plain object is sent as-is (routes deliberately return OpenAI-shaped bodies like `{ created, data }`).
- `Response` / `Body` instances are injected via `injectTo(ctx)`.
- Anything thrown becomes a `FailureBody`: HTTP **200** with `{ code, message, data }`, where `code` is one of the negative codes in `src/api/consts/exceptions.ts`. Clients must inspect `code`, not the HTTP status.

Body parsing: a custom middleware handles `application/json` first (strips BOM/NBSP/CRLF, tries to repair trailing commas, single quotes, unquoted keys) and sets `ctx._jsonProcessed`; `koa-body` + formidable then handles everything else, including `multipart/form-data` up to 100 MB. Multipart field values arrive as strings, so routes coerce `duration`, `intelligent_ratio`, `sample_strength` by hand.

## Token, region and proxy semantics

`Authorization: Bearer <sessionid>` is a browser cookie value, not an issued API key.

- `us-`, `hk-`, `jp-`, `sg-` prefixes select the international site; no prefix means the China site. `parseRegionFromToken` drives base URLs, `aid` (assistant id), `Lan`/`Loc` headers, model maps, upload endpoints and forced resolutions.
- A comma-separated token list is accepted; each handler picks one at random (`_.sample`) for crude load balancing.
- A proxy URL can be prefixed to the token as `http://user:pass@host:port@<sessionid>` (or `socks5://...@<sessionid>`); `parseProxyFromToken` splits on the last `@` and `request()` installs a proxy agent.

## Generation pipeline

`src/api/controllers/core.ts` is the upstream client. Its `request(method, uri, token, options)`:

1. resolves region → base URL (`src/api/consts/common.ts`, `src/api/consts/dreamina.ts`)
2. computes the `Sign` header (md5 of `9e2c|last7charsOfUri|platform|version|deviceTime||11ac`) and fake browser headers/cookie
3. injects default query params (`aid`, `region`, `da_version`, `web_version`, ...) unless `noDefaultParams` is set
4. retries network/HTTP failures up to `RETRY_CONFIG.MAX_RETRY_COUNT`, then maps upstream `ret` codes through `lib/error-handler.ts` (`1015` token expired, `5000` insufficient credits, `4001` content filtered, ...)

Then for images (`api/controllers/images.ts`) and videos (`api/controllers/videos.ts`):

1. check credits, auto-collect the daily credit grant when the balance is 0
2. upload inputs through the ImageX proof flow (`lib/image-uploader.ts`, `lib/video-uploader.ts`); China-site images also go through `checkImageContent`
3. build `draft_content` / `metrics_extra` with `api/builders/payload-builder.ts` (resolution lookup, `core_param`, blend abilities, omni `meta_list` prompt parsing)
4. `POST /mweb/v1/aigc_draft/generate` → `aigc_data.history_record_id`
5. poll `POST /mweb/v1/get_history_by_ids` with `lib/smart-poller.ts` until status 10/50, status 30 (failure), stable item count, or limits; images expect ~4 items, video 1
6. extract URLs with `lib/image-utils.ts` — images from `item.image.large_images[0].image_url`; video prefers `/mweb/v1/get_local_item_list` (decoded, watermark-free `video_list.*.main_url`) and falls back to `transcoded_video` / `play_url` / `download_url`

Two video functions modes share the route: `first_last_frames` (0/1/2 images auto-detected) and `omni_reference` (up to 9 images + 3 videos referenced as `@image_file_1` in the prompt; seedance-2.0 models only).

## Where changes belong

| Change | File |
| --- | --- |
| API contract / validation / multipart handling | `src/api/routes/*.ts` |
| Generation flow, credits, payload assembly | `src/api/controllers/*.ts` |
| Draft JSON shape, resolution tables, metrics | `src/api/builders/payload-builder.ts` |
| Model names → upstream req keys, region maps, defaults, polling and retry config | `src/api/consts/common.ts` |
| Upstream hosts, app/web versions | `src/api/consts/dreamina.ts` |
| Signing, headers, cookies, ImageX upload | `src/api/controllers/core.ts` |
| Poll exit rules and intervals | `src/lib/smart-poller.ts` |
| Upstream `ret` code → APIException mapping | `src/lib/error-handler.ts` |
| Public error codes | `src/api/consts/exceptions.ts` (and `src/lib/consts/exceptions.ts` for framework errors) |

Adding a model generally means: add it to the relevant `IMAGE_MODEL_MAP*` / `VIDEO_MODEL_MAP*` entry, make sure its duration/resolution rules are handled in the route or controller, and update `/v1/models` if it should be discoverable.

## Endpoints

- `GET /` — service banner
- `GET /ping` — `"pong"` (Docker healthcheck)
- `GET /v1/models` — static model list
- `POST /v1/images/generations` — text-to-image; `ratio` + `resolution` only, `size`/`width`/`height` are rejected
- `POST /v1/images/compositions` — image-to-image / multi-image; JSON `images` array or multipart `images` files (max 10)
- `POST /v1/videos/generations` — text/image/first-last-frame/omni video; `response_format: url|b64_json`
- `POST /token/check`, `POST /token/points`, `POST /token/receive` — session validity and credit management

## Conventions and gotchas

- ESM package (`"type": "module"`) with `moduleResolution: NodeNext`, but local imports use explicit `.ts` extensions and the `@/*` → `src/*` alias. `tsup` bundles everything, so the built output does not rely on path aliases at runtime. Keep new imports in the same style; do not run `dist` through plain `tsc`.
- `tsconfig.json` is `noEmit` and `allowImportingTsExtensions`; `npm run type-check` is the only static gate.
- Handlers should throw `APIException(EX.*, message)` to get a meaningful client-facing `code`; plain `Error` becomes `code: -1`.
- Generation is blocking. There is no queue, job store, or callback endpoint; scaling means adding long client/upstream timeouts.
- `intelligent_ratio` only takes effect for `jimeng-4.0/4.1/4.5/4.6/5.0`; `nanobanana` is international-only with forced resolutions (US: 1024×1024 @2k; HK/JP/SG: 1k); `nanobananapro` has its own 4k table.
- `vercel.json` exists, but the flow relies on long in-process polling and file logging; the Vercel build is not the primary deployment path.
- `jimeng-api/` at the repo root is a separate Claude Code skill (Python client + `Skill.md`), not part of the server build.
- Logging goes to stdout and `./logs/YYYY-MM-DD.log` (skipped when `process.env.VERCEL` is set). `requestLog: true` in dev logs every inbound/outbound request at info level, which is verbose.
