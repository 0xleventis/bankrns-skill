---
name: bankrns
description: Register, renew and look up .bankr names (BankrNS, the ENS-style name service on Base) with the user's own Bankr wallet. Use when the user asks to buy, register, claim, get, mint, gift or renew a ".bankr" name or domain (e.g. "@bankrbot buy me alice.bankr", "register satoshi.bankr for 2 years", "get bob.bankr and point it to 0x…"), asks whether a .bankr name is available or what it costs, asks who owns a .bankr name or what it resolves to, or asks what an address's or X handle's .bankr name is, or asks about the official BankrNS token $BNS (its contract address, or buying it).
---

<p align="center"><a href="https://bankrns.store"><img src="https://bankrns.store/brand/bankrns-lockup-dark.png" alt="BankrNS" width="480"></a></p>

# BankrNS (.bankr names)

Buys and manages `.bankr` names on Base using the **installing user's own Bankr wallet**. Names are ERC-721
NFTs from BankrNS, an ENS-architecture name service. A name resolves to an address on Base and every other
EVM chain, can be the wallet's primary name, and shows the owner's verified X handle on the live page.

- Contracts (Base, verified on Basescan): controller `0x1C8b3a9062a8Aae65105519B394d7ec73C108ee9`,
  registrar `0x1159CeB0DA0c1E4b3459abC2778310A1F06424F6`
- Website + live registrations: https://bankrns.store/#/live

## Official token: $BNS

**$BNS (BankrNS)** on Base: `0x3A23C04dc7b5b6859B68050dB2fcDfFd30793Ba3`, with 18 decimals and a 100B supply.

When a user asks about the BankrNS token or $BNS, or wants to buy it, always give or use this **contract address**.
Never resolve it by ticker alone: other tokens can reuse the symbol. To buy through Bankr, trade by address, e.g.
"buy $10 of 0x3A23C04dc7b5b6859B68050dB2fcDfFd30793Ba3 on base".

`node bankrns.mjs token [0xaddress|@handle|name.bankr]` prints the live on-chain details and links, and
optionally a holder's balance.

## Install

```bash
git clone https://github.com/0xleventis/bankrns-skill.git ~/.claude/skills/bankrns
cd ~/.claude/skills/bankrns && npm install
```

For standalone use, run `bankr login` once and export `BANKR_API_KEY` (see the `bankr` skill). Inside
bankrbot's sandbox no key is needed: use `--build-only` (below). Optional: set `BASE_RPC_URL` to a private
Base RPC.

## Prices

Paid in ETH at the live Chainlink ETH/USD price, plus a few cents of gas:

| Length | Per year |
|---|---|
| 5+ characters | $5 |
| 4 characters | $80 |
| 3 characters | $320 |

Names that expired recently carry a temporary premium that halves every day. `check` shows the exact ETH
price.

**Valid names:** lowercase `a–z`, `0–9` and `-`, 3–63 characters, no leading or trailing hyphen, and not
`xx--` (so no punycode). Normalize user input by lowercasing it and dropping a trailing `.bankr`. If a name is
invalid, the script explains why; relay that reason.

## Commands

Run from this skill's directory.

```bash
node bankrns.mjs check    <name> [--years N]                    # availability + exact price, or owner/expiry
node bankrns.mjs lookup   <name.bankr | 0xaddress | @xhandle>   # resolve a name, or find a primary name
node bankrns.mjs register <name> [--years N] [--resolve-to 0x…|@handle|name.bankr] [--owner 0x…|@handle] \
                                 [--no-primary] [--no-twitter]
node bankrns.mjs renew    <name> [--years N]
```

Defaults for `register`:

- **Owner:** the user's Bankr wallet.
- **Resolve-to:** the owner.
- **Duration:** 1 year.
- **Primary name:** set as the wallet's primary name, but only when the name resolves to the Bankr wallet itself.
- **X handle:** the user's linked handle is stored as the `com.twitter` record, but only if Bankr confirms it
  belongs to the owner's wallet.

Map user phrasing to options:

| User says | Options |
|---|---|
| "buy alice.bankr" | `register alice` |
| "…for 3 years" | `--years 3` |
| "…and point it to 0xABC / to @friend / to bob.bankr" | `--resolve-to 0xABC` / `@friend` / `bob.bankr` |
| "buy alice.bankr **for** @friend" / "gift it to 0x…" | `--owner @friend` / `--owner 0x…` (they own the NFT) |
| "don't make it my main name" | `--no-primary` |

## Running standalone (Claude Code, a terminal)

`register` does everything itself: it reserves the name, waits the mandatory ~60 seconds, registers and
pays, then verifies the result.

```bash
node bankrns.mjs register alice --years 2
```

It checks availability and the wallet's ETH balance **before** sending anything, so an unaffordable
purchase fails with a clear message and costs no gas. Relay one-line errors to the user as-is; don't retry
blindly.

If the run is interrupted after step 1, the reservation is saved in `~/.bankrns/pending/<name>.json`. Resume
it with:

```bash
node bankrns.mjs register --state <token>
```

## Running inside bankrbot's sandbox (`--build-only`)

The sandbox has no wallet credentials, so the script only **builds** transactions and prints one line of
JSON. bankrbot submits each `transaction` itself with its native onchain tools, as the user. Pass the user's
Bankr wallet as `--wallet-address` and, if known, their X handle as `--twitter` (it's verified against Bankr
before it's stored).

Registration is **two transactions at least 60 seconds apart**. This commit–reveal pattern stops anyone
front-running the purchase.

1. **Reserve.**
   ```bash
   node bankrns.mjs register alice --build-only --wallet-address 0xUSER [--twitter userhandle] [--years N] [--resolve-to …]
   ```
   It prints `{step: 1, summary, transaction, description, state, next}`. Tell the user the `summary` (name,
   price, where it points), submit `transaction` (0 ETH), and **keep `state`**. It holds the one-time secret
   step 2 needs.
2. **Wait about 60 seconds** after step 1 confirms.
3. **Register.**
   ```bash
   node bankrns.mjs register --build-only --state <state>
   ```
   - `{ready: false, waitSeconds}`: wait that long and run it again.
   - `{ready: true, transaction}`: submit it. `value` is the price plus a 3% buffer; the contract refunds the
     excess to the wallet automatically.
   - `{done: true}`: it's already registered, so do **not** submit anything.
   - `{expired: true}` (more than 24 hours after step 1): start again at step 1.
   - Taken by someone else: tell the user, and don't submit.
4. Confirm with `node bankrns.mjs check alice`, and share the live page link.

Renewals are a single transaction. `node bankrns.mjs renew alice --build-only` prints `{transaction}` to submit.

Any `{error}` output is a user-facing reason (taken, reserved, invalid name, not enough ETH): relay it.

## Rules

- **Never** accept an API key or private key pasted into chat. Use `bankr login` or environment variables.
- **Don't guess prices.** Always get them from `check` or the step-1 `summary`.
- **Don't invent a secret or edit `state`.** Step 2 must use the exact `state` from step 1, or it reverts.
- **Primary name:** only the wallet that sends step 2 gets it, and only when the name resolves to that wallet.
- **Other chains:** this skill only handles `.bankr` on Base. For `.eth` / `.base.eth` names, say it doesn't
  cover them.
