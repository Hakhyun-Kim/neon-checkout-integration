# 08 — Storage and identity: two decisions, and the paths not taken

> Current implementation and verification limits: [12 — Review](12-review.md).

Two questions came up late, and both had answers that looked obvious and were
wrong. This records what was chosen, how it is verified, and — because it is the
more useful half — what was built or considered and then dropped.

---

## Decision 1 — Two storage backends, not one

### What the question actually was

It arrived disguised as hosting. A sandbox purchase needs a public HTTPS webhook
endpoint, which needs somewhere to run, and the moment "somewhere" became a
container the JSON ledger stopped working. That framing made it look like
plumbing: *files do not survive on Cloud Run, so pick a database.*

That framing was wrong, and noticing why is the whole decision.

`JsonRepository` gets its idempotency from a promise queue that serialises writes
**inside one process**. Cloud Run runs several instances. Neon retries a webhook
for up to 36 hours, so redelivery is routine — and two instances receiving the
same redelivered event would both read "not yet processed" and both grant the
item.

So moving to a container does not merely lose a file. **It loses exactly-once.**
The fix is not a database, it is a transaction; the database is just where
transactions are available.

### What was chosen

`FirestoreRepository`, with the entire fulfillment inside `runTransaction`:

```js
return this.db.runTransaction(async (tx) => {
  // Firestore requires every read before any write, so both documents are
  // fetched up front rather than checked as we go.
  const [seen, pendingSnapshot] = await Promise.all([tx.get(eventRef), tx.get(checkoutRef)]);
  if (seen.exists) return { duplicate: true };
  if (!pendingSnapshot.exists) throw new PermanentRejection('unknown checkout reference');
  // …account, sku, quantity, amount — unchanged from the JSON version
  tx.set(playerRef, { entitlements: { [entitlement]: { grantedAt, purchaseId } } }, { merge: true });
  tx.set(eventRef, { purchaseId, at, expiresAt });
  tx.update(checkoutRef, { status: 'fulfilled', purchaseId });
});
```

A `PermanentRejection` thrown inside the transaction aborts the whole thing, so a
rejected webhook cannot leave a half-written grant.

**Both backends stay.** `JsonRepository` is the only thing that makes "clone this
and watch a purchase complete" true without a cloud project, an emulator, or a
credential. Losing that to avoid one file of duplication would be a bad trade. A
factory picks by environment; `store-api.mjs` did not change by a single line,
because the interface it uses is five methods.

Two smaller choices inside it:

- **Rate limiting keeps a per-account document of recent timestamps** rather than
  querying checkouts by account and time. The query version needs a composite
  index, which is one more artifact to deploy and one more thing to fail at
  deploy time. One document read, no index.
- **Expiry is a field, not a loop.** Documents carry `expiresAt` so a Firestore
  TTL policy can retire them. The JSON version needed a manual prune because
  nothing else would do it.

### How it is verified

The same suite runs against both backends. "Same interface" is a claim, and a
claim has to be tested rather than asserted:

```bash
npm run store:check                                    # JSON, always
FIRESTORE_EMULATOR_HOST=127.0.0.1:8787 npm run store:check   # both
```

```
✅ JSON 원장:      catalogue · country · price contract · signature · replay · environment · rate limit · live call path
✅ Firestore 원장: catalogue · country · price contract · signature · replay · environment · rate limit · live call path
```

Getting there required making the tests backend-neutral: every assertion that had
reached into the JSON file (`repository.load()`) now goes through
`pendingCheckout()` and `purchases()`, and the checkout reference is read from
the mock `redirectUrl` — the same value a real client receives.

The emulator needs a JVM (`gcloud components install cloud-firestore-emulator`).
On Windows the env-var prefix does not work in PowerShell; set it first:
`$env:FIRESTORE_EMULATOR_HOST = '127.0.0.1:8787'`.

### What was dropped

| Considered | Why not |
|---|---|
| **Cloud Run + JSON, `min=max=1` instance** | Survives ordinary traffic, loses everything on redeploy or a cold start. Would have made "entitlements are durable" a claim that is true most days. |
| **Firestore only, drop the JSON backend** | Kills the credential-free demo. Anyone reviewing the integration would need a GCP project before seeing a purchase complete. |
| **The existing GCE VM + JSON file** | Genuinely simpler in code — no transaction, no second backend, no dependency. Rejected because the setup cost lands on infrastructure instead (firewall, DNS, TLS, systemd) and produces nothing reusable. It also puts a public listener on a machine running an unrelated service. |
| **Deploying any of it right now** | Deferred, see below. |

The dependency cost is real and worth naming: `@google-cloud/firestore` is this
project's **first runtime dependency**, in a codebase whose entire premise was
zero external assets and no runtime deps. It is loaded through a dynamic import,
so the JSON path and the browser bundle never touch it — but `npm ci` does.

