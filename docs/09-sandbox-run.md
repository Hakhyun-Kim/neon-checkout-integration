# 09 — Sandbox run: what it proved, and one blocker

A live run against the Neon sandbox on 2026-09-04, through a public tunnel with
real webhook delivery. Two purchases completed and were fulfilled. Refunds could
not be tested end to end because every refund attempt failed server-side — that
is written up as a defect report below, with the questions it raises.

Purchase identifiers are sandbox values. Account bearer credentials are redacted from current text; earlier history and screenshots were not rewritten.

---

## What completed

| | Purchase 1 | Purchase 2 |
|---|---|---|
| `purchaseId` | `97783f1a-e73e-4b87-b141-6eba04e187dc` | `66e506fe-7e0b-422f-9a8a-269f825fd555` |
| `orderNumber` | `9BKP-47RY-KJLL` | `29QS-Y3S5-VW7H` |
| Payment method | `naver_pay` (vendor `stripe`) | `unknown_card` (vendor `stripe`) |
| Amount | `490000` KRW (₩4,900) | `490000` KRW |
| Outcome | fulfilled by signed webhook | fulfilled by signed webhook |

Server log (account bearer credentials redacted):

```
[store] webhook fulfilled CELESTIAL_BANNER for [sandbox-account-A]
[store] webhook rejected: invalid signature
[store] webhook rejected: invalid signature
[store] webhook fulfilled CELESTIAL_BANNER for [sandbox-account-B]
```

The two rejections in the middle are a deliberate negative test: an unsigned and
a forged webhook posted at the live endpoint, both answered `403`.

Three things this confirmed that no local test could:

- **The amount check ran against a real event.** The recorded purchase carries
  `price: 490000`, `currency: "KRW"`, `currencySwitched: false` — the webhook's
  amount was compared against the intent recorded at checkout creation and agreed.
- **Fulfillment is bound to the account, not the browser.** Both payments were
  made in a different browser from the one that opened the checkout. The
  entitlement landed on the `accountId` recorded at creation, and the original
  browser saw it appear.
- **The token identity path works on a live server.** With a bearer token set in
  one browser, the same requests resolved to a different player than the cookie
  did: token → `b9e0d4f0` (owns nothing), cookie → `4bac1ed1` (owns the banner).
  That is the path Unity, Unreal and cross-origin web clients depend on.

## Answers the run produced

Several open questions were settled by looking at what actually arrived.

**Korean payment methods** — the hosted checkout page, rendered from
`languageLocale: "ko-KR"`, offered:

```
카드 · Google Pay · Samsung Pay · Kakao Pay · Naver Pay
```

카카오페이, 네이버페이 and 삼성페이 are live for KR. This is the first question a
Korean studio asks, and the answer is good.

**Tax is inclusive, 10%.** The checkout page showed `세금에 ₩445 포함`, and
`GET /purchases/{id}` confirms `taxAmount: 44500`, `taxRate: 0.1`,
`totalAmount: 490000`. The displayed price is what the player pays.

**Settlement is in USD, with roughly a 10% fee.**

```json
"fee": { "fxRate": 0.0007426716, "feeAmount": 37,
         "netProceedsAmount": 294, "settlementCurrency": "USD" }
```

₩4,900 settles as $2.94 net after a $0.37 fee. Worth knowing before a studio
prices in KRW.

**`checkoutId` exists, but not where you would look for it.**
`POST /checkout` returns only `{ token, redirectUrl }`. The `checkoutId`
(`61f11705-…`) appears on the purchase object retrieved later. This caused a real
defect on our side: we were recording `checkoutId: undefined`, which a JSON store
drops silently and Firestore rejects outright.

**Partial refunds are item-level.** `POST /purchases/{id}/refund` accepts
`items: [{ itemId, quantity }]`, and the purchase object exposes the `itemId`
(`bcd8b9c4-…`) and a `refundableQuantity` per item.

---

## Defect report — refunds fail in sandbox

**Summary.** Every refund attempt against a completed sandbox purchase fails with
a server-side error, through both the Console UI and the REST API, on two
different payment methods.

**Environment.** Sandbox. Merchant account provisioned 2026-09-03. Purchases made
2026-09-04 in KRW, `playerCountry: KR`.

**Steps to reproduce.**

1. Complete a checkout for a single item, KRW 490000.
2. Attempt a refund, either from the Console transaction view or via
   `POST /purchases/{purchaseId}/refund` with `X-API-KEY` and body `{}`.

**Expected.** The refund is accepted and a `refund.processed` webhook follows.

**Actual.** Console shows `Refund failed. Please try again or contact support.`
The REST API returns:

```json
{"statusCode":500,"code":"UNKNOWN_ERROR","message":"Unhandled error",
 "errors":[{"source":"server","message":"An error occurred. Please try again."}]}
```

**Affected purchases.**

