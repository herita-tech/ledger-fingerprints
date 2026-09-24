# The Herita ledger rulebook

**Status: draft.** The genesis checkpoint was published on 17 September 2026
(`checkpoints/000000000002.json` in `herita-tech/ledger`, covering the first production instrument), so
from that checkpoint on this document describes history as well as intent. Whether it is in force as
the rules of the system is a decision for counsel, not for this file. Where a rule is not yet enforced
in code, it says so.

This is the artifact the Electronic Trade Documents Act 2023 s.2(5) directs a court to weigh as **the
rules of the system**. It is written to be read without access to Herita, and it is kept with the
event export it governs so that the two cannot drift apart.

## 1. What this rulebook is for

Herita issues electronic promissory notes. A holder needs to answer three questions without trusting
Herita and, if necessary, without Herita existing:

1. **Is this document what it claims to be?** — its hash appears in an event of the ledger.
2. **What is this instrument's current state?** — replay its events and see.
3. **Has the record been altered?** — the chain is hash-linked and anchored externally.

Everything below exists to make those three answerable without taking Herita's word for them. §10 says
who can answer which of them today, and §14 what is still missing.

## 2. What is published, and to whom

**The ledger is private.** Its event count and timing would show how much business the platform does
and when a new customer arrives, which is commercially confidential to Herita and to its customers.
Nothing about volume is public.

Each anchor cycle writes, to the designated channel:

| Artifact                 | Contents                                                       |
| ------------------------ | -------------------------------------------------------------- |
| **Event export**         | Every event's envelope and public payload, in sequence order   |
| **Checkpoint cores**     | Format 1 or 2 (§9), immutable once written                     |
| **Attestations**         | Timestamp proofs keyed by core hash, append-only               |
| **This rulebook**        | The rules in force for the versions present in that export     |
| **The offline verifier** | One readable file, runnable with no Herita systems in the loop |

The designated channel is a private repository, deposited with the escrow agent on every production
release. The escrow agent, auditors and regulators under confidentiality, and Herita's own monitoring
read it. A financing party or buyer receives the records for its own notes, including a **ledger
record** per note: that note's events with a proof that they are its complete history as of one
format-2 core (§9), anchored at the first core covering its latest event, never the newest. Anyone
holding a note's document can look that one note up at Herita's verify page, which shows that note's
events and nothing about any other note or the ledger as a whole, and can get its ledger record by
presenting the document itself — a document hash alone is not enough.

From the first format-2 core, one more thing is public: a **fingerprint** per hourly slot, written
whether or not anything happened, carrying only the slot, the core's hash, the previous core's hash and
Herita's signature (§9). It reveals nothing about volume, and it lets anyone check that the private
ledger was not rewritten.

The channel was public from 17 to 22 September 2026. What was published then stays valid under these
rules, and anyone who kept a copy can still verify it.

**Never published, to anyone:** documents, amounts, party names, invoice numbers, bank details, or the
salts that open party commitments. The export carries hashes and opaque commitments only. Documents are
always retrievable by the party that holds them.

## 3. The event record

Events form a single global chain. Each event carries an envelope and two payload sections.

**The envelope** commits to exactly ten fields: `eventId`, `instrumentId`, `instrumentSeq`,
`occurredAt`, `payloadPrivateHash`, `payloadPublicHash`, `prevHash`, `schemaVersion`, `seq`, `type`.
Its hash is the SHA-256 of that object in canonical form.

**Canonical form is RFC 8785 (JCS).** Every hash in this system is taken over JCS bytes, so two
parties independently serialising the same record obtain the same hash.

**Linkage.** `seq` is contiguous from 1 with no gaps — a hole means a row was removed, and is a hard
failure, not a warning. Each event's `prevHash` is the preceding event's `eventHash`. The first event
links to a fixed genesis constant, the SHA-256 of the empty string:

