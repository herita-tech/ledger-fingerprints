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
- `verifier/verify-ledger.js`: the offline verifier.
- `rulebook/`: the rules the verifier checks, and what each verdict can and cannot establish.

## Checking a note

A holder of a note has two PDFs from Herita: the signed note, and its **ledger record**. The ledger
record carries its proof as an attached file, `ledger-record-<note>.json`. Save it with any PDF
reader's attachments panel, or `pdfdetach -saveall ledger-record-<note>.pdf`.

Then, from a clone of this repository, with [Bun](https://bun.sh) 1.3 or later:

```sh
bun verifier/verify-ledger.js pack /path/to/ledger-record-<note>.json \
  --attestation-key attestation-key.pem \
  --fingerprints fingerprints/ \
  --pdf /path/to/signed-note.pdf
```

It checks, with no network and no Herita account:

- the note's events are intact and follow the rules, and are its **complete** history as of one
  hourly snapshot;
- that snapshot's fingerprint is signed by the key you supplied, and appears in this public list, so
  you were shown the same ledger as everyone else;
- the signed note is the document the note was issued or endorsed with.

**Check the key before trusting it.** The key's id is the SHA-256 of its public key. For Herita's
production ledger it is `1189873e644ebe4df5b8a2628bff33417b8b3648edbea99ab1967a651dadbf45`; take that
value from your agreement with Herita, not from this page, because a replaced repository would name
its own key. Compute the id of the file here with:

```sh
openssl pkey -pubin -in attestation-key.pem -outform DER | shasum -a 256
```

## Confirming when a snapshot existed

The verifier follows each OpenTimestamps proof to the block it reaches, then names the block and the
merkle root that block must have. To confirm it, supply the block's header from a node you run or a
source you trust, one `<height> <header hex>` line per block, and run again with
`--block-headers headers.txt`. The verifier checks the header commits to the proof and carries real
proof of work, and reports when the snapshot existed by. From a public explorer:

```sh
h=<height>
echo "$h $(curl -s https://mempool.space/api/block/$(curl -s https://mempool.space/api/block-height/$h)/header)" >> headers.txt
```

The check is only as independent as the header's source. Two explorers agreeing is a reasonable bar;
a node you run is the strongest.

## With an AI agent

Paste this into a coding agent (Claude Code, Codex, Cursor) opened in a folder holding the signed
note and its ledger record:

> Verify a Herita promissory note offline, and change none of my files.
>
> 1. Clone https://github.com/herita-tech/ledger-fingerprints into a new folder.
> 2. Check that the SHA-256 of `attestation-key.pem`'s public key
>    (`openssl pkey -pubin -in attestation-key.pem -outform DER | shasum -a 256`) equals the key id in
>    my agreement with Herita. Ask me for it if you do not have it. Stop if they differ.
> 3. Save the `ledger-record-*.json` attachment from the ledger record PDF.
> 4. Run `bun verifier/verify-ledger.js pack <that json> --attestation-key attestation-key.pem
>    --fingerprints fingerprints/ --pdf <the signed note>`.
> 5. For every block the output names, fetch the 80-byte header from both mempool.space and
>    blockstream.info. Stop if they differ. Write them to `headers.txt` as `<height> <header hex>`
>    lines and run step 4 again with `--block-headers headers.txt`.
> 6. Report the verdict and every line under "notes" and "What this run cannot prove" exactly as
>    printed. Do not summarise a failure as a pass.

## Keeping your own copy

Nothing here depends on GitHub staying up or on Herita keeping this repository. A holder of a
ledger record already has that record's fingerprint, signature and proof inside the PDF. For
everything else, hold the whole history yourself: `git clone` this repository and run `git fetch`
on a schedule, and you have an independent copy of every fingerprint ever published.

Software Heritage, the public archive run by Inria with UNESCO, also takes a copy of this
repository every day at Herita's request (`.github/workflows/software-heritage.yml`). Every commit
there carries a permanent identifier (a SWHID) that a report can cite:
<https://archive.softwareheritage.org/browse/origin/directory/?origin_url=https://github.com/herita-tech/ledger-fingerprints>

## The verifier

`verifier/verify-ledger.js` is one self-contained JavaScript file with no dependencies beyond the
runtime's own `crypto`, `fs` and `path`. It reads files and prints a verdict; it makes no network
request. It is not minified, so it can be read before it is run. `--help` lists every option.

The file published here was built from Herita's source at commit `64f90f9f`. Its SHA-256 is
`5a381dd7b502f912bb638e28a93c179aa0a40825c1cbefdff40da030073f6aa6`.
