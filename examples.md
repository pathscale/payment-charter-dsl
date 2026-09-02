# Payment Charter DSL — examples

Date 2026-09-02. Worked examples against [`spec.md`](spec.md). Every accepted
example compiles; every rejected one names the rule and error code it violates.

These double as the seed of the conformance corpus (§11), so they are written to be lifted
into `conformance/` rather than paraphrased.

---

## 1 · The smallest useful charter

```
charter solo version 1
resolver common@41
timezone UTC+00:00

  asset USDC_circle = mint://USDC/Circle/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp

  limit spend
    amount 100.00 USDC_circle
    per fixed day
```

Static ceiling: 100.00 USDC_circle per calendar day, UTC+00:00. No scope, so one accumulator for
everything. No escalation, so exhaustion denies.

---

## 2 · The household example

*"Food, $200 a week. Video games on my wishlist that are on discount, unlimited."*

The first half is direct. The second half is the interesting one, and **it cannot be written as
stated.**

### 2a · The half that works

```
charter household version 4
resolver common@41
timezone UTC-05:00

  asset USDC_circle = mint://USDC/Circle/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp

  group groceries = { mcc:5411, mcc:5422, mcc:5451, mcc:5462 }

  limit food
    amount 200.00 USDC_circle
    for merchant.category in groceries
    per fixed week
    scope account
```

**The `for` clause is doing the work, and an earlier draft of this example omitted it.** Without
it the limit reads "$200 a week" and *means* "$200 a week on everything" — every limit applies to
every request unless it says otherwise. The charter would have been a correct document expressing
a policy nobody wrote.

Note what S24 then does to this charter: with `food` the only limit, a payment to a non-grocery
merchant matches no limit and is **denied**. That is the intended reading of "food, $200 a week"
by someone who has written down only that one rule, and it is the safe one.

### 2b · The half that must be refused

```
  limit games
    amount unlimited when merchant.category is video_games and offer.discounted    # REJECTED
```

Two independent refusals, and the second is the one that matters.

**`unlimited` in a root charter — E314.** Under H4 it means "this level adds no constraint",
which is only finite when a parent supplies one. `household` extends nothing, so the ceiling is
unbounded and S5 fails. In a *child* charter the same word is legal and means "defer to my
parent's ceiling."

**`discounted` is not in the vocabulary, and adding it would be the bug.** A discount is an
offered price compared against a reference price, and **the reference comes from the merchant.**
So the predicate granting unlimited spend would be under the control of the party being paid.
Inflate the reference, declare a discount, and the charter authorises without bound.

This is the exact shape §7 S2 exists to prevent, arriving from a direction that looks harmless.
The provenance model already names it: a discount claim is `Plane::Merchant` at best, and
`provenance is at least merchant` is the tier a charter should be *tightening* on, never
unlocking on.

### 2c · What the author actually meant, expressed safely

The real intent is not "trust the merchant's discount". It is **"I will pay up to this much for
this item."** That is a price the principal states, so it is `Plane::Principal`, and it is a
literal, so S1 and S5 hold.

```
  prices wishlist = {
    item:hollow-knight-silksong at 45.00 USDC_circle,
    item:factorio-space-age     at 35.00 USDC_circle,
    item:outer-wilds            at 25.00 USDC_circle,
  }

  limit wishlist_games
    amount from wishlist
    per fixed month
    scope item
```

`amount from wishlist` takes each item's ceiling from the table. `scope item` accumulates per
item, so buying Silksong does not consume Factorio's allowance.

And the behaviour is what the author wanted: a game at or below their stated price goes through
without asking; above it, refused. **A merchant cannot manufacture the condition, because the
condition is a number the principal wrote down.** A sale is precisely when the offer falls under
that number.

**The static ceiling is 105.00, not 45.00.** An earlier draft of this section said 45.00 — the
largest entry — and that was wrong in a way worth keeping visible, because it is the mistake
this whole language is built to make impossible.

Under `scope item` each item has its own accumulator. Three items with their own ceilings, all
open at once, is `45 + 35 + 25` of window exposure. The largest entry is the ceiling on any one
*item*; it is not the ceiling on the *charter*, and S5 asks for the second. Read the maximum and
you under-state what the document authorises by a factor of the table's length.

