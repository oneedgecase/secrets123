---
name: or-sdk
description: >-
  Integrates OneReach platform services via @or-sdk/* packages. Use when the user asks to use
  OneReach Files, users, accounts, agents, bots, flows, discovery, authorizer, permissions, or
  other @or-sdk clients. Covers install + Creator wiring; package README is the API source of truth.
---

# OneReach SDK (`@or-sdk/*`) in Creator

Platform SDK packages live under the `@or-sdk` npm scope. **Do not** copy method docs into
Creator skills. After installing a package, read its README and follow it.

## Hard rules (read first)

1. **Detect project type before asking for secrets.**
   - **API project** (`functions/`, `openapi.yaml`, Flow Step): auth = `step.config.authorization`.
     **Never** ask the user for `OR_TOKEN`. **Never** put `OR_TOKEN` in Secrets instructions.
   - **Frontend project** (Vite/React/Vue/…): auth = `OR_TOKEN` from Secrets.
2. Prefer **direct service URL** secrets (`FILES_URL`, etc.) over `DISCOVERY_URL`.
   When telling the user which Secrets to add, **always include the concrete value** from the
   **Current environment service URLs** block in your system instructions (key + exact URL).
   Example: `FILES_URL` = `https://files-api.svc.qa.api.onereach.ai` — so they can paste it
   into Secrets without guessing. Never leave `<env>` placeholders in the user-facing setup steps.
3. Follow package README pitfalls (for Files: **no leading `/`** on keys/prefixes; upload
   prefixes must end with `/`, e.g. `uploads/images/` — never `/uploads/images`).
4. In the final reply, only ask the user for secrets that apply to **this** project type.

## Workflow

1. Pick the package for the task (see catalog below).
2. Install:
   ```bash
   npm install @or-sdk/<package>
   ```
3. **Read** `node_modules/@or-sdk/<package>/README.md` — construct, when to use / when not,
   pitfalls, method map. Types live in `dist/types/` and `src/`.
4. Wire auth + **direct service URL** using the rules below for this project type.

If the README is missing from `node_modules`, the package publish surface is wrong — still prefer
types under `node_modules/@or-sdk/<package>/` over inventing APIs.

## Construct: prefer direct service URL

When the service URL is known, pass it explicitly (e.g. `filesApiUrl`, `sdkUrl` — see package README).
This is **faster** and avoids discovery load. Use `discoveryUrl` only when the direct URL is unavailable.

```typescript
// Recommended
new Client({ token, filesApiUrl: process.env.FILES_URL });

// Fallback
new Client({ token, discoveryUrl: process.env.DISCOVERY_URL });
```

## Creator wiring

**NEVER** create or use `.env*` files. Credentials go on the project **Secrets** page only.
Restart **Preview** after saving secrets.

### Secret matrix

| Secret | API project | Frontend project | Purpose |
|--------|-------------|------------------|---------|
| `FILES_URL` / package `*_URL` | Yes (preferred) | Yes (preferred) | Direct API base URL |
| `DISCOVERY_URL` | Fallback only | Fallback only | Service discovery |
| `OR_TOKEN` | **No — forbidden** | Yes | Browser bearer token |

### API projects (`@onereach/flow-sdk`) — default for `functions/**`

**Do not ask for `OR_TOKEN`.** Tell the user only about the service URL secret (e.g. `FILES_URL`).

Pass the Flow Step instance into the handler. Generated `index.mjs` calls
`handler(event, this)` — write handlers as:

```javascript
// functions/{slug}/uploads/images/post.mjs
import { Files } from '@or-sdk/files';

export default async function (event, step) {
  const client = new Files({
    token: step.config.authorization,
    filesApiUrl: process.env.FILES_URL,
  });
  // follow package README for methods / path rules
  return { ok: true };
}
```

Do **not** edit `index.mjs` manually — it is regenerated on save.
Never return or log `step.config.authorization`.

**User setup message for API + Files (copy this shape, fill URL from Current environment service URLs):**
> Add this secret on the **Secrets** page, then restart Preview:
> - `FILES_URL` = `https://files-api.svc.…` ← paste the exact `FILES_URL` value from **Current environment service URLs**
>
> Do **not** add `OR_TOKEN` — the API runtime already provides authorization.

### Frontend (Vite)

Expose the secrets you use (merge into existing `vite.config.ts`). Example for Files:

```typescript
define: {
  'import.meta.env.FILES_URL': JSON.stringify(process.env.FILES_URL ?? ''),
  'import.meta.env.OR_TOKEN': JSON.stringify(process.env.OR_TOKEN ?? ''),
},
```

```typescript
new SomeClient({
  token: () => import.meta.env.OR_TOKEN,
  filesApiUrl: import.meta.env.FILES_URL,
});
```

Exact option names (`filesApiUrl`, `sdkUrl`, …) are in that package's README.

## Package catalog (Creator-relevant)

Install only what you need. Read each package README before coding.

| Package | Use for |
|---------|---------|
| `@or-sdk/files` | Upload / download / list / delete account files & folders |
| `@or-sdk/authorizer` | Auth helpers (not a package named `@or-sdk/auth`) |
| `@or-sdk/discovery` | Resolve service URLs by service key (when direct URLs are unknown) |
| `@or-sdk/users` | Users / profiles |
| `@or-sdk/accounts` | Account management |
| `@or-sdk/account-settings` | Account settings |
| `@or-sdk/agents` | Agents API |
| `@or-sdk/bots` | Bots |
| `@or-sdk/flows` | Flows |
| `@or-sdk/permissions` | Permissions |
| `@or-sdk/api-tokens` | API tokens |
| `@or-sdk/contacts` | Contacts |
| `@or-sdk/settings` | Settings |
| `@or-sdk/mcp-tools` | MCP servers / OAuth packages |
| `@or-sdk/idw` | IDW REST API |
| `@or-sdk/idw-skill` | IDW Web Skill iframe `postMessage` protocol |

There is **no** `@or-sdk/auth` — use `@or-sdk/authorizer` or the separate `or-auth` skill for
cookie/login page protection when that is the task.

## Checklist

- [ ] Correct `@or-sdk/<package>` installed
- [ ] Agent read `node_modules/@or-sdk/<package>/README.md`
- [ ] Direct service URL on Secrets when known; `DISCOVERY_URL` only as fallback
- [ ] **API project:** `step.config.authorization` only — **no** `OR_TOKEN` asked or documented
- [ ] **Frontend:** `OR_TOKEN` + Vite `define` when needed
- [ ] Files keys/prefixes have no leading `/`; upload prefixes end with `/`
- [ ] No duplicated method docs in project files — follow the package README
