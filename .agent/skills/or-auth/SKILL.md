---
name: or-auth
description: >-
  Adds OneReach token-based authorization to a frontend web app. Use when the user asks to protect
  a page with OneReach authorization, add OneReach auth, secure a page with OneReach login, or
  mentions VITE_AUTH_URL / VITE_SDK_API_URL auth flow. Covers React, Vue, Svelte, Angular, and vanilla JS.
---

# OneReach Token Authorization

## Steps

1. **Install dependency**
   ```bash
   npm install js-cookie
   npm install -D @types/js-cookie
   ```

2. **Copy `lib/auth.ts`** — use the implementation below verbatim, saved to `lib/auth.ts` (or `src/lib/auth.ts`).

3. **Add env variables** to `.env`:
   ```
   VITE_SDK_API_URL=https://sdkapi.qa.api.onereach.ai
   VITE_AUTH_URL=https://auth.qa.onereach.ai
   VITE_TOKEN_COOKIE_NAME=or      # optional, default: "or"
   ```
   For non-Vite bundlers replace `VITE_` with the appropriate prefix (`NEXT_PUBLIC_`, `REACT_APP_`, etc.) and update `import.meta.env.*` reads in `auth.ts`.

4. **Add TypeScript env types** to `vite-env.d.ts` (Vite projects only):
   ```typescript
   /// <reference types="vite/client" />
   interface ImportMetaEnv {
     readonly VITE_SDK_API_URL?: string;
     readonly VITE_AUTH_URL?: string;
     readonly VITE_TOKEN_COOKIE_NAME?: string;
   }
   interface ImportMeta { readonly env: ImportMetaEnv; }
   ```

5. **Wire up `authorizePage`** — call it once when the protected page/component initialises.

---

## `lib/auth.ts`

```typescript
import Cookies from 'js-cookie';

export interface SdkSessionValidationResponse {
  allow?: boolean;
  tokenType?: string;
  username?: string;
  accountId?: string;
  userId?: string;
  role?: string;
  twoFactorEnabled?: boolean;
  multiUserId?: string;
  expire?: number;
  token?: string;
}

const AUTH_REDIRECT_STORAGE_KEY = 'flow-auth-redirect-path';

function maskToken(token: string) {
  if (!token) return 'missing';
  if (token.length <= 10) return `${token.slice(0, 2)}***${token.slice(-2)}`;
  return `${token.slice(0, 6)}...${token.slice(-4)}`;
}

function logAuthDebug(message: string, details?: Record<string, unknown>) {
  if (details) { console.info(`[auth] ${message}`, details); return; }
  console.info(`[auth] ${message}`);
}

const DEFAULT_TOKEN_COOKIE_NAME = 'token';

export function getTokenCookieName() {
  return import.meta.env.VITE_TOKEN_COOKIE_NAME?.trim() || DEFAULT_TOKEN_COOKIE_NAME;
}

export function getSdkApiUrl() {
  return import.meta.env.VITE_SDK_API_URL?.trim() || '';
}

export function getAuthUrl() {
  return import.meta.env.VITE_AUTH_URL?.trim() || '';
}

/**
 * Reads the auth token from the cookie.
 * The cookie value may be a raw token string OR a JSON-stringified object with a `token` field.
 */
export function getAuthToken() {
  const cookieName = getTokenCookieName();
  const raw = Cookies.get(cookieName) || '';
  let token = raw;
  if (raw) {
    try {
      const parsed = JSON.parse(raw) as Record<string, unknown>;
      if (typeof parsed?.token === 'string') token = parsed.token;
    } catch {
      // not JSON — use raw value directly
    }
  }
  logAuthDebug('read auth token', { cookieName, tokenFound: Boolean(token), tokenPreview: maskToken(token) });
  return token;
}

/** Calls GET {sdkApiUrl}/auth/token and returns the session payload on success. Throws on failure. */
export async function validateSdkSession(token?: string): Promise<SdkSessionValidationResponse> {
  const sdkApiUrl = getSdkApiUrl();
  if (!sdkApiUrl) {
    logAuthDebug('missing SDK API URL');
    throw new Error('Set VITE_SDK_API_URL before validating the session.');
  }
  const validationUrl = `${sdkApiUrl}/auth/token`;
  logAuthDebug('start session validation request', { validationUrl, tokenPreview: maskToken(token || '') });
  const response = await fetch(validationUrl, {
    headers: { Accept: 'application/json', Authorization: token ?? '' },
  });
  const text = await response.text();
  let payload: SdkSessionValidationResponse | null = null;
  try { payload = text ? (JSON.parse(text) as SdkSessionValidationResponse) : null; } catch { payload = null; }
  logAuthDebug('received session validation response', { validationUrl, status: response.status, allow: payload?.allow, username: payload?.username });
  if (!response.ok) throw new Error(`Session validation failed: ${response.status} ${response.statusText}${text ? ` - ${text}` : ''}`);
  if (!payload?.allow) throw new Error(text || 'Session validation response did not allow access.');
  logAuthDebug('session validation succeeded', { sdkApiUrl, allow: payload.allow, username: payload.username });
  return payload;
}

export function clearRedirectToAuthState() {
  sessionStorage.removeItem(AUTH_REDIRECT_STORAGE_KEY);
  logAuthDebug('cleared auth redirect state');
}

export function hasRedirectedToAuthForCurrentPage() {
  return sessionStorage.getItem(AUTH_REDIRECT_STORAGE_KEY) === window.location.href;
}

function rememberRedirectToAuth() {
  sessionStorage.setItem(AUTH_REDIRECT_STORAGE_KEY, window.location.href);
}

/**
 * Redirects to VITE_AUTH_URL with `redirectPath` set to the current URL.
 * Returns false without redirecting if this page already triggered a redirect —
 * prevents infinite loops when the token is still invalid after returning from auth.
 */
export function redirectToAuth() {
  const authUrl = getAuthUrl();
  if (!authUrl) throw new Error('Set VITE_AUTH_URL before redirecting to auth.');
  const redirectPath = window.location.href;
  if (hasRedirectedToAuthForCurrentPage()) {
    logAuthDebug('skip redirect — this page already redirected once', { redirectPath });
    return false;
  }
  rememberRedirectToAuth();
  const redirectTarget = new URL(authUrl, window.location.origin);
  redirectTarget.searchParams.set('redirectPath', redirectPath);
  logAuthDebug('redirect to auth', { authUrl, redirectTarget: redirectTarget.toString(), redirectPath });
  window.location.assign(redirectTarget.toString());
  return true;
}
```

