# 12 — Current integration review (2026-09-05)

Scope: this week's changes through game commit 783c4cd, compared with the parent
of cf309c6, and documentation through 9c7238d. Corrections below were prepared for the 2026-09-05 repository handoff. Earlier documents describe stages of development;
this page qualifies their current claims.

## Separation and reuse

Payment code lives in `server/`, two app modules, markup/styles and startup wiring.
No simulation, tactics or balance files changed. The smallest reuse boundary is
`server/neon-client.mjs`, an injected-fetch HTTP adapter with no game dependencies.
The complete service also includes application policy: catalog, identity, markets,
storage, refunds, transfer codes and saves. Porting means replacing those policies.
The banner UI and tour are game-specific adapters.

The initial checkout example has grown substantially. Accounts, saves, Firestore
and the tour are additional scope; the whole integration is not a 350-line SDK.

## Corrections

| Problem | Change and evidence |
|---|---|
| Development server served repository files, including credentials and ledger | Public build allowlist; real HTTP private-path regression checks |
| JSON disk failure left an uncommitted grant in memory | Clone, persist, then publish; filesystem failure/retry/reopen regression |
| Refund before purchase mapping was discarded | Retain purchase-ID tombstones; both backends suppress a later grant |
| Firestore completed checkouts retained pending TTL | Clear expiresAt on fulfilled/refunded records to preserve refund lookup |
| English return URL lost locale | Preserve lang on return/cancel; remove only callback parameters |
| Tour modified wave/HP and implied mock signature verification | Observe normal gameplay; label mock grant and entitlement count accurately |
| Tour could overlap requests and repeat completed state | Serialize navigation; end automatic playback at step 13 |
| Launcher could inherit hosted settings | Explicit mock/sandbox/JSON selection; tour refuses hosted mode |

Neon's [Hosted Checkout overview](https://docs.neonpay.com/docs/about-hosted-checkout)
and [webhook guide](https://docs.neonpay.com/docs/webhooks-and-callbacks) were checked
on 2026-09-05. The hosted page is a separate Neon surface; HMAC authenticates
webhooks and non-2xx deliveries are retried for up to 36 hours. The mock tour
does not prove the hosted page or valid signature delivery.

## Remaining production work

1. **Concurrent pending sessions:** `already_owned` rejects checkout after an
   entitlement exists. Two concurrent requests were reproduced as HTTP 201/201 with two provider calls. Actual double charging was not tested; Neon may invalidate older sessions.
   Add atomic reservation and expiry/reconciliation. Define refunds when multiple
   purchases cover one entitlement; current revocation removes it unconditionally.
2. **Cross-origin markets:** bearer identity works, but explicit market choice is
   cookie-based. Native/cross-origin clients need another supported preference contract.
3. **Saves:** endpoints are tested; game save/load buttons remain local. A supplied
   stale baseVersion is rejected, but omitting it permits an unconditional write.
4. **Transfer codes:** individual claims are transactional. Concurrent Firestore
   issuance can retain two codes because the query precedes the write batch.
   Bearer account IDs are credentials, not public identifiers or verified users.
5. **Environment isolation:** Firestore namespaces environments; JSON uses one
   file path. Use separate data directories for sandbox and production.
6. **Recovery:** the outbound request now has a 10-second timeout. Still add recovery for polling network errors,
   and reconciliation for unknown purchase references. Persisted early refunds
   are handled separately from unknown purchase events.

Embedded, Direct, native deep links, deployment and dispute policy are not
implemented merely by choosing another client. These are limits, not guarantees.

## Verification

- `npm.cmd ci`: passed; 26 development-inclusive advisories reported.
- `npm.cmd audit --omit=dev`: zero runtime advisories.
- Initial `npm.cmd run check`: passed before the defects above were found.
- `node scripts/balance-check.mjs 60`: all nine combinations passed (540 runs).
  Normal novice completion was 38%; normal regular completion was 90%.
- Updated `npm.cmd run store:check`: passed on JSON and Firestore emulator.
- `npm.cmd run service:check`: passed, including private-file HTTP checks.
- Desktop English tour: all 13 steps manually advanced; observed 201, forged 403,
  Owned, transfer, duplicate:true, owned 409, revoked:true and late-grant ignored.
  Browser warning/error log was empty.

No new sandbox charge or refund was performed in this review; the refund
blocker was resolved the next day ([09](09-sandbox-run.md)).

Same-day follow-ups, in order:

- Full `npm.cmd run check` passed after the fixes; the revised store suite
  passed on JSON and the Firestore emulator.
- 390×844: store and tour scroll separately, Owned visible, no horizontal
  overflow. Mock Buy returned to `?lang=en` with Owned. Console clean.
- Concurrent checkout creation reproduced with a controlled provider
  (`201/201`, two provider calls); the live vendor outcome was not tested.
- Added: Hosted-only redirect validation (token-only responses rejected), an
  abortable 10-second outbound timeout, own-property catalog lookup.
- The paid flag moved from the hidden legacy champion panel to the title HUD.
- Split-origin deployment needs the API origin in the shipped CSP
  `connect-src`; the API base meta tag alone is not enough.
- The full gate passed again after these changes; the emulator-backed
  repository tests predate the adapter-only edits and the JSON/HTTP suite was
  rerun. Game balance unchanged.

Not verified here: full accessibility-label translation. Existing screenshots
remain historical.

## Second review — 2026-09-06, before sending

A last adversarial pass over `server/` and the deploy script, after the live run.
Each row has a test in `scripts/store-server-check.mjs` or a step in
`deploy/cloud-run.sh`; the JSON suite and the Firestore suite (emulator) both pass.

| Found | Fix |
|---|---|
| The create-checkout response names the checkout id `id`; the code read `checkoutId` and stored `undefined`. This had been on the question list for Neon in [05](05-open-questions-for-neon.md) as "no checkoutId in the response". | Read `id` first; the check asserts the recorded id and the 201 body. |
| The amount check compared against the original currency (`initialCurrency`), so after a country switch on the hosted page a purchase settled in another currency would have been refused as `amount does not match` and acknowledged 200: paid, never granted. | Compare only when the settled currency equals the checkout currency; record `currencySwitched`; a KRW intent settled as 499 USD is now granted and flagged. |
| A second paid purchase of an already-owned permanent item overwrote the grant, and refunding either purchase removed the item. | The first grant stays; the duplicate is recorded with `duplicateGrant: true`; revoke removes the entitlement only for the purchase that granted it. |
| Account ids are bearer credentials and were written to logs. | Logs carry a 12-character SHA-256 handle. |
| A malformed percent-encoded cookie from any app on the origin threw inside `decodeURIComponent` and turned every store call into a 500. | Decode defensively and split on the first `=`. |
| `expiresAt` was written on intents, dedup records, limiter docs and transfer codes, but no Firestore TTL policy existed, so they accumulated. | The deploy script enables TTL on the four collection groups (idempotent). |
| A redeploy without `--allowed-origins` reset CORS to the local defaults and silently broke the shared link at its first fetch. | The Pages origin is in the default; the smoke run checks the preflight. |
| A carried `returnPath` query could place its own `api=` ahead of the server's; the cookie `Secure` flag came from `PUBLIC_URL` rather than the request. | Reserved keys are stripped from the carried query; `Secure` follows `x-forwarded-proto`. |

Still open, unchanged: the checkout rate limit is keyed on the self-issued
account id (an address-based limiter is next), and a second pending checkout for
the same permanent item is still accepted until question 1 in
[05](05-open-questions-for-neon.md) has an answer.
