# 05 — Remaining questions for Neon

Updated 2026-09-06. Earlier open questions mixed historical gaps with current
ones. These are the questions that still affect production behavior.

1. **Checkout lifecycle and duplicate sessions.** Which checkout states are safe
   to expire through the documented expire endpoint, and what happens when the
   same account opens a second checkout for the same permanent SKU before the
   first settles: is there a server-side guard, or is that the integrator's to
   build? (Today the service refuses a checkout only after a grant exists.)
   *Corrected 2026-09-06:* an earlier version of this question said the
   creation response carried no checkout id. It does: the field is `id`
   (`{ id, token, redirectUrl, externalProvider }`, confirmed with a direct
   sandbox call). Our adapter had been reading `checkoutId`; fixed the same day
   in `server/store-api.mjs`.
2. **Reconciliation.** Two halves. The local half is ours: the intent is
   recorded after the create call today (`createNeonCheckout`, then
   `recordCheckout` in `server/store-api.mjs`), so a create that times out after
   Neon created the checkout leaves an orphan on Neon's side; before going live
   the intent should be written first as `creating` and marked `pending` on
   success, with `creating` records expiring on the same TTL. The half only Neon
   can answer: can purchases be listed by `externalReferenceId` or `accountId`,
   so a nightly job can catch a signed `purchase.completed` that arrived with no
   matching local intent? Today such events are acknowledged 200 and logged, and
   early refunds are retained by purchase ID.
3. **Refunds and disputes.** Partially resolved 2026-09-06 ([09](09-sandbox-run.md)
   addendum): item-level refunds (`{"items":[{"itemId": purchase.items[].id,
   "quantity": n}]}`) succeed, and a real `refund.processed` revoked the
   entitlement end to end. Still for Neon: the **total-refund path
   returns 500** in sandbox (Console and API alike) — `{}` and the documented
   `{"fee": 0}` both, while the same endpoint answers `400`/`415` cleanly for
   genuinely malformed bodies, so this is a handler failure and not a rejected
   request ([09](09-sandbox-run.md), 2026-09-07 re-verification). The purchase
   object also names the id `items[].id` while the refund request wants
   `itemId`. `dispute.opened` / `dispute.closed` are version-1 events
   carrying only a `purchaseId`, with no merchant-side trigger in the sandbox:
   is there a test trigger, does `dispute.closed` carry an outcome, and is a
   v2 shape with account and items planned? Our default would be to freeze the
   entitlement on open and revoke on a lost close.
4. **Production operations.** Confirm secret rotation overlap, outbound API rate
   limits, and supported reconciliation/monitoring procedures.
5. **Market rollout.** Confirm currently enabled Korean payment methods for the
   merchant account and checkout type; the methods in [09](09-sandbox-run.md) are
   observations from that sandbox run, not a universal availability guarantee.

## Resolved or clarified

- Environment checks are implemented. Firestore uses separate namespaces; JSON
  still needs separate data directories for environment isolation.
- The [create-checkout reference](https://docs.neonpay.com/reference/createcheckout)
  explicitly lists bundleContents (default empty) and taxCode (digital_goods).
  Earlier notes saying these fields were unconfirmed describe the earlier review.
  The single cosmetic payload remains the smaller shape already used in sandbox.
- The [webhook guide](https://docs.neonpay.com/docs/webhooks-and-callbacks) says retry
  data is identical and non-2xx requests are retried for up to 36 hours. Local
  deduplication uses event ID and checkout state.
- The [checkout details](https://docs.neonpay.com/reference/getcheckout) and
  [expire endpoint](https://docs.neonpay.com/reference/expirecheckout) are documented.
  Do not invent a local expiry duration: the documented default can be configured
  per merchant, and cancellation of a browser page does not prove payment failed.

See [12 — Current review](12-review.md) for implementation limits and test results.