There is a second, sharper reason the two halves cannot be separated. `scope item` keys an
accumulator on an identifier the merchant supplies. On its own, a merchant that varies the item
id draws a fresh allowance every time and the aggregate is unbounded — the §2b failure again,
one level down: the merchant can no longer manufacture the *condition*, but can manufacture an
endless supply of fresh *accumulators*. What closes it is the table, which enumerates every id
that can resolve to money at all. So `scope item` is only ever legal alongside
`amount from <prices>`, and an item absent from the table is refused rather than unbounded.

> **`prices` and `scope item` are deferred**, not adopted — see `spec.md` §13.1, which records
> the corrected design. The motivation holds and the shape is nearly right, but a proposal whose
> published bound was off by a factor of three is not finished, and `scope item` additionally
> needs an item identity in §6's field table, which S2 makes a reviewed engine change rather
> than an author's to make. Syntax added can only be removed by breaking documents already
> written; syntax deferred costs a version number.

---

## 2A · Petty cash, and the boundary that is easy to get wrong

*"The agent can spend $100 this month without asking me. Over that, or anything $50 or more,
needs my approval."*

```
charter assistant version 1
resolver common@41
timezone UTC-05:00

  asset USDC_circle = mint://USDC/Circle/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp

  approvers owner = { sam }

  limit petty_cash
    amount 100.00 USDC_circle
    per fixed month in UTC-05:00
    scope agent
    escalate at least 50.00 USDC_circle require 1 of owner up to 2000.00 USDC_circle within 3 days
    escalate when exhausted require 1 of owner up to 2000.00 USDC_circle within 3 days
```

Read it back against the sentence:

| Request | Outcome |
|---|---|
| 30.00, 20.00 spent this month | **allow** — under 50, under the monthly cap |
| 49.99 | **allow** |
| **50.00** | **escalate** — this is the whole point, see below |
| 60.00 | **escalate** — at or above 50 |
| 10.00, with 95.00 already spent | **escalate** — the month is exhausted |
| 10.00, with 95.00 spent, no approver answers within 3 days | **deny**, and the reservation is released (§8.1.5) |

**`at least`, not `above`.** "Anything fifty dollars or more" includes fifty dollars.
`escalate above 50.00 USDC_circle` fires on `> 50.00`, so a payment of exactly 50.00 goes through
unattended — silently, with nothing malformed and the charter compiling cleanly. It is also the
single most likely amount for a rule about fifty dollars to actually meet. The language offers
both forms and neither as a default, because the author has to state which edge they mean;
[`spec.md`](spec.md) §8.2.3 is the rule.

**Both escalations are needed, and they are different questions.** `at least 50.00` asks about
*this payment's size*. `when exhausted` asks about *the month's remaining allowance*. A $10
payment in a month with $5 left trips the second and not the first. Declaring only the first
would let the agent quietly drain the last of the budget in small pieces; declaring only the
second would let a single $80 payment through unasked. S15 permits one of each.

**`up to 2000.00` is not decoration.** S5 requires every escalation to name a finite ceiling, so
the charter states two numbers a reader can find without running anything: **100.00** is the
most the agent moves alone in a month, and **2000.00** is the most it moves with Sam answering
each time. There is no third number and no way to write one.

**What happens when Sam lowers it mid-month.** Suppose 80.00 is spent and Sam decides 100 was
too generous, editing to 50.00. Everything denies until the month rolls — the meter reads 80.00
against a ceiling of 50.00. It does **not** restart the month, which would let the agent spend
130.00 in a month Sam capped first at 100 and then at 50. [`spec.md`](spec.md) §8.4A: an edit
changes the ceiling, never the meter.

### 2A.1 · The same thing at company scale

A CFO wants central control of petty cash across departments and staff. That is the hierarchy in
§3, and nothing new is needed:

```
COMPANY_WIDE        CFO writes this. Company-wide prohibitions and the total cap.
  └── DEPT_ENG      Department head. May only tighten.
        └── MANAGER_ALICE
              └── AGENT_BUILDBOT
```

Four properties fall out, and each is a rule already stated rather than a feature added for this:

- **A department cannot exceed the company.** H1 takes the minimum over the chain per request, so
  a department writing 40000 under a company cap of 250000 faces 40000, and one writing 400000
  faces 250000 with its own number dead text (W2).
