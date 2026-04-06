# 0x AI

Official [0x](https://docs.0x.org/home/home) AI tools for developers building token swaps and DeFi integrations.

## Quick Start

```bash
npx skills add 0xProject/0x-ai
```

Works with Claude Code, Cursor, GitHub Copilot, and other AI coding agents.

## Skills

### 0x-api

Step-by-step guide for executing token swaps using the 0x API (Swap API v2 and Gasless API v2).

**Use this skill when you want to:**
- Swap tokens on all [supported EVM chains](https://docs.0x.org/docs/introduction/supported-chains)
- Do a gasless swap without holding ETH for gas
- Integrate 0x into a dApp in TypeScript or Python
- Use 0x with a Gnosis Safe or multisig wallet
- Migrate from 0x Swap v1 to v2
- Debug 0x API errors

**Supported flows:**
- AllowanceHolder: recommended — simple approve + send tx, works with multisigs
- Permit2: advanced — time-limited approvals, EIP-712 signing
- Gasless API: no native token (e.g., ETH) needed, fee deducted from sell tokens

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
```

## Getting a 0x API Key

Sign up at https://dashboard.0x.org/create-account
