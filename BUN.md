# Running OmniRoute on Bun (Windows)

Measured on 2026-08-27 · Bun 1.4.0 · Node v24.16.0 · Windows 11 26200 · OmniRoute 3.8.51

**Short version: build with Node, run with Bun.**

```bash
bun install          # works — see trustedDependencies below
npm run build        # MUST be Node. Bun deadlocks here.
bun run start:bun    # works
```

## What works under Bun

| gate | result |
|---|---|
| `bun install` | 1617 packages, exit 0 |
| `bun run test:bun:db` | 22 pass / 0 fail |
| `bun scripts/dev/run-next.mjs start` (on a Node-produced build) | serves |

Endpoints answer identically under both runtimes, on the same build:

| endpoint | node | bun |
|---|---|---|
| `/healthz` | 200 | 200 |
| `/api/health` | 200 | 200 |
| `/` | 307 | 307 |
| `/v1/models` | 401 | 401 |

Boot time, two rounds with the order swapped: node 12 s / 4 s, bun 3 s / 3 s.
Bun is not slower. The sample is too small to claim more than that.

## What does not work under Bun

### 1. The production build deadlocks

`bun scripts/build/build-next-isolated.mjs` never finishes on Windows.
Reproduced twice, the second time under the exact conditions where the Node
build succeeded:

| run | outcome |
|---|---|
| node | **exit 0, 18 m 36 s** |
| bun #1 | last file written at t+95 s, then idle; killed at 2 h 54 m. CPU duty cycle 3.8 % |
| bun #2 | printed `Creating an optimized production build ...`, then idle; killed by a 45 min timeout |

Child processes at the stall: `bun.exe` (the spawned `next build`) and
`esbuild.exe`. The parent is alive and waiting; nothing is consuming CPU.

Keep `build`, `build:backend`, `build:cli` and `build:release` on Node.

### 2. `omniroute doctor` crashes Bun

```
panic: NAPI FATAL ERROR: Error::New napi_get_last_error_info
oh no: Bun has crashed. This indicates a bug in Bun, not your code.
```

Reproduced 3/3, ~370 ms in, before the first check is printed. Node runs the
same command and reports 49 checks. Every top-level import of
`bin/cli/commands/doctor.mjs` loads fine on its own under Bun, so the fault is
in the combination, not in any single module. Not narrowed further.

Use `node bin/omniroute.mjs doctor`.

### 3. `bin/omniroute.mjs serve` is not the source-tree entry point

It expects the published package layout and exits with
`Server not found at: <repo>/app/server.js`. This is not a runtime difference —
Node fails the same way. From a source checkout use `scripts/dev/run-next.mjs start`.

## trustedDependencies

Bun blocks lifecycle scripts for packages it does not trust. Eight were blocked
here:

`@parcel/watcher` · `@playwright/browser-chromium` · `@swc/core` · `koffi` ·
`onnxruntime-node` · `protobufjs` · `tls-client-node` · `unrs-resolver`

Six of them ship prebuilt binaries and survive being blocked. **`@playwright/browser-chromium`
does not** — its `install.js` is what registers the browser, so with it blocked
the browser-automation paths have nothing to run. The `trustedDependencies`
array in `package.json` unblocks all eight; after adding it, `bun pm untrusted`
reports 0.

## Other Bun notes

- `bun install` warns twice: *Bun currently only supports one level of nested
  `overrides`* (`package.json` minimatch overrides). Harmless here.
- The repo's own `postinstall` runs `node scripts/build/postinstall.mjs`, and
  122 of the 177 scripts hardcode `node`. A "pure Bun" checkout is not a
  realistic goal; Node stays a build-time requirement.
- `--max-old-space-size` in `dev` and in the build script is a V8 flag. Bun uses
  JavaScriptCore and ignores it.
