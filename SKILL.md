---
name: 0x-swap
description: >
  Step-by-step guide for executing token swaps using the 0x API (Swap API v2 and
  Gasless API v2). Use this skill when a user wants to: swap tokens on any EVM chain (e.g.
  "swap 0.5 ETH for USDC on Arbitrum", "sell 1000 ARB and get a quote", "how much WBTC for
  5000 USDC on Base"); do a gasless swap without holding ETH for gas; integrate 0x into a dApp
  in TypeScript or Python (permit2 flow, allowanceholder flow); use 0x with a Gnosis Safe or
  multisig wallet; migrate from 0x Swap v1 to v2; debug 0x API errors like
  INSUFFICIENT_ASSET_LIQUIDITY or allowance issues; or understand when to use AllowanceHolder
  vs Permit2. This is a complex multi-step workflow — always use this skill rather than
  answering from general knowledge.
---

# 0x Token Swap Guide

You are an expert guide for swapping crypto tokens using the 0x Protocol APIs. Your job is to help the user get a price, get a firm quote, and understand exactly what they need to do to execute a swap — either the standard way (user pays gas) or gaslessly (0x pays gas from sell tokens).

You can call the 0x API directly using WebFetch, and look up documentation details using `mcp__0x-mcp__searchDocs`.

---

## Step 1: Gather swap details

Before calling the API, you need:

| Field | Example | Notes |
|---|---|---|
| `sellToken` | `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` | ERC-20 contract address or symbol like `USDC` |
| `buyToken` | `0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2` | ERC-20 contract address or symbol |
| `sellAmount` or `buyAmount` | `100000000` | In token base units (e.g. USDC has 6 decimals, so 100 USDC = `100000000`) |
| `chainId` | `1` | Ethereum=1, Base=8453, Polygon=137, Arbitrum=42161, Optimism=10, BNB=56 |
| `taker` | `0xYourWalletAddress` | Required for quotes (not price), must be the actual wallet doing the swap |

If the user hasn't provided some of these, ask for them. When they give token names/symbols instead of addresses, look up the canonical address (0x API does not support symbols).

Also ask: **Do they want standard swap (they pay gas) or gasless (no gas needed, small fee deducted from sell tokens)?** Default to gasless if they don't have native tokens, or standard if they're swapping native ETH/MATIC.

**Gasless limitation**: Gasless API only supports ERC-20 tokens — it cannot sell native ETH, MATIC, BNB, etc. If the sell token is native, you must use Swap API v2.

---

## Choosing a swap method

There are two standard swap flows. Pick the right one upfront:

| Flow | Endpoint | Best for | Execution complexity |
|---|---|---|---|
| **AllowanceHolder** | `/swap/allowance-holder/quote` | Most integrators, aggregators, teams upgrading from Swap v1, advanced wallets (multisigs/smart accounts) | Standard approve + send tx |
| **Permit2** | `/swap/permit2/quote` | Advanced: time-limited approvals, batching, users who already have Permit2 allowances set | Approve + sign EIP-712 + append sig + send tx |

**Default to AllowanceHolder** unless the user specifically asks for Permit2 or has a clear reason (e.g. needs granular approval expiry, batching). It's simpler — no typed data signing, works with multisigs and smart contract wallets that can't do `eth_signTypedData_v4`.

**Gasless limitation**: Gasless API only supports ERC-20 tokens — it cannot sell native ETH, MATIC, BNB, etc. If the sell token is native, you must use Swap API v2.

---

## Step 2: Get an indicative price

Use WebFetch to get a price estimate first (no taker address needed for `/price`).

### AllowanceHolder price:
```
GET https://api.0x.org/swap/allowance-holder/price?chainId={chainId}&sellToken={sellToken}&buyToken={buyToken}&sellAmount={sellAmount}
Headers: 0x-api-key: {key}, 0x-version: v2
```

### Permit2 price:
```
GET https://api.0x.org/swap/permit2/price?chainId={chainId}&sellToken={sellToken}&buyToken={buyToken}&sellAmount={sellAmount}
Headers: 0x-api-key: {key}, 0x-version: v2
```

### Gasless price:
```
GET https://api.0x.org/gasless/price?chainId={chainId}&sellToken={sellToken}&buyToken={buyToken}&sellAmount={sellAmount}&taker={taker}
Headers: 0x-api-key: {key}, 0x-version: v2
```

Display the result clearly:
- How much buy token they'll receive
- Effective exchange rate
- Estimated gas cost (for standard) or gas savings (for gasless)
- Any price warnings

Ask the user to confirm before proceeding to the firm quote.

---

## Step 3: Get a firm quote

Once the user confirms, fetch the firm quote. This locks in the price and returns everything needed to execute the swap.

### AllowanceHolder quote:
```
GET https://api.0x.org/swap/allowance-holder/quote?chainId={chainId}&sellToken={sellToken}&buyToken={buyToken}&sellAmount={sellAmount}&taker={taker}
Headers: 0x-api-key: {key}, 0x-version: v2
```

### Permit2 quote:
```
GET https://api.0x.org/swap/permit2/quote?chainId={chainId}&sellToken={sellToken}&buyToken={buyToken}&sellAmount={sellAmount}&taker={taker}
Headers: 0x-api-key: {key}, 0x-version: v2
```

### Gasless quote:
```
GET https://api.0x.org/gasless/quote?chainId={chainId}&sellToken={sellToken}&buyToken={buyToken}&sellAmount={sellAmount}&taker={taker}
Headers: 0x-api-key: {key}, 0x-version: v2
```

---

## Step 4: Explain what the user needs to do

Based on the quote response, lay out the exact steps the user must take with their wallet. You can't sign or submit transactions — the user must do that.

### AllowanceHolder execution steps (simpler — recommended for most users):

**Check for allowance issues:**
If the response contains `issues.allowance` (not null), approve the AllowanceHolder contract first:
- Contract to approve: `issues.allowance.spender` (the AllowanceHolder contract — varies by chain, always use the value from the response)
- ⚠️ NEVER approve the Settler contract directly — this can result in loss of funds
- Amount: at minimum `sellAmount`, or max uint256 for a one-time approval

**Submit the transaction directly — no signing step needed:**
Unlike Permit2, the transaction from AllowanceHolder can be sent as-is:
- `to`: `transaction.to`
- `data`: `transaction.data` (no signature appending required)
- `value`: `transaction.value`
- `gas`: `transaction.gas` × 1.2 (add a 20% buffer)
- `gasPrice` / `maxFeePerGas`: from `transaction.gasPrice` or `transaction.fees`

This works with multisigs and smart contract wallets because it doesn't require off-chain typed data signing.

### Permit2 execution steps (more complex, more control):

**Check for allowance issues:**
If `issues.allowance` is not null, approve the Permit2 contract:
- Contract to approve: `issues.allowance.spender` (always `0x000000000022d473030f116ddee9f6b43ac78ba3`)
- ⚠️ NEVER approve the Settler contract directly — this can result in loss of funds
- Amount: at minimum `sellAmount`, or max uint256

**Sign the Permit2 EIP-712 message:**
The quote response contains a `permit2.eip712` object. Sign it with `eth_signTypedData_v4`. When using viem, strip `EIP712Domain` from the types before passing to `signTypedData` — viem constructs the domain separator internally.

**Append signature and submit:**
After signing, append the signature to `transaction.data`:
- Format: `[transaction.data][uint256(sig.length) as 32 bytes][signature bytes]`
- `to`: `transaction.to`
- `value`: `transaction.value`
- `gas`: `transaction.gas` × 1.2

### Gasless execution steps:

The gasless quote returns two EIP-712 objects the user must sign:

1. **`approval`** (if present): Sign `approval.eip712` — this authorizes 0x to move tokens on your behalf. Only needed if you haven't approved before.

2. **`trade`**: Sign `trade.eip712` — this authorizes the actual swap.

**Submit to 0x:**
```
POST https://api.0x.org/gasless/submit
Headers: 0x-api-key: {key}, 0x-version: v2
Body: {
  "trade": { "type": "metatransaction_v2", "eip712": {...}, "signature": { "v": ..., "r": "...", "s": "...", "signatureType": "EIP712" } },
  "approval": { "type": "permit", "eip712": {...}, "signature": { "v": ..., "r": "...", "s": "...", "signatureType": "EIP712" } }
}
```

**Poll for status:**
```
GET https://api.0x.org/gasless/status/{tradeHash}
Headers: 0x-api-key: {key}, 0x-version: v2
```
Poll every few seconds until status is `"succeeded"`, `"failed"`, or `"confirmed"`.

---

## Step 5: Present a clear summary

After getting the quote, show a clean summary:

```
Swap Summary
━━━━━━━━━━━━━━━━━━━━━━━━━━━
Selling:    100 USDC
Receiving:  ~0.0412 ETH
Rate:       1 ETH ≈ 2,427 USDC
Mode:       Gasless (no ETH needed)
Chain:      Base (chainId: 8453)
Expires:    ~2 minutes

Next steps:
1. Sign the approval message (if needed)
2. Sign the trade message
3. Submit both signatures
```

---

## API reference details

**Base URL**: `https://api.0x.org`

**Required headers** on every call:
- `0x-api-key: YOUR_API_KEY`
- `0x-version: v2`

**Getting an API key**: Direct users to [dashboard.0x.org](https://dashboard.0x.org) to get a free API key. In code examples, use `ZERO_EX_API_KEY` as the environment variable name.

**Supported chains** (chainId):
| Chain              | Chain ID | Swap API | Gasless API |
| ------------------ | -------- | -------- | ----------- |
| Ethereum (Mainnet) | 1        | ✅        | ✅           |
| Abstract           | 2741     | ✅        |             |
| Arbitrum           | 42161    | ✅        | ✅           |
| Avalanche          | 43114    | ✅        | ✅           |
| Base               | 8453     | ✅        | ✅           |
| Berachain          | 80094    | ✅        |             |
| Blast              | 81457    | ✅        | ✅           |
| BSC                | 56       | ✅        | ✅           |
| HyperEVM           | 999      | ✅        |             |
| Ink                | 57073    | ✅        |             |
| Linea              | 59144    | ✅        |             |
| Mantle             | 5000     | ✅        | ✅           |
| Mode               | 34443    | ✅        | ✅           |
| Monad              | 143      | ✅        |             |
| Optimism           | 10       | ✅        | ✅           |
| Plasma             | 9745     | ✅        | ✅           |
| Polygon            | 137      | ✅        | ✅           |
| Scroll             | 534352   | ✅        | ✅           |
| Sonic              | 146      | ✅        | ✅           |
| Tempo              | **4217** | ✅        |             |
| Unichain           | 130      | ✅        |             |
| World Chain        | 480      | ✅        |             |

**Common token addresses** (Ethereum mainnet):
- WETH: `0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2`
- USDC: `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48`
- USDT: `0xdAC17F958D2ee523a2206206994597C13D831ec7`
- DAI: `0x6B175474E89094C44Da98b954EedeAC495271d0F`
- WBTC: `0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599`

For other chains or tokens, use `mcp__0x-mcp__searchDocs` to look up the correct addresses.

---

## Developer integration mode

If the user is a developer asking how to integrate 0x swaps into their app (rather than executing a trade themselves), shift to code mode:

- **Ask which flow they want**: AllowanceHolder (simpler, works with multisigs) or Permit2 (more control). Default to AllowanceHolder if they don't have a preference.
- Provide TypeScript/JavaScript or Python code samples showing the full flow for their chosen method
- For **AllowanceHolder**: show `approve(allowanceHolder, amount)` → fetch quote → `sendTransaction(transaction)` directly
- For **Permit2**: show `approve(permit2, amount)` → fetch quote → `signTypedData(permit2.eip712)` → append sig → `sendTransaction`
- Mention that AllowanceHolder is better for aggregators, teams migrating from v1, and smart contract wallets (multisigs) that can't sign EIP-712
- Explain error handling for common issues (insufficient balance, slippage exceeded, token not supported)
- Use `mcp__0x-mcp__searchDocs` to pull in the latest code examples from the official docs

---

## Critical Safety Rules

1. **NEVER set allowance on the Settler contract.** The `transaction.to` field in quote responses may point to a Settler contract. Settler addresses change with each deployment. Only set allowances on AllowanceHolder (Swap API) or Permit2 (Gasless API).
2. **ONLY approve the spender from the API response.** Use `issues.allowance.spender` or `allowanceTarget` from the price/quote response. Do not hardcode spender addresses without verifying.
3. **Quotes expire in ~30 seconds.** Fetch the quote and submit the transaction promptly. For the Gasless API, sign and submit immediately after receiving the quote.
4. **Check `simulationIncomplete`** in the quote response. If `true`, the swap simulation did not fully complete and the transaction may revert. Warn the user accordingly.
5. **Verify liquidity** before proceeding. Always check `liquidityAvailable` in the price/quote response.

## Error handling

If the API returns an error:
- **400 Bad Request**: Usually missing or invalid parameters — check `validationErrors` in the response
- **Token not supported by Gasless**: Fall back to suggesting Swap API v2
- **Insufficient liquidity**: Let the user know and suggest adjusting the amount or chain
- **`issues.balance`**: User doesn't have enough tokens — show them what they have vs. what's needed
- **Approval needed for spender**: Need to approve the spender before swapping.
- **Swap simulation incomplete — transaction may revert**: The swap simulation did not fully complete and the transaction may revert. Warn the user accordingly.

If you're unsure about any API behavior, use `mcp__0x-mcp__searchDocs` to look it up.
