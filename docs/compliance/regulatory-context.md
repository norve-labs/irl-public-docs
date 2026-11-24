# Regulatory context (informative)

*Context, not a compliance claim.* IRL does not certify or ensure compliance with any regulation, and Norve is not a compliance provider. This page describes what IRL records and what those records do and do not prove. Whether they are useful for your obligations depends on who you are and where you operate. Not legal advice.

---

## What IRL records

For every order it authorizes, IRL keeps:

- **The mandate verdict**, made before the order is sent: agent status, model hash, venue, asset and notional cap. The verdict is recorded with the trace but is not part of the sealed hash.
- **The sealed snapshot**: the agent's stated intent and context, hashed with SHA-256 over IRL canonical JSON (whitepaper §7.1) before execution. The snapshot includes the decision time (`valid_time`) and the record time (`txn_time`).
- **The binding to the exchange order id**: `final_proof = SHA-256(reasoning_hash ‖ exchange_tx_id)`, plus a `MATCHED` or `DIVERGENT` status reported by the engine. The status is not part of the sealed hash.
- **Inclusion in a Merkle root** that is anchored on Bitcoin through OpenTimestamps, in batches.

Some rejections leave no trace: an unknown or inactive agent, a model hash mismatch, or a venue or asset outside the mandate is refused before a trace is created. Mandate rejections from the policy check are recorded as `HALTED` on a best-effort basis.

In `SHADOW_MODE`, out-of-mandate orders are recorded but not blocked.

## What a verifier can check

Anyone with a bundle can check, with the open-source verifier and without trusting Norve's servers:

- The `final_proof` binding between the sealed hash and the exchange order id.
- The timestamps and the Merkle inclusion of each record.
- The Bitcoin anchor, with the standard OpenTimestamps client and public block data.

For `snapshot_version` 1 traces, recomputing the `reasoning_hash` itself requires the full sealed snapshot, which a public bundle does not include.

## What IRL does not prove

- That the agent's reasoning was correct, honest or profitable.
- Which model actually ran. The model hash is declared by the operator; IRL proves that the declared hash matched the registered one.
- That `valid_time` reflects when inference happened. The agent supplies it; IRL only enforces `valid_time < txn_time`.
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
