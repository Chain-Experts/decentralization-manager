# `governance-settlement-v0`

A Token Standard V2 batch settlement that cannot execute until a
decentralised party's members have confirmed it to threshold.

Depends on `governance-action-v1` and the Splice V2 API packages. Nothing
application-specific: adopt it as it is.

## Who this is for

Any application that moves value to several parties at once and should not
let one operator release it:

- **Paying agents and corporate trustees** — a coupon, a dividend or a
  redemption paid to every holder in one transaction.
- **Venues and exchanges** — a trade with fee legs, where the venue is the
  executor and the counterparties should not depend on it alone.
- **Treasuries and payroll** — a scheduled run of many payments, released
  on a committee's authority rather than one person's.
- **Redemption and paying agents for funds** — same shape, different event.

If your settlement is a V2 batch and your answer to *"who can release
this?"* is *"more than one party"*, this is the module.

## What was missing, and why it exists

`governance-action-v1` gives you the engine: propose, confirm to threshold,
execute. What it does not give you is the action. Every application writes
its own `GovernableAction`, and for batch settlement everybody writes the
same one.

The part that is easy to get wrong is not the governance. It is the
**authority arithmetic**, and it bites late:

Token Standard V2 settles a batch only with the authority of every party in
the settlement's `executors` — `SettlementFactory_SettleBatch` is
controlled by its `actors`, and the standard's default implementation
requires `actors` to equal the `executors`. So the governance party has to
be named among the executors **when the allocations are created**, not when
the settlement is attempted.

Get that wrong and nothing complains until the settle itself, with:

```text
'actors' does not have the same elements as one of 'allowed actors'.
actors: [proposer] allowed actors: [[proposer, governanceParty]]
```

which points at the settlement and not at the allocation that caused it,
hours or days earlier.

This module encodes the arithmetic once, correctly, so that adopting
multi-party authorisation for a batch is a matter of naming a party rather
than of understanding the standard's authorisation model.

## What you get

- **No application code to strip out.** It depends only on
  `governance-action-v1` and three Splice V2 API packages.
- **The authority arithmetic is right by construction** — the proposal
  carries the executors the settlement needs, and `executeImpl` runs with
  the governance party's authority plus the proposer's, which is exactly
  that set.
- **The three properties are tested** on the IDE ledger, with no network
  and no application: below threshold refused, proposer acting alone
  refused, at threshold settled.
- **LF 2.2**, matching `governance-action-v1`. The Splice V2 packages are
  2.1 and a 2.2 package data-depends on them without trouble.

## How to adopt it

1. **When you allocate**, name the decentralised party among the
   settlement's `executors`. This is the step that matters, and it happens
   before any governance does.
2. **The proposer files a `BatchSettlementProposal`.** The proposer is your
   application's executor — a paying agent, a venue, a treasury. It must be
   a member of the governance rules or an *additional proposer*; an
   application service party usually wants the latter, so that it may
   propose and never confirm.
3. **The members confirm** through the Decentralization Manager, to
   threshold.
4. **Any member executes.** The engine exercises `GovernableAction_Execute`
   and `executeImpl` performs the `SettleBatch`.

Two things that are easy to miss:

- The executing node runs `executeImpl`, so anything it touches that lives
  on another participant must travel with the request as a **disclosed
  contract** — typically the registry's rules contract and the sender's
  locked holdings.
- Every participant hosting a member must have **vetted every package the
  settlement touches**, asset packages included. Package names resolve only
  to a version vetted by every informee, so one node missing one package
  fails the whole submission.

## What it deliberately does not do

- **It does not choose your asset.** Any V2 instrument; the module never
  names one.
- **It does not hide the run from the executor.** Executors are observers
  on the allocations, by the standard's design. That is the trade an
  application makes when it asks a committee to approve a payout, and it
  should be stated in the application's own terms.
- **It does not consume anything application-specific.** `executeImpl` is a
  plain `SettleBatch`. If your application needs its own choice exercised —
  a run consumed, a receipt written, records updated — write a proposal
  whose `executeImpl` calls *that* choice, and use this module as the
  pattern. That is the right way round: this one is the general case, and a
  specific one is a forty-line template.

## Tests

`BatchSettlementTest.daml` — three scripts, IDE ledger, no network:

| Script | Expects |
| --- | --- |
| below threshold | the engine refuses |
| proposer alone, outside the vote | the standard refuses the batch |
| at threshold | settled; every leg paid |
