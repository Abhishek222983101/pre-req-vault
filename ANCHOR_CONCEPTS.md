# Solana & Anchor Concepts — In Detail

This explains every concept the challenge assumes, at a depth useful for a technical reader.

## 1. Program (Smart Contract)
On Solana, a "program" is the on-chain code — the equivalent of a smart contract on Ethereum.
Programs are **stateless**: they contain no mutable storage of their own. All data they read or
write lives in separate **accounts**. A program is identified by a Program ID (a Solana
address / public key). Our program `pre_req_vault` has ID
`4MPh2ZymUjEd6XZqYFNemJF1cBSdUBuNNcnjbTkwRCZx`.

Key idea: a program is just an executable blob; it only does something when someone sends it an
**instruction** along with the **accounts** that instruction will touch.

## 2. Account
An account is a mutable 128-byte-header + arbitrary data blob on Solana, each with its own
address. Accounts can hold:
- **Lamports (SOL)** — every account has a balance; this is how SOL is stored.
- **Program data / state** — e.g., our `vault_state` account stores two bump bytes.
- **Executable code** — a program itself is stored in an executable account.

Accounts are passed into instructions explicitly; the program checks their addresses and owners.
"Owner" matters: only the owning program can modify an account's data. User wallets own their
own accounts; the System Program owns newly created system accounts; a program owns the PDAs
it derives.

## 3. Instruction
An instruction is a single callable action in a program — like a function. Ours:
`initialize`, `deposit`, `withdraw`, `close`. An instruction call specifies:
- the program to call,
- the accounts it needs (with read/write/signer flags),
- any arguments (e.g., `amount: u64`, `github: string`).

Transactions bundle one or more instructions and are signed by the required signers.

## 4. Anchor
Anchor is a Rust framework that removes boilerplate from Solana development. It provides:
- `#[program]` to declare instruction handlers,
- `#[derive(Accounts)]` structs that **automatically validate** accounts (seeds, bumps, mutability,
  ownership) before the handler runs,
- `declare_id!` for the program address,
- `declare_program!` to import another program's IDL as a typed client,
- code generation of an IDL and TypeScript types.

Without Anchor you would hand-write account deserialization, manual seeds checks, and raw CPI
calls. Anchor makes the `withdraw` CPI we added only a few lines.

## 5. PDA — Program Derived Address
A PDA is an address **deterministically derived** from:
- a set of **seeds** (byte strings, e.g., `b"vault"` + `vault_state.key()`), and
- a **program ID** (the program that "owns" the PDA).

Crucially, a PDA is **off the ed25519 curve by construction**, so it has **no private key** —
no one can ever produce a signature for it. That means the *only* way to sign for a PDA is for
the owning program to do it internally (via `CpiContext::new_with_signer` with signer seeds).

In our program:
- `vault_state` PDA: seeds `[b"state", user]`.
- `vault` PDA: seeds `[b"vault", vault_state.key()]`.
- `application_account` PDA: seeds `[b"prereqs", user]`, but its owning program is the
  **registration** program (`seeds::program = application_program.key()`), so registration — not
  our vault — owns and initializes it.

PDAs let programs "own" addresses and accounts without needing a keypair.

## 6. Bump
Because a PDA must be off-curve, the derivation tries the seeds with an extra single byte called
the **bump** (0–255). It searches downward from 255 for the first value that pushes the address
off-curve; that value is the canonical bump. The bump is stored (our `vault_state` keeps
`vault_bump` and `state_bump`) so the program can later recreate the exact signer seeds:
`[b"vault", vault_state.key(), vault_bump]`. Anchor's `#[account(bump)]` can also store/verify it
automatically.

## 7. CPI — Cross-Program Invocation
A CPI is when one program calls another program's instruction **from inside its own instruction**.
That is exactly what `withdraw` does: after moving SOL, it calls
`registration.initialize(github)` via:
```rust
let cpi_ctx = CpiContext::new(self.application_program.key(), cpi_accounts);
initialize(cpi_ctx, crate::GITHUB_USERNAME.to_string())?;
```
CPIs run **in the same transaction**, so either the whole thing succeeds or it all reverts.
Importantly, when program A CPIs into program B, B still sees A as the caller; B's account
constraints (e.g., "this PDA must be signed by B") are enforced normally. CPIs are how Solana
programs compose — our vault program reuses the registration program instead of re-implementing
identity storage.

## 8. IDL — Interface Definition Language
An IDL is a JSON description of a program's public interface: its instructions, account
structures, arguments, and error codes. It is the Solana analogue of an Ethereum ABI. The
`registration.json` IDL tells Anchor:
- the registration program's address,
- that it has an `initialize(github: string)` instruction,
- the accounts that instruction needs (`user`, `account`, `system_program`),
- that `account` is a PDA with seeds `[b"prereqs", user]`.

`declare_program!(registration);` reads this JSON and generates a **typed Rust client**
(`registration::cpi::initialize`, `registration::program::Q3PreReqsRs`), so we can call it
safely from Rust. IDLs also generate TypeScript clients for tests.

## 9. Supporting concepts
- **Lamports & SOL** — 1 SOL = 1,000,000,000 lamports. `deposit`/`withdraw`/`close` move lamports.
- **Signer** — an account that signed the transaction (`user`); required to authorize spending.
- **System Program** — Solana's built-in program for creating accounts and transferring SOL.
  Our transfers use `system_program::transfer`.
- **Rent** — accounts must hold enough lamports to be rent-exempt; the registration account's
  creation is funded by `user` via `system_program`.
- **Devnet** — a Solana test network with free airdropped SOL; we deploy and test there.
- **declare_id! / keys sync** — the program's on-chain address must match its deploy keypair;
  `anchor keys sync` rewrites `declare_id!` to the generated keypair after `anchor build`.

## 10. End-to-end summary
A user calls `initialize` (creates `vault_state`), `deposit` (funds `vault`), then `withdraw`.
`withdraw` returns SOL to the user **and** CPIs into the registration program, which writes the
user's GitHub username into `application_account`. A grader verifies that PDA now contains the
expected username — proving the vault program was correctly extended with a CPI.
