---
name: openapi-fetch
description: Type-safe fetch client for OpenAPI schemas using openapi-fetch and openapi-typescript. Use when writing, reviewing, or refactoring code that makes API calls with openapi-fetch, creates API clients from OpenAPI schemas, or uses openapi-typescript generated types. Triggers on imports of openapi-fetch or createClient usage with OpenAPI paths types.
license: MIT
metadata:
  author: prescottprue
  version: "1.0.0"
---

# openapi-fetch

Comprehensive guide for `openapi-fetch` — a type-safe fetch client that uses OpenAPI schemas for zero-overhead type checking at build time.

## When to Apply

Reference these guidelines when:
- Creating API clients from OpenAPI schemas
- Making type-safe HTTP requests (GET, POST, PUT, DELETE, PATCH, etc.)
- Implementing middleware for auth, logging, caching, or error handling
- Configuring query/body/path serialization
- Testing API clients with mocks
- Integrating with React, Vue, Svelte, Next.js, or Nuxt

## Installation

```bash
npm i openapi-fetch
npm i -D openapi-typescript typescript
```

### Schema Generation

Generate TypeScript types from your OpenAPI schema:

```bash
npx openapi-typescript ./path/to/api/v1.yaml -o ./src/lib/api/v1.d.ts
```

This produces a `paths` type export used to type the client.

**Recommended:** Enable `noUncheckedIndexedAccess` in `tsconfig.json` for enhanced type safety.

## Creating a Client

```typescript
import createClient from "openapi-fetch";
import type { paths } from "./my-openapi-3-schema";

const client = createClient<paths>({
  baseUrl: "https://myapi.dev/v1/",
});
```

### `createClient` Options

| Option | Type | Description |
|--------|------|-------------|
| `baseUrl` | `string` | Prefix all fetch URLs |
| `fetch` | `(input: Request) => Promise<Response>` | Custom fetch instance (default: `globalThis.fetch`) |
| `Request` | `typeof Request` | Custom Request constructor |
| `querySerializer` | `QuerySerializer \| QuerySerializerOptions` | Global query param serializer |
| `bodySerializer` | `BodySerializer` | Global body serializer |
| `pathSerializer` | `PathSerializer` | Global path param serializer |
| `headers` | `HeadersOptions` | Default headers for all requests |
| `requestInitExt` | `Record<string, unknown>` | Extension object passed as 2nd argument to fetch |
| *(any fetch option)* | | Any valid `RequestInit` option (`mode`, `cache`, `credentials`, `signal`, etc.) |

## Making Requests

All HTTP methods are available: `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `OPTIONS`, `HEAD`, `TRACE`.

```typescript
// GET request
const { data, error, response } = await client.GET("/blogposts/{post_id}", {
  params: {
    path: { post_id: "123" },
    query: { format: "json" },
  },
});

// POST request
const { data, error } = await client.POST("/blogposts", {
  body: {
    title: "My Post",
    content: "Hello world",
  },
});

// PUT request
const { data, error } = await client.PUT("/blogposts/{post_id}", {
  params: { path: { post_id: "123" } },
  body: { title: "Updated Title" },
});

// DELETE request
const { data, error } = await client.DELETE("/blogposts/{post_id}", {
  params: { path: { post_id: "123" } },
});
```

### Response Structure

Every method returns an object with:
- `data` — Present only on 2xx responses. Typed from the schema's success response.
- `error` — Present only on 4xx/5xx responses. Typed from the schema's error response.
- `response` — The raw `Response` object with status, headers, etc.

`data` and `error` are **mutually exclusive** — check one to narrow the type:

```typescript
const { data, error } = await client.GET("/blogposts/{post_id}", {
  params: { path: { post_id: "123" } },
});

