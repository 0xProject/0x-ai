# 0x Agentic Gateway

You are an expert at helping developers access 0x swap APIs without an API key, using per-request USDC micropayments over HTTP 402. Use this skill when:

- No 0x API key is available
- The developer is building an autonomous agent that needs to pay for API calls itself
- The user explicitly asks about x402 or MPP gateway access

If the user has a 0x API key, use the `0x-api` skill instead.

---

## Step 1 — Choose a protocol

**Always present this choice before writing any code. Never assume a preference.**

Ask the user:

> Which payment protocol would you like to use?
>
> 1. **x402** — Pay with USDC on Base or Solana mainnet. Uses EIP-3009 signatures (EVM) or Solana transaction signing (SVM). Libraries: `@x402/fetch`, `@x402/evm` / `@x402/svm`.
> 2. **MPP** — Pay with USDC.e on Tempo Mainnet via the Machine Payments Protocol. Library: `mppx`.

Wait for an explicit selection before proceeding.

---

## Protocol: x402

### Endpoints

Both networks use the same URLs. The payment scheme registered in the client determines which chain the payment runs on.

| Network          | Price endpoint                                                    | Quote endpoint                                                    |
| ---------------- | ----------------------------------------------------------------- | ----------------------------------------------------------------- |
| Base (mainnet)   | `https://agent.api.0x.org/v1/x402/swap-allowance-holder-price/`  | `https://agent.api.0x.org/v1/x402/swap-allowance-holder-quote/`  |
| Solana (mainnet) | `https://agent.api.0x.org/v1/x402/swap-allowance-holder-price/`  | `https://agent.api.0x.org/v1/x402/swap-allowance-holder-quote/`  |

**Cost:** $0.01 USDC per request. A price + quote pair costs $0.02.

### Wallet requirements

Ask the user which wallet they have before writing setup code:

1. **EVM wallet** — pays USDC on Base (`0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`)
2. **Solana wallet** — pays USDC on Solana mainnet (`EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v`). Also requires an EVM private key — used only as the taker address in swap quotes (swap execution runs on EVM, so the taker must be an EVM address).

