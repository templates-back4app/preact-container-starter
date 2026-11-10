# Deploy through the Back4app MCP — preact-container-starter — 2026-11-10

Repo commit uploaded: `5d26569` (branch `main`, clean tree).

## Context (same for the 9 starters measured this day)

- Date: 2026-11-10 (all times UTC).
- Account kind: permanent, signed in (`whoami` -> `accountKind: "permanent"`, `limits: {freeAppsUsed: 19, maxFreeApps: 50}` before this run). Apps created on the Free plan.
- Client: Claude Code driving the Back4app MCP connector; terminal on macOS (Node v25.8.1, curl 8.7.1); validation requests left from Sao Paulo (CloudFront POP `GRU1-P5`).
- Upload CLI: `@back4app/cli` 0.3.2, run as `npx -y @back4app/cli upload --url <upload-link>` inside the starter folder. The upload link carries a one-time token and is not recorded here. `create_upload` returned `expiresAt` 15 minutes after the call.
- "create -> READY" below = deployment `createdAt` to the `updatedAt` of the `get_deployment` answer that first said `READY`. It is the platform's own clock, not the wall time of the polling.

## Tools called, in order

Part A: `create_upload` (no appId) -> CLI upload in the folder -> `create_web_app` -> `get_web_app` -> `get_deployment` -> `get_runtime_logs` (and once more 8 minutes later)

Part B1: `create_upload` (no appId) -> CLI upload in the folder -> `create_web_app` (`builder: "BUILDPACK"`) -> `get_web_app` -> `get_deployment` (x2, the second one FAILED with the build log tail) -> `get_web_app`

## Part A — Dockerfile (builder not specified)

### Upload (CLI output)

```text
Packing .../preact-container-starter ...
27 file(s), 169 KB (2 left out)
Sending 15 new file(s) ...
Upload ready.
uploadId: eccc8feb-b847-4351-ba19-5c02e1a816b6
manifestSha256: 952837f00e47968cb95bcdf44a94b9bd1482d92fcab7a9d31b54050031f05b45
```

- Wall time: 7 s. No "may hold secrets" message for this folder.
- The repo tracks 20 files; the CLI packed 27, which is every file in the folder outside `node_modules/` and `.git/` (local `dist/` included: the CLI does not read `.gitignore`).

### App and deployment

- `create_web_app` arguments: `name: "mcp-preact"`, `port: 8080`, `uploadId`, `manifestSha256`; `builder` not specified.
- appId `2d94f92b-84b9-4bb0-ac86-f8cd4b091593` · deploymentId `20e898f9-2f4e-4e9c-b2dd-17d18c3b953b`
- App `createdAt` 12:54:37.966Z · deployment `createdAt` 12:54:38.127Z
- Status seen: `BUILDING` / `BUILDING_IMAGE` (12:54:38) -> `READY` (`updatedAt` 12:54:56.094Z) -> **18.0 s**
- Public URL: 404 at 12:54:46, first 200 at 12:54:59.

Settings reported by `get_web_app`:

- `builder`: `DOCKERFILE` · `port`: `8080` · `startCommand`: `null` · `healthCheckPath`: `"/"` · `isStaticSite`: `false` · `rootDir`: `""` · `runtimeVersion`: `null` · `envVarNames`: `[]` · `plan`: `Free` · `source.kind`: `UPLOAD`
- `url`: https://mcppreact-14em1ksp.back4.app
- `urlExpiresAt`: 2026-11-10T13:54:38.066Z (60 minutes after the app was created)
- `get_deployment` returned no `suggestions` and no `portDetected` on this deployment.

### Validation on the public URL (12:55Z)

- `GET /` -> 200, `content-type: text/html`, `server: nginx/1.27.5`, `via: 1.1 ... .cloudfront.net (CloudFront)`, no `content-encoding` (473 B document).
- JS referenced by the HTML: `/assets/index-e_QwmrLR.js` -> 200, **9,816 B** transferred with `Accept-Encoding: gzip` (17.97 kB on disk per the build output). Headers: `content-encoding: gzip`, `server: nginx/1.27.5`, `cache-control: public, max-age=31536000, immutable`, `via: ... (CloudFront)`. Same 9,816 B as the GitHub-source deploy of 2026-11-09.
- Deep route: `GET /some/deep/route` -> 200, 473 B (`index.html`).
- Missing asset: `GET /assets/nope.js` -> 404.

### Who compressed the JS: the platform edge, not the starter's nginx

