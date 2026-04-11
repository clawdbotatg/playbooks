# Comprehensive dApp Build Playbook

From spec to shipped, audited, production dApp. Every step, every gate, every rule. Nothing left to guess.

This playbook synthesizes six existing build pipelines, the complete ethskills.com skill library, and Scaffold-ETH 2 documentation into a single authoritative reference. It covers architecture, discovery, planning, contract development, frontend development, auditing, deployment, QA, and delivery.

---

## Philosophy

**The cardinal rule: spec fidelity over pipeline completion.**

A build that compiles, passes tests, and deploys is worthless if it doesn't implement what the spec says. Three principles:

1. **Build the thing, then prove the thing works.** Every stage that produces code has a corresponding verification gate that checks the code does what the spec requires — not just that it runs.
2. **Never optimize for pipeline progress.** If a later stage reveals the architecture is wrong, go back. Moving forward with a broken foundation creates exponentially more work later.
3. **Fork real chains, use real protocols.** Empty local chains hide integration bugs. Always `yarn fork --network base` so contracts interact with real Uniswap pools, real token contracts, real oracle data.

---

## Pipeline Overview

```
SPEC ─> DISCOVER ─> PLAN ─> SCAFFOLD ─> BUILD ─> AUDIT ─> FIX ─> DEPLOY ─> SHIP ─> QA ─> DELIVER
  │        │          │        │          │        │                │         │        │
  │        │          │        │          │        └─> FIX ─────────┘         │        │
  │        │          │        │          │                                    │        │
  │        │          │        │          └── compile ─┬─> tests (terminal)   │        │
  │        │          │        │                       ├─> frontend ──────────┘        │
  │        │          │        │                       └─> deploy ────────────┘        │
  │        │          │        │                                                       │
  └────────┴──────────┴────────┴──────── regression loop ◄────────────────────────────┘
```

Every arrow includes a gate. If the gate fails, you go back. No exceptions.

### Three Phases (hard boundaries — never combine)

| Phase | Environment | Gate to exit |
|-------|-------------|-------------|
| **Phase 1: Build** | Local fork of target chain | Contracts compile, tests pass, frontend builds, UI loads, all flows work |
| **Phase 2: Deploy** | Live network (Base, mainnet, etc.) | Contracts deployed, verified on block explorer, `deployedContracts.ts` updated |
| **Phase 3: Ship** | Production (BGIPFS) | Frontend deployed to IPFS, CID captured, live app tested, QA checklist passes |

### Model Routing

| Task | Model | Rationale |
|------|-------|-----------|
| Spec parsing, shell commands, env checks, deploy commands | cheap (minimax-m2.7) | JSON extraction, no generation |
| Architecture plan, step extraction, evaluation, tests, frontend, medium fixes | medium (claude-sonnet-4.6) | Reasoning needed but forgiving |
| Contract code gen, complex architecture, semantic review | expensive (claude-opus-4.6) | Solidity correctness is critical |

### Token Limit Guide

| Content type | Default limit | When to increase |
|-------------|---------------|-----------------|
| Single contract (<200 LOC) | 4096 | — |
| 2-3 contracts | 16384 | If output truncated |
| 3+ contracts or complex DeFi | 32768 | Always for complex |
| Tests (any complexity) | 32768 | Tests are verbose |
| Frontend (single page) | 16384 | If 5+ components |

Signs of truncation: JSON parse fails, output ends mid-sentence, only partial files written.

---

## Stage 0: Read the Spec

**Input:** Job description (on-chain, API, or spec file)
**Output:** Mental model + `SPEC_REQUIREMENTS.md`
**Model:** cheap

Before writing a single line of code.

### 0.1 — Read Everything

1. Read the job description (on-chain `description` field, API, or `job.md`)
2. Read ALL client messages — clients add requirements, scope changes, and preferences AFTER posting. Chat is authoritative; the on-chain description is baseline.
3. Extract:
   - **What the app does** (one sentence)
   - **What contracts are needed** (names, functions, interactions)
   - **What external protocols are involved** (exact addresses — verify on-chain, never guess)
   - **What the frontend looks like** (pages, flows, aesthetics)
   - **Who the client is** (wallet address = owner of everything deployed)
   - **What chain** (default: Base, chain ID 8453)
   - **How money flows** (yield mechanism, reward distribution, fee structure)

### 0.2 — Define What Goes Onchain vs Offchain

Put it onchain ONLY if it requires: trustless ownership, trustless exchange, composability with other protocols, censorship resistance, or permanent commitments. Everything else stays offchain.

**Most MVPs need 0-2 smart contracts.** Three is the upper bound for an initial release.

### 0.3 — State Transition Audit

For every planned contract function:

| Function | Who calls it? | Why would they? | What if nobody calls it? | Gas incentive needed? |
|----------|--------------|-----------------|--------------------------|----------------------|

Smart contracts cannot execute themselves. Every function needs a caller who pays gas. If your answer to "who calls it?" is "the team" — redesign.

### 0.4 — Chain Selection

Pick ONE chain. Prove product-market fit first.

| Need | Chain | Why |
|------|-------|-----|
| Consumer/social, AI agents, cheapest gas | Base | Coinbase integration, ERC-8004, Smart Wallet |
| Deepest DeFi liquidity | Arbitrum | GMX, Pendle, widest protocol coverage |
| MEV protection | Unichain | TEE block building |
| Native account abstraction | zkSync Era | Built-in AA |
| Mobile/real-world payments | Celo | MiniPay, sub-cent fees |
| High-value DeFi, governance | Mainnet | Canonical security |

### 0.5 — Identify dApp Archetype

| Archetype | Contracts | Notes |
|-----------|-----------|-------|
| Token launch | 1-2 | Token + optional vesting |
| NFT collection | 1 | ERC-721 + IPFS metadata |
| Marketplace | 0-2 | Often zero — integrate existing DEX |
| Lending/Vault | 0-1 | ERC-4626 vault or wrap existing protocol |
| DAO/Governance | 1-3 | Governor + token + timelock |
| AI agent service | 0-1 | Optional ERC-8004 registration |
| Data dashboard | 0 | Frontend-only, reads from APIs and contracts |

### 0.6 — Write SPEC_REQUIREMENTS.md

List every requirement as a numbered, testable item:

```
1. The vault uses wstETH as the underlying asset
2. Yield is harvested via Uniswap swap: wstETH → WETH → CLAWD
3. 50% of harvested CLAWD is burned, 50% distributed to stakers
4. Client address owns all contracts
```

NOT: "the vault works with ETH staking" (untestable).

### 0.7 — Check for Unknowns

