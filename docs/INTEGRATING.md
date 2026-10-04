# Integrating with a Canton you already run

[`CUSTOM_DAML_TEMPLATES.md`](CUSTOM_DAML_TEMPLATES.md) covers writing a
`GovernableAction` and driving it: the package layout, `proposal_cid` and its
placeholder action, `disclosed_contracts`, granting propose-only rights. It
assumes Decentralization Manager is already talking to your participant.

This page is about getting to that point, and about the things that bite
afterwards. Seven items. Four are configuration on the Canton side, where the
errors name the vote rather than the config and so read as governance
faults. Three are operational, and each arrives long after the step that
caused it.

Everything here was found by doing it: first against our own
five-participant Canton, then on DevNet against a party shared with the
BitSafe team, through to a governed Token Standard V2 settlement.

## 1. Canton must have authentication enabled — even locally

A node with no `auth-services` at all will fail every vote:

```text
INVALID_TOKEN(8): The submitted request is missing a user-id:
Cannot default user_id field because claims do not specify an user-id.
Is authentication turned on?
```

Onboarding and package distribution still work, because those go through the
Admin API. It is the first `/governance/confirm` that fails, which makes it
look like a governance problem rather than a configuration one.

A command needs a user id, and Canton takes it from the token's `sub` claim.
With no auth service there are no claims and nothing to default from. Add, on
the participant's **`ledger-api`**:

```hocon
ledger-api {
  auth-services = [{
    type = unsafe-jwt-hmac-256
    target-audience = "https://canton.network.global"
    secret = "unsafe"
  }]
}
```

Those are the values `DECPM_CANTON_HMAC_AUDIENCE` and
`DECPM_CANTON_HMAC_SECRET` default to, so DecMan lines up with no further
configuration.

## 2. `auth-services` belongs on `ledger-api`, not `http-ledger-api`

Putting it on the JSON API block too stops Canton booting. Worth saying
explicitly, because a config file usually lists the two blocks next to each
other and it is a natural mistake.

## 3. Tokens need `exp`, and Canton caps how far out it may be

Two different errors, one after the other:

```text
Could not verify JWT token: token has no expiration time
Could not verify JWT token: token lifetime (2099-01-01T00:00:00Z) too long
```

A demo token that never expires is convenient, and Canton refuses both that and
a distant expiry. The fix is on the same block:

```hocon
ledger-api { max-token-lifetime = Inf }
```

which is what the Splice LocalNet bundle sets
(`conf/canton/app.conf`). Then any `exp` is accepted.

## 4. Use `participant_admin`; do not create a ledger user first

With `target-audience` set, Canton reads audience-based tokens and takes the
user id from `sub` — so **the user must already exist**. Creating one from the
bootstrap console needs a token the console does not have, so it is a loop.

Canton creates exactly one user for itself, `participant_admin`, with
`ParticipantAdmin` rights. Name it in the token and the loop disappears:

```text
DECPM_CANTON_HMAC_SUBJECT=participant_admin
```

Any party the governance flow acts as still needs `CanActAs` granted to that
user, as usual.

## 5. Peers need every package your action touches, not just yours

Distributing your own DARs to a peer is not enough. Naming a decentralised
party as an approver makes the peer's participant a stakeholder — and, if the
party is also an executor, it must validate everything the action does.

Ours settles a Token Standard V2 batch, so the peer needed the asset packages
too. Without them:

```text
UNRESOLVED_PACKAGE_NAME(11): Interpretation error: Update failed due to a
failed package name resolution: splice-test-token-v2
```

Package names resolve to a version vetted by **every** informee, so one node
missing one package fails the whole submission. The error names the package
but not the node, and it arrives at the settlement rather than at
distribution — long after the step that caused it.

Worth a line in the DAR-distribution docs: send the transitive set your
`executeImpl` reaches, not only the package your action is defined in.

## 6. A member with no node can lock the party out of its own rules

The one that cost a day. Adding a governance member is a single dialog, and
its failure mode is a party that can no longer govern itself.

We added a party as a member that had no Decentralization Manager of its
own - it was a leftover application party, added in error. A member that
cannot confirm still counts towards the threshold. With three members at a
threshold of three, two confirmations were reachable and three were
required, so **every** action was stuck, including the remove-member action
that would have fixed it.

Two things make this sharper than it sounds:

- `get_member_party_id` resolves the confirming member from the node's
  stored credentials, taking the **first** whose `dec_party_id` matches. So
  one node casts exactly one confirmation, and adding a second credential
  for the same decentralised party does not give you a second vote.
- The UI does not let you edit **Member Party ID** once saved.

The way out is `PUT /party-config`, which accepts `member_party_id` and
treats absent credential fields as "keep existing". Point the node at the
stranded member, confirm, point it back:

```text
PUT /party-config   { dec_party_id, member_party_id: <stranded>, user_id, ... }
POST /governance/confirm
PUT /party-config   { dec_party_id, member_party_id: <original>, user_id, ... }
```

Worth a warning in the add-member dialog: **a member that cannot confirm
still counts towards the threshold.** Worth a line in the docs too, that
`PUT /party-config` is the escape hatch when it happens.

## 7. The UI cannot execute an action that needs disclosed contracts

`CUSTOM_DAML_TEMPLATES.md` documents `disclosed_contracts` on
`POST /governance/execute`, and documents it well. What is not said is that
**the Approvals tab has no way to supply them**. Its Execute button submits
with none, so a custom `GovernableAction` whose `executeImpl` reaches a
contract the executing node has not seen fails on the click:

```text
CONTRACT_NOT_FOUND(11): Contract could not be found with id 00bf0947...
```

The id in the message is the contract that should have been disclosed - in
our case the registry's `TokenRules`. Nothing in the error says
"disclosure", so it reads as a missing contract rather than a missing
parameter.

Pasting them in would not help even if the dialog offered it: our two blobs
are 692 and 1,644 characters, and a settlement with one locked holding per
sender grows from there.

So for an action of this kind, **execute over the API, not from the UI**.
Worth saying on the Execute button itself, or disabling it for actions whose
`executeImpl` is known to need a choice context.

The failure is harmless - the proposal and its confirmations survive, and a
subsequent API execute succeeds.

## Two worked examples

**Locally, in Docker.** `judge/govern.sh` in
[Indivisa](https://github.com/Chain-Experts/Indivisa) does items 1 to 4
against a five-participant Canton: peer mesh, onboarding at threshold 2,
member parties, governance rules, admitting an additional proposer, then
propose, confirm, execute. It is a port of `hackathon/seed.sh` and is
deliberately readable as a reference. It skips DAR distribution, because
that Canton vets the packages at bootstrap - worth knowing that the step is
optional when you control the node.

**On DevNet, against a party shared with your team.**
`infra/bitsafe/govern-devnet.ps1` and its neighbours do the same where the
second member is somebody else's. Two of them exist only because of items 5
and 6:

| Script | Why |
| --- | --- |
| `verify-packages.ps1` | Asks the ledger which packages resolve for the shared party, rather than trusting a distribution's reported status |
| `confirm-as-member.ps1` | Casts a confirmation as a member the node is not configured with, via `PUT /party-config` - the escape hatch for item 6 |

A coupon settled through that party on 29 September 2026 at two of two,
update id
`1220eb437ef12213805d81db4f425056c1b0fdf2eb4f1de3f1882d037eefbd60c6ac`.