- **Every payment draws down every level.** H2: a leaf agent's $30 debits the leaf, the manager,
  the department and the company. There is no accounting in which the department's spending is
  invisible to the CFO.
- **The CFO's prohibitions are absolute.** `prohibit sanctions when merchant.country in sanctioned` at
  the root holds for every agent beneath it, and no department exception and no local quorum
  lifts it (H6). A department may add prohibitions of its own; it cannot narrow one it inherited.
- **A tightening propagates immediately, mid-window.** The CFO lowering the company cap on the
  14th binds every department at once, and hands nobody a fresh allowance (§8.4A.3). This is
  what makes "central management of spending" mean anything: if a mid-window edit reset the
  meters, tightening the company cap would *increase* what could be spent that month, and the
  control would be worse than useless.

The CFO's document names departments' ceilings and the company's prohibitions. It does not have
to enumerate employees, and it cannot be worked around by one.

---

## 2B · One asset, several chains

USDC is not one balance. Circle issues it on Solana, on Ethereum and elsewhere; the deployments
are the same asset — same issuer, same redemption, 1:1, same decimals — held in different
wallets. Moving between them costs a fee and takes time.

Declared as two assets with two limits, "a hundred a month" quietly becomes two hundred:

```
  asset USDC_solana = mint://USDC/Circle/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp
  asset USDC_ethereum = mint://USDC/Circle/0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48/eip155:1

  limit sol_monthly
    amount 100.00 USDC_solana
    per fixed month

  limit eth_monthly
    amount 100.00 USDC_ethereum
    per fixed month
```

Each limit's static ceiling is 100.00 and the charter's is **200.00**. Nothing is wrong per
rule; the document simply does not say what its author meant. A compiler SHOULD warn here (W4):
two declarations the resolver reports as the same asset, capped separately.

### 2B.1 · What the author meant

```
  asset USDC_ethereum = mint://USDC/Circle/0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48/eip155:1
  asset USDC_solana = mint://USDC/Circle/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp

  asset group USDC_circle_group = { USDC_ethereum, USDC_solana }

  limit monthly
    amount 100.00 USDC_circle
    per fixed month
```

One accumulator across both chains. Static ceiling **100.00**, wherever it settles.

### 2B.2 · Why the members keep their own names

Because a cap can be correct in aggregate and badly placed in particular. Bridging back from
Ethereum costs real money, so a controller may well want a hundred in total and not much of it
stranded on the expensive chain. That needs no new construct — it is another limit, and §8.3
already takes the most restrictive:

```
  limit monthly
    amount 100.00 USDC_circle
    per fixed month

  limit ethereum_exposure
    amount 25.00 USDC_ethereum
    per fixed month
```

A Solana payment draws `monthly` only. An Ethereum payment draws both, and is refused past 25.00
even with 90.00 of the group's allowance untouched (S23). Prohibiting a chain outright is
ordinary too: `prohibit no_mainnet when asset is USDC_ethereum`.

An opaque set — one alias secretly binding two references — would have made the aggregate
expressible and every one of these impossible.

### 2B.3 · The ceiling is authority, not liquidity

Say the wallets hold 60.00 on Solana and 40.00 on Ethereum, under the `monthly` limit above.

| Request | What happens |
|---|---|
| 50.00 | Settles on Solana. One payment, 50.00 off the monthly ceiling. The charter does not choose the chain and does not care which fits. |
| 70.00 | **Fits nowhere.** Neither wallet holds 70.00, and the charter cannot conjure it. |

The second row is the one that matters. The charter authorises 70.00 — there is 100.00 of
allowance and the group spans both chains — and the payment still cannot be made in one go.
Consolidating first is a *second payment*: it draws the same allowance, it pays a bridge fee
that also draws it, it takes time, and under §8.1.4 its reservation is held until the source
chain proves the transfer cannot land.

None of that is the charter's problem to solve, and it is important that it does not try. A
limit bounds what may leave. Reading it as a spendable balance is how a bound turns into a
guess, and the engine would have to consult wallet state that S2 keeps out of conditions for
good reason.

**One request settles once.** An executor MAY NOT pay 70.00 as 40.00 from one chain and 30.00
from the other and report one payment. Those are two requests, separately reserved and
separately counted — otherwise a `count` limit is counting something other than transactions.

---

## 2C · Cards, tokens and wallets

