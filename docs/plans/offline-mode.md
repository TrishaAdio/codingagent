# Offline Mode Plan — Remove Online OAuth Login, Support API-Key-Only

## Goal

Remove all hard dependencies on Anthropic's OAuth infrastructure (`platform.claude.com` /
`claude.com`), replace `/login` / `/logout` with local API Key management, and ensure
the app works with any OpenAI-compatible API endpoint (DeepSeek, local LLM, proxy) without
phoning home.

## Current State (baseline)

The `.env` already bypasses most online paths — `ANTHROPIC_BASE_URL` points to DeepSeek,
`ANTHROPIC_AUTH_TOKEN` provides the bearer token directly. This causes
`isFirstPartyAnthropicBaseUrl()` to return `false`, which already skips Bootstrap, Settings
Sync, Remote Managed Settings, Datadog analytics, and Fast Mode checks. What remains is a
cleanup: removing the dead OAuth code and rewiring the user-facing login/logout surface.

---

## Phase 1 — Feature Gate (`ALLOW_OFFLINE`)

**Goal:** Add a feature flag so offline-aware code paths can be toggled at build time
without breaking upstream compatibility.

### 1.1 Register the flag

File: `src/types/bun-bundle.d.ts`

- Add type declaration for `feature('ALLOW_OFFLINE')` → `boolean`.

### 1.2 Add to dev shim

File: `plugins/bunBundleDev.ts`

- Add `ALLOW_OFFLINE` to the known-flags map (default `false`, enable via `FEATURE_FLAGS`).

### 1.3 Gate `isAnthropicAuthEnabled()`

File: `src/utils/auth.ts:100-149`

- Add early return: `if (feature('ALLOW_OFFLINE')) return false`.
- This single gate disables OAuth across the entire auth layer.

---

## Phase 2 — Rewire `/login` and `/logout`

**Goal:** Replace the OAuth-browser-flow with a local API Key configuration UI.

### 2.1 Rewrite `/login` command

Files: `src/commands/login/index.ts`, `src/commands/login/login.tsx`

Current behavior: Opens `ConsoleOAuthFlow` inside a `Dialog` → browser OAuth → stores
OAuth tokens + API Key.

New behavior:
- Render a `Dialog` with a text input for the API Key (masked).
- On submit: validate the key with a lightweight test call (single message to the
  configured `ANTHROPIC_BASE_URL`).
- Store the key via the existing `saveGlobalConfig` / keychain path.
- Bump `authVersion` in AppState (preserve the existing reactivity pattern).
- Remove the `ConsoleOAuthFlow` import.

### 2.2 Rewrite `/logout` command

Files: `src/commands/logout/index.ts`, `src/commands/logout/logout.tsx`

Current behavior (`performLogout`):
1. Flush telemetry
2. Remove API Key from keychain/config
3. Wipe OAuth tokens from secure storage
4. Clear OAuth token cache, trusted device tokens, betas, GrowthBook, remote managed
   settings, policy limits
5. Save config with `oauthAccount: undefined`

New behavior:
- Keep step 1 (flush telemetry) — still useful locally.
- Keep step 2 (remove API Key).
- Remove steps 3–5 (OAuth-specific cleanup).
- Add: clear any cached API Key from memory.

### 2.3 Rewire `claude auth` CLI

File: `src/cli/handlers/auth.ts`

- `authLogin()` (lines 112–230): Remove OAuth flow, replace with:
  - Read `ANTHROPIC_AUTH_TOKEN` or `ANTHROPIC_API_KEY` from env / prompt the user.
  - Store it to `~/.claude.json` → `primaryApiKey`.
- `authLogout()`: Remove OAuth-specific cleanup.
- `authStatus()` (lines 232–319): Remove OAuth token checks from the `loggedIn`
  computation; only check API Key presence.

### 2.4 Disable `/login` when using 3P services

File: `src/commands.ts:337`

Current check: `!isUsing3PServices()` gates login/logout registration.

No change needed — the existing logic already conditionally excludes login from the REPL
when using Bedrock/Vertex/Foundry. Offline mode behaves the same way.

---

## Phase 3 — Remove OAuth Service Layer

**Goal:** Delete the OAuth-specific modules and stub out references.

### 3.1 Remove OAuth service directory

Delete: `src/services/oauth/` (entire directory)