if (error) {
  // error is typed, data is undefined
  console.error(error);
  return;
}
// data is typed, error is undefined
console.log(data.title);
```

### Fetch Options Per Request

| Option | Type | Description |
|--------|------|-------------|
| `params` | `{ path?, query?, header?, cookie? }` | Path, query, header, and cookie parameters |
| `body` | `object` | Request body (typed from schema) |
| `parseAs` | `"json" \| "text" \| "arrayBuffer" \| "blob" \| "stream"` | Response parsing method (default: `"json"`) |
| `querySerializer` | `QuerySerializer \| QuerySerializerOptions` | Per-request query serializer |
| `bodySerializer` | `BodySerializer` | Per-request body serializer |
| `pathSerializer` | `PathSerializer` | Per-request path serializer |
| `baseUrl` | `string` | Override base URL for this request |
| `fetch` | `fetch` | Override fetch for this request |
| `headers` | `HeadersOptions` | Per-request headers (merged with client defaults) |
| `middleware` | `Middleware[]` | Per-request middleware |
| *(any fetch option)* | | Any valid `RequestInit` option |

### Generic `request` Method

For dynamic HTTP methods:

```typescript
const { data, error } = await client.request("GET", "/blogposts/{post_id}", {
  params: { path: { post_id: "123" } },
});
```

### `parseAs` Option

Control how the response body is parsed:

```typescript
// Get response as text
const { data } = await client.GET("/file", { parseAs: "text" });

// Get response as blob
const { data } = await client.GET("/image", { parseAs: "blob" });

// Get response as stream
const { data } = await client.GET("/large-file", { parseAs: "stream" });

// Get response as arrayBuffer
const { data } = await client.GET("/binary", { parseAs: "arrayBuffer" });
```

## Middleware

Middleware intercepts requests and responses for all fetches. Use for auth, logging, caching, error handling, etc.

### Middleware Interface

```typescript
import type { Middleware } from "openapi-fetch";

const myMiddleware: Middleware = {
  async onRequest({ request, schemaPath, params, id, options }) {
    // Modify or return a new Request
    // Return a Response to short-circuit (skip fetch + remaining onRequest handlers)
    // Return undefined to skip this middleware
    return request;
  },
  async onResponse({ request, response, schemaPath, params, id, options }) {
    // Modify or return a new Response
    // Return undefined to skip
    return response;
  },
  async onError({ request, error, schemaPath, params, id, options }) {
    // Return nothing: logs without re-throwing
    // Return an Error: throws the modified error
    // Return a Response: treats the request as successful
  },
};
```

### Callback Parameters

| Name | Type | Available in | Description |
|------|------|-------------|-------------|
| `request` | `Request` | all | Current Request object |
| `response` | `Response` | `onResponse` | Response from endpoint |
| `error` | `unknown` | `onError` | Error thrown by fetch |
| `schemaPath` | `string` | all | Original OpenAPI path (e.g., `"/blogposts/{post_id}"`) |
| `params` | `object` | all | Original params object (`query`, `header`, `path`, `cookie`) |
| `id` | `string` | all | Unique ID for this request |
| `options` | `MergedOptions` | all | Read-only client options |

### Registering and Ejecting

```typescript
client.use(myMiddleware);       // Register
client.use(mw1, mw2, mw3);     // Register multiple
client.eject(myMiddleware);     // Unregister
```

### Execution Order

- `onRequest` handlers run in registration order
- `onResponse` handlers run in **reverse** order

### Auth Middleware Pattern

```typescript
import type { Middleware } from "openapi-fetch";

let accessToken: string | undefined = undefined;

const authMiddleware: Middleware = {
  async onRequest({ request }) {
    if (!accessToken) {
      const authRes = await someAuthFunc();
      accessToken = authRes.accessToken;
    }
    request.headers.set("Authorization", `Bearer ${accessToken}`);
    return request;
  },
};

client.use(authMiddleware);
```

### Conditional Middleware (Skip Certain Routes)

```typescript
const UNPROTECTED_ROUTES = ["/v1/login", "/v1/logout", "/v1/public/"];