An **instrument** is where money comes from: a Visa virtual card, a Mastercard network token, a
Solana wallet. Limits shape around instruments and categories at least as often as around
assets, which is what the `for` clause is for.

```
charter corporate-cards version 1
resolver full@41
timezone UTC-05:00

  asset USD_iso4217 = unit://USD/ISO4217

  instrument mastercard_token = card://mastercard/tok_d4e5f6
  instrument visa_gold = card://visa/tok_g7h8i9
  instrument visa_virtual = card://visa/tok_a1b2c3

  approvers finance = { cfo, controller }

  limit visa_monthly
    amount 1000.00 USD_iso4217
    for instrument is visa_virtual
    per fixed month

  limit mastercard_monthly
    amount 50.00 USD_iso4217
    for instrument is mastercard_token
    per fixed month

  limit gold_monthly
    amount 1000000.00 USD_iso4217
    for instrument is visa_gold
    per fixed month
    escalate at least 5000.00 USD_iso4217 require 2 of finance up to 1000000.00 USD_iso4217 within 5 days
```

### 2C.1 · An instrument is an identity, not a ceiling

`instrument visa_gold = card://visa/tok_g7h8i9` binds a name and nothing else. Every cap is an
ordinary limit (S25), so instrument caps get windows, scopes, exceptions and escalations for
free instead of growing a parallel construct that would eventually need all four anyway.

**The handle is not a credential** (S26). `tok_g7h8i9` is a network token id — an opaque
reference that identifies which card without being able to charge it. A charter is the one
document in this system guaranteed to be copied: reviewed, diffed, signed, handed to an auditor.
A PAN in it is a conformance failure, and a compiler SHOULD reject a handle that looks like one.

### 2C.2 · Omission denies, which is what makes this safe

A fourth card, added to the wallet and not to the charter, cannot be spent from. No limit
applies to a request naming it, so S24 denies (E219).

That is the whole reason instruments carry no ceiling of their own. The alternative design put
an `up to` on the declaration so a forgotten limit could not mean unbounded spending — but a
declaration that is simply absent already means denied, and one mechanism is better than two.

### 2C.3 · Go buy me a car

`visa_gold` authorises a million dollars, and the language is entirely comfortable with that
because the number is a literal in the source. S5 reads it without running anything; the charter
says a million and means a million.

What makes it a policy rather than a hole is the escalation. Anything at or above `5000.00`
takes two of finance, so the autonomous ceiling on that card is 5000.00 and the accompanied one
is 1000000.00. Two numbers, both in the file, and the second is only reachable with two humans
approving the exact payment digest (§8.5).

The car is fine. The car is fine *because* somebody has to say so.

### 2C.4 · What the whole charter authorises

Per S5.1, the static maximum over a month is the **sum** of the limits over `USD_iso4217`:

```
  visa_monthly          1000.00
  mastercard_monthly       50.00
  gold_monthly        1000000.00
                    ------------
                     1001050.00
```

Not 1000000.00, and not any of the three numbers written. This is the figure a compiler emits
and the one that answers "how much can this thing spend", and it is exactly the arithmetic that
turns two hundred-a-month caps on equivalent assets into two hundred a month (§2B). Three
sensible per-card limits still add up, and a controller is owed the total rather than left to
compute it.

---

## 3 · Organizational depth

```
COMPANY_WIDE
  └── DEPT_ENG
        └── MANAGER_ALICE
              └── AGENT_BUILDBOT
```

Four documents, one chain. Each extends exactly one parent (E312 forbids two).

### 3a · Root

```
charter company-wide version 7
resolver full@41
timezone UTC+00:00

  asset USDC_circle = mint://USDC/Circle/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp

  group sanctioned    = { country:PRK, country:IRN }
  approvers treasury  = { cfo, controller, deputy }

  prohibit sanctions when merchant.country in sanctioned

  limit company_monthly
    amount 250000.00 USDC_circle
    per fixed month
    escalate when exhausted require 2 of treasury up to 400000.00 USDC_circle within 3 days

  limit any_single_payment
    amount 50000.00 USDC_circle
    per rolling 10 seconds
    escalate above 25000.00 USDC_circle require 2 of treasury up to 50000.00 USDC_circle within 2 days
```

Static maximum: **400000.00** on the escalated path, 250000.00 autonomous. Both literals.