Contents:
- `index.ts` — `OAuthService` class (PKCE flow)
- `crypto.ts` — `codeVerifier`/`codeChallenge` generation
- `auth-code-listener.ts` — localhost HTTP server for redirect capture
- `client.ts` — token exchange, profile fetch
- `getOauthProfile.ts` — profile fetch helpers

### 3.2 Remove OAuth constants

Delete or gut: `src/constants/oauth.ts`

This file defines:
- Production/staging/local OAuth URLs (`platform.claude.com`, `claude.com`,
  `api.anthropic.com`)
- Client ID `9d1c250a-e61b-44d9-88ed-5944d1962f5e`
- Scope definitions
- `BASE_API_URL` used by bootstrap.ts

If `BASE_API_URL` is needed elsewhere, extract it to a separate constants file. Otherwise
delete the entire file.

### 3.3 Remove CLI auth handler OAuth paths

File: `src/cli/handlers/auth.ts`

- Remove `startOAuthFlow()` call and surrounding logic.
- Remove `OAuthService` import.
- Remove `installOAuthTokens()`, `refreshOAuthToken()`, `refreshAndClearOAuthToken()`.
- Remove OAuth-specific CLI flags (`--email`, `--sso`, `--console`, `--claudeai`).

### 3.4 Remove OAuth imports from main.tsx

File: `src/main.tsx`

- Remove any OAuth-related imports and initialization (profile loading, token refresh on
  startup).

---

## Phase 4 — Simplify `src/utils/auth.ts`

**Goal:** Strip OAuth token management; keep only API Key paths.

### 4.1 Remove OAuth token functions

Remove the following exports from `src/utils/auth.ts`:

| Function | Rationale |
|----------|-----------|
| `getClaudeAIOAuthTokens()` | No OAuth tokens stored |
| `saveClaudeAIOAuthTokens()` | No OAuth tokens to save |
| `clearClaudeAIOAuthTokens()` | No OAuth tokens to clear |
| `checkAndRefreshOAuthTokenIfNeeded()` | No token refresh needed for API keys |
| `refreshOAuthToken()` | No OAuth flow |
| `handleOAuth401Error()` | API Key 401s are terminal, not refreshable |
| `installOAuthTokens()` | No OAuth installation |
| `hasProfileScope()` | No OAuth scopes |
| `getOauthAccountInfo()` | No OAuth account |
| `isClaudeAISubscriber()` | Always `false` — all users are API users |
| `getSubscriptionType()` | Always `null` |

### 4.2 Simplify `getAuthTokenSource()` (lines 153–206)

After cleanup, only three sources remain:
1. `apiKeyHelper` (bare mode)
2. `ANTHROPIC_AUTH_TOKEN` env var
3. `ANTHROPIC_API_KEY` env var / keychain

Remove: `CLAUDE_CODE_OAUTH_TOKEN`, OAuth FD token, keychain OAuth token branches.

### 4.3 Simplify `getAnthropicApiKeyWithSource()` (lines 226–348)

Keep only: env var, apiKeyHelper, keychain `primaryApiKey`.

Remove: OAuth-based key creation for Console users.

### 4.4 Remove `isManagedOAuthContext()`

File: `src/utils/auth.ts`

This function checks for `CLAUDE_CODE_OAUTH_TOKEN` / CCR disk paths. No longer relevant.

### 4.5 Remove `withOAuth401Retry()`

File: `src/utils/http.ts:115`

Replace callers with standard `fetch()` — API Key 401s are terminal errors, not retryable.

---

## Phase 5 — Clean Up Cascading Dependencies

**Goal:** Remove all remaining imports and references to deleted functions.

### 5.1 `src/services/api/client.ts`

- Line 5: Remove `checkAndRefreshOAuthTokenIfNeeded` import.
- Line 132: Remove `await checkAndRefreshOAuthTokenIfNeeded()` call. API Key is set once
  at startup.
- Lines 301–305: Remove subscriber vs API-key branching. Always use API Key path.

### 5.2 `src/services/api/bootstrap.ts`

- Already skipped (line 48: `getAPIProvider() !== 'firstParty'`).
- Remove the entire file or guard with `if (feature('ALLOW_OFFLINE')) return null` at
  the top.

### 5.3 `src/services/settingsSync/index.ts`

