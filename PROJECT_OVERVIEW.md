# Vault Program — Project Overview & Solana/Anchor Concepts

> Use the **"Brief Architecture Overview"** block (Section 1) as a seed prompt for Gemini to
> generate a long-form project overview. Sections 2–3 are the detailed reference.

---

## 1. Brief Architecture Overview (paste into Gemini)

```
Project: "Prerequisite Vault" — an Anchor (Solana) program for a builders-cohort entrance challenge.
A user opens a personal SOL vault, deposits and withdraws, and on withdrawal the program
performs a Cross-Program Invocation (CPI) into an external "registration" program that records
the user's GitHub username on-chain.

Key pieces:
- Program: pre_req_vault (deployed on devnet at 4MPh2ZymUjEd6XZqYFNemJF1cBSdUBuNNcnjbTkwRCZx).
- External program: registration / Q3PreReqsRs (TRBZyQHB3m68FGeVsqTK39Wm4xejadjVhP5MAZaKWDM),
  with instruction initialize(github: string).
- Accounts: user (signer), vault_state PDA [state, user], vault PDA [vault, vault_state],
  application_account PDA [prereqs, user] (owned by registration program), system_program.
- Instructions: initialize, deposit, withdraw, close.
- withdraw transfers SOL from the vault back to the user (vault PDA signs) AND calls
  registration.initialize(github) via CPI, creating application_account.
- State flow: initialize -> deposit -> withdraw(+CPI registers GitHub) -> close.
- IDL-driven: the registration interface is consumed from registration.json.

Ask Gemini to expand this into a detailed project overview (purpose, architecture, data flow,
security model, and how the CPI extension satisfies the challenge) for a technical reader.
```

---

## 2. Detailed Project Overview

### Purpose
The Vault program is a minimal but realistic Solana smart contract built with the Anchor
framework. It lets a user create a personal vault (a Program Derived Address that holds SOL),
deposit SOL into it, withdraw SOL back, and finally close the vault. The entrance challenge
extends the `withdraw` instruction so that, in the same transaction, it also calls a separate
"registration" program — proving the cadet can read an unfamiliar codebase, set up a toolchain,
extend a program with a CPI, deploy it, and verify on-chain state.

### Architecture
- **Stateless program, stateful accounts.** Program code holds no data. All state lives in
  Solana *accounts* passed into each instruction. This is the core Solana execution model.
- **Two PDAs per user.** `vault_state` (seeds `["state", user]`) stores two bump bytes.
  `vault` (seeds `["vault", vault_state]`) is a plain system account holding deposited lamports.
  Deriving `vault` from `vault_state` (not `user` directly) lets the program sign for `vault`
  using the stored `vault_bump`.
- **External registration program.** A second, already-deployed program (`registration`) owns an
  `application_account` PDA (seeds `["prereqs", user]`) recording the cadet's GitHub username.
  Our vault program never touches that account's data directly; it only *calls* registration and
  lets it initialize the account.
- **IDL-driven client.** The registration program's interface is described by `registration.json`
  (an IDL). Anchor's `declare_program!` macro turns that JSON into a typed Rust CPI client.

### Instruction-by-instruction data flow
1. **initialize(user, vault_state, vault, system_program)** — creates `vault_state` and records
   `vault_bump` + `state_bump`.
2. **deposit(user, vault_state, vault, system_program, amount)** — transfers `amount` lamports
   from `user` into `vault`.
3. **withdraw(user, vault_state, vault, application_account, application_program, system_program, amount)**
   — transfers `amount` lamports from `vault` back to `user` (the `vault` PDA signs with
   `vault_bump`), then performs a **CPI** to `registration.initialize(github)`, which creates
   `application_account` storing the GitHub handle.
4. **close(user, vault_state, vault, system_program)** — returns all remaining lamports from
   `vault` to `user` and closes `vault_state`.

### Security model
- PDAs have **no private key**; only the owning program can sign for them, so user funds in
  `vault` can only move via the program's logic.
- Account constraints (`seeds`/`bump`) are enforced by Anchor/ Solana, preventing substitution.
- The CPI targets a **fixed** registration program address (from the IDL), so the verification
  cannot be faked by calling a different program.

### How the CPI extension satisfies the challenge
The challenge required extending `withdraw` to record the cadet on-chain without calling the
registration program directly from a client. By invoking `registration.initialize` *inside* the
vault program's `withdraw`, the recording is provably done by our deployed program — verifiable
on-chain (the `application_account` PDA contains the GitHub username).
