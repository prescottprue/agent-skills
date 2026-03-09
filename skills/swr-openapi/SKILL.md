---
name: swr-openapi
description: React data fetching with swr-openapi - type-safe SWR hooks generated from OpenAPI schemas. Use when writing, reviewing, or refactoring React components that fetch data using swr-openapi, openapi-fetch, or openapi-typescript. Triggers on tasks involving React data fetching with OpenAPI clients.
license: MIT
metadata:
  author: prescottprue
  version: "1.0.0"
---

# SWR OpenAPI

Comprehensive guide for using `swr-openapi` — type-safe SWR bindings for OpenAPI schemas via `openapi-fetch` and `openapi-typescript`.

## When to Apply

Reference these guidelines when:
- Writing React components that fetch data from an OpenAPI-defined API
- Setting up type-safe data fetching with SWR and OpenAPI schemas
- Implementing pagination with `useInfinite`
- Managing cache revalidation and mutation with `useMutate`
- Refactoring existing SWR or fetch code to use `swr-openapi`

## Installation

```bash
npm i swr-openapi openapi-fetch
npm i -D openapi-typescript typescript
```

Enable `noUncheckedIndexedAccess` in `tsconfig.json` for enhanced type safety.

## Schema Generation

Generate TypeScript types from your OpenAPI schema:

```bash
npx openapi-typescript ./path/to/api/v1.yaml -o ./src/lib/api/v1.d.ts
```

This produces a `paths` type export used to type the client and all hooks.

## Setup — Creating Hooks

Create a client with `openapi-fetch`, then use hook builders to export typed hooks. Each builder takes a `client` and a `prefix` string. The prefix ensures SWR avoids cache collisions between different APIs.

```typescript
import createClient from "openapi-fetch";
import {
  createQueryHook,
  createImmutableHook,
  createInfiniteHook,
  createMutateHook,
} from "swr-openapi";
import { isMatch } from "lodash-es"; // recommended compare function
import type { paths } from "./my-openapi-3-schema"; // generated types

const client = createClient<paths>({
  baseUrl: "https://myapi.dev/v1/",
});

const prefix = "my-api";

export const useQuery = createQueryHook(client, prefix);
export const useImmutable = createImmutableHook(client, prefix);
export const useInfinite = createInfiniteHook(client, prefix);
export const useMutate = createMutateHook(client, prefix, isMatch);
```

### Hook Builder Signatures

```typescript
createQueryHook(client, prefix)       // → useQuery
createImmutableHook(client, prefix)   // → useImmutable
createInfiniteHook(client, prefix)    // → useInfinite
createMutateHook(client, prefix, compare) // → useMutate
```

- `client` — An `openapi-fetch` client instance.
- `prefix` — A unique string per client. Used only for SWR cache key uniqueness, not included in actual requests.
- `compare` — (`createMutateHook` only) A `CompareFn` `(init: any, partialInit: any) => boolean` for matching cache keys during mutation. Lodash `isMatch` is recommended.

## Hook API Reference

### `useQuery`

Type-safe wrapper over `useSWR`. Fetches data via `client.GET()`.

```typescript
const { data, error, isLoading, isValidating, mutate } = useQuery(
  path,
  init,
  config,
);
```

**Parameters:**
- `path` — Any endpoint that supports `GET` requests (type-checked against schema).
- `init` — Fetch options for the endpoint (params, headers, etc.), or `null` to skip the request (conditional fetching).
- `config` — *(optional)* SWR configuration options.

**Returns:** An SWR response (`data`, `error`, `isLoading`, `isValidating`, `mutate`).

**How it works internally:**

```typescript
function useQuery(path, ...[init, config]) {
  return useSWR(
    init !== null ? [prefix, path, init] : null,
    async ([_prefix, path, init]) => {
      const res = await client.GET(path, init);
      if (res.error) {
        throw res.error;
      }
      return res.data;
    },
    config,
  );
}
```

**Example:**

```typescript
import { useQuery } from "./my-api";

function BlogPost({ postId }: { postId: string }) {
  const { data, error, isLoading } = useQuery("/blogposts/{post_id}", {
    params: {
      path: { post_id: postId },
    },
  });

  if (isLoading || !data) return "Loading...";
  if (error) return `An error occurred: ${error.message}`;
  return <div>{data.title}</div>;
}
```

**Conditional fetching (pass `null` as init to skip):**

```typescript
const { data } = useQuery(
  "/blogposts/{post_id}",
  postId ? { params: { path: { post_id: postId } } } : null,
);
```

### `useImmutable`

Identical API to `useQuery`, but wraps `useSWRImmutable` — disables automatic revalidations. Use for data that doesn't change.

```typescript
const { data, error, isLoading, isValidating, mutate } = useImmutable(
  path,
  init,
  config,
);
```

Parameters and return values are the same as `useQuery`.

### `useInfinite`

Type-safe wrapper over `useSWRInfinite` for paginated data.

```typescript
const { data, error, isLoading, isValidating, mutate, size, setSize } =
  useInfinite(path, getInit, config);
```

**Parameters:**
- `path` — Any endpoint supporting `GET` requests.
- `getInit` — `(pageIndex: number, previousPageData?: ResponseData) => FetchOptions | null` — Returns fetch options for each page, or `null` to stop loading.
- `config` — *(optional)* SWR infinite options.

**Returns:** An SWR infinite response (`data`, `error`, `isLoading`, `isValidating`, `mutate`, `size`, `setSize`).