- Already skipped (line 213: `getAPIProvider() !== 'firstParty' ||
  !isFirstPartyAnthropicBaseUrl()`).
- No change needed.

### 5.4 `src/services/remoteManagedSettings/syncCache.ts`

- Already skipped (lines 53–58).
- No change needed.

### 5.5 `src/services/analytics/datadog.ts`

- Already skipped (line 169: `getAPIProvider() !== 'firstParty'`).
- No change needed.

### 5.6 `src/utils/api.ts`

- Lines 200–201: Already gated behind `isFirstPartyAnthropicBaseUrl()`. Verify no
  additional changes needed.

### 5.7 `src/utils/toolSearch.ts`

- Lines 301–302: Already gated behind `isFirstPartyAnthropicBaseUrl()`. No change needed.

### 5.8 `src/utils/fastMode.ts`

- Line 113: Already skipped for non-first-party.
- Review effort configuration for offline models.

### 5.9 `src/hooks/useManageMCPConnections.ts`

- `authVersion` reactivity still works — bumped on login (new API Key stored). No change
  needed to the hook itself.

### 5.10 `src/hooks/useVoiceEnabled.ts`

- `authVersion` reactivity preserved. Voice can be configured via local settings without
  OAuth.

### 5.11 `src/commands/logout/logout.tsx` — `clearAuthRelatedCaches()`

- Remove OAuth-specific cache keys: `oauthTokens`, `trustedDeviceTokens`, `betas`.
- Keep local caches that are still relevant.

---

## Phase 6 — Verify

### 6.1 TypeScript compilation

```bash
bun run typecheck
```

Fix all type errors from removed imports/exports.

### 6.2 Runtime smoke test

```bash
bun run start -- -p "hello"
```

### 6.3 Verify commands

- `/login` → prompts for API Key, stores it, bumps `authVersion`.
- `/logout` → clears API Key, clear caches.
- `/doctor` → reports offline mode, no OAuth errors.
- All offline tools (Bash, Read, Write, Grep, Glob) → work normally.

---

## Summary of Files Modified/Deleted

### Deleted

| Path | Reason |
|------|--------|
| `src/services/oauth/` | Entire OAuth service layer |
| `src/constants/oauth.ts` | OAuth URLs, client IDs, scopes |

### Modified

| Path | Change |
|------|--------|
| `src/types/bun-bundle.d.ts` | Add `ALLOW_OFFLINE` type |
| `plugins/bunBundleDev.ts` | Add `ALLOW_OFFLINE` flag |
| `src/utils/auth.ts` | Remove OAuth functions, simplify to API-Key-only |
| `src/utils/http.ts` | Remove `withOAuth401Retry()` |
| `src/cli/handlers/auth.ts` | Replace OAuth with API Key management |
| `src/commands/login/index.ts` | Rewire to API Key input |
| `src/commands/login/login.tsx` | Rewrite UI for API Key input |
| `src/commands/logout/index.ts` | Simplify logout |
| `src/commands/logout/logout.tsx` | Remove OAuth cleanup |
| `src/services/api/client.ts` | Skip OAuth token refresh |
| `src/services/api/bootstrap.ts` | Early-return in offline mode |
| `src/main.tsx` | Remove OAuth init |
| `src/hooks/useManageMCPConnections.ts` | Review authVersion usage |
| `src/hooks/useVoiceEnabled.ts` | Review authVersion usage |

---

## Risks & Notes

1. **authVersion reactivity** — Phase 2 preserves the `authVersion++` bump on login.
   This ensures MCP connection management, voice hooks, and any other `authVersion`
   consumers continue to react to login/logout events without modification.

2. **GrowthBook** — Currently fetches feature flag configuration from the remote
   Bootstrap API. In offline mode this is already skipped. If offline feature flags are
   needed, add a local JSON fallback in `src/services/analytics/`.

3. **Keychain dependency** — API Key storage uses `@ant/secure-storage` (macOS Keychain)
   or `~/.claude.json`. The latter works cross-platform. The stubs already provide no-op
   replacements for `@ant/*` packages, so keychain failures won't crash the app.

4. **Rollback** — Using a `feature('ALLOW_OFFLINE')` gate in Phase 1 means all changes
   can be compile-time reverted by omitting the flag. No permanent damage to the upstream
   code architecture.
