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
timezone Europe/London

  asset USDC = mint://USDC/Circle/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp

  limit spend
    amount 100.00 USDC
    per fixed day
```

Static ceiling: 100.00 USDC per calendar day, Europe/London. No scope, so one accumulator for
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
timezone America/New_York

  asset USDC = mint://USDC/Circle/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp

  group groceries = { mcc:5411, mcc:5422, mcc:5451, mcc:5462 }

  limit food
    amount 200.00 USDC
    per fixed week
    scope account
```

### 2b · The half that must be refused

```
  limit games
    amount unlimited when category is video_games and offer.discounted    # REJECTED
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
    item:hollow-knight-silksong at 45.00 USDC,
    item:factorio-space-age     at 35.00 USDC,
    item:outer-wilds            at 25.00 USDC,
  }

  limit wishlist_games
    amount from wishlist
    per fixed month
    scope item
```

`amount from wishlist` takes each item's ceiling from the table. `scope item` accumulates per
item, so buying Silksong does not consume Factorio's allowance. The static ceiling is the
maximum entry — 45.00 — computable by reading the file, exactly as S5 requires.

And the behaviour is what the author wanted: a game at or below their stated price goes through
without asking; above it, refused. **A merchant cannot manufacture the condition, because the
condition is a number the principal wrote down.** A sale is precisely when the offer falls under
that number.

> **`prices` and `scope item` are a proposed addition**, not in `spec.md` yet. They arise
> from this example and they preserve every invariant: values stay literals, the ceiling set
> stays finite, and no condition consults accumulated state. Adding them is an engine change
> under review, which is the process §7 S2 describes — not something an author can do.

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
timezone Europe/London

  asset USDC = mint://USDC/Circle/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp

  group sanctioned    = { country:PRK, country:IRN }
  approvers treasury  = { cfo, controller, deputy }

  limit company_monthly
    amount 250000.00 USDC
      except deny when category in sanctioned
    per fixed month
    escalate when exhausted require 2 of treasury up to 400000.00 USDC within 3 days

  limit any_single_payment
    amount 50000.00 USDC
    per rolling 10 seconds
    escalate above 25000.00 USDC require 2 of treasury up to 50000.00 USDC within 2 days
```

Static maximum: **400000.00** on the escalated path, 250000.00 autonomous. Both literals.

### 3b · Department

```
charter dept-eng version 3
extends company-wide@7
resolver full@41
timezone Europe/London

  asset USDC = mint://USDC/Circle/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp

  group cloud = { mcc:7372, mcc:4816 }
  approvers eng_leads = { alice, bob }

  limit dept_monthly
    amount 40000.00 USDC
      except 60000.00 USDC when category in cloud
    per fixed month
    escalate when exhausted require 1 of eng_leads up to 60000.00 USDC within 1 days
```

The cloud exception raises the *department's own* ceiling to 60000. It does not touch the
company's 250000 — H1 takes the minimum, so a cloud payment faces min(60000, 250000).

### 3c · Manager

```
charter manager-alice version 2
extends dept-eng@3
resolver full@41
timezone Europe/London

  asset USDC = mint://USDC/Circle/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp

  limit team_monthly
    amount 8000.00 USDC
    per fixed month
    scope agent

  limit team_daily
    amount 1500.00 USDC
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
timezone Europe/London

  asset USDC = mint://USDC/Circle/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp

  group ci_vendors = { mcc:7372 }

  limit burst
    amount 200.00 USDC
    per rolling 5 minutes

  limit routine
    amount 50.00 USDC
      except unlimited when category in ci_vendors
      except deny      when provenance is at least merchant
    per fixed day
```

Both interesting lines are in `routine`:

- **`unlimited` is legal here** (H4) because this charter extends one. It resolves to the
  parent's effective ceiling — min(1500 daily from Alice, 40000/60000 from the department,
  250000 from the company). Finite, so S5 holds by induction.
- **`deny when provenance is at least merchant`** refuses anything where any field came from
  the counterparty or worse. A merchant-supplied amount is exactly the injection this system
  exists to stop, and `is at least` keeps holding if a worse plane is ever added — an
  enumerated `in { merchant, network }` would not (W1).

### 3e · What a request actually faces

BuildBot requests **900.00 USDC** to a CI vendor, all fields principal-stated.

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
  limit merchant_quoted
    amount 25.00 USDC
      except 500.00 USDC when provenance.amount is principal
      except deny        when provenance.recipient is at least merchant
    per rolling 24 hours
    scope counterparty
```

Three tiers from one limit. A price the human typed: 500. A price the agent inferred or the
merchant quoted: 25. A *recipient address* supplied by the merchant: refused outright,
regardless of amount — because a wrong recipient is unrecoverable in a way a wrong amount is
not.

Note the dotted fields. Bare `provenance` is the tier, the maximum over all four, so
`provenance is principal` would demand every field be principal-stated and reject a perfectly
ordinary invoice where only the venue came from the network.

---

## 5 · Rehearsal on devnet

Same charter, one segment different:

```
  asset USDC = mint://USDC/Circle/4zMMC9srt5Ri5X14GAgXhaHii3GnPAEERYPJgZJDncDU/solana:EtWTRABZaYq6iMfeYKouRu166VU2xqa1
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
    amount 500.00 USDC
      except 400.00 EURC when counterparty is 7xKX…
    per fixed day
```
A cap of "500 USDC or 400 EURC" is two limits pretending to be one, and its ceiling is not a
quantity of anything.

**E304 — overlapping exceptions**
```
    amount 100.00 USDC
      except 500.00 USDC when counterparty in suppliers
      except 900.00 USDC when counterparty is 7xKX…      # a member of suppliers
```
Both hold for that counterparty. S4 rejects and names both rules. Under a first-match language
this pays 500 or 900 depending on parser internals.

**E306 — escalation below base**
```
    amount 500.00 USDC
    escalate above 200.00 USDC require 2 of finance up to 300.00 USDC
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
    amount 40000.00 USDC
    per fixed month
    escalate when exhausted require 1 of eng_leads up to 300000.00 USDC
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

§1.1 of the specification defines exactly one canonical rendering of a charter, and
`roundtrip/` conformance is a byte comparison against it. The example in §4 of the spec is
formatted for reading — aligned `=`, declarations in the author's order. This is the same
charter as an emitter MUST produce it.

```
charter acme-treasury version 7
resolver common@41
timezone Europe/London

  asset USDC = mint://USDC/Circle/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp

  group hardware = { mcc:5045, mcc:5732 }
  group trusted_suppliers = { 7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU }

  approvers finance = { alice, bob, carol }

  limit burst
    amount 100.00 USDC
    per rolling 5 minutes
    scope agent

  limit daily_spend
    amount 500.00 USDC
      except 5000.00 USDC when counterparty in trusted_suppliers
      except deny when date after 2026-12-20 and date before 2027-01-02
    per fixed day in Europe/London
    scope agent
    escalate above 200.00 USDC require 2 of finance up to 5000.00 USDC within 1 days
    escalate when exhausted require 2 of finance up to 5000.00 USDC within 1 days

  limit transaction_count
    count 20
    per fixed day in Europe/London
    scope agent

  limit untrusted_counterparty
    amount 50.00 USDC
      except 0.00 USDC when provenance is principal
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
5. **`except deny when …` is one line.** Nothing wraps, ever.
6. **`0 USDC` became `0.00 USDC`** — money carries exactly the asset's minor-unit digits.

The `except` clauses of `daily_spend` are already in byte order, so they do not move. Had they
not been, they would have been sorted: S4 forces them disjoint, so their order means nothing.
