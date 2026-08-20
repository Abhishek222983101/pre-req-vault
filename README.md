# Prerequisite Vault — Builders Cohort (Q3 2026)

Submission for the Turbin3 / Builders Cohort **Prerequisite Challenge**.

A Solana program (built with the [Anchor](https://www.anchor-lang.com) framework) that lets a
user open a personal SOL vault, deposit/withdraw, and — on `withdraw` — performs a
**Cross-Program Invocation (CPI)** into an external registration program to record the cadet's
GitHub username on-chain.

▶️ **Explainer video:** https://youtu.be/ZnfnucHY44A
📐 **Architecture diagram:** `architecture_diagram.drawio` (and `ARCHITECTURE.md`)

## Deployed Program

| Item | Value |
|---|---|
| Our Vault program ID | `4MPh2ZymUjEd6XZqYFNemJF1cBSdUBuNNcnjbTkwRCZx` |
| Registration program ID | `TRBZyQHB3m68FGeVsqTK39Wm4xejadjVhP5MAZaKWDM` |
| Cluster | Solana **devnet** |
| Deployer wallet | `HuHZuySqKNeHcBCUajcgBvdYdjWjtubJFA3iJr1j1c3P` |
| Recorded GitHub handle | `Abhishek222983101` |

## Architecture

Solana programs are **stateless** — all state lives in **accounts**. This program uses PDAs
(Program Derived Addresses): addresses derived from seeds + the program ID that have **no
private key**, so only the owning program can sign for them.

### Accounts
- **user** — signer wallet; pays fees and owns the deposited SOL.
- **vault_state** (PDA) — seeds `[b"state", user]`; stores `vault_bump` + `state_bump`.
- **vault** (PDA) — seeds `[b"vault", vault_state]`; holds the deposited lamports.
- **application_account** (PDA) — seeds `[b"prereqs", user]`; **owned by the registration
  program**; stores the GitHub username.
- **application_program** — the external `registration` program (`TRBZy...`).
- **system_program** — used for SOL transfers.

### Instructions
1. `initialize` — creates `vault_state`, records bumps.
2. `deposit(amount)` — transfers SOL `user → vault`.
3. `withdraw(amount)` — transfers SOL `vault → user` (vault PDA signs) **and** CPIs into
   `registration.initialize(github)`, creating `application_account`.
4. `close` — returns remaining SOL `vault → user` and closes `vault_state`.

### The CPI extension (the task)
`withdraw` now calls `registration::cpi::initialize` with the cadet's GitHub username. This is
why, after a successful withdrawal, the registration program holds the cadet's details —
proving the vault program was extended as required (and not by calling the registration program
directly from a client).

## On-chain verification

The registration PDA for the deployer wallet:
`B9ivbA7kYwHz7TPKJLj1K6c9G5nww1VGNvBc3RC2Qgk6`
contains the string **`Abhishek222983101`** — confirming the CPI recorded the username.

## What was done (challenge tasks)
- **Task 1** — Understood the Vault codebase (instructions, accounts, state flow).
- **Task 2** — Extended `withdraw` with the registration CPI; deployed to devnet; `anchor test`
  passes 4/4; verified on-chain.
- **Task 3** — Architecture diagram (`architecture_diagram.drawio` + `ARCHITECTURE.md`).
- **Task 4** — ≤3-min explainer video (linked above) with captions.

## Project layout
```
programs/pre-req-vault/src/   # Anchor program (lib.rs, instructions/, state.rs, constants.rs)
idls/registration.json         # IDL of the external registration program
tests/pre-req-vault.ts         # devnet test (initialize -> deposit -> withdraw+CPI -> close)
architecture_diagram.drawio    # Task 3 diagram
ARCHITECTURE.md                # diagram + written explanation
ANCHOR_CONCEPTS.md             # Solana/Anchor concepts reference
PROJECT_OVERVIEW.md            # project overview
```

## Build & verify (reference)
```bash
pnpm install
anchor build
anchor keys sync
anchor deploy --provider.cluster devnet
# set ANCHOR_PROVIDER_URL + ANCHOR_WALLET, then:
pnpm exec ts-mocha -p ./tsconfig.json -t 1000000 "tests/pre-req-vault.ts"
```

## Submission
- Public repo: https://github.com/Abhishek222983101/pre-req-vault
- Video: https://youtu.be/ZnfnucHY44A
- Form: https://forms.gle/zGPY8svmPdMQg3rG9
