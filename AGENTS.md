# AGENTS.md — orientation for any AI or human landing here

This repository is **documentation only**. It explains a Neon Hosted Checkout
integration; the code it describes lives in a separate repository.

- **Code:** <https://github.com/Hakhyun-Kim/constellation-defense> — the payment
  service is under `server/`, its client under `src/app/neon-store.js` and
  `src/app/neontour.js`. That repo's `AGENTS.md` has the file map and the
  conventions that keep the checkout from breaking.
- **This repo:** design write-up. Start at `README.md`, then `docs/00` →
  `docs/16`; `docs/15` is the Korean, diagram-led walkthrough of the same flow,
  and `docs/16` covers clients that have no browser (Unity, Unreal, launchers).
  `docs/evidence/` has screenshots and a walkthrough; `docs/10` is the
  claims ledger (what each stage proves and does not).

## To run or review the integration

You cannot run anything from this repo — clone the code repo and use its
launcher. On Windows, `start-demo.bat` does everything; by hand, on any platform:

```bash
git clone https://github.com/Hakhyun-Kim/constellation-defense
cd constellation-defense && npm install && npm run build
cp .env.example .env      # mock mode is already on — no credentials needed
npm run serve
```

Then open the checkout inspector: a bot plays the game (`demo=expert`) while the
panel explains the store with source excerpts and live server responses; you make
the purchases and refunds yourself:

```
http://127.0.0.1:8642/?lang=en&demo=expert&tour=neon&mute
```

## If you edit these docs

- Keep them a record of an engineering integration. This repo is public: no
  wording that frames the work as a job application or homework.
- Facts about the Neon API must be checked against Neon's live documentation, not
  assumed. What could not be confirmed is listed as an open question in `docs/05`
  rather than stated — keep that boundary.
- `docs/10` widens only when a new capability is genuinely proven; do not quietly
  broaden an earlier claim.
