# 0x AI

Official 0x AI tools for developers building token swaps and DeFi integrations.

## Quick Start

```bash
npx skills add 0xProject/0x-ai
```

Works with Claude Code, Cursor, GitHub Copilot, and 30+ other AI coding agents.

## Skills

### 0x-swap

Step-by-step guide for executing token swaps using the 0x Protocol API (Swap API v2 and Gasless API v2).

**Use this skill when you want to:**
- Swap tokens on any EVM chain (Ethereum, Base, Arbitrum, Polygon, Optimism, BNB Chain)
- Do a gasless swap without holding ETH for gas
- Integrate 0x into a dApp in TypeScript or Python
- Use 0x with a Gnosis Safe or multisig wallet
- Migrate from 0x Swap v1 to v2
- Debug 0x API errors

**Supported flows:**
- AllowanceHolder: recommended — simple approve + send tx, works with multisigs
- Permit2: advanced — time-limited approvals, EIP-712 signing
- Gasless API: no native token (e.g., ETH) needed, fee deducted from sell tokens

## Manual Installation (Claude Code)

```bash
/plugin install 0x-swap@claude-plugins-official
```

Or invoke directly in any session:
```
/0x-swap
```

## Getting a 0x API Key

Sign up at https://dashboard.0x.org/create-account
