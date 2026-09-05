# 05 — Remaining questions for Neon

Updated 2026-09-05. Earlier open questions mixed historical gaps with current
ones. These are the questions that still affect production behavior.

1. **Checkout lifecycle and duplicate sessions.** How should Hosted clients obtain
   the checkout ID used by the documented GET/expire endpoints when the recorded
   sandbox creation response contains redirectUrl and token but no checkoutId?
   Which states are safe to replace, and what guarantees apply when the same
   account opens a second checkout for the same permanent SKU?
2. **Reconciliation.** What is the recommended recovery path when a signed
   purchase arrives without a matching local intent, or checkout creation returns
   an uncertain network result? The current implementation logs unknown purchase
   references and retains early refunds by purchase ID.
3. **Refunds and disputes.** Partially resolved 2026-09-06 ([09](09-sandbox-run.md)
   addendum): item-level refunds (`{"items":[{"itemId": purchase.items[].id,
   "quantity": n}]}`) succeed, and a real `refund.processed` revoked the
   entitlement end to end. Still for Neon: the **empty-body full-refund path
   returns 500** in sandbox (Console and API alike), and the purchase object
   names the id `items[].id` while the refund request wants `itemId`. Disputes
   remain untriggerable from the merchant side — define whether they revoke on
   open or final loss.
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
