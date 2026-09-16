---
name: http-swagger-integration
description: >-
  Use when the user provides an OpenAPI/Swagger schema URL and wants to implement
  API integrations. Fetches the schema, maps endpoints, handles auth, and generates
  typed client code.
---

# HTTP Swagger Integration

When the user shares an OpenAPI/Swagger schema URL, fetch it and build a typed API client.

## Workflow

1. **Fetch the schema** — `GET <schema-url>` and parse the JSON (OpenAPI 3.x or Swagger 2.x).
2. **Extract base URL** — use `servers[0].url` (OpenAPI 3) or `host` + `basePath` + `schemes[0]` (Swagger 2).
3. **Map endpoints** — for each `paths` entry extract: method, path, `operationId`, `summary`, parameters, request body schema, response schemas, and security requirements.
4. **Resolve auth** — see Auth section below.
5. **Generate client** — create typed functions per operation (see Code shape).
6. **Document** — add an `API.md` listing each operation, its inputs, outputs, and auth requirement.

## Auth

| Scheme | Where it lives | How to apply |
|--------|---------------|--------------|
| `http / basic` | `Authorization: Basic base64(user:pass)` | Encode at call time; never hard-code credentials |
| `http / bearer` | `Authorization: Bearer <token>` | Accept token as env var or parameter |
| `apiKey / header` | Custom header named in `name` field | Inject from env or config |
| `apiKey / query` | Query string param named in `name` field | Append to URL |
| `oauth2` | Bearer token after OAuth flow | Document required scopes; implement token fetch if asked |

Read credentials from **environment variables** or a config object — never inline secrets.

## Code shape

Generate a module (e.g. `src/api/<title>.ts`) with:

```ts
// One typed function per operationId
export async function <operationId>(
  params: <ParamsType>,
  auth: <AuthType>,
): Promise<<ResponseType>> {
  const url = `${BASE_URL}<path>`;
  const response = await fetch(url, {
    method: '<METHOD>',
    headers: { ...authHeaders(auth), 'Content-Type': 'application/json' },
    body: params.body ? JSON.stringify(params.body) : undefined,
  });
  if (!response.ok) throw new Error(`<operationId> failed: ${response.status}`);
  return response.json() as Promise<<ResponseType>>;
}
```

- Derive **TypeScript types** from the JSON Schema in the spec (inline interfaces are fine for small schemas).
- Path parameters: interpolate into the URL string.
- Query parameters: append with `URLSearchParams`.

## Example (from the OneReach HTTP Swagger schema)

Schema URL: `https://files.staging.api.onereach.ai/.../swaggerSchema.json`

```ts
const BASE_URL = 'https://em.staging.api.onereach.ai/http/5d2035e4-6da3-4280-8b19-8337492d7210';

interface TestSwaggerResponse { status: number }
interface BasicAuth { username: string; password: string }

function authHeaders(auth: BasicAuth): Record<string, string> {
  const encoded = btoa(`${auth.username}:${auth.password}`);
  return { Authorization: `Basic ${encoded}` };
}

export async function httpswaggerGetTestSwagger(auth: BasicAuth): Promise<TestSwaggerResponse> {
  const response = await fetch(`${BASE_URL}/test-swagger`, {
    headers: authHeaders(auth),
  });
  if (!response.ok) throw new Error(`httpswaggerGetTestSwagger failed: ${response.status}`);
  return response.json();
}
```

## Checklist

- [ ] Schema fetched and parsed
- [ ] Base URL extracted
- [ ] One typed function per `operationId`
- [ ] Auth handled via env vars / config (no hard-coded secrets)
- [ ] Path and query params correctly interpolated
- [ ] `API.md` created: lists endpoints, params, responses, auth
- [ ] Error handling on non-2xx responses
