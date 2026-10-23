# JPay — Technical Execution Plan (`PLAN.md`)

> **Product:** JPay — *Just Pay with Monad* · Web3-to-QRIS payment gateway
> **Source of truth:** `PRD.md` (v0.1) · **Event:** Monad Blitz, Yogyakarta · **Budget:** 8 hours
> **Audience:** an AI Coding Agent (Cursor / OpenCode / Claude Code). Execute phases **in order**.

---

## 0. Agent Operating Rules (READ FIRST)

1. `PRD.md` defines *what* to build. This file defines *how*. Where this file deviates from the PRD, the deviation is listed in **§0.2** and is binding.
2. Do the phases strictly in order. **Do not start Phase N+1 until Phase N's verification checklist passes.**
3. Do **not** swap libraries, add frameworks, or add out-of-scope features (KYC, multichain, merchant dashboard, native app, MEV protection, real PJSP settlement).
4. Never commit secrets. `.env.local` is git-ignored. `DEPLOYER_PRIVATE_KEY` is **server/script only** (never prefix with `NEXT_PUBLIC_`). Use a throwaway testnet key.
5. **Money rule:** token amounts are `bigint` in code and **decimal strings** in JSON. Never use floating point for wei/USDC units. IDR is an integer (Rupiah has no minor unit in practice).
6. Anything touching `window`, camera, `sessionStorage`, wagmi or Privy hooks must live in a `"use client"` module.
7. If blocked >15 minutes on one problem, apply the fallback written next to that step and move on. The demo must work end-to-end more than it must be elegant.
8. After each phase: run the verification commands, tick the checkboxes, then `git add -A && git commit -m "phase N: ..."`.

### 0.1 Fixed Technical Decisions

| Concern | Decision |
|---|---|
| Framework | Next.js **14** App Router, TypeScript, Tailwind CSS **3** (what `create-next-app@14` installs) |
| Contracts toolchain | **Hardhat 2** + `@nomicfoundation/hardhat-toolbox` (ethers v6) + OpenZeppelin v5. Solidity `0.8.28`, `evmVersion: "cancun"` |
| Frontend web3 | `wagmi@2` + `viem@2` + `@privy-io/wagmi` (Privy ↔ wagmi bridge) + `@tanstack/react-query@5` |
| `ethers` usage | **Scripts/tests only** (deploy, listener helpers). Frontend + API routes use **viem** exclusively. |
| Unit tests (TS) | `vitest` in `__tests__/` · Contract tests: Hardhat/mocha in `test/` |
| Network | Monad Testnet — chain id `10143` (**verify against docs.monad.xyz before Hour 1; update `.env` if it changed**) |
| QR scanning | `html5-qrcode`, dynamically imported client-side only |
| Hosting | Vercel (serverless API routes) |

### 0.2 Deviations / Clarifications vs. PRD (binding)

| # | PRD says | Reality / Decision |
|---|---|---|
| D1 | F5: backend "listens" for `PaymentExecuted` | Vercel serverless cannot hold a persistent listener. **Primary path:** client calls `POST /api/settlement {txHash}` right after confirmation; the route fetches the receipt, decodes the `PaymentExecuted` log, validates it, and returns mock `SUCCESS`. **Optional demo path:** `scripts/settlement-listener.ts` (long-running `watchContractEvent`) prints a "merchant sees Rupiah" terminal for the pitch. |
| D2 | F1: "Embedded wallet auto-funds" | Not possible on testnet. Provide a **Fund Wallet helper** (show address, faucet link, one-click **mint MockUSDC**). Pre-fund demo wallets before judging. |
| D3 | USDC on Monad Testnet | No canonical USDC. Deploy **`MockUSDC`** (6 decimals, public `mint`) alongside the gateway; set as the `usdc` token. |
| D4 | Event `PaymentExecuted(...)` | Signature kept exactly; `txRef` is additionally marked `indexed` (does not change the PRD arg list) and the contract enforces **unique `txRef`** (replay/idempotency guard). `txRef` = server-issued `quoteId`. |
| D5 | Receipt shows merchant name | Merchant **name is not on-chain** (only NMID). Pass it to the receipt via `?m=` query param + `sessionStorage`; everything else is decoded from the chain. |
| D6 | Quote integrity | The contract cannot verify the quote (out of scope, PRD §5.2). The 30 s TTL is enforced **client-side + server timestamp**. Document this as a known MVP limitation in the pitch Q&A. |
| D7 | — | Monad charges **gas limit, not gas used**. Never hard-code large `gas` values; let wagmi/viem estimate. |

---

## 1. Repository Setup & Project Structure

### 1.1 Bootstrap

```bash
npx create-next-app@14 jpay --typescript --tailwind --app --eslint --no-src-dir --import-alias "@/*"
cd jpay

# runtime deps
npm install @privy-io/react-auth@^2 @privy-io/wagmi@^1 wagmi@^2 viem@^2 @tanstack/react-query@^5 ethers@^6 html5-qrcode clsx

# contracts + tooling
npm install -D hardhat@^2.22 @nomicfoundation/hardhat-toolbox@^5 @openzeppelin/contracts@^5 dotenv cross-env tsx vitest@^2
```

> If npm reports peer-dependency conflicts between `@privy-io/react-auth`, `@privy-io/wagmi`, and `wagmi`, read the Privy "wagmi" integration docs and pin the exact compatible majors. Only use `--legacy-peer-deps` as a last resort.

### 1.2 `package.json` (target shape)

```json
{
  "name": "jpay",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "dev:https": "next dev --experimental-https",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "compile": "cross-env TS_NODE_PROJECT=tsconfig.hardhat.json hardhat compile",
    "test:contracts": "cross-env TS_NODE_PROJECT=tsconfig.hardhat.json hardhat test",
    "deploy:testnet": "cross-env TS_NODE_PROJECT=tsconfig.hardhat.json hardhat run scripts/deploy.ts --network monadTestnet",
    "verify:testnet": "cross-env TS_NODE_PROJECT=tsconfig.hardhat.json hardhat verify --network monadTestnet",
    "listener": "tsx scripts/settlement-listener.ts"
  },
  "dependencies": {
    "@privy-io/react-auth": "^2.0.0",
    "@privy-io/wagmi": "^1.0.0",
    "@tanstack/react-query": "^5.59.0",
    "clsx": "^2.1.1",
    "ethers": "^6.13.0",
    "html5-qrcode": "^2.3.8",
    "next": "^14.2.0",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "viem": "^2.21.0",
    "wagmi": "^2.12.0"
  },
  "devDependencies": {
    "@nomicfoundation/hardhat-toolbox": "^5.0.0",
    "@openzeppelin/contracts": "^5.0.2",
    "@types/node": "^20",
    "@types/react": "^18",
    "@types/react-dom": "^18",
    "cross-env": "^7.0.3",
    "dotenv": "^16.4.5",
    "eslint": "^8",
    "eslint-config-next": "^14.2.0",
    "hardhat": "^2.22.0",
    "postcss": "^8",
    "tailwindcss": "^3.4.1",
    "tsx": "^4.19.0",
    "typescript": "^5",
    "vitest": "^2.1.0"
  }
}
```

### 1.3 Exact Directory Tree

```
jpay/
├── PRD.md
├── PLAN.md
├── .env.example
├── .env.local                      # git-ignored, created from .env.example
├── .gitignore                      # + artifacts/ cache/ typechain-types/ .env.local
├── hardhat.config.ts
├── next.config.mjs
├── tsconfig.json                   # Next config (excludes contracts tooling)
├── tsconfig.hardhat.json           # CommonJS config used ONLY by hardhat scripts
├── vitest.config.ts
├── package.json
│
├── contracts/                      # Solidity sources (Hardhat sources dir)
│   ├── JPayGateway.sol
│   └── mocks/
│       └── MockUSDC.sol
├── scripts/
│   ├── deploy.ts                   # deploys + writes lib/contracts/config.ts
│   └── settlement-listener.ts      # optional live "merchant terminal" for the pitch
├── test/                           # Hardhat (mocha/chai) contract tests
│   └── JPayGateway.test.ts
├── __tests__/                      # Vitest unit tests
│   ├── qris-parser.test.ts
│   └── pricing.test.ts
│
├── app/
│   ├── layout.tsx
│   ├── globals.css
│   ├── providers.tsx               # Privy + React Query + wagmi
│   ├── page.tsx                    # Landing / Connect / Scan trigger
│   ├── scan/page.tsx               # Live scanner + manual IDR entry modal
│   ├── pay/page.tsx                # Quote confirm + sign + pending
│   ├── receipt/[txHash]/page.tsx   # Digital receipt
│   └── api/
│       ├── quote/route.ts          # IDR -> USD -> MON/USDC
│       └── settlement/route.ts     # Verify event + mock IDR settlement
│
├── components/
│   ├── header.tsx
│   ├── connect-button.tsx
│   ├── qris-scanner.tsx
│   ├── manual-amount-modal.tsx
│   ├── quote-card.tsx
│   ├── quote-countdown.tsx
│   ├── token-toggle.tsx
│   ├── pay-button.tsx
│   ├── tx-progress.tsx
│   ├── receipt-card.tsx
│   ├── settlement-badge.tsx
│   ├── fund-wallet-helper.tsx
│   └── ui/
│       ├── button.tsx
│       ├── card.tsx
│       ├── badge.tsx
│       └── spinner.tsx
│
├── hooks/
│   ├── use-jpay-contract.ts        # approve + payWithMON / payWithUSDC
│   ├── use-quote.ts                # fetch + TTL countdown + auto-refresh
│   └── use-payment-session.ts      # sessionStorage-backed flow state
│
├── lib/
│   ├── chains.ts                   # monadTestnet (defineChain)
│   ├── env.ts                      # typed public env access
│   ├── pricing.ts                  # pure bigint quote math (unit-tested)
│   ├── rates.ts                    # live/fallback rate fetchers (server only)
│   ├── session.ts                  # sessionStorage helpers
│   ├── contracts/
│   │   └── config.ts               # GENERATED by scripts/deploy.ts (address + ABI)
│   └── qris/
│       ├── crc16.ts
│       ├── parser.ts               # EMVCo TLV decoder
│       └── samples.ts              # test vectors (also used by dev "paste QR" input)
│
├── types/
│   ├── qris.ts
│   ├── quote.ts
│   └── settlement.ts
│
└── utils/
    ├── format.ts                   # IDR / token / address / hash formatters
    └── cn.ts                       # clsx wrapper
```

### 1.4 Config Files

**`tsconfig.json`** — keep the generated file but add `exclude` so `next build` never type-checks Hardhat code:

```json
{
  "compilerOptions": {
    "target": "es2020",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": { "@/*": ["./*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules", "contracts", "scripts", "test", "hardhat.config.ts", "typechain-types", "artifacts", "cache"]
}
```