### 3b · Department

```
charter dept-eng version 3
extends company-wide@7
resolver full@41
timezone UTC+00:00

  asset USDC_circle = mint://USDC/Circle/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp

  group cloud = { mcc:7372, mcc:4816 }
  approvers eng_leads = { alice, bob }

  limit dept_monthly
    amount 40000.00 USDC_circle
      except 60000.00 USDC_circle when merchant.category in cloud
    per fixed month
    escalate when exhausted require 1 of eng_leads up to 60000.00 USDC_circle within 1 days
```

The cloud exception raises the *department's own* ceiling to 60000. It does not touch the
company's 250000 — H1 takes the minimum, so a cloud payment faces min(60000, 250000).

### 3c · Manager

```
charter manager-alice version 2
extends dept-eng@3
resolver full@41
timezone UTC+00:00

  asset USDC_circle = mint://USDC/Circle/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp

  limit team_monthly
    amount 8000.00 USDC_circle
    per fixed month
    scope agent

  limit team_daily
    amount 1500.00 USDC_circle
    per fixed day
    scope agent
```

`scope agent` means per agent under Alice, not 8000 shared. H2 still applies: every one of those
agents' payments also draws down `dept_monthly` and `company_monthly`.

### 3d · Leaf

```
charter agent-buildbot version 11
extends manager-alice@2
resolver full@41
timezone UTC+00:00

  asset USDC_circle = mint://USDC/Circle/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp

  group ci_vendors = { 9WzDXwBbmkg8ZTbNMqUxvQRAyrZzDsGYdLVL9zYtAWWM, 3n1LSbDqQBTfLQ6RCLDDGvVBqLTKZUnJDkAcHZzWyHAr }

  prohibit merchant_supplied when provenance is at least merchant

  limit burst
    amount 200.00 USDC_circle
    per rolling 5 minutes

  limit routine
    amount 50.00 USDC_circle
      except unlimited when counterparty in ci_vendors
    per fixed day
```

**`ci_vendors` is a set of addresses, not a set of MCCs, and an earlier draft had it the other
way round.** As `{ mcc:7372 }` this clause read "if the merchant is categorised as a CI vendor,
remove the cap" — and an MCC is assigned by the merchant's acquirer, not stated by the
principal. That is §2b arriving in a quieter costume: the predicate that lifts the bound sits
under the control of someone other than the person the bound protects. W5 warns on exactly this
shape.

As a set of addresses the controller wrote down, the same clause is principal-stated and safe,
which is the same repair §2c made for the wishlist. **Merchant-derived data may select which
bounded limit applies; it must not remove a bound.** A category-shaped budget is written as its
own limit with `for merchant.category in …`, where every branch still lands on a literal.

Both interesting lines:

- **`unlimited` is legal here** (H4) because this charter extends one. It resolves to the
  parent's effective ceiling — min(1500 daily from Alice, 40000/60000 from the department,
  250000 from the company). Finite, so S5 holds by induction.
- **`prohibit merchant_supplied when provenance is at least merchant`** refuses anything where
  any field came from the counterparty or worse. A merchant-supplied amount is exactly the
  injection this system exists to stop, and `is at least` keeps holding if a worse plane is ever
  added — an enumerated `in { merchant, network }` would not (W1).

  Note what moving this out of `routine` bought. As an exception it constrained one limit, so
  the same danger had to be restated in every limit the leaf declared, and a limit added later
  would silently not have it. As a prohibition it is one line covering the whole document and
  everything beneath it (H6). It also composes with `except unlimited when counterparty in
  ci_vendors` without S4 complaining, where two exception clauses that overlap on a CI vendor
  with merchant-stated provenance would have been E304.

### 3e · What a request actually faces

BuildBot requests **900.00 USDC_circle** to a CI vendor, all fields principal-stated.

| Level | Limit | Resolved ceiling | Reserved | Verdict |
|---|---|---|---|---|
| leaf | `burst` | 200.00 / 5 min | 0 | **deny** — 900 > 200 |
| leaf | `routine` | unlimited → parent | 640.00 | allow |
| manager | `team_daily` | 1500.00 | 640.00 | allow |
| manager | `team_monthly` | 8000.00 | 5100.00 | allow |
| dept | `dept_monthly` | 60000.00 (cloud) | 38400.00 | allow |
| company | `company_monthly` | 250000.00 | 96000.00 | allow |

