### FeeSplitter + Pluggable Revenue Share: Design Doc

|                    |                      |
| ------------------ | -------------------- |
| Author             | _0xDiscotech_        |
| Created at         | _2025-08-21_         |
| Initial Reviewers  | _Tynes, Agus, Joxes_ |
| Need Approval From | _Tynes, Agus, Joxes_ |
| Status             | _Draft_              |

## Purpose

Make fee revenue sharing dynamic and per‑chain extensible. Allow any chain to plug in their own revenue share logic without forking the `FeeSplitter` but integrating with it.

## Summary

`FeeSplitter` aggregates fees from all the `FeeVault`s on L2, enforces the configured checks, and then disburses funds to recipients based on shares computed by a pluggable `SharesCalculator`. The calculator returns an array of disbursements—each with a `recipient`, `withdrawalNetwork`, `amount`, and optional `metadata`—and `FeeSplitter` iterates through this list to execute each payout as an L2 transfer or an L1 withdrawal. We ship with a `StandardSuperchainRevenueShare` calculator by default, and chains can swap in their own module as needed. Each disbursement emits a single aggregate event containing arrays of `recipients`, `networks`, and `amounts`.

## Problem Statement + Context

Fixed fee shares are too rigid: chains want custom splits, L1/L2 destinations, and richer policies. We need:

- A safe, permissioned way to plug in chain‑specific logic.
- A stable, minimal surface in `FeeSplitter` to keep operations simple.
- Backwards‑compatible behavior by default (standard superchain revshare).

## Proposed Solution

High‑level flow:

1. `FeeSplitter.disburseFees()` checks interval and vault config, pulls eligible funds from vaults, and computes both `gross` and `net` amounts (and per‑vault totals for future granularity).
2. `FeeSplitter` calls the chain‑configured `SharesCalculator` with the inputs to compute disbursements.
3. `FeeSplitter` validates outputs (basic invariants) and then pays each item:
   - L2: native transfer
   - L1: use `L2ToL1MessagePasser` to withdraw
4. Emit one aggregate event (arrays). Update internal accounting.

Interfaces and ownership:

- `SharesCalculator` naming: use `IRevenueShareCalculator` (short: `RevenueShareCalculator`). Method: `computeDisbursements(gross, net[, perVault])`.
- Admin (`ProxyAdmin.owner()`) can set the calculator via `setShareCalculator(address)` and update the disbursement interval.
- The default calculator is `StandardSuperchainRevenueShare` (our previous 2‑recipient, gross/net max behavior).

Inputs to the calculator:

- `gross` (sum of all vault withdrawals this call).
- `net` (sum of Sequencer/Base this call).
- Optionally per‑vault amounts (to enable more complex policies). We’ll pass them; calculators can ignore.

Outputs from the calculator:

- `Disbursement[]` where each item is `{recipient, withdrawalNetwork, amount}`. We also provision a `metadata` bytes field slot; see below.

Metadata and recipients:

- We’ll support an optional `metadata` bytes blob per disbursement. As agreed, we’ll `TSTORE` the metadata on `FeeSplitter` right before the corresponding send, so recipients that rely on it can read it in their payable fallback/receive code path. We keep `SafeCall.send()` to stay compatible with EOAs and multisigs (no new interface requirement).

Invariants and validation:

- Sum of `amounts` MUST equal total collected (rounding policy below).
- `recipients` MUST be non‑zero.
- If any payout fails (recipient reverts or bridge call fails), revert the whole transaction.
- If the calculator returns zero total, short‑circuit (no interval consumed).
- Misconfigured vault (wrong network or recipient) reverts (consistent with current behavior).

Rounding:

- Any last‑wei rounding delta goes to the last `recipient` returned by the calculator.

Events:

- One aggregate event on each disbursement with arrays:
  - `recipients[]`, `networks[]`, `amounts[]`, `total`, `calculator` address.

Extensibility:

- Start with `StandardSuperchainRevenueShare` to preserve current semantics.
- RAAS providers can deploy their own calculators and the chain can point to them via `setShareCalculator`.

### Invariants