| purchaseId | orderNumber | method | result |
|---|---|---|---|
| `97783f1a-e73e-4b87-b141-6eba04e187dc` | `9BKP-47RY-KJLL` | `naver_pay` / stripe | Console: failed · API: 500 |
| `66e506fe-7e0b-422f-9a8a-269f825fd555` | `29QS-Y3S5-VW7H` | `unknown_card` / stripe | API: 500 |

**Notably, the purchases look refundable.** `GET /purchases/{id}` reports
`status: "complete"` and `items[0].refundableQuantity: 1` for both. Nothing in the
purchase state explains the failure, and it reproduces across payment methods, so
it does not appear to be specific to Naver Pay.

**Questions this raises.**

1. Are refunds supported in the sandbox environment at all, or is this the
   expected behaviour there? The documentation does not say.
2. If they are supported, is there a prerequisite — a settlement delay, a
   merchant setting, a required `fee` field — that a `{}` body misses?
3. Does the same limitation apply to `dispute.opened` / `dispute.closed`
   simulation? Those cannot be triggered from the merchant side at all, so
   verifying that path may need Neon to fire a test event.

**Impact on this integration.** Revocation is implemented and tested, but against
a synthetic event rather than one Neon produced. That distinction is preserved
below rather than glossed over.

---

## What was verified with a synthetic event, and what that does not prove

With no way to obtain a real `refund.processed`, the revocation path was exercised
by constructing the event from the **real purchase data**, signing it with the
configured webhook secret, and delivering it to the live endpoint through the same
tunnel Neon uses.

`externalReferenceId` was deliberately set to `null`, because the documented
example has it null — which makes the `purchaseId` lookup the only route to the
original checkout, and therefore the thing most worth testing.

```
POST /api/webhooks/neon   → 200 {"received":true,"duplicate":false,"revoked":true}
```

Resulting state:

| Check | Result |
|---|---|
| Entitlement on the refunded account | removed — `{}` |
| Purchase record | kept, stamped `refundedAt` and `refundId` |
| Checkout status | `fulfilled` → `refunded` |
| **The other player's entitlement** | **untouched** — account matching held |
| Second refund event, different `event.id` | `200 {"ignored":"checkout is already refunded"}` |
| Late `purchase.completed` for that checkout | `200 {"ignored":"checkout is already refunded"}` |

That last row matters more than it looks: without it, a purchase webhook arriving
after a refund would silently resurrect a revoked entitlement.

---

## Resolution addendum (2026-09-06) — real refunds work with an explicit items body

A fresh run against a Cloud Run-hosted deployment of this service (stable
webhook endpoint, no tunnel) settled the defect report above:

- **`POST /purchases/{id}/refund` with body `{}` still fails** — reproduced
  the same day on a brand-new completed purchase (`status: "complete"`,
  `refundableQuantity: 1`, a different account and payment method than the
  original report): `500 {"code":"UNKNOWN_ERROR","message":"Unhandled error"}`.
  Question 2 above is therefore answered: the failure is specific to the
  empty-body (full-refund) path, and it remains a vendor-side bug worth
  reporting.
- **The item-level body succeeds.** One field-name subtlety: the purchase
  object exposes the item id as `items[].id`, while the refund request wants
  it as `itemId` (a wrong/empty id returns a clean
  `400 PURCHASE_ITEMS_TO_REFUND_NOT_FOUND`, which is how the mapping was
  found):

```
POST /purchases/{purchaseId}/refund
X-API-KEY: <sandbox key>
{"items":[{"itemId":"<purchase.items[].id>","quantity":1}]}
→ 201 { refundId, purchaseId, totalAmount: 490000, … }
```

- **The webhook loop closed in about one second.** Refund created at
  23:21:11.692Z; `refund.processed` was delivered to the live endpoint and
  answered `200` at 23:21:12; the entitlement read back `{}` immediately
  after. Purchase, fulfilment and now refund have all been verified against
  real Neon-produced events — the synthetic-event caveat above is history
  for the refund path, and remains only for disputes.

**What this does not prove.** It does not confirm the shape of the event Neon
actually sends — field names, whether `accountId` is populated, whether
`externalReferenceId` is null in practice as the documentation example suggests, or
what a partial refund looks like on the wire. Our handler is verified against the
documented schema. The schema itself is still taken on trust, and will stay that
way until a refund succeeds.

---

## Still unverified after this run

- A real `refund.processed` event (blocked by the defect above).
- `dispute.opened` / `dispute.closed` — version 1, and carrying only a
  `purchaseId`, so revocation policy for chargebacks remains undesigned.
- Whether `bundleContents` and `taxCode` are accepted on the **checkout request**.
  They appear on the purchase object, which is suggestive but not the same thing;
  they were removed from our request rather than guessed at.
- Behaviour when a player switches country on the hosted page, which is the one
  case where our amount check deliberately does not reject.