**Denied by `burst`.** §8.3 joins with `deny` on top, and the denial reports every limit that
refused — here, one. The agent's remedy is four payments of 200 across twenty minutes, which is
what a burst limit is for.

Change the request to **1400.00** and `burst` still denies. Change it to **150.00** and every
level allows.

Now suppose `team_daily` is exhausted at 1500 and BuildBot asks for 10.00:

- `team_daily` is exhausted, and Alice's charter declares **no** `when exhausted` escalation →
  that limit **denies** (§8.4).
- Had Alice declared one, the quorum would be Alice's own approvers, because *her* level imposed
  the binding constraint.
- If instead `company_monthly` were the exhausted one, **H3 requires `treasury`.** Alice cannot
  convene her own reports to spend past a company limit — and neither can the department's
  `eng_leads`.

---

## 4 · Provenance in practice

```
  prohibit merchant_recipient when provenance.recipient is at least merchant

  limit merchant_quoted
    amount 25.00 USDC_circle
      except 500.00 USDC_circle when provenance.amount is principal
    per rolling 24 hours
    scope counterparty
```

Three tiers, and the third is a different construct on purpose. A price the human typed: 500. A
price the agent inferred or the merchant quoted: 25. A *recipient address* supplied by the
merchant: refused outright, regardless of amount — because a wrong recipient is unrecoverable in
a way a wrong amount is not.

That last one is a prohibition rather than a ceiling of zero, and §8.2.1 is why. A ceiling of
zero is exhaustion, and exhaustion escalates wherever a `when exhausted` clause exists — so
expressed as a ceiling, "never pay an address the merchant chose" would show up in an approver's
queue at 2am as an ordinary over-limit request awaiting a quorum. It is not one. It is the
attack this limit was written to stop, and the correct behaviour is that no human is offered the
chance to wave it through.

Note the dotted fields. Bare `provenance` is the tier, the maximum over all four, so
`provenance is principal` would demand every field be principal-stated and reject a perfectly
ordinary invoice where only the venue came from the network.

---

## 5 · Rehearsal on devnet

Same charter, one segment different:

```
  asset USDC_circle = mint://USDC/Circle/4zMMC9srt5Ri5X14GAgXhaHii3GnPAEERYPJgZJDncDU/solana:EtWTRABZaYq6iMfeYKouRu166VU2xqa1
```

Every limit, escalation, quorum and window exercised for real against a network that
**structurally cannot be confused for production** — the genesis identity differs, so a devnet
charter fails S7 on mainnet and vice versa. Copying this file to production is a compile error,
not a very expensive afternoon.

---

## 6 · Refusal gallery

Each of these is a conformance case in `parse/reject/`.

**E307 — mixed assets in one limit**
```
  limit mixed
    amount 500.00 USDC_circle
      except 400.00 EURC when counterparty is 7xKX…
    per fixed day
```
A cap of "500 USDC_circle or 400 EURC" is two limits pretending to be one, and its ceiling is not a
quantity of anything.

**E304 — overlapping exceptions**
```
    amount 100.00 USDC_circle
      except 500.00 USDC_circle when counterparty in suppliers
      except 900.00 USDC_circle when counterparty is 7xKX…      # a member of suppliers
```
Both hold for that counterparty. S4 rejects and names both rules. Under a first-match language
this pays 500 or 900 depending on parser internals.

**E306 — escalation below base**
```
    amount 500.00 USDC_circle
    escalate above 200.00 USDC_circle require 2 of finance up to 300.00 USDC_circle
```
An escalated ceiling under the autonomous one means approval *lowers* the limit.

**E402 — unknown mint**
```
  asset SCAM = mint://USDC/Circle/6666666666666666666666666666666666666666666/solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp
```
Fails closed. "Never seen" and "fine" must not produce the same outcome.

**E401 — segment disagreement**
```
  asset USDC = mint://USDC/Tether/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4Us…
```
The mint is real and the issuer is wrong. The error names the segment. This is the check that
makes the redundancy worth its length — a bare address could only be wrong silently.

**E313 — escalating past a higher level**
```
charter dept-eng version 3
extends company-wide@7
  limit dept_monthly
    amount 40000.00 USDC_circle
    per fixed month
    escalate when exhausted require 1 of eng_leads up to 300000.00 USDC_circle
```
300000 exceeds the company's static maximum, so this asks a department lead to approve past a
company constraint. H3 refuses.