```
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

**The payload split.** The public payload carries the instrument id, document hashes, party
commitments, and a small number of enumerated fields. The private payload carries the material a party
holds — amounts, signature references, storage keys — and appears in the export only as a hash. A
holder with the private bytes can prove they match; anyone else treats the hash as opaque.

**`seq` order governs.** `occurredAt` is informational until bounded by an anchor. Where the two
disagree, sequence wins.

## 4. Event types

Twelve types are defined. Six are **enabled** — the append can produce them today:

`ISSUE` · `ENDORSE` · `FUND` · `REPAYMENT_SETTLED` · `GATE_REJECTED` · `KILLED`

Six are **reserved** — defined but not yet enabled: `ACCEPT` · `TRANSFER_CONTROL` · `DISCHARGE` ·
`CONVERT_TO_PAPER` · `VOID` · `CORRECTION`.

**A reserved type appearing in any chain is, by construction, a write that did not go through the
append.** A verifier must report it as such. Enabling a type is a versioned rulebook change that ships
its replay behaviour, its evidence policy and its party-slot rules together; events written while a
type was reserved keep failing verification forever.

**Instrument-legal versus operational.** `ISSUE`, `ACCEPT`, `ENDORSE`, `TRANSFER_CONTROL`, `DISCHARGE`,
`CONVERT_TO_PAPER` and `VOID` change the legal state of the note and require the corresponding legal
act and artifact. `FUND`, `REPAYMENT_SETTLED`, `GATE_REJECTED` and `KILLED` are operational facts. The
log never asserts a legal act that did not occur as such, and never omits an operational fact that
did.

`FUND` and `REPAYMENT_SETTLED` each record a declaration made in Herita by, or on behalf of, the party
entitled to make it: that the supplier payment was sent, and that the repayment was received. Herita
holds no client funds, has no connection to any bank account and does not observe money moving. The
chain commits to the declaration having been recorded, and to when; it is not evidence that funds
moved.

## 5. Deriving an instrument's state

An instrument's state is the fold of its own events in `instrumentSeq` order. Every step must be a
legal transition: the first event must be `ISSUE`, a terminal state accepts nothing further, and the
product issues at most one endorsement per instrument.

Two funding chains exist, and which one an instrument follows is written on its face. Under the
**endorser** model the note is endorsed to the financing party before funding: `ISSUE` → `ENDORSE` →
`FUND`. Under the **named-payee** model (schema version 6 and later) the payee and controller are the
financing party from `ISSUE` onward, no endorsement ever occurs, and the chain runs `ISSUE` → `FUND`
directly. `FUND` from `issued` is legal only for events written under schema version 6 or later; in a
chain written under an earlier version it remains, as it always was, proof of an out-of-band write.

**Erratum, 23 September 2026.** The paragraph above originally said that under the named-payee model
"the payee and controller are the financing party". From this change they are the **named payee** — the
bank the note is drawn to — and the named-payee `ISSUE` is appended at the payout rather than after
Herita's checks, so the note first exists at the moment it is paid for (delivery against payment), in
the named payee's hands. The rule profile is unchanged: the party-slot table has always admitted any
party in the `ISSUE` slots, so this corrects a description, not a rule, and needs no schema version. No
production instrument was written under the old sentence: no named-payee programme had drawn an advance
in production when it was corrected.

This fold is enforced twice — once by the append, and again by the verifier. The second is the one
that matters: the fraud this system exists to detect is an event that never went through the append,
hand-written with correct hashes and linkage. Without an independent fold, such an event verifies
clean.

**A slice that begins mid-chain cannot judge an instrument whose history began earlier.** The verifier
reports those instruments as unjudged rather than passing them. "First event is `ISSUE`" is
established only by a run rooted at the start of the chain, or over an instrument's full history.

## 6. Evidence policy

Each type must carry the evidence its legal act produces — an `ISSUE` without the signed note's hash,
or without the parties it binds, is refused. The policy both requires and forbids: fields not allowed
on a type are rejected on that type.

The precise claim this buys, stated plainly: **the chain commits to the evidence supplied and every
event satisfies the declared policy.** It does not prove human authorisation beyond what the
referenced artifacts themselves prove.

## 7. Parties

Party identity is published as a **salted commitment**, never in clear:

```
commitment = sha256( salt ‖ JCS(descriptor) )
```

One independent random salt per event and party slot, so the same company in two slots of one event
produces unlinkable commitments. Whoever holds the salt proves the binding by recomputation.

A descriptor binds `partyId`, the name as registered at event time, country, and an **external
identity**. `partyId` alone is an internal identifier and independently meaningless, so the external
identity is what makes a commitment mean anything without Herita. Names are recorded verbatim and never
normalised; cross-event identity is byte-exact id equality, never name comparison.

From schema version 5 a descriptor **states which namespace its external identity lives in**, and the
three are not equally strong:

| Scheme     | Identity                  | Strength                                                           |
| ---------- | ------------------------- | ------------------------------------------------------------------ |
| `register` | Commercial-register id    | External and immutable. The registry, not Herita, is the authority |
| `vat`      | VAT identification number | External and immutable. For entity forms with no register id       |
| `name`     | The party's legal name    | **Weak.** Mutable by its owner, and resolvable to no registry      |

**Precedence is enforced, not advisory.** A descriptor claiming a weaker scheme while carrying a
stronger identifier is refused at the append and reported as a failure by the verifier, so a party that
has a register id can never be committed under its name.

**What a `name` commitment is worth, stated plainly.** A legal name can be changed by the party that
holds it. Two events naming the same company therefore need not carry the same identity string, and a
reader outside Herita cannot resolve a renamed party to any registry. `partyId` still links the events,
so the chain does not break — what degrades is what the commitment proves about _who_ the party was.
Anyone relying on such a commitment is entitled to know that, which is why the scheme is published in
the descriptor rather than left to be inferred.

Schema versions **4 and earlier** predate the scheme field: for those events a commercial register id
was mandatory, and they are still judged that way. A rule is added going forward, never applied
backwards — see §8.

Four roles exist: `drawer`, `payee`, `endorsee`, `controller`. Payee and controller are distinct — one
is entitled to the money, the other holds electronic control — and replay folds them independently.

**Recourse parties are evidenced by the instrument, not the envelope.** Where a note carries an
additional obligor, such as a co-signing company, that fact is in the signed document whose hash the
chain commits to, not in the party commitments, which track the control chain.

## 8. How the rules change

The state machine, the evidence policy, the party-slot table and the enabled type set together form
one **rule profile**, identified by a schema version. The current version is **6**.

- Every rule change ships as a new schema version carrying its full profile.
- **An event is judged forever under the profile in force when it was written**, never under a later
  one.
- Profiles are append-only rulebook content, published with the export.
- Mutating a profile in place is not supported, and an instrument does not change profile mid-history.

A verifier that predates an event's schema version cannot confirm what a later rulebook permitted. It
reports those events as unjudged rather than as tampering — but a version it cannot corroborate never
buys an event out of the checks, or a forger would simply inflate the field.

## 9. Checkpoints and anchoring

There are two checkpoint formats. Format 1 was in force from the genesis checkpoint; format 2 is in
force from the first format-2 core, whose predecessor is the last format-1 core, so the two form one
chain. Cores of each format are judged forever under that format's rules.

**Format 1.** A core is `{ headHash, seq, prevCoreHash, exportDigest }` and is **immutable once
written**. Timestamp proofs are separate append-only records keyed by the core's hash, because such
proofs are replaced when they upgrade, and a core that embedded one would change its own bytes.

**The canonical-head rule is extension, not height.** A checkpoint is canonical only if it extends the
previously accepted one: its export must contain the prior checkpoint's head event unchanged at the
prior sequence. "Highest anchored sequence wins" would be unsafe — anchoring proves time, not truth,
so a fabricated longer branch could otherwise outrank honest history. A non-extending checkpoint is
rejected **regardless of its sequence**, and any two anchored checkpoints where neither extends the
other are cryptographic proof of a fork: evidence of fraud, never an ambiguity the verifier
adjudicates.

Cores are signed by a **dedicated attestation key held apart from the credential that writes to the
publication channel**, so obtaining write access to the published repository is not sufficient to mint
an accepted checkpoint. The signature algorithm is ECDSA over NIST P-256 with SHA-256. A key is
identified by the SHA-256 of its public key in SPKI DER form — deliberately not by any cloud provider
identifier, which would be meaningless once the provider account is gone.

Key rotation and cessation are announced by signed statements inside the checkpoint chain itself, with
the outgoing key signing the handover and a notice window before it takes effect, so key history is
verifiable offline. A statement is signed over its RFC 8785 canonical form without the signature field,
with its `kind` inside the signed bytes. An unsigned statement, or one whose signature does not verify
against the attestation key, counts for nothing: it cannot establish finality or change a key.

**Anchoring is not publication.** A timestamp proves a hash existed; it does not publish the events,
identify the latest head, or let a stale holder discover later events. Discoverability comes from the
checkpoints in the designated channel.

**Format 2.** A core is cut for every hourly slot, whether or not anything happened, and is private:

| Field             | Meaning                                                                                |
| ----------------- | -------------------------------------------------------------------------------------- |
| `v`               | `2`                                                                                    |
| `tick`            | The slot: the start of a UTC hour                                                      |
| `nonce`           | 32 random bytes, so no core can be predicted from another                              |
| `prevCoreHash`    | The previous core's hash, format 1 or 2                                                |
| `instrumentsRoot` | The root of the instrument map, below                                                  |
| `detailsHash`     | The RFC 8785 hash of `{ seq, headHash, eventsRoot, exportDigest }`, which stay private |

`seq`, `headHash` and `exportDigest` mean what they mean in format 1. `eventsRoot` is the RFC 9162
Merkle tree hash over every event hash in sequence order. The core's hash is the RFC 8785 hash of the
core.

The **instrument map** commits to every instrument's complete history. Each instrument's entry is its
event count and a running digest `d_k = SHA-256(0x02 ‖ d_(k-1) ‖ eventHash_k)`, starting from 32 zero
bytes, placed in a sparse Merkle tree at the position `SHA-256(instrumentId)` names. A leaf is
`SHA-256(0x00 ‖ key ‖ count as 8 bytes big-endian ‖ digest)`, a node `SHA-256(0x01 ‖ left ‖ right)`, an
empty subtree 32 zero bytes, and a subtree holding one leaf is that leaf. Because each instrument has
exactly one place in the map, a record of one instrument that matches the root is its whole history as
of that core: a record with a later event left out, or a second history for the same instrument, cannot
match.

**The fingerprint** is the public record of a core: `{ v, tick, coreHash, prevCoreHash, signature,
witnesses }`. The signature is by the attestation key over the RFC 8785 form of `{ kind:
"herita-ledger-fingerprint", v, tick, coreHash, prevCoreHash }`. `witnesses` holds co-signatures over the
same bytes by an independent party that has checked the core against the private ledger; none is
appointed yet, and a verifier does not rely on them. The core's hash is what is time-stamped.

The format-2 rules:

- **One fingerprint per slot, each naming the one before.** Two signed fingerprints for the same slot,
  or two naming the same predecessor, are proof of a fork.
- **A slot is never backfilled.** A slot with no fingerprint is an outage, not a sign of activity. A
  core that was stored but not published is published with the next slot's fingerprint, so the chain of
  fingerprints never has a hole.
- **Extension across idle hours.** Consecutive cores may attest to the same `seq`, and must then attest
  to the same head. Two cores at the same `seq` with different heads or event roots are proof of a fork,
  whether or not their slots are adjacent.
- **Time.** A fingerprint whose earliest time-stamp is more than 24 hours after its slot is reported:
  the time-stamp bounds when the core existed, not the slot it claims.

## 10. What a verifier can and cannot establish

This section is the one to read before quoting any verdict.

| Claim                                          | Established by                                                               |
| ---------------------------------------------- | ---------------------------------------------------------------------------- |
| The chain is internally intact                 | Any verifier, any version, offline, forever                                  |
| This document is the one committed to          | Re-hashing the document against the event                                    |
| This instrument's state **as of** checkpoint C | A rooted run plus that checkpoint                                            |
| This is the **current** state                  | Only by checking the designated channel for the latest accepted core         |
| The private ledger was not rewritten           | The full export and cores, checked against the public fingerprints           |
| The fingerprint list is Herita's and unforked  | Anyone, from the public list and the attestation key alone                   |
| This note's history is complete as of core C   | Its ledger record: the events plus the instrument-map proof against C        |
| This PDF is the note the record describes      | The verifier's `--pdf`, matched against the ISSUE or ENDORSE hash            |
| Core C existed by time T                       | Its OpenTimestamps proof, confirmed against a block header the holder trusts |

**An archive proves "as of", never "latest".** Finding a later checkpoint proves a holder's position
existed; it cannot prove no still-later checkpoint exists. Currentness is a protocol, not an
assumption: it requires checking the designated channel. Because the channel is private, only those
who can read it — the escrow agent, an auditor, Herita — can establish currentness today. A holder
without that access has its position **as of** the latest records it was given.

**The public fingerprints are checkable by anyone and prove less than the old public export did.** They
prove that Herita signed one linear history, hour by hour, and when each hour's core existed. They do
not show any event. Checking that the private ledger matches them needs the ledger itself, which is what
the escrow agent and auditors hold; a split view shown to one holder and not another is caught when
someone with the full ledger checks it against the list, and earlier once an independent witness
co-signs each fingerprint.

Where required evidence is absent, the verifier says "evidence unavailable" — never a clean pass.

**Integrity never depends on understanding the rules.** Hash linkage, canonical form and the three
hashes are checkable by any verifier at any generation. An old tool can therefore honestly report
"the chain is intact; I cannot judge this instrument's legality" — which is a far more useful answer
than refusing to speak at all.

## 11. Duplicate financing

An instrument may be financed against a given receivable once. The check runs inside the same
transaction as the `ISSUE` append and its outcome is recorded as committed evidence in the event, so a
holder can see that the constraint ran rather than take it on trust.

The check rests on keyed one-way values derived from the receivable's identity; the values themselves
are never published, because a deterministic hash of a low-entropy identity is a dictionary away from
cleartext. The constraint's versions and verdict are published; its inputs are not.

Under the named-payee model the claim is taken when Herita's checks pass and the funding request is
sent, before any bank is asked to pay, and the `ISSUE` appended at the payout picks that same claim up.
**The claim may therefore predate the `ISSUE` by the funding window**; the verdict the event commits is
the one reached when the claim was taken.

Stated honestly: this constrains Herita's own issuance path. It is not a claim that no instrument
anywhere was financed twice — a receivable financed entirely outside this system is the same
taker-beware risk paper bills have always carried.

## 12. Instruments that predate the ledger

Instruments issued before the ledger was recording are marked as such, once, at cutover, and are never
retrofitted with events. This is a recorded fact, not an inference from an empty chain — because "has
no events" also describes an instrument whose `ISSUE` went missing, and the two must never share a
verdict. A verifier reports the first as predating the ledger and the second as a defect.

## 13. Reliability evidence

Under s.2(5) a court weighs the rules of the system and any independent audit. Herita's position rests
on:

- **This rulebook**, kept with the export it governs.
- **The acceptance suite** — including adversarial drills in which a chain is deliberately forged,
  truncated, re-ordered, double-endorsed and re-anchored, and must be caught.
- **The offline verifier**, kept in source with the export so the claims here are checkable rather than
  asserted.
- **An information security management system built to ISO 27001**, covering what cryptography does
  not: who can reach production, how change is controlled, how access is logged. The anchor proves
  tampering did not happen; the controls constrain whether it could. Herita is not yet certified; see
  §14.

ISO 27001 does not establish exclusive control, uniqueness, or non-forgery. Those are properties of
the signatures, the chain and the anchor. It should never be offered as the answer to "how do you
guarantee uniqueness".

## 14. What is not yet true

Stated here rather than left to be discovered:

- The genesis checkpoint (seq 2, published 17 September 2026 21:20 UTC) has been verified offline by
  Herita with the published verifier, including its signature against the published attestation key,
  and its OpenTimestamps attestation has upgraded to a block-bound proof. No independent party has
  verified it yet.
- The ledger has no public anchor. Since the channel became private, no signed, time-stamped record of
  its checkpoints is public, so only a party that can read the channel or the escrow deposit can check
  independently that the ledger was not rewritten. A public fingerprint per hour, which reveals nothing
  about volume, is planned to close this.
- The software and the designated channel are deposited with the escrow agent on every production
  release; the three-party escrow agreement that names a beneficiary is not yet signed.
- `FUND` and `REPAYMENT_SETTLED` rest on a declaration alone. No payment evidence is attached, and no
  value date is recorded: their `occurredAt` is the time the declaration was recorded.
- Six event types remain reserved, including control transfer.
- The rules above are enforced in code; the acceptance suite covers them; no independent technical
  audit has yet been performed.
- Herita's information security management system is being built to ISO 27001. It has not been
  audited and is not certified.
