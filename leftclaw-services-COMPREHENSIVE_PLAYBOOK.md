# Comprehensive dApp Build Playbook

From job spec to finished, deployed, audited decentralized application. Every step, every gate, every verification. No shortcuts, no trust, no "it should work."

**The cardinal rule: do the thing, verify the thing works, verify the thing does what it is supposed to without stubs, fix anything repeatedly until solid, then continue.**

AI is sloppy. AI hallucinates. AI says "done" when it isn't. AI skips tests. AI doesn't double-check. This playbook exists because of that. Every stage has explicit verification gates. Every gate has concrete pass/fail criteria. No stage advances on vibes.

---

## Table of Contents

1. [Philosophy](#1-philosophy)
2. [Pipeline Overview](#2-pipeline-overview)
3. [Phase 0: Architecture & Planning](#3-phase-0-architecture--planning)
4. [Phase 1: Scaffold & Discovery](#4-phase-1-scaffold--discovery)
5. [Phase 2: Smart Contract Development](#5-phase-2-smart-contract-development)
6. [Phase 3: Contract Audit & Fixes](#6-phase-3-contract-audit--fixes)
7. [Phase 4: Frontend Development](#7-phase-4-frontend-development)
8. [Phase 5: Frontend QA & Fixes](#8-phase-5-frontend-qa--fixes)
9. [Phase 6: Full Integration Audit](#9-phase-6-full-integration-audit)
10. [Phase 7: Deploy Contracts to Live Chain](#10-phase-7-deploy-contracts-to-live-chain)
11. [Phase 8: Deploy Frontend to IPFS](#11-phase-8-deploy-frontend-to-ipfs)
12. [Phase 9: Live User Journey Walkthrough](#12-phase-9-live-user-journey-walkthrough)
13. [Phase 10: README & Delivery](#13-phase-10-readme--delivery)
14. [Appendix A: Contract Writing Rules](#appendix-a-contract-writing-rules)
15. [Appendix B: Frontend Writing Rules](#appendix-b-frontend-writing-rules)
16. [Appendix C: SE2 Footguns](#appendix-c-se2-footguns)
17. [Appendix D: Real Build Failures](#appendix-d-real-build-failures)
18. [Appendix E: Verification Commands](#appendix-e-verification-commands)
19. [Appendix F: Verified Contract Addresses](#appendix-f-verified-contract-addresses)
20. [Appendix G: Quick Reference Tables](#appendix-g-quick-reference-tables)
21. [Appendix H: Model & Cost Routing](#appendix-h-model--cost-routing)

---

## 1. Philosophy

Three principles govern this pipeline:

1. **Build the thing, then prove the thing works.** Every stage that produces code has a corresponding verification gate that checks the code does what the spec requires — not just that it compiles. Compilation is necessary but not sufficient.

2. **Never optimize for pipeline progress.** If Phase 4 reveals the architecture is wrong, go back to Phase 0. Moving forward with a broken foundation creates exponentially more work later. A build that compiles, passes tests, and deploys is worthless if it doesn't implement what the spec says.

3. **Never trust, always verify.** Never trust "all done" from a subagent. Never trust a claim that branding was removed. Never trust that addresses are correct. Run the verification commands. Read the files. Check the explorer. If you can't verify it, it didn't happen.

**The hyperstructure test:** Could this run forever with no team behind it? If yes, you built a protocol. If no, you built a service. Know which one you're building.

**The walkaway test:** If the owner disappears, can users still withdraw their funds? The answer must be yes.

---

## 2. Pipeline Overview

```
PHASE 0: ARCHITECTURE & PLANNING
  Spec Ingestion → Onchain Litmus → Discovery → Architecture Plan → Spec Verification Gate
    │
PHASE 1: SCAFFOLD & DISCOVERY  
  Scaffold SE2 → Discover On-Chain ABIs → Discover Package Exports → Register External Contracts
    │
PHASE 2: SMART CONTRACTS
  Write Contracts → Write Deploy Script → Compile ─┬─ Write Tests → Run Tests (terminal branch)
                                                     │
PHASE 3: CONTRACT AUDIT                              │
  Audit → File Issues → Fix Issues → Recompile       │
    │                                                 │
PHASE 4: FRONTEND DEVELOPMENT                        │
  Branding Cleanup → Write Components → Write Pages → Build Frontend
    │
PHASE 5: FRONTEND QA
  QA Audit → File Issues → Fix Issues → Rebuild
    │
PHASE 6: FULL INTEGRATION AUDIT
  Contracts + Frontend Together → Final Safety Check
    │
PHASE 7: DEPLOY CONTRACTS ─────────────────────────────┐
  Deploy → Verify on Explorer → Test with Real Wallet   │
    │                                                    │
PHASE 8: DEPLOY FRONTEND                                │
  Rebuild with Live Addresses → Upload to IPFS → Verify │
    │
PHASE 9: LIVE USER JOURNEY WALKTHROUGH
  Walk Through Every Flow → Fix → Redeploy if Needed
    │
PHASE 10: README & DELIVERY
  Write README → Complete Job → Deliver to Client
```

**Regression rules:**
- Bug in production frontend → return to Phase 7, fix locally, redeploy frontend
- Bug in contract logic → return to Phase 2, fix, retest, redeploy everything
- Never patch production bugs without proper phase regression

**Parallel branch rule:** Tests are verification, not prerequisites for deploy. The compile gate blocks both branches, but tests don't block deploy or frontend.

```
compile ─┬─> tests (terminal — blocks nothing)
         ├─> frontend → build
         └─> deploy → verify → rebuild frontend with live addresses → IPFS
```

---

## 3. Phase 0: Architecture & Planning

Before touching code, answer every question in this phase. Bad architecture shipped fast is worse than no architecture shipped slow.

### 0.1 — Read the Spec

Read everything about the job before writing a single line of code.

1. Read the job description (on-chain `description` field or from the API)
2. Read ALL messages via `GET /api/job/{id}/messages` — clients add requirements, scope changes, and preferences via chat AFTER posting the job. The on-chain description is the baseline; the chat may override it entirely.
3. Extract and document:
   - **What the app does** (one sentence)
   - **What contracts are needed** (names, functions, interactions)
   - **What external protocols are involved** (Uniswap, Lido, Chainlink, etc.)
   - **What the yield/reward/value mechanism is** (how does money flow?)
   - **What the frontend should look like and do**
   - **Who the client is** (`job.client` address — every privileged role in every contract you deploy MUST be set to this address)
   - **What chain** (default: Base, chain ID 8453)

**Lesson learned (Job #39):** Client posted a "Windows 95 aesthetic" requirement via chat. The on-chain description said nothing about it. Skipping messages would have shipped a completely wrong frontend.

### 0.2 — Onchain Litmus Test

Ask: "Does this actually need a blockchain?" Put it onchain ONLY if it requires trustless ownership, trustless exchange, composability with other protocols, censorship resistance, or permanent commitments.

Everything else stays offchain: profiles, search, images, frequently-changing logic, analytics, leaderboards.

**Most MVPs need 0-2 smart contracts.** A token needs one. An NFT collection needs one. A marketplace using existing DEX liquidity might need zero. Three contracts is the upper bound for an initial release.

### 0.3 — State Transition Audit

For every function in your planned contracts, document:

| Function | Who calls it? | Why would they? | What if nobody calls it? | Gas incentive needed? |
|----------|--------------|-----------------|--------------------------|----------------------|

Smart contracts cannot execute themselves. There is no cron job, no scheduler, no background process. Every function needs a caller who pays gas. If your answer to "who calls it?" is "the team" or "an admin" — redesign. Make it callable by anyone with aligned incentives.

### 0.4 — Chain Selection

Pick ONE chain. Prove product-market fit. Expand later with CREATE2 for consistent addresses.

| Need | Chain | Why |
|------|-------|-----|
| Consumer/social, AI agents, cheapest gas | Base | Coinbase integration, ERC-8004, x402, Smart Wallet |
| Deepest DeFi liquidity, yield strategies | Arbitrum | GMX, Pendle, Camelot, widest protocol coverage |
| MEV protection | Unichain | TEE block building, time-ordered txs |
| Native account abstraction | zkSync Era | Built-in AA, no bundlers |
| Mobile/real-world payments | Celo | MiniPay, sub-cent fees |
| High-value DeFi, governance, identity | Mainnet | Canonical security, composability |

Real costs (early 2026): ETH transfer on mainnet ~$0.004, on Base ~$0.0003. Uniswap swap on mainnet ~$0.036, on Base ~$0.002.

### 0.5 — Identify dApp Archetype

| Archetype | Contracts | Notes |
|-----------|-----------|-------|
| Token launch | 1-2 | Token + optional vesting |
| NFT collection | 1 | ERC-721 + IPFS metadata |
| Marketplace | 0-2 | Often zero — integrate existing DEX |
| Lending/Vault | 0-1 | ERC-4626 vault or wrap existing protocol |
| DAO/Governance | 1-3 | Governor + token + timelock |
| AI agent service | 0-1 | Optional ERC-8004 registration |
| Data dashboard / tracker | 0 | Frontend-only, reads from APIs and contracts |

### 0.6 — Identify Required Skills

Before writing any code, determine which ethskills modules apply:

| Building with... | Read |
|-----------------|------|
| Any smart contract | `security`, `testing` |
| OpenZeppelin contracts | `openzeppelin` |
| ERC-721 / NFTs | `erc-721` |
| External contract interaction | `standards` |
| Token approvals / DeFi | `frontend-ux` |
| Indexing / historical data | `indexing` (The Graph or Ponder) |
| Sign-in with Ethereum | `siwe` |
| HTTP payments | `x402` |
| Agent identity | `standards` (ERC-8004 section) |
| Database / off-chain storage | `drizzle-neon` |
| Batch transactions | `eip-5792` |
| Subgraph indexing | `subgraph` |

All skills: `https://ethskills.com/<skill>/SKILL.md`

### 0.7 — Identify Key Addresses

Before writing any code, collect every address you'll need:
- Client wallet (owner of all deployed contracts)
- Token contracts (addresses, decimals, standard behaviors)
- Protocol contracts (routers, pools, oracles, factories)
- Chain-specific addresses (WETH, bridge contracts)

**Never hallucinate contract addresses.** For every address:

```bash
# Verify contract exists
cast code <address> --rpc-url $ALCHEMY_RPC_URL
# Returns 0x if contract doesn't exist on that chain

# Verify token identity
cast call <addr> "symbol()(string)" --rpc-url $ALCHEMY_RPC_URL

# Verify pool exists  
cast call <factory> "getPool(address,address,uint24)(address)" <tokenA> <tokenB> <fee> --rpc-url $ALCHEMY_RPC_URL

# Verify router
cast call <addr> "factory()(address)" --rpc-url $ALCHEMY_RPC_URL
```

Cross-reference all addresses against the [Verified Contract Addresses](#appendix-f-verified-contract-addresses) table.

### 0.8 — Identify External Protocol Behaviors

For each external protocol:
- Is the token standard ERC20? Any non-standard behaviors (rebasing, fee-on-transfer, blocklist)?
- What pool fee tiers exist? (Uniswap V3: 100, 500, 3000, 10000)
- Is the protocol available on the target chain? (e.g., Lido's wstETH on Base is bridged, not native — no `wrap()`, `unwrap()`, `stETH()` functions)
- What are the correct interface function signatures?

### 0.9 — Secret Management Plan

Decide NOW where every secret lives. Bots scan GitHub in real-time and exploit leaked keys within seconds.

Rules:
- All secrets go in `.env.local` (gitignored)
- Never hardcode keys in config files
- Never use public RPCs for production (always Alchemy with API key)
- Verify `.gitignore` includes: `.env`, `.env.*`, `*.key`, `*.pem`, `broadcast/`, `cache/`
- Pre-commit check: `git diff --cached --name-only | grep -iE '\.env|key|secret|private'`
- Source audit: `grep -r "0x[a-fA-F0-9]{64}" .` should return zero results

### 0.10 — Write the Architecture Plan

Create `PLAN.md` in the repo root covering:

- **One-sentence summary** of what the app does
- **Smart contracts** — each contract, its storage, its functions, events, errors, access control. Be specific about types.
- **External integrations** — every external protocol, contract address on target chain, which functions we call, interface requirements, risk notes
- **Frontend** — pages, components, user flows, what data is read from which contract
- **Security considerations** — attack vectors specific to this design, mitigations, trust assumptions
- **Deployment plan** — order of deployment (dependencies matter), constructor arguments for each contract, post-deploy configuration steps
- **Ownership** — ALL privileged roles → `job.client` address

### 0.11 — Write the User Journey

Create `USERJOURNEY.md` covering:

**Happy path (step by step):**
1. User opens the app URL
2. User sees [what?] — describe the landing state with no wallet connected
3. User clicks Connect Wallet — describe what happens
4. User is on wrong network — describe the Switch Network prompt
5. User performs the primary action — describe each click, each transaction, each confirmation
6. User sees the result — describe the success state

**Edge cases (must cover all):**
- No wallet installed
- Wrong network connected
- Insufficient balance (for gas AND for the token/action)
- Transaction rejected by user
- Transaction reverted on-chain
- Slow transaction (pending state)
- Multiple rapid clicks (double-submit prevention)
- Mobile wallet via WalletConnect
- Zero balance, max amount, insufficient allowance

### 0.12 — Spec Verification Gate

**STOP. Before writing any code, verify the plan implements the spec.**

Go through PLAN.md line by line against the job requirements:
- [ ] **Underlying asset is correct.** If the spec says "ETH staking yield," the vault MUST use wstETH/stETH as underlying (they appreciate), NOT WETH/ETH (which don't)
- [ ] **Yield mechanism is real.** Can you explain exactly how yield appears in the contract's accounting? If `availableYield()` would return 0 under normal conditions, the mechanism is broken
- [ ] **Swap path exists on-chain.** For every swap, verify the pool exists with cast call. Zero address = pool doesn't exist
- [ ] **External addresses are real.** Every address has been verified with a cast call
- [ ] **Owner is CLIENT.** Constructor must pass client address, not `msg.sender`
- [ ] **Token decimals are correct.** USDC = 6, WETH = 18. Wrong decimals = all math breaks
- [ ] **Every state transition has an identified caller with an incentive**

If ANY requirement fails: fix the plan. Do NOT proceed to code.

**Deliverable:** PLAN.md, USERJOURNEY.md, verified spec requirements.

---

## 4. Phase 1: Scaffold & Discovery

### 1.1 — Create the Project

```bash
npx -y create-eth@latest -s foundry leftclaw-service-job-{JOBID}
cd leftclaw-service-job-{JOBID}
```

This gives you:
- `packages/foundry/` — Solidity contracts, tests, deploy scripts
- `packages/nextjs/` — Next.js frontend with RainbowKit, wagmi, viem, DaisyUI

**Do NOT manually create Foundry or Next.js projects.** Always use `create-eth`.

### 1.2 — Read AGENTS.md

After scaffolding, read `AGENTS.md` in the project root. It contains the authoritative reference for hooks, components, conventions, and code style. Do not proceed until you've read it.

### 1.3 — Discover On-Chain Interfaces

**This is the stage that prevents hallucinated interfaces.** Before any code is generated, verify what actually exists on-chain.

For every external contract address the dApp will interact with:

```bash
cast interface <address> --chain base
```

This returns the actual function signatures the contract exposes. Store the result as your `discovery.json`.

**What this catches:** Contracts that exist on mainnet with rich interfaces (Lido wstETH with `wrap()`, `unwrap()`, `stETH()`) but are bridged on L2s as simple ERC20s. Without discovery, the LLM writes code against the mainnet interface and the deploy reverts.

### 1.4 — Discover Package Exports

Read the actual TypeScript exports from installed packages:

```bash
# SE2 hooks
cat packages/nextjs/hooks/scaffold-eth/index.ts

# UI components  
cat packages/nextjs/components/scaffold-eth/index.tsx

# Scaffold config patterns
cat packages/nextjs/scaffold.config.ts

# Deploy script pattern
cat packages/foundry/script/DeployHelpers.s.sol
cat packages/foundry/script/Deploy.s.sol

# Next.js config
cat packages/nextjs/next.config.ts

# CSS setup
head -5 packages/nextjs/styles/globals.css
```

**What this catches:** Import path errors. The LLM writes `import { Address } from "~~/components/scaffold-eth/Address"` but the actual export is from `@scaffold-ui/components`. Without discovery, the frontend build fails repeatedly.

Record discovered exports:
- **Hooks:** `useScaffoldReadContract`, `useScaffoldWriteContract`, `useScaffoldEventHistory`, `useDeployedContractInfo`, `useTargetNetwork`, etc.
- **UI Components:** `Address`, `AddressInput`, `Balance`, `EtherInput` from `@scaffold-ui/components`
- **Other Components:** `RainbowKitCustomConnectButton` from `~~/components/scaffold-eth`
- **Import Paths:** `~~` maps to `packages/nextjs/` root

### 1.5 — Register External Contracts

If your app reads from existing deployed contracts, register them NOW in `packages/nextjs/contracts/externalContracts.ts`:

```typescript
import { GenericContractsDeclaration } from "~~/utils/scaffold-eth/contract";

const externalContracts = {
  8453: {  // Base chain ID — must be number, not string
    USDC: {
      address: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      abi: [/* minimal ABI — only functions the frontend calls */],
    },
  },
} as const;

export default externalContracts satisfies GenericContractsDeclaration;
```

**Rule:** Only include ABI entries the frontend actually calls. Don't dump full ABIs.

**Pre-deploy trick (from yet-another-builder):** Include project contracts in `externalContracts.ts` with `0x0000000000000000000000000000000000000000` placeholder addresses. This lets the frontend type-check and compile before deployment. After deploy, `deployedContracts.ts` takes precedence.

**Never manually edit `deployedContracts.ts`.** It is auto-generated by `yarn deploy`.

### 1.6 — Initialize Git + GitHub

```bash
git init
git config user.name "clawdbotatg"
git config user.email "clawdbotatg@users.noreply.github.com"
git add -A
git commit -m "feat: scaffold SE2 Foundry for <project-name>"
gh repo create clawdbotatg/leftclaw-service-job-{JOBID} --public --source=. --remote=origin --push
```

### 1.7 — Verification Gate

- [ ] `packages/foundry/` exists
- [ ] `packages/nextjs/` exists
- [ ] `forge build` compiles the default YourContract.sol
- [ ] Repo visible on GitHub
- [ ] Discovery completed: on-chain ABIs verified, package exports recorded
- [ ] External contracts registered in `externalContracts.ts`
- [ ] `yarn start` runs without errors

**Stop condition:** Do NOT write contracts or frontend. Stop at scaffold.

---

## 5. Phase 2: Smart Contract Development

### 2.1 — Delete Scaffold Defaults

```bash
rm packages/foundry/contracts/YourContract.sol
rm packages/foundry/script/DeployYourContract.s.sol
rm packages/foundry/test/YourContract.t.sol
```

### 2.2 — Write Contracts

Location: `packages/foundry/contracts/`

See [Appendix A: Contract Writing Rules](#appendix-a-contract-writing-rules) for all mandatory patterns.

**Key rules:**
- All privileged roles (owner, admin, treasury) MUST be set to `job.client`
- Use OpenZeppelin contracts where applicable
- Only call functions that exist in your discovery results
- Emit events for every state change
- One contract per file for complex builds (prevents token limit truncation)

### 2.3 — Write Interfaces

Create `packages/foundry/contracts/interfaces/` for external protocol interfaces. Only include the functions you actually call. Base your interfaces on what discovery found — not what you think should exist.

### 2.4 — Write Deploy Script

Location: `packages/foundry/script/`

**Deployer-first ownership pattern (critical for multi-contract systems):**

```solidity
// 1. Deploy with deployer as initial owner
// 2. Configure cross-references (setHarvester, setRewardsDistributor, etc.)
// 3. Transfer ownership to client via transferOwnership()
// Client must call acceptOwnership() on each contract (Ownable2Step)
```

You can't configure cross-references after transferring ownership because only the owner can call admin functions. Deploy → configure → transfer.

**Deploy script requirements:**
- Always inherit `ScaffoldETHDeploy`, use `ScaffoldEthDeployerRunner` modifier on `run()`
- Push to `deployments` array: `deployments.push(Deployment("MyContract", address(myContract)));`
- Use `vm.envOr` for addresses — never `address(0)` as default
- Zero-address checks in every constructor
- RPC endpoints: always use Alchemy, never public RPCs

### 2.5 — Compile

```bash
cd packages/foundry && forge build
```

**Verify-fix loop:** If compilation fails, parse error location, send broken file + error to fixer, get fixed file. Max 3 retries.

### 2.6 — Write Tests

Location: `packages/foundry/test/`

**Test priority:**
1. **Unit tests** — edge cases, failure modes, access control. NOT getters.
2. **Fuzz tests** — any function with math. Minimum 1000 runs. Use `bound()` not `vm.assume()`.
3. **Fork tests** — any interaction with external protocols.
4. **Invariant tests** — stateful protocols. Properties that must always hold.

**What to test:**
- Constructor state: all immutables set correctly
- Happy path: full user journey (deposit → stake → harvest → claim → withdraw)
- Access control: unauthorized callers revert
- Edge cases: zero amounts, max amounts, reentrancy attempts
- Full lifecycle: deploy → interact → verify state

**Critical fix loop rule:** If a test assertion fails, the test expectation is probably wrong — fix the test, not the contract. Prevent the LLM from breaking working contract logic to make a bad test pass.

### 2.7 — Run Tests

```bash
forge test -vvv
```

### 2.8 — Contract Verification Gate

**Go through your spec requirements again.** For each requirement:
1. Find the specific contract code that implements it
2. Trace the execution path
3. Verify the math

Functional verification checklist:
- [ ] **Underlying asset:** `vault.asset()` returns the correct token address
- [ ] **Yield calculation:** Walk through with example numbers. Does the function return the right percentage?
- [ ] **Swap path:** The encoded path matches real Uniswap pools
- [ ] **Owner is CLIENT:** Every `Ownable(_owner)` constructor passes the client address
- [ ] **No stub functions:** Every function has a real implementation. No `// TODO` or empty bodies.
- [ ] **No hardcoded test values:** All addresses, fees, and parameters match mainnet values
- [ ] **Tests pass:** `forge test` exits 0
- [ ] **Compilation clean:** `forge build` exits 0 with no warnings

**Deliverable:** Contract source files, deploy scripts, tests — all passing.

**Stop condition:** Do NOT deploy. Do NOT write frontend code.

---

## 6. Phase 3: Contract Audit & Fixes

### 3.1 — Standard Audit

Fetch and follow: **https://ethskills.com/audit/SKILL.md**

This deploys parallel specialist agents across 19 security domains with 500+ checklist items:

- General analysis and architecture
- Math precision and overflow
- Token standards (ERC-20, ERC-721, ERC-4626)
- DeFi protocols (AMM, lending, staking)
- Cross-chain bridges
- Proxy/upgrade patterns
- Signature verification
- Governance
- Oracle manipulation
- Assembly/low-level code
- Flash loan vectors
- DoS vectors

**Critical checks:**
1. Reentrancy — guards on all external-calling functions
2. Access control — no leftover deployer privileges, Ownable2Step correct
3. Integer safety — overflow/underflow, unsafe casting
4. External call safety — return values checked, CEI pattern followed
5. Token handling — SafeERC20, no raw `transfer()`
6. Slippage protection — all swaps have minimum output
7. Oracle safety — TWAP checks, manipulation resistance
8. First depositor attacks — virtual shares for ERC4626

### 3.2 — Deep Audit (for complex contracts)

**SKIP if simple:** basic storage, simple getters/setters, < 100 lines, no token swaps, no reentrancy vectors.

**DO if complex:** token swaps, multi-contract interactions, financial logic, upgradeable proxies, > 200 lines.

Deep audit focuses on:
- Cross-contract interaction risks
- Pool observation history requirements (wrap `pool.observe()` in try/catch)
- Zero-staker edge cases in rewards contracts
- Principal tracking drift over many operations
- Rounding errors at scale
- Economic attack vectors (sandwich, flash loan manipulation)
- TWAP protection parameter reasonableness
- Slippage bounds

### 3.3 — File Issues

```bash
gh label create "job-{ID}" --repo clawdbotatg/leftclaw-service-job-{ID} --color "0e8a16" --force
gh label create "contract-audit" --repo clawdbotatg/leftclaw-service-job-{ID} --color "d93f0b" --force

# For each Medium+ finding:
gh issue create --repo clawdbotatg/leftclaw-service-job-{ID} \
  --title "[SEVERITY] Finding title" \
  --body "**Location:** file:function\n**Description:** ...\n**Recommendation:** ..." \
  --label "job-{ID},contract-audit"
```

### 3.4 — Fix All Findings

1. **Critical** — must fix, no exceptions
2. **High** — must fix
3. **Medium** — fix or document as accepted with clear reasoning
4. **Low** — fix if trivial, otherwise document
5. **Info** — no action required

After each fix:
```bash
forge build && forge test
```

Close each issue with commit reference.

### 3.5 — Verification Gate

- [ ] `forge build` exits 0
- [ ] `forge test` passes
- [ ] Zero Critical/High findings remain open
- [ ] Every Medium finding is either fixed or documented

**Stop condition:** Do NOT deploy.

---

## 7. Phase 4: Frontend Development

### 4.1 — Start the Fork

**Always fork the target network.** Never use `yarn chain` (empty local chain). Fork mode gives you real protocol state.

```bash
yarn fork --network base
```

Then enable interval mining:
```bash
cast rpc anvil_setIntervalMining 1
```

**Critical:** During local development, the frontend's target network must be `chains.foundry` (chain ID 31337), regardless of which network you're forking. Only switch to the real chain ID when deploying to the live network.

Deploy to the fork:
```bash
yarn deploy
```

Start the frontend:
```bash
yarn start
```

### 4.2 — SE2 Branding Cleanup (MANDATORY)

This is not optional. AI agents treat the scaffold as sacred — don't.

- [ ] **Footer.tsx** — Remove "Fork me", "Built with heart at BuidlGuidl", support links, `nativeCurrencyPrice` badge
- [ ] **Header.tsx** — Replace SE2 logo/text with project name, remove "Debug Contracts" nav link
- [ ] **getMetadata.ts** — Change `titleTemplate`, default title, description. Use `process.env.NEXT_PUBLIC_PRODUCTION_URL` for OG image base
- [ ] **README.md** — Replace entirely with project content
- [ ] **Favicon** — Replace `packages/nextjs/public/favicon.ico`
- [ ] **manifest.json** — Change "Scaffold-ETH 2 DApp" to your app name
- [ ] **wagmiConnectors.tsx** — Change `appName: "scaffold-eth-2"` to your app name, add `phantomWallet`
- [ ] **Block explorer** — Delete `packages/nextjs/app/blockexplorer/` (crashes static export)
- [ ] **Debug page** — Delete `packages/nextjs/app/debug/` (uses `force-dynamic`, incompatible with `output: "export"`)

### 4.3 — Configuration

**scaffold.config.ts:**
```typescript
targetNetworks: [chains.foundry],  // chains.base for production
pollingInterval: 3000,  // NOT 30000
```

**globals.css — fix pill-shaped inputs:** Change `--radius-field: 9999rem` to `--radius-field: 0.5rem` in BOTH theme blocks (light and dark).

### 4.4 — Write Components

See [Appendix B: Frontend Writing Rules](#appendix-b-frontend-writing-rules) for all mandatory patterns.

**Key rules:**
- Every page component must be `"use client"`
- Use SE2 hooks — never raw wagmi
- Use SE2 components — never raw HTML for addresses/balances
- DaisyUI semantic classes — never hardcoded dark backgrounds
- Four-state button flow: Connect → Network → Approve → Action (one at a time)
- Dual-state approve protection: `approvalSubmitting` + `approveCooldown`

### 4.5 — Build Frontend

```bash
yarn next:build
```

### 4.6 — Verify in Browser

Start the dev server and actually use every feature:

```bash
yarn start   # http://localhost:3000
```

- [ ] Page loads without errors
- [ ] Wallet connects
- [ ] All contract interactions work end-to-end
- [ ] No console errors
- [ ] Four-state button flow works
- [ ] Loading states on all transaction buttons
- [ ] Error messages are human-readable
- [ ] Mobile viewport looks acceptable

### 4.7 — Verification Gate

- [ ] `yarn next:build` exits 0
- [ ] App loads at localhost:3000
- [ ] Every feature works in the browser
- [ ] No stubs or placeholder data

**Stop condition:** Do NOT deploy to IPFS.

---

## 8. Phase 5: Frontend QA & Fixes

### 5.1 — QA Audit

Fetch and follow: **https://ethskills.com/qa/SKILL.md**

Run every check. Report PASS/FAIL. Do NOT fix anything yet.

#### Ship-Blocking Checks (ALL must PASS)

- [ ] Wallet connection shows a BUTTON, not text ("Please connect your wallet" = FAIL)
- [ ] Wrong network shows a Switch button
- [ ] One button at a time (Connect → Network → Approve → Action)
- [ ] Approve button locked with BOTH `approvalSubmitting` AND `approveCooldown` on `disabled` prop
- [ ] `approvalSubmitting` clears in `finally {}` block (handles rejection)
- [ ] SE2 footer branding removed (BuidlGuidl, "Fork me", support links)
- [ ] Tab title is the app name, NOT "Scaffold-ETH 2"
- [ ] README describes THIS project, not the SE2 template
- [ ] Favicon is not the SE2 default
- [ ] Contracts verified on block explorer (after deploy)
- [ ] No raw wagmi hooks outside scaffold-eth internals
- [ ] No zero-address (`0x000...0`) placeholders in components
- [ ] No hardcoded deployed contract addresses as string literals — use `useDeployedContractInfo`
- [ ] BGIPFS deployed (production frontend is on IPFS, not Vercel)

#### Should-Fix Checks

- [ ] Contract address displayed with `<Address />`
- [ ] Every address input uses `<AddressInput />` — no raw `<input type="text">`
- [ ] USD values next to all token/ETH amounts
- [ ] OG image is absolute production URL (not relative path or localhost)
- [ ] `pollingInterval` is 3000 (not 30000 default)
- [ ] RPC overrides use env var AND env var is confirmed set on hosting platform
- [ ] `--radius-field` changed from `9999rem` to `0.5rem`
- [ ] Button loaders use inline `<span className="loading loading-spinner loading-sm" />`, NOT `className="... loading"` on the button
- [ ] Phantom wallet in RainbowKit wallet list
- [ ] No hardcoded dark backgrounds — uses `bg-base-200 text-base-content`
- [ ] Every contract error mapped to a human-readable message via `getParsedError`
- [ ] `appName` changed in wagmiConnectors.tsx
- [ ] manifest.json updated
- [ ] Block explorer disabled/removed

#### Mobile Checks

- [ ] All transaction buttons deep link to wallet app (fire TX first, then `setTimeout(openWallet, 2000)`)
- [ ] Wallet detection checks WalletConnect session data, not just `connector.id`
- [ ] No deep link when `window.ethereum` exists (already in wallet's in-app browser)

### 5.2 — File Issues

```bash
gh label create "frontend-audit" --repo clawdbotatg/leftclaw-service-job-{ID} --color "f9d0c4" --force
# For each FAIL:
gh issue create --repo clawdbotatg/leftclaw-service-job-{ID} \
  --title "[FAIL] <item>" --body "..." --label "job-{ID},frontend-audit"
```

### 5.3 — Fix All Issues

Fix each one. Close each issue with a commit reference.

### 5.4 — Rebuild and Re-verify

```bash
yarn next:build
```

- [ ] ALL ship-blocker items now PASS
- [ ] ALL should-fix items now PASS or documented
- [ ] Build still passes after fixes

**Stop condition:** Do NOT deploy to IPFS.

---

## 9. Phase 6: Full Integration Audit

One final pass on everything — contracts AND frontend together.

### 6.1 — Safety Check

- [ ] Users can always withdraw (no lockups, no admin freeze without user consent)
- [ ] Owner cannot steal user deposits
- [ ] Reentrancy guards on all entry points
- [ ] SafeERC20 used throughout
- [ ] TWAP/slippage protection on all swaps
- [ ] All privileged roles set to client address

### 6.2 — Frontend-Contract Integration Check

- [ ] Frontend passes correct args to every contract function (check ABI match)
- [ ] Slippage parameters are non-zero
- [ ] All contract addresses in frontend match deployed addresses
- [ ] External contracts registered correctly

### 6.3 — Secrets Check

- [ ] No secrets in the repo (private keys, API keys, mnemonics)
- [ ] `.gitignore` includes `.env`, `*.key`, `broadcast/`, `cache/`
- [ ] `grep -r "0x[a-fA-F0-9]{64}" packages/` returns zero results
- [ ] `grep -rn "g.alchemy.com/v2/[A-Za-z0-9]" packages/` returns zero results

### 6.4 — Build Check

```bash
cd packages/foundry && forge build      # contracts still compile
forge test                              # tests still pass
yarn next:build                         # frontend still builds
```

File issues for any findings. Label: `job-{ID}`, `full-audit`. Fix all.

---

## 10. Phase 7: Deploy Contracts to Live Chain

### 7.1 — Configure RPC

In `packages/foundry/foundry.toml`:
```toml
[rpc_endpoints]
base = "${ALCHEMY_RPC_URL}"
```

**NEVER use `https://mainnet.base.org` or any public RPC.** Always Alchemy.

### 7.2 — Fund the Deployer

```bash
yarn generate    # Generate deployer account (if needed)
yarn account     # View deployer address and balance
```

Fund the deployer with ETH on the target chain.

### 7.3 — Deploy

```bash
yarn deploy --network base
```

Verify `deployedContracts.ts` is updated with the live addresses.

### 7.4 — Verify on Block Explorer

```bash
yarn verify --network base
```

**Every deployed contract must show verified source code with a green checkmark.** Unverified contracts are a trust red flag.

Check manually: open each contract address on Basescan, look for the "Contract" tab with a checkmark.

If `yarn verify` fails, verify individually:
```bash
forge verify-contract <ADDRESS> <CONTRACT_NAME> --chain-id 8453 --watch
```

### 7.5 — Verify On-Chain State

For every deployed contract:
```bash
# Check ownership
cast call <address> "owner()(address)" --rpc-url $ALCHEMY_RPC_URL
cast call <address> "pendingOwner()(address)" --rpc-url $ALCHEMY_RPC_URL

# Check cross-references
cast call <vault> "harvester()(address)" --rpc-url $ALCHEMY_RPC_URL
```

### 7.6 — Test with Real Wallet

- Switch `scaffold.config.ts` to `targetNetworks: [chains.base]`
- Run `yarn start`
- Connect a real wallet
- Test with small amounts ($1-10)
- Verify all flows work end-to-end
- File GitHub issues for any problems

### 7.7 — Verification Gate

- [ ] All contracts deployed (addresses recorded)
- [ ] All contracts verified on explorer (green checkmark)
- [ ] `deployedContracts.ts` updated with real addresses
- [ ] Cross-references configured (if applicable)
- [ ] Ownership transfer initiated to client address
- [ ] All flows work with real wallet on live chain
- [ ] No constructor args were zero addresses

---

## 11. Phase 8: Deploy Frontend to IPFS

### 8.1 — Pre-Deploy Checklist

- [ ] `targetNetworks: [chains.base]` in scaffold.config.ts (not foundry/localhost)
- [ ] `pollingInterval: 3000`
- [ ] `burnerWalletMode: "localNetworksOnly"` (no burner wallet in production)
- [ ] OG image is 1200x630px, `NEXT_PUBLIC_PRODUCTION_URL` set
- [ ] RPC env vars set on hosting platform
- [ ] SE2 branding fully removed
- [ ] Block explorer and debug pages deleted

### 8.2 — Node 25+ localStorage Polyfill

Create `packages/nextjs/polyfill-localstorage.cjs`:
```javascript
if (typeof globalThis.localStorage === "undefined") {
  const store = {};
  globalThis.localStorage = {
    getItem: (k) => store[k] ?? null,
    setItem: (k, v) => { store[k] = String(v); },
    removeItem: (k) => { delete store[k]; },
    clear: () => { Object.keys(store).forEach(k => delete store[k]); },
    get length() { return Object.keys(store).length; },
    key: (i) => Object.keys(store)[i] ?? null,
  };
}
```

**Must be in `packages/nextjs/`** — not the project root. The build command runs in the nextjs package context.

### 8.3 — Build

Clean build:
```bash
rm -rf packages/nextjs/.next packages/nextjs/out
```

Build:
```bash
NEXT_PUBLIC_PRODUCTION_URL="https://yourapp.yourname.eth.link" \
  NEXT_PUBLIC_IPFS_BUILD=true \
  NEXT_PUBLIC_IGNORE_BUILD_ERROR=true \
  yarn build
```

### 8.4 — IPFS Routing Requirements

Three conditions must be met:
1. `output: "export"` in `next.config.ts`
2. `trailingSlash: true` — IPFS gateways resolve directories to `index.html` but not bare filenames
3. No pages that crash during prerender — crashed pages get silently skipped, producing 404s

### 8.5 — Upload to IPFS

```bash
yarn ipfs
```

**OR with bgipfs directly:**

```bash
npx bgipfs init --token $BGIPFS_TOKEN --endpoint https://upload.bgipfs.com
npx bgipfs upload packages/nextjs/out
```

**The `--endpoint` flag is MANDATORY.** Without it, a bad config is created pointing to localhost.

### 8.6 — Validate Upload

`yarn ipfs` exit code is NOT reliable — the script uses `|| echo` which swallows errors.

**The only valid signal is an IPFS CID in the output.** Look for:
- `Qm` followed by 44 alphanumeric characters, OR
- `bafy` followed by 50+ lowercase alphanumeric characters

If no CID appears, the upload failed regardless of exit code.

### 8.7 — OG Metadata Fix (Chicken-and-Egg)

After first deploy, you know the CID. Rebuild with the real URL:
```bash
export NEXT_PUBLIC_PRODUCTION_URL="https://<CID>.ipfs.community.bgipfs.com"
yarn build
npx bgipfs upload packages/nextjs/out
```

The CID will change. The OG image URL in the new build points to the old CID, which still works on IPFS (content-addressed = permanent).

### 8.8 — Verify the Deployment

```bash
LIVE_URL="https://<CID>.ipfs.community.bgipfs.com/"

# HTTP check
curl -s -o /dev/null -w "%{http_code}" "$LIVE_URL"   # must return 200

# HTML renders
curl -s "$LIVE_URL" | head -20

# Title correct
curl -s "$LIVE_URL" | grep -i "<title>"

# OG metadata not localhost
curl -s "$LIVE_URL" | grep -i "og:image"
```

- Open the IPFS gateway URL in a browser
- Test wallet connection, contract interactions, all flows
- Verify OG image renders when sharing the URL
- Test on mobile

### 8.9 — ENS Subdomain (if applicable)

Two mainnet transactions:
1. Create subdomain at `app.ens.domains/yourname.eth` → "Subnames" → "New subname"
2. Set content hash: "Records" → "Other" → paste `ipfs://{CID}` in Content Hash field

Use `.eth.link` (not `.eth.limo`) for better mobile support.

---

## 12. Phase 9: Live User Journey Walkthrough

Open the live app (IPFS URL or ENS subdomain) in a browser with a real wallet. Follow `USERJOURNEY.md` step by step as a real user.

- Actually click every button
- Actually connect your wallet
- Actually submit transactions
- Actually verify the results

If ANYTHING is broken or doesn't match the user journey doc:
1. Go back to Phase 7 (fix locally with live contracts)
2. File issues, fix them, redeploy frontend
3. Re-run this walkthrough until the entire journey works perfectly

**Phase regression rules:**
- Bug in production frontend → return to Phase 7, fix locally, redeploy
- Bug in contract logic → return to Phase 2, fix, retest, redeploy everything
- Never patch production bugs without proper phase regression

---

## 13. Phase 10: README & Delivery

### 10.1 — Write README

**Must include:**
- What the app does (2-3 sentences)
- Contract addresses on the target chain with explorer links
- How to run locally (`yarn fork`, `yarn deploy`, `yarn start`)
- Architecture decisions and non-obvious implementation details
- Link to the live app
- Client actions needed (e.g., `acceptOwnership()` on each contract)

**Must NOT include:**
- Explanations of what React, Solidity, or Ethereum are
- SE2 template boilerplate
- Padding or filler content

### 10.2 — Deliver

1. Upload final deliverables to IPFS
2. Call `completeJob(jobId, resultURL)` — resultURL must be the FULL IPFS URL:
   - Format: `https://{CID}.ipfs.community.bgipfs.com/`
   - Do NOT pass just the raw CID
3. Send the live working app URL to the client via `POST /api/job/{id}/messages` with type `bot_message`
4. The job is complete

---

## Appendix A: Contract Writing Rules

### Architecture

- **Use OpenZeppelin.** Check what's installed (`packages/foundry/lib/openzeppelin-contracts/contracts/`) before writing custom implementations.
- **Ownable2Step over Ownable.** Two-step ownership transfer prevents fat-finger mistakes.
- **ReentrancyGuard on all external-facing state-changing functions.**
- **CEI pattern (Checks-Effects-Interactions).** State changes before external calls.
- **Custom errors over require strings.** `error ZeroAddress();` not `require(addr != address(0), "zero")`.
- **SafeERC20 for all token transfers.** `token.safeTransfer()` not `token.transfer()`.
- **Never use `tx.origin`.**
- **Never use infinite approvals** — approve exact amounts or 3-5x.
- **Emit events for every state change.**
- **NatSpec comments on all public/external functions.**
- **`immutable`/`constant` where appropriate.**

### External Protocol Integration

- **Only call functions that exist in discovery.** If the ABI doesn't show `wrap()`, don't call `wrap()`.
- **On L2s, bridged tokens are plain ERC20s.** They don't have the rich interfaces their L1 originals have. Design around this.
- **For token conversions on L2, use Uniswap swaps.** Not Lido wrap/unwrap, not native bridge calls.

### ERC4626 Vaults

- **Virtual shares / dead shares for inflation protection.** Use `_decimalsOffset()` >= 3.
- **Underlying asset must be consistent.** If vault holds wstETH, the ERC4626 asset IS wstETH.
- **Override `_deposit` and `_withdraw` (internal hooks), NOT `deposit` and `withdraw` (public functions).** Public function overrides cause double-nonReentrant.
- **Convenience deposit functions** (depositETH, depositWETH): Use `previewDeposit()` + `_mint()` directly. Do NOT call `super.deposit()`.

### Uniswap V3 Swaps

- Always set `amountOutMinimum` > 0 (slippage protection)
- Always set `deadline` (block.timestamp for on-chain; caller-provided for keeper txs)
- TWAP oracle check BEFORE the swap, not after
- Wrap `pool.observe()` in try/catch (new pools may lack observation history)

### Deploy Scripts

- **Always inherit `ScaffoldETHDeploy`.** Use `ScaffoldEthDeployerRunner` modifier.
- **Deploy contracts inline.** `new MyContract(args)` inside `run()`.
- **Push to `deployments` array.**
- **Wire addresses after all deploys.** Deploy A, deploy B(address(A)), then A.setB(address(B)).
- **Constructor args: use `vm.envOr` for addresses.** Never use `address(0)` as default.
- **Zero-address checks in every constructor.**
- **RPC: always Alchemy.** Never public RPCs.

### Access Control

- Owner = client address
- If a keeper/harvester role is needed, add it as a separate role the owner can set
- Walkaway test: if the owner disappears, can users still withdraw? Must be yes.

---

## Appendix B: Frontend Writing Rules

### Imports

- **SE2 hooks:** `import { useScaffoldReadContract } from "~~/hooks/scaffold-eth";`
- **UI components:** `import { Address, Balance, EtherInput, AddressInput } from "@scaffold-ui/components";`
- **Connect button:** `import { RainbowKitCustomConnectButton } from "~~/components/scaffold-eth";`
- **Deployed contract info:** `import { useDeployedContractInfo } from "~~/hooks/scaffold-eth";`
- **Notifications:** `import { notification } from "~~/utils/scaffold-eth";`
- **Error parsing:** `import { getParsedError } from "~~/utils/scaffold-eth";`
- **Path alias:** `~~` maps to `packages/nextjs/` root. Always use it.

Verify no raw wagmi usage:
```bash
grep -rn "useWriteContract\|useReadContract" packages/nextjs/ | grep -v scaffold-eth | grep -v node_modules
```
Any match = bug.

### Component Patterns

- **Every page component must be `"use client"`.** SE2 uses Next.js App Router.
- **`export default` required** on every page/component.
- **Never use raw wagmi hooks** for contract interaction.
- **Never edit `deployedContracts.ts`** manually.
- **External contracts go in `externalContracts.ts`.**

### Critical Address Rules (THE #1 BUG CLASS)

1. **NEVER** hardcode a deployed contract address as a string literal in any React file
2. **NEVER** pass a deployed contract address as a prop from page.tsx into a child component
3. Components that need a deployed address MUST call `useDeployedContractInfo("ContractName")` internally
4. `deployedContracts.ts` is provided for REFERENCE only — addresses must NOT be copied into components
5. `page.tsx` should compose components, NOT know any contract addresses
6. External contracts (tokens) are accessed via `contractName` in hooks — never hardcoded

### SE2 Hook Patterns

```typescript
// READ from contract
const { data } = useScaffoldReadContract({
  contractName: "MyContract",
  functionName: "myFunction",
  args: [arg1, arg2],
});

// WRITE to contract
const { writeContractAsync, isPending } = useScaffoldWriteContract("MyContract");
await writeContractAsync({
  functionName: "myFunction",
  args: [arg1],
  value: parseEther("0.01"), // for payable
});

// Get deployed contract address
const { data: contractInfo } = useDeployedContractInfo("MyContract");
const contractAddress = contractInfo?.address;
```

### Four-State Button Flow (Mandatory)

Show exactly ONE primary button at a time:

```
1. Not connected  → Connect Wallet button (RainbowKitCustomConnectButton or useConnectModal)
2. Wrong network  → Switch to [Chain] button (useSwitchChain)
3. Needs approval → Approve button (with dual-state locking)
4. Ready          → Action button (Stake/Deposit/Swap/etc.)
```

**Never show Approve and Action buttons simultaneously.**

### Approve Button: Dual-State Protection

`isPending` from wagmi clears when the wallet returns the tx hash — NOT when the tx confirms. This creates a window where the button re-enables mid-flight.

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
  {isPending || approvalSubmitting ? "Approving..." : approveCooldown ? "Confirming..." : "Approve"}
</button>
```

- `approvalSubmitting` covers click-to-hash gap
- `approveCooldown` covers confirm-to-cache-refresh gap
- Both must be on the `disabled` prop
- `approvalSubmitting` must clear in `finally {}` (handles rejection)

### Button Loading States — DaisyUI Gotcha

```tsx
// WRONG — DaisyUI "loading" class on a btn replaces content with a full-width spinner
<button className={`btn btn-primary ${isPending ? "loading" : ""}`}>

// CORRECT — inline spinner span, text stays visible
<button className="btn btn-primary" disabled={isPending}>
  {isPending && <span className="loading loading-spinner loading-sm mr-2" />}
  {isPending ? "Staking..." : "Stake"}
</button>
```

### Styling — DaisyUI Semantic Classes

```tsx
// CORRECT — responds to light/dark theme toggle
<div className="min-h-screen bg-base-200 text-base-content">
<div className="card bg-base-100 shadow-xl">
<button className="btn btn-primary">

// WRONG — hardcoded dark background, ignores theme system
<div className="min-h-screen bg-[#0a0a0a] text-white">
```

### Display Standards

- Show USD values next to ALL token/ETH amounts: `"0.5 ETH (~$1,250)"`
- Display the contract address using `<Address />`
- Format amounts with `formatEther()` / `formatUnits()` — never show raw wei/BigInt
- Map every contract error to a human-readable message

### Error Handling

```tsx
import { getParsedError } from "~~/utils/scaffold-eth";
import { notification } from "~~/utils/scaffold-eth";

try {
  await writeContractAsync({ functionName: "...", args: [...] });
} catch (e) {
  const parsed = getParsedError(e);
  notification.error(parsed);
}
```

Never `console.error(err)` alone. Users can't see the console.

### Mobile Deep Linking

RainbowKit v2 does NOT auto-deep-link to wallet apps. You must do it:

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
3. Check WalletConnect session data — `connector.id` alone won't tell you which wallet.
4. Wrap EVERY write call.

### Number Formatting

```typescript
import { formatUnits, parseUnits, formatEther, parseEther } from "viem";

// Display: BigInt → human readable
formatEther(weiAmount)           // "1.5" (18 decimals)
formatUnits(usdcAmount, 6)       // "100.0" (6 decimals)

// Input: human readable → BigInt
parseEther("1.5")                // 1500000000000000000n
parseUnits("100", 6)             // 100000000n
```

Never display raw BigInt values to users.

---

## Appendix C: SE2 Footguns

These are specific to Scaffold-ETH 2 and will bite you if you don't know about them.

| Footgun | Fix |
|---------|-----|
| Pre-existing TS error in `useScaffoldEventHistory.ts:132` | `(deployedContractData as any).deployedOnBlock` |
| Font loading — `<link>` tag for Google Fonts | Use `next/font/google` in layout.tsx |
| `polyfill-localstorage.cjs` in wrong directory | Must be in `packages/nextjs/`, not project root |
| Build output in wrong directory | Upload `packages/nextjs/out/`, not `out/` at root |
| Block explorer crashes static export | Delete `app/blockexplorer/` directory |
| Debug page incompatible with static export | Delete `app/debug/` directory |
| `deployedContracts.ts` manually edited | Never edit — it's auto-generated by `yarn deploy` |
| Default `pollingInterval: 30000` | Change to 3000 in scaffold.config.ts |
| `--radius-field: 9999rem` | Change to 0.5rem in both theme blocks |
| `appName: "scaffold-eth-2"` in wagmiConnectors | Change to your app name |
| `nativeCurrencyPrice` in Footer | Remove — renders ETH price badge on all networks |
| `bgipfs init` without `--endpoint` flag | Creates bad config pointing to localhost |
| Using `useWriteContract` instead of `useScaffoldWriteContract` | Scaffold hooks wait for block confirmation; raw wagmi doesn't |
| `yarn ipfs` exit code swallows errors | Only valid signal is CID in stdout (Qm... or bafy...) |
| Tailwind v3 directives in v4 project | Replace `@tailwind base;` with `@import "tailwindcss";` |
| `@apply` with DaisyUI tokens | Use className attributes only |
| `getDefaultConfig()` in layout.tsx | Replace with SE2 `ScaffoldEthAppWithProviders` pattern |

---

## Appendix D: Real Build Failures

These are not theoretical — they happened on actual builds.

| Mistake | Job | Impact |
|---------|-----|--------|
| Subagent reported "all SE2 branding removed" but didn't change the files | #46 | 15 of 21 QA items still failing. Full audit+fix cycle wasted. |
| Approve button re-enabled between wallet hash return and on-chain confirmation | #43 | Users could double-approve, wasting gas |
| TWAP oracle check ran AFTER the swap instead of before | #46 | Sandwich attack protection completely ineffective |
| `minWstETHOut: 0n` in frontend deposit call | #46 | Zero slippage protection despite contract having the parameter |
| Withdraw principal tracking subtracted raw assets instead of proportional | #46 | Share accounting drift over time |
| OG image pointed to `localhost:3000` | #43, #46 | Social sharing/unfurling completely broken |
| Deploy script transferred ownership before configuring cross-references | #46 | Nobody could call admin functions |
| `pool.observe()` called without try/catch | #46 | Would revert on new pools |
| Hardcoded `nextJobId: 19` in skill file when live contract was at 42 | leftclaw | Workers couldn't see real jobs |
| Used public RPC `mainnet.base.org` as fallback | leftclaw | API endpoints failing intermittently |
| `description` vs `descriptionCID` field name mismatch | leftclaw | Empty descriptions in API responses |

---

## Appendix E: Verification Commands

Run these after EVERY build stage to verify claims before proceeding:

```bash
# === COMPILATION ===
cd packages/foundry && forge build && echo "CONTRACTS: PASS" || echo "CONTRACTS: FAIL"
yarn next:build && echo "FRONTEND: PASS" || echo "FRONTEND: FAIL"

# === SE2 BRANDING CHECK ===
grep -c "scaffold-eth-2" packages/nextjs/services/web3/wagmiConnectors.tsx && echo "appName: FAIL" || echo "appName: PASS"
grep -c "Scaffold-ETH 2" packages/nextjs/utils/scaffold-eth/getMetadata.ts && echo "title: FAIL" || echo "title: PASS"
grep -c "BuidlGuidl" packages/nextjs/components/Footer.tsx && echo "footer: FAIL" || echo "footer: PASS"
grep -c "9999rem" packages/nextjs/styles/globals.css && echo "radius: FAIL" || echo "radius: PASS"
ls packages/nextjs/app/blockexplorer 2>/dev/null && echo "blockexplorer: FAIL" || echo "blockexplorer: PASS"

# === APPROVAL FLOW CHECK ===
grep -c "approvalSubmitting" packages/nextjs/app/page.tsx && echo "approvalSubmitting: PRESENT" || echo "approvalSubmitting: MISSING"
grep -c "approvalCooldown\|approveCooldown" packages/nextjs/app/page.tsx && echo "approveCooldown: PRESENT" || echo "approveCooldown: MISSING"
grep -c "finally" packages/nextjs/app/page.tsx && echo "finally block: PRESENT" || echo "finally block: MISSING"

# === RAW WAGMI CHECK ===
grep -rn "useWriteContract\|useReadContract" packages/nextjs/ | grep -v scaffold-eth | grep -v node_modules && echo "RAW WAGMI: FAIL" || echo "RAW WAGMI: PASS"

# === HARDCODED BACKGROUNDS ===
grep -rn 'bg-\[#0\|bg-black\|bg-gray-9\|bg-zinc-9' packages/nextjs/app/ && echo "DARK BG: FAIL" || echo "DARK BG: PASS"

# === SECRETS CHECK ===
grep -rn "0x[a-fA-F0-9]\{64\}" packages/ && echo "POSSIBLE SECRET: FAIL" || echo "SECRETS: PASS"
grep -rn "g.alchemy.com/v2/[A-Za-z0-9]" packages/ && echo "HARDCODED RPC: FAIL" || echo "RPC: PASS"

# === OG IMAGE ===
grep -rn "localhost" packages/nextjs/utils/ packages/nextjs/app/layout.tsx && echo "LOCALHOST REF: FAIL" || echo "LOCALHOST: PASS"

# === PUBLIC RPC ===
grep -n "mainnet.base.org\|base.llamarpc" packages/foundry/foundry.toml && echo "PUBLIC RPC: FAIL" || echo "RPC: PASS"
```

**Never trust "all done" from a subagent. Verify before advancing.**

---

## Appendix F: Verified Contract Addresses

ALWAYS cross-reference any address in deploy scripts or params against this table.

### Base (Chain ID: 8453)

| Contract | Address |
|----------|---------|
| WETH | `0x4200000000000000000000000000000000000006` |
| wstETH | `0xc1CBa3fCea344f92D9239c08C0568f6F2F0ee452` |
| USDC | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` |
| DAI | `0x50c5725949A6F0c72E6C4a641F24049A917DB0Cb` |
| Uniswap V3 Router | `0x2626664c2603336E57B271c5C0b26F421741e481` |
| Uniswap V3 Factory | `0x33128a8fC17869897dcE68Ed026d694621f6FDfD` |
| Aave V3 Pool | `0xA238Dd80C259a72e81d7e4664a9801593F98d1c5` |

### Ethereum Mainnet (Chain ID: 1)

| Contract | Address |
|----------|---------|
| WETH | `0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2` |
| wstETH | `0x7f39C581F595B53c5cb19bD0b3f8dA6c935E2Ca0` |
| stETH | `0xae7ab96520DE3A18E5e111B5EaAb095312D7fE84` |
| USDC | `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` |
| USDT | `0xdAC17F958D2ee523a2206206994597C13D831ec7` |
| DAI | `0x6B175474E89094C44Da98b954EedeAC495271d0F` |
| Uniswap V3 Router | `0xE592427A0AEce92De3Edee1F18E0157C05861564` |
| Uniswap V3 Factory | `0x1F98431c8aD98523631AE4a59f267346ea31F984` |
| Aave V3 Pool | `0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2` |
| Chainlink ETH/USD | `0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419` |

### Arbitrum (Chain ID: 42161)

| Contract | Address |
|----------|---------|
| WETH | `0x82aF49447D8a07e3bd95BD0d56f35241523fBab1` |
| USDC | `0xaf88d065e77c8cC2239327C5EDb3A432268e5831` |
| Uniswap V3 Router | `0xE592427A0AEce92De3Edee1F18E0157C05861564` |

### Optimism (Chain ID: 10)

| Contract | Address |
|----------|---------|
| WETH | `0x4200000000000000000000000000000000000006` |
| USDC | `0x0b2C639c533813f4Aa9D7837CAf62653d097Ff85` |

---

## Appendix G: Quick Reference Tables

### File Locations

| What | Where |
|------|-------|
| Smart contracts | `packages/foundry/contracts/` |
| Contract tests | `packages/foundry/test/` |
| Deploy scripts | `packages/foundry/script/` |
| Frontend pages | `packages/nextjs/app/` |
| Components | `packages/nextjs/components/` |
| Scaffold hooks | `packages/nextjs/hooks/scaffold-eth/` |
| UI components | `@scaffold-ui/components` |
| Auto-generated ABIs | `packages/nextjs/contracts/deployedContracts.ts` |
| External contracts | `packages/nextjs/contracts/externalContracts.ts` |
| Network config | `packages/nextjs/scaffold.config.ts` |
| Styles | `packages/nextjs/styles/globals.css` |
| Wallet config | `packages/nextjs/services/web3/wagmiConnectors.tsx` |
| API routes | `packages/nextjs/app/api/` |
| Foundry config | `packages/foundry/foundry.toml` |
| Metadata | `packages/nextjs/utils/scaffold-eth/getMetadata.ts` |

### Commands

```bash
# Development
yarn chain                          # Start local Anvil
yarn fork --network base            # Fork Base mainnet
yarn deploy                         # Deploy contracts locally
yarn start                          # Start frontend (localhost:3000)

# Quality
yarn lint                           # Lint everything
yarn format                         # Format everything
yarn next:build                     # Build frontend
forge test -vvv                     # Run contract tests
forge test --match-test testFuzz    # Run fuzz tests only

# Production
yarn deploy --network base          # Deploy to live network
yarn verify --network base          # Verify on block explorer
yarn ipfs                           # Deploy frontend to IPFS
```

### Import Paths

```typescript
// Scaffold hooks
import { useScaffoldReadContract, useScaffoldWriteContract } from "~~/hooks/scaffold-eth";
import { useDeployedContractInfo } from "~~/hooks/scaffold-eth";
import { useScaffoldEventHistory } from "~~/hooks/scaffold-eth";

// UI components
import { Address, AddressInput, Balance, EtherInput } from "@scaffold-ui/components";

// Connect button
import { RainbowKitCustomConnectButton } from "~~/components/scaffold-eth";

// Notifications
import { notification } from "~~/utils/scaffold-eth";

// Error parsing
import { getParsedError } from "~~/utils/scaffold-eth";

// Viem utilities
import { formatEther, parseEther, formatUnits, parseUnits } from "viem";
```

### Skill Reference URLs

All skills at `https://ethskills.com/<name>/SKILL.md`:

| Skill | When to read it |
|-------|----------------|
| `orchestration` | Before planning phases |
| `security` | Before writing any contract |
| `testing` | Before writing tests |
| `openzeppelin` | When using OZ contracts |
| `erc-721` | When building NFTs |
| `standards` | For ERC-8004, x402, EIP-7702 |
| `frontend-ux` | Before building any UI with wallet interaction |
| `frontend-playbook` | Before deploying frontend |
| `qa` | After deployment, before sharing |
| `audit` | For contract security review |
| `gas` | When estimating costs or choosing chains |
| `l2s` | When choosing or deploying to L2 |
| `tools` | When setting up dev environment |
| `wallets` | When handling keys, multisig, or account abstraction |
| `concepts` | When designing incentives or explaining to users |
| `indexing` | When you need historical onchain data |
| `money-legos` | When composing with DeFi protocols |

SE2 docs: `https://docs.scaffoldeth.io/SKILL.md`
SE2 AGENTS.md: `https://github.com/scaffold-eth/scaffold-eth-2/blob/main/AGENTS.md`

---

## Appendix H: Model & Cost Routing

| Step type | Model | Rationale |
|-----------|-------|-----------|
| Spec reading, shell commands, env checks | cheap (minimax-m2.7) | JSON extraction, no generation needed |
| Architecture planning, step extraction, evaluation | medium (claude-sonnet-4.6) | Requires reasoning, not just execution |
| Contract code gen | expensive (claude-opus-4.6) | Solidity correctness is critical, hard to fix |
| Deploy script code gen | medium | Templated pattern, less creative |
| Test code gen | medium | Tests are fixable, medium quality sufficient |
| Frontend component code gen | medium | TypeScript is more forgiving than Solidity |
| Audit analysis | medium | Pattern matching + reasoning |
| Semantic code review | expensive | Needs strong reasoning for spec-vs-code comparison |
| QA checklist automation | cheap | Mostly grep/pattern checks |

Target budget per build: $2-5 for a standard 3-contract dApp.

---

## Reference: Escalation Protocol

If you hit anything you cannot resolve at any stage:

1. Post escalation: `POST /api/job/{id}/messages` with `{ type: "escalation", metadata: { question: "...", stage: "current_stage" } }`
2. Set stage to blocked: `logWork(jobId, "Blocked: <reason>", "blocked")`
3. STOP. Do not continue work while blocked.
4. Before resuming: `GET /api/job/{id}/messages` — check for `escalation_response`

## Reference: Client Ownership

Every privileged role in every contract MUST be set to `job.client`:
- `owner`, `admin`, `deployer`, `feeOwner`, `treasury`, `governor`
- Constructor args that take an admin/owner address
- `transferOwnership` calls
- Multisig signer slots

**Never use your own address, Austin's address, or any CLAWD internal wallet as owner.** Read `job.client` from the job data at runtime.

## Reference: Retry & Escalation Rules

- **Compile fails:** Parse error, fix, retry. Max 3 attempts.
- **Test fails:** If assertion fails, fix the test (not the contract). Max 3 attempts.
- **Build fails (Module not found — npm):** `yarn add <package>`, retry.
- **Build fails (Module not found — local):** Fix import path using discovered exports.
- **Deploy reverts:** Check constructor args against discovery ABIs. Fix mismatched interfaces.
- **After 3 failed retries on critical path:** Stop and report. Provide error + suggested fix for manual intervention.
- **After 3 failed retries on non-blocking branch (tests):** Log failure, continue. Tests failing doesn't block deploy.

## Reference: Auto-Fix Strategies

| Error type | Fix strategy |
|------------|-------------|
| Solidity compiler error | Parse error location, send broken file + error to LLM, get fixed file |
| Forge test failure (EVM revert) | Diagnose whether bug is in contract or test. Fix the right file. |
| Module not found (npm package) | `yarn add <package>`, retry build |
| Module not found (local alias) | Fix import path using discovered package exports |
| TypeScript error | Send broken file + error to LLM, get fixed file |
| `yarn ipfs` no CID | Check if debug/blockexplorer pages were removed. Remove and retry. |
| Deploy revert | Check deploy script constructor args against discovery ABIs |
| CSS build error | Replace Tailwind v3 directives with v4, strip `@apply` with DaisyUI tokens |