- Local references for `dist/assets/index-e_QwmrLR.js` (17,970 B): `gzip -6` 7,353 B, `gzip -1` 7,959 B. The public URL delivered 9,816 B with `content-encoding: gzip`: neither.
- Local check on 2026-11-10 (this starter's `nginx.conf` and `dist/`, `nginx:1.27-alpine`, Docker 29.5.2): a request with `Accept-Encoding: gzip` gets 7,335 B gzip; the same request plus a `Via:` header gets 17,970 B and no `content-encoding`. nginx's `gzip_proxied` defaults to `off`, which disables compression for requests that arrive through a proxy, and the Back4app edge is one.
- On the solidjs, lit and angular apps deployed the same hour, the nginx access log shows the uncompressed size sent for the JS file while the client received gzip. (This app's runtime log came back empty, so the same line could not be read here.)
- Consequence: `gzip on; gzip_comp_level 6;` in the starter's `nginx.conf` does nothing on this platform as written. The reader still gets gzip (from the edge), at a weaker ratio than nginx level 6 would give: 9,816 B instead of about 7,335 B.
- The fix would be `gzip_proxied any;` in `nginx.conf`. NOT applied and NOT tested on Back4app.

### Runtime logs

`get_runtime_logs` returned 0 lines at 12:55Z and again 0 lines at 13:03Z, although the same nginx image on the solidjs and lit apps returned 26 and 27 lines minutes apart. An empty answer from this tool did not mean the container wrote nothing.

## Part B1 — Buildpack on a static app: FAILED

Same commit, same folder, uploaded again:

```text
27 file(s), 169 KB (2 left out)
Back4App already had every file: nothing to send.
Upload ready.
uploadId: 2bd8c9cd-a3f5-4bc2-9199-9641ae629b4a
manifestSha256: 952837f00e47968cb95bcdf44a94b9bd1482d92fcab7a9d31b54050031f05b45
```

Wall time 4 s; same `manifestSha256` as Part A, new `uploadId`.

- `create_web_app` arguments: `name: "mcp-preact-buildpack"`, `builder: "BUILDPACK"`, `uploadId`, `manifestSha256`. No `port`. The answer carried no `suggestions`.
- appId `e022c612-839e-49ec-b61a-aa66ad20eb33` · deploymentId `170bf401-b6a3-4db8-8120-b78c39169ddd`
- App `createdAt` 12:56:37.795Z · deployment `createdAt` 12:56:37.961Z -> `FAILED` (`updatedAt` 12:58:11.155Z), 93.2 s after creation.
- `get_web_app`, during the build and after the failure: `builder: "BUILDPACK"`, `port: null`, `startCommand: null`, `healthCheckPath: "/"`, `isStaticSite: false`, `runtimeVersion: null`, `state: "OFFLINE"`. A URL was assigned (https://mcppreactbuildpack-8fv6l5iy.back4.app, `urlExpiresAt` 13:56:37.893Z) and answered 404 the whole time.
- `get_deployment` on the failed deployment:

```json
"error": {
  "code": "DEPLOYMENT_FAILED",
  "message": "The deployment failed. The build log says at which step and why."
}
```

  plus `buildLogTail` (so `get_build_logs` was not needed). No `suggestions`, no `portDetected`.
- Decisive build log lines, verbatim. The build itself passed:

```text
- Running `npm ci`
added 88 packages in 11s
- Running `npm run build`
dist/assets/index-e_QwmrLR.js   17.97 kB │ gzip: 7.44 kB
✓ built in 422ms
- Pruning dev dependencies
- Configuring default processes
- Skipping default web process (no start script defined)
no default process type
Your app was built successfully, but it has no start process: the buildpack found no Procfile or start script. Set a Start Command in Settings (e.g. "gunicorn app:app" or "node index.js") and redeploy.
deployment failed
```

- Result: the buildpack treats a Vite app as a Node app, builds `dist/`, and then has nothing to run. It did not turn the app into a static site (`isStaticSite` stayed `false`). Deep-route status and JS `Content-Encoding` could not be measured: nothing was served.
- What the fix would be (not applied, not tested): keep the Dockerfile (builder left unspecified picks it), or give the buildpack a process that serves `dist/` on `$PORT` with a fallback to `index.html`: a `start` script plus a static server in `dependencies`, or a `startCommand`. The app was left as it failed.

## Not tested

- `gzip_proxied any;` on Back4app.
- Any of the fixes above; a buildpack deploy of a static app that does go READY (so: no deep-route or compression numbers for the buildpack path).
- Whether another setting or tool can mark an app as a static site.
- Not tested anywhere in this series: `update_web_app` with a confirmationId, `rollback_web_app`, `confirm_action`, custom domains, env vars at creation, GitHub as the code source through MCP, behaviour after `urlExpiresAt`, a temporary (not signed in) account, the `files` upload path for clients with no terminal.

## Compression fix verified on the public URL (2026-11-10)

- Cause: requests reach the container through the platform's CDN with a `Via` header, and nginx's `gzip_proxied` is off by default, so nginx answered them uncompressed and the CDN compressed the response itself.
- Fix: `gzip_proxied any;` in nginx.conf (commit 8c7e2aa), redeployed to the same app with `deploy_web_app`.
- Main JS transferred from the public URL (`Accept-Encoding: gzip`): before 9,816 B, after 7,335 B. The "after" figure equals this bundle compressed by nginx at gzip_comp_level 6.