---

## `authorizePage` — single entry point

```typescript
import { clearRedirectToAuthState, getAuthToken, redirectToAuth, validateSdkSession } from './lib/auth';

async function authorizePage(
  onSuccess: (username: string) => void,
  onError: (message: string) => void,
) {
  const token = getAuthToken() || undefined;
  try {
    const session = await validateSdkSession(token);
    clearRedirectToAuthState();
    onSuccess(session.username ?? 'this user');
  } catch (error) {
    const message = error instanceof Error ? error.message : 'Your session could not be verified';
    onError(`${message} Redirecting to sign in...`);
    const redirected = redirectToAuth();
    if (!redirected) onError(`${message} Redirect skipped to avoid a loop.`);
  }
}
```

---

## Framework wiring

### React
```typescript
import { useEffect } from 'react';
useEffect(() => {
  authorizePage(
    (username) => { /* set authorized state */ },
    (message) => { /* set error state */ },
  );
}, []);
```

### Vue 3
```typescript
import { onMounted } from 'vue';
onMounted(() => authorizePage((username) => { /* authorized */ }, (message) => { /* error */ }));
```

### Svelte
```typescript
import { onMount } from 'svelte';
onMount(() => authorizePage((username) => { /* authorized */ }, (message) => { /* error */ }));
```

### Angular
```typescript
ngOnInit() {
  authorizePage((username) => { /* authorized */ }, (message) => { /* error */ });
}
```

### Vanilla JS / HTML
```html
<div id="auth-status">Validating your session...</div>
<div id="protected" hidden>Protected content here</div>
<script type="module">
  import { authorizePage } from './lib/auth.js';
  authorizePage(
    () => { document.getElementById('auth-status').hidden = true; document.getElementById('protected').hidden = false; },
    (message) => { document.getElementById('auth-status').textContent = message; },
  );
</script>
```

---

## How it works

1. Reads auth token from browser cookie (raw string or JSON `{ token }` shape).
2. Sends `GET {VITE_SDK_API_URL}/auth/token` with `Authorization: <token>`.
3. Access granted when response is 2xx **and** `payload.allow === true`.
4. On failure → redirects to `VITE_AUTH_URL?redirectPath=<current-url>`.
5. `sessionStorage` guard prevents infinite redirect loops (`clearRedirectToAuthState()` resets it after success).