- All fee vaults MUST be correctly configured (`WithdrawalNetwork = L2` and `RECIPIENT = FeeSplitter`) or `disburseFees` MUST revert.
- `SharesCalculator` output MUST be valid: `sum(amounts) == totalCollected`, each `recipient != address(0)`, and each `withdrawalNetwork` is supported; otherwise `disburseFees` MUST revert.
- Disbursement MUST be atomic: if any individual payout fails (L2 transfer or L1 withdrawal), `disburseFees` MUST revert the entire transaction.
- If no funds are collected for the call, `disburseFees` MUST return early and MUST NOT consume the disbursement interval.
- Cross‑function reentrancy MUST be prevented: `disburseFees` is non‑reentrant and `receive` MUST revert while a disbursement is in progress.

### Resource Usage

- `disburseFees` does:
  - Up to four vault withdrawals (each with eligibility checks).
  - One external call to the calculator (view/pure).
  - Iterates through the returned `disbursements` and performs transfers/bridge calls.
  - Emits one aggregate event.
- Compared to “vault → L1 direct,” this adds an extra step at disbursement time (slightly higher gas). Acceptable on L2.
- Admin setters (`setShareCalculator`, `setFeeDisbursementInterval`) are trivial (single storage write + event).

### Single Point of Failure and Multi Client Considerations

- No client (op‑geth/op‑reth) changes needed.
- Opt‑in and per‑chain selectable calculator. The `FeeSplitter` surface remains stable.

## Failure Mode Analysis

- DoS on a very big recipients array returned by the calculator, leading to a large number of disbursements and OOG
- DoS due to very high gas consumption by a recipient ending in OOG
- Misconfigured recipients that don't allow receiving the fees from the splitter, leading to a failed disbursement.
- Buggy SharesCalculator that returns incorrect or malformed outputs, leading to a failed disbursement.
-

## Impact on Developer Experience

- Chains get a clean plug‑in point to customize revshare without forking `FeeSplitter`.
- Default behavior matches the current two‑recipient superchain revshare until a chain swaps calculators.
- Recipients don’t need to implement a new interface; we continue using `SafeCall.send()`. Advanced recipients can optionally read metadata via `TSTORE` if they want more context.

## Alternatives Considered

- Keep fixed two‑recipient split
  - Simpler, but not flexible; each chain ends up forking or wrapping.
- Require a recipient interface and pass metadata via calldata
  - Breaks EOAs/multisigs and adds constraints on recipients; not ideal for compatibility.
- Emit per‑recipient events instead of an aggregate event
  - Noisy and more expensive; we prefer a single aggregate event with arrays.

## Risks & Uncertainties

- `TSTORE` availability/assumptions: we rely on transient storage being available on the target L2; if disabled, recipients can’t read metadata that way (fallback still works via plain ETH transfer).
- DoS via many recipients: we currently don’t cap outputs or enforce min payout amounts. Low risk (calculator is set by the permissioned chain op and can be updated quickly), but we’ll keep it in mind. If needed, we can add `MAX_PAYOUTS_PER_TX` and `MIN_PAYOUT_AMOUNT` as admin‑settable limits.
- Calculator bugs: if a calculator misbehaves (sum mismatch or downstream recipients revert), disbursements will revert. We’ll validate outputs and keep calculator swaps gated by `ProxyAdmin.owner()`.

Open items we decided

- `withdrawalNetwork` is part of the calculator output (per‑recipient L2/L1 destination).
- `metadata`: a bytes metadata field is supported; `FeeSplitter` `TSTORE`s it before each send so recipients that want it can read it. We keep `SafeCall.send()` to avoid breaking EOAs/multisigs.
- Inputs to calculator: we pass `gross`, `net`, and `perVault` totals (calculator can ignore what it doesn’t need).
- Failure policy: revert the whole disbursement if any payout fails.
- Rounding: last wei goes to the last `recipient`.
- Events: single aggregate event with arrays per disbursement.
- Gas forwarded: no special limit initially (set by chain op, can be updated quickly if needed). We can add a cap later if we see an issue.
