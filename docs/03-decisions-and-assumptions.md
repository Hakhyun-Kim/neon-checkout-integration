# 03 — Decisions and assumptions

> Current implementation and verification limits: [12 — Review](12-review.md).

## Why this app

A blank starter project would have been easier. This integration went into a game
that was already built and shipped instead, because the interesting part of a
payments integration is not the happy path in a clean sample — it is fitting
payments into a codebase that already has rules. This one had three that mattered:

- `src/engine/` must stay free of DOM, Three.js, and audio so the Node
  verification scripts can import it. Payment code therefore lives entirely
  outside the engine.
- The game ships as a static bundle on GitHub Pages. The store had to degrade to
  invisible rather than broken when no server exists.
- Everything is bilingual (ko/en) with stable save ids across languages, so the
  store had to be localized from the first commit, not retrofitted.

## Why Hosted Checkout

| Option | Why not / why |
|---|---|
| **Hosted** ✅ | Chosen. The redirect + webhook pair is the part of the integration with real failure modes worth demonstrating, and it is the same flow a desktop or mobile client would use. |
| Embedded | Would add `@neonpay/js` and an iframe, but the server work is identical. `POST /api/store/checkout` already returns `checkoutId`/`token` when Neon supplies them, so this is a client-side change only. |
| Direct | Same server contract; differs by opening the `redirectUrl` in an external browser and returning via deep link. Relevant for this game's Electron build. |
| Storefront | Out of scope here — this integration deliberately does not build a web shop. |

The server was written so that the choice of checkout surface is a client
decision, not an architectural one. That was the point.

## Why cosmetics, and only cosmetics

The three SKUs (`CELESTIAL_BANNER`, `AURORA_SPIRES`, `GOLDEN_SENTINELS`; the
banner was the only one until 2026-09-05) are permanent, purely visual, and have
no combat effect. Three reasons:

1. **Fulfillment is provable.** A cosmetic either appears or does not. A currency
   pack would blur "did fulfillment work" with "did the economy apply it".
2. **No probability disclosure.** A random or lootbox item would pull in Korea's
   확률형 아이템 disclosure obligations — see [04](04-korea-market-notes.md).
3. **It is idempotent by nature.** Granting a permanent flag twice is harmless,
   which makes the replay test unambiguous. A consumable would need a stronger
   claim about exactly-once semantics than a JSON file can honestly make.

## Price representation

Neon expresses price as 100× the currency's base unit: `KRW: 490000` for ₩4,900,
`USD: 499` for $4.99.

KRW is worth calling out. The won has no circulating subunit, so ₩4,900 encoded as
`490000` looks wrong to a Korean engineer's eye and is a standing invitation to a
100×-off bug. The multiplier is therefore written once, in one frozen table, and
the human-readable string is **derived** from the integer through
`Intl.NumberFormat` rather than typed by hand:

```js
formatPrice(490000, 'KRW')  // "₩4,900"  — 0 decimals, correct for KRW
formatPrice(499, 'USD')     // "$4.99"
```

An earlier draft hardcoded `'₩4,900'` next to the integer, and the test asserted
that hardcoded string — so price and display could drift apart with every test
still green. The test now compares against `formatPrice(...)`, so the assertion
fails if the integer and its display ever disagree.

## Country and currency

Neon enforces one currency per country and requires `currency` to match
`playerCountry`. Getting this wrong is not cosmetic: it declares the player's tax
jurisdiction and decides which payment methods they are offered.

Billing country is resolved **server-side**, in this precedence:

1. an explicit player choice, stored in a cookie (the picker in the store modal);
2. a platform geo header (`cf-ipcountry`, `x-vercel-ip-country`, …), and only
   where `TRUST_GEO_HEADERS=1` declares that a proxy sets them and strips the
   client's own copy — otherwise the header is caller-supplied input that lets
   any caller choose its own tax jurisdiction;
3. `KR` as the default.

**No language signal appears in that list, at any position.** An earlier draft
derived country from the game's ko/en toggle, which declared a Korean player
reading English to be in the US. The first correction replaced it with the
region subtag of `Accept-Language` — the same mistake one layer down. A browser
language is not a location: `en-US` is what an English-configured machine sends
from Seoul as readily as from Seattle, and on a deployment with no CDN in front
of it that was the only inferred signal that ever fired, so it decided. The
client no longer sends a country at all — a SKU and a display language, nothing
else — and no inference reaches `playerCountry`.

### A weak signal may recommend a market; it may not declare one

Removing the browser region outright costs something real: a visitor outside
Korea meets KRW pricing and a foreign card before they find the market picker.
That demand is legitimate, so the signal was kept and demoted. The catalogue
response carries a `suggestion` built from the `Accept-Language` region, and the
store renders it above the prices as a single sentence with one button — *"Your
browser looks like US. Switch billing to USD?"* — annotated with why it is a
question rather than a default: billing region sets tax and payment methods.
Accepting it posts `/api/store/market`, which is the explicit selection from
step 1, and the suggestion disappears. The suggestion is also suppressed
whenever it agrees with the resolved market or the player has already chosen.

