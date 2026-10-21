# Pre-req Vault — with registration CPI

Anchor vault program submitted for the Turbin3 Q4 2026 prerequisite challenge, Part 1.

Based on the provided [`pre-req-vault`](https://github.com/ShrinathNR/pre-req-vault) program.

## What changed

`withdraw` now performs a Cross-Program Invocation into the registration program
(`TRBZyQHB3m68FGeVsqTK39Wm4xejadjVhP5MAZaKWDM`) and calls its `initialize(github)`
instruction, which creates the `["prereqs", user]` application account and records the
GitHub handle.

- `withdraw` signature: `withdraw(ctx, amount: u64, github: String)`.
- The GitHub handle is passed in as an instruction argument rather than hardcoded, so the
  program stays reusable; the caller supplies it.
- `CpiContext::new` is given `self.application_program.key()`. In anchor-lang 1.x this
  method takes a `Pubkey`, not an `AccountInfo`.
- The TS test asserts the CPI actually happened: the application account must exist, be
  owned by the registration program, and contain the handle.

## Accounts

| Account | Seeds | Owner |
|---|---|---|
| `vault_state` | `["state", user]` | vault program |
| `vault` | `["vault", vault_state]` | system program (lamports only) |
| `application_account` | `["prereqs", user]` | registration program |

## Program id

```
CJD3rkMekCepEyHQboCumjmj7t1hY3defRkRTQRrpopV
```

A fresh keypair is used because the upstream program id is owned by another upgrade
authority; deploying to it would be rejected.

## Flow

1. `initialize` — create `vault_state`, store both bumps.
2. `deposit(amount)` — system transfer user → vault.
3. `withdraw(amount, github)` — system transfer vault → user (vault PDA signs with its
   seeds), then CPI `registration::initialize(github)`.
4. `close` — drain the vault to the user, close `vault_state` and refund rent.

Both actions in `withdraw` are atomic: if the registration fails, the withdrawal reverts.

## Run it

Requires Rust, Solana CLI, Anchor 1.2.1, pnpm, and ~1.5 SOL on devnet.

```sh
anchor build
anchor keys sync
anchor test            # Anchor.toml targets devnet
```

Set `GITHUB_USERNAME` in `tests/pre-req-vault.ts` first — registration is one per wallet,
so the value is final once the test runs.

## Verified on devnet

`anchor test` — 4 passing (initialize, deposit, withdraw-with-CPI, close).

The CPI is visible in the invoke stack of the withdraw transaction, at depth 2
(a direct client call would be depth 1):

```
Program CJD3rkMekCepEyHQboCumjmj7t1hY3defRkRTQRrpopV invoke [1]
Program log: Instruction: Withdraw
Program 11111111111111111111111111111111 success
Program TRBZyQHB3m68FGeVsqTK39Wm4xejadjVhP5MAZaKWDM invoke [2]
Program log: Instruction: Initialize
Program 11111111111111111111111111111111 success
Program TRBZyQHB3m68FGeVsqTK39Wm4xejadjVhP5MAZaKWDM success
Program CJD3rkMekCepEyHQboCumjmj7t1hY3defRkRTQRrpopV success
```

- withdraw tx: `3EP7DpYrS9EGwYGbFDY4LEhhHSjjHDfC7rXfukNa3LzVTJ7kzLMinC7VzzYBhitA13R8fY4fnuk1atQkn5YTwiZG`
- close tx: `4iTffRNGMXMoLdJe9G6C34NefFvDFi1vAkkM5bZDsmMeG2sDiz1pckoGvSNNb2zkHVUxYRj4yCcZnM6ZEkYKeuhW`

Registration account created by the CPI, decoded from devnet:

```
application_account : AZvRmk4uwz3rxPN5HZ3GRvynkAmKeKoBxx7YVCnDozzd
owner               : TRBZyQHB3m68FGeVsqTK39Wm4xejadjVhP5MAZaKWDM  (registration program)
user                : 547yykCo1dL41cZnXFoFmd8wW9QNC2ofKUMzbdfrBUej
bump                : 254
pre_req_ts / rs     : false / false
github              : "Init-Init"
```

Explorer:
`https://explorer.solana.com/address/AZvRmk4uwz3rxPN5HZ3GRvynkAmKeKoBxx7YVCnDozzd?cluster=devnet`

## Notes

- `Anchor.toml` sets `skip_local_validator = true` and `cluster = "devnet"`: the test must
  run against devnet because the registration program only exists there.
- The litesvm suite under `programs/pre-req-vault/tests/` is commented out upstream, so the
  TypeScript test is the real check.
