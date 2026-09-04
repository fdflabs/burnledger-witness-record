# BurnLedger witness record

An append-only record of the signed tree heads of the
[BurnLedger](https://burnledger.io) transparency log, each one timestamped into
the Bitcoin blockchain via [OpenTimestamps](https://opentimestamps.org).

This repository contains no source code. Only observations.

## What this is, precisely

BurnLedger runs an RFC 6962 style append-only log of verification records. A
witness polls that log's public endpoint, checks each signed tree head against a
pinned public key and against the previously accepted head, and appends what it
accepted here. Every entry is then stamped into Bitcoin.

**This is not an independent witness. BurnLedger runs it.** An independent
witness is one operated by someone else, and that remains open. What this record
gives you is narrower and still worth having:

- **History cannot be revised.** Each entry's `entry_hash` covers every prior
  entry, and each `anchors/NNNNNN.json` is a byte-for-byte copy of its entry
  whose SHA-256 is committed to Bitcoin. Rewriting an entry either breaks every
  hash after it, or requires a fresh stamp — and a fresh stamp lands in a block
  mined on the day of the rewrite, while the `observed_at` inside the stamped
  bytes claims an earlier date. You do not need to have kept an old copy to see
  that gap.
- **A reference to check against.** If you hold a BurnLedger verification record,
  the tree head it was issued under should be consistent with what is recorded
  here. A signed head at a tree size where this record shows a different root is
  evidence of equivocation, and it is evidence that verifies without our
  cooperation.

**What it does not give you.** It cannot detect a different history served
privately to one party who never checks this record, and it cannot detect
omission — a record that was never appended to the log is invisible to every
witness, including one you run yourself.

## Verifying an entry yourself

You need no BurnLedger code and no permission.

1. Fetch the log's public key from
   `https://burnledger.io/.well-known/burnledger-keys`, or take it from a
   verification record you already hold. Compare it to the `pinned_key` field.
2. Take the entry's `signed_payload` — the exact canonical bytes the log signed —
   and verify `signature` over it with stock Ed25519.
3. Confirm `entry_hash` chains: it is the SHA-256 of the entry's fields together
   with `prev`.
4. Verify the Bitcoin timestamp:
   `ots verify anchors/NNNNNN.json.ots` (the file it attests to is
   `anchors/NNNNNN.json`). `ots info` prints the block height; any block explorer
   will confirm that block's merkle root and its time.

Step 4 is the one that matters. Steps 1 to 3 prove the record is internally
consistent; step 4 proves it is not backdated, and it depends on nobody's
honesty — including ours.

## Files

| Path | What |
|---|---|
| `heads.jsonl` | one line per accepted signed tree head, hash-chained |
| `anchors/NNNNNN.json` | a byte-for-byte copy of entry NNNNNN, the thing stamped |
| `anchors/NNNNNN.json.ots` | its OpenTimestamps proof |
| `evidence/` | records written when a check failed, and the bootstrap anchor |
| `latest.json` | the most recent run's status — liveness, not evidence |

A gap in the sequence, or a run that stopped, means the witness stopped — not
that the log misbehaved. Absence of an entry is not evidence of anything.
