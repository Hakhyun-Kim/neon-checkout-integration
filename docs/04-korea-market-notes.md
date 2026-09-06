# 04 — Korean market notes

These are the things I would put on the table in a first working session before
shipping Neon checkout into Korea — some of them questions for Neon, some of them
work on the integrating side. They are the reason the integration is shaped the way it is.
Everything legal below is flagged as **to verify** — it reflects how I have seen
these questions handled, not legal advice.

## 1. KRW arithmetic is a live bug source

The won has no circulating subunit, but Neon expresses price as 100× the base
unit. ₩4,900 is sent as `490000`. Every Korean engineer's instinct is that the
trailing `00` is a mistake, and a 100×-off price is both easy to introduce and
catastrophic to ship. Mitigation used here: the multiplier is encoded once, in a
frozen table, server-side, and never recomputed anywhere else.

Related: `Intl.NumberFormat('ko-KR', { style: 'currency', currency: 'KRW' })`
yields `₩4,900` with no decimals. Any display path that hardcodes two decimals
will look broken to a Korean player. (Here the string is derived with
`Intl.NumberFormat` in `server/catalog.mjs`, and the test asserts against
`formatPrice()`, so the integer and the display cannot drift — see
[03](03-decisions-and-assumptions.md).)

## 2. Payment rails Korean players actually expect

A checkout that offers only international cards will convert poorly in Korea.
The list that has to be checked off, roughly in order of importance:

- **간편결제** — 카카오페이, 네이버페이, 토스페이, 페이코
- **국내 카드사** — BC, 신한, KB, 삼성, 현대, with 무이자 할부 expectations on
  larger baskets
- **휴대폰 소액결제** (carrier billing) — significant for younger and unbanked
  players, and subject to monthly limits
- **계좌이체 / 가상계좌**
- **상품권** — 컬쳐랜드, 해피머니 — still meaningful in Korean game commerce

**To confirm with Neon:** which of these are live for KR today, which are on the
roadmap, and whether the set differs between Hosted, Embedded, and Direct
checkout. This single answer usually decides whether a Korean launch is viable.

## 3. Minors — the highest-risk item

Under the Korean Civil Act, a contract entered into by a minor without their legal
guardian's consent can generally be cancelled by the guardian, and Korean game
companies see this as a routine chargeback-like flow, not an edge case. A
family-facing title selling directly on the web carries real exposure here.

Practical mitigations to design in from the start:

- an explicit guardian-consent step before a minor's first purchase;
- per-account and per-period spend caps;
- a documented, fast refund path so a dispute never has to escalate;
- keeping the purchase record (who, what, when, how much) auditable — the
  `purchases` ledger in this implementation exists partly for that.

**To confirm with Neon:** how refunds initiated for this reason are processed, and
what the merchant-of-record split of responsibility is.

## 4. Item design carries legal weight

Korea's game legislation requires probability disclosure for 확률형 아이템. This
is a large part of why the items sold here are deterministic cosmetics (a banner
and two castle decorations): they stay entirely outside that regime. Selling gacha through a
web shop needs the disclosure surface designed alongside the checkout, not after.

## 5. Consumer protection and refunds

Korea's e-commerce law provides a withdrawal period (청약철회), with carve-outs for digital
content that has already been supplied or consumed — the boundary matters and is
exactly where disputes land. **To verify:** whether Neon as merchant of record
handles the withdrawal notice and refund obligation, or whether the game must surface its own policy in-game. Also worth confirming: 현금영수증 (cash receipt)
issuance, which Korean buyers expect for cash-equivalent payment methods.

## 6. Data and settlement

- **PIPA / 개인정보보호법** — if payment data is processed outside Korea, the
  cross-border transfer must be disclosed and consented. To verify: what Neon
  discloses on the hosted page and what has to be added in-game.
- **전자금융거래법** — the merchant-of-record model is what makes Neon workable
  for a foreign publisher selling into Korea; it is worth stating plainly up
  front, because it is usually the first legal question raised.
- **VAT** — 10% on digital goods. To confirm: inclusive or exclusive display, and
  who remits.

## 7. Why D2C matters here commercially

Korean publishers have been building their own web shops precisely to move
purchases off the 30% mobile store fee, and Korean players are already used to
buying game currency on a web page and seeing it in-game. The behaviour Neon needs
does not have to be taught in this market — which makes the integration friction,
the payment method list, and the refund story the whole conversation.

## 8. Localization is not translation

The store is bilingual from the first commit: SKU copy, button labels, and status
messages all come from the same localized table, and the game keeps stable save
ids across languages so switching language never forks a player's data. The
`languageLocale` field sent to Neon (`ko-KR` / `en-US`) makes the hosted page
match the game. What is deliberately *not* solved here is that language is being
used as a proxy for country — see [03](03-decisions-and-assumptions.md).
