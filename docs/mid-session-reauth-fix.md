# mcp-remote — Mid-Session Re-Authentication Fix

Local fork of [`geelen/mcp-remote`](https://github.com/geelen/mcp-remote) **v0.1.38**.
This patch fixes mid-session OAuth re-authentication for MCP servers that issue a `401 Unauthorized`
on a tool call after the initial session is already established (for example, when the server
revokes the access token to force a fresh consent / enrollment flow).

> **TL;DR:** Without this patch, mcp-remote silently swallows the mid-session 401, the JSON-RPC
> request to the remote server hangs forever, and the MCP client (Claude Desktop, etc.) shows
> a generic "tool execution failed" error — even though the user already completed the new
> OAuth flow in their browser.

---

## 1. Symptom

Concrete reproduction with `zpi-zkpay` (a Streamable-HTTP MCP server with custom OAuth):

1. Claude Desktop starts. mcp-remote performs initial OAuth → tokens cached → tools/list returns OK.
2. User calls a tool that succeeds (`tools/call` → 200). Tokens still valid.
3. User calls a tool that the server requires re-enrollment for. Server replies `401` and
   internally sets the user state to "enrollment pending".
4. mcp-remote tries `grant_type=refresh_token` → server returns `400` (refresh blocked while
   enrollment is pending). **Expected:** mcp-remote opens the browser, waits for the user to
   re-authorize, redeems the new code, replays the original `tools/call`. **Actual:** Claude
   shows "Tool execution failed".

Server-side logs (representative window):

```text
[REQ] POST /mcp
[MCP] Executing tool 'zpi-setup-payment' …
[Enrollment] Queued full-OAuth enrollment for user X, method=vgs_credit_card
[RES] POST /mcp → 401 (8ms)

[REQ] POST /token grant_type=refresh_token
[OAuth] Refresh token grant REJECTED for user X: enrollment pending — forcing full OAuth re-auth
[RES] POST /token → 400

# ↓↓↓ The smoking gun — mcp-remote replays the *previous* auth code
[REQ] POST /token grant_type=authorization_code code=_TN40ddH...
[OAuth] Token exchange REJECTED: code already used
[RES] POST /token → 400
[REQ] POST /token grant_type=authorization_code code=_TN40ddH...   ← duplicate
[OAuth] Token exchange REJECTED: code already used
[RES] POST /token → 400

# Browser opens, user enrolls, server redirects with NEW code …
[REQ] GET /authorize …
[REQ] POST /login …
[OAuth] Redirecting to: http://localhost:8720/oauth/callback?code=--QnC0SG…

# … but mcp-remote never POSTs /token with the new code. The original
# tools/call request times out from Claude's perspective.
```

---

## 2. Root cause

### Bug A — `mcpProxy.onServerError` swallows mid-session `UnauthorizedError`

In [`src/lib/utils.ts`](../src/lib/utils.ts), the bidirectional proxy forwards client → server
messages with:

```ts
transportToServer.send(message).catch(onServerError)
```

`onServerError` only logs the error. When the SDK's
`StreamableHTTPClientTransport.send()` throws `UnauthorizedError` mid-session (because the
server returned 401 and the SDK's internal `auth()` helper could not silently recover via
refresh), `onServerError` swallows it and **never** triggers the OAuth callback flow that
already exists for the *initial* connect path (in `connectToRemoteServer` /
`coordinateAuth`). The original JSON-RPC request id is never replied to, so the MCP client
hangs.

### Bug B — `setupOAuthCallbackServerWithLongPoll` reuses a stale auth code

Even after Bug A is fixed and we route mid-session `UnauthorizedError` through the existing
callback-server flow, a second bug surfaces. The callback server stores the most recent code
in a closure variable that is **never cleared**:

```ts
let authCode: string | null = null
…
app.get(options.path, (req, res) => {
  authCode = code   // sticks for the lifetime of the process
  authCompletedResolve(code)
  options.events.emit('auth-code-received', code)
})

const waitForAuthCode = (): Promise<string> =>
  new Promise((resolve) => {
    if (authCode) { resolve(authCode); return }   // ← returns stale code
    options.events.once('auth-code-received', resolve)
  })
```

`createLazyAuthCoordinator.initializeAuth()` memoizes the auth state for the process lifetime,
so on a second invocation `waitForAuthCode()` immediately resolves with the **previous**,
already-redeemed authorization code. The SDK then `POST /token` with that stale code, the
server returns `400 "code already used"`, and the OAuth flow aborts. The browser-driven new
OAuth round still completes, but its fresh code is delivered to a callback server whose
`waitForAuthCode` consumer has already resolved.

This is what produces the duplicate stale-code `POST /token` pair in the server logs above.

### 2.1 Why initial connect works but mid-session does not

The same OAuth machinery exists in two completely separate code paths inside upstream
mcp-remote, and **only one of them has 401 recovery wired in**:

| Path | Triggered by | 401 handler | Result |
| --- | --- | --- | --- |
| **A. Initial connect** — [`connectToRemoteServer`](../src/lib/utils.ts) (the `else if (error instanceof UnauthorizedError)` branch around line 578) | mcp-remote startup, before `mcpProxy` exists | Calls `authInitializer()` → `waitForAuthCode()` → `transport.finishAuth(code)` → recursively reconnects | OAuth flow runs, browser opens, tokens persisted, session established |
| **B. Mid-session send** — `mcpProxy.transportToServer.send(message).catch(onServerError)` (upstream, around line 213) | Any `tools/call` after `mcpProxy` is running | `onServerError` only **logs** the error | JSON-RPC reply never arrives, MCP client hangs / shows "Tool execution failed" |

Path A's recovery is bound to its own call site: it can only run during `client.connect(transport)`,
because that is the only point at which `connectToRemoteServer` controls the transport
end-to-end. Once `connectToRemoteServer` returns successfully, ownership transfers to
`mcpProxy` and Path A is unreachable for the rest of the process lifetime.

That is exactly why the bug only manifests when a *user-initiated* `tools/call` (e.g.
"Setup payment") triggers a 401, and never on Claude-Desktop-initiated startup.

### 2.2 Can we force mid-session traffic through Path A instead of patching Path B?

**No, not cleanly.** Path A's entry point is the construction of a new transport plus
`client.connect(transport)`. To re-enter it from inside `mcpProxy` you would have to:

1. Detach the live `transportToServer` (and lose any in-flight request → response id
   mapping it carries).
2. Tear down the transport.
3. Construct a fresh transport and call `connectToRemoteServer(...)` again, which
   internally re-runs the MCP `initialize` handshake and re-issues `tools/list`.
4. Reattach the new transport to `mcpProxy` without dropping the local stdio session that
   Claude Desktop is already using.

Step 4 is the blocker. mcp-remote's design assumes the remote transport is long-lived for
the lifetime of the local stdio session: there is no swap-transport hook on the proxy, the
local `Client` has cached server capabilities from the prior `initialize`, and any pending
JSON-RPC ids would be orphaned. Even if you wired the swap, the work involved is strictly
greater than what this patch does — and it would still need an `authInitializer` reference
inside `mcpProxy` (the very thing the patch adds) to know *when* to swap.

The minimal correct fix is therefore to give Path B the recovery semantics Path A already
has, against the same long-lived transport. That is what §3 implements:
`mcpProxy.onServerSendError` calls the same `authInitializer` / `waitForAuthCode` /
`finishAuth(code)` sequence that Path A uses, but on the existing `transportToServer`,
and replays the failed message instead of reconnecting.

---

## 3. Patch

Two files changed; both keep full backward compatibility.

### 3.1 `src/lib/utils.ts` — handle mid-session 401 in the proxy

`mcpProxy(...)` accepts a new optional `authInitializer` parameter. The local→remote send
path now routes errors through `onServerSendError`, which:

1. Forwards a JSON-RPC `-32603` error to the client if the failure is **not** a 401-style
   error (preserves prior swallow-as-log behaviour for non-auth errors but with a clear reply
   to the client instead of a hang).
2. On `UnauthorizedError` and with `authInitializer` available, coalesces concurrent
   re-auth attempts via a single `midSessionAuthInFlight` promise so multiple in-flight
   tools/call requests share one OAuth round-trip.
3. Awaits `authInitializer()` (which starts/reuses the OAuth callback server),
   `waitForAuthCode()`, then `transportToServer.finishAuth(code)`.
4. Replays the original message. If the replay also fails with a 401 (`isReplay = true`),
   surfaces the error to the client to prevent infinite loops.

### 3.2 `src/lib/utils.ts` — consume-once `authCode` cache

In `setupOAuthCallbackServerWithLongPoll`:

- Added a sticky `authCompleted: boolean` flag — used by the cross-instance `/wait-for-auth`
  long-poll endpoint so that secondary instances still correctly observe "auth completed".
- `waitForAuthCode()` now sets `authCode = null` immediately when handing it to the caller
  (both the cached-value branch and the live-event branch), so a subsequent mid-session
  re-auth waits for the **next** `auth-code-received` event instead of replaying the
  already-redeemed code.

### 3.3 `src/proxy.ts` — wire `authInitializer` into `mcpProxy`

`runProxy()` already constructed `authInitializer` for the initial connect. The patch
forwards the same closure into `mcpProxy({ …, authInitializer })`.

---

## 4. End-to-end trace after the fix

For the same `zpi-zkpay` reproduction:

1. Tool call → server `401` → SDK throws `UnauthorizedError`.
2. `mcpProxy.onServerSendError` catches it, calls `authInitializer()`.
3. `createLazyAuthCoordinator` returns the memoized auth state. Callback server still
   running on `http://127.0.0.1:8720`.
4. `waitForAuthCode()` finds `authCode === null` (consumed during initial OAuth) and
   subscribes to `auth-code-received`.
5. SDK's internal `auth()` already kicked off `redirectToAuthorization` → browser opens at
   `/authorize`. User completes enrollment.
6. Server redirects to `http://localhost:8720/oauth/callback?code=NEW`.
7. Callback handler stores `authCode = NEW`, emits `auth-code-received` → our
   `waitForAuthCode` resolves with `NEW` (and immediately consumes it back to null).
8. `transportToServer.finishAuth(NEW)` POSTs `/token` with the **new** code → 200, new
   tokens written to `~/.mcp-auth/mcp-remote-0.1.38/…_tokens.json`.
9. Original `tools/call` is replayed → 200 → JSON-RPC response delivered to the MCP client.

If step 8 fails (e.g. user closed the browser), the patched code path returns a JSON-RPC
`-32603` error to the client with a clear message instead of hanging.

---

## 5. Building

This fork uses `pnpm` (pinned via the `packageManager` field; Corepack will fetch the right
version on first install).

```bash
cd /Users/vietle/go/src/github.com/zeroproof/mcp-remote
pnpm install --prefer-offline
pnpm build
```

Build output:

```
dist/
├── chunk-*.js     # shared bundle (utils.ts lives here)
├── client.js      # mcp-remote-client entry
├── client.d.ts
├── proxy.js       # mcp-remote entry — this is what Claude Desktop runs
└── proxy.d.ts
```

The two relevant entry points are:

- `dist/proxy.js` — bin `mcp-remote` (stdio ↔ remote HTTP/SSE proxy)
- `dist/client.js` — bin `mcp-remote-client` (debug client)

Both are ES modules with a Node shebang. Run them directly with `node`.

### Quick sanity check

```bash
# Confirm the patched symbols made it into the build:
grep -n "midSessionAuthInFlight\|onServerSendError\|Mid-session" dist/chunk-*.js | head
# expected: matches in `mcpProxy` showing the new handler/state
```

---

## 6. Wiring into Claude Desktop

Claude Desktop's MCP config lives at:

```
~/Library/Application Support/Claude/claude_desktop_config.json
```

Replace any `npx -y mcp-remote@<ver>` invocation with the patched local build:

```jsonc
{
  "mcpServers": {
    "zpi-zkpay": {
      "command": "/Users/vietle/.nvm/versions/node/v22.21.1/bin/node",
      "args": [
        "/Users/vietle/go/src/github.com/zeroproof/mcp-remote/dist/proxy.js",
        "http://localhost:3002/mcp"
      ],
      "env": {
        "PATH": "/Users/vietle/.nvm/versions/node/v22.21.1/bin:/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin"
      }
    }
  }
}
```

Notes:

- Use the **absolute path** to a Node ≥ 18 binary as `command` so Claude Desktop's PATH
  doesn't matter. The example uses the nvm-managed Node 22; adjust if you use Homebrew or
  asdf.
- Pass the remote MCP URL as a positional arg, exactly as you would to `npx -y mcp-remote`.
- Any extra mcp-remote CLI flags (`--header`, `--allow-http`, `--transport`, `--debug`,
  `--callback-port`, etc.) are passed after the URL.

After editing, fully **quit** Claude Desktop (`Cmd-Q`, not just close the window) and
relaunch.

### Verifying the patch is live

mcp-remote logs go to `~/Library/Logs/Claude/mcp-server-<name>.log`. On a mid-session 401
you should see new log lines:

```
Mid-session auth required. Initializing auth flow...
…
Completing mid-session authorization...
Mid-session authorization completed.
Replaying original request after mid-session re-auth.
```

If instead you see only `Error from remote server: UnauthorizedError…` with no follow-up,
the old (unpatched) `mcp-remote` is still running — double-check the `command`/`args` in
`claude_desktop_config.json` and that you fully quit Claude Desktop before reopening.

### Forcing a fresh auth round (optional)

If you want to test the *initial* OAuth path as well, clear the cached tokens:

```bash
rm -rf ~/.mcp-auth/mcp-remote-0.1.38/
```

(Leave the cache in place to test only the mid-session 401 path; the bug requires an
already-valid initial session.)

---

## 7. Files touched

| File                          | Change                                                                             |
| ----------------------------- | ---------------------------------------------------------------------------------- |
| `src/lib/utils.ts`            | `mcpProxy` accepts `authInitializer`; new `onServerSendError`; consume-once `authCode` |
| `src/proxy.ts`                | Forward `authInitializer` into `mcpProxy({...})`                                   |
| `docs/mid-session-reauth-fix.md` | This document                                                                   |

No test files were modified; the existing 8 `mcpProxy({...})` call sites in
`src/lib/utils.test.ts` continue to compile and pass because `authInitializer` is optional.

---

## 8. Distribution & installer rollout

The local `dist/` build is enough for development on this machine, but the official
`zero-proof-intent` and `zk-attestation-service` installers (`install.sh` / `install.ps1`)
still pull **upstream `mcp-remote@0.1.38` from the public npm registry** and never see the
patch. To roll the fix out to all users we need to (a) publish the fork to npm under a
distinct name and (b) point every installer + uninstaller at that package.

### 8.1 Task: publish the fork to npm

**Package name:** `@zeroproofai/mcp-remote` (scoped under the existing `@zeroproofai` org;
keeps the upstream `mcp-remote` name free and makes the divergence obvious).

**Recommended initial version:** `0.1.38-zeroproofai.1` (preserves the upstream base version,
adds a vendor pre-release suffix that increments per patch).

Subtasks:

1. **Update `package.json`** in this repo:
   - `"name": "@zeroproofai/mcp-remote"`
   - `"version": "0.1.38-zeroproofai.1"`
   - `"publishConfig": { "access": "public" }` (scoped packages default to private).
   - Update the `"bugs"` and `"repository"` URLs to point at the zeroproof fork.
   - Keep the `"bin"` block unchanged (`mcp-remote → dist/proxy.js`,
     `mcp-remote-client → dist/client.js`) so the CLI surface is identical.
2. **Add a `prepublishOnly` script** that runs `pnpm build` so the published tarball
   always contains a fresh `dist/`:
   ```json
   "scripts": { "prepublishOnly": "pnpm build" }
   ```
3. **Verify `files` field** in `package.json` includes only `dist/`, `README.md`, and
   `LICENSE` (drop `src/`, `test/`, `docs/` from the published tarball — keep the package
   small).
4. **Dry-run publish:** `pnpm publish --dry-run --access public` and inspect the file list.
5. **Publish:** `npm login` (under the `@zeroproofai` org) then
   `pnpm publish --access public`.
6. **Smoke test from a clean dir:**
   ```bash
   mkdir /tmp/mcp-remote-smoke && cd /tmp/mcp-remote-smoke
   npm init -y && npm install -g @zeroproofai/mcp-remote@0.1.38-zeroproofai.1
   which mcp-remote && mcp-remote --version 2>&1 | head -3
   ```
7. **Tag the commit** in this repo: `git tag v0.1.38-zeroproof.1 && git push --tags`.

For each future patch: bump the suffix (`-zeroproof.2`, `.3`, …), repeat steps 4–7.

### 8.2 Task: update `zero-proof-intent` install scripts

Files (and their identical duplicates under
`zk-attestation-service/attester/scripts/` — both copies must be edited):

- [zero-proof-intent/scripts/install.sh](../../zero-proof-intent/scripts/install.sh)
- [zero-proof-intent/scripts/install.ps1](../../zero-proof-intent/scripts/install.ps1)
- [zero-proof-intent/scripts/uninstall.sh](../../zero-proof-intent/scripts/uninstall.sh)
- [zero-proof-intent/scripts/uninstall.ps1](../../zero-proof-intent/scripts/uninstall.ps1)
- `zk-attestation-service/attester/scripts/install.sh`
- `zk-attestation-service/attester/scripts/install.ps1`
- `zk-attestation-service/attester/scripts/uninstall.sh` (if present)
- `zk-attestation-service/attester/scripts/uninstall.ps1` (if present)

#### 8.2.1 `install.sh` changes

Current state (`MCP_REMOTE_VERSION="0.1.38"`, line 44; install via
`npm install -g "mcp-remote@${MCP_REMOTE_VERSION}"`, line 235):

```diff
- MCP_REMOTE_VERSION="0.1.38"
+ MCP_REMOTE_PACKAGE="@zeroproof/mcp-remote"
+ MCP_REMOTE_VERSION="0.1.38-zeroproof.1"
```

```diff
  test_global_mcp_remote_cli() {
      command -v mcp-remote &>/dev/null
  }
+
+ # True only if the installed mcp-remote is the zeroproof fork at the expected version.
+ test_global_mcp_remote_is_zeroproof() {
+     command -v mcp-remote &>/dev/null || return 1
+     local installed
+     installed="$(npm ls -g --depth=0 --json 2>/dev/null \
+         | node -e 'let s="";process.stdin.on("data",d=>s+=d).on("end",()=>{try{const j=JSON.parse(s);const d=j.dependencies||{};process.stdout.write(d["@zeroproof/mcp-remote"]?.version||"")}catch(e){}})' 2>/dev/null)"
+     [[ "$installed" == "$MCP_REMOTE_VERSION" ]]
+ }
```

```diff
  initialize_global_mcp_remote() {
-     if test_global_mcp_remote_cli; then
+     if test_global_mcp_remote_is_zeroproof; then
          ok "mcp-remote@${MCP_REMOTE_VERSION} already installed globally"
          return 0
      fi
+     # Remove an upstream mcp-remote install if it would shadow the fork.
+     if test_global_mcp_remote_cli; then
+         info "Removing upstream mcp-remote (replacing with ${MCP_REMOTE_PACKAGE}@${MCP_REMOTE_VERSION})"
+         npm uninstall -g mcp-remote </dev/null || true
+     fi
      if ! command -v npm &>/dev/null; then
          warn "npm not found; zpi-zkpay will fall back to npx -y in Claude config"
          return 1
      fi
-     info "Installing mcp-remote@${MCP_REMOTE_VERSION} globally..."
-     if npm install -g "mcp-remote@${MCP_REMOTE_VERSION}" </dev/null; then
+     info "Installing ${MCP_REMOTE_PACKAGE}@${MCP_REMOTE_VERSION} globally..."
+     if npm install -g "${MCP_REMOTE_PACKAGE}@${MCP_REMOTE_VERSION}" </dev/null; then
          if test_global_mcp_remote_cli; then
              ok "mcp-remote@${MCP_REMOTE_VERSION} installed"
              return 0
          fi
      fi
      warn "Global mcp-remote install failed; zpi-zkpay will fall back to npx -y in Claude config"
      return 1
  }
```

If the script writes the npx fallback into `claude_desktop_config.json`, that fallback also
needs to point at the scoped package:

```diff
- "args": ["-y", "mcp-remote@${MCP_REMOTE_VERSION}", "${MCP_URL}"]
+ "args": ["-y", "${MCP_REMOTE_PACKAGE}@${MCP_REMOTE_VERSION}", "${MCP_URL}"]
```

(Search for `mcp-remote@` in the config-emission block — there's only one site.)

#### 8.2.2 `install.ps1` changes

Same shape, PowerShell idioms (`$McpRemoteVersion` lives near the top of the file,
install at line 168):

```diff
- $McpRemoteVersion = "0.1.38"
+ $McpRemotePackage = "@zeroproof/mcp-remote"
+ $McpRemoteVersion = "0.1.38-zeroproof.1"
```

```diff
- & npm install -g "mcp-remote@$McpRemoteVersion"
+ & npm install -g "$McpRemotePackage@$McpRemoteVersion"
```

Add the equivalent "uninstall upstream `mcp-remote` if it shadows the fork" guard before
the install call. Update the npx fallback string in the Claude config emitter the same way
as `install.sh`.

#### 8.2.3 `uninstall.sh` / `uninstall.ps1` changes

Both uninstallers currently have **no mcp-remote logic**. Add a step that removes the
zeroproof fork (and any leftover upstream copy) so users get a clean removal:

`uninstall.sh`:

```bash
# ── Remove zeroproof mcp-remote ──────────────────────────────
if command -v npm &>/dev/null; then
    if npm ls -g --depth=0 --json 2>/dev/null | grep -q '"@zeroproof/mcp-remote"'; then
        info "Removing @zeroproof/mcp-remote..."
        npm uninstall -g @zeroproof/mcp-remote </dev/null || true
    fi
    # Best-effort: also remove the upstream package if a previous install left it behind.
    if npm ls -g --depth=0 --json 2>/dev/null | grep -q '"mcp-remote"'; then
        npm uninstall -g mcp-remote </dev/null || true
    fi
fi
# Wipe the on-disk OAuth cache so the next install starts clean.
rm -rf "${HOME}/.mcp-auth" 2>/dev/null || true
```

`uninstall.ps1`:

```powershell
# ── Remove zeroproof mcp-remote ──────────────────────────────
if (Get-Command npm -ErrorAction SilentlyContinue) {
    $globalLs = & npm ls -g --depth=0 --json 2>$null | Out-String
    if ($globalLs -match '"@zeroproof/mcp-remote"') {
        Write-Info "Removing @zeroproof/mcp-remote..."
        & npm uninstall -g "@zeroproof/mcp-remote" 2>$null
    }
    if ($globalLs -match '"mcp-remote"') {
        & npm uninstall -g mcp-remote 2>$null
    }
}
$mcpAuth = Join-Path $env:USERPROFILE ".mcp-auth"
if (Test-Path $mcpAuth) { Remove-Item -Recurse -Force $mcpAuth -ErrorAction SilentlyContinue }
```

#### 8.2.4 Migration for already-installed users

Existing installs have `mcp-remote@0.1.38` (upstream) globally. The patched `install.sh` /
`install.ps1` above already detect that case and uninstall the upstream package before
installing `@zeroproof/mcp-remote`, so re-running the installer is the migration path. No
manual user action needed beyond:

```bash
curl -fsSL https://… /install.sh | bash    # macOS / Linux
# or
iwr https://… /install.ps1 -useb | iex     # Windows
```

After re-running, users must fully quit Claude Desktop and relaunch (the proxy is loaded
once at app startup).

### 8.3 Acceptance checklist

- [ ] `@zeroproofai/mcp-remote@0.1.38-zeroproofai.1` is published and installs cleanly via
      `npm install -g`.
- [ ] `which mcp-remote` resolves to the global npm bin and `mcp-remote --version`
      reports `0.1.38-zeroproofai.1`.
- [ ] [zero-proof-intent/scripts/install.sh](../../zero-proof-intent/scripts/install.sh)
      and `.ps1` install the fork on a clean machine.
- [ ] Both installers cleanly upgrade a machine that already has upstream
      `mcp-remote@0.1.38` (the upstream-uninstall guard runs).
- [ ] [zero-proof-intent/scripts/uninstall.sh](../../zero-proof-intent/scripts/uninstall.sh)
      and `.ps1` remove the fork and the on-disk `~/.mcp-auth` cache.
- [ ] The same edits land in
      `zk-attestation-service/attester/scripts/{install,uninstall}.{sh,ps1}` (no drift
      between the two copies).
- [ ] End-to-end: a fresh `install.sh` run on macOS, followed by Claude Desktop relaunch
      and a `zpi-setup-payment` re-enrollment trigger, exercises the mid-session re-auth
      path described in §4 and the original `tools/call` succeeds.