If anything is ambiguous or impossible, escalate NOW. Don't guess. Don't start building until you have clear requirements.

**Validation gate:** You can explain the full user flow, every contract interaction, and every external protocol integration from memory. Every requirement in SPEC_REQUIREMENTS.md is testable.

---

## Stage 1: Discover the Environment

**Input:** Spec, external protocol addresses
**Output:** `discovery.json`
**Model:** cheap (shell commands only)

**This stage prevents hallucinated interfaces.** Before any code is generated, verify what actually exists on-chain.

### 1.1 — Verify Every External Address

For every protocol address in the spec:

```bash
# Does the contract exist?
cast code <address> --rpc-url $ALCHEMY_RPC_URL
# Returns 0x = contract doesn't exist on this chain

# What is it?
cast call <address> "symbol()(string)" --rpc-url $ALCHEMY_RPC_URL
cast call <address> "decimals()(uint8)" --rpc-url $ALCHEMY_RPC_URL
```

**Never hallucinate contract addresses.** Verify with `cast call`:
- Token addresses: `symbol()`, `decimals()`, `name()`
- Pool addresses: `token0()`, `token1()`, `fee()`
- Router addresses: `factory()`, `WETH9()`
- General: `cast code <addr>` — if `0x`, the contract doesn't exist

### 1.2 — Discover On-Chain Interfaces

For every external protocol, probe which functions actually exist:

```bash
cast call <address> "function_signature" --rpc-url $ALCHEMY_RPC_URL
```

Store results in `discovery.json`:

```json
{
  "onChain": {
    "0xc1CBa3fCea344f92D9239c08C0568f6F2F0ee452": {
      "name": "wstETH (Base)",
      "functions": [
        "function balanceOf(address) view returns (uint256)",
        "function approve(address,uint256) returns (bool)",
        "function transfer(address,uint256) returns (bool)",
        "function transferFrom(address,address,uint256) returns (bool)",
        "function decimals() view returns (uint8)"
      ],
      "note": "Bridged ERC20 only. No wrap(), unwrap(), stETH(), or Lido staking functions."
    }
  }
}
```

**What this catches:** Contracts that exist on mainnet with rich interfaces (Lido wstETH with `wrap()`, `unwrap()`, `stETH()`) but are bridged on L2s as simple ERC20s. Without discovery, the LLM writes code against the mainnet interface and the deploy reverts.

### 1.3 — Verify External Protocol Behaviors

For each external protocol:
- Is the token standard ERC20? Any non-standard behaviors (rebasing, fee-on-transfer, blocklist)?
- What pool fee tiers exist? (Uniswap V3: 100, 500, 3000, 10000)
- Is the protocol available on the target chain?
- Verify swap pools exist: `cast call <FACTORY> "getPool(address,address,uint24)(address)" <TOKEN_A> <TOKEN_B> <FEE>`

### 1.4 — Identify Required Skills

| Building with... | Read |
|-----------------|------|
| Any smart contract | `security`, `testing` |
| OpenZeppelin contracts | `openzeppelin` |
| ERC-721 / NFTs | `erc-721` |
| External contract interaction | `standards` |
| Token approvals / DeFi | `frontend-ux` |
| Indexing / historical data | `indexing` |
| Sign-in with Ethereum | `siwe` |
| HTTP payments | `x402` |
| Batch transactions | `eip-5792` |
| Database / off-chain storage | `drizzle-neon` |
| Subgraph indexing | `subgraph` |

All skills: `https://ethskills.com/<skill>/SKILL.md`

**Validation gate:** Every external address verified on-chain. Every protocol behavior confirmed. `discovery.json` written. Skills identified.

---

## Stage 2: Plan

**Input:** Spec, discovery.json, skills
**Output:** `PLAN.md`, `USERJOURNEY.md`, `SPEC_REQUIREMENTS.md`
**Model:** medium

### 2.1 — Write PLAN.md

The plan MUST contain:

1. **One-sentence summary** of what the app does
2. **Architecture diagram** (text-based) showing contract relationships
3. **Contract specifications** for each contract:
   - Name, purpose, inheritance chain
   - State variables with types
   - Every external function with signature, access control, and description
   - Events and custom errors
   - Which external protocols it calls (exact function signatures from discovery.json)
4. **Yield/reward mechanism** (if applicable): where value comes from, how it's captured, how it's converted, how it's distributed
5. **Deploy script** with exact constructor arguments, including all external addresses
6. **Post-deploy configuration** steps
7. **Frontend specification**: pages, components, user flows, which SE2 hooks for each interaction
8. **External addresses** — every address used, verified on-chain
9. **Security considerations** — attack vectors and mitigations
10. **Client address** — explicitly state that `job.client` owns all contracts

### 2.2 — Write USERJOURNEY.md

Document what the user sees and does at every step:

**Happy path:**
1. User opens the app URL → describe landing state (no wallet)
2. User clicks Connect Wallet → what happens
3. User is on wrong network → Switch Network prompt
4. User performs primary action → each click, each transaction, each confirmation
5. User sees result → success state

**Edge cases (must cover ALL):**
- No wallet installed
- Wrong network connected
- Insufficient balance (gas AND token)
- Transaction rejected by user
- Transaction reverted on-chain
- Slow transaction (pending state)
- Multiple rapid clicks (double-submit prevention)
- Mobile wallet via WalletConnect
- Zero balance, max amounts

### 2.3 — Rules for the Plan

**Use only what discovery found.** If discovery shows wstETH on Base is a plain ERC20, the plan must not include `wstETH.wrap()` or `stETH.submit()`. Use alternatives (Uniswap swap instead of Lido wrap).

**Never reference imagined APIs.** Every external call must map to a function signature in `discovery.json`.

**Never reference imagined imports.** Every frontend import must map to a verified package export.

### 2.4 — Spec Verification Gate

Before writing any code, go through SPEC_REQUIREMENTS.md line by line:

1. Find where in PLAN.md each requirement is addressed
2. Verify the plan's approach actually satisfies the requirement