**`tsconfig.hardhat.json`** (Hardhat's ts-node needs CommonJS):

```json
{
  "compilerOptions": {
    "target": "es2020",
    "module": "commonjs",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "strict": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["./hardhat.config.ts", "./scripts", "./test", "./typechain-types"]
}
```

**`hardhat.config.ts`**

```ts
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";
import * as dotenv from "dotenv";

dotenv.config({ path: ".env.local" });
dotenv.config();

const PK = process.env.DEPLOYER_PRIVATE_KEY; // must be 0x-prefixed

const config: HardhatUserConfig = {
  solidity: {
    version: "0.8.28",
    settings: { optimizer: { enabled: true, runs: 200 }, evmVersion: "cancun" },
  },
  networks: {
    hardhat: {},
    monadTestnet: {
      url: process.env.MONAD_RPC_URL || "https://testnet-rpc.monad.xyz",
      chainId: Number(process.env.NEXT_PUBLIC_CHAIN_ID || 10143),
      accounts: PK ? [PK] : [],
    },
  },
  // Monad explorer verification goes through Sourcify (verify against docs.monad.xyz if this fails)
  sourcify: {
    enabled: true,
    apiUrl: "https://sourcify-api-monad.blockscout.com",
    browserUrl: "https://testnet.monadexplorer.com",
  },
  etherscan: { enabled: false },
  paths: { sources: "./contracts", tests: "./test", cache: "./cache", artifacts: "./artifacts" },
};

export default config;
```

**`vitest.config.ts`**

```ts
import { defineConfig } from "vitest/config";
import path from "node:path";

export default defineConfig({
  test: { include: ["__tests__/**/*.test.ts"], environment: "node" },
  resolve: { alias: { "@": path.resolve(__dirname, ".") } },
});
```

**`next.config.mjs`**

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  webpack: (config) => {
    // silence optional deps pulled in by wallet libs
    config.externals.push("pino-pretty", "lokijs", "encoding");
    return config;
  },
};
export default nextConfig;
```

### 1.5 `.env.example`

```bash
# ───────── Public (exposed to browser) ─────────
NEXT_PUBLIC_PRIVY_APP_ID=                 # Privy dashboard → App ID
NEXT_PUBLIC_CHAIN_ID=10143                # Monad Testnet (verify in Monad docs)
NEXT_PUBLIC_MONAD_RPC_URL=https://testnet-rpc.monad.xyz
NEXT_PUBLIC_EXPLORER_URL=https://testnet.monadexplorer.com
NEXT_PUBLIC_JPAY_CONTRACT_ADDRESS=        # optional override; deploy script writes lib/contracts/config.ts
NEXT_PUBLIC_USDC_ADDRESS=                 # optional override (MockUSDC by default)
NEXT_PUBLIC_ENABLE_USDC=true              # set false to ship MON-only (Hour-4 fallback)
NEXT_PUBLIC_FAUCET_URL=                   # Monad testnet faucet link for the fund helper

# ───────── Server only ─────────
MONAD_RPC_URL=https://testnet-rpc.monad.xyz
TREASURY_ADDRESS=0x...                    # JPay settlement treasury (receives MON/USDC)
DEPLOYER_PRIVATE_KEY=0x...                # THROWAWAY testnet key. NEVER NEXT_PUBLIC_.
USDC_ADDRESS=                             # leave empty → deploy script deploys MockUSDC

# Pricing / quote engine
QUOTE_TTL_SECONDS=30
FORCE_MOCK_RATES=false                    # true → deterministic demo rates, no network calls
MOCK_IDR_PER_USD=16000
MOCK_MON_USD=0.05                         # PLACEHOLDER — tune so demo payments are affordable from faucet funds
```

### 1.6 Shared Types

**`types/qris.ts`**

```ts
export interface ParsedQRIS {
  merchantNMID: string;          // Tag 51 → subtag 02
  merchantName: string;          // Tag 59
  amountIDR: number | null;      // Tag 54; null if static / open-amount
  merchantCity?: string;         // Tag 60
  isStatic: boolean;             // Tag 01 === "11"
  raw: string;
}

export type QRISErrorCode =
  | "NOT_QRIS"
  | "MALFORMED_TLV"
  | "BAD_CRC"
  | "MISSING_NMID"
  | "INVALID_AMOUNT";
```

**`types/quote.ts`**

```ts
export type PayToken = "MON" | "USDC";

export interface QuoteRequest {
  amountIDR: number;             // integer, 1..20_000_000
}

export interface Quote {
  quoteId: `0x${string}`;        // bytes32, used as on-chain txRef
  amountIDR: number;
  rates: { idrPerUsd: number; monUsd: number; usdcUsd: number };
  rateSource: "live" | "partial-live" | "mock";
  mon: { amountWei: string; display: string };      // 18 decimals
  usdc: { amountUnits: string; display: string };   // 6 decimals
  issuedAt: number;              // ms epoch
  expiresAt: number;             // ms epoch (issuedAt + ttl)
  ttlSeconds: number;
}
```

**`types/settlement.ts`**

```ts
export type SettlementStatus = "PENDING" | "SUCCESS" | "FAILED";

export interface SettlementResponse {
  status: SettlementStatus;
  txHash: `0x${string}`;
  settlementRef?: string;        // e.g. "JPAY-3F9A21C0B7"
  settledAt?: string;            // ISO
  latencyMs: number;
  mock: true;
  error?: string;
  event?: {
    payer: `0x${string}`;
    merchantNMID: string;
    amountToken: string;         // wei / USDC units as decimal string
    amountIDR: string;
    isNative: boolean;
    txRef: `0x${string}`;
  };
}
```

### 1.7 Chain Definition — `lib/chains.ts`

```ts
import { defineChain } from "viem";

const rpc = process.env.NEXT_PUBLIC_MONAD_RPC_URL || "https://testnet-rpc.monad.xyz";
const explorer = process.env.NEXT_PUBLIC_EXPLORER_URL || "https://testnet.monadexplorer.com";

export const monadTestnet = defineChain({
  id: Number(process.env.NEXT_PUBLIC_CHAIN_ID || 10143),
  name: "Monad Testnet",
  nativeCurrency: { name: "Monad", symbol: "MON", decimals: 18 },
  rpcUrls: { default: { http: [rpc] } },
  blockExplorers: { default: { name: "Monad Explorer", url: explorer } },
  testnet: true,
});

export const explorerTxUrl = (hash: string) => `${explorer}/tx/${hash}`;
export const explorerAddressUrl = (addr: string) => `${explorer}/address/${addr}`;
```

### 1.8 Phase 0 Verification

- [ ] `npm run dev` serves the default page at `http://localhost:3000`
- [ ] `npm run typecheck` passes
- [ ] `npm run build` passes
- [ ] `.env.local` created from `.env.example` with a real `NEXT_PUBLIC_PRIVY_APP_ID`
- [ ] Privy dashboard: enabled Email + Google + Wallet login; added `http://localhost:3000` (and later the Vercel domain) to **Allowed origins**
- [ ] Throwaway wallet funded from the Monad testnet faucet
- [ ] Stub `lib/contracts/config.ts` committed (zero addresses, empty ABI) so imports compile before Phase 1

---

## 2. Phase 1 — Smart Contracts (`JPayGateway.sol`) · Hour 1–2

### 2.1 `contracts/JPayGateway.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title JPayGateway
/// @notice Accepts MON/USDC from customers and records QRIS-linked payments on Monad.
/// @dev MVP: funds are forwarded straight to the JPay treasury; IDR payout is off-chain (mocked).
contract JPayGateway is ReentrancyGuard {
    using SafeERC20 for IERC20;

    address public immutable treasury;
    IERC20 public immutable usdc;

    /// @notice txRef values already consumed (idempotency / replay guard)
    mapping(bytes32 => bool) public usedRefs;

    event PaymentExecuted(
        address indexed payer,
        string merchantNMID,
        uint256 amountToken,
        uint256 amountIDR,
        bool isNative,
        bytes32 indexed txRef
    );

    constructor(address _treasury, address _usdc) {
        require(_treasury != address(0), "JPay: zero treasury");
        require(_usdc != address(0), "JPay: zero usdc");
        treasury = _treasury;
        usdc = IERC20(_usdc);
    }

    /// @notice Pay merchant using native $MON
    function payWithMON(string calldata merchantNMID, uint256 amountIDR, bytes32 txRef)
        external
        payable
        nonReentrant
    {
        require(msg.value > 0, "JPay: zero payment");
        require(amountIDR > 0, "JPay: zero IDR");
        require(bytes(merchantNMID).length > 0, "JPay: empty NMID");
        _consumeRef(txRef);

        (bool sent, ) = treasury.call{value: msg.value}("");
        require(sent, "JPay: MON transfer failed");

        emit PaymentExecuted(msg.sender, merchantNMID, msg.value, amountIDR, true, txRef);
    }

    /// @notice Pay merchant using USDC (caller must have approved this contract)
    function payWithUSDC(
        string calldata merchantNMID,
        uint256 amountUSDC,
        uint256 amountIDR,
        bytes32 txRef
    ) external nonReentrant {
        require(amountUSDC > 0, "JPay: zero payment");
        require(amountIDR > 0, "JPay: zero IDR");
        require(bytes(merchantNMID).length > 0, "JPay: empty NMID");
        _consumeRef(txRef);

        usdc.safeTransferFrom(msg.sender, treasury, amountUSDC); // reverts on insufficient balance/allowance

        emit PaymentExecuted(msg.sender, merchantNMID, amountUSDC, amountIDR, false, txRef);
    }

    function _consumeRef(bytes32 txRef) private {
        require(txRef != bytes32(0), "JPay: empty ref");
        require(!usedRefs[txRef], "JPay: ref already used");
        usedRefs[txRef] = true;
    }
}
```

### 2.2 `contracts/mocks/MockUSDC.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

/// @notice TESTNET ONLY. Public-mint 6-decimal stand-in for USDC.
contract MockUSDC is ERC20 {
    constructor() ERC20("Mock USDC", "USDC") {}

    function decimals() public pure override returns (uint8) {
        return 6;
    }

    /// @dev capped per call so a demo wallet can't be accidentally flooded
    function mint(address to, uint256 amount) external {
        require(amount <= 1_000e6, "MockUSDC: max 1000 per mint");
        _mint(to, amount);
    }
}
```

### 2.3 `test/JPayGateway.test.ts`

```ts
import { expect } from "chai";
import { ethers } from "hardhat";
import { loadFixture } from "@nomicfoundation/hardhat-toolbox/network-helpers";

const NMID = "ID1026400000001";
const ref = (n: number) => ethers.zeroPadValue(ethers.toBeHex(n), 32);

async function deploy() {
  const [payer, treasury, other] = await ethers.getSigners();
  const usdc = await (await ethers.getContractFactory("MockUSDC")).deploy();
  const gw = await (await ethers.getContractFactory("JPayGateway")).deploy(treasury.address, await usdc.getAddress());
  return { gw, usdc, payer, treasury, other };
}

describe("JPayGateway", () => {
  it("payWithMON forwards MON to treasury and emits PaymentExecuted", async () => {
    const { gw, payer, treasury } = await loadFixture(deploy);
    const v = ethers.parseEther("31.25");
    await expect(gw.connect(payer).payWithMON(NMID, 25000n, ref(1), { value: v }))
      .to.emit(gw, "PaymentExecuted")
      .withArgs(payer.address, NMID, v, 25000n, true, ref(1));
    await expect(gw.connect(payer).payWithMON(NMID, 25000n, ref(2), { value: v })).to.changeEtherBalance(treasury, v);
  });

  it("reverts on zero value, zero IDR, empty NMID", async () => {
    const { gw } = await loadFixture(deploy);
    await expect(gw.payWithMON(NMID, 1n, ref(1), { value: 0 })).to.be.revertedWith("JPay: zero payment");
    await expect(gw.payWithMON(NMID, 0n, ref(1), { value: 1n })).to.be.revertedWith("JPay: zero IDR");
    await expect(gw.payWithMON("", 1n, ref(1), { value: 1n })).to.be.revertedWith("JPay: empty NMID");
  });

  it("rejects a reused txRef", async () => {
    const { gw } = await loadFixture(deploy);
    await gw.payWithMON(NMID, 1n, ref(7), { value: 1n });
    await expect(gw.payWithMON(NMID, 1n, ref(7), { value: 1n })).to.be.revertedWith("JPay: ref already used");
  });

  it("payWithUSDC pulls approved USDC to treasury and emits", async () => {
    const { gw, usdc, payer, treasury } = await loadFixture(deploy);
    const amt = 1_562_500n; // 1.5625 USDC
    await usdc.mint(payer.address, amt);
    await usdc.approve(await gw.getAddress(), amt);
    await expect(gw.payWithUSDC(NMID, amt, 25000n, ref(3)))
      .to.emit(gw, "PaymentExecuted")
      .withArgs(payer.address, NMID, amt, 25000n, false, ref(3));
    expect(await usdc.balanceOf(treasury.address)).to.equal(amt);
  });

  it("payWithUSDC reverts without allowance / balance", async () => {
    const { gw, usdc, payer } = await loadFixture(deploy);
    await usdc.mint(payer.address, 100n);
    await expect(gw.payWithUSDC(NMID, 100n, 1n, ref(4))).to.be.reverted; // no allowance
    await expect(gw.payWithUSDC(NMID, 0n, 1n, ref(5))).to.be.revertedWith("JPay: zero payment");
  });
});
```

### 2.4 `scripts/deploy.ts` (deploys, then **generates `lib/contracts/config.ts`**)

```ts
import { ethers, artifacts, network } from "hardhat";
import fs from "node:fs";
import path from "node:path";