const authMiddleware: Middleware = {
  onRequest({ schemaPath, request }) {
    if (UNPROTECTED_ROUTES.some((path) => schemaPath.startsWith(path))) {
      return undefined; // Skip this middleware
    }
    request.headers.set("Authorization", `Bearer ${accessToken}`);
    return request;
  },
};
```

### Caching Middleware Pattern

```typescript
const cache = new Map<string, Response>();
const getCacheKey = (request: Request) => `${request.method}:${request.url}`;

const cacheMiddleware: Middleware = {
  onRequest({ request }) {
    const cached = cache.get(getCacheKey(request));
    if (cached) {
      return cached.clone(); // Return Response to short-circuit
    }
  },
  onResponse({ request, response }) {
    if (response.ok) {
      cache.set(getCacheKey(request), response.clone());
    }
  },
};
```

### Error-Throwing Middleware

By default, `openapi-fetch` does **not** throw on 4xx/5xx — it returns `{ error }`. To throw instead:

```typescript
const throwOnError: Middleware = {
  onResponse({ response }) {
    if (!response.ok) {
      throw new Error(`${response.url}: ${response.status} ${response.statusText}`);
    }
  },
};
```

### Body Statefulness

Request/Response bodies are stateful (can only be read once). When reading without modification, **clone first**:

```typescript
const loggingMiddleware: Middleware = {
  async onResponse({ response }) {
    const data = await response.clone().json(); // Clone before reading
    console.log(data);
    return undefined; // Don't modify
  },
};
```

## Query Serialization

### Object Configuration

```typescript
const client = createClient<paths>({
  baseUrl: "https://myapi.dev/v1/",
  querySerializer: {
    array: {
      style: "pipeDelimited", // "form" (default) | "spaceDelimited" | "pipeDelimited"
      explode: false,         // default: true
    },
    object: {
      style: "form",          // "form" | "deepObject" (default)
      explode: true,          // default: true
    },
    allowReserved: false,      // default: false
  },
});
```

### Custom Function

```typescript
const client = createClient<paths>({
  baseUrl: "https://myapi.dev/v1/",
  querySerializer(queryParams) {
    let s = "";
    for (const [k, v] of Object.entries(queryParams)) {
      if (v !== undefined) {
        s += `${s ? "&" : ""}${k}=${encodeURIComponent(String(v))}`;
      }
    }
    return s;
  },
});
```

### Per-Request Override

```typescript
const { data } = await client.GET("/search", {
  params: { query: { tags: ["a", "b", "c"] } },
  querySerializer: {
    array: { style: "pipeDelimited", explode: false },
  },
});
```

## Body Serialization

### FormData

```typescript
const { data } = await client.POST("/upload", {
  body: { file: myFile, name: "photo" },
  bodySerializer(body) {
    const fd = new FormData();
    for (const [name, value] of Object.entries(body)) {
      fd.append(name, value);
    }
    return fd;
  },
});
```

### URL-Encoded

Provide the content-type header and pass body as an object:

```typescript
const { data } = await client.POST("/form", {
  body: { username: "user", password: "pass" },
  headers: { "Content-Type": "application/x-www-form-urlencoded" },
});
```

## Path Serialization

Custom path parameter formatting:

```typescript
const client = createClient<paths>({
  baseUrl: "https://myapi.dev/v1/",
  pathSerializer(pathname, pathParams) {
    let result = pathname;
    for (const [key, value] of Object.entries(pathParams)) {
      result = result.replace(`{${key}}`, encodeURIComponent(String(value)));
    }
    return result;
  },
});
```

## Path-Based Client

Alternative API where methods are accessed via path:

```typescript
import { createPathBasedClient, wrapAsPathBasedClient } from "openapi-fetch";
import type { paths } from "./my-schema";

// Option 1: Create directly (no middleware support)
const client = createPathBasedClient<paths>({ baseUrl: "https://myapi.dev/v1/" });
const { data } = await client["/blogposts/{post_id}"].GET({
  params: { path: { post_id: "123" } },
});

