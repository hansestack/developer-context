# Hansestack Leak-Check API — AI Agent Context

Machine-readable integration spec. Follow it exactly; do not invent endpoints, fields, or response shapes not listed here.

## 1. System Prompt / Goal

You are building an HTTP client that checks whether a password appears in a known data-breach corpus via the Hansestack Leak-Check API, using the k-anonymity model so that neither the plaintext password nor the full password hash ever leaves the client. Implement precisely the algorithm in Section 3 and the fail-open behavior in Section 4. Do not add retries, caching, or extra request fields beyond what is specified.

## 2. API Endpoint & Auth

- Base URL (fixed, do not templatize or make configurable): `https://api.hansestack.de/leakcheck`
- Method: `GET`
- Path: `/v1/prefixes/{prefix}`
- Full request: `GET https://api.hansestack.de/leakcheck/v1/prefixes/{PREFIX}`
- Auth header (required on every request): `X-API-Key: <API_KEY>`
- Optional header: `Accept: application/json`
- `{prefix}` is a URL path segment, not a query parameter: 5 uppercase hexadecimal characters (e.g. `5B4E3`).
- Rate-limit headers, returned on every response (200 and 429): `X-RateLimit-Limit` (burst quota for your tier), `X-RateLimit-Remaining` (tokens left right now), `X-RateLimit-Reset` (Unix epoch seconds at which the token bucket refills). This is a token-bucket, not a fixed window. Reading these is optional and must never block the request path — see Section 4 for handling `429`.

## 3. The k-Anonymity Algorithm (Core Logic)

Exact sequence — do not deviate:

1. Compute `hash = SHA1(plaintext_password)` → 40-character hex digest.
2. Uppercase the digest: `hash = hash.toUpperCase()`.
3. Split the digest:
   - `prefix = hash[0:5]` (first 5 characters)
   - `suffix = hash[5:40]` (remaining 35 characters)
4. Send `GET <BASE_URL>/v1/prefixes/{prefix}` with **only** the prefix. Never transmit the password, the full hash, or the suffix over the network.
5. On HTTP 200, parse the response body as a **flat JSON object** mapping suffix → breach count: `{"<suffix>": <count>, ...}`. There is no wrapper key (no `.suffixes` field, no envelope) — the response body itself is the map.
6. Look up the locally computed `suffix` as a key in that map (case-sensitive, uppercase hex):
   - Key present → password is compromised; leak count = the value.
   - Key absent → password not found in this dataset.

Reference pseudocode:

```
function checkPasswordLeak(password: string) -> { leaked: bool, count: int }:
    hash   = SHA1(password).toUpperCase()
    prefix = hash[0:5]
    suffix = hash[5:40]

    response = HTTP GET "https://api.hansestack.de/leakcheck/v1/prefixes/{prefix}"
        headers: { "X-API-Key": API_KEY, "Accept": "application/json" }
        timeout: 500ms

    if response.status != 200:
        return { leaked: false, count: 0 }   # fail-open, see Section 4

    suffixMap = parseJSON(response.body)      # flat { SUFFIX: count }
    if suffix in suffixMap:
        return { leaked: true, count: suffixMap[suffix] }
    return { leaked: false, count: 0 }
```

## 4. Mandatory Fail-Open Implementation (Critical)

Hansestack is an additional security layer, never a single point of failure. This is non-negotiable — implement in every generated client:

- Set an explicit HTTP client timeout of **500ms** on every request to this API. Never rely on a language/library default timeout.
- Treat all of the following as **non-fatal** and catch them explicitly:
  - Network errors (DNS failure, connection refused, TLS errors)
  - Timeouts
  - Any 5xx response status
  - HTTP 429 (rate limit exceeded)
- On any of the above: do not throw/propagate an exception that halts the caller's flow. Return a "not leaked" / neutral result so sign-up, login, and password-change flows continue exactly as if the check had not been performed. Log the failure internally (non-blocking) if a logger is available.
- Do not add blocking synchronous retries in the request path — a single attempt bounded by the 500ms timeout is sufficient for fail-open behavior.
- On `429`, read `X-RateLimit-Reset` for logging/telemetry only (e.g. to back off future background jobs) — never delay or retry the current request because of it; fail-open immediately like any other non-200 response.
- HTTP 4xx other than 429 (e.g. 401 invalid API key, 400 malformed prefix) indicates a client misconfiguration, not a leak. Still fail-open at runtime (never block the user), but log/surface these distinctly since they signal a broken integration rather than a rate limit or outage.
