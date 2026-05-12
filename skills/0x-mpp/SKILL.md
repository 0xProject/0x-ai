---
name: 0x-mpp
description: >
  Access the 0x Swap API (AllowanceHolder price and quote) without an API key,
  paying per request via the MPP Tempo protocol. Use this skill when a user wants
  to: get a 0x swap price or quote without managing an API key; build an autonomous
  agent that pays for 0x API access on-chain; use the MPP (Machine Payment Protocol)
  with mppx and viem; or test a keyless 0x integration on testnet. Payment is made
  in pathUSD on Tempo Testnet (chainId 42431) via the mppx library. The 0x API key
  is injected server-side by the AgentPay proxy — callers only need a funded wallet.
license: MIT
---

# 0x Swap API via MPP (Keyless)

Access 0x swap prices and quotes without a 0x API key. Each request is paid
per-call in **pathUSD on Tempo Testnet** using the
[MPP (Machine Payment Protocol)](https://agentpay.alchemy.com/docs) and the
[`mppx`](https://github.com/wevm/mppx) library.

The 0x API key is configured server-side in the AgentPay dashboard — callers
only need a wallet funded with testnet pathUSD.

## Endpoints

| Endpoint | Description |
|---|---|
| `swap-allowance-holder-price` | Indicative price — no `taker` required |
| `swap-allowance-holder-quote` | Firm quote with transaction calldata — requires `taker` |

**Base URL:**
```
https://agent-proxy.alchemy.com/v1/mpp-tempo-testnet/86b0ec6d1d0b3dc0
```

## Payment details

| Field | Value |
|---|---|
| Protocol | MPP Tempo Testnet |
| Chain | Tempo Testnet (chainId 42431) |
| Token | pathUSD (`0x20c0000000000000000000000000000000000000`) |
| Cost per request | ~0.001 pathUSD |
| Library | `mppx` + `viem` |

## How it works

1. Client sends request → AgentPay responds with **HTTP 402** and a Tempo payment challenge
2. `mppx/client` signs a transaction on Tempo Testnet using your wallet
3. Client retries with `Authorization: Payment <credential>` header
4. AgentPay verifies payment and forwards to the 0x API
5. Real 0x price/quote response is returned

## Prerequisites

- Node.js 18+
- A wallet with **pathUSD** on Tempo Testnet (chainId 42431)

## Installation

```bash
npm install mppx viem dotenv tsx
```

## Environment variables

```bash
# .env
PRIVATE_KEY=0x<your-64-char-hex-key>
```

## Code

```typescript
import 'dotenv/config'
import { Mppx, tempo } from 'mppx/client'
import { privateKeyToAccount } from 'viem/accounts'

const account = privateKeyToAccount(process.env.PRIVATE_KEY as `0x${string}`)

const mppx = Mppx.create({
  methods: [tempo.charge({ account, mode: 'pull' })],
  polyfill: false,
})

const BASE_URL = 'https://agent-proxy.alchemy.com/v1/mpp-tempo-testnet/86b0ec6d1d0b3dc0'

// Swap params — WETH → USDC on Base (chainId 8453)
const params = new URLSearchParams({
  chainId: '8453',
  sellToken: '0x4200000000000000000000000000000000000006', // WETH on Base
  buyToken:  '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',  // USDC on Base
  sellAmount: '1000000000000000',                            // 0.001 WETH
})

// Price — no taker required
const priceRes = await mppx.fetch(`${BASE_URL}/swap-allowance-holder-price/?${params}`)
const price = await priceRes.json()
console.log('buyAmount:', price.buyAmount)
console.log('liquidityAvailable:', price.liquidityAvailable)

// Quote — taker required
params.set('taker', account.address)
const quoteRes = await mppx.fetch(`${BASE_URL}/swap-allowance-holder-quote/?${params}`)
const quote = await quoteRes.json()
console.log('transaction.to:', quote.transaction?.to)
console.log('transaction.gas:', quote.transaction?.gas)
```

## Response fields

**Price (`swap-allowance-holder-price`):**
- `buyAmount` — tokens received (base units)
- `sellAmount` — tokens sold (base units)
- `estimatedPriceImpact` — slippage estimate
- `liquidityAvailable` — must be `true` before executing
- `issues.allowance` — non-null if taker needs to approve

**Quote (`swap-allowance-holder-quote`):**
All price fields, plus:
- `transaction.to` — contract to call (do **not** approve this address)
- `transaction.data` — calldata for the swap
- `transaction.gas` — estimated gas (add 20% buffer)
- `transaction.gasPrice` — gas price
- `issues.allowance.spender` — address to approve (use this, not `transaction.to`)

> ⚠️ Quotes expire in ~30 seconds. Submit the transaction immediately after fetching.

## Execution (after getting a quote)

```typescript
import { createWalletClient, http, erc20Abi, maxUint256 } from 'viem'
import { base } from 'viem/chains'

const walletClient = createWalletClient({ account, chain: base, transport: http() })

// 1. Approve if needed
if (quote.issues?.allowance) {
  await walletClient.writeContract({
    address: quote.sellToken,
    abi: erc20Abi,
    functionName: 'approve',
    args: [quote.issues.allowance.spender, maxUint256],
  })
}

// 2. Send the swap transaction
const txHash = await walletClient.sendTransaction({
  to:       quote.transaction.to,
  data:     quote.transaction.data,
  value:    BigInt(quote.transaction.value),
  gas:      BigInt(Math.floor(Number(quote.transaction.gas) * 1.2)),
  gasPrice: BigInt(quote.transaction.gasPrice),
})
```

## Reference implementation

Full working test script: [jlin27/agentpay-mpp](https://github.com/jlin27/agentpay-mpp)

## Related skills

- [`0x-api`](../0x-api/SKILL.md) — standard 0x integration with an API key
- [`0x-x402`](../0x-x402/SKILL.md) — same keyless endpoints, paying via x402 (USDC on Base Sepolia)
