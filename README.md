# Herita ledger fingerprints

Herita records every step in the life of each electronic promissory note in a private, append-only
ledger. Every hour, whether or not anything happened, Herita seals the whole ledger into a snapshot,
signs it with a key held in a hardware security module, and time-stamps it through OpenTimestamps.

This repository publishes only the opaque fingerprint of each snapshot. It reveals nothing about
volume, timing or parties, but each fingerprint pins down the entire ledger at that hour, so any later
rewrite would no longer match what was published here.

- `fingerprints/`: one signed fingerprint per hour, each naming the one before it.
- `attestations/`: the OpenTimestamps proofs for each fingerprint.
- `latest.json`: the newest fingerprint.
- `attestation-key.pem`: the public half of the signing key.

A holder of a note checks its ledger record against this list, offline, with Herita's verifier.
