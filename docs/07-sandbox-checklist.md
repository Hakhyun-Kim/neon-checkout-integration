# 07 — Sandbox checklist

Everything that has to happen in the Neon Console and on your machine to turn the
mock demo into a real sandbox purchase. Work top to bottom; each step says how you
know it worked.

## A. Before you touch the code

**A1. Accept the Console invitation and confirm which environment you are in.**
Note whether sandbox and production are separate accounts, separate keys, or one
account with a flag. This determines whether `NEON_ENVIRONMENT` can stay `sandbox`
permanently or has to move with the key.

**A2. Provision a sandbox API key.** Copy the secret key. This is the value for
`NEON_API_KEY` and it never leaves the server — it must not appear in
`dist/game.js`, in the repo, or in any screenshot you share.

**A3. Check whether the SKU must exist in the Console.** This integration sends
the item inline (`sku`, `name`, `price`, `quantity`) on `POST /checkout`. Some
platforms additionally require the SKU to be registered in an inventory/offer
catalogue. Look for **Inventory / Offers** in the Console and confirm whether
`CELESTIAL_BANNER` needs to be created there. If it does, create it with the same
SKU string — the webhook matches on that exact value.

**A4. Confirm KR/KRW is enabled for your account.** Neon binds currency to
`playerCountry` and enforces one currency per country. If KRW is not enabled, the
KR path will fail and you will only be able to test US/USD — worth knowing before
you debug the wrong thing.

## B. Make your machine reachable

Neon must reach two things: the browser redirect target and your webhook endpoint.
A `127.0.0.1` address satisfies neither.

> **B0. Alternative — deploy the service once instead of tunnelling.**
> `deploy/cloud-run.sh` in the game repo deploys `server/index.mjs` to Cloud
> Run with the keys in Secret Manager (APIs, Firestore creation, IAM, deploy,
> and `/healthz` · `/readyz` · forged-webhook smoke checks are automated;
> `--dry-run` prints everything first, `--smoke-checkout` creates one real
> sandbox checkout). Then register `<service-url>/api/webhooks/neon` in step C
> once, and any machine can run a purchase by pointing its local client at the
> service (the script prints the two `index.html` edits: the `neon-api-base`
> meta and a CSP `connect-src` entry). Cross-origin clients identify with the
> bearer token, so the B2 cookie caveat does not apply; the billing-region
> picker is cookie-based and stays at geo/default resolution in this setup
> (see doc 12 — the dedicated gateway path in doc 13 is what solves that).
> The webhook target stays stable and no credential leaves the server.

**B1. Start a tunnel.**

```bash
cloudflared tunnel --url http://localhost:8642
```

Copy the `https://…trycloudflare.com` URL it prints. (`ngrok http 8642` works the
same way.)

**B2. Use that URL to open the game, not `localhost`.** This matters more than it
looks: `successUrl` is built from `PUBLIC_URL`, and if the player is on
`localhost` while the redirect lands on the tunnel host, the session cookie does
not follow and the purchase attaches to a different account id. The server warns
in the log when it sees this mismatch — do not ignore that line.

## C. Register the webhook

**C1.** In the Console, add a webhook listener pointing at:

```
https://<your-tunnel>/api/webhooks/neon
```

**C2.** Subscribe to `purchase.completed`, version 2.

**C3.** Copy the listener's shared secret — this is `NEON_WEBHOOK_SECRET`. Without
it every delivery is rejected with `403` and nothing is ever granted.

**C4.** If the Console offers a "send test event" button, use it now. Expected
result: `200` with `{"received":true,"ignored":"unknown checkout reference"}`,
because a synthetic event has no matching checkout in your ledger. Seeing `200`
rather than `500` is the point — a non-2xx would put you in a 36-hour retry loop.

## D. Configure and run

**D1.** Fill in `.env` (copy from `.env.example`):

```ini
NEON_MOCK_CHECKOUT=0
NEON_API_KEY=<sandbox secret key>
NEON_WEBHOOK_SECRET=<listener secret>
NEON_ENVIRONMENT=sandbox
PUBLIC_URL=https://<your-tunnel>
```

**D2.** `npm run serve`, then read the startup lines. The server warns about a
missing key, a missing webhook secret, and a `PUBLIC_URL` Neon cannot reach. A
clean start prints none of those.

**D3.** Open `https://<your-tunnel>` in a browser — not `localhost`.

## E. Buy something

**E1.** Open **별빛 상점**, confirm the price reads ₩4,900 (or $4.99 if your
billing region resolves to US), and press buy.

**E2.** You should land on Neon's hosted page. If you get a `502` from our server
instead, the API rejected the checkout payload — the response body is logged
server-side, and the most likely causes are an unregistered SKU (A3), a disabled
currency (A4), or a field name Neon does not accept.

**E3.** Pay with a sandbox instrument. Ask Neon for the test card numbers and
whether any Korean methods (카카오페이, 토스, carrier billing) have sandbox
equivalents — this is worth knowing regardless, because it is the first thing
anyone shipping a Korean launch needs to answer.

**E4.** Confirm all four of these, in order:

| Where | What you should see |
|---|---|
| Server log | `[store] webhook fulfilled CELESTIAL_BANNER for <account>` |
| `.data/neon-store.json` | the checkout flipped to `status: "fulfilled"`, a `purchases` entry with the real `purchaseId` and `orderNumber` |
| Browser | the store modal switches to **보유 중 / Owned** within ~18s |
| Game HUD | the 🚩 badge next to the champion name |

## F. Negative tests worth running in sandbox

These are the ones that actually prove the integration, and they are cheap once
the happy path works.

**F1. Replay.** Re-send the same delivery from the Console. Expected: `200`,
`.data/neon-store.json` still shows exactly one entry in `purchases`.

**F2. Refund.** Issue a refund from the Console. Expected: `200` with
`{"revoked":true}`, the log line `refund webhook revoked …`, the checkout flipped
to `status: "refunded"`, the purchase record kept but stamped with `refundedAt`,
and the banner gone from the game within a reload.

This is the step most worth watching, because the documented example has
`externalReferenceId: null` — so the lookup that has to work is the one by
`purchaseId`. If revocation silently does nothing, that is where to look.

**F3. Environment guard.** Temporarily set `NEON_ENVIRONMENT=production` and buy
again. Expected: `200` with `{"ignored":"environment mismatch: isSandbox=true"}`
and no grant. Set it back to `sandbox`.

**F4. Bad signature.** Send any body to the webhook endpoint without a valid
`x-neon-digest`. Expected: `403`.

```bash
curl -i -X POST https://<your-tunnel>/api/webhooks/neon -d '{}'
```

## G. Capture evidence

- The server log showing checkout creation and webhook fulfillment.
- The `.data/neon-store.json` ledger after a purchase — **redact the account id**
  and confirm no key appears in it.
- A screenshot of the store before and after (buy button → 보유 중, badge visible).
- A note of anything that differed from these expectations. Discrepancies are the
  questions in [05](05-open-questions-for-neon.md) turning into answers.

## H. Before you push

- [ ] `.env` is not committed (`git check-ignore -v .env` prints a match).
- [ ] `.data/` is not committed.
- [ ] No API key or webhook secret appears anywhere in `git grep`.
- [ ] `npm run check` passes.
- [ ] The tunnel URL is not hardcoded anywhere — it belongs only in `.env`.
