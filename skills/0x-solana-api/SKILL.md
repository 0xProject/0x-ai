---
name: 0x-solana-api
description: >
  Guide for swapping tokens on Solana with the 0x Solana Swap API. Use this skill when a
  user wants to swap SPL or Token-2022 tokens or native SOL on Solana, integrate 0x swaps
  into a Solana app, wallet, or bot, build and send a transaction from 0x swap
  instructions, charge a swap fee on Solana, or debug a failing 0x Solana swap. For EVM
  chains use the 0x-api skill instead.
mcp_servers:
  - name: 0x-mcp
    url: https://docs.0x.org/_mcp/server
license: MIT
---

# 0x Solana Swap Guide

The 0x Solana Swap API returns the instructions for a swap. The integrator puts them in a transaction, the user's wallet signs it, and the integrator sends it.

## Where the details live

Read these before writing code; they are kept current, this skill is not a copy of them.

- Guide: https://docs.0x.org/solana/guides/get-started.md
- Request and response fields: https://docs.0x.org/api-reference/solana-swap-ap-is/swap/instructions.md
- Swap fees: https://docs.0x.org/solana/guides/swap-fees.md
- Errors and common questions: https://docs.0x.org/solana/need-help/faq.md
- Runnable example: https://github.com/0xProject/0x-examples/tree/main/solana-example
- Anything else: `mcp__0x-mcp__searchDocs`

## Rules that are easy to get wrong

- Native SOL is `So11111111111111111111111111111111111111111`. `So11111111111111111111111111111111111111112` is the wSOL mint; use it only when the user wants wSOL in a token account.
- Tokens are mint addresses, not symbols. Amounts are in base units.
- Always request `transaction_version: "v1"`, and build a v1 transaction. Use v0 only for a wallet that cannot sign v1.
- In a v1 transaction, set both the compute unit limit and the loaded accounts data size limit (measure them by simulating). If they are not set, they are zero and the transaction fails.
- The `taker` in the request must be the wallet that signs.
- Request instructions right before the user signs. If the transaction expires, request new instructions; never resend the old transaction.
- Keep the API key on a server and read it from an environment variable.

## Your role

Give the user code to run. Do not call the 0x API yourself, and never ask for or handle a private key.
