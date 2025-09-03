# FeeSplitter + Pluggable Revenue Shares Calculator

|                    |                      |
| ------------------ | -------------------- |
| Author             | _0xDiscotech_        |
| Created at         | _2025-08-21_         |
| Initial Reviewers  | _Tynes, Agus, Joxes_ |
| Need Approval From | _Tynes, Agus, Joxes_ |
| Status             | _Draft_              |

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Purpose](#purpose)
- [Summary](#summary)
- [Problem Statement + Context](#problem-statement--context)
- [Proposed Solution](#proposed-solution)
  - [Invariants](#invariants)
  - [Resource Usage](#resource-usage)
  - [Single Point of Failure and Multi Client Considerations](#single-point-of-failure-and-multi-client-considerations)
- [Failure Mode Analysis](#failure-mode-analysis)
- [Impact on Developer Experience](#impact-on-developer-experience)
- [Alternatives Considered](#alternatives-considered)
- [Risks & Uncertainties](#risks--uncertainties)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Purpose

Make fee revenue sharing dynamic and per‑chain extensible. Allow any chain to plug in their own revenue share
logic without forking the `FeeSplitter` but integrating with it.

## Summary

`FeeSplitter` aggregates fees from all `FeeVault`s on L2, enforces the configured checks, and then disburses
funds to recipients based on shares computed by a pluggable `SharesCalculator`. The calculator returns an
array of disbursements—each containing a `recipient` and `amount`—and `FeeSplitter` iterates through this list
to execute each payout as an L2 transfer. We ship with a
[`SuperchainRevSharesCalculator`](./superchain-revshares-calc.md) by default, and chains can swap
in their own module as needed.

It's possible that chain operators may desire the option to withdraw part of the fees to L1.
This can be achieved using the default [`L1Withdrawer`](./l1-withdrawer.md), but the modular
architecture allows operators to use any other solution that better suits their needs. Each disbursement emits
a single aggregate event containing arrays of `recipients` and `amounts`, alongside the `grossRevenue`.

## Problem Statement + Context

Fixed fee shares are too rigid: chains want custom splits, L1/L2 destinations, and richer policies. We need:

- A safe, permissioned way to plug in chain‑specific logic.
- A stable, minimal surface in `FeeSplitter` to keep operations simple.

## Proposed Solution

High‑level flow:

1. `FeeSplitter.disburseFees()` checks the interval and vault config, pulls eligible funds from vaults,
   and computes the gross and per‑vault revenue for granularity.
2. `FeeSplitter` calls the chain‑configured `SharesCalculator` with the inputs to compute disbursements.
3. `FeeSplitter` validates outputs (basic invariants) and then transfers the corresponding amount to each
   recipient.
4. Emits one aggregate event (arrays). Updates the `lastDisbursementTime`.

Interfaces and ownership:

- `SharesCalculator` naming: use `ISharesCalculator`.
  Method: `getRecipientsAndValues(per vault fee revenue)`.
- Admin (`ProxyAdmin.owner()`) can set the calculator via `setSharesCalculator(address)` and update the disbursement interval.
- The default calculator is `SuperchainRevSharesCalculator` (our previous 2‑recipient, gross/net max behavior).

Inputs to the calculator:

- Per‑vault amounts (to enable complex policies).

Outputs from the calculator:

- We keep `SafeCall.send()` to stay compatible with EOAs and multisigs (no new interface requirement).
- `ShareInfo[]` where each item contains both a `recipient` to receive the fees and an `amount` for the
  corresponding share of the fees.

Invariants and validation:

- Sum of `amounts` MUST equal total collected (rounding policy below).
- `recipients` MUST be non‑zero.
- If any payout fails, revert the whole transaction.
- If the calculator returns zero total, short‑circuit (no interval consumed).
- Misconfigured vault (wrong network or recipient) reverts (consistent with current behavior).

Events:

- One single aggregated `FeesDisbursed` event on each disbursement with `grossRevenue` and arrays:
  - `recipients[]`, `amounts[]`.

Extensibility:

- Start with `SuperchainRevSharesCalculator` to preserve current semantics.
- RAAS providers can deploy their own calculators and the chain can point to them via `setSharesCalculator`.

### Invariants

- All fee vaults MUST be correctly configured (`WithdrawalNetwork = L2` and `RECIPIENT = FeeSplitter`) or
  `disburseFees` MUST revert.
- `SharesCalculator` output MUST be valid: `sum(amounts) == totalCollected` and each `recipient != address(0)`;
  otherwise `disburseFees` MUST revert.
- Disbursement MUST be atomic: if any individual payout fails, `disburseFees` MUST revert the entire transaction.
- If no funds are collected for the call, `disburseFees` MUST revert.
- Cross‑function reentrancy MUST be prevented: `disburseFees` is non‑reentrant enforced by a `TSTORE` flag
  with a check, enabling receiving funds if and only if the disburse fees process has been initiated.

### Resource Usage

- `disburseFees` does:
  - Up to four vault withdrawals (each with eligibility checks).
  - One external call to the calculator (view/pure).
  - Iterates through the returned `disbursements` and performs transfers.
  - Emits one aggregate event.
- Compared to "vault → L1 direct," this adds an extra step at disbursement time (slightly higher gas).
  This is acceptable on L2.
- Admin setters (`setShareCalculator`, `setFeeDisbursementInterval`) are trivial (single storage write + event).

### Single Point of Failure and Multi Client Considerations

- No client (op‑geth/op‑reth) changes needed.
- Opt‑in and per‑chain selectable calculator. The `FeeSplitter` surface remains stable.

## Failure Mode Analysis

- DoS on a very big recipients array returned by the calculator, leading to a large number of disbursements and OOG
- DoS due to very high gas consumption by a recipient ending in OOG
- Misconfigured recipients that don't allow receiving the fees from the splitter, leading to a failed disbursement.
- Buggy SharesCalculator that returns incorrect or malformed outputs, leading to a failed disbursement.

## Impact on Developer Experience

- Chains get a clean plug‑in point to customize revshare without forking `FeeSplitter`.
- Default behavior matches the current two‑recipient superchain revshare until a chain swaps calculators.
- Recipients don’t need to implement a new interface; we continue using `SafeCall.send()`.

## Alternatives Considered

- Keep fixed two‑recipient split
  - Simpler, but not flexible; each chain ends up forking or wrapping.
- Require a recipient interface and pass metadata via calldata
  - Breaks EOAs/multisigs and adds constraints on recipients; not ideal for compatibility.
- Emit per‑recipient events instead of an aggregate event
  - Noisy and more expensive; we prefer a single aggregate event with arrays.

## Risks & Uncertainties

- DoS via many recipients: we currently don’t cap outputs or enforce min payout amounts.
  Low risk (calculator is set by the permissioned chain op and can be updated quickly), but we’ll keep it in
  mind. If needed, we can add `MAX_PAYOUTS_PER_TX` and `MIN_PAYOUT_AMOUNT` as admin‑settable limits.
- Calculator bugs: if a calculator misbehaves (sum mismatch or downstream recipients revert),
  disbursements will revert. We’ll validate outputs and keep calculator swaps gated by `ProxyAdmin.owner()`.

Open items we decided

- Failure policy: revert the whole disbursement if any payout fails.
- Events: single aggregate event with arrays per disbursement.
- Gas forwarded: no special limit initially (set by chain op, can be updated quickly if needed).
  We can add a cap later if we see an issue.