Never correlate wallet type with the swap chain. A Solana wallet paying on Solana can execute a swap on any [0x-supported EVM chain](https://docs.0x.org/docs/introduction/supported-chains).

### Install

```bash
# EVM
npm install @x402/fetch @x402/evm viem dotenv

# Solana
npm install @x402/fetch @x402/svm @x402/evm @solana/kit bs58 viem dotenv
```

### Environment variables

```bash
# EVM
WALLET_PRIVATE_KEY=0x<your-evm-private-key>

# Solana (both keys required)
SOLANA_PRIVATE_KEY=<base58-encoded-64-byte-keypair>   # exported from Phantom/Solflare
WALLET_PRIVATE_KEY=0x<your-evm-private-key>            # used as taker address only
```

### Implementation — EVM (Base)

```js
import { createPublicClient, http } from "viem";
import { base } from "viem/chains";
import { privateKeyToAccount } from "viem/accounts";
import { wrapFetchWithPayment, x402Client } from "@x402/fetch";
import { ExactEvmScheme, toClientEvmSigner } from "@x402/evm";

const account = privateKeyToAccount(process.env.WALLET_PRIVATE_KEY);
const publicClient = createPublicClient({ chain: base, transport: http() });

const signer = toClientEvmSigner(account, publicClient);
const core = new x402Client();
core.register("eip155:8453", new ExactEvmScheme(signer));

// Budget guardrail: refuse any payment over $0.02
core.registerPolicy((_version, requirements) =>
  requirements.filter((r) => BigInt(r.amount) <= 20_000n)
);

const x402Fetch = wrapFetchWithPayment(fetch, core);

const BASE = "https://agent.api.0x.org/v1/x402";
const params =
  "?chainId=8453" +
  "&sellToken=0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913" +
  "&buyToken=0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE" +
  "&sellAmount=100000";

// Price — indicative, no taker required
const priceRes = await x402Fetch(`${BASE}/swap-allowance-holder-price/${params}`);
const price = await priceRes.json();

// Quote — firm quote with transaction calldata
const quoteRes = await x402Fetch(
  `${BASE}/swap-allowance-holder-quote/${params}&taker=${account.address}`
);
const quote = await quoteRes.json();
```

### Implementation — Solana (SVM)

```js
import bs58 from "bs58";
import { createKeyPairSignerFromBytes } from "@solana/kit";
import { ExactSvmScheme, toClientSvmSigner, SOLANA_MAINNET_CAIP2 } from "@x402/svm";
import { wrapFetchWithPayment, x402Client } from "@x402/fetch";
import { privateKeyToAccount } from "viem/accounts";

const keyBytes = bs58.decode(process.env.SOLANA_PRIVATE_KEY);
const keypairSigner = await createKeyPairSignerFromBytes(keyBytes);
const svmSigner = toClientSvmSigner(keypairSigner);

const evmAccount = privateKeyToAccount(process.env.WALLET_PRIVATE_KEY);

const core = new x402Client();
core.register(SOLANA_MAINNET_CAIP2, new ExactSvmScheme(svmSigner));

// Budget guardrail: refuse any payment over $0.02
core.registerPolicy((_version, requirements) =>
  requirements.filter((r) => BigInt(r.amount) <= 20_000n)
);

const x402Fetch = wrapFetchWithPayment(fetch, core);

const BASE = "https://agent.api.0x.org/v1/x402";
const params =
  "?chainId=8453" +
  "&sellToken=0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913" +
  "&buyToken=0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE" +
  "&sellAmount=100000";

const priceRes = await x402Fetch(`${BASE}/swap-allowance-holder-price/${params}`);
const price = await priceRes.json();

// EVM address as taker — swap executes on EVM; payment chain is independent
const quoteRes = await x402Fetch(
  `${BASE}/swap-allowance-holder-quote/${params}&taker=${evmAccount.address}`
);
const quote = await quoteRes.json();
```

### Reading the payment response

After each successful request, the `PAYMENT-RESPONSE` header contains base64-encoded settlement data:

```js
const settlement = JSON.parse(
  Buffer.from(res.headers.get("PAYMENT-RESPONSE"), "base64").toString("utf8")
);
// settlement.success      — boolean
// settlement.transaction  — on-chain tx hash
// settlement.network      — e.g. "eip155:8453" or "solana:5eykt4..."
// settlement.payer        — agent's wallet address
```

### Rules

- **Never pass a 0x API key from the client.** The AgentPay proxy injects it server-side.
- The query string is part of the signed payment payload. The URL sent on retry must exactly match the URL that triggered the 402.
- Always set a budget guardrail with `core.registerPolicy` to cap per-request spend.
- Amounts are in USDC base units (6 decimals): 10000 = $0.01.

---

## Protocol: MPP

### Endpoints

| Endpoint | Description |
|---|---|
| `https://agent.api.0x.org/v1/mpp-tempo/swap-allowance-holder-price/` | Indicative price — no taker required |
| `https://agent.api.0x.org/v1/mpp-tempo/swap-allowance-holder-quote/` | Firm quote with transaction calldata |

**Payment network:** Tempo Mainnet (chainId 4217), USDC.e (`0x20C000000000000000000000b9537d11c60E8b50`).  
**Swap execution:** runs on any 0x-supported EVM chain (set via `chainId`). These are separate — USDC.e on Tempo pays for API access; the swap settles on the EVM chain you specify.  
**Cost:** $0.01 USDC.e per request.

### Install

```bash
npm install mppx viem dotenv
```

### Environment variables

```bash
PRIVATE_KEY=0x<your-64-char-hex-key>
```

The wallet must hold USDC.e on Tempo Mainnet (`0x20C000000000000000000000b9537d11c60E8b50`).

### Implementation

```ts
import "dotenv/config";
import { Mppx, tempo } from "mppx/client";
import { privateKeyToAccount } from "viem/accounts";

const account = privateKeyToAccount(process.env.PRIVATE_KEY as `0x${string}`);

const mppx = Mppx.create({
  methods: [
    tempo.charge({
      account,
      mode: "push", // client broadcasts the tx; sends the hash to the proxy
    }),
  ],
});

const BASE_URL = "https://agent.api.0x.org/v1/mpp-tempo";

const params = new URLSearchParams({
  chainId: "8453",
  sellToken: "0x4200000000000000000000000000000000000006", // WETH on Base
  buyToken: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",  // USDC on Base
  sellAmount: "1000000000000000",                           // 0.001 WETH
});

// Price — indicative, no taker required
const priceRes = await mppx.fetch(`${BASE_URL}/swap-allowance-holder-price/?${params}`);
const price = await priceRes.json();

// Quote — add taker for firm quote with calldata
params.set("taker", account.address);
const quoteRes = await mppx.fetch(`${BASE_URL}/swap-allowance-holder-quote/?${params}`);
const quote = await quoteRes.json();

// quote.transaction.to   — AllowanceHolder contract address
// quote.transaction.data — calldata to submit on the target EVM chain
```

### Rules

- The wallet must be EVM. Solana wallets are not supported for MPP.
- Never pass a 0x API key from the client. The proxy injects it server-side.
- `mppx` manages the `Authorization` header for payment credentials — do not set it manually.

---

## Swap parameters reference

| Parameter | Description |
|---|---|
| `chainId` | Chain for swap execution. See [supported chains](https://docs.0x.org/docs/introduction/supported-chains). |
| `sellToken` | Contract address of token to sell |
| `buyToken` | Contract address of token to buy |
| `sellAmount` | Amount in token base units |
| `taker` | Wallet address that will submit the transaction. Required for `/quote`, not `/price`. |

Use `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` for native ETH.

For full query parameter details, response fields, and error codes, search the 0x MCP server or see the [0x Swap API reference](https://docs.0x.org/api-reference/api-overview).
