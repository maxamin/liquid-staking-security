# Liquid Staking Audit Checklist

This chapter collects the earlier material into a checklist. Use the first sections for auditing a liquid staking protocol, and the last section for auditing a protocol that integrates an LST. Each item names how to verify it, so a finding is a reproducible artefact, not an opinion.

## Scoping questions

1. Rebasing or reward-bearing? Is there a wrapper, and which version do integrators use?
2. Which contracts hold ETH: pool, buffer, withdrawal vault, execution-layer rewards vault, queue?
3. What are the withdrawal credentials of protocol validators, and which contract do they point to?
4. Who reports beacon state: committee, proofs, or both? What is the quorum?
5. How are operators admitted, bonded, and removed?
6. Which roles can upgrade, pause, or change parameters, and with what delay?

## Deposits and operators

| # | Check | How to verify |
|---|---|---|
| D1 | Deposit front-running is prevented (guardian root check, pre-deposit, or equivalent) | Read code; devnet reproduction from chapter 2 |
| D2 | Operator keys validated: length, duplicates, deposit signature | Unit tests submitting malformed and duplicate keys |
| D3 | Withdrawal credentials of all validators point to the expected vault | Beacon API script from chapter 2 |
| D4 | Operator bond covers realistic correlated slashing | Calculation using consensus spec penalties |
| D5 | Protocol can force exits (EIP-7002 or pre-signed exits) and access to that path is restricted | Read code; test unauthorised calls revert |
| D6 | Per-operator validator limits enforced | Fuzz operator key submissions |

## Oracle and accounting

| # | Check | How to verify |
|---|---|---|
| A1 | Quorum is a majority of independent members; membership changes delayed | `cast call` member list and quorum on a fork |
| A2 | Reports for old or duplicate reference slots rejected | Unit test |
| A3 | Rate increase per report bounded (APR cap) | Fuzz test from chapter 3 |
| A4 | Rate decrease bound does not block real slashing reporting; special mode exists | Scenario test with mass slashing |
| A5 | No double counting between beacon balance and withdrawal vault | Invariant: pooled ETH equals buffer + CL balance + vault − locked for queue |
| A6 | Share rounding favours the protocol, never the user | Fuzz `submit` / transfer / redeem round trips |
| A7 | Proof-based accounting (if used) handles beacon state layout changes across forks | Review generalised index constants per fork |

## Withdrawals

| # | Check | How to verify |
|---|---|---|
| W1 | Payout is the minimum of request-time and finalisation rate | Invariant test from chapter 4 |
| W2 | Requests finalised strictly in order; batch boundaries correct | Fuzz with random batch sizes |
| W3 | Claims cannot be repeated; NFT transfer moves the claim | Unit test |
| W4 | User-provided hints validated | Test with wrong hints |
| W5 | Reentrancy safe around ETH sends and ERC-721 callbacks | Slither reentrancy detectors; manual review |
| W6 | Queue liveness without admin action when funds exist | Invariant: finalisable requests eventually finalise in the handler model |

## Governance and operations

| # | Check | Evidence |
|---|---|---|
| G1 | Upgrades behind a timelock with public monitoring | On-chain timelock config |
| G2 | Pause roles separated from upgrade roles | Role table |
| G3 | Emergency runbook covering oracle stall, mass slashing, key compromise | Runbook and drill record |
| G4 | Bug bounty and safe harbor in place | Immunefi listing or SEAL safe harbor adoption |

## Integrator checklist

| # | Check | How to verify |
|---|---|---|
| I1 | Price method chosen for the integration's role (chapter 5) | Read oracle adapter |
| I2 | Rate growth capped if exchange rate is used | Adapter code and fuzz test |
| I3 | Rebasing balances not cached; or wrapper used | Grep for stored balances |
| I4 | No strict equality on received amounts | Grep `==` on transfer results |
| I5 | Behaviour defined if LST pauses or depegs 5–10% | Fork simulation with price shocks |
| I6 | Bridged or restaked variants treated as separate risk | Asset config review |

## Using AI assistants

An LLM can help enumerate entry points, draft handler-based invariant tests, or summarise long oracle specs. The model's statements are hypotheses. A check passes or fails only on tool output: a Foundry test result, a Slither finding you reviewed, or a beacon API query. If a model says "reports cannot be replayed", write the test that tries. See **ai-assisted-audit-playbook**.

## Worked example

_Not executed in this book's CI; verify locally._

Turn A3, A5 and W1 into one invariant suite for a toy LST.

1. Reuse the toy LST, oracle and queue from chapters 3 and 4 in one Foundry project.
2. Write a handler with actions: `submit`, `transfer`, `report(deltaBps, dt)`, `requestWithdrawal`, `finalise`, `claim`. Bound `deltaBps` to [-500, +50] per report.
3. Track ghost variables: total ETH in, total ETH out, rate at each request.
4. Invariants: `ETH in − ETH out == pool balance + locked` (A5); no claim exceeds `shares × rateAtRequest` (W1); rate growth per report within cap (A3).
5. Run `forge test --match-contract LSTInvariants` with `runs = 500` and `depth = 100` in `foundry.toml`. Seed a bug in each component in turn and confirm a failing sequence is reported.
6. Save the suite; it becomes the regression harness for future changes.

## Common pitfalls

- Auditing the pool contract while treating the oracle and guardians as out of scope.
- Using pre-Pectra exit and credential assumptions.
- Skipping governance roles because "the DAO is trusted".
- Reporting model-generated findings without a reproducing test.
- Checking the LST contract but not the wrapper that integrators actually use.

## Exercises

1. Run the integrator checklist against one lending market's LST configuration and record evidence for each row.
2. Extend the worked example invariants with D6 by adding operators and per-operator caps.
3. Write a one-page runbook for "oracle quorum submits an inflated report", including who pauses what and in which order.

## Further reading

- Lido documentation: <https://docs.lido.fi>
- Rocket Pool documentation: <https://docs.rocketpool.net>
- Foundry invariant testing: <https://book.getfoundry.sh/forge/invariant-testing>
- Slither: <https://github.com/crytic/slither>
- Security Alliance safe harbor: <https://securityalliance.org>
