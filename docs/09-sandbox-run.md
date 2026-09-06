# 09 — Sandbox run: what it proved, the refund defect, and its resolution

A live run against the Neon sandbox on 2026-09-04, through a public tunnel with
real webhook delivery. Two purchases completed and were fulfilled that day, and
two more on 5–6 September against the Cloud Run deployment (addendum). Refunds failed
server-side that day; that is written up as a defect report below, with the
questions it raised. On 2026-09-06, against the Cloud Run deployment, an
item-level refund succeeded end to end (resolution addendum at the end).

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

카카오페이, 네이버페이 and 삼성페이 are live for KR. This is the first question for a
Korean launch, and the answer is good.

One onboarding observation from the same page: Google Pay is preselected on the
KR page and opens the real wallet sheet with the tester's real cards, so a first
sandbox purchase looks like a live charge. The page's test-environment label and
`isSandbox: true` on every webhook are the reassurance; Kakao Pay and Naver Pay
give a lighter test approval. Worth one line in any note handed to QA before a
Korean test pass.

**Tax is inclusive, 10%.** The checkout page showed `세금에 ₩445 포함`, and
`GET /purchases/{id}` confirms `taxAmount: 44500`, `taxRate: 0.1`,
`totalAmount: 490000`. The displayed price is what the player pays.

**Settlement is in USD, with roughly a 10% fee.**

```json
"fee": { "fxRate": 0.0007426716, "feeAmount": 37,
         "netProceedsAmount": 294, "settlementCurrency": "USD" }
```

₩4,900 settles as $2.94 net after a $0.37 fee. Worth knowing before pricing in
KRW.

**The checkout id is called `id`.** *Corrected 2026-09-06.* This section used
to say `POST /checkout` returns only `{ token, redirectUrl }`. A direct sandbox
call on 2026-09-06 returned `{ id, token, redirectUrl, externalProvider }`, as
the reference documents; our adapter was reading `checkoutId`, so it recorded
`undefined`, which a JSON store drops silently and Firestore rejects outright.
The purchase object exposes the same value as `checkoutId` (`61f11705-…`). The
adapter now reads `id` first.

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
webhook endpoint, no tunnel). Two more purchases completed there, both fulfilled
by the signed webhook (purchase 3 through the local client pointed at the
service, purchase 4 through the shared Pages link, clicked by hand):

| | Purchase 3 (2026-09-05) | Purchase 4 (2026-09-06) |
|---|---|---|
| `purchaseId` | `1dddcc05-a630-49e2-acd8-d9d0142c08e3` | in the Cloud Run log, not copied here |
| `purchase.completed` answered 200 | 15:26:41Z | 23:48:11Z |
| Refund | item-level refund the next day, `refund.processed` 23:21:12Z (below) | `POST /api/store/refund` 202 at 23:48:39Z, `refund.processed` 23:48:42Z |

The same run settled the defect report above:

- **`POST /purchases/{id}/refund` with body `{}` still fails** — reproduced
  the same day on a brand-new completed purchase (`status: "complete"`,
  `refundableQuantity: 1`, a different account and payment method than the
  original report): `500 {"code":"UNKNOWN_ERROR","message":"Unhandled error"}`.
  Question 2 above is therefore answered: the failure is specific to the
  total-refund path, and it remains a vendor-side bug worth reporting — see
  the 2026-09-07 re-verification below, which pins down which half of the
  endpoint is at fault.
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

**What this does not prove (as of 2026-09-04).** It did not confirm the shape of
the event Neon actually sends — field names, whether `accountId` is populated,
whether `externalReferenceId` is null in practice as the documentation example
suggests, or what a partial refund looks like on the wire. Our handler was
verified against the documented schema until the 2026-09-06 run above delivered
a real `refund.processed` that the handler accepted and acted on; a partial
refund on the wire is still unseen.

---

### Re-verified 2026-09-07 — the total-refund path, not the empty body

Before submission the claim was re-tested on a purchase completed that day
(`04eecc95-5a3a-4dda-a467-e2942228e866`, KRW 4,900, Kakao Pay through Stripe
test), deliberately looking for a way in which the fault would be ours:

| Request body | Result |
|---|---|
| `{}` | `500 UNKNOWN_ERROR` |
| `{"fee": 0}` — the documented total-refund shape | `500 UNKNOWN_ERROR` |
| `{"items": []}` | `500 UNKNOWN_ERROR` |
| `{"fee": "x"}` | `400 INVALID_REQUEST` · `fee must be number`, `items must have required property` |
| no body at all | `415 INVALID_REQUEST` · `unsupported media type undefined` |
| `{"items":[{"itemId": "<items[].id>", "quantity": 1}]}` | `201`, `refund.processed` delivered, entitlement revoked |

The `400` and `415` rows are the point. The endpoint validates request bodies
and names both branches of its own `anyOf` when one is malformed, so `{}` and
`{"fee": 0}` pass validation and then fail inside the handler. The [refund
guide](https://docs.neonpay.com/docs/refunds) describes a total refund as
taking an optional `fee` that defaults to zero, which makes both of those
bodies the documented way to request one. So the narrower reading — "the empty
body was malformed" — does not hold: what fails is the **total-refund path**,
on its documented request, while partial refunds on the same purchase succeed.
A non-zero `fee` was not tried; it would have consumed the only refundable
purchase.

## Still unverified after this run

- ~~A real `refund.processed` event (blocked by the defect above).~~ Resolved
  2026-09-06 (addendum): a real event arrived and revoked; the empty-body refund
  path still returns 500.
- `dispute.opened` / `dispute.closed` — version 1, and carrying only a
  `purchaseId`, so revocation policy for chargebacks remains undesigned.
- Whether `bundleContents` and `taxCode` are accepted on the **checkout request**.
  They appear on the purchase object, which is suggestive but not the same thing;
  they were removed from our request rather than guessed at.
- Behaviour when a player switches country on the hosted page, which is the one
  case where our amount check deliberately does not reject.
