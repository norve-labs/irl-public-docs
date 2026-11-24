# Regulatory context (informative)

*Context, not a compliance claim.* IRL does not certify or ensure compliance with any regulation, and Norve is not a compliance provider. This page describes what IRL records and what those records do and do not prove. Whether they are useful for your obligations depends on who you are and where you operate. Not legal advice.

---

## What IRL records

For every order it authorizes, IRL keeps:

- **The mandate verdict** (policy result and policy id, version and hash), made before the order is sent. Agent status, model hash, venue and asset are checked first; an order that fails them gets no trace. The verdict is recorded with the trace but is not part of the sealed hash.
- **The sealed snapshot**: the agent's stated intent and context, hashed with SHA-256 over IRL canonical JSON (whitepaper §7.1) before execution. The snapshot includes the decision time (`valid_time`) and the record time (`txn_time`).
- **The binding to the exchange order id**: `final_proof = SHA-256(reasoning_hash ‖ exchange_tx_id)`, plus a `MATCHED` or `DIVERGENT` status reported by the engine. The status is not part of the sealed hash.
- **Inclusion in a Merkle root** that is anchored on Bitcoin through OpenTimestamps, in batches.

Some rejections leave no trace: an unknown or inactive agent, a model hash mismatch, or a venue or asset outside the mandate is refused before a trace is created. Mandate rejections from the policy check are recorded as `HALTED` on a best-effort basis.

In `SHADOW_MODE`, notional-cap and side violations are recorded but not blocked; registry checks (status, model hash, venue, asset) still block.

## What a verifier can check

Anyone with a bundle can check, with the open-source verifier and without trusting Norve's servers:

- That each `final_proof` equals SHA-256 of its `reasoning_hash` and exchange order id (consistency, not tamper evidence: `final_proof` is not anchored).
- That each record's `reasoning_hash` is in the Merkle root of the anchor covering its `txn_time`.
- The Bitcoin anchor, with the standard OpenTimestamps client and public block data, using the `.ots` file downloaded from the norve.dev verifier (`ots verify -d <merkle_root> anchor.ots`; run `ots upgrade anchor.ots` first if the receipt is still pending). The Python client checks against a local Bitcoin Core node; without one it reports the block height and root to check by hand. The files written today by `irl-verify --dump-ots` lack the OpenTimestamps file header and are not accepted by the standard client.

For `snapshot_version` 1 traces, recomputing the `reasoning_hash` itself requires the full sealed snapshot, which a public bundle does not include.

## What IRL does not prove

- That the agent's reasoning was correct, honest or profitable.
- Which model actually ran. The model hash is declared by the operator; IRL checks at authorization that the declared hash matches the registered one; the check is not part of the sealed hash.
- That `valid_time` reflects when inference happened. The agent supplies it; IRL only enforces `valid_time < txn_time`.
- That the exchange order id was not replaced after binding. Check it against the venue's records.
- That a bundle is complete. Anchoring detects edits to anchored records, not omitted records.
- That the `MATCHED`/`DIVERGENT` status or the mandate verdict was not edited after the fact. They are recorded, not sealed.
- That the market regime signal (MTA) was accurate. That is the MTA operator's responsibility.

## Responding to a request about a specific trade

1. Find the trace by exchange order id or client order id:
   ```sql
   SELECT trace_id, txn_time, reasoning_hash, final_proof, verification_status
   FROM irl.reasoning_traces
   WHERE exchange_tx_id = '<exchange_order_id>'
      OR client_order_id = '<internal_order_id>';
   ```
2. Fetch the full trace: `GET /irl/trace/<trace_id>`.
3. Export the bundle and let the requester verify it with the open-source verifier.

---

*This page replaces the former Compliance Guide and Regulatory Mapping, which paired regulations with IRL features. Those pairings were removed because IRL does not establish compliance with any regulation.*