**Limit/Offset pagination:**

```typescript
useInfinite("/something", (pageIndex, previousPageData) => {
  // Stop if no more data
  if (previousPageData && !previousPageData.hasMore) return null;
  // First page
  if (!previousPageData) return { params: { query: { limit: 10 } } };
  // Subsequent pages
  return { params: { query: { limit: 10, offset: 10 * pageIndex } } };
});
```

**Cursor-based pagination:**

```typescript
useInfinite("/something", (pageIndex, previousPageData) => {
  if (previousPageData && !previousPageData.nextCursor) return null;
  if (!previousPageData) return { params: { query: { limit: 10 } } };
  return {
    params: { query: { limit: 10, cursor: previousPageData.nextCursor } },
  };
});
```

### `useMutate`

Type-safe wrapper around SWR's global `mutate`. Updates and revalidates the client-side cache for specific endpoints.

```typescript
const mutate = useMutate();

await mutate([path, init], data, options);
```

**Parameters:**
- `path` — Any endpoint supporting `GET` requests.
- `init` — *(optional)* Partial fetch options to narrow which cache entries to match.
- `data` — *(optional)* Data to update the cache, or an async function for remote mutation.
- `options` — *(optional)* SWR mutate options.

**Returns:** A promise containing an array where each item is updated data for a matched key or `undefined`.

**Example — revalidate all entries for a path:**

```typescript
const mutate = useMutate();

// Revalidate all /blogposts cache entries
await mutate(["/blogposts"]);
```

**Example — revalidate a specific entry:**

```typescript
// Revalidate a specific blog post
await mutate(["/blogposts/{post_id}", { params: { path: { post_id: "123" } } }]);
```

**Example — optimistic update:**

```typescript
await mutate(
  ["/blogposts/{post_id}", { params: { path: { post_id: "123" } } }],
  { ...currentData, title: "Updated Title" },
  { revalidate: false },
);
```

## Key Patterns

### File Organization

```
src/
  lib/
    api/
      v1.d.ts          # Generated types (do not edit)
      client.ts         # Client + hook exports
  components/
    MyComponent.tsx     # Uses hooks from client.ts
```

### Multiple API Clients

Use distinct prefixes for each API to avoid cache collisions:

```typescript
const clientA = createClient<pathsA>({ baseUrl: "https://api-a.dev/v1/" });
const clientB = createClient<pathsB>({ baseUrl: "https://api-b.dev/v1/" });

export const useQueryA = createQueryHook(clientA, "api-a");
export const useQueryB = createQueryHook(clientB, "api-b");
```

### Error Handling

Errors thrown by `swr-openapi` are the `error` property from the `openapi-fetch` response. The shape matches the error schemas defined in your OpenAPI spec:

```typescript
const { data, error } = useQuery("/endpoint", { /* ... */ });

if (error) {
  // error is typed according to the endpoint's error response schema
  console.error(error);
}
```

### Conditional Fetching

Pass `null` as `init` to prevent the request from firing:

```typescript
const { data } = useQuery(
  "/users/{id}",
  userId ? { params: { path: { id: userId } } } : null,
);
```

### SWR Cache Key Structure

`swr-openapi` uses a tuple `[prefix, path, init]` as the SWR cache key. Understanding this is important for debugging and for `useMutate`:
- `prefix` — The unique string passed to the hook builder
- `path` — The OpenAPI path string (e.g., `"/blogposts/{post_id}"`)
- `init` — The fetch options object (params, headers, etc.)

When `init` is `null`, the key becomes `null` and SWR skips the request.

### Init Parameter Structure

The `init` object mirrors `openapi-fetch`'s request options:

```typescript
{
  params: {
    path: { post_id: "123" },      // URL path parameters
    query: { limit: 10, offset: 0 }, // Query string parameters
    header: { "X-Custom": "value" }, // Request headers
    cookie: { session: "abc" },      // Cookies
  },
  body: { title: "New Post" },       // Request body (for non-GET)
}
```

For `GET` endpoints, only `params` is typically used since GET requests don't have a body.

### Optional Init

If an endpoint has no required parameters, `init` can be omitted entirely:

```typescript
// Endpoint with no required params — init is optional
const { data } = useQuery("/blogposts");

// Equivalent to:
const { data } = useQuery("/blogposts", {});

// With SWR config but no params:
const { data } = useQuery("/blogposts", {}, { refreshInterval: 5000 });
```

## Exported Types

The library exports utility types for extracting request/response types from your schema:

```typescript
import type { TypesForGetRequest, TypesForRequest, CompareFn } from "swr-openapi";
import type { paths } from "./my-schema";

// Extract types for a specific GET endpoint
type BlogPostTypes = TypesForGetRequest<paths, "/blogposts/{post_id}">;

type Data = BlogPostTypes["Data"];       // Response data type
type Error = BlogPostTypes["Error"];     // Error response type
type Init = BlogPostTypes["Init"];       // Full init/options type
type Path = BlogPostTypes["Path"];       // Path parameters type
type Query = BlogPostTypes["Query"];     // Query parameters type
type Headers = BlogPostTypes["Headers"]; // Header parameters type
```

`TypesForRequest` is the generic version that accepts any HTTP method:

```typescript
import type { TypesForRequest } from "swr-openapi";

type PostTypes = TypesForRequest<paths, "post", "/blogposts">;
```