**E311 — no limits**
```
charter empty version 1
resolver common@41
timezone UTC
```
Permitting everything must not be expressible by omission.

**E315 — asset permitted nowhere in the chain**

An intent naming an asset with no cap at any level is denied at evaluation. There is no
inheritance of permission by silence.

---

## 7 · Evaluation vectors

For `conformance/eval/`. Each is a charter plus a request sequence plus expected decisions.

1. **The 101st unit.** 100.00 allowance, `escalate when exhausted`. One hundred 1.00 requests
   allow; the 101st **escalates**, and does not fail. Without the escalation clause it denies.
2. **Concurrency.** Same allowance, one hundred 1.00 requests in flight and none settled. The
   101st is refused — reservation, not settlement (§8.1.2).
3. **Release needs proof of death.** A reservation is held through a 90-second timeout and
   released only on blockhash expiry. Releasing on the timer and then having the transaction
   land is the double-spend this vector exists to catch.
4. **Count never returns.** 20 attempts all failing consume all 20. The 21st denies.
5. **Window attribution.** Reserved 23:59:59, settles 00:00:30. Charged to the earlier day.
6. **DST, both directions.** A 23-hour and a 25-hour local day each grant one allowance.
7. **Conflict at runtime.** Two exceptions that could not be separated statically both hold.
   The limit denies and the charter is flagged broken — it does not pay the first match.
8. **Hierarchy minimum.** Child declares 5000 under a parent of 500. Effective ceiling is 500,
   and the child's number is dead text (W2).
9. **`unlimited` is bounded.** Leaf declares `unlimited`; the effective ceiling equals the
   parent's, and a request above it is refused.
10. **Escalation authority.** A company-level exhaustion is not satisfiable by department
    approvers (E313), and is satisfiable by treasury.

## 8 · The canonical text form

§1.2 of the specification defines exactly one canonical rendering of a charter, and
`roundtrip/` conformance is a byte comparison against it. The example in §4 of the spec is
formatted for reading — aligned `=`, declarations in the author's order. This is the same
charter as an emitter MUST produce it.

```
charter acme-treasury version 7
resolver common@41
timezone UTC+00:00

  asset USDC_circle = mint://USDC/Circle/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp

  group hardware = { mcc:5045, mcc:5732 }
  group trusted_suppliers = { 7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU }

  approvers finance = { alice, bob, carol }

  prohibit holiday_freeze when date after 2026-12-20 and date before 2027-01-02

  limit burst
    amount 100.00 USDC_circle
    per rolling 5 minutes
    scope agent

  limit daily_spend
    amount 500.00 USDC_circle
      except 5000.00 USDC_circle when counterparty in trusted_suppliers
    per fixed day in UTC+00:00
    scope agent
    escalate above 200.00 USDC_circle require 2 of finance up to 5000.00 USDC_circle within 1 days
    escalate when exhausted require 2 of finance up to 5000.00 USDC_circle within 1 days

  limit transaction_count
    count 20
    per fixed day in UTC+00:00
    scope agent

  limit untrusted_counterparty
    amount 50.00 USDC_circle
      except 0.00 USDC_circle when provenance is principal
    per rolling 24 hours
    scope counterparty
```

Six differences from §4, each of them the point:

1. **Limits are sorted by name** — `burst`, `daily_spend`, `transaction_count`,
   `untrusted_counterparty` — not left in authoring order.
2. **`group` declarations are sorted** too, so `hardware` precedes `trusted_suppliers`.
3. **Alignment padding is gone.** `group hardware = {` has one space around `=`, not enough
   spaces to line up with the longest name in the block. Alignment makes a name change rewrite
   unrelated lines.
4. **A blank line separates each declaration kind** and each `limit`; consecutive `group`
   declarations are not separated.
5. **`prohibit` sorts after `approvers` and before every `limit`**, which is also the order it
   is evaluated in (§8.2.1).
6. **`0 USDC_circle` became `0.00 USDC_circle`** — money carries exactly the asset's minor-unit digits, and
   nothing wraps, ever.

Exception clauses would have been sorted by byte value had there been more than one: S4 forces
them disjoint, so their order means nothing.
