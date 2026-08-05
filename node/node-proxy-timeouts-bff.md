# Node fetch timeouts + why a thin BFF replaces nginx here
<!-- verified: 2026-08 -->

## Set your own timeout below the platform's
Node's `fetch` (undici) defaults to a 300s `headersTimeout`/`bodyTimeout` and throws `UND_ERR_HEADERS_TIMEOUT` — which reaches the browser as an opaque 502 with no useful message.
```js
const res = await fetch(url, { signal: AbortSignal.timeout(<MS_BELOW_PLATFORM_TIMEOUT>) });
```
Example (platform timeout is 60s, so time out client-side at 55s):
```js
const res = await fetch(url, { signal: AbortSignal.timeout(55000) });
```
**Why it bites:** without this, whichever platform timeout fires first (Cloud Run's, a load balancer's) wins and reports its own generic error — there's no way to distinguish "upstream timed out" from "upstream errored" from the logs. Setting a timeout below the platform's means your own code produces the error, so it can name itself: return `504` for your own timeout, `502` for a genuine upstream failure.

## Handle the abort landing mid-response-body
If the abort fires after headers are already sent to the client, a new status code can't be sent — destroy the socket instead of trying to write an error response:
```js
req.on('aborted', () => res.socket?.destroy());
```

## Why not just use nginx here
nginx can't serve an authenticated SPA against a token-gated backend — it has no way to mint an identity token per request. A small Node BFF (~140 lines) that fetches the ID token via ADC and proxies through does the job nginx can't.
