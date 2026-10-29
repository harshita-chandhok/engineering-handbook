# 01 · HTTP & API Fundamentals

How a request travels, how HTTP behaves under failure, how to design a stable API, and how to measure it.

## 1. Life of a request
URL parse → DNS → TCP → TLS → HTTP request → load balancer → app server (middleware → handler) → database → response.

| Stage | Why it exists | Cost / trade-off |
|---|---|---|
| DNS | Routers move data by numbers; names are for humans | Cached via TTL → answers can be stale |
| TCP (3-way handshake) | Both sides prove they can hear *and* speak; agree packet numbers | 1 round trip (RTT) |
| TLS | Stops eavesdropping (encryption), tampering (integrity), impersonation (certificate) | ~1 RTT (TLS 1.3), 2 RTT (TLS 1.2) |
| Load balancer | Capacity and survival: many identical servers, health checks | Forces stateless servers |

- Fresh HTTPS request ≈ 3 RTT + processing; a request on a reused connection ≈ 1 RTT + processing.
- Physics sets a floor on latency, so reuse connections, make fewer round trips, and move servers closer (CDN / regions).
- Run independent calls in parallel (total ≈ slowest call, not the sum).

## 2. HTTP semantics
- **Stateless:** every request carries what the server needs, so any server can answer any request. State lives in a shared store (DB / Redis) or in a client-held token (cookie / JWT).
- **Safe** = changes nothing. **Idempotent** = repeating leaves the same final state as doing it once.

| Method | Safe | Idempotent |
|---|---|---|
| GET | yes | yes |
| POST | no | no |
| PUT | no | yes |
| PATCH | no | depends (`set email` yes, `increment` no) |
| DELETE | no | yes |

- **Why idempotency matters:** a lost response looks identical to a lost request, so the client can't tell whether to retry. Retrying a non-idempotent request risks doing it twice. Fix for POST: **idempotency key** + server-side dedupe (unique constraint, store the result).
- **Retry rule:** retry only idempotent requests, only on temporary failures (timeout, 502/503/504, 429 after `Retry-After`), with backoff and jitter. Never retry a 4xx unchanged.

## 3. Status codes
200 OK · 201 Created (+ `Location`) · 204 No Content · 400 malformed · 401 not authenticated · 403 not allowed · 404 not found · 409 conflict · 422 invalid · 429 rate limited · 500 server bug · 502/503/504 gateway/overload/timeout.
Never return 200 with an error body: monitoring, caches and retry logic all treat it as success.

## 4. API design
- Resources are nouns, methods are verbs (`DELETE /orders/5`, not `GET /deleteOrder?id=5`).
- One error shape: `{ "error": { "code", "message", "details", "request_id" } }`. Clients switch on the stable `code`, never the message.
- Add `X-Request-Id` to every response and log line.
- Version (`/v1`). Non-breaking: new optional field/endpoint. Breaking: remove/rename a field, change a type, new required field → needs `/v2`.
- Validate all input at the edge. Take the user identity from the auth token, not the URL.

## 5. Pagination
- **Offset:** `LIMIT n OFFSET k`. Walks and discards k rows (slow on deep pages) and drifts when rows are inserted or deleted (duplicates / skips).
- **Cursor (keyset):** "after the last item I saw."
```sql
SELECT * FROM items
WHERE (sort_col, id) < (:last_sort, :last_id)   -- tuple order = ORDER BY order
ORDER BY sort_col DESC, id DESC
LIMIT 21;                                       -- page size + 1 → know if a next page exists
-- index: (sort_col DESC, id DESC)
```
- Sort key must be unique → add `id` as a tie-breaker, or rows silently vanish.
- Cursor is **opaque** (encoded, ideally signed): clients only copy it, so its contents can change without breaking them. It never grants permission; auth is checked on every request.
- Cursor limits: no jumping to "page 57" and no cheap exact totals. Use offset only for small admin tables.

## 6. Latency, throughput, capacity
- **Little's Law:** in flight (demand) = arrival rate × time each request holds the resource.
- **Capacity** = pool size ÷ hold time. If arrival rate > capacity, the backlog grows by the difference *every second*.
- **Pool size** = demand ÷ target utilization. Never hold a DB connection across a slow external call (e.g. an LLM).
- **Utilization cliff:** response time ≈ service time ÷ (1 − utilization), so 50% → 2×, 90% → 10×, 95% → 20×. Keep servers around 50-70% busy.

## 7. Percentiles
- The mean can describe no real request. Track p50 / p95 / p99.
- Tail amplification: P(at least one slow in n) = 1 − (1 − p)ⁿ. Example: 50 calls at 1% slow → ~40%.
- Chained dependencies multiply: five 99.9% services give ~99.5% success overall.
- Percentiles can't be averaged across servers; compute them from histograms.

## 8. LLM calls are just HTTP
2-30 s POSTs → timeouts, retry budget, 429 + `Retry-After`, streaming (SSE) and time-to-first-token, and proxy idle timeouts (504) apply.

## Interview one-liners
- "Retries are safe only for idempotent operations; for POST I'd add an idempotency key."
- "Offset pagination drifts and gets slower with depth; I'd use a keyset cursor with a unique tie-breaker."
- "I'd watch p99, because tails multiply across fan-out."
- "I'd keep utilization well below 100% because latency explodes near saturation."
