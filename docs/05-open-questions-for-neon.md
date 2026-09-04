# 05 — Open questions for Neon

Questions worth sending to Neon before taking this to production, grouped by
whether they block a launch. Everything here is something the published
documentation did not settle.

## Blocking

1. **Sandbox lifecycle.** Is there a dedicated sandbox environment or key prefix,
   and test payment instruments for each supported method? The implementation
   reads `isSandbox` on the event but does not yet enforce environment separation,
   because I do not know whether sandbox and production share a webhook listener.

2. **Ignorable events.** Neon retries any non-2xx for up to 36 hours. What is the
   recommended response for an event a merchant intentionally does not handle —
   2xx to stop retries, or a specific status? Same question for a
   `purchase.completed` whose `externalReferenceId` is unknown to us (which, if it
   ever happens, is unrecoverable by retry).

3. **Dispute semantics, and partial refunds.** Refunds are now handled — see the
   answered section below. What is still open: `dispute.opened` is version 1 and
   carries only a `purchaseId`, so does `dispute.closed` report an outcome we can
   act on, and is the merchant expected to revoke on open or on an unfavourable
   close? Separately, can a single-item purchase be partially refunded, and if so
   what does `items[].quantity` look like?

4. **KR payment methods.** Which Korean rails are live today (카카오페이,
   네이버페이, 토스, carrier billing, 상품권), and does the set differ by checkout
   type? See [04](04-korea-market-notes.md).

## Important, not blocking

5. **Item schema.** Are `bundleContents` and `taxCode` valid on the checkout item
   payload? They are sent by this implementation and appear on the purchase event,
   but I could not confirm them in the checkout request reference — I would rather
   remove them than send fields that are silently ignored.

6. **Checkout response fields.** The reference documents `redirectUrl`; the
   embedded flow needs `checkoutId` and a client token. Are all three always
   returned, so one server endpoint can serve Hosted, Embedded, and Direct
   without branching?

7. **Signature details.** Confirmed: `x-neon-digest`, HMAC-SHA256 over the raw
   body. Is there a timestamp component or replay window, and is there a key
   rotation procedure (dual secrets during rollover)?

8. **`accountId` and `POST /auth/token`.** For a game with no login — this one
   uses an `HttpOnly` cookie — what is the recommended identity strategy, and does
   Neon deduplicate accounts across it?

9. **Country resolution.** Given that currency must match `playerCountry`, does
   Neon resolve the player's country itself on the hosted page, and what happens
   if the value we send disagrees with where they actually pay from?

## Worth knowing

10. **Idempotency on our side vs. yours.** We dedupe on `event.id`. Is `event.id`
    stable across retries of the same delivery, and unique per delivery attempt?
11. **Webhook source IPs or mTLS**, if any, for allowlisting.
12. **Rate limits** on `POST /checkout`.