---

## Decision 2 — Identity by token, not only by cookie

### What the question actually was

"Can the game stay on GitHub Pages and only the webhook live on a server?"

No — because checkout creation needs the secret key, so it needs a server too.
But the interesting part is what happens if you *do* split them: the API ends up
on a different origin from the game, and the identity cookie stops working.
`SameSite=Lax` will not travel cross-site, `SameSite=None` is blocked by default
in Safari and Firefox, and the failure mode is one already met in this project —
**a payment that succeeds while the player owns nothing.**

The same wall appears for any non-browser client. Unity and Unreal have no cookie
jar.

### What was chosen

Identity accepts a bearer token first and falls back to the cookie:

```js
function account(req, res, config) {
  const token = bearerToken(req);          // Authorization: Bearer <uuid>
  if (token) return token;
  const current = cookies(req)[PLAYER_COOKIE];
  if (PLAYER_RE.test(current || '')) return current;
  const id = randomUUID();
  appendCookie(res, PLAYER_COOKIE, id, config);
  return id;
}
```

The catalogue response now returns `playerId` so a token client can persist it,
and CORS is an explicit allowlist. `Access-Control-Allow-Credentials` is
deliberately **not** sent: cross-origin traffic uses tokens, precisely so that no
part of the design depends on third-party cookies surviving.

About 40 lines, and the effect is that client type stops constraining server
design:

| Client | Cookie only | With tokens |
|---|---|---|
| Same-origin web | ✅ | ✅ |
| Game on a CDN, API elsewhere | ❌ | ✅ |
| Unity / Unreal | ❌ | ✅ |
| Mobile, launcher | ❌ | ✅ |

**The honest caveat:** cookie and token are both bearer credentials bound to a
device, and neither survives clearing storage. Making the id readable by the
client gives up what `HttpOnly` was buying. That is a real trade, taken because a
web-only identity mechanism is a worse constraint than a readable device id for a
cosmetic purchase. A game with accounts replaces both with its own player id and
`POST /auth/token`.

### How it is verified

Added to the same suite, and running against both backends:

- entitlements resolve from a bearer token with no cookie present;
- a different token sees nothing;
- a malformed token is treated as a new identity rather than accepted;
- preflight from an allowlisted origin returns `204` with `Authorization`
  permitted, and **without** `Access-Control-Allow-Credentials`;
- preflight from any other origin returns `403`;
- requests with no `Origin` header are unaffected.

### What was dropped

| Considered | Why not |
|---|---|
| **`SameSite=None; Secure` cookies** | The direct fix, and it fails by default in Safari and Firefox. Would have reproduced the exact bug this project already hit once. |
| **Cookie only, accept web-only** | Cheapest, and it makes the server unusable for the clients this integration most wants to claim it supports. |
| **Wrapping the web app in Unity/Unreal to hold the key** | Does not work at all, for a different reason — see below. |

---

## The one that was never viable

Worth recording because it is the first question anyone integrating this asks: *could the
client hold the API key if it were a compiled game rather than a web page?*

No. A shipped binary runs on the player's machine. Unity IL2CPP metadata gives up
string constants to a dumper; Mono decompiles to near-source; Unreal `.pak` files
unpack; a webview is still JavaScript, usually with remote debugging attached.
Encrypting the key just moves the problem to the decryption key sitting beside it.
Obfuscation raises cost without changing the category.

The rule is not "keep keys out of JavaScript". It is **keep secrets out of
anything the user runs** — and a stolen payment key is a merchant-account
compromise, not a per-player one.

What a native client *does* change is the checkout surface (Direct rather than
Hosted) and identity (a token in save data, which is the decision above). The
server is unchanged. See [00 — Integration guide](00-integration-guide.md).

---

## Deployment, deferred (then done)

> Update 2026-09-06: carried out after all. `deploy/cloud-run.sh` deploys
> `server/index.mjs` to Cloud Run (Seoul) with Firestore and the keys in Secret
> Manager; purchase and refund webhooks have landed there
> ([09](09-sandbox-run.md)). The reasoning below is why it waited until the
> ledger was concurrency-safe.

Cloud Run deployment was prepared — `Dockerfile`, `HOST` binding, `PUBLIC_URL`
falling back to the request origin so there is no chicken-and-egg with the
deployed URL — and then, at that point, **not carried out**.

The reason is the same one that started this document. Deployment was buying a
public webhook endpoint, and an already-running tunnel buys that too. Everything
else it would have cost — five APIs enabled, a Firestore database whose location
and mode are permanent, an image registry — was infrastructure spend that leaves
nothing behind in the code.

The value moved into the code instead: a ledger that is correct under
concurrency, and a server that does not care what kind of client is calling it.
Those survive whether or not anything is ever deployed. The `Dockerfile` stays,
so deploying later is a command rather than a project.
