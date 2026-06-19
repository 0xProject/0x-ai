# 0x AI

Official [0x](https://docs.0x.org/home/home) AI tools for developers building token swaps and DeFi integrations.

## Quick Start

```bash
npx skills add 0xProject/0x-ai
```

Works with Claude Code, Cursor, GitHub Copilot, and other AI coding agents.

## Which skill should I use?

| What you're doing | Use this skill |
| --- | --- |
| Swap tokens and you have (or can create) a 0x API key | `0x-api` |
| Swap tokens without an API key, autonomous agent paying per-request, or you explicitly want x402/MPP | `0x-agentic-gateway` |

## Skills

### `skills/0x-api`

Step-by-step guide for executing token swaps using the 0x API (Swap API v2 and Gasless API v2). Covers AllowanceHolder, Permit2, and Gasless flows in TypeScript and Python across all supported EVM chains.

- **Auth:** API key via `0x-api-key` header
- **Setup:** sign up at [dashboard.0x.org](https://dashboard.0x.org/create-account)
- **Entry point:** [`skills/0x-api/SKILL.md`](skills/0x-api/SKILL.md)

### `skills/0x-agentic-gateway`

Specialized skill for accessing 0x swap APIs without an API key, using per-request USDC micropayments over HTTP 402. Covers both x402 (EVM and Solana wallets) and MPP (Tempo Mainnet) payment protocols.

- **Auth:** wallet-based payment — USDC on Base or Solana (x402) or USDC.e on Tempo Mainnet (MPP)
- **Protocols:** x402 (`@x402/fetch`, `@x402/evm` / `@x402/svm`) or MPP (`mppx`)
- **Setup:** fund a wallet with USDC (x402) or USDC.e (MPP); no API key needed
- **Entry point:** [`skills/0x-agentic-gateway/SKILL.md`](skills/0x-agentic-gateway/SKILL.md)

## MCP Server

The 0x MCP server gives your AI agent live access to 0x documentation and API references — so it can look up the latest endpoints, parameters, and code examples without relying on training data.

**Endpoint:** `https://docs.0x.org/_mcp/server`

Automatically configured when you install via `npx skills add 0xProject/0x-ai`. To add it manually:

```json
{
  "mcpServers": {
    "0x-mcp": {
      "type": "url",
      "url": "https://docs.0x.org/_mcp/server"
    }
  }
}
```

## Usage (after install)

Once installed, invoke in any session:
```
/0x-api
/0x-agentic-gateway
```

## Getting a 0x API Key

Sign up at https://dashboard.0x.org/create-account