For DeFi projects specifically:
- [ ] **Underlying asset is correct.** If spec says "ETH staking yield," vault MUST use wstETH (appreciates), NOT WETH (doesn't)
- [ ] **Yield mechanism is real.** Walk through with example numbers
- [ ] **Swap path exists on-chain.** Verify every pool exists via factory.getPool()
- [ ] **External addresses are real.** Every address verified with cast call
- [ ] **Owner vs deployer is correct.** Constructor passes CLIENT as owner, not msg.sender
- [ ] **Token decimals are correct.** USDC=6, WETH/wstETH=18

If ANY requirement fails: fix the plan. Do NOT proceed to code.

**Validation gate:** PLAN.md has all 10 sections. USERJOURNEY.md covers happy path AND edge cases. Every external address verified. Spec verification gate passes.

---

## Stage 3: Scaffold

**Input:** Plan
**Output:** SE2 project skeleton
**Model:** cheap (deterministic)

### 3.1 — Create the Project

```bash
npx -y create-eth@latest -s foundry <project-name>
cd <project-name> && yarn install
```

This gives you:
- `packages/foundry/` — Solidity contracts, tests, deploy scripts
- `packages/nextjs/` — Next.js frontend with RainbowKit, wagmi, viem, DaisyUI

**Do NOT manually create Foundry or Next.js projects.** Always use `create-eth`.

### 3.2 — Read AGENTS.md

After scaffolding, read `AGENTS.md` in the project root. It contains the authoritative reference for hooks, components, conventions, and code style. Do not proceed until you've read it.

### 3.3 — Discover Package Exports (Post-Scaffold)

Now that packages are installed, read actual exports:

```bash
# SE2 hooks
cat packages/nextjs/hooks/scaffold-eth/index.ts

# UI components
cat packages/nextjs/node_modules/@scaffold-ui/components/dist/types/index.d.ts

# Scaffold-eth component re-exports
cat packages/nextjs/components/scaffold-eth/index.tsx
```

Store in `discovery.json`:

```json
{
  "packageExports": {
    "hooks": ["useScaffoldReadContract", "useScaffoldWriteContract", "useScaffoldEventHistory", "useDeployedContractInfo", "useTargetNetwork", "useTransactor", "useSelectedNetwork"],
    "uiComponents": {
      "@scaffold-ui/components": ["Address", "AddressInput", "Balance", "EtherInput", "BaseInput"],
      "~~/components/scaffold-eth": ["BlockieAvatar", "Faucet", "FaucetButton", "RainbowKitCustomConnectButton"]
    },
    "importPaths": {
      "Address": "@scaffold-ui/components",
      "Balance": "@scaffold-ui/components",
      "EtherInput": "@scaffold-ui/components",
      "AddressInput": "@scaffold-ui/components",
      "RainbowKitCustomConnectButton": "~~/components/scaffold-eth"
    }
  }
}
```

### 3.4 — Read Scaffolded Structure

```bash
# Deploy pattern
cat packages/foundry/script/DeployHelpers.s.sol
cat packages/foundry/script/Deploy.s.sol

# CSS setup
head -5 packages/nextjs/styles/globals.css

# Next.js and scaffold config
cat packages/nextjs/next.config.ts
cat packages/nextjs/scaffold.config.ts
```

### 3.5 — Delete Scaffold Defaults

```bash
rm packages/foundry/contracts/YourContract.sol
rm packages/foundry/script/DeployYourContract.s.sol
rm packages/foundry/test/YourContract.t.sol
```

**Validation gate:** SE2 project structure exists. `forge build` compiles. Package exports discovered and stored.

---

## Stage 4: Write Contracts

**Input:** PLAN.md, discovery.json
**Output:** Compiled Solidity contracts
**Model:** expensive (contract gen), medium (deploy scripts)

### 4.1 — Contract Architecture Rules

- **Use OpenZeppelin.** Check `packages/foundry/lib/openzeppelin-contracts/contracts/` for what's installed before writing custom implementations.
- **Ownable2Step over Ownable.** Two-step ownership prevents fat-finger mistakes.
- **ReentrancyGuard** on all external-facing state-changing functions.
- **CEI pattern (Checks-Effects-Interactions).** State changes before external calls.
- **SafeERC20** for all token operations. `safeTransfer`, `safeTransferFrom`, `forceApprove`.
- **Custom errors over require strings.** `error ZeroAddress();` not `require(addr != address(0), "zero")`.
- **Emit events for every state change.** Events are your frontend API.
- **Never use `tx.origin`.** Never use infinite approvals. Never use `type(uint256).max` for approvals.
- **Add NatSpec** to all public/external functions.
- **Use `immutable`/`constant`** where appropriate.
- **Cap pagination limits** to prevent gas DoS.

### 4.2 — External Protocol Integration

- **Only call functions that exist in discovery.json.** If the ABI doesn't show `wrap()`, don't call `wrap()`.
- **On L2s, bridged tokens are plain ERC20s.** They don't have the rich interfaces their L1 originals have.
- **For token conversions on L2, use Uniswap swaps.** Not Lido wrap/unwrap, not native bridge calls.
- Write interface files for external protocols — only the functions you actually call.

### 4.3 — ERC4626 Vaults (if applicable)

- **Virtual shares / dead shares for inflation protection.** `_decimalsOffset()` >= 3.
- **Override `_deposit` and `_withdraw`** (internal hooks), NOT `deposit` and `withdraw` (public functions).
- **Convenience deposit functions** (depositETH, depositWETH): Use `previewDeposit()` + `_mint()` directly.
- **Underlying asset must be consistent.** If vault holds wstETH, ERC4626 asset IS wstETH.
- Track principal separately from yield if yield is redirected.

### 4.4 — Uniswap V3 Swaps (if applicable)

- Always set `amountOutMinimum` > 0 (slippage protection)
- Always set `deadline` (block.timestamp for on-chain)
- TWAP oracle check BEFORE the swap
- Wrap `pool.observe()` in try/catch (new pools may lack history)
- Verify pool exists via factory before coding the swap path

### 4.5 — Access Control

- All privileged roles (owner, admin, treasury, governor) MUST be set to `job.client`
- Never hardcode addresses. Use constructor parameters.
- If a keeper/harvester role is needed, add it as a separate role the owner can set
- **Walkaway test:** if the owner disappears, can users still withdraw? The answer must be yes.
- Use deployer-first pattern for cross-contract configuration: deploy with deployer as temp owner, configure cross-references, then transfer ownership to client.

### 4.6 — Write Deploy Scripts

Location: `packages/foundry/script/`

- **Always inherit `ScaffoldETHDeploy`.** Use `ScaffoldEthDeployerRunner` modifier on `run()`.
- **Deploy contracts inline.** `new MyContract(args)` inside `run()`.
- **Push to `deployments` array.** `deployments.push(Deployment("MyContract", address(myContract)));`
- **Wire addresses after all deploys.** Deploy A, deploy B(address(A)), then A.setB(address(B)).
- **Constructor args: use `vm.envOr`** for addresses with real mainnet address as default.
- **Zero-address checks** in every constructor.
- **RPC endpoints: always use Alchemy.** Never public RPCs.

### 4.7 — Write Tests

Location: `packages/foundry/test/`

Test priority:
1. **Unit tests** — edge cases, failure modes, access control. NOT getters.
2. **Fuzz tests** — any function with math. Minimum 1000 runs. Use `bound()` not `vm.assume()`.
3. **Fork tests** — any interaction with external protocols.
4. **Invariant tests** — stateful protocols.

Required coverage:
- Constructor state: all immutables set correctly
- Happy path for every function
- Full lifecycle: deploy → interact → verify state
- Reverts: unauthorized callers, invalid inputs, insufficient balances, reentrancy attempts
- Edge cases: zero amounts, max amounts, empty strings

**Critical rule for test failures:** If a test assertion fails during auto-fix, fix the test, not the contract. Don't break working contract logic to make a bad test pass.

### 4.8 — Register External Contracts

If the frontend reads from contracts NOT deployed by the pipeline, register in `packages/nextjs/contracts/externalContracts.ts`:

```typescript
import { GenericContractsDeclaration } from "~~/utils/scaffold-eth/contract";

const externalContracts = {
  8453: {  // Chain ID as number key
    WETH: {
      address: "0x4200000000000000000000000000000000000006",
      abi: [/* only functions the frontend calls */],
    },
  },
} as const;

export default externalContracts satisfies GenericContractsDeclaration;
```

**Rules:**
- Chain ID MUST be a number key (not string)
- Only include ABI entries the frontend actually calls
- Never manually edit `deployedContracts.ts` — it's auto-generated by `yarn deploy`

### 4.9 — Compile and Test

```bash
cd packages/foundry
forge build     # Must compile with zero errors
forge test -vvv # Must pass all tests
```

**Validation gate:** All contracts compile. All tests pass. Deploy script follows ScaffoldETHDeploy pattern. External contracts registered.

---

## Stage 5: Contract Audit

**Input:** Compiled contracts
**Output:** Audit report, GitHub issues
**Model:** medium (audit analysis)

### 5.1 — Standard Audit Checklist

Read every contract file. Check:

**Critical checks:**
1. Reentrancy — guards on all external-calling functions
2. Access control — Ownable2Step correct, no leftover deployer privileges
3. Integer safety — overflow/underflow, unsafe casting
4. External call safety — return values checked, CEI pattern followed
5. Token handling — SafeERC20, no raw `transfer()`
6. Slippage protection — all swaps have minimum output
7. Oracle safety — TWAP checks, manipulation resistance
8. First depositor attacks — virtual shares for ERC4626
9. Input validation — zero addresses, zero amounts, bounds
10. Events emitted for all state changes
11. No infinite approvals
12. Token decimals handled correctly (USDC=6, WETH=18)
13. Multiply before divide (no precision loss)

**DeFi-specific checks:**
14. Flash loan vectors — state manipulation in a single block?
15. Sandwich attack vectors — swaps protected?
16. Yield accounting — can principal and yield drift?
17. Walkaway safety — can users always withdraw?
18. Price manipulation — spot price vs TWAP divergence
19. Zero-staker edge cases in rewards contracts
20. Rounding errors at scale

### 5.2 — File GitHub Issues

For every finding (Medium severity or above):

```bash
gh issue create --repo <repo> \
  --title "[SEVERITY] Finding title" \
  --body "**Location:** file:function\n**Description:** What's wrong\n**Recommendation:** Concrete fix" \
  --label "job-<id>,contract-audit"
```

### 5.3 — Deep Audit (for complex contracts)

**Skip if simple:** <100 lines, no swaps, no reentrancy vectors, no complex access control.

**Do if complex:** Token swaps, multi-contract interactions, financial operations, >200 lines. Fetch and follow `https://ethskills.com/audit/SKILL.md` — 19 security domains, 500+ checklist items, parallel specialist agents.

### 5.4 — Fix All Findings

1. Fix by severity: Critical (must fix) → High (must fix) → Medium (fix or document) → Low (fix if trivial)
2. Close each issue with commit reference
3. Re-run `forge build && forge test` after each fix
4. Zero Critical/High findings remain open

**Validation gate:** All Critical/High fixed. All Medium addressed. Tests still pass.

---

## Stage 6: Build Frontend

**Input:** Compiled contracts, deployedContracts.ts, PLAN.md, USERJOURNEY.md, discovery.json
**Output:** Working frontend
**Model:** medium (frontend gen), expensive (complex pages)

### 6.1 — Configuration (Before Writing Components)

**scaffold.config.ts:**
```typescript
targetNetworks: [chains.base],      // or target chain
pollingInterval: 3000,              // NOT 30000
```

During local dev with fork: use `chains.foundry` (chain ID 31337).

**wagmiConnectors.tsx:**
- Change `appName` from `"scaffold-eth-2"` to your app name
- Add `phantomWallet` to wallets array

### 6.2 — SE2 Branding Cleanup (Mandatory)

Every item must be addressed. AI agents treat the scaffold as sacred — don't.

- [ ] **Footer.tsx** — Remove "Fork me", "Built with heart at BuidlGuidl", "Support" links, `nativeCurrencyPrice` badge
- [ ] **Header.tsx** — Replace SE2 logo and name, remove "Debug Contracts" nav link
- [ ] **layout.tsx / getMetadata.ts** — Change title from "Scaffold-ETH 2", update description
- [ ] **README.md** — Replace entirely with project content
- [ ] **Favicon** — Replace SE2 default
- [ ] **manifest.json** — Update app name
- [ ] **blockexplorer** — Rename to `_blockexplorer-disabled` (crashes static export)
- [ ] **debug page** — Remove `app/debug/` (uses `force-dynamic`, incompatible with IPFS export)

### 6.3 — Styling Rules

**DaisyUI semantic classes:**
```tsx
// CORRECT — responds to light/dark theme toggle
<div className="min-h-screen bg-base-200 text-base-content">
<div className="card bg-base-100 shadow-xl">
<button className="btn btn-primary">

// WRONG — hardcoded dark background, ignores theme system
<div className="min-h-screen bg-[#0a0a0a] text-white">
```

**No hardcoded dark backgrounds.** Use `bg-base-100`, `bg-base-200`, `bg-base-300`, `text-base-content`.

**Tailwind v4 CSS imports:** `@import "tailwindcss";` not v3 `@tailwind base;` directives.

**No `@apply` with DaisyUI tokens.** Use className attributes only.

**Fix pill-shaped inputs:** Change `--radius-field` from `9999rem` to `0.5rem` in `packages/nextjs/styles/globals.css` in BOTH theme blocks (light and dark).

### 6.4 — Import Rules

**Use discovery.json.packageExports.importPaths.** Don't guess import paths.

```typescript
// SE2 hooks
import { useScaffoldReadContract } from "~~/hooks/scaffold-eth";
import { useScaffoldWriteContract } from "~~/hooks/scaffold-eth";
import { useScaffoldEventHistory } from "~~/hooks/scaffold-eth";
import { useDeployedContractInfo } from "~~/hooks/scaffold-eth";

// UI components
import { Address, Balance, EtherInput, AddressInput } from "@scaffold-ui/components";

// Connect button
import { RainbowKitCustomConnectButton } from "~~/components/scaffold-eth";

// Utilities
import { notification, getParsedError } from "~~/utils/scaffold-eth";
```

**Path alias:** `~~` maps to `packages/nextjs/` root. Always use it.

### 6.5 — Component Rules

- **Every page component must be `"use client"`.** SE2 uses Next.js App Router.
- **`export default` required** on every page/component.
- **Never use raw wagmi hooks** for contract interaction. Always `useScaffoldReadContract`, `useScaffoldWriteContract`, `useScaffoldEventHistory`.
- **Never edit `deployedContracts.ts`** manually. It's auto-generated.
- **Never hardcode deployed contract addresses as string literals.** Components that need a deployed address MUST call `useDeployedContractInfo("ContractName")` internally.
- **Never pass deployed addresses as props.** `page.tsx` composes components, doesn't know addresses.

### 6.6 — Four-State Wallet Flow (Mandatory)

The app must show exactly ONE primary button at a time:

```
1. Not connected  → Connect Wallet button (RainbowKitCustomConnectButton)
2. Wrong network  → Switch to [Chain] button
3. Needs approval → Approve button (with dual-state locking)
4. Ready          → Action button (Stake/Deposit/Swap)
```

**Never show Approve and Action buttons simultaneously.** Never skip the network check.

```tsx
import { useAccount, useChainId, useSwitchChain } from "wagmi";
import { useConnectModal } from "@rainbow-me/rainbowkit";

const { isConnected } = useAccount();
const chainId = useChainId();
const { switchChain } = useSwitchChain();
const { openConnectModal } = useConnectModal();

if (!isConnected) return <button onClick={() => openConnectModal?.()}>Connect Wallet</button>;
if (chainId !== base.id) return <button onClick={() => switchChain({ chainId: base.id })}>Switch to Base</button>;
if (!hasAllowance) return <ApproveButton />;
return <ActionButton />;
```

### 6.7 — Approve Button: Dual-State Protection (Critical)

`isPending` from wagmi clears when the wallet returns the tx hash — NOT when the tx confirms on-chain. This creates a window where the button re-enables mid-flight.

```tsx
const [approvalSubmitting, setApprovalSubmitting] = useState(false);
const [approveCooldown, setApproveCooldown] = useState(false);

const handleApprove = async () => {
  if (approvalSubmitting || approveCooldown) return;
  setApprovalSubmitting(true);
  try {
    await approveWrite({ functionName: "approve", args: [spender, amount] });
    setApproveCooldown(true);
    setTimeout(() => { setApproveCooldown(false); refetchAllowance(); }, 4000);
  } catch (e) {
    notifyError("Approval failed");
  } finally {
    setApprovalSubmitting(false);  // MUST be in finally — handles rejection
  }
};

<button disabled={isPending || approvalSubmitting || approveCooldown}>
  {(isPending || approvalSubmitting) && <span className="loading loading-spinner loading-sm mr-2" />}
  {isPending || approvalSubmitting ? "Approving..." : "Approve"}
</button>
```

- `approvalSubmitting` covers click-to-hash gap
- `approveCooldown` covers confirm-to-cache-refresh gap (4 seconds)
- Both must be on the `disabled` prop
- `approvalSubmitting` MUST clear in `finally {}` (handles wallet rejection)

### 6.8 — Button Loading States

```tsx
// WRONG — DaisyUI "loading" class replaces content with full-width spinner
<button className={`btn btn-primary ${isPending ? "loading" : ""}`}>

// CORRECT — inline spinner, text stays visible
<button className="btn btn-primary" disabled={isPending}>
  {isPending && <span className="loading loading-spinner loading-sm mr-2" />}
  {isPending ? "Staking..." : "Stake"}
</button>
```

### 6.9 — SE2 Components (Always Use These)

| Need | Component | Never use |
|------|-----------|-----------|
| Display address | `<Address />` | Raw truncated hex string |
| Input address | `<AddressInput />` | `<input type="text" placeholder="0x..." />` |
| Display balance | `<Balance />` | Raw wei number |
| Input ETH amount | `<EtherInput />` | `<input type="number" />` |

### 6.10 — Display Standards

- **USD values everywhere.** Every token/ETH amount: `0.5 ETH (~$1,250)`
- **Contract address displayed.** Using `<Address />` component.
- **Human-readable amounts.** `formatEther()` / `parseEther()` from viem. Never show raw wei.
- **Human-readable errors.** Map every contract revert to a user-facing message using `getParsedError`.

```typescript
import { getParsedError } from "~~/utils/scaffold-eth";
import { notification } from "~~/utils/scaffold-eth";

try {
  await writeContractAsync({ functionName: "...", args: [...] });
} catch (e) {
  const parsed = getParsedError(e);
  notification.error(parsed.includes("rejected") ? "Transaction rejected" : parsed);
}
```

### 6.11 — Mobile Deep Linking

RainbowKit v2 does NOT auto-deep-link to wallet apps. You must implement it:

```typescript
const writeAndOpen = useCallback(
  <T,>(writeFn: () => Promise<T>): Promise<T> => {
    const promise = writeFn(); // fire TX first
    setTimeout(openWallet, 2000); // deep link after relay
    return promise;
  },
  [openWallet],
);
```

Rules:
1. Fire TX first, deep link second. Never `window.location.href` before the write call.
2. Skip deep link if `window.ethereum` exists (in-app browser).
3. Check WalletConnect session data in localStorage — `connector.id` alone won't tell you which wallet.
4. Wrap EVERY write call — approve, action, claim, batch.

### 6.12 — API Routes (if needed)

Location: `packages/nextjs/app/api/<name>/route.ts`

For external API proxying, caching, or server-side logic. Secrets stay server-side.

### 6.13 — Build and Verify

```bash
yarn next:build
```

**Validation gate:** `yarn next:build` succeeds. App loads at localhost:3000. All flows work. No console errors.

---

## Stage 7: Frontend QA

**Input:** Built frontend
**Output:** QA report with PASS/FAIL per item
**Model:** medium

### Ship-Blocking Checks (must ALL pass before proceeding)

- [ ] Wallet connection shows a BUTTON, not text ("Please connect your wallet" = fail)
- [ ] Wrong network shows a Switch button
- [ ] One button at a time (Connect → Network → Approve → Action)
- [ ] Approve button locked with both `approvalSubmitting` AND `approveCooldown`
- [ ] SE2 footer branding removed (no BuidlGuidl links, no "Built with SE2")
- [ ] Tab title is the app name, NOT "Scaffold-ETH 2"
- [ ] SE2 README replaced with project documentation
- [ ] Contracts verified on block explorer
- [ ] No zero-address placeholders (`0x000...0`) in components
- [ ] No hardcoded deployed addresses as string literals in React files
- [ ] No raw wagmi hooks outside scaffold-eth internals

### Should-Fix Checks (report but don't block)

- [ ] Contract address displayed with `<Address />`
- [ ] Every address input uses `<AddressInput />`
- [ ] USD values next to all token/ETH amounts
- [ ] OG image is absolute production URL (not relative path)
- [ ] `pollingInterval` is 2000-3000 (not 30000)
- [ ] RPC overrides set AND env vars confirmed on hosting platform
- [ ] Favicon updated from SE2 default
- [ ] `--radius-field` changed from `9999rem` to `0.5rem`
- [ ] Contract errors mapped to human-readable messages
- [ ] No hardcoded dark backgrounds — uses `bg-base-200 text-base-content`
- [ ] Button loaders use inline spinner, not DaisyUI `loading` class
- [ ] Phantom wallet in RainbowKit wallet list
- [ ] Mobile deep linking for all transaction buttons
- [ ] Mobile wallet detection checks WC session data

### Automated Checks

```bash
# Raw wagmi hooks (should only be in scaffold-eth internals)
grep -rn "useWriteContract\|useReadContract\|useContractEvent" packages/nextjs/app/

# Raw text inputs for addresses
grep -rn 'type="text"' packages/nextjs/app/ | grep -i "addr\|owner\|recip\|0x"

# Hardcoded dark backgrounds
grep -rn 'bg-\[#0\|bg-black\|bg-gray-9\|bg-zinc-9' packages/nextjs/app/

# DaisyUI loading class on buttons
grep -rn '"loading"' packages/nextjs/app/

# Leaked secrets
git diff --cached | grep -iE "apikey|api_key|secret|password|0x[a-fA-F0-9]{64}"
```

Fix all ship-blocking items. File GitHub issues for findings with label `frontend-audit`.

**Validation gate:** All ship-blocking items pass. Should-fix items have a plan.

---

## Stage 8: Full Integration Audit

**Input:** All code — contracts, frontend, deploy scripts, tests, configuration
**Output:** Final audit report
**Model:** medium

Final review across ALL components together:

- [ ] Every requirement in SPEC_REQUIREMENTS.md is implemented and working
- [ ] No critical security defects
- [ ] No risk of locked or lost funds
- [ ] No stub functions or TODO comments
- [ ] All external addresses verified on-chain
- [ ] Deploy script constructor args match contract constructors
- [ ] Frontend reads/writes match contract function signatures
- [ ] Event names in useScaffoldEventHistory match contract events
- [ ] scaffold.config.ts targets the correct chain
- [ ] No hardcoded private keys or API keys in any file
- [ ] `.gitignore` excludes `.env`, `node_modules`, `out/`, `cache/`, `broadcast/`
- [ ] No secrets in the repo

Create GitHub issues with label `full-audit`. Fix all findings.

**Validation gate:** Zero open findings. All code consistent across contracts and frontend.

---

## Stage 9: Deploy Contracts to Live Network (Phase 2)

**Input:** Audited, tested contracts
**Output:** Deployed, verified contracts on live chain
**Model:** cheap (shell commands)

### 9.1 — Configure RPC

In `packages/foundry/foundry.toml`:
```toml
[rpc_endpoints]
base = "${ALCHEMY_RPC_URL}"
```

**NEVER use public RPCs.** Always Alchemy with API key. If `ALCHEMY_API_KEY` is not available, STOP.

### 9.2 — Deploy

```bash
cd packages/foundry
forge script script/Deploy.s.sol \
  --rpc-url <ALCHEMY_RPC_URL> \
  --account agent-deployer --password agent \
  --broadcast --ffi
node scripts-js/generateTsAbis.js
```

Or via SE2: `yarn deploy --network base`

### 9.3 — Verify on Block Explorer

```bash
yarn verify --network base
```

**Every deployed contract must show verified source with a green checkmark.** Unverified contracts are a trust red flag.

Check manually: open each address on Basescan → "Contract" tab → look for checkmark.

### 9.4 — Verify On-Chain State

```bash
# Check ownership
cast call <address> "owner()(address)" --rpc-url $ALCHEMY_RPC_URL

# Check cross-references
cast call <vault> "harvester()(address)" --rpc-url $ALCHEMY_RPC_URL

# Check immutables
cast call <vault> "asset()(address)" --rpc-url $ALCHEMY_RPC_URL
```

### 9.5 — Verify deployedContracts.ts

Check that `packages/nextjs/contracts/deployedContracts.ts` was auto-generated with:
- Correct chain ID
- Real addresses (not `0x000...0`)
- Complete ABIs
- `deployedOnBlock` field

### 9.6 — Post-Deploy Configuration

If contracts need cross-references (setHarvester, setRewardDistributor):
- Call configuration functions while deployer is still owner
- Transfer ownership to client address
- Client must call `acceptOwnership()` on each contract (Ownable2Step)

### 9.7 — Test with Real Wallet

- Run the app locally: `yarn start`
- Connect a real wallet, test with small amounts ($1-10)
- Verify all flows end-to-end

**Validation gate:** All contracts deployed and verified. deployedContracts.ts updated. Cross-references configured. Ownership transferred. Live wallet test passes.

---

## Stage 10: Deploy Frontend to IPFS (Phase 3)

**Input:** Deployed contracts, working frontend
**Output:** Live production frontend on IPFS
**Model:** cheap (shell commands)

### 10.1 — Pre-Deploy Checklist

- [ ] `scaffold.config.ts` targets production network (not foundry/localhost)
- [ ] `targetNetworks` set to `[chains.base]` (or target chain)
- [ ] `burnerWalletMode: "localNetworksOnly"` (no burner wallet in production)
- [ ] RPC env vars set on hosting platform
- [ ] OG image is absolute production URL, 1200x630px
- [ ] SE2 branding fully removed
- [ ] `NEXT_PUBLIC_PRODUCTION_URL` set
- [ ] No secrets in committed code

### 10.2 — Remove Incompatible Pages

```bash
rm -rf packages/nextjs/app/debug/
rm -rf packages/nextjs/app/blockexplorer/
```

These use `force-dynamic` which is incompatible with `output: "export"` required by IPFS.

### 10.3 — CSS Sanitization

Replace Tailwind v3 directives with v4:
```css
/* v3 (remove) */
@tailwind base;
@tailwind components;
@tailwind utilities;

/* v4 (use) */
@import "tailwindcss";
```

Strip any `@apply` with DaisyUI tokens.

### 10.4 — Build

```bash
cd packages/nextjs
rm -rf .next out

NEXT_PUBLIC_IPFS_BUILD=true \
NEXT_PUBLIC_IGNORE_BUILD_ERROR=true \
NEXT_PUBLIC_PRODUCTION_URL="https://yourapp.yourname.eth.link" \
NODE_OPTIONS="--require ./polyfill-localstorage.cjs" \
yarn build
```

**localStorage polyfill** (for Node 25+): Create `packages/nextjs/polyfill-localstorage.cjs`:

```javascript
if (typeof globalThis.localStorage !== "undefined" &&
    typeof globalThis.localStorage.getItem !== "function") {
  const store = new Map();
  globalThis.localStorage = {
    getItem: (key) => store.get(key) ?? null,
    setItem: (key, value) => store.set(key, String(value)),
    removeItem: (key) => store.delete(key),
    clear: () => store.clear(),
    key: (index) => [...store.keys()][index] ?? null,
    get length() { return store.size; },
  };
}
```

### 10.5 — IPFS Routing Requirements

Three conditions must be met:
1. `output: "export"` in `next.config.ts`
2. `trailingSlash: true` — IPFS gateways resolve directories to `index.html` but not bare filenames
3. No pages that crash during prerender — crashed pages get silently skipped, producing 404s

### 10.6 — Upload to IPFS

```bash
yarn ipfs
```

### 10.7 — Validate the Upload

`yarn ipfs` exit code is NOT reliable. The script swallows errors.

**The only valid signal is an IPFS CID in the output:**
- `Qm` followed by 44 alphanumeric characters, OR
- `bafy` followed by 50+ lowercase alphanumeric characters

If no CID appears in stdout/stderr, the upload failed regardless of exit code.

### 10.8 — Verify the Deployment

- Open the IPFS gateway URL in a browser
- Check CID actually changed from last deploy
- Test wallet connection, contract interactions, all flows
- Verify OG image renders when sharing the URL
- Test on mobile

### 10.9 — ENS Subdomain (if applicable)

Two mainnet transactions:
1. Create subdomain at `app.ens.domains/yourname.eth` → "Subnames"
2. Set content hash: "Records" → "Other" → paste `ipfs://{CID}`

Use `.eth.link` (not `.eth.limo`) for better mobile support.

**Validation gate:** IPFS CID captured. Live app loads. All flows work. OG image renders.

---

## Stage 11: Live User Journey Walkthrough

**Input:** Live app (IPFS URL or ENS subdomain)
**Output:** Walkthrough report
**Model:** cheap

Open the live app in a browser with a real wallet. Follow USERJOURNEY.md step by step as a real user.

- Actually click every button
- Actually connect your wallet
- Actually submit transactions
- Actually verify the results

If ANYTHING is broken:

**Phase regression rules:**
- Bug in production frontend → return to Phase 2 (fix locally, redeploy frontend)
- Bug in contract logic → return to Phase 1 (fix, retest, redeploy everything)
- Never patch production bugs without proper phase regression

**Validation gate:** Entire user journey works perfectly on the live app.

---

## Stage 12: Final QA

**Input:** Live deployed app
**Output:** Final QA report (pass/fail per item)
**Model:** medium

Re-run the full QA checklist from Stage 7 against the LIVE deployment. Plus:

- [ ] BGIPFS deployed — production frontend is on IPFS, not Vercel
- [ ] CID is correct and resolves
- [ ] App works on mobile browser
- [ ] No console errors in production
- [ ] Verify the live CID matches what was just uploaded

**Validation gate:** All ship-blocking items pass on live deployment.

---

## Stage 13: Documentation & Delivery

**Input:** Completed, deployed app
**Output:** README, delivery notification
**Model:** cheap

### 13.1 — Write README

Include:
- What the app does (2-3 sentences)
- Contract addresses and chain
- How to run locally (`yarn fork`, `yarn deploy`, `yarn start`)
- Architecture decisions and non-obvious implementation details
- Link to the live app

Do NOT include:
- Explanations of what React, Solidity, or Ethereum are
- SE2 template boilerplate
- Padding or filler content

### 13.2 — Deliver

1. Push all code to GitHub
2. Upload deliverables to IPFS
3. Report the live working app URL
4. The job is complete

---

## Reference: Secrets Management

**Never commit secrets.** No exceptions.

Before every commit:
```bash
git diff --cached | grep -iE '\.env|key|secret|private'
grep -rn "0x[a-fA-F0-9]{64}" packages/
grep -rn "g.alchemy.com/v2/[A-Za-z0-9]" packages/
```

Ensure `.gitignore` includes: `.env`, `.env.*`, `*.key`, `*.pem`, `broadcast/`, `cache/`

**If you accidentally commit a secret, treat it as already compromised. Rotate it immediately.**

---

## Reference: RPC Rules

**NEVER use public RPCs** (`mainnet.base.org`, `base.llamarpc.com`, `eth.llamarpc.com`, etc.) for any chain calls. Always use Alchemy endpoints with an API key.

If `ALCHEMY_API_KEY` is not available in `.env` or `foundry.toml`, STOP. Do not fall back to a public RPC.

---

## Reference: Client Ownership

Every privileged role in every contract MUST be set to `job.client`:
- `owner`, `admin`, `deployer`, `feeOwner`, `treasury`, `governor`
- Constructor args that take an admin/owner address
- `transferOwnership` calls
- Multisig signer slots

**Never use your own address or any internal wallet as owner.**

---

## Reference: Dependency Graph (Step Execution)

```
scaffold ─> contracts ─> deploy script ─> compile ─┬─> tests (terminal branch — blocks nothing)
                                                     │
                                                     ├─> frontend components ─> page/layout ─> build ─────────┐
                                                     │                                                         │
                                                     └─> deploy base ─> verify ──────────────────────────────> rebuild ─> BGIPFS
```

Rules:
- `tests` depends on `compile`. Nothing depends on `tests`.
- `deploy --network base` depends on `compile`. NOT on tests, NOT on frontend.
- `frontend components` depends on `compile` (so deployedContracts.ts structure exists).
- `rebuild frontend` depends on `verify` (real addresses in deployedContracts.ts) AND `build` (frontend compiles).
- `yarn ipfs` depends on `rebuild frontend`.

---

## Reference: Auto-Fix Strategies

| Error type | Fix strategy |
|------------|-------------|
| Solidity compiler error | Parse error location, send broken file + error to LLM, get fixed file |
| Forge test failure (assertion) | Fix the test, NOT the contract |
| Forge test failure (EVM revert) | Diagnose whether bug is in contract or test |
| Module not found (npm) | `yarn add <package>`, retry build |
| Module not found (local alias) | Fix import path using discovery.json packageExports |
| TypeScript error | Send broken file + error to LLM |
| `yarn ipfs` no CID | Check if debug/blockexplorer pages removed. Remove and retry. |
| Deploy revert | Check constructor args against discovery.json ABIs |
| CSS build error | Replace v3 directives with v4, strip @apply with DaisyUI tokens |

---

## Reference: Common Failure Modes

| Failure | Root Cause | Prevention |
|---------|-----------|------------|
| Deploy reverts calling `wrap()` on L2 wstETH | Bridged tokens are plain ERC20s | Discovery stage probes real interfaces |
| Frontend imports from wrong path | Import path guessing | Discovery reads actual package exports |
| `deployedContracts.ts` has `0x000...0` | LLM wrote placeholder addresses | Never edit manually; auto-generated by `yarn deploy` |
| Address passed as prop to component | Wrong abstraction | Component calls `useDeployedContractInfo` internally |
| Approval button re-enables mid-flight | Only checking `isPending` | Dual-state: `approvalSubmitting` + `approveCooldown` |
| Dark mode breaks | Hardcoded `bg-[#0a0a0a]` | Use `bg-base-200 text-base-content` |
| Pill-shaped inputs | `--radius-field: 9999rem` default | Change to `0.5rem` in both theme blocks |
| IPFS 404 on subpages | Missing `trailingSlash: true` | Verify next.config.ts before IPFS deploy |
| `localStorage.getItem is not a function` | Node 25+ SSG | Add polyfill to NODE_OPTIONS |
| Page with dynamic couldn't be exported | debug/blockexplorer pages not removed | Delete before IPFS build |
| Token truncation in LLM output | Token limit too low | Increase to 16384/32768 |
| Test fixes break contract logic | LLM fixes contract to pass bad test | Rule: fix the test, not the contract |

---

## Reference: Verified Contract Addresses (Base, Chain ID 8453)

| Contract | Address |
|----------|---------|
| WETH | `0x4200000000000000000000000000000000000006` |
| wstETH | `0xc1CBa3fCea344f92D9239c08C0568f6F2F0ee452` |
| USDC | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` |
| DAI | `0x50c5725949A6F0c72E6C4a641F24049A917DB0Cb` |
| Uniswap V3 Router | `0x2626664c2603336E57B271c5C0b26F421741e481` |
| Uniswap V3 Factory | `0x33128a8fC17869897dcE68Ed026d694621f6FDfD` |

Always cross-reference any address in deploy scripts against this table. **LLMs hallucinate addresses.**

---

## Reference: Complete File Inventory

Every completed build should produce:

```
builds/job-{id}-{timestamp}/
  SPEC_REQUIREMENTS.md        # Testable requirements
  discovery.json              # On-chain ABIs, package exports
  PLAN.md                     # Architecture plan
  USERJOURNEY.md              # Step-by-step user flow
  steps.json                  # Executable step DAG (pipeline mode)
  evaluation.json             # Plan score (pipeline mode)
  execution-log.json          # Per-step results (pipeline mode)
  qa-report.json              # QA checklist results
  skills/                     # Cached SKILL.md files
  project/                    # The actual dApp
    packages/foundry/
      contracts/              # Solidity source
      script/                 # Deploy scripts
      test/                   # Forge tests
      broadcast/              # Deploy transaction logs
    packages/nextjs/
      app/                    # Next.js pages
      components/             # React components
      contracts/              # deployedContracts.ts, externalContracts.ts
      styles/                 # CSS
```

---

## Reference: Skill URLs

All skills at `https://ethskills.com/<name>/SKILL.md`:

| Skill | When to read |
|-------|-------------|
| `ship` | Before starting any dApp |
| `orchestration` | Before planning phases |
| `security` | Before writing any contract |
| `testing` | Before writing tests |
| `openzeppelin` | When using OZ contracts |
| `erc-721` | When building NFTs |
| `standards` | For ERC-8004, x402, EIP-7702 |
| `frontend-ux` | Before building any UI with wallet interaction |
| `frontend-playbook` | Before deploying frontend |
| `qa` | After deployment, before sharing |
| `audit` | For high-value contract security review |
| `gas` | When estimating costs or choosing chains |
| `l2s` | When choosing or deploying to L2 |
| `tools` | When setting up dev environment |
| `wallets` | When handling keys, multisig, account abstraction |
| `concepts` | When designing incentives or explaining to users |
| `indexing` | When you need historical onchain data |
| `money-legos` | When composing with DeFi protocols |

SE2 docs: `https://docs.scaffoldeth.io/SKILL.md`
SE2 AGENTS.md: `https://github.com/scaffold-eth/scaffold-eth-2/blob/main/AGENTS.md`

---

## Reference: Quick Commands

```bash
# Development
yarn chain                     # Start local Anvil
yarn fork --network base       # Fork Base mainnet
cast rpc anvil_setIntervalMining 1  # Enable block mining
yarn deploy                    # Deploy contracts locally
yarn start                     # Start frontend (localhost:3000)

# Quality
forge test -vvv                # Run contract tests
forge test --match-test testFuzz  # Fuzz tests only
slither packages/foundry/contracts/  # Security analysis
yarn next:build                # Build frontend

# Production
yarn deploy --network base     # Deploy to live network
yarn verify --network base     # Verify on block explorer
yarn ipfs                      # Deploy frontend to IPFS

# Verification
cast call <addr> "owner()(address)" --rpc-url $ALCHEMY_RPC_URL
cast call <addr> "symbol()(string)" --rpc-url $ALCHEMY_RPC_URL
cast code <addr> --rpc-url $ALCHEMY_RPC_URL
```

---

## What This Playbook Does NOT Cover

- **Subgraph/indexer deployment.** The Graph integration is a separate pipeline.
- **Chainlink Automation setup.** Keeper registration is a post-deploy operational task.
- **ENS subdomain configuration.** Requires manual mainnet transactions.
- **Multi-chain deploys.** Assumes a single target chain per build.
- **Upgradeable contracts (proxies).** Adds deploy complexity not covered here.
- **EIP-5792 batch transactions.** Read the `eip-5792` skill when needed.
- **SIWE (Sign-In with Ethereum).** Read the `siwe` skill when needed.

These are extensions. Get the base pipeline working first.
