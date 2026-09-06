# 14 — AI command journal

What I asked the AI to do, what came back, and what each instruction changed.
Kept so the division of labour in [06](06-ai-usage.md) can be checked rather
than taken on trust. The early phases are reconstructed from the timeline;
from 4 September the instructions are quoted from the sessions, translated
from Korean.

Tools: Claude Code for the payment integration, the dedicated server, the
reviews and these documents; Codex for the original game and one independent
review pass.

## At a glance

| When | Instruction, in short | Result |
|---|---|---|
| Late Aug | Compare browser game / Unity / Unreal; I decide. Then build the server to this design. | `server/` first pass, green tests |
| Early Sep | Three review passes, kept separate: docs cross-check, attacker read, run it. | 18 defects, four methods |
| Early Sep | Run the tour and screenshot it. Explain on the game screen, not in docs. | 2 sandbox purchases, Firestore, accounts, tour |
| 4 Sep | Re-review the week; tone down completion claims; write a handoff. | 7 persistence and demo fixes |
| 5 Sep, day | Replace the auto tour with a hands-on inspector: real source, real HTTP. | 5-stage inspector, 3 cosmetics, per-item refund |
| 5 Sep, evening | Dedicated server: the server is the logic, clients are viewers under auth. Document why. | Authoritative server, roles, ADR |
| 5 Sep, night | Gateway: clients talk only to the dedicated server, which brokers the store. | Protocol v2, 29 assertions, 2-tab E2E |
| 6 Sep | Deploy for real, test the refund, put it on screen, make a link enough to play. | Cloud Run live, refund resolved, shareable link |

## Entries

### Late August — platform choice and first pass

> "Research where to attach payments: the existing browser game, Unity or
> Unreal. I make the decision." Then: "Build the server modules to this
> design: SKU allowlist, server-owned prices, fulfilment only from a signed
> webhook, the client sends SKU and locale only."

- The AI read Neon's docs, confirmed there is no Unity or Unreal SDK, and laid
  out the comparison. After the decision it drafted `server/`, the store UI
  and an integration test.
- Pattern: decisions mine, execution delegated. The first pass ended with
  green tests, which is exactly what the next phase took apart.

### Early September — adversarial review

> "Three passes, not mixed: cross-check every claim against Neon's docs; read
> the code as an attacker; run it."

- 18 defects across four methods, none overlapping
  ([03](03-decisions-and-assumptions.md)). Eleven were in code the AI had
  written and were found by the same AI re-reading it as an attacker. One test
  asserted the hardcoded price the same pass had introduced.
- Pattern: the same model finds different defects under a different role.

### Early September — after the sandbox invite

> Repeated: "Run the tour and screenshot it." "Explain it on the game screen,
> not in a document." Two changes of direction for the tour. "Public repos
> carry facts; context stays private."

- Two sandbox purchases, Firestore in use, the payment service split out,
  accounts and server saves, the guided tour. Every screenshot round found
  leftover Korean or a mislabelled response that the tests had passed.
- Pattern: the deliverable is the code running on screen, not the code.

### 4 September — weekly review and handoff

> "Re-review this week's integration and fix it. Tone down the completion
> claims. Write a handoff so I can continue on the laptop."

- Seven fixes, among them the static dev server exposing `.env`, a grant
  surviving a failed disk write, and refunds arriving before the purchase
  mapping ([12](12-review.md)). A handoff document.
- Pattern: "verified" and "not yet" in the same sentence became the house
  style.

### 5 September, daytime — interactive inspector

> "The auto tour is small and detached from play. Make it a hands-on demo:
> buy for real, show the actual source excerpts and HTTP evidence, three
> castle cosmetics, a scenario you can lose."

- Five-stage inspector, castle close-up, per-item refund, redacted evidence
  export. All source comments moved to English behind a token-equality guard.

### 5 September, evening — dedicated server

