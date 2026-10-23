# Node Operators, Withdrawal Credentials and DVT

A liquid staking protocol does not run validators itself in any physical sense. Operators do. The protocol's job is to make sure operators can sign blocks and attestations but can never take the staked ETH, and that one operator's failure does not sink everyone. This chapter covers how operators are chosen, how withdrawal credentials bind validators to the protocol, and how distributed validator technology (DVT) spreads key risk.

## Withdrawal credentials

Each validator has two pieces of key material. The **signing key** (BLS) signs attestations and blocks; whoever holds it can get the validator slashed. The **withdrawal credentials**, fixed at the first deposit, decide where the ETH goes when it leaves the beacon chain.

| Prefix | Meaning | Notes |
|---|---|---|
| `0x00` | BLS withdrawal key | Original format; must be changed to `0x01` before withdrawals (one-time BLS-to-execution change) |
| `0x01` | Execution-layer address | Rewards above 32 ETH are swept automatically; full exits go to the address |
| `0x02` | Execution-layer address, compounding | Introduced with the Pectra upgrade (EIP-7251); allows effective balance up to 2048 ETH |

For a liquid staking protocol the withdrawal credentials must be `0x01` or `0x02` pointing at the protocol's withdrawal vault contract. Pectra also added EIP-7002, which lets the withdrawal-credential address trigger exits from the execution layer. That gives the protocol (or whatever contract holds the credential) a way to force-exit a misbehaving operator's validators without the operator's cooperation, and it also makes the credential contract more powerful, so its access control matters more.

## The deposit front-running problem

The beacon chain uses the withdrawal credentials from the **first** valid deposit for a given validator public key. Later deposits top up the balance but cannot change the credentials. If an operator submits a pubkey to the protocol, then front-runs the protocol's 32 ETH deposit with a 1 ETH deposit of its own using credentials it controls, the protocol's 32 ETH ends up in a validator the operator can withdraw.

Lido addressed this class of issue in 2021 with a deposit security module: a committee of guardians signs off that the deposit contract's root has not changed in a way that would include a front-running deposit, and deposits revert otherwise. Rocket Pool requires operators to pre-deposit part of the stake and checks credentials before assigning the rest. When you audit a new protocol, find its answer to this attack and read the code that enforces it.

```solidity
// Sketch of a deposit guard: refuse to deposit if the deposit root moved
function depositBuffered(bytes32 expectedRoot, bytes calldata guardianSigs) external {
    require(depositContract.get_deposit_root() == expectedRoot, "root changed");
    require(_quorumSigned(expectedRoot, guardianSigs), "guardians");
    _depositKeys(); // uses pre-registered pubkeys with protocol credentials
}
```

The guard's own security depends on guardian keys, quorum and liveness. If guardians stop signing, deposits stop: that is the intended fail-safe direction.

## Operator selection

| Model | How operators join | Main risk | Examples |
|---|---|---|---|
| Curated | DAO or company approves | Centralisation; governance capture | Lido curated module, cbETH |
| Bonded permissionless | Anyone with a bond (e.g. RPL or ETH) | Bond smaller than possible loss; Sybil operators | Rocket Pool minipools, Lido Community Staking Module |
| DVT clusters | Several operators share one validator | Cluster coordination bugs; correlated client software | Lido Simple DVT module using Obol and SSV |

Questions for any model:

1. What is the operator's bond, and does it cover the worst realistic slashing loss including correlation penalties?
2. Can the protocol force-exit an operator's validators (EIP-7002 or pre-signed exits)?
3. Is there a limit on validators per operator?
4. Who can add, remove or change an operator's reward address and key set?
5. Are operator public keys validated (length, duplicates, deposit signature) before being used?

## Distributed validator technology

DVT splits a validator's signing key into shares held by several operators, using threshold BLS signatures. A cluster of, say, four operators with threshold three can keep signing if one is offline, and no single operator can sign alone. Obol's Charon middleware and the SSV Network are the two widely used implementations.

DVT reduces single-operator key theft and downtime. It does not remove slashing risk from bugs in the coordination layer, nor from all cluster members running the same client with the same bug. Check client diversity within clusters, the distributed key generation (DKG) ceremony, and how keys are rotated when an operator leaves.

## Worked example

_Not executed in this book's CI; verify locally._

Inspect withdrawal credentials of protocol validators with a beacon node API on a public testnet or a local devnet.

1. Start a local devnet with Kurtosis' `ethereum-package` (it runs execution and consensus clients together) or use a public Holesky or Hoodi beacon API endpoint you trust.
2. Query a validator:

```bash
curl -s $BEACON/eth/v1/beacon/states/head/validators/<index> | jq '.data.validator.withdrawal_credentials'
```

3. Write a Python script that, for a list of validator indices, decodes the prefix and the address (last 20 bytes) and flags any validator whose credential is not `0x01`/`0x02` pointing at the expected vault:

```python
import requests, sys
BEACON, VAULT = sys.argv[1], sys.argv[2].lower()
for idx in sys.argv[3:]:
    wc = requests.get(f"{BEACON}/eth/v1/beacon/states/head/validators/{idx}").json()["data"]["validator"]["withdrawal_credentials"]
    prefix, addr = wc[:4], "0x" + wc[-40:]
    ok = prefix in ("0x01", "0x02") and addr == VAULT
    print(idx, prefix, addr, "OK" if ok else "MISMATCH")
```

4. On the devnet, make a deposit with wrong credentials and confirm the script flags it.

## Common pitfalls

- Trusting operator-submitted public keys without checking the deposit signature and duplicates.
- Assuming DVT removes slashing risk entirely.
- Sizing operator bonds against single-validator slashing, ignoring correlation penalties.
- Forgetting that the withdrawal-credential contract can now trigger exits, which makes it a high-value target.
- Letting the same governance key add operators and change withdrawal vault code with no delay.

## Exercises

1. Read Lido's `DepositSecurityModule` contract and write down exactly what guardians sign and what check stops a front-run deposit.
2. Simulate the front-running attack on a Kurtosis devnet: deposit 1 ETH with your credentials, then 32 ETH with "protocol" credentials, and show which credentials stick.
3. Compare the operator bond required by Rocket Pool and Lido CSM with the maximum slashing loss for one validator in a correlated event, using the consensus specs.

## Further reading

- EIP-7251 (increase max effective balance): <https://eips.ethereum.org/EIPS/eip-7251>
- EIP-7002 (execution layer triggerable exits): <https://eips.ethereum.org/EIPS/eip-7002>
- Ethereum consensus specs: <https://github.com/ethereum/consensus-specs>
- Obol documentation: <https://docs.obol.org>
- SSV Network documentation: <https://docs.ssv.network>