async function main() {
  const [deployer] = await ethers.getSigners();
  const treasury = process.env.TREASURY_ADDRESS || deployer.address;
  console.log(`Network: ${network.name} | Deployer: ${deployer.address} | Treasury: ${treasury}`);

  // 1) USDC: use provided address, else deploy MockUSDC
  let usdcAddress = process.env.USDC_ADDRESS;
  if (!usdcAddress) {
    const usdc = await (await ethers.getContractFactory("MockUSDC")).deploy();
    await usdc.waitForDeployment();
    usdcAddress = await usdc.getAddress();
    console.log("MockUSDC deployed:", usdcAddress);
  }

  // 2) Gateway
  const gw = await (await ethers.getContractFactory("JPayGateway")).deploy(treasury, usdcAddress);
  await gw.waitForDeployment();
  const gwAddress = await gw.getAddress();
  const deployTx = gw.deploymentTransaction();
  const receipt = await deployTx?.wait();
  console.log("JPayGateway deployed:", gwAddress);

  // 3) Generate lib/contracts/config.ts
  const gwAbi = (await artifacts.readArtifact("JPayGateway")).abi;
  const erc20Abi = (await artifacts.readArtifact("MockUSDC")).abi; // includes approve/allowance/balanceOf/decimals/mint

  const out = `// AUTO-GENERATED by scripts/deploy.ts — DO NOT EDIT BY HAND
export const DEPLOYMENT = {
  chainId: ${network.config.chainId},
  jpayGateway: "${gwAddress}",
  usdc: "${usdcAddress}",
  treasury: "${treasury}",
  deployedAtBlock: ${receipt?.blockNumber ?? 0},
  deployTxHash: "${deployTx?.hash ?? ""}",
} as const;

export const JPAY_GATEWAY_ADDRESS = (process.env.NEXT_PUBLIC_JPAY_CONTRACT_ADDRESS || DEPLOYMENT.jpayGateway) as \`0x\${string}\`;
export const USDC_ADDRESS = (process.env.NEXT_PUBLIC_USDC_ADDRESS || DEPLOYMENT.usdc) as \`0x\${string}\`;

export const JPAY_GATEWAY_ABI = ${JSON.stringify(gwAbi, null, 2)} as const;

export const ERC20_ABI = ${JSON.stringify(erc20Abi, null, 2)} as const;
`;
  const target = path.join(__dirname, "..", "lib", "contracts", "config.ts");
  fs.mkdirSync(path.dirname(target), { recursive: true });
  fs.writeFileSync(target, out);
  console.log("Wrote", target);

  console.log("\nVerify with:");
  console.log(`  npm run verify:testnet -- ${gwAddress} ${treasury} ${usdcAddress}`);
}

main().catch((e) => { console.error(e); process.exit(1); });
```

### 2.5 Compile, Test, Deploy, Verify

```bash
npm run compile                      # compiles contracts/*.sol
npm run test:contracts               # all tests green
npm run deploy:testnet               # deploys + writes lib/contracts/config.ts
npm run verify:testnet -- <GATEWAY> <TREASURY> <USDC>
# if MockUSDC was deployed by the script and you want it verified too:
npm run verify:testnet -- <MOCK_USDC>
```

> **Fallback:** if explorer verification fails after 15 min, skip it and note the contract address + tx link in the README; revisit in Hour 7. Contract *deployment* is the hard requirement.

### 2.6 Phase 1 Verification

- [ ] `npm run test:contracts` → all passing
- [ ] `lib/contracts/config.ts` regenerated with real addresses and full ABI
- [ ] Contract visible on the Monad testnet explorer (`<EXPLORER>/address/<GATEWAY>`)
- [ ] Manual smoke test: call `payWithMON` once from the explorer or a small script; `PaymentExecuted` visible in logs
- [ ] `npm run typecheck` still passes with the generated config
- [ ] Commit `config.ts` (addresses are public) — **never** commit `.env.local`

---

## 3. Phase 2 — EMVCo QRIS Parser & Camera Scanner · Hour 2–3

### 3.1 Format Cheat-Sheet (what the parser must understand)

EMVCo merchant-presented QR = concatenated **TLV**: `TT` (2-digit tag) + `LL` (2-digit decimal length) + `VV` (value of `LL` chars). Some tags (26–51, 62) contain **nested TLV**.

| Tag | Meaning | Use |
|---|---|---|
| `00` | Payload format (`01`) | Must be first; used to reject non-QRIS |
| `01` | Initiation method: `11` static, `12` dynamic | `isStatic` |
| `26`–`45` | Merchant account info (acquirer-specific) | NMID **fallback** only |
| **`51`** | National QRIS merchant info (`00`=`ID.CO.QRIS.WWW`, **`02`=NMID**, `03`=criteria) | **NMID** |
| `53` | Currency (`360` = IDR) | sanity check |
| **`54`** | Transaction amount (may be `25000` or `25000.00`) | **amountIDR** (absent ⇒ static/open) |
| `58` | Country (`ID`) | sanity check |
| **`59`** | Merchant name | **merchantName** |
| `60` | Merchant city | optional display |
| `63` | CRC16-CCITT-FALSE (poly `0x1021`, init `0xFFFF`) of everything up to and **including** `6304` | integrity |

### 3.2 `lib/qris/crc16.ts`

```ts
/** CRC-16/CCITT-FALSE (poly 0x1021, init 0xFFFF). Returns 4-char uppercase hex. */
export function crc16(input: string): string {
  let crc = 0xffff;
  for (let i = 0; i < input.length; i++) {
    crc ^= input.charCodeAt(i) << 8;
    for (let b = 0; b < 8; b++) {
      crc = crc & 0x8000 ? ((crc << 1) ^ 0x1021) & 0xffff : (crc << 1) & 0xffff;
    }
  }
  return crc.toString(16).toUpperCase().padStart(4, "0");
}
```

### 3.3 `lib/qris/parser.ts`

```ts
import { crc16 } from "./crc16";
import type { ParsedQRIS, QRISErrorCode } from "@/types/qris";

export class QRISParseError extends Error {
  constructor(public code: QRISErrorCode, message: string) {
    super(message);
    this.name = "QRISParseError";
  }
}

/** Walk a flat TLV string into { tag: value }. Throws MALFORMED_TLV on any structural problem. */
export function decodeTLV(payload: string): Record<string, string> {
  const out: Record<string, string> = {};
  let i = 0;
  while (i < payload.length) {
    if (i + 4 > payload.length) {
      throw new QRISParseError("MALFORMED_TLV", `Truncated TLV header at offset ${i}`);
    }
    const tag = payload.slice(i, i + 2);
    const lenStr = payload.slice(i + 2, i + 4);
    if (!/^\d{2}$/.test(tag) || !/^\d{2}$/.test(lenStr)) {
      throw new QRISParseError("MALFORMED_TLV", `Non-numeric tag/length at offset ${i}`);
    }
    const len = parseInt(lenStr, 10);
    const start = i + 4;
    const end = start + len;
    if (end > payload.length) {
      throw new QRISParseError("MALFORMED_TLV", `Tag ${tag} length ${len} overruns payload`);
    }
    out[tag] = payload.slice(start, end);
    i = end;
  }
  return out;
}

export function verifyCRC(raw: string): boolean {
  if (raw.length < 8 || raw.slice(-8, -4) !== "6304") return false;
  return crc16(raw.slice(0, -4)) === raw.slice(-4).toUpperCase();
}

function extractNMID(root: Record<string, string>): string {
  // Primary: Tag 51 → subtag 02
  if (root["51"]) {
    try {
      const nmid = decodeTLV(root["51"])["02"];
      if (nmid) return nmid;
    } catch { /* fall through */ }
  }
  // Fallback: first account-info tag (26–45) with a subtag 02
  for (let t = 26; t <= 45; t++) {
    const v = root[String(t)];
    if (!v) continue;
    try {
      const nmid = decodeTLV(v)["02"];
      if (nmid) return nmid;
    } catch { /* keep looking */ }
  }
  throw new QRISParseError("MISSING_NMID", "No NMID found (Tag 51 subtag 02)");
}

export interface ParseOptions {
  /** Reject payloads with a bad CRC. Default true. Set false only for debugging. */
  strictCRC?: boolean;
}

export function parseEMVCoQRIS(raw: string, opts: ParseOptions = {}): ParsedQRIS {
  const { strictCRC = true } = opts;
  const payload = (raw ?? "").trim();

  if (!payload.startsWith("000201")) {
    throw new QRISParseError("NOT_QRIS", "Not an EMVCo QR payload");
  }
  if (strictCRC && !verifyCRC(payload)) {
    throw new QRISParseError("BAD_CRC", "CRC16 check failed — QR may be damaged or tampered");
  }

  const root = decodeTLV(payload);
  const merchantNMID = extractNMID(root);
  const merchantName = (root["59"] ?? "").trim() || "Unknown Merchant";

  let amountIDR: number | null = null;
  if (root["54"] !== undefined) {
    const n = Number(root["54"]);
    if (!Number.isFinite(n) || n <= 0) {
      throw new QRISParseError("INVALID_AMOUNT", `Invalid amount "${root["54"]}"`);
    }
    amountIDR = Math.round(n); // "25000.00" -> 25000
  }

  return {
    merchantNMID,
    merchantName,
    amountIDR,
    merchantCity: root["60"]?.trim() || undefined,
    isStatic: root["01"] === "11",
    raw: payload,
  };
}
```

### 3.4 `lib/qris/samples.ts` — valid test vectors (CRCs pre-computed)

```ts
export const QRIS_SAMPLES = {
  // static, no Tag 54 → app must ask for the amount
  STATIC:
    "00020101021126570015COM.EXAMPLE.WWW0117936000000000000010206EX00010303UMI51440014ID.CO.QRIS.WWW0215ID10264000000010303UMI5204581253033605802ID5914WARUNG BU SARI6010YOGYAKARTA61055528162070703A016304704F",
  // dynamic, Rp 25.000
  DYNAMIC_25000:
    "00020101021226570015COM.EXAMPLE.WWW0117936000000000000010206EX00010303UMI51440014ID.CO.QRIS.WWW0215ID10264000000020303UMI5204581253033605405250005802ID5914KOPI MALIOBORO6010YOGYAKARTA61055528162070703A016304579F",
  // dynamic, decimal amount "15000.00"
  DYNAMIC_DECIMAL:
    "00020101021226570015COM.EXAMPLE.WWW0117936000000000000010206EX00010303UMI51440014ID.CO.QRIS.WWW0215ID10264000000030303UMI520458125303360540815000.005802ID5915ANGKRINGAN TUGU6010YOGYAKARTA61055528162070703A016304D9DD",
  // same as DYNAMIC_25000 but last CRC char altered
  BAD_CRC:
    "00020101021226570015COM.EXAMPLE.WWW0117936000000000000010206EX00010303UMI51440014ID.CO.QRIS.WWW0215ID10264000000020303UMI5204581253033605405250005802ID5914KOPI MALIOBORO6010YOGYAKARTA61055528162070703A0163045790",
} as const;
```

### 3.5 `__tests__/qris-parser.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { parseEMVCoQRIS, decodeTLV, verifyCRC, QRISParseError } from "../lib/qris/parser";
import { crc16 } from "../lib/qris/crc16";
import { QRIS_SAMPLES as S } from "../lib/qris/samples";

