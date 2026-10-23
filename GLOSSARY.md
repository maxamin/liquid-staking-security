# Glossary

- **LST (liquid staking token):** a transferable token representing staked ETH plus accrued rewards.
- **Rebasing token:** a token whose `balanceOf` changes without transfers as rewards are reported.
- **Reward-bearing token:** a token with fixed balances whose redemption rate rises over time.
- **Withdrawal credentials:** the beacon chain field that decides where a validator's ETH goes on withdrawal (`0x01`, `0x02` point to an execution address).
- **Deposit front-running:** pre-registering a validator pubkey with attacker credentials so a later protocol deposit is captured.
- **DVT (distributed validator technology):** splitting a validator key among operators using threshold signatures.
- **Oracle report:** an on-chain submission of beacon chain balances and exits that updates LST accounting.
- **Sanity checker:** contract logic that rejects implausible oracle reports.
- **Bunker mode:** a Lido mode that changes withdrawal finalisation while significant losses may be pending.
- **Withdrawal queue:** the contract that orders and settles redemption requests.
- **Churn limit:** the beacon chain cap on how many validators (or how much stake) can enter or exit per epoch.
- **Depeg:** market price of the LST moving below its redemption value.
