---
name: 0x-solana-api
description: >
  Guide for swapping tokens on Solana with the 0x Solana Swap API
  (POST /solana/swap-instructions). Use this skill when a user wants to: swap SPL or
  Token-2022 tokens or native SOL on Solana (e.g. "swap 1 SOL for USDC", "sell BONK for
  SOL"); integrate 0x swaps into a Solana app, wallet, or bot in TypeScript; build, sign, and
  send the transaction from 0x swap instructions with Solana Kit; charge a swap fee on
  Solana; sponsor a user's transaction fees; or debug 0x Solana errors like NO_POOLS_FOUND,
  ROUTING_FAILED, or a v1 transaction that fails on compute units. For EVM chains use the
  0x-api skill instead.
mcp_servers:
  - name: 0x-mcp
    url: https://docs.0x.org/_mcp/server
license: MIT
---

# 0x Solana Swap Guide

You help developers swap tokens on Solana with the 0x Solana Swap API. The API returns the instructions for a swap; the integrator puts them in a transaction, the user's wallet signs it, and the integrator sends it.

Call `mcp__0x-mcp__searchDocs` for anything this skill does not cover. Give the user code to run; do not call the 0x API yourself and never ask for or handle a private key.

Runnable reference: [0x-examples/solana-example](https://github.com/0xProject/0x-examples/tree/main/solana-example) does the whole flow below in one TypeScript file. With only an API key it quotes and simulates a swap.

---

## Step 1: Gather swap details

| Field | Example | Notes |
|---|---|---|
| `token_in`, `token_out` | `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` (USDC) | Base58 mint addresses. The API does not accept symbols. |
| `amount_in` | `1000000` | Base units of `token_in`. 1 SOL = 1,000,000,000 lamports; 1 USDC = 1,000,000. |
| `taker` | wallet address | The wallet that signs. The instructions are built for this wallet; a different signer fails. |
| `slippage_bps` | `50` | Optional. Default 50 (0.5%). |

**Native SOL is `So11111111111111111111111111111111111111111`** (ends in 1). The instructions wrap and unwrap it. `So11111111111111111111111111111111111111112` (ends in 2) is the wSOL mint: use it only when the user wants wSOL in a token account.

---

## Step 2: Request swap instructions

Always send `transaction_version: "v1"`. Only `0x-api-key` is required; Solana does not use `0x-version`. Call the API from a backend so the key stays private.

```ts
const response = await fetch("https://api.0x.org/solana/swap-instructions", {
  method: "POST",
  headers: { "0x-api-key": process.env.ZERO_EX_API_KEY!, "Content-Type": "application/json" },
  body: JSON.stringify({
    token_in: "So11111111111111111111111111111111111111111", // native SOL
    token_out: "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v", // USDC
    amount_in: 1_000_000_000, // 1 SOL
    taker: takerAddress,
    slippage_bps: 50,
    transaction_version: "v1",
  }),
});
const quote = await response.json();
if (!response.ok) throw new Error(`${quote.code}: ${quote.error} (zid ${quote.zid})`);
```

The response has `instructions` (execute in order), `amount_out` (expected output, base units), `min_amount_out` (least the taker receives; below this the transaction fails), `route_plan`, and `zid` (quote it to 0x support). For v1, `address_lookup_tables` is always empty.

Optional fields:
- `recipient`: send the output elsewhere: a wallet (native SOL), a token account, or `{"associatedTokenAccount": {"owner": "<wallet>"}}`.
- `sponsor`: a wallet that pays the transaction fee and account rent. The sponsor becomes the fee payer and both sponsor and taker sign.
- `swap_fee_ppm`, `swap_fee_recipient`, `swap_fee_side`: see Step 5.
- `reserve_transaction_accounts`: accounts to leave free for instructions the integrator adds; see Step 4.
- `disabled_sources`: DEX names to exclude; `GET /solana/enabled-sources` lists them.

---

## Step 3: Build, sign, and send a v1 transaction

Use `@solana/kit` 8.5 or later. `@solana/web3.js` 1.x cannot build v1 transactions, and the user's wallet must support signing them.

```ts
import {
  AccountRole, appendTransactionMessageInstructions, createTransactionMessage,
  estimateResourceLimitsFactory, getAddressDecoder, pipe,
  setTransactionMessageComputeUnitLimit, setTransactionMessageFeePayerSigner,
  setTransactionMessageLifetimeUsingBlockhash, setTransactionMessageLoadedAccountsDataSizeLimit,
  setTransactionMessagePriorityFeeLamports, signTransactionMessageWithSigners,
} from "@solana/kit";

const decode = getAddressDecoder();
const role = (s: boolean, w: boolean) =>
  s ? (w ? AccountRole.WRITABLE_SIGNER : AccountRole.READONLY_SIGNER)
    : (w ? AccountRole.WRITABLE : AccountRole.READONLY);
const instructions = quote.instructions.map((ix: any) => ({
  programAddress: decode.decode(Uint8Array.from(ix.program_id)),
  accounts: ix.accounts.map((a: any) => ({
    address: decode.decode(Uint8Array.from(a.pubkey)),
    role: role(a.is_signer, a.is_writable),
  })),
  data: Uint8Array.from(ix.data),
}));

const { value: blockhash } = await rpc.getLatestBlockhash().send();
const message = pipe(
  createTransactionMessage({ version: 1 }),
  (m) => setTransactionMessageFeePayerSigner(taker, m),
  (m) => setTransactionMessageLifetimeUsingBlockhash(blockhash, m),
  (m) => appendTransactionMessageInstructions(instructions, m),
  (m) => setTransactionMessagePriorityFeeLamports(10_000n, m), // v1: total lamports
);

// A v1 transaction with no limits set has limits of zero and fails.
const estimate = await estimateResourceLimitsFactory({ rpc })(message);
const withLimits = pipe(
  message,
  (m) => setTransactionMessageComputeUnitLimit(Math.min(Math.ceil(estimate.computeUnitLimit * 1.1), 1_400_000), m),
  (m) => setTransactionMessageLoadedAccountsDataSizeLimit(
    Math.min(Math.ceil(estimate.loadedAccountsDataSizeLimit * 1.1), 64 * 1024 * 1024), m),
);
const signed = await signTransactionMessageWithSigners(withLimits);
// then sendAndConfirmTransactionFactory({ rpc, rpcSubscriptions })(signed, { commitment: "confirmed" })
```

v1 rules to get right:
- **Set both the compute unit limit and the loaded accounts data size limit.** Unset means zero.
- **Do not add `ComputeBudget` instructions.** Limits and priority fee are message fields.
- **The priority fee is a total in lamports**, not micro-lamports per compute unit.
- **No address lookup tables.**
- Look up a v1 transaction with `getTransaction(..., { maxSupportedTransactionVersion: 1 })`.

Only if the user's wallet cannot sign v1: request `transaction_version: "v0"`, load the returned `address_lookup_tables` when compiling, add `ComputeBudget` instructions, and reserve room with `reserve_transaction_bytes`. v0 is limited to 1,232 bytes, so large trades get worse prices.

---

## Step 4: Landing the swap

- Request instructions right before the user signs. A quote a few seconds old is likelier to fail on `min_amount_out`.
- The simulation in Step 3 runs the whole swap: show its error to the user instead of asking for a signature.
- Add a priority fee when the network is busy.
- If the blockhash expires before the transaction lands, request new instructions and build a new transaction. Never resend the old one.
- If the integrator adds instructions (transfer, tip, memo), set `reserve_transaction_accounts` to the number of accounts they add that the swap does not use, and keep it as small as possible: reserved accounts are accounts the route cannot use.

---

## Step 5: Swap fees

- `swap_fee_ppm`: parts per million, not basis points. `10000` = 1%.
- `swap_fee_recipient`: a wallet (native SOL fee), an existing token account, or `{"associatedTokenAccount": {"owner": "<wallet>"}}`, which the instructions create if needed.
- `swap_fee_side`: `buy` (output token, default) or `sell` (input token).
- Each field takes a comma-separated list for several fees. Fees on one side apply one after another.
- Each entry is capped at `100000` (10%) by default. `amount_out` is already net of buy-side fees.

---

## Error handling

Errors return `{ "code", "error", "zid" }`.

| Code | Status | Meaning and fix |
|---|---|---|
| `NO_POOLS_FOUND`, `ROUTING_FAILED` | 422 | No route at this size. Check the mints and that `amount_in` is in base units; try a smaller amount. |
| `TOKEN_NOT_FOUND` | 422 | The API does not know this mint. Check the address. |
| `TOKEN_GRADUATED` | 422 | The pump.fun curve has migrated; retry shortly so the new pool is used. |
| `TOKEN_HAS_UNSUPPORTED_EXTENSIONS` | 400 | Token-2022 token with an active transfer hook, non-transferable, or paused. Not routable. |
| `UNSUPPORTED_TOKEN_PROGRAM` | 400 | The mint is not an SPL Token or Token-2022 mint. |
| `XSTOCK_NOT_SUPPORTED` | 422 | Backed xStock tokens are not routed. |
| `INPUT_OUTPUT_SAME_TOKEN`, `NULL_AMOUNT_IN`, `INVALID_SLIPPAGE`, `INVALID_PUBKEY`, `INVALID_REQUEST_BODY`, `ASSOCIATED_TOKEN_ACCOUNT_FOR_NATIVE` | 400 | Fix the request field named in `error`. |
| `INVALID_SWAP_FEE_PPM`, `INVALID_SWAP_FEE_SIDE`, `INVALID_SWAP_FEE_LIST_LENGTHS`, `INCOMPLETE_SWAP_FEE` | 400 | Fee fields: same number of entries, both `swap_fee_ppm` and `swap_fee_recipient` set, each fee within the cap. |
| `INVALID_SOURCE`, `ALL_SOURCES_DISABLED` | 400 | `disabled_sources` has an unknown name or excludes every DEX. |
| `SPONSOR_*`, `SPONSORED_*` | 400/422 | Sponsorship constraint; read `error`. |
| `WALLET_BLOCKED`, `TOKEN_BLOCKED` | 403 | Refused by screening. |
| `INTERNAL_ERROR` | 500/503 | Retry; if it persists, contact support with the `zid`. |

On-chain failures: a v1 compute or loaded-data-size error means the limits were not set; a slippage failure means the price moved past `min_amount_out`, so re-quote closer to signing or raise `slippage_bps`.