describe("crc16", () => {
  it("matches the standard CCITT-FALSE check value", () => {
    expect(crc16("123456789")).toBe("29B1");
  });
});

describe("parseEMVCoQRIS", () => {
  it("parses a static QRIS (no amount)", () => {
    const r = parseEMVCoQRIS(S.STATIC);
    expect(r).toMatchObject({
      merchantNMID: "ID1026400000001",
      merchantName: "WARUNG BU SARI",
      amountIDR: null,
      isStatic: true,
      merchantCity: "YOGYAKARTA",
    });
  });

  it("parses a dynamic QRIS with amount", () => {
    const r = parseEMVCoQRIS(S.DYNAMIC_25000);
    expect(r).toMatchObject({ merchantNMID: "ID1026400000002", merchantName: "KOPI MALIOBORO", amountIDR: 25000, isStatic: false });
  });

  it("rounds decimal amounts", () => {
    expect(parseEMVCoQRIS(S.DYNAMIC_DECIMAL).amountIDR).toBe(15000);
  });

  it("verifies CRC on valid samples and rejects a bad one", () => {
    expect(verifyCRC(S.STATIC)).toBe(true);
    expect(verifyCRC(S.BAD_CRC)).toBe(false);
    expect(() => parseEMVCoQRIS(S.BAD_CRC)).toThrowError(QRISParseError);
    expect(() => parseEMVCoQRIS(S.BAD_CRC, { strictCRC: false })).not.toThrow();
  });

  it("rejects non-QRIS payloads", () => {
    expect(() => parseEMVCoQRIS("https://example.com")).toThrowError(/EMVCo/);
    expect(() => parseEMVCoQRIS("")).toThrowError(QRISParseError);
  });

  it("rejects truncated / malformed TLV", () => {
    expect(() => decodeTLV("0002")).toThrowError(QRISParseError);
    expect(() => decodeTLV("00ZZ01")).toThrowError(QRISParseError);
    expect(() => decodeTLV("0099abc")).toThrowError(QRISParseError); // length overrun
  });
});
```

### 3.6 `components/qris-scanner.tsx`

Requirements: client-only; dynamic-import `html5-qrcode`; rear camera; single-fire callback; clean teardown (React StrictMode mounts twice in dev); permission-denied UI; **two demo-safety fallbacks** — (a) upload a QR image, (b) paste a raw QRIS string (shown only when `NODE_ENV !== "production"` or `?debug=1`).

```tsx
"use client";
import { useEffect, useRef, useState } from "react";

interface Props {
  onScan: (decodedText: string) => void;
  onError?: (message: string) => void;
}

const REGION_ID = "jpay-qr-region";