> "Continue from where the other AI stopped; commit and push first."
>
> "Keep Pages as the plain game; move the Neon run instructions elsewhere.
> The machine running this may be a Mac, so cover macOS and check it works."
>
> "Change to a dedicated-server structure. Later I want Unity and Unreal
> sample tests against the same shared server. The server is the real logic;
> clients are viewers under an auth structure. Document why."
>
> "Make it runnable on Windows and macOS immediately. Is Docker more
> convenient? Decide yourself."
>
> "'Running' means the server up, the web client shown, demo mode on
> immediately, with a button to switch the demo off and just play."
>
> "Show the code structure, progress and flow as diagrams on the game screen."

- Commits in order: inspector verification → README split and macOS launchers
  → dedicated server ([13](13-dedicated-server.md)) → this journal.
- The delegated call was Docker. Plain Node launchers became the default; the
  Docker path shipped, marked as not executed.

### 5 September, night — store gateway

> "Why two servers? Is the old one a leftover?" → "Can the plain game stay a
> light client mode, with server mode separate?" → "The point of the
> dedicated server is that it talks to the payment service and clients talk
> only to it. Does that hold? Document it." → "Go ahead: implement the
> gateway and everything it needs, then update the docs and flow charts."

- Two processes are a role split, not a leftover. The README was rewritten
  with client mode as the baseline. The gateway idea was assessed first
  (sound; the payment service stays behind the server, the hosted redirect is
  the one exception), then built: protocol v2, allowlisted store relay,
  brokered identity, a per-connection cookie jar, delivered cosmetics in the
  shared snapshot, webhook paths refused by name. 29 assertions against a real
  in-process payment service, plus a two-tab browser run.
- Pattern: scope was agreed in prose before "implement everything" was
  delegated.

### 5 September, late — clean-up and delivery

> "Clean up the code, no duplication, make the Neon separation visible."
> "Encrypt the keys for hand-off, and prepare a Cloud Run deploy script."
> "One README: run → tech review → history."

- Bot policy promoted to `src/bot.js`; a latent failure in
  `balance:report:check` found and fixed; orphaned tour i18n removed.
  `scripts/secrets.mjs` (scrypt + AES-256-GCM) with its limit stated;
  `deploy/cloud-run.sh` with dry-run and smoke checks.

### 6 September — live, refund, shareable link

> "Deploy it for real, through the smoke checks." "Test the refund and record
> it." "Put the refund in the demo: code, sequence, Owned disappearing." "Let
> the Pages build take the service address as a parameter, so a link is
> enough to play."

- Cloud Run deployment. What the real deploy caught that a dry run could not:
  new-project Cloud Build IAM, the front end intercepting `/healthz`, an exit
  code swallowed by `tail`.
- Refund resolved: the empty-body request still returns 500, the item-level
  body returns 201 and `refund.processed` arrives about a second later
  ([09](09-sandbox-run.md)).
- Pages `?api=` parameter with the CSP as the allowlist, return URLs
  validated. The full lifecycle was then clicked through by someone else.

## Patterns

1. **Direction short, criteria explicit.** "The server is the logic, clients
   are viewers" was the whole design; transport, snapshots versus lockstep and
   the auth shape were proposed by the AI with code evidence, and recorded.
2. **A running demo is the acceptance criterion,** not passing tests.
3. **The person at the other end appears in the instruction:** a Mac, a
   button to turn the demo off, the architecture on screen.
4. **The honesty rule never relaxed.** Unity, Unreal, Docker and physical
   macOS stayed marked as not executed, in the docs and on screen.
5. **Interrupt-and-resume was assumed.** Handoff notes made "continue from
   where the other AI stopped" a one-line instruction.

## Verified and not, as of 6 September

| Verified | Not yet |
|---|---|
| Sandbox purchase and refund end to end through the webhooks (Cloud Run) | Unity / Unreal samples executed (neither engine installed) |
| The shared link, full lifecycle, clicked through by someone else | Docker image executed (no Docker on the build machine) |
| `dedicated:check` 29 assertions and the two-tab gateway run | Physical macOS run (ubuntu CI covers POSIX) |
| Full gate and the 540-run balance baseline unchanged | Player input through the server; dispute events |