The distinction is the point: an inference decides **what to offer**, a person
decides **what is declared to a merchant of record**. A country is never rewritten to
reach a price, either: this catalogue prices two markets, and every other
country is handled by **Neon's Global Store** in USD — `marketFor` falls back to
a global USD market rather than to `KR`, so a player in Japan is declared `JP`
with USD pricing instead of being declared Korean on the way to a price row.
Only real country codes are accepted; the ISO user-assigned ranges (`AA`,
`QM`-`QZ`, `XA`-`XZ`, `ZZ`) are refused, which is exactly what geolocation
returns when it does not know — Cloudflare's `XX`, or `T1` for Tor. Where the player actually
is should come from IP — Neon exposes localized pricing by IP — and that is a
deployment change rather than a code one: `deploy/README.md` carries the
runbook for putting a geolocating proxy in front of the service, closing the
direct path so the header cannot be forged, and verifying both halves;
`deploy/cloud-run.sh --trust-geo-headers` flips the flag.

Regression tests cover each edge: `locale=en` moves nothing, an `en-US` browser
resolves `KR` and is offered `US`, a `ko-KR` browser is offered nothing,
accepting the offer resolves `US` and silences it, a forged `cf-ipcountry` is
ignored until the deployment trusts its proxy, and an explicit selection
outranks every inferred signal. The flow was also walked in a browser: the
prices move from ₩ to $ on the click, and back through the picker.

## successUrl must be the player's own origin

`successUrl` is built from `PUBLIC_URL`. If that origin differs from the one the
player is actually browsing, the session cookie does not travel with the redirect,
a fresh account id is minted on return, and the purchase belongs to nobody.

This is not hypothetical — it happened during verification. The game was open at
`http://localhost:8642` while `PUBLIC_URL` defaulted to `http://127.0.0.1:8642`,
and fulfillment came back `404 checkout not found`. Two changes followed:
`PUBLIC_URL` now defaults to the origin the request actually arrived on, and when
it *is* configured and disagrees with that origin, the server warns at checkout
time. For a tunnelled sandbox this reduces to one rule: open the game at the
tunnel URL, never at localhost.

## Webhook response codes

Neon retries any non-2xx response for up to 36 hours. The handler therefore splits
failures by whether a retry could ever help:

| Situation | Response |
|---|---|
| Invalid or missing signature | `403` — deliberately noisy; a misconfiguration, not traffic |
| Unhandled type, wrong version, environment mismatch, malformed body | `200 {ignored}` |
| Unknown reference, account/SKU/amount mismatch, already-fulfilled checkout | `200 {ignored}` |
| Storage or network failure | `5xx` — a retry genuinely might succeed |

Every ignored case is logged with its reason, so "accepted" and "quietly dropped"
stay distinguishable in the log.

## Storage

A single JSON file behind a serialized write queue, with atomic
temp-file-then-rename, plus pruning of stale pending checkouts and of the
idempotency ledger (30-day window, comfortably longer than Neon's 36-hour retry).
Chosen because it is auditable in review — a reader can open
`.data/neon-store.json` and see exactly what happened.

It is still not production storage: single-process, no cross-collection
transactions. The whole surface is one file (`server/repository.mjs`), so
replacing it with Postgres is a contained change.

## Verification

`npm run store:check` runs the API against a real HTTP server on an ephemeral port
and asserts:

- the catalogue is localized and priced by the server, with the display string
  derived from the integer;
- switching UI language does **not** move the billing country;
- no language signal moves the billing country, an explicit choice does, the
  browser region only produces a suggestion, a forged geography header does
  nothing until the deployment trusts its proxy, and an unsupported country is
  rejected;
- a client-supplied `price`, `country`, and `currency` are all ignored;
- a correctly signed `purchase.completed` is fulfilled;
- the same event delivered twice grants once, and a *different* event pointing at
  an already-fulfilled checkout grants nothing;
- unknown references, foreign event types, environment mismatches, wrong accounts,
  wrong amounts, and malformed bodies all return `200` with a reason;
- a bad or absent signature returns `403`;
- checkout creation is rate-limited.

It is wired into the project's existing `npm run check` gate — 29 verification
scripts plus the build (listed in the code repo's `scripts/README.md`) — which
passes.

Beyond the automated gate, the flow was driven in a real browser: buy → redirect →
return → poll → owned, with the network log, the server log, and the ledger file
checked at each step. That is how the origin-mismatch bug above was found, and how
a CSS regression was caught in which the product description column collapsed to
one character wide inside the 430px modal.

## What is still not done

Honest list. None of it blocks the demo; all of it is known.

| Gap | Impact |
|---|---|
| No dispute handling | `dispute.opened` is version 1 and carries only a `purchaseId` — no account, no items. Revoking on open versus on an unfavourable close is a product policy call, and the `dispute.closed` outcome field is still unverified. Refunds are handled. |
| Amount check is partial | Verified when settlement currency equals the currency the checkout was created in. If the player switches country on the hosted page, the purchase is recorded with a `currencySwitched` flag rather than re-verified. |
| `taxCode` / `bundleContents` removed | They were being sent without confirmation in the checkout request reference. Removed rather than guessed; now a question instead of an assumption. |
| No real accounts | Player identity is an `HttpOnly` cookie. Clearing cookies loses entitlements. A real title would use its own player id plus `POST /auth/token`. |
| Rate limit is per-account, in-ledger | Fine for one process; a distributed deployment needs a shared limiter. |
| Store button sits behind the journey-screen overlay | Pre-existing game chrome: the 도감 and 별의 축복 buttons share that topbar and are equally covered on that screen. Not introduced by the store, but it means the button is reachable from the encounter phase, not the map. |