// Option 2: Wrap existing client (preserves middleware)
const baseClient = createClient<paths>({ baseUrl: "https://myapi.dev/v1/" });
baseClient.use(authMiddleware);
const pathClient = wrapAsPathBasedClient(baseClient);
const { data } = await pathClient["/blogposts"].GET();
```

**Note:** `createPathBasedClient` does not support middleware. Use `wrapAsPathBasedClient` with an existing client if you need middleware.

## Testing

### Mocking Requests with vi.fn / jest.fn

Supply a mock fetch to the client:

```typescript
import createClient from "openapi-fetch";
import { expect, test, vi } from "vitest";
import type { paths } from "./my-schema";

test("my request", async () => {
  const mockFetch = vi.fn();
  const client = createClient<paths>({
    baseUrl: "https://my-site.com/api/v1/",
    fetch: mockFetch,
  });

  const reqBody = { name: "test" };
  await client.PUT("/tag", { body: reqBody });

  const req = mockFetch.mock.calls[0][0];
  expect(req.url).toBe("/tag");
  expect(await req.json()).toEqual(reqBody);
});
```

### Mocking Responses with MSW

Mock Service Worker is recommended for mocking API responses:

```typescript
import { http, HttpResponse } from "msw";
import { setupServer } from "msw/node";
import createClient from "openapi-fetch";
import { afterEach, beforeAll, afterAll, expect, test } from "vitest";
import type { paths } from "./my-schema";

const server = setupServer();

beforeAll(() => {
  server.listen({
    onUnhandledRequest: (request) => {
      throw new Error(`No request handler found for ${request.method} ${request.url}`);
    },
  });
});

afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test("my API call", async () => {
  const rawData = { test: { data: "foo" } };
  const BASE_URL = "https://my-site.com";

  server.use(
    http.get(`${BASE_URL}/api/v1/foo`, () =>
      HttpResponse.json(rawData, { status: 200 }),
    ),
  );

  const client = createClient<paths>({ baseUrl: BASE_URL });
  const { data, error } = await client.GET("/api/v1/foo");

  expect(data).toEqual(rawData);
  expect(error).toBeUndefined();
});
```

**Important:** `server.listen()` must be called before `createClient` is used so MSW can intercept requests.

## Exported Types

Key types available from `openapi-fetch`:

```typescript
import type {
  Client,
  ClientOptions,
  Middleware,
  FetchOptions,
  FetchResponse,
  MaybeOptionalInit,
  ParseAs,
  QuerySerializer,
  QuerySerializerOptions,
  BodySerializer,
  PathSerializer,
  HeadersOptions,
  MethodResponse,
  ClientPathsWithMethod,
  PathBasedClient,
} from "openapi-fetch";
```

### `MethodResponse` — Extract Response Data Type

```typescript
import type { MethodResponse } from "openapi-fetch";

// Extract the data type for a specific endpoint
type BlogPost = MethodResponse<typeof client, "get", "/blogposts/{post_id}">;
```

### `ClientPathsWithMethod` — Get All Paths for a Method

```typescript
import type { ClientPathsWithMethod } from "openapi-fetch";

type GettablePaths = ClientPathsWithMethod<typeof client, "get">;
```

### `MaybeOptionalInit` — Init Type for a Path/Method

Determines whether the init parameter is required or optional based on the schema:

```typescript
import type { MaybeOptionalInit } from "openapi-fetch";
import type { paths } from "./my-schema";

type BlogPostInit = MaybeOptionalInit<paths["/blogposts/{post_id}"], "get">;
```

## Exported Utilities

Helper functions for custom serialization:

```typescript
import {
  createQuerySerializer,
  defaultPathSerializer,
  defaultBodySerializer,
  createFinalURL,
  mergeHeaders,
  serializePrimitiveParam,
  serializeObjectParam,
  serializeArrayParam,
  removeTrailingSlash,
} from "openapi-fetch";
```

## System Requirements

- **Browsers:** Modern browsers with Fetch API support
- **Node.js:** >= 18.0.0
- **TypeScript:** >= 4.7 (5.0+ recommended)
