# Caching guide for backend APIs

A shared guide for any project where an app (mobile or web) calls an API and the backend runs on the edge (Cloudflare Workers or similar). The code examples are in Swift and Cloudflare Workers, but the principles apply to other platforms.

What caching is for:

- **Faster for the user**, especially the first time and on a weak network.
- **Less load and lower cost** for the most expensive part: the database, third-party APIs, AI models, TTS services…
- **Less risk of being blocked** when you depend on an external service that is rate-limited or unofficial.
- **Works offline** (client-side cache).

---

## 1. Cache layers, from nearest the user to farthest

| Layer | Where | Shared between users? | How long it lives | Good for |
|---|---|---|---|---|
| App RAM | Memory, per screen or per session | No | Until the screen or app closes | Data just used; avoid refetching when returning to a screen |
| App disk | `Caches/` (iOS), IndexedDB (web) | No | Until it's purged | Data used again and again, media files (audio, images, SVG), offline use |
| Client HTTP cache | `URLCache`, the browser cache | No | Per `Cache-Control` | GET responses with the right cache headers |
| CDN / edge cache | The data center near the user | Yes, **per data center** | Per TTL; may be evicted early | Public GET responses, or POSTs turned into a GET key (section 4) |
| Persistent store | KV, R2/S3, Redis | Yes, global or one region | Until you delete it | Expensive results that must be kept long and shared globally |
| Origin | Database, external API | — | — | — |

How to choose:

- **Start with the client** (RAM + disk). It costs nothing on the server and helps returning users right away.
- **Add an edge cache** when many users ask for the same thing (dictionary content, articles, audio for sample sentences). This layer is usually free.
- **Add a persistent store only** when measurements show the edge cache misses a lot and the origin is expensive or risky. It's the layer with the most operational work (storage, cleanup, abuse).
- Always **measure before and after** (section 8). Don't add a cache layer because it "sounds good".

---

## 2. Designing the cache key

A wrong key is the most common source of cache bugs: returning data for a different request, or never hitting the cache.

- **Put every parameter that changes the result into the key**: language, voice, speed, output format, data version… Missing one returns the wrong data.
- **Normalize before building the key**: trim whitespace, collapse repeated whitespace, sort query parameters by name, normalize Unicode if needed. Normalize *exactly the way the backend processes the input*, so two requests that give the same result get the same key.
- **Hash long content** (SHA-256, then hex). The key stays short, doesn't expose the content, and doesn't hit URL length limits.
- **Add a version prefix** (`v1/…`). When the output format changes or a data bug is fixed, bump the version to drop the whole old cache without deleting entries one by one.
- **On the client, key by content, not by screen.** A key like `voice|text` lets the same sentence on two screens share one file. A key like `screenId|itemIndex` doesn't.

```js
const params = { text: normalize(text), voice, rate, pitch, format };
const hash = await sha256Hex(JSON.stringify(params));   // fixed key order
const key = `v1/${hash}`;
```

---

## 3. What must never be cached

