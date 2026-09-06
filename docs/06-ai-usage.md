# 06 — How AI was used

What matters is not that AI was used, but which decisions were human and how the
output was verified.

## Tools

- **Claude Code** (Anthropic) — this integration: server design review, the Korean
  market analysis, documentation, and an adversarial code review of my own
  implementation.
- **Codex** — the original game, then the independent 2026-09-05 integration review, persistence fixes, documentation update and browser verification.

## Division of labour

**Mine.** The choices that shape the result:

- using a game I had already shipped rather than a starter pack, and accepting the
  constraints that came with it;
- Hosted checkout as the surface, with a server boundary reusable by other clients; Embedded and Direct adapters remain unimplemented;
- deterministic cosmetics as the SKUs, chosen for provable fulfillment and
  to stay outside Korea's probability-disclosure regime;
- the trust boundary — client sends a SKU, never a price; only a verified webhook
  grants;
- treating the redirect/webhook race as the central design problem;
- every judgement in the Korean market notes.

**AI-assisted.** Execution and review:

- drafting the server modules and the client store UI against my design;
- an adversarial review pass over the finished code, which is where the hardening
  list in [03](03-decisions-and-assumptions.md) came from — including the 36-hour
  retry consequence of returning non-2xx, the language-as-country conflation, and
  the `.env.example` that no loader reads;
- cross-checking the implementation against Neon's live documentation, which
  confirmed `x-neon-digest`/HMAC-SHA256, event `version: 2`, the 100× price
  format, and the one-currency-per-country rule, and flagged `taxCode` as
  unverified;
- this documentation set.

## Verification

Nothing here is trusted because a model produced it.

- The integration test (`npm run store:check`) runs against a real HTTP server and
  covers signature rejection, replay, cross-account and cross-amount tampering,
  environment mismatch, and rate limiting.
- The project's full `npm run check` gate — 29 verification scripts plus the
  build — passes with the integration in place.
- The flow was then driven in a real browser. That step earned its keep twice: it
  found a session-losing origin mismatch between `localhost` and `127.0.0.1`, and
  a CSS regression that squeezed the product description to one character wide.
  Neither was visible in the code or in the passing tests.
- Every factual claim about the Neon API was checked against the published
  documentation. What could not be confirmed at the time — `taxCode`,
  `bundleContents` — was removed from the request rather than assumed, and
  turned into a question in [05](05-open-questions-for-neon.md). (The current
  API reference confirms both; the fields are back.)
- The remaining gaps are published in [03](03-decisions-and-assumptions.md) rather
  than quietly fixed, because an honest account of what is unfinished is more
  useful than a clean-looking one.

The instruction-by-instruction record — what was asked, what came back, and
what each instruction changed — is [14 — AI command journal](14-ai-command-journal.md).

