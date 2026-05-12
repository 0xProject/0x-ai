---
name: 0x-x402
description: >
  Access the 0x Swap API (AllowanceHolder price and quote) without an API key,
  paying per request via the x402 protocol. Use this skill when a user wants to:
  get a 0x swap price or quote without managing an API key; build an autonomous
  agent that pays for 0x API access with USDC; use the x402 protocol with
  @x402/fetch and viem; or test a keyless 0x integration on testnet. Payment is
  made in USDC on Base Sepolia (chainId 84532) via EIP-3009
  transferWithAuthorization signed off-chain. The 0x API key is injected
  server-side by the AgentPay proxy — callers only need a funded wallet.
license: MIT
---

# 0x Swap API via x402 (Keyless)

Access 0x swap prices and quotes without a 0x API key. Each request is paid
per-call in **USDC on Base Sepolia** using the
[x402 protocol](https://github.com/coinbase/x402) and the
[`@x402/fetch`](https://www.npmjs.com/package/@x402/fetch) library.

The 0x API key is configured server-side in the AgentPay dashboard — callers
only need a wallet funded with testnet USDC.

## Endpoints

| Endpoint | Description |
|---|---|
| `swap-allowance-holder-price` | Indicative price — no `taker` required |
| `swap-allowance-holder-quote` | Firm quote with transaction calldata — requires `taker` |

**Base URL:**
```
https://agent-proxy.alchemy.com/v1/x402-testnet/86b0ec6d1d0b3dc0
```

## Payment details

| Field | Value |
|---|---|
| Protocol | x402 (`exact` scheme) |
| Chain | Base Sepolia (chainId 84532) |
| Token | USDC (`0x036CbD53842c5426634e7929541eC2318f3dCF7e`) |
| Cost per request | 0.01 USDC (10,000 base units) |
| Auth header | `PAYMENT-SIGNATURE` |
| Library | `@x402/fetch` + `@x402/evm` + `viem` |

Get testnet USDC from the [Circle Testnet Faucet](https://faucet.circle.com/).

## How it works

1. Client sends request → AgentPay responds with **HTTP 402** and an x402 challenge in the `payment-required` header
2. `@x402/fetch` parses the challenge and signs a USDC **EIP-3009 `transferWithAuthorization`** payload off-chain
3. Client retries with `PAYMENT-SIGNATURE: <base64-encoded-payload>` header
4. AgentPay verifies and settles the payment on-chain
5. Real 0x price/quote response is returned, with a `PAYMENT-RESPONSE` header containing the settlement tx hash

## Prerequisites

- Node.js 18+
- A wallet with **USDC** on Base Sepolia (`0x036CbD53842c5426634e7929541eC2318f3dCF7e`)
- Testnet USDC from [faucet.circle.com](https://faucet.circle.com/)

## Installation

```bash
npm install @x402/fetch @x402/evm viem dotenv
```

## Environment variables

```bash
# .env
WALLET_PRIVATE_KEY=0x<your-64-char-hex-key>
```

## Code

```javascript
import 'dotenv/config'
import { createPublicClient, http } from 'viem'
import { baseSepolia } from 'viem/chains'
import { privateKeyToAccount } from 'viem/accounts'
import { wrapFetchWithPayment, x402Client } from '@x402/fetch'
import { ExactEvmScheme, toClientEvmSigner } from '@x402/evm'

const account = privateKeyToAccount(process.env.WALLET_PRIVATE_KEY)

const publicClient = createPublicClient({ chain: baseSepolia, transport: http() })

const signer = toClientEvmSigner(account, publicClient)
const core = new x402Client()
core.register('eip155:84532', new ExactEvmScheme(signer))

// Budget guardrail: refuse to sign any payment over $0.02
core.registerPolicy((_version, requirements) =>
  requirements.filter((r) => BigInt(r.amount) <= 20_000n)
)

const x402Fetch = wrapFetchWithPayment(fetch, core)

const BASE_URL = 'https://agent-proxy.alchemy.com/v1/x402-testnet/86b0ec6d1d0b3dc0'

// Swap params — USDC → ETH on Base mainnet (chainId 8453)
// Note: x402 payment is on Base Sepolia; the swap query targets Base mainnet
const params = new URLSearchParams({
  chainId:    '8453',
  sellToken:  '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // USDC on Base
  buyToken:   '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE', // native ETH
  sellAmount: '100000',                                      // 0.1 USDC
})

// Price — no taker required
const priceRes = await x402Fetch(`${BASE_URL}/swap-allowance-holder-price/?${params}`)
const price = await priceRes.json()
console.log('buyAmount:', price.buyAmount)
console.log('liquidityAvailable:', price.liquidityAvailable)

// Quote — taker required
params.set('taker', account.address)
const quoteRes = await x402Fetch(`${BASE_URL}/swap-allowance-holder-quote/?${params}`)
const quote = await quoteRes.json()
console.log('transaction.to:', quote.transaction?.to)
console.log('transaction.gas:', quote.transaction?.gas)

// Read settlement from PAYMENT-RESPONSE header
const settlement = JSON.parse(
  Buffer.from(quoteRes.headers.get('PAYMENT-RESPONSE'), 'base64').toString()
)
console.log('tx hash:', settlement.transaction)
console.log('explorer:', `https://sepolia.basescan.org/tx/${settlement.transaction}`)
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

**`PAYMENT-RESPONSE` header** (base64-encoded JSON):
- `success` — whether payment settled
- `transaction` — on-chain tx hash
- `network` — chain where payment settled (`eip155:84532`)
- `payer` — wallet address that paid

> ⚠️ Quotes expire in ~30 seconds. Submit the transaction immediately after fetching.
>
> ⚠️ The query chain (`chainId=8453`, Base mainnet) and the payment chain (Base Sepolia) are independent. The 0x API does not support testnets for swap data — always query mainnet chain IDs.

## Execution (after getting a quote)

```javascript
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

Full working test script: [jlin27/agentpay-x402](https://github.com/jlin27/agentpay-x402)

## Related skills

- [`0x-api`](../0x-api/SKILL.md) — standard 0x integration with an API key
- [`0x-mpp`](../0x-mpp/SKILL.md) — same keyless endpoints, paying via MPP (pathUSD on Tempo Testnet)