- **Errors**: 4xx, 5xx, timeouts, empty or invalid responses. Caching an error means the user gets the error for the whole TTL. Only write to the cache after checking the result is valid.
- **Private user data** (needs a login, has `Authorization` or a cookie) in a shared cache. If it really must be cached, put the user id in the key and use a private cache only.
- **Sensitive content** the user entered (personal documents, OCR'd photos…) in a shared persistent store. Caching on the user's own device is fine. If needed, let the client send a flag like `cache: false`.
- **Responses with `Set-Cookie`.** Most CDNs won't cache them or, worse, cache one person's cookie and serve it to someone else.

---

## 4. Edge cache on Cloudflare Workers (Cache API)

`caches.default` is free, needs no binding, and is very fast (a few ms on a hit). What to know:

- **It's per data center.** Users in two different countries each pay for their own first request. Entries may be evicted before the TTL if they're rarely used.
- **Keys must be GET requests.** For a POST API, build a synthetic URL from a hash of the body and use it as the key.
- **It doesn't work on `*.workers.dev` domains.** Attach a custom domain, otherwise `cache.put` does nothing.
- Responses put in the cache need `Cache-Control: public, max-age=…`.
- **Write to the cache with `ctx.waitUntil`**, so the user doesn't wait for the write. Remember to pass `ctx` to the handler.
- Add an **`X-Cache: HIT/MISS`** header to responses to verify and measure the hit rate.

```js
const CACHE_VERSION = "v1";
const CACHE_TTL = 30 * 24 * 60 * 60;

export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    const body = await request.json();
    const params = normalizeParams(body);

    const cache = caches.default;
    const key = new Request(`${url.origin}/__cache/${CACHE_VERSION}/${await sha256Hex(JSON.stringify(params))}`);

    const hit = await cache.match(key);
    if (hit) {
      return new Response(hit.body, { headers: { "Content-Type": "application/json", "X-Cache": "HIT" } });
    }

    const json = JSON.stringify(await expensiveWork(params));   // throw on failure → never cached

    ctx.waitUntil(cache.put(key, new Response(json, {
      headers: { "Content-Type": "application/json", "Cache-Control": `public, max-age=${CACHE_TTL}` },
    })));
    return new Response(json, { headers: { "Content-Type": "application/json", "X-Cache": "MISS" } });
  },
};

async function sha256Hex(str) {
  const digest = await crypto.subtle.digest("SHA-256", new TextEncoder().encode(str));
  return [...new Uint8Array(digest)].map((b) => b.toString(16).padStart(2, "0")).join("");
}
```

**Keep the response format unchanged** when adding a cache. That way app versions already in users' hands benefit immediately, without an update.

For **public GET** APIs, you can let the CDN cache on its own with `Cache-Control` headers (plus Cache Rules in the dashboard if needed), and **sort query parameters in a fixed order** on the client so identical requests have identical URLs.

---

## 5. Persistent stores: when you need one and which to pick

Only when the result is **expensive** (AI, TTS, paid APIs), **reused by many users**, and the edge cache **misses a lot** because users are spread across regions or the data is rarely requested.

| Option | Good for | Notes |
|---|---|---|
| Object storage (R2, S3) | Binary files (audio, images), in large numbers | Cheap; R2 has no egress fees. Lifecycle rules delete by *file age*, not by last access |
| KV (Workers KV) | Very many reads, very few writes | The free tier allows very few writes per day, so it's **a poor fit** for a cache that writes constantly. A write takes tens of seconds to be visible everywhere |
| Redis with a REST API (e.g. Upstash) | Small key–values, per-key TTL, counters, rate limits | Callable from a Worker over HTTP. The free tier is limited by storage and command count |
| TCP-only Redis (e.g. Redis Cloud's free tier) | Backends running on traditional servers | The free tier is tiny (tens of MB). Hard and slow to reach from edge functions, because the database lives in one region |

Free-tier limits and prices change often. **Check the pricing page again** before deciding.

When you use a persistent store:

- Put the **Cache API in front of** the store: Cache API → store → origin.
- Estimate the storage: average entry size × expected number of entries. For example, 48 kbps MP3 audio takes about 6 KB per second.
- Have a plan for **cleanup** (TTL, lifecycle) and **abuse prevention** (section 7).

---

## 6. Client-side cache

### RAM

- A dictionary keyed by the cache key, living per screen or per session. Very fast, but gone when the screen closes.
- **In-flight dedupe:** store the running `Task` by key. A second request for the same key `await`s that same `Task`.

```swift
private var inFlight: [String: Task<Item, Error>] = [:]

func item(for key: String) async throws -> Item {
    if let cached = cache[key] { return cached }
    if let running = inFlight[key] { return try await running.value }
    // Unstructured: if the caller is cancelled (e.g. a debounced prefetch), the result still lands in the cache.
    let task = Task { try await fetch(key) }
    inFlight[key] = task
    defer { inFlight[key] = nil }
    let value = try await task.value
    cache[key] = value
    return value
}
```

### Disk

- iOS: use the `Caches/` directory. The system may purge it when the device runs low on space, so **always treat data there as something that can disappear**.
- The file name is a hash of the key (with the version prefix), written with `.atomic`.
- **Cap the size** and delete the least recently used files (LRU): update the file's modification date on every read; when over the cap, delete by oldest date down to about 75% of the cap so cleanup doesn't run constantly. Run cleanup at most once per session.
- **Read and write files off the main thread.** In projects whose default isolation is `MainActor`, mark the type `nonisolated` and the functions `@concurrent`.
- Writing the cache must not delay what's shown: hand the data to the UI first, write the file afterwards (`Task { await write(...) }`).

### HTTP cache (`URLCache`)

- Respect `Cache-Control` from the server. It's the cheapest option when the backend sets the headers right.
- The default capacity is small. If you rely on it, configure a larger `URLCache` for that session.
- Responses that are large relative to the cache capacity may not be stored. For large files, cache on disk yourself.
- When measuring performance, remember that the previous run's `URLCache` makes the numbers look falsely good. Uninstall the app before measuring.

### Prefetching

- Prefetch what the user will most likely tap next (e.g. the top 3 search results), **with a debounce** (300–500 ms) because the list changes with every keystroke.
- Share the in-flight dedupe above. Otherwise a detail screen that opens while the prefetch is still running sends a second request.
- Skip it when the data is already on the device (offline pack, cache).

---

## 7. Safety and abuse prevention

A shared cache turns a public endpoint into something that can be filled up or freeloaded on.

- **Limit input size in the backend.** Don't rely on the client having trimmed it, because anyone can call the endpoint directly.
- **Rate-limit** by IP or by user (a CDN rule or in code).
- For persistent stores: cap the storage, set a lifecycle, watch the growth rate.
- Validate parameters against an allowlist (languages, voices, formats) so an attacker can't generate endless distinct keys.

---

## 8. Measuring

Measure **before** adding the cache and **after**, with the same scenario.

```sh
# Separate connection, processing and cache costs
curl -s -o /dev/null -D - \
  -w "dns=%{time_namelookup} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total}\n" \
  "https://api.example.com/lookup?term=..." | grep -iE "x-cache|cf-cache-status|age|dns="
```

- Call twice in a row with the same input: the first must be `MISS`, the second `HIT`, and **the bodies must be identical** (compare with `cmp`).
- Run it locally (`wrangler dev`) before deploying. If a commit deploys automatically, this is the last check.
- Track the hit rate through logs or headers. A low hit rate usually means the key isn't normalized, or the data simply doesn't repeat much (in which case the cache isn't worth it).
- Tell the costs apart: **connection** (DNS + TLS; fix by warming or keeping connections), **query** (fix with a cache) and **rendering** (fix on the client; see "First-use latency" in `silent-bugs-checklist.md`).

---

## 9. Checklist when adding a cache

- [ ] The key has every parameter that affects the result, is normalized, and has a version prefix.
- [ ] Only successful, valid results are cached.
- [ ] No private user data in shared layers.
- [ ] Writing the cache doesn't block the response (`waitUntil` on the server, background write on the client).
- [ ] There's a size cap and a cleanup method.
- [ ] The backend limits input size and has rate limiting.
- [ ] The response format is unchanged (old clients still work).
- [ ] There's a HIT/MISS header or a log to measure with.
- [ ] MISS → HIT with identical bodies was tested before deploying.
- [ ] The README records the TTL, how to clear the cache (bump the version), and what is deliberately not cached.