export function QrisScanner({ onScan, onError }: Props) {
  const scannerRef = useRef<import("html5-qrcode").Html5Qrcode | null>(null);
  const firedRef = useRef(false);
  const [camError, setCamError] = useState<string | null>(null);
  const [starting, setStarting] = useState(true);

  useEffect(() => {
    let cancelled = false;
    (async () => {
      const { Html5Qrcode } = await import("html5-qrcode");
      if (cancelled) return;
      const scanner = new Html5Qrcode(REGION_ID, { verbose: false });
      scannerRef.current = scanner;
      try {
        await scanner.start(
          { facingMode: "environment" },
          { fps: 10, qrbox: { width: 260, height: 260 } },
          (text) => {
            if (firedRef.current) return;
            firedRef.current = true;
            onScan(text);
          },
          () => {} // per-frame decode failures are normal; ignore
        );
        if (!cancelled) setStarting(false);
      } catch (e) {
        const msg = e instanceof Error ? e.message : String(e);
        setCamError("Camera unavailable. Allow camera access (HTTPS required) or upload a QR image.");
        onError?.(msg);
        setStarting(false);
      }
    })();

    return () => {
      cancelled = true;
      const s = scannerRef.current;
      if (s) s.stop().then(() => s.clear()).catch(() => {});
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  async function handleFile(e: React.ChangeEvent<HTMLInputElement>) {
    const file = e.target.files?.[0];
    if (!file) return;
    try {
      const { Html5Qrcode } = await import("html5-qrcode");
      const decoded = await new Html5Qrcode(REGION_ID, { verbose: false }).scanFile(file, false);
      if (!firedRef.current) { firedRef.current = true; onScan(decoded); }
    } catch {
      onError?.("Could not read a QR code from that image.");
    }
  }

  return (
    <div className="w-full max-w-sm mx-auto">
      <div id={REGION_ID} className="overflow-hidden rounded-2xl bg-black aspect-square" />
      {starting && <p className="mt-2 text-center text-sm text-slate-500">Starting camera…</p>}
      {camError && <p className="mt-2 text-center text-sm text-red-600">{camError}</p>}
      <label className="mt-3 block text-center text-sm text-violet-700 underline cursor-pointer">
        Upload QR image instead
        <input type="file" accept="image/*" className="hidden" onChange={handleFile} />
      </label>
    </div>
  );
}
```

> **Local phone testing:** browsers only expose the camera on **HTTPS or localhost**. Use `npm run dev:https`, or a tunnel (ngrok/cloudflared), or test on the deployed Vercel URL.

### 3.7 Phase 2 Verification

- [ ] `npm run test` → parser suite green (CRC check value `29B1`, 3 valid samples, bad CRC, non-QRIS, malformed TLV)
- [ ] Scanner component renders on `/scan` without SSR errors (`window is not defined` must not appear)
- [ ] Scanning a printed/on-screen QR of each `QRIS_SAMPLES` string returns the right NMID/name/amount (generate QR images from the strings with any online QR generator)
- [ ] Camera permission denied → error text + upload fallback both work
- [ ] Navigating away stops the camera (no lingering camera indicator)

---

## 4. Phase 3 — Price Oracle & Quote Engine · Hour 3–4

### 4.1 Flow

`POST /api/quote {amountIDR}` → fetch (or reuse ≤10 s cached) rates → pure bigint math → return `Quote` with `quoteId` (random bytes32), `issuedAt`, `expiresAt = issuedAt + 30 s`.

Rate sources, in priority order, each with a **1.5 s timeout**; any failure degrades to env mock values (and `rateSource` reflects it):

| Rate | Primary (no API key) | Fallback |
|---|---|---|
| IDR per USD | `https://open.er-api.com/v6/latest/USD` → `rates.IDR` | `MOCK_IDR_PER_USD` |
| MON in USD | CoinGecko `https://api.coingecko.com/api/v3/simple/price?ids=monad&vs_currencies=usd` → `monad.usd` (**verify the id via `/coins/list` first**) | `MOCK_MON_USD` |
| USDC in USD | fixed `1.00` | — |

`FORCE_MOCK_RATES=true` skips the network entirely — use this for **judging day** so latency and amounts are deterministic.

### 4.2 `lib/pricing.ts` — pure, unit-tested math

```ts
import { formatUnits } from "viem";
import type { Quote } from "@/types/quote";

export interface Rates { idrPerUsd: number; monUsd: number; usdcUsd: number }

const E8 = 100_000_000n;
const toE8 = (n: number) => BigInt(Math.round(n * 1e8));
const ceilDiv = (a: bigint, b: bigint) => (a + b - 1n) / b;

/** token_wei = idr / idrPerUsd / monUsd * 1e18 — computed in scaled integers, rounded UP (never underpay). */
export function computeQuote(
  amountIDR: number,
  rates: Rates,
  opts: { now?: number; ttlSeconds?: number; quoteId: `0x${string}`; rateSource: Quote["rateSource"] }
): Quote {
  if (!Number.isInteger(amountIDR) || amountIDR <= 0) throw new Error("amountIDR must be a positive integer");
  const now = opts.now ?? Date.now();
  const ttl = opts.ttlSeconds ?? 30;

  const idr = BigInt(amountIDR);
  const idrPerUsdE8 = toE8(rates.idrPerUsd);
  const monUsdE8 = toE8(rates.monUsd);
  const usdcUsdE8 = toE8(rates.usdcUsd);

  const monWei = ceilDiv(idr * E8 * E8 * 10n ** 18n, idrPerUsdE8 * monUsdE8);
  const usdcUnits = ceilDiv(idr * E8 * E8 * 10n ** 6n, idrPerUsdE8 * usdcUsdE8);

  return {
    quoteId: opts.quoteId,
    amountIDR,
    rates,
    rateSource: opts.rateSource,
    mon: { amountWei: monWei.toString(), display: formatUnits(monWei, 18) },
    usdc: { amountUnits: usdcUnits.toString(), display: formatUnits(usdcUnits, 6) },
    issuedAt: now,
    expiresAt: now + ttl * 1000,
    ttlSeconds: ttl,
  };
}
```

### 4.3 `__tests__/pricing.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { computeQuote } from "../lib/pricing";

const id = ("0x" + "ab".repeat(32)) as `0x${string}`;

describe("computeQuote", () => {
  it("Rp25.000 @ 16.000 IDR/USD and $0.05/MON → 31.25 MON, 1.5625 USDC", () => {
    const q = computeQuote(25000, { idrPerUsd: 16000, monUsd: 0.05, usdcUsd: 1 }, { now: 1_000, quoteId: id, rateSource: "mock" });
    expect(q.mon.amountWei).toBe("31250000000000000000");
    expect(q.usdc.amountUnits).toBe("1562500");
    expect(q.expiresAt - q.issuedAt).toBe(30_000);
  });
  it("rounds up and never returns zero for tiny amounts", () => {
    const q = computeQuote(1, { idrPerUsd: 16000, monUsd: 0.05, usdcUsd: 1 }, { quoteId: id, rateSource: "mock" });
    expect(BigInt(q.mon.amountWei)).toBeGreaterThan(0n);
    expect(BigInt(q.usdc.amountUnits)).toBeGreaterThan(0n);
  });
  it("rejects non-positive / non-integer IDR", () => {
    const r = { idrPerUsd: 16000, monUsd: 0.05, usdcUsd: 1 };
    expect(() => computeQuote(0, r, { quoteId: id, rateSource: "mock" })).toThrow();
    expect(() => computeQuote(10.5, r, { quoteId: id, rateSource: "mock" })).toThrow();
  });
});
```

### 4.4 `lib/rates.ts` (server-only)

```ts
import type { Rates } from "./pricing";
import type { Quote } from "@/types/quote";

const TIMEOUT_MS = 1500;
const CACHE_MS = 10_000;
let cache: { at: number; rates: Rates; source: Quote["rateSource"] } | null = null;

async function getJson<T>(url: string): Promise<T> {
  const res = await fetch(url, { signal: AbortSignal.timeout(TIMEOUT_MS), cache: "no-store" });
  if (!res.ok) throw new Error(`${url} → ${res.status}`);
  return res.json() as Promise<T>;
}

export async function getRates(): Promise<{ rates: Rates; source: Quote["rateSource"] }> {
  const mockIdr = Number(process.env.MOCK_IDR_PER_USD ?? 16000);
  const mockMon = Number(process.env.MOCK_MON_USD ?? 0.05);

  if (process.env.FORCE_MOCK_RATES === "true") {
    return { rates: { idrPerUsd: mockIdr, monUsd: mockMon, usdcUsd: 1 }, source: "mock" };
  }
  if (cache && Date.now() - cache.at < CACHE_MS) return { rates: cache.rates, source: cache.source };

  const [fx, mon] = await Promise.allSettled([
    getJson<{ rates: { IDR: number } }>("https://open.er-api.com/v6/latest/USD"),
    getJson<{ monad: { usd: number } }>("https://api.coingecko.com/api/v3/simple/price?ids=monad&vs_currencies=usd"),
  ]);

  const idrPerUsd = fx.status === "fulfilled" && fx.value.rates?.IDR > 0 ? fx.value.rates.IDR : mockIdr;
  const monUsd = mon.status === "fulfilled" && mon.value.monad?.usd > 0 ? mon.value.monad.usd : mockMon;
  const live = [fx, mon].filter((r) => r.status === "fulfilled").length;
  const source: Quote["rateSource"] = live === 2 ? "live" : live === 1 ? "partial-live" : "mock";

  const rates = { idrPerUsd, monUsd, usdcUsd: 1 };
  cache = { at: Date.now(), rates, source };
  return { rates, source };
}
```

### 4.5 `app/api/quote/route.ts`

```ts
import { NextResponse } from "next/server";
import { randomBytes } from "node:crypto";
import { computeQuote } from "@/lib/pricing";
import { getRates } from "@/lib/rates";

export const dynamic = "force-dynamic";
const MAX_IDR = 20_000_000;

export async function POST(req: Request) {
  let body: { amountIDR?: unknown };
  try { body = await req.json(); } catch { return NextResponse.json({ error: "Invalid JSON" }, { status: 400 }); }

  const amountIDR = Number(body.amountIDR);
  if (!Number.isInteger(amountIDR) || amountIDR < 1 || amountIDR > MAX_IDR) {
    return NextResponse.json({ error: `amountIDR must be an integer between 1 and ${MAX_IDR}` }, { status: 400 });
  }

  const { rates, source } = await getRates();
  const quote = computeQuote(amountIDR, rates, {
    quoteId: `0x${randomBytes(32).toString("hex")}`,
    ttlSeconds: Number(process.env.QUOTE_TTL_SECONDS ?? 30),
    rateSource: source,
  });
  return NextResponse.json(quote, { headers: { "Cache-Control": "no-store" } });
}
```

### 4.6 `hooks/use-quote.ts` (client)

Behavior contract:
- Input: `amountIDR | null`. Fetches a quote on mount/amount change.
- Exposes `{ quote, secondsLeft, isExpired, refresh, loading, error }`.
- `secondsLeft` ticks every 250 ms from `quote.expiresAt`; when it reaches 0 → `isExpired = true` and the pay button is disabled until `refresh()`; **auto-refresh** once when it expires *and the tab is idle in `ready` state*.
- Never reuse an expired `quoteId` (each `quoteId` is used as the on-chain `txRef`, which is single-use).

### 4.7 Phase 3 Verification

- [ ] `npm run test` → pricing tests green
- [ ] `curl -s -X POST localhost:3000/api/quote -H 'content-type: application/json' -d '{"amountIDR":25000}'` returns a quote with `expiresAt - issuedAt = 30000`
- [ ] Bad inputs (`0`, `-5`, `"abc"`, `1e12`) → HTTP 400
- [ ] With network off / `FORCE_MOCK_RATES=true` → `rateSource: "mock"` and response time < 300 ms
- [ ] Two consecutive calls return different `quoteId`s
- [ ] **Affordability check:** `MOCK_MON_USD` tuned so a Rp25.000 payment costs less than your faucet-funded balance

---

## 5. Phase 4 — Privy Auth & Web3 Integration · Hour 3–5

### 5.1 `app/providers.tsx`

```tsx
"use client";
import { PrivyProvider } from "@privy-io/react-auth";
import { WagmiProvider, createConfig } from "@privy-io/wagmi";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { http } from "wagmi";
import { monadTestnet } from "@/lib/chains";

const wagmiConfig = createConfig({
  chains: [monadTestnet],
  transports: { [monadTestnet.id]: http(monadTestnet.rpcUrls.default.http[0]) },
  pollingInterval: 250, // Monad blocks are sub-second; poll fast for snappy confirmations
});

const queryClient = new QueryClient();

export default function Providers({ children }: { children: React.ReactNode }) {
  return (
    <PrivyProvider
      appId={process.env.NEXT_PUBLIC_PRIVY_APP_ID!}
      config={{
        loginMethods: ["email", "google", "wallet"],
        appearance: { theme: "light", accentColor: "#6D28D9", showWalletLoginFirst: false },
        defaultChain: monadTestnet,
        supportedChains: [monadTestnet],
        embeddedWallets: {
          ethereum: { createOnLogin: "users-without-wallets" },
          // OPTIONAL (latency/UX): our own confirm screen already exists, so hide Privy's second modal.
          // Verify the exact key in the Privy docs for your installed version before enabling:
          // showWalletUIs: false,
        },
      }}
    >
      <QueryClientProvider client={queryClient}>
        <WagmiProvider config={wagmiConfig}>{children}</WagmiProvider>
      </QueryClientProvider>
    </PrivyProvider>
  );
}
```

Wire into `app/layout.tsx`: wrap `{children}` with `<Providers>`; add `<Header />`; mobile-first container (`max-w-md mx-auto`).

### 5.2 `hooks/use-jpay-contract.ts`

```ts
"use client";
import { useCallback } from "react";
import { useAccount, usePublicClient, useWalletClient } from "wagmi";
import { type Hex } from "viem";
import { JPAY_GATEWAY_ABI, JPAY_GATEWAY_ADDRESS, ERC20_ABI, USDC_ADDRESS } from "@/lib/contracts/config";
import { monadTestnet } from "@/lib/chains";

export interface PayMONArgs { merchantNMID: string; amountIDR: number; txRef: Hex; valueWei: bigint }
export interface PayUSDCArgs { merchantNMID: string; amountIDR: number; txRef: Hex; amountUnits: bigint }

export type PayStage = "idle" | "approving" | "awaiting-signature" | "confirming" | "confirmed";

export function useJPayContract(onStage?: (s: PayStage) => void) {
  const { address } = useAccount();
  const publicClient = usePublicClient({ chainId: monadTestnet.id });
  const { data: walletClient } = useWalletClient({ chainId: monadTestnet.id });

  const assertReady = useCallback(() => {
    if (!address || !walletClient || !publicClient) throw new Error("Wallet not connected");
    return { address, walletClient, publicClient };
  }, [address, walletClient, publicClient]);

  /** Native MON payment. Returns the confirmed tx hash. */
  const payWithMON = useCallback(async (a: PayMONArgs): Promise<Hex> => {
    const { address, walletClient, publicClient } = assertReady();

    // simulate first → clean revert reasons, and no wasted signature on a doomed tx
    const { request } = await publicClient.simulateContract({
      address: JPAY_GATEWAY_ADDRESS, abi: JPAY_GATEWAY_ABI, functionName: "payWithMON",
      args: [a.merchantNMID, BigInt(a.amountIDR), a.txRef], value: a.valueWei, account: address,
    });
    onStage?.("awaiting-signature");
    const hash = await walletClient.writeContract(request); // NOTE: no manual `gas` (Monad bills gas LIMIT)
    onStage?.("confirming");
    const receipt = await publicClient.waitForTransactionReceipt({ hash, pollingInterval: 200 });
    if (receipt.status !== "success") throw new Error("Transaction reverted");
    onStage?.("confirmed");
    return hash;
  }, [assertReady, onStage]);

  /** USDC payment: approve exact amount if allowance is short, then pay. */
  const payWithUSDC = useCallback(async (a: PayUSDCArgs): Promise<Hex> => {
    const { address, walletClient, publicClient } = assertReady();

    const allowance = (await publicClient.readContract({
      address: USDC_ADDRESS, abi: ERC20_ABI, functionName: "allowance", args: [address, JPAY_GATEWAY_ADDRESS],
    })) as bigint;

    if (allowance < a.amountUnits) {
      onStage?.("approving");
      const approveHash = await walletClient.writeContract({
        address: USDC_ADDRESS, abi: ERC20_ABI, functionName: "approve",
        args: [JPAY_GATEWAY_ADDRESS, a.amountUnits], account: address, chain: monadTestnet,
      });
      await publicClient.waitForTransactionReceipt({ hash: approveHash, pollingInterval: 200 });
    }

    const { request } = await publicClient.simulateContract({
      address: JPAY_GATEWAY_ADDRESS, abi: JPAY_GATEWAY_ABI, functionName: "payWithUSDC",
      args: [a.merchantNMID, a.amountUnits, BigInt(a.amountIDR), a.txRef], account: address,
    });
    onStage?.("awaiting-signature");
    const hash = await walletClient.writeContract(request);
    onStage?.("confirming");
    const receipt = await publicClient.waitForTransactionReceipt({ hash, pollingInterval: 200 });
    if (receipt.status !== "success") throw new Error("Transaction reverted");
    onStage?.("confirmed");
    return hash;
  }, [assertReady, onStage]);

  /** Testnet helper: mint MockUSDC to the connected wallet. */
  const mintMockUSDC = useCallback(async (amountUnits: bigint = 100_000_000n): Promise<Hex> => {
    const { address, walletClient, publicClient } = assertReady();
    const hash = await walletClient.writeContract({
      address: USDC_ADDRESS, abi: ERC20_ABI, functionName: "mint", args: [address, amountUnits],
      account: address, chain: monadTestnet,
    });
    await publicClient.waitForTransactionReceipt({ hash });
    return hash;
  }, [assertReady]);

  return { payWithMON, payWithUSDC, mintMockUSDC };
}
```

Error mapping helper (put in `utils/format.ts`): map viem errors → friendly strings — `User rejected` → "You cancelled the signature", `insufficient funds` → "Not enough MON for this payment + gas", `ref already used` → "Quote already used — refresh the quote", else the first line of `shortMessage`.

### 5.3 Supporting Components

- `components/connect-button.tsx` — `usePrivy()`: `!ready` → skeleton; `!authenticated` → **"Sign in / Connect Wallet"** → `login()` (modal offers Google/Email/Wallet); authenticated → truncated address + **Log out**.
- `components/fund-wallet-helper.tsx` — shows the connected address (copy button), MON balance (`useBalance`), MockUSDC balance (`useReadContract` → `balanceOf`), a **"Get testnet MON"** link (`NEXT_PUBLIC_FAUCET_URL`), and a **"Mint 100 test USDC"** button (calls `mintMockUSDC`). Shown when MON balance is low or on the landing page under the connected state.

### 5.4 Phase 4 Verification

- [ ] Google/Email login creates an embedded wallet; address visible after login
- [ ] External wallet (MetaMask) connects and is auto-switched/prompted to Monad Testnet
- [ ] Session persists on page refresh and across `/` → `/scan` → `/pay`
- [ ] From a temporary dev button: `payWithMON` with a tiny `valueWei` succeeds, returns a hash, and the tx appears in the explorer with the `PaymentExecuted` log
- [ ] `payWithUSDC` flow: mint → approve → pay succeeds (**Hour-4 checkpoint — see §10.1**)
- [ ] Rejecting the signature surfaces a friendly message (no unhandled promise rejection)
- [ ] `npm run typecheck` and `npm run build` pass

---

## 6. Phase 5 — Mock Settlement API & Event Listener · Hour 5–6

### 6.1 Contract of the API

`POST /api/settlement` body `{ "txHash": "0x…" }` (also `GET ?txHash=0x…`, used by the receipt page for reloads).

Server steps (target **< 1 s** after on-chain confirmation, since the client calls it only after the receipt is confirmed):
1. Validate `txHash` with viem `isHash`.
2. `waitForTransactionReceipt` (timeout 8 s, polling 150 ms). Timeout ⇒ `PENDING`.
3. `status !== "success"` ⇒ `FAILED`.
4. Decode logs with `parseEventLogs({ abi, eventName: "PaymentExecuted" })`, keep only logs **emitted by `JPAY_GATEWAY_ADDRESS`**. None ⇒ `FAILED` ("No PaymentExecuted event from JPayGateway").
5. Mock payout: `settlementRef = "JPAY-" + keccak256(txHash).slice(2, 12).toUpperCase()` — **deterministic**, so repeated calls are idempotent without a database.
6. Return `SettlementResponse` with `status: "SUCCESS"`, `latencyMs`, `mock: true`.

### 6.2 `app/api/settlement/route.ts`

```ts
import { NextResponse } from "next/server";
import { createPublicClient, http, isHash, keccak256, parseEventLogs, type Hex } from "viem";
import { monadTestnet } from "@/lib/chains";
import { JPAY_GATEWAY_ABI, JPAY_GATEWAY_ADDRESS } from "@/lib/contracts/config";
import type { SettlementResponse } from "@/types/settlement";

export const dynamic = "force-dynamic";

const client = createPublicClient({
  chain: monadTestnet,
  transport: http(process.env.MONAD_RPC_URL || monadTestnet.rpcUrls.default.http[0]),
});

async function settle(txHash: Hex): Promise<SettlementResponse> {
  const t0 = Date.now();
  const base = { txHash, mock: true as const };
  const done = (r: Omit<SettlementResponse, "txHash" | "mock" | "latencyMs">): SettlementResponse =>
    ({ ...base, latencyMs: Date.now() - t0, ...r });

  let receipt;
  try {
    receipt = await client.waitForTransactionReceipt({ hash: txHash, timeout: 8_000, pollingInterval: 150 });
  } catch {
    return done({ status: "PENDING" });
  }
  if (receipt.status !== "success") return done({ status: "FAILED", error: "Transaction reverted" });

  const logs = parseEventLogs({ abi: JPAY_GATEWAY_ABI, eventName: "PaymentExecuted", logs: receipt.logs })
    .filter((l) => l.address.toLowerCase() === JPAY_GATEWAY_ADDRESS.toLowerCase());
  if (logs.length === 0) return done({ status: "FAILED", error: "No PaymentExecuted event from JPayGateway" });

  const a = logs[0].args;
  return done({
    status: "SUCCESS",
    settlementRef: `JPAY-${keccak256(txHash).slice(2, 12).toUpperCase()}`,
    settledAt: new Date().toISOString(),
    event: {
      payer: a.payer, merchantNMID: a.merchantNMID,
      amountToken: a.amountToken.toString(), amountIDR: a.amountIDR.toString(),
      isNative: a.isNative, txRef: a.txRef,
    },
  });
}

export async function POST(req: Request) {
  const { txHash } = (await req.json().catch(() => ({}))) as { txHash?: string };
  if (!txHash || !isHash(txHash)) return NextResponse.json({ error: "Invalid txHash" }, { status: 400 });
  return NextResponse.json(await settle(txHash));
}

export async function GET(req: Request) {
  const txHash = new URL(req.url).searchParams.get("txHash");
  if (!txHash || !isHash(txHash)) return NextResponse.json({ error: "Invalid txHash" }, { status: 400 });
  return NextResponse.json(await settle(txHash));
}
```

### 6.3 Optional: `scripts/settlement-listener.ts` ("merchant terminal" for the pitch)

Run in a second terminal on the demo laptop; prints a line the instant a payment lands. Reinforces the "merchant sees Rupiah" story on screen.

```ts
import "dotenv/config";
import { createPublicClient, http, formatUnits } from "viem";
import { monadTestnet } from "../lib/chains";
import { JPAY_GATEWAY_ABI, JPAY_GATEWAY_ADDRESS } from "../lib/contracts/config";

const client = createPublicClient({ chain: monadTestnet, transport: http(process.env.MONAD_RPC_URL), pollingInterval: 250 });

console.log("👂 JPay settlement listener on", JPAY_GATEWAY_ADDRESS);
client.watchContractEvent({
  address: JPAY_GATEWAY_ADDRESS,
  abi: JPAY_GATEWAY_ABI,
  eventName: "PaymentExecuted",
  onLogs: (logs) => {
    for (const l of logs) {
      const a = l.args as any;
      const tokenStr = a.isNative ? `${formatUnits(a.amountToken, 18)} MON` : `${formatUnits(a.amountToken, 6)} USDC`;
      console.log(`✅ [MOCK SETTLEMENT] Rp ${a.amountIDR.toString()} → NMID ${a.merchantNMID} | paid ${tokenStr} | tx ${l.transactionHash}`);
    }
  },
});
```

> `tsx` resolves the project's ESM-style imports; the script uses **relative** imports (no `@/` alias).

### 6.4 Phase 5 Verification

- [ ] `curl -s -X POST localhost:3000/api/settlement -H 'content-type: application/json' -d '{"txHash":"<real tx>"}'` → `status: "SUCCESS"`, `settlementRef` present, `latencyMs` < 1000 once the tx is confirmed
- [ ] Same call twice → identical `settlementRef` (idempotent)
- [ ] Non-JPay tx hash → `FAILED` with message; garbage input → HTTP 400; unknown hash → `PENDING` after timeout
- [ ] `npm run listener` prints a line when a payment is executed from the UI

---

## 7. Phase 6 — Frontend Pages & UI Components · Hour 4–6

### 7.1 Shared Flow State — `lib/session.ts` + `hooks/use-payment-session.ts`

```ts
// lib/session.ts
import type { ParsedQRIS } from "@/types/qris";
import type { PayToken } from "@/types/quote";

export interface PaymentSession {
  qris: ParsedQRIS;
  amountIDR: number;          // from QR (dynamic) or manual entry (static)
  token: PayToken;            // default "MON"
}
const KEY = "jpay:session";

export const saveSession = (s: PaymentSession) => { try { sessionStorage.setItem(KEY, JSON.stringify(s)); } catch {} };
export const loadSession = (): PaymentSession | null => { try { const v = sessionStorage.getItem(KEY); return v ? JSON.parse(v) : null; } catch { return null; } };
export const clearSession = () => { try { sessionStorage.removeItem(KEY); } catch {} };
```

The hook wraps these with React state so `/pay` survives a refresh. Pages navigate with `router.push`; **no sensitive data goes in the URL** (only `txHash` and `?m=merchantName` for the receipt).

### 7.2 Formatters — `utils/format.ts`

```ts
import { formatUnits } from "viem";

export const formatIDR = (n: number | bigint | string) =>
  new Intl.NumberFormat("id-ID", { style: "currency", currency: "IDR", maximumFractionDigits: 0 }).format(Number(n));

export const formatToken = (raw: bigint | string, decimals: number, maxFrac = 4) => {
  const s = formatUnits(BigInt(raw), decimals);
  const [i, f = ""] = s.split(".");
  return f ? `${i}.${f.slice(0, maxFrac).replace(/0+$/, "") || "0"}` : i;
};

export const shortAddr = (a: string) => `${a.slice(0, 6)}…${a.slice(-4)}`;
export const shortHash = (h: string) => `${h.slice(0, 10)}…${h.slice(-8)}`;
```

### 7.3 `app/page.tsx` — Landing / Connect / Scan Trigger

Tasks:
- [ ] Hero: logo, tagline **"Just Pay with Monad"**, one-line value prop ("Pay any QRIS merchant with crypto. The merchant receives Rupiah.")
- [ ] 3-step explainer strip: **Scan → Confirm → Done** (icons, no jargon)
- [ ] `<ConnectButton />` as the primary CTA when logged out
- [ ] When authenticated: large **"Scan QRIS"** button → `/scan`; show short address + MON/USDC balances + `<FundWalletHelper />`
- [ ] Footer chip: "Monad Testnet" + network status dot
- [ ] Mobile-first, no horizontal scroll at 360 px; Tailwind only

### 7.4 `app/scan/page.tsx` — Live Scanner + Manual Entry

Logic:

```tsx
"use client";
// 1) Guard: if !authenticated → router.replace("/")
// 2) <QrisScanner onScan={handleScan} onError={setError} />
// 3) handleScan(text):
//      try { const parsed = parseEMVCoQRIS(text) } catch (e) { show error banner mapped from QRISParseError.code; re-arm scanner }
//      if (parsed.amountIDR) → saveSession({ qris: parsed, amountIDR: parsed.amountIDR, token: "MON" }); router.push("/pay")
//      else → open <ManualAmountModal merchant={parsed} onConfirm={(amt) => { saveSession(...); router.push("/pay") }} />
```

Tasks:
- [ ] Full-width camera viewport with scan-frame overlay and "Point at a QRIS code" hint
- [ ] Error banner copy per code: `NOT_QRIS` → "That's not a QRIS code"; `BAD_CRC` → "QR looks damaged — try again"; `MISSING_NMID` → "Couldn't find merchant ID"; `MALFORMED_TLV` → "Unreadable QR"; each with **Scan again** (remount scanner via `key` bump)
- [ ] `ManualAmountModal`: shows merchant name + NMID, numeric IDR input (integer only, `inputMode="numeric"`, thousands separators on blur), quick-amount chips (10.000 / 25.000 / 50.000), validation 1 ≤ amount ≤ 20.000.000, **Continue** / **Cancel**
- [ ] Dev-only (`NODE_ENV !== "production"` or `?debug=1`): "Paste QRIS string" textarea + buttons that inject each `QRIS_SAMPLES` entry (demo insurance when the camera misbehaves)

### 7.5 `app/pay/page.tsx` — Quote Confirmation & Payment

> Any component using `useSearchParams` **must** be wrapped in `<Suspense>` or `next build` fails.

State machine (single `useReducer`):

```
loading-quote → ready → (expired → ready) → approving* → awaiting-signature → confirming → settling → done
                                       └──────────────→ error (retry → ready)
* USDC path only, when allowance is short
```

Tasks:
- [ ] On mount: `loadSession()`; if missing → `router.replace("/scan")`
- [ ] Header card: **merchant name**, city, NMID (small), big **IDR amount** (`formatIDR`)
- [ ] `<TokenToggle />`: MON | USDC (hide USDC when `NEXT_PUBLIC_ENABLE_USDC !== "true"`); show the user's balance under each; disable a token if balance < required (with "Insufficient balance" hint + fund helper link)
- [ ] `<QuoteCard />`: "You pay **31.25 MON**", "≈ **Rp25.000**", rate line (`1 MON ≈ Rp800`), `rateSource` pill (Live / Mock)
- [ ] `<QuoteCountdown />`: 30 s progress ring/bar; at 0 → "Quote expired" + auto-refresh (new `quoteId`)
- [ ] `<PayButton />`: label by state — **"Pay 31.25 MON"** / "Approving USDC…" / "Confirm in wallet…" / "Confirming on Monad…" / "Settling Rupiah…"; disabled when expired, loading, or insufficient balance
- [ ] On click: ensure wallet connected & on Monad Testnet (`useSwitchChain` if needed) → call `payWithMON` / `payWithUSDC` with `txRef = quote.quoteId` → on confirmed hash `POST /api/settlement` → `router.push(\`/receipt/${hash}?m=${encodeURIComponent(merchantName)}\`)`
- [ ] `<TxProgress />`: 4-step indicator (Sign → On-chain → Settle → Receipt) with elapsed-time counter (the **< 3 s** pitch metric — show "Completed in 2.4 s" on the receipt)
- [ ] Error handling via the friendly-error mapper; **Try again** returns to `ready` with a fresh quote; never leave the UI stuck in a pending state
- [ ] Double-submit guard (disable button + ref flag) — a re-used `txRef` reverts on-chain

### 7.6 `app/receipt/[txHash]/page.tsx` — Digital Receipt

Data sources: `txHash` from the route param (`useParams`); `merchantName` from `?m=` (fallback `sessionStorage` session, else "Merchant (NMID …)"); **everything else from `GET /api/settlement?txHash=`** (decoded on-chain event) so the receipt is verifiable and survives reloads.

Tasks:
- [ ] Validate `txHash` with `isHash`; invalid → "Receipt not found" state
- [ ] Poll `GET /api/settlement` every 700 ms until `status !== "PENDING"` (max ~10 s)
- [ ] `<ReceiptCard />` (screenshot-friendly, white card, subtle border, JPay logo):
  - [ ] Big **Rp amount paid** (from event `amountIDR`)
  - [ ] **Merchant name** + NMID
  - [ ] **Spent:** `formatToken(amountToken, isNative ? 18 : 6)` + symbol `MON` / `USDC`
  - [ ] **Tx Hash** (`shortHash`, copy button, link → `explorerTxUrl(hash)`)
  - [ ] **Paid by** (`shortAddr(payer)`), timestamp
  - [ ] **Settlement reference** (`JPAY-…`) and `<SettlementBadge />`: `PENDING` (amber, spinner) → `SUCCESS` (green ✓ "IDR settled") / `FAILED` (red)
  - [ ] Small print: "Demo settlement — simulated IDR payout (testnet)"
- [ ] Actions: **Share** (`navigator.share` with fallback to copy link), **Pay again** (`clearSession()` → `/scan`)
- [ ] Wrap `useSearchParams` usage in `<Suspense>`

### 7.7 UI Component Notes

- Visual language: violet/indigo brand (`#6D28D9` accent), slate neutrals, 2xl rounded cards, generous tap targets (≥ 48 px), large numerals for amounts.
- `components/ui/*`: minimal Tailwind primitives (`Button` with `variant`, `loading` props; `Card`; `Badge` with tone; `Spinner`).
- Accessibility: buttons have labels, countdown has `aria-live="polite"`, color is never the only status signal.

### 7.8 Phase 6 Verification

- [ ] `/` → login → `/scan` → scan `DYNAMIC_25000` → `/pay` → pay with MON → `/receipt/<hash>` shows SUCCESS (full manual run)
- [ ] Static QR → manual amount modal → same flow
- [ ] Quote expiry works (wait 30 s → disabled → auto-refresh → pay works with the new quote)
- [ ] Insufficient balance state is shown and blocks payment
- [ ] Page refresh on `/pay` and `/receipt/<hash>` still renders (session + on-chain data)
- [ ] Layout checked at 360 × 640 and 390 × 844
- [ ] `npm run build` passes (no `useSearchParams` Suspense errors)

---

## 8. Phase 7 — AI Agent Prompting Strategy (8-Hour Blitz Checklist)

### 8.1 Schedule → Prompt Map

| Prompt | Hours | Covers | Gate to pass before continuing |
|---|---|---|---|
| **1. Setup & Environment** | 0–1 | §1 | `build` + `typecheck` green, Privy origins set |
| **2. Contract Deployment** | 1–2 | §2 | tests green, deployed, `config.ts` generated |
| **3. QRIS Parser & Scanner** | 2–3 | §3 | parser tests green, scanner works |
| **4. Quote Engine + Privy + Contract Hooks** | 3–5 | §4, §5 | **Hour-4 checkpoint:** one MON payment succeeds from the UI |
| **5. Settlement + Pages** | 5–6 | §6, §7 | full flow works in the browser |
| **6. E2E Hardening, Polish & Deploy** | 6–8 | §9, §10 | Vercel live, backup video recorded |

Paste each prompt into the agent **one at a time**. Do not paste the next until the previous prompt's *Definition of Done* is true. Put `PRD.md` and `PLAN.md` in the repo root first so the agent can read them.

---

### Prompt 1 — Setup & Environment (Hour 0–1)

````text
You are a senior full-stack Web3 engineer building the JPay hackathon MVP. Read PRD.md and PLAN.md in the repo root. PLAN.md §0 (rules + deviations) is binding. Execute ONLY PLAN.md §1 (Repository Setup) now.

Tasks:
1. Bootstrap the Next.js 14 app exactly as in §1.1 (App Router, TypeScript, Tailwind, no src dir, alias @/*). If the repo already has files, scaffold in place without deleting PRD.md / PLAN.md.
2. Install dependencies exactly as in §1.1 / §1.2. Resolve peer-dep conflicts by pinning compatible majors (do not use --legacy-peer-deps unless nothing else works, and tell me if you did).
3. Create the directory tree from §1.3 with empty placeholder files where needed (keep them compiling).
4. Write: tsconfig.json (with excludes), tsconfig.hardhat.json, hardhat.config.ts, vitest.config.ts, next.config.mjs, .env.example, and the .gitignore additions — all as specified in §1.4/§1.5.
5. Create types/qris.ts, types/quote.ts, types/settlement.ts, lib/chains.ts, utils/format.ts, utils/cn.ts exactly per §1.6, §1.7, §7.2.
6. Create a STUB lib/contracts/config.ts exporting DEPLOYMENT (zero addresses), JPAY_GATEWAY_ADDRESS, USDC_ADDRESS, JPAY_GATEWAY_ABI = [] as const, ERC20_ABI = [] as const, so imports compile before deployment. (Phase 1 will overwrite it.)
7. Put a minimal home page ("JPay — Just Pay with Monad") so the app renders.
8. Add all package.json scripts from §1.2.

Constraints: do not implement features from later phases. Do not add dependencies not listed in PLAN.md. Never write secrets into tracked files.

Verify (run these and show me the output):
- npm run typecheck
- npm run build
- npm run dev  (confirm http://localhost:3000 renders, then stop it)

Definition of Done: all three commands succeed; directory tree matches §1.3; then commit with message "phase 0: scaffold". Finally, list the manual steps I must do myself (create Privy app + allowed origins, fund throwaway wallet, fill .env.local).
````

### Prompt 2 — Contract Deployment (Hour 1–2)

````text
Continue the JPay build. Read PLAN.md §2 and execute ONLY Phase 1 (Smart Contracts).

Tasks:
1. Create contracts/JPayGateway.sol and contracts/mocks/MockUSDC.sol exactly as in §2.1 and §2.2 (OpenZeppelin v5 imports, Solidity 0.8.28 via hardhat config).
2. Create test/JPayGateway.test.ts as in §2.3. Make every test pass. If a test fails because of a library-version difference (e.g. revertedWith vs revertedWithCustomError), fix the TEST, not the contract's behavior.
3. Create scripts/deploy.ts as in §2.4. It must deploy MockUSDC (when USDC_ADDRESS is empty), deploy JPayGateway(treasury, usdc), and GENERATE lib/contracts/config.ts containing the real addresses and full ABIs (JPAY_GATEWAY_ABI and ERC20_ABI, both `as const`).
4. Run the toolchain and report results at each step.

Verify (run and show output):
- npm run compile
- npm run test:contracts
- npm run deploy:testnet        (requires DEPLOYER_PRIVATE_KEY and funded wallet in .env.local — if it's missing, STOP and tell me exactly what to fill in)
- npm run verify:testnet -- <GATEWAY> <TREASURY> <USDC>   (if verification fails, show the error and continue; verification is non-blocking)
- npm run typecheck

Definition of Done: contract tests pass; contracts deployed on Monad Testnet; lib/contracts/config.ts regenerated with real addresses; typecheck passes; explorer URL for the gateway printed in your final message. Commit "phase 1: contracts deployed". Do not touch frontend code except lib/contracts/config.ts.
````

### Prompt 3 — QRIS Parser & Scanner (Hour 2–3)

````text
Continue the JPay build. Read PLAN.md §3 and execute ONLY Phase 2 (EMVCo QRIS Parser & Camera Scanner).

Tasks:
1. Create lib/qris/crc16.ts, lib/qris/parser.ts, lib/qris/samples.ts exactly as in §3.2–§3.4. The parser must: decode flat TLV, verify CRC16-CCITT-FALSE, extract NMID from Tag 51 subtag 02 (fallback: tags 26–45 subtag 02), Tag 59 merchant name, Tag 54 amount (null when absent; round "15000.00" to 15000), Tag 01 static flag, Tag 60 city, and throw typed QRISParseError codes (NOT_QRIS, MALFORMED_TLV, BAD_CRC, MISSING_NMID, INVALID_AMOUNT).
2. Create __tests__/qris-parser.test.ts as in §3.5. All tests must pass.
3. Create components/qris-scanner.tsx per §3.6: client component, html5-qrcode imported dynamically inside useEffect, rear camera, fire-once callback, clean teardown that survives React StrictMode double-mount, permission-denied message, and an "Upload QR image instead" fallback.
4. Create a temporary app/scan/page.tsx that renders the scanner, runs parseEMVCoQRIS on the result, and prints the parsed object (or the error code) on screen. Add a dev-only (NODE_ENV !== "production") "paste QRIS string" textarea plus buttons for each QRIS_SAMPLES entry. (The final scan UX is built in the later pages phase.)

Verify (run and show output):
- npm run test
- npm run typecheck
- npm run build
- Then tell me how to manually test with a phone/webcam (HTTPS requirement: npm run dev:https or tunnel) and which sample QR images to generate from lib/qris/samples.ts.

Definition of Done: tests green; /scan renders with no SSR "window is not defined" error; sample strings parse correctly in the dev UI; bad CRC and non-QRIS show clear errors. Commit "phase 2: qris parser + scanner".
````

### Prompt 4 — Quote Engine + Privy + Contract Hooks (Hour 3–5, Hour-4 checkpoint inside)

````text
Continue the JPay build. Read PLAN.md §4 and §5 and execute Phase 3 (Price Oracle & Quote Engine) and Phase 4 (Privy Auth & Web3 Integration).

Part A — Quote engine (§4):
1. lib/pricing.ts (pure bigint math, rounds UP), lib/rates.ts (live fetch with 1.5s timeout, 10s cache, env mock fallback, FORCE_MOCK_RATES), app/api/quote/route.ts (POST {amountIDR} -> Quote with random bytes32 quoteId and 30s TTL), hooks/use-quote.ts (countdown + auto-refresh, never reuse an expired quoteId).
2. __tests__/pricing.test.ts per §4.3.

Part B — Privy + web3 (§5):
3. app/providers.tsx per §5.1 (PrivyProvider > QueryClientProvider > WagmiProvider from @privy-io/wagmi, Monad Testnet only, email/google/wallet login, embedded wallet on login). Wrap app/layout.tsx with it.
4. hooks/use-jpay-contract.ts per §5.2: payWithMON, payWithUSDC (check allowance -> approve exact amount if short -> pay), mintMockUSDC. Use simulateContract before writeContract. Do NOT hard-code gas limits (Monad bills the gas limit).
5. components/connect-button.tsx and components/fund-wallet-helper.tsx per §5.3.
6. A temporary dev-only page app/dev/page.tsx with buttons: "Pay 1 test MON" (hard-coded NMID, tiny amount, fresh random txRef) and "Mint 100 USDC + Pay 1 test USDC". Remove or guard it behind NODE_ENV !== "production" later.

Verify (run and show output):
- npm run test
- npm run typecheck && npm run build
- curl -s -X POST http://localhost:3000/api/quote -H 'content-type: application/json' -d '{"amountIDR":25000}'  (with npm run dev running)
- Ask me to log in via the browser and click "Pay 1 test MON" on /dev; confirm the tx hash appears and the PaymentExecuted log shows on the explorer.

HOUR-4 CHECKPOINT: If MON payment works from the UI -> continue and test USDC. If the USDC path is not working within 20 minutes, set NEXT_PUBLIC_ENABLE_USDC=false in .env.example/.env.local, leave the USDC code in place behind the flag, and tell me. MON-only is an acceptable MVP.

Definition of Done: quote API + tests green; login works (Google/email embedded wallet and external wallet); one real MON payment succeeds from the UI on Monad Testnet; build passes. Commit "phase 3-4: quote + privy + contract hooks".
````

### Prompt 5 — Settlement API + Frontend Pages (Hour 5–6)

````text
Continue the JPay build. Read PLAN.md §6 and §7 and execute Phase 5 (Mock Settlement API) and Phase 6 (Frontend Pages).

Part A — Settlement (§6):
1. app/api/settlement/route.ts exactly per §6.2: POST {txHash} and GET ?txHash=; waitForTransactionReceipt, decode PaymentExecuted logs emitted by JPAY_GATEWAY_ADDRESS, return mock SUCCESS with deterministic settlementRef (keccak256(txHash) prefix), latencyMs, status PENDING/SUCCESS/FAILED.
2. scripts/settlement-listener.ts per §6.3 (relative imports, no @/ alias).

Part B — Pages (§7). Build all of these for real (replace the temporary /scan page and keep /dev dev-only):
3. lib/session.ts + hooks/use-payment-session.ts (sessionStorage-backed).
4. components: header, ui/{button,card,badge,spinner}, token-toggle, quote-card, quote-countdown, pay-button, tx-progress, manual-amount-modal, receipt-card, settlement-badge.
5. app/page.tsx (landing + connect + scan CTA + fund helper), app/scan/page.tsx (live scanner + ManualAmountModal for static QRIS + error banners per error code), app/pay/page.tsx (quote confirmation, MON/USDC toggle gated by NEXT_PUBLIC_ENABLE_USDC, 30s countdown + auto-refresh, pay button states, TxProgress with elapsed timer, friendly error mapping, double-submit guard, then POST /api/settlement and router.push to the receipt), app/receipt/[txHash]/page.tsx (poll GET /api/settlement every 700ms, show merchant name from ?m=, IDR paid, MON/USDC spent, Tx Hash linked to explorer, settlement badge PENDING->SUCCESS, share + pay-again buttons).
6. Wrap every component that uses useSearchParams in <Suspense>. Mobile-first at 360px width. Use txRef = quote.quoteId.

Verify (run and show output):
- npm run typecheck && npm run test && npm run build
- curl the settlement endpoint with a real tx hash from Prompt 4; confirm SUCCESS and latencyMs < 1000, and that two calls return the same settlementRef.
- Walk me through a manual run: login -> scan DYNAMIC_25000 sample -> pay with MON -> receipt shows SUCCESS. Then the STATIC sample via manual amount modal.

Definition of Done: the full flow works in the browser for dynamic and static QRIS with MON (and USDC if enabled); receipt reload works; build passes. Commit "phase 5-6: settlement + pages".
````

### Prompt 6 — E2E Hardening, Polish & Deploy (Hour 6–8)

````text
Final phase for the JPay build. Read PLAN.md §9 (QA matrix), §10 (demo-day hardening) and the Vercel section, then execute them.

Tasks:
1. Run the §9 QA matrix. For each row, report PASS/FAIL. Fix every FAIL that touches the core flow (scan -> quote -> pay -> receipt). Add unit tests for any bug you fix in parser/pricing.
2. Reliability pass: double-submit protection, stuck-pending recovery (timeouts with a clear retry), expired-quote handling, wrong-network handling (prompt switch to Monad Testnet), insufficient-balance messaging, camera-denied fallback, and an obvious error boundary on /pay and /receipt.
3. Performance pass: measure scan->receipt latency on testnet and display "Completed in X.Xs" on the receipt (elapsed time from the pay click to settlement SUCCESS). Make sure /api/quote and /api/settlement add no avoidable latency (module-level clients, cached rates).
4. UI polish within 45 minutes max: consistent spacing, loading skeletons, empty states, readable amounts, copy-to-clipboard feedback. No new features.
5. Guard or delete app/dev/page.tsx in production. Make sure no secret is exposed (grep the repo for DEPLOYER_PRIVATE_KEY and any 0x private-key-looking strings; only .env.local may contain them and it must be git-ignored).
6. Write README.md: what JPay is (pitch sentence: "Your grandmother's warung can accept crypto today — without knowing what crypto is."), architecture diagram (mermaid from PRD §4.1), setup steps, env vars, deployed contract + explorer links, demo script, known MVP limitations (mocked settlement, quote not enforced on-chain, testnet MockUSDC).
7. Deploy to Vercel: list the exact env vars to set in the Vercel dashboard (every NEXT_PUBLIC_* plus MONAD_RPC_URL, QUOTE_TTL_SECONDS, FORCE_MOCK_RATES=true for demo day, MOCK_* values; NOT the deployer private key), remind me to add the Vercel domain to Privy Allowed Origins, then run a production smoke test on the deployed URL from a phone.

Verify (run and show output):
- npm run typecheck && npm run test && npm run test:contracts && npm run build
- npx vercel --prod (or give me the dashboard steps if the CLI is not logged in)

Definition of Done: QA matrix all PASS on the production URL; 0 failed payments across 10 consecutive runs; README complete; commit "phase 7: hardening + deploy". List anything still risky for the live demo.
````

---

## 9. QA Matrix (used in Prompt 6, Hour 6–7)

| # | Scenario | Expected |
|---|---|---|
| 1 | Dynamic QRIS Rp25.000, pay MON | Receipt SUCCESS, amounts match the quote |
| 2 | Dynamic QRIS decimal `15000.00`, pay MON | Treated as Rp15.000 |
| 3 | Static QRIS → manual Rp50.000 | Modal validates; flow completes |
| 4 | Pay with USDC (fresh wallet → mint → approve → pay) | Two txs then receipt SUCCESS (skip if USDC disabled) |
| 5 | USDC when allowance already sufficient | Skips approve, single tx |
| 6 | Quote expires while on `/pay` | Button disabled → auto-refresh → pay works with new `quoteId` |
| 7 | Reuse of an old `quoteId` (replay) | Reverts `ref already used`; friendly message; retry with fresh quote works |
| 8 | User rejects the signature | "You cancelled the signature"; UI returns to `ready` |
| 9 | Wallet with insufficient MON | Payment blocked, fund helper shown |
| 10 | Non-QRIS QR (e.g., a URL QR) | "That's not a QRIS code", scanner re-arms |
| 11 | Corrupted QRIS (bad CRC) | "QR looks damaged" |
| 12 | Camera permission denied | Error text + image-upload fallback |
| 13 | Reload `/pay` mid-flow | Session restored, fresh quote |
| 14 | Reload `/receipt/<hash>` | Receipt rebuilt from chain; merchant name from `?m=` |
| 15 | `/receipt/0xdeadbeef` | "Receipt not found" |
| 16 | Wrong network in external wallet | Prompt/auto-switch to Monad Testnet |
| 17 | Network slow (RPC delay) | Progress states visible; no stuck spinner > 20 s without retry UI |
| 18 | 10 consecutive happy-path payments | 0 failures; record scan→receipt latency (target < 3 s on-chain + settle) |
| 19 | Mobile 360 × 640 & 390 × 844 | No horizontal scroll; tap targets ≥ 48 px |
| 20 | `npm run build` + Vercel prod URL | Passes; camera works over HTTPS |

---

## 10. Hardening, Hour-4 Checkpoint & Deployment

### 10.1 Hour-4 Hard Checkpoint (from PRD §8)

At **Hour 4**, answer one question: *does a MON payment complete from the UI and show a `PaymentExecuted` log on the explorer?*
- **Yes** → continue with USDC.
- **No** → **freeze scope**, set `NEXT_PUBLIC_ENABLE_USDC=false`, and fix the MON path only. USDC code stays in the repo behind the flag (it renders nothing when disabled).

### 10.2 Demo-Day Reliability Checklist

- [ ] `FORCE_MOCK_RATES=true` on Vercel for judging (deterministic amounts, no third-party API latency)
- [ ] `MOCK_MON_USD` tuned so Rp25.000 costs a few MON at most (faucet balance is limited)
- [ ] Two pre-funded demo wallets (MON + minted USDC) logged in on two phones
- [ ] Printed + on-screen QR codes for each `QRIS_SAMPLES` string (dynamic **and** static)
- [ ] Optional terminal on the projector running `npm run listener` ("merchant sees Rupiah")
- [ ] Backup screen recording of the full flow (< 60 s) saved offline
- [ ] Paid RPC / secondary RPC URL ready in `.env` in case the public RPC rate-limits
- [ ] Privy **Allowed Origins** includes the production Vercel domain (+ preview domain if used)
- [ ] Browser camera permission pre-granted on demo devices; phone on stable Wi-Fi/hotspot
- [ ] Contract + explorer links in the README and pitch deck slide

### 10.3 Vercel Deployment

```bash
npm i -g vercel         # or use the dashboard
vercel link
# set env vars (Production + Preview), NEVER the deployer key:
vercel env add NEXT_PUBLIC_PRIVY_APP_ID
vercel env add NEXT_PUBLIC_CHAIN_ID
vercel env add NEXT_PUBLIC_MONAD_RPC_URL
vercel env add NEXT_PUBLIC_EXPLORER_URL
vercel env add NEXT_PUBLIC_ENABLE_USDC
vercel env add NEXT_PUBLIC_FAUCET_URL
vercel env add MONAD_RPC_URL
vercel env add QUOTE_TTL_SECONDS
vercel env add FORCE_MOCK_RATES
vercel env add MOCK_IDR_PER_USD
vercel env add MOCK_MON_USD
vercel --prod
```

`lib/contracts/config.ts` is committed, so the deployed addresses ship with the build; `NEXT_PUBLIC_JPAY_CONTRACT_ADDRESS` / `NEXT_PUBLIC_USDC_ADDRESS` are only needed to override them.

### 10.4 Troubleshooting Quick Reference

| Symptom | Likely cause → fix |
|---|---|
| `window is not defined` on `/scan` | `html5-qrcode` imported at module top → dynamic `import()` inside `useEffect` |
| Camera blank on phone | Not HTTPS → `npm run dev:https`, tunnel, or Vercel URL |
| Privy login popup fails / origin error | Origin missing in Privy dashboard → add `localhost:3000` and the Vercel domain |
| `useAccount` shows disconnected after Privy login | Provider order wrong → `PrivyProvider > QueryClientProvider > WagmiProvider (from @privy-io/wagmi)` |
| Hardhat: `Cannot use import statement outside a module` | Missing `TS_NODE_PROJECT=tsconfig.hardhat.json` → use the npm scripts |
| `next build` type errors in `scripts/` or `test/` | `exclude` list missing in `tsconfig.json` |
| `useSearchParams() should be wrapped in a suspense boundary` | Wrap the component in `<Suspense>` |
| Payment reverts `ref already used` | Reused `quoteId` → force a fresh quote after every attempt |
| `insufficient funds` despite balance | Gas is billed on the gas **limit** on Monad → don't set manual `gas`; keep spare MON |
| Contract verification fails | Check Sourcify URL in `hardhat.config.ts` against Monad docs; non-blocking |
| Rates endpoint slow | Set `FORCE_MOCK_RATES=true` |

---

## 11. Master Definition of Done (MVP)

- [ ] `JPayGateway.sol` deployed (and ideally verified) on Monad Testnet; address in README
- [ ] Scan any sample QRIS → correct NMID / merchant name / IDR amount (static uses manual entry)
- [ ] Quote shows IDR ⇄ MON/USDC with 30 s validity and auto-refresh
- [ ] Privy login (Google/email embedded wallet + external wallet) works end-to-end
- [ ] On-chain payment emits `PaymentExecuted`; settlement API returns mock `SUCCESS` in < 1 s after confirmation
- [ ] Receipt page: Tx Hash (explorer link), Merchant Name, IDR Paid, MON/USDC Spent, Settlement badge
- [ ] 0 failed payments across 10 consecutive live runs; scan→receipt visibly < 3 s
- [ ] Deployed on Vercel; backup demo video recorded; pitch line rehearsed:
  **"Your grandmother's warung can accept crypto today — without knowing what crypto is."**

*End of PLAN.md — JPay v0.1 MVP execution roadmap*
