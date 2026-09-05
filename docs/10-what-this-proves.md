# 10 — What this proves, stage by stage

> Current implementation and verification limits: [12 — Review](12-review.md).

A claims ledger. Each stage says what it demonstrates, what evidence backs it, and
what it deliberately does not cover. The point of separating them is that "it
works" is five different claims, and they were earned at different times by
different means.

---

## Stage 1 — A checkout that cannot be tampered with

**Claim.** A player can buy an item, and nothing the browser says can change what
they are charged or what they receive.

**Evidence.**

- The client sends `{ sku, locale }`. Price, currency and billing country are
  resolved server-side from a frozen catalogue. The suite posts a deliberate
  `price: 1, country: 'US', currency: 'USD'` alongside a valid SKU and asserts all
  three are ignored.
- Entitlements are written in exactly one place: a `purchase.completed` webhook
  whose HMAC-SHA256 digest verifies against the raw body, and whose account, SKU,
  quantity and amount match an intent recorded before the player left.
- Typing `?purchase=return` by hand grants nothing.

**Not covered.** The player identity is a bearer credential bound to a device, not
an account. Stage 5 is where that changes.

---

## Stage 2 — Behaviour that survives the failure cases

**Claim.** The integration behaves correctly when things go wrong, not only when
they go right.

**Evidence.** 30-plus assertions, run against **both** storage backends:

| Failure | Behaviour |
|---|---|
| Forged or absent signature | `403`, logged |
| Same event delivered twice | granted once |
| A different event for an already-fulfilled checkout | refused |
| Wrong account, wrong amount, wrong environment | refused, nothing written |
| Unknown reference, unhandled event type, malformed body | `200 {ignored}` + reason |
| Storage failure | `5xx`, so Neon retries |
| A late purchase webhook after a refund | refused — a revoked item cannot be resurrected |
| Buying a permanent item you already own | `409`, no second charge |

The split between `200 {ignored}` and `5xx` is the whole design: Neon retries
non-2xx for up to 36 hours, so the question at every branch is not whether
something succeeded but whether a retry could change the answer.

**Not covered.** Dispute handling. `dispute.opened` is version 1 and carries only
a `purchaseId`; whether a chargeback should revoke on open or on an unfavourable
close is a studio policy decision, not a technical one.

---

## Stage 3 — It works against the real Neon, and the ledger is correct under concurrency

**Claim.** This is not a mock that resembles an integration.

**Evidence.** Two sandbox purchases completed and were fulfilled by signed
webhooks — `9BKP-47RY-KJLL` via Naver Pay, `29QS-Y3S5-VW7H` via card. Both
verified against the recorded intent including the amount. Full account in
[09 — Sandbox run](09-sandbox-run.md), with screenshots in
[evidence](evidence/).

The live run also caught two defects that a passing suite on two backends had
agreed were fine: a field production never sends (`checkoutId`), and an API that
would sell a permanent item twice.

For concurrency: `JsonRepository` gets its idempotency from a promise queue inside
one process, which stops being true the moment there is more than one instance.
`FirestoreRepository` puts the whole fulfillment inside a transaction, and passes
the same suite.

**Not covered.** A real `refund.processed` event. Neon's sandbox refuses every
refund with a `500`, through both the Console and the REST API, on two payment
methods — reported in [09](09-sandbox-run.md). Revocation is therefore verified
against a synthetic event built from real purchase data, and that distinction is
kept rather than blurred.

---

## Stage 4 — A service, not a demo server

**Claim.** The payment API is deployable on its own terms, and does not care what
kind of client is calling it.

**Evidence.**

- `server/index.mjs` is a standalone entry point. It serves the API and health
  endpoints and **nothing else** — the check asserts that `/`, `/index.html` and
  `/dist/game.js` all return `404`. The game bundle is a cached static asset; this
  process holds secrets. They had no business sharing a port.
- **Configuration is judged once, at boot.** A service that cannot serve payments
  refuses to start rather than failing at the player's first tap, which is the
  most expensive moment to discover it. Missing key, missing webhook secret,
  missing public URL, or the contradiction of mock payments in production — all
  fatal. The dev server treats the same problems as warnings, because opening the
  game with no credentials is the normal development path.
- `/healthz` answers liveness; `/readyz` reads the ledger and reports which
  backend and environment are live. `SIGTERM` clears readiness first, so a load
  balancer stops routing while an in-flight fulfillment finishes writing.
- Logs switch to JSON on `LOG_FORMAT=json`, with fields on the events worth
  querying later. Grants and refusals carry the account and purchase ids.
- Identity accepts a bearer token before falling back to a cookie, and CORS is an
  explicit allowlist without `Allow-Credentials` — so a game on a CDN, a Unity
  client, or a launcher all reach the same service without it changing.

**Not covered.** Nothing is deployed. The reasoning is in
[08](08-storage-and-identity.md): deployment was buying a public webhook endpoint,
and a tunnel bought that too. The `Dockerfile` runs the service entry point, so
deploying later is a command rather than a project.

---

## Stage 5 — A purchase that survives the device

**Claim.** What a player bought belongs to them, not to the browser they bought
it in.

**Evidence.** A transfer code moves the account: `POST /api/account/transfer-code`
issues one (stored only as a SHA-256 hash, shown once, single-use, 24-hour
expiry), and `POST /api/account/claim` lets a device with no cookie and no token
adopt that account and read the entitlement it already owned. Nothing is
migrated — the second device simply presents the id that always held the grant.

Server saves move with it, versioned rather than timestamped: a write carrying a
stale `baseVersion` is refused with `409` **and the server's current copy**, so
two devices on one account cannot silently delete each other's progress. Device
preferences — graphics, key bindings, language — deliberately stay local.

The guided tour performs this live, with `credentials: 'omit'` standing in for a
device that has never been here.

**Not covered.** Who owns the account. A transfer code is a transfer mechanism,
not authentication: whoever holds it holds the account. Email or OAuth belongs in
that slot for a real title, and the reasoning for not building it here is in
[11](11-accounts-and-saves.md).

---

## What each stage cost to be sure of

Worth recording, because the effort was not where it looked like it would be.

| Stage | Where the confidence came from |
|---|---|
| 1 | Reading Neon's documentation, then writing tests that attack our own API |
| 2 | Writing the failure cases before believing the happy path |
| 3 | Actually paying, twice, with real money-shaped money |
| 4 | Asking what would break if this ran as more than one process |
| 5 | Asking what a player loses when they change phones |

Every stage was green on the previous stage's tests when the next stage's problems
were found.
