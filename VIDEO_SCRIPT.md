# Task 4 — Video Narration Script (≤3 minutes)

> Speak this over a screen-share of `architecture_diagram.drawio` (or `ARCHITECTURE.md`).
> Face is NOT required — clear voice + the diagram on screen is exactly what they ask for.
> Pace: ~140 words/min → this script is ~430 words ≈ 2:50.

---

Hi, I'm Abhishek, and this is my submission for the Prerequisite Challenge.

The project is a Solana program called the Vault, built with the Anchor framework.
On Solana, a program is a smart contract — but it is stateless. It holds no data of its
own. All the data lives in separate accounts that get passed into each instruction.
That is the mental model the whole project is built on.

The Vault lets a user open a personal SOL vault, deposit funds into it, withdraw them,
and finally close the vault. There are four instructions: initialize, deposit, withdraw,
and close.

Let me walk through the accounts. The "user" is the signer wallet — it pays fees and
owns the SOL. The "vault_state" is a Program Derived Address, a PDA, with seeds "state"
plus the user. It simply stores two bump values. The "vault" is another PDA, with seeds
"vault" plus the vault_state; this is the account that actually holds the deposited SOL.

A PDA is the important idea here. It is an address derived deterministically from seeds
and the program ID, and it has no private key. Because there is no private key, only our
program can sign on its behalf — and that is what keeps the user's funds safe.

The part I actually built is the extension. The challenge asked me to modify the
"withdraw" instruction so that, when a user withdraws, the program also calls — through a
Cross-Program Invocation, a CPI — an external "registration" program. That registration
program records the user's GitHub username on-chain, inside an account it owns called
"application_account".

So the full flow is: a user initializes their vault, deposits SOL, then withdraws. On
withdrawal, the vault program returns the SOL to the user AND makes a CPI into the
registration program, passing my GitHub username, which creates the application account.
Finally, close returns any leftover SOL and closes the vault state.

I deployed this to devnet and verified it: the registration account for my wallet now
contains my GitHub handle, which proves the CPI really happened.

That is the Vault program — a small but complete example of accounts, PDAs, and program
composition on Solana. Thank you.
