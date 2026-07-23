# x402 V1 → V2 Migration Guide

Field notes from migrating a **live, payment-gated API** from x402 V1 to V2 on Base mainnet — the undocumented gotchas, working code from a service in production, and a checklist at the bottom.

Written from [x402ai](https://github.com/ukenal/x402ai), which settles real USDC on Base via the Coinbase CDP facilitator. Every code sample here is running in production, not reconstructed from the spec.

**Last verified:** July 2026 · `@x402/*` 2.19.0 · `@coinbase/x402` 2.1.0

**In a hurry?** → [Migration checklist](#migration-checklist)

---

## Why you're probably being forced into this

If you built an x402 service before mid-2026, you're on V1. You may not have noticed, because V1 services keep serving 402s and keep settling payments. Nothing crashes. The migration pressure comes from two directions instead:

**Discovery rejects you.** Submitting a V1 origin to x402scan returns, in effect, *"x402 v1 response detected — migrate to v2 spec"*, plus a second complaint about the absence of a discovery document. Since x402scan is where autonomous agents actually find services, a V1 service is invisible to the thing that generates traffic.

**Agents are already failing against you silently.** Check your logs for `invalid_payload` errors you never explained. Those are V2-capable clients attempting payment against V1 middleware and bouncing. You've been losing real requests, and the error message doesn't say why.

That second one is the part I'd flag hardest. I had those errors in my logs for weeks and filed them mentally as noise.

---

## What actually changed

Five things, in rough order of how much time they'll cost you:

1. **The package line moved.** `x402-hono` is frozen at 1.2.0. The live line is the scoped `@x402/*` namespace.
2. **The challenge moved from the body to a header.** A V2 402 has an *empty* JSON body and carries the challenge base64-encoded in a `payment-required` header.
3. **Network identifiers are CAIP-2 now.** `eip155:8453` rather than the old string form.
4. **CDP facilitator auth is supported natively.** If you were reaching into SDK internals to mint JWTs, you can delete that.
5. **`X-Forwarded-Proto` is ignored.** If you terminate TLS at a tunnel or proxy, your resource URLs will advertise as `http://` unless you set them explicitly.

---

## The package landscape

This tripped me up before I wrote a line of code, because searching for x402 packages surfaces the frozen ones first.

**Frozen (V1):** `x402-hono` @ 1.2.0 — final release, do not build on it.

**Live (V2):** the scoped namespace —
- `@x402/hono` (or your framework's equivalent)
- `@x402/core`
- `@x402/evm`
- `@x402/extensions`
- `@coinbase/x402` — for CDP facilitator config

At time of writing the `@x402/*` packages were at 2.19.0 and `@coinbase/x402` at 2.1.0. Check current versions; this line moves.

### Install gotcha

```bash
npm install @x402/hono@2 @x402/core@2 @x402/evm@2 @coinbase/x402@2 --legacy-peer-deps
```

You will likely need `--legacy-peer-deps`. The optional paywall peer dependency wants React 19; if your project is on React 18 (or has any other React in the tree), npm refuses the install outright. The paywall is optional and unrelated to server-side payment gating, so the flag is safe here — but know *why* you're using it rather than reflexively adding it.

---

## The server rewrite

**Before (V1):** middleware took a routes object and a facilitator, with auth headers you assembled yourself.

**After (V2):** you build a resource server, register a scheme against a network, then hand that to the middleware.

### Watch the import subpaths

This cost me time before a single line of logic ran. The V2 packages use subpath exports, and the obvious top-level import is not the one you want:

```js
import { paymentMiddleware } from '@x402/hono'
import { x402ResourceServer, HTTPFacilitatorClient } from '@x402/core/server'
import { createFacilitatorConfig } from '@coinbase/x402'
import { ExactEvmScheme } from '@x402/evm/exact/server'
```

`@x402/core/server`, not `@x402/core`. `@x402/evm/exact/server`, not `@x402/evm`.

### Building the handler

```js
const facilitatorClient = new HTTPFacilitatorClient(
  createFacilitatorConfig(
    process.env.CDP_API_KEY_NAME,
    process.env.CDP_API_KEY_PRIVATE_KEY
  )
)

const resourceServer = new x402ResourceServer(facilitatorClient)
  .register('eip155:8453', new ExactEvmScheme())

const routes = {
  'POST /api/embed': {
    accepts: {
      scheme: 'exact',
      price: prices['/api/embed'],
      network: 'eip155:8453',
      payTo: process.env.WALLET_ADDRESS,
    },
    resource: new URL('/api/embed', baseUrl).toString(),
    description: 'Text embeddings (768-dim) via nomic-embed-text. POST {input: string}.',
    mimeType: 'application/json',
  },
  // ...one entry per paid route
}

return paymentMiddleware(routes, resourceServer)
```

`register()` is fluent — it returns the resource server, so it chains off the constructor.

### Route config shape changed

V1 nested `description` under a `config` key — and silently ignored a top-level `description`, which is a fun thing to discover after wondering why your route descriptions never showed up anywhere.

V2 keys routes by `METHOD /path`, takes `accepts` as a single object rather than an array, and puts `description` and `mimeType` at the top level as siblings of `accepts`.

---

## Deleting the JWT hack

If you were on CDP's facilitator under V1, there's a decent chance you did what I did: imported `generateJwt` through `@coinbase/cdp-sdk`'s internal `_cjs` path via `createRequire`, then hand-built auth headers on every request.

That was never supported. It worked, but it depended on the internal module layout of someone else's package.

`createFacilitatorConfig(apiKeyId, apiKeySecret)` from `@coinbase/x402` is the supported replacement. The entire hack — the `createRequire` shim, the internal import, the `createAuthHeaders` function — deletes cleanly. This was the single most satisfying part of the migration and the strongest argument for doing it even if discovery weren't forcing your hand.

---

## The `https` gotcha — and the workaround you can now delete

Under V1, if you terminated TLS upstream (Cloudflare tunnel, nginx, anything), the middleware saw the internal request as plain HTTP and advertised your resource URLs as `http://`. The standard fix was middleware that injected forwarded headers:

```js
// V1-era workaround
app.use('*', async (c, next) => {
  c.req.raw.headers.set('X-Forwarded-Proto', 'https')
  c.req.raw.headers.set('X-Forwarded-Host', baseUrl.host)
  await next()
})
```

**V2 ignores those headers.** So the workaround silently stops doing anything, your resource URLs revert to `http://`, and the middleware that was supposed to prevent exactly that is still sitting in your file looking load-bearing.

The V2 fix is the explicit `resource` field per route, shown above. Runtime resolution is effectively `routeConfig.resource || adapter.getUrl()` — anything you set explicitly wins, anything you don't falls back to the derived (wrong) URL. Set it on every paid route; there is no global override.

Once that's in place, delete the forwarded-header middleware. I still had mine months after the migration, which is how this section came to be written.

---

## Verifying your 402

This is where people think the migration broke something.

A correct V2 402 returns an **empty JSON body** — literally `{}`. The challenge lives in the `payment-required` response header, base64-encoded. If you're used to V1's body-carried challenge, an empty body reads like a bug.

To check it:

```bash
curl -s -i -X POST https://your-api.example/api/endpoint \
  -H "Content-Type: application/json" \
  -d '{"input":"check"}' \
  | grep -i payment-required \
  | sed 's/payment-required: //' | tr -d '\r' | base64 -d | jq .
```

You're looking for `x402Version: 2`, your CAIP-2 network (`eip155:8453` for Base mainnet), the correct asset and `payTo` address, and a `resource` field that starts with `https://`.

---

## Two things worth adding while you're in there

Neither is required by V2, but both are easier to do during a migration than to retrofit.

**Validate input *before* the payment gate.** Register your validation middleware ahead of the payment middleware so a malformed request gets a `400` instead of a `402`. Otherwise you're asking an agent to pay before you tell it the request was never going to work — which is bad behavior, and worse when the payer is an automated client with no human to notice.

**Guard the startup window.** The payment handler is built asynchronously (it constructs a facilitator client), so there's a window where the server can accept connections before the handler exists. A delegate middleware that returns `503 service initializing` until the handler resolves is cheap insurance against serving unprotected routes during a restart.

---

## Client side

The client rewrite is smaller than the server one, and the same subpath trap applies:

```python
from x402 import x402ClientConfig, SchemeRegistration
from x402.mechanisms.evm.exact.client import ExactEvmScheme
from x402.http.clients.requests import wrapRequestsWithPaymentFromConfig

config = x402ClientConfig(
    schemes=[
        SchemeRegistration(
            network="eip155:8453",
            client=ExactEvmScheme(signer=account),
        ),
    ],
)

session = wrapRequestsWithPaymentFromConfig(requests.Session(), config)
resp = session.post(ENDPOINT, json={"input": "..."}, timeout=90)
```

That's the entire integration. The wrapper handles the whole dance — sends the unpaid request, receives the 402, decodes the header challenge, signs the EIP-3009 authorization, retries. You never hand-parse a challenge.

### The settlement receipt is also in a header

Symmetric with the request side, and easy to miss: on success, settlement details come back base64-encoded in an `X-PAYMENT-RESPONSE` header (some stacks emit `payment-response` — check both). Decode it for the transaction hash:

```python
settle = resp.headers.get("X-PAYMENT-RESPONSE") or resp.headers.get("payment-response")
info = json.loads(base64.b64decode(settle))
print(f"https://basescan.org/tx/{info.get('transaction')}")
```

If you're logging anything about payments, that header is where the on-chain proof lives.

### Your payer wallet doesn't need ETH

Worth stating plainly because it surprises people: **EIP-3009 is gasless for the payer.** The buyer signs a transfer authorization; the facilitator submits and pays gas. A payer wallet needs USDC on Base and nothing else — no ETH, ever.

This is also why a legitimate agent wallet can show zero self-submitted transactions on a block explorer while having successfully paid you many times. When I got my first payment from an unfamiliar address, its transaction history was empty, which reads as suspicious until you remember the payer never submits anything.

Keep your V1 client around as a `.bak` until the V2 one settles. Mine earned its keep.

---

## Discovery: the part that's actually required

Migrating to V2 isn't sufficient for x402scan. It also wants a discovery document, and in practice **`/openapi.json` is what gets crawled.**

An OpenAPI 3.1 document, served *before* your payment gate, with per-endpoint payment metadata:

- `x-payment-info` carrying a fixed decimal USD amount (`0.010000`, not `0.01` — use consistent decimal precision)
- `protocols: [{ x402: {} }]`
- request and response schemas
- the 402 response documented
- free endpoints (`/health`, discovery routes) marked `security: []`

Three things I'd flag:

**Match your response field names to your actual handlers.** My OpenAPI advertised generic field names while the handlers returned `embedding`, `summary`, and `answer`. A crawler probing the documented shape gets a mismatch.

**Unify your request field names before you publish the spec.** My three endpoints had grown independently and took `input`, `text`, and `question` respectively — while the OpenAPI doc advertised `input` for all three. Two of the three would have failed any automated probe. Standardizing on one field name across every endpoint is worth doing regardless; a public spec makes it mandatory.

**Two different price representations coexist, and it's easy to mix them up.** The V1-shaped `/.well-known/x402` manifest expresses price in atomic token units as a string — `'10000'` for $0.01 USDC, which has six decimals. The OpenAPI `x-payment-info` block expresses it as decimal USD — `'0.010000'`. Same price, two encodings, and no error if you put the wrong one in the wrong place.

Worth noting: my `/.well-known/x402` manifest was still V1-shaped (`x402Version: 1`, `network: 'base'` rather than `eip155:8453`) at registration time and nothing complained, because registration went through `/openapi.json`. If you're short on time, prioritize the OpenAPI doc — but know you're carrying a stale manifest.

---

## Two things that cost me time and aren't written down anywhere

**The CDP facilitator has a minimum price floor.** Price anything below it and settlement fails. I ended up at $0.01 / $0.02 / $0.04 across three endpoints, which sits just above it. If your business model assumed sub-cent pricing, verify against the current floor before building around it.

**`x402.org`'s facilitator is testnet-only.** It's the obvious default when you're reading the spec, and it will not settle on mainnet. For Base mainnet you need CDP's, or another mainnet-capable facilitator.

---

## Migration checklist

- [ ] Confirm you're on V1 (check for `invalid_payload` errors in your logs)
- [ ] Install `@x402/*` scoped packages with `--legacy-peer-deps`
- [ ] Use the correct subpaths: `@x402/core/server`, `@x402/evm/exact/server`
- [ ] Rewrite handler: `HTTPFacilitatorClient` → `x402ResourceServer` → `.register('eip155:8453', new ExactEvmScheme())` → `paymentMiddleware`
- [ ] Delete any JWT/internal-path hack; use `createFacilitatorConfig`
- [ ] Restructure routes: method+path keys, `accepts` object, sibling `description`/`mimeType`
- [ ] Add explicit `resource` field per paid route (fixes `http://`)
- [ ] Delete the now-dead `X-Forwarded-Proto` middleware
- [ ] Verify 402 via the `payment-required` header, not the body
- [ ] Move input validation ahead of the payment gate (400, not 402)
- [ ] Rewrite the client; keep V1 as `.bak`
- [ ] Confirm on-chain settlement with a real (small) payment — tx hash is in the `X-PAYMENT-RESPONSE` header
- [ ] Unify request field names across endpoints
- [ ] Publish OpenAPI 3.1 at `/openapi.json`, before the payment gate
- [ ] Match documented response fields to actual handler output
- [ ] Submit origin to x402scan

---

## If you're starting now

Do the OpenAPI document first, even before the code migration. It forces you to confront your API's inconsistencies — mismatched field names, undocumented response shapes, endpoints that grew independently — while you can still fix them freely. I did it last and had to go back and standardize things I'd have caught earlier.

And migrate before you need to. The V1 stack still works right up until the moment discovery matters, and by then you're doing an urgent migration instead of a calm one.

---

## Corrections and additions welcome

The x402 package line moves fast, and some of this will go stale. If something here is wrong or has changed, open an issue or a PR — I'd rather this stay accurate than stay mine.

---

*Written by Landy Ukena. I run [x402ai](https://api.x402ai.dev) — three payment-gated AI inference endpoints on Base mainnet, self-hosted and settling real USDC via the Coinbase CDP facilitator. [LinkedIn](https://www.linkedin.com/in/landyukena/)*
