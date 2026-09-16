---
name: rwc-component
description: >-
  Use when building or embedding a web app inside OneReach Rich Web Chat (RWC) via iframe.
  Covers postMessage event bus (host ↔ iframe), mandatory rwc-integration.md for RWC-side work, and README for developers.
---

# RWC Component (Rich Web Chat embedding)

RWC loads **customer web apps inside an iframe**. Your app runs in a **child origin**; RWC runs in the **parent**. All coordination uses **`window.postMessage`** — there is no shared JS context.

## Goals

1. Implement a small **event bus** on `window` that **sends** and **receives** structured messages over `postMessage`.
2. **Validate** message shape (channel / envelope) before handling data.
3. Create **`rwc-integration.md` at the project root** (required) — handoff doc for the **RWC / host team**: what they must implement, how the integration works end-to-end, and operational notes (see section below).
4. Update **`README.md`** (or a clear section in it) for **developers** cloning the repo: what the app is, how to run it, and pointer to `rwc-integration.md` for embedding.

## Architecture

```text
┌─────────────────────────────────────┐
│  RWC host (parent window)           │
│  postMessage → iframe.contentWindow │
│  ← postMessage from iframe         │
└──────────────┬──────────────────────┘
               │ iframe
┌──────────────▼──────────────────────┐
│  Your embedded app (child)          │
│  parent = window.parent             │
└─────────────────────────────────────┘
```

- **Outbound (app → RWC):** `window.parent.postMessage(payload, '*')`
- **Inbound (RWC → app):** `window.addEventListener('message', handler)`

Use a **single envelope** for every message so both sides stay consistent:

```ts
type RwcMessage<T = unknown> = {
  /** Namespace avoids collisions with third-party scripts */
  channel: 'onereach:rwc';
  /** Event name, e.g. `ready`, `user-message`, `resize` */
  type: string;
  payload?: T;
  /** Optional correlation id for request/response */
  requestId?: string;
};
```

Use `'*'` as the `postMessage` target so messages work regardless of parent URL (RWC host may vary by environment).

## Event bus interface (implement in the embedded app)

Provide a minimal module, e.g. `src/lib/rwc-event-bus.ts`:

1. **`initRwcEventBus()`** — registers a single `message` listener on `window`.
2. **`onRwcMessage(type, handler)`** — subscribe; handler receives `(payload, rawMessageEvent)`.
3. **`postToRwc(type, payload?, requestId?)`** — builds envelope and calls `parent.postMessage(..., '*')`.
4. **`disposeRwcEventBus()`** — remove listener (tests / HMR).

**Handling rules:**

- Reject if `typeof event.data !== 'object'` or `event.data?.channel !== 'onereach:rwc'`.
- Never `eval` or `new Function` on message data.

**Handshake:** After your app mounts, post `{ channel: 'onereach:rwc', type: 'app-ready', payload: { version: '1.0.0' } }` (or the contract RWC documents for your tenant). Listen for a parent `rwc-ready` or equivalent if the product defines one — align names with existing RWC docs or internal specs when available.

## Parent (RWC) expectations

When changing **host-side** behavior, that code usually lives outside this Creator project; in **iframe-only** tasks, focus on:

- Correct **postMessage** target (child `contentWindow`) from host docs.
- **CSP** and **frame-ancestors** / **X-Frame-Options** on the embedded app so RWC can load it.

If the user provides **exact message type names** from RWC product docs, use those names instead of placeholders in code, `rwc-integration.md`, and README.

## `rwc-integration.md` (required — RWC / host side)

**Always add this file at the project root** when this skill applies. It is the contract for people implementing or operating **Rich Web Chat (parent window)**, not only iframe app code. Write it in clear prose; use tables where helpful.

Include at least:

1. **Purpose** — One paragraph: what the embedded app does and how RWC loads it (iframe URL / deployment surface if known).
2. **Architecture** — Short diagram or bullet flow: parent ↔ `postMessage` ↔ iframe, `channel: 'onereach:rwc'`, and who initiates which messages.
3. **What RWC must implement**
   - When and how the host **creates the iframe** (src, sandbox attributes if any, sizing).
   - How the host **sends** messages to the child (`iframe.contentWindow.postMessage(..., '*')` or documented target) and **listens** for child messages (`window.addEventListener('message', ...)`).
   - **Handshake order** (e.g. child posts `app-ready` first, then host posts `rwc-ready`, or whatever matches product spec — state explicitly if unknown and mark as TBD).
4. **Message contract** — Table: `type`, direction (**host → iframe** / **iframe → host**), payload shape, when it fires, and idempotency / retries if relevant.
5. **Session / auth / PII** — What the host may pass in payloads (e.g. correlation ids, **no** long-lived secrets in postMessage); link or reference internal security docs if the user supplied them.
6. **CSP / framing** — What the embedded app expects (`frame-ancestors`, headers) so RWC can load it; any host-side CSP notes.
7. **Environments** — Staging vs production differences (origins, feature flags) if known; otherwise “TBD — align with RWC platform team”.
8. **Verification** — Checklist for QA on the RWC side (DevTools, expected console logs, sample message sequence).
9. **Contacts / next steps** — Placeholder for owning team or ticket if not specified.

Keep **iframe app implementation** (TypeScript modules, npm scripts) in README; keep **RWC platform behavior** in `rwc-integration.md`.

## README (developers)

In `README.md` (or a dedicated section), cover:

1. **What this app is** and that it is designed for **iframe embedding in RWC**; link to **`rwc-integration.md`** for host-side work.
2. **Local dev** — run commands, env vars, and how to mock parent postMessage (e.g. small HTML shell).
3. **Troubleshooting** — DevTools → Application → Frames and console if messages do not arrive.

## Implementation checklist

- [ ] Event bus module with `onereach:rwc` channel validation
- [ ] App emits **ready** after UI is safe to receive host commands
- [ ] Typed handlers for product-specific events (fill from RWC spec when known)
- [ ] **`rwc-integration.md`** at repo root with RWC-side requirements and full integration narrative
- [ ] **`README.md`** updated for developers with pointer to `rwc-integration.md`
- [ ] No secrets in postMessage payloads; auth via short-lived tokens or host-established session patterns only as documented by security

## Testing

- Unit-test the bus with `MessageEvent` mocks (data shape).
- Manual: load app inside a test parent page that mirrors RWC postMessage sequence.
