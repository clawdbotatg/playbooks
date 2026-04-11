# Comprehensive dApp Building Playbook

From spec to finished, deployed, audited decentralized application. Every step, every gate, every verification, every known failure mode.

This playbook synthesizes six prior playbooks, the LeftClaw build pipeline, ethskills.com standards, SE2 AGENTS.md, and lessons from real builds into a single authoritative document. Follow it in order. Skip nothing.

---

## Table of Contents

**Philosophy & Architecture**
1. [Core Philosophy](#1-core-philosophy)
2. [Pipeline Overview](#2-pipeline-overview)
3. [The Three-Phase Model](#3-the-three-phase-model)

**Pre-Code**
4. [Stage 0: Spec Ingestion](#4-stage-0-spec-ingestion)
5. [Stage 1: Architecture & Planning](#5-stage-1-architecture--planning)
6. [Stage 2: Discovery](#6-stage-2-discovery)
7. [Stage 3: Spec Verification Gate](#7-stage-3-spec-verification-gate)

**Build (Phase 1 — Local)**
8. [Stage 4: Repository & Scaffold](#8-stage-4-repository--scaffold)
9. [Stage 5: Smart Contracts](#9-stage-5-smart-contracts)
10. [Stage 6: Deploy Scripts](#10-stage-6-deploy-scripts)
11. [Stage 7: Tests](#11-stage-7-tests)
12. [Stage 8: Compile & Test Gate](#12-stage-8-compile--test-gate)
13. [Stage 9: Contract Verification Gate](#13-stage-9-contract-verification-gate)
14. [Stage 10: Contract Audit](#14-stage-10-contract-audit)
15. [Stage 11: Contract Audit Fixes](#15-stage-11-contract-audit-fixes)
16. [Stage 12: Frontend Development](#16-stage-12-frontend-development)
17. [Stage 13: Frontend QA Audit](#17-stage-13-frontend-qa-audit)
18. [Stage 14: Frontend QA Fixes](#18-stage-14-frontend-qa-fixes)
19. [Stage 15: Full Integration Audit](#19-stage-15-full-integration-audit)

**Deploy (Phase 2 — Live Contracts)**
20. [Stage 16: Deploy Contracts](#20-stage-16-deploy-contracts)
21. [Stage 17: Live Contract Verification](#21-stage-17-live-contract-verification)

**Ship (Phase 3 — Production)**
22. [Stage 18: Deploy Frontend to BGIPFS](#22-stage-18-deploy-frontend-to-bgipfs)
23. [Stage 19: Live App Testing](#23-stage-19-live-app-testing)
24. [Stage 20: Live User Journey Walkthrough](#24-stage-20-live-user-journey-walkthrough)
25. [Stage 21: README & Delivery](#25-stage-21-readme--delivery)

**Reference**
26. [Model Routing](#26-model-routing)
27. [Skill Routing](#27-skill-routing)
28. [File Preloading for Sub-Agents](#28-file-preloading-for-sub-agents)
29. [Retry & Escalation Rules](#29-retry--escalation-rules)
30. [Regression Protocol](#30-regression-protocol)
31. [Solidity Rules](#31-solidity-rules)
32. [Frontend Rules](#32-frontend-rules)
33. [BGIPFS Deploy Rules](#33-bgipfs-deploy-rules)
34. [Security Checklist](#34-security-checklist)
35. [Verified Contract Addresses](#35-verified-contract-addresses)
36. [SE2 Footguns](#36-se2-footguns)
37. [Common Failure Modes](#37-common-failure-modes)
38. [QA Checklists](#38-qa-checklists)
39. [Automation Reference](#39-automation-reference)

---

## 1. Core Philosophy

### Spec fidelity over pipeline completion

A build that compiles, passes tests, and deploys is **worthless** if it doesn't implement what the spec says. The pipeline must verify not just code health (does it compile?) but spec fidelity (does it do the right thing?).

### Do the thing, prove the thing works

Every stage that produces code has a corresponding verification gate. The gate checks two things:
1. **Does the code run?** (compiles, tests pass, frontend builds)
2. **Does the code do what the spec requires?** (correct underlying asset, correct swap path, correct yield mechanism)

Gate #2 is where AI agents fail. They optimize for #1 and skip #2.

### Never optimize for pipeline progress

If a later stage reveals the architecture is wrong, go back. Moving forward with a broken foundation creates exponentially more work. A regression to Stage 1 costs one stage. Shipping a broken product costs a complete rewrite.

### Discover before you generate

Before generating ANY code, discover what actually exists on-chain and in the project. AI agents hallucinate interfaces (calling `wstETH.wrap()` on Base where wstETH is a plain bridged ERC20), hallucinate import paths, and hallucinate addresses. Discovery eliminates this class of bugs entirely.

### Fork real chains

Never use `yarn chain` (empty local chain). Always `yarn fork --network base` so contracts interact with real Uniswap pools, real token contracts, real oracle data. Empty chains hide integration bugs that surface on deployment.

---

## 2. Pipeline Overview

```
SPEC INGESTION ─► ARCHITECTURE ─► DISCOVERY ─► SPEC VERIFICATION GATE
                                                        │
                                        ┌───────────────┘
                                        ▼
                               SCAFFOLD + REPO
                                        │
                    ┌───────────────────┼───────────────────┐
                    ▼                   ▼                   ▼
              CONTRACTS           DEPLOY SCRIPTS        INTERFACES
                    │                   │                   │
                    └───────────────────┼───────────────────┘
                                        ▼
                               COMPILE + TEST GATE
                                        │
                               CONTRACT VERIFICATION GATE
                                        │
                               CONTRACT AUDIT ─► FIXES
                                        │
                                   FRONTEND
                                        │
                               FRONTEND QA ─► FIXES
                                        │
                               FULL INTEGRATION AUDIT
                                        │
                    ┌───────────────────┘
                    ▼
           DEPLOY CONTRACTS (Phase 2)
                    │
           LIVE CONTRACT VERIFICATION
                    │
           DEPLOY FRONTEND (Phase 3)
                    │
           LIVE APP TEST
                    │
           USER JOURNEY WALKTHROUGH
                    │
           README + DELIVERY
```

Every `─►` is a gate. If the gate fails, you go back. No exceptions.

---

## 3. The Three-Phase Model

From the ethskills orchestration skill. Development follows three strict phases with hard gates between them.

| Phase | Environment | What's Live | Gate to Exit |
|-------|-------------|-------------|--------------|
| **Phase 1: Build** | Local fork of target chain | Nothing — all localhost | Contracts compile, tests pass, frontend builds, full user journey works locally |
| **Phase 2: Deploy** | Live network + local UI | Contracts on-chain | Contracts deployed, verified on explorer, `deployedContracts.ts` updated, all flows work with real wallet |
| **Phase 3: Ship** | Production (BGIPFS) | Everything | Frontend on IPFS, CID captured, QA checklist passes, user journey works end-to-end live |

**Never combine phases. Never skip a gate.**

**Regression rules:**
- Production frontend bug → go back to Phase 2 (fix locally against live contracts)
- Contract logic bug → go back to Phase 1 (fix contracts, retest, redeploy everything)
- Never patch production directly

---

## 4. Stage 0: Spec Ingestion

**Input:** Job description (on-chain or provided)
**Output:** `SPEC_REQUIREMENTS.md`
**Model:** minimax-m2.7 (reading and extracting, no code generation)

### Actions

1. **Read the job description** — on-chain `description` field or from API
2. **Read ALL messages** — `GET /api/job/{id}/messages`. Clients add requirements, preferences, and scope changes via chat AFTER posting. Chat is authoritative. On-chain description is the baseline; chat may override it.
3. **Extract and document:**
   - What the app does (one sentence)
   - What tokens/protocols are involved (exact names — addresses come in Discovery)
   - What the yield/reward/value mechanism is (how does money flow?)
   - Who owns what (`job.client` address = owner of everything)
   - What chain (default: Base)
   - What external contracts are needed

### The Onchain Litmus Test

Ask: "Does this app actually need a blockchain?" If it could be a regular web app, flag it. Every contract must have a reason to exist onchain.

### State Transition Audit

For every planned contract function:

| Function | Who calls it? | Why would they? | What if nobody calls it? | Gas incentive? |
|----------|--------------|-----------------|--------------------------|----------------|

Smart contracts cannot execute themselves. Every function needs a caller who pays gas. If the answer to "who calls it?" is "the team" — redesign.

### Write SPEC_REQUIREMENTS.md

List every requirement as a numbered, testable item:

```
1. The vault uses wstETH as its underlying ERC4626 asset
2. Deposits can be made in ETH, WETH, or wstETH directly
3. Yield = totalAssets() - totalPrincipalWstETH (wstETH appreciation)
4. Harvester swaps yield via multi-hop: wstETH → WETH → CLAWD
5. 50% of harvested CLAWD is burned to 0xdead
6. 50% is distributed via Synthetix StakingRewards to vault share stakers
7. TWAP oracle protects against sandwich attacks on the swap
8. Client address owns all contracts
```

Each item must be **testable** — "the vault uses wstETH as underlying" not "the vault works with ETH staking."

### Verification

SPEC_REQUIREMENTS.md exists with numbered, testable requirements. Every requirement maps to a specific behavior, not a vague goal.

---

## 5. Stage 1: Architecture & Planning

**Input:** SPEC_REQUIREMENTS.md, job description
**Output:** `PLAN.md`, `USERJOURNEY.md`
**Model:** claude-sonnet-4.6

### PLAN.md Contents (all 10 sections required)

1. **One-sentence summary** of what the app does
2. **Architecture diagram** (text-based) showing contract relationships
3. **Contract specifications** for each contract:
   - Name, purpose, inheritance chain
   - State variables with types
   - Every external function with signature, access control, and behavior
   - Events and custom errors
   - Which external protocols it calls (exact function signatures)
4. **Yield/reward mechanism** step by step:
   - Where value comes from
   - How it's captured (e.g., wstETH appreciation)
   - How it's converted (e.g., multi-hop swap path)
   - How it's distributed (e.g., burn + staking rewards)
5. **Deploy script** with exact constructor arguments
6. **Post-deploy configuration** steps (setHarvester, setRewardDistributor)
7. **Deployer-first ownership pattern:**
   ```
   1. Deploy with deployer as initial owner
   2. Configure cross-references (requires owner)
   3. Transfer ownership to client via transferOwnership()
   4. Client calls acceptOwnership() (Ownable2Step)
   ```
8. **Frontend specification:** tabs/sections, what each shows, SE2 hooks for each interaction
9. **External addresses** — every address needed (verified in Stage 2)
10. **Security considerations** — attack vectors and mitigations

### USERJOURNEY.md Contents

**Happy path** (step by step):
1. User opens app URL — describe landing state with no wallet
2. User clicks Connect Wallet — describe what happens
3. User is on wrong network — describe Switch Network prompt
4. User performs primary action — each click, each transaction, each confirmation
5. User sees result — describe success state

**Edge cases** (must cover ALL):
- No wallet installed
- Wrong network connected
- Insufficient balance (for gas AND for the token)
- Transaction rejected by user
- Transaction reverted on-chain
- Slow transaction (pending state)
- Multiple rapid clicks (double-submit)
- Mobile wallet via WalletConnect

### Verification

- PLAN.md has all 10 sections
- USERJOURNEY.md covers happy path + at least 6 edge cases
- The walkaway test passes: if the owner disappears, can users still withdraw?

---

## 6. Stage 2: Discovery

**Input:** PLAN.md, SPEC_REQUIREMENTS.md
**Output:** `discovery.json`
**Model:** minimax-m2.7

**This is the stage that prevents hallucinated interfaces.** Before any code is generated, verify what actually exists.

### 2A: Discover On-Chain Interfaces

For every external contract address in the plan, fetch the real ABI:

```bash
cast interface <address> --chain base
```

Store the result. What this catches: contracts that exist on mainnet with rich interfaces (Lido wstETH with `wrap()`, `unwrap()`, `stETH()`) but are bridged on L2s as simple ERC20s. Without discovery, the LLM writes code against the mainnet interface and the deploy reverts.

### 2B: Verify Every Address

```bash
# Verify contract exists
cast code <address> --rpc-url $ALCHEMY_RPC_URL
# Returns 0x if no contract exists

# Verify token identity
cast call <address> "symbol()(string)" --rpc-url $ALCHEMY_RPC_URL
cast call <address> "decimals()(uint8)" --rpc-url $ALCHEMY_RPC_URL

# Verify pool exists (Uniswap V3)
cast call <FACTORY> "getPool(address,address,uint24)(address)" <tokenA> <tokenB> <fee> --rpc-url $ALCHEMY_RPC_URL
# Zero address = pool doesn't exist

# Verify router
cast call <ROUTER> "factory()(address)" --rpc-url $ALCHEMY_RPC_URL

# Verify ownership target
cast call <address> "owner()(address)" --rpc-url $ALCHEMY_RPC_URL
```

**Never hallucinate contract addresses.** Cross-reference against the [Verified Contract Addresses](#35-verified-contract-addresses) table. If an address has wrong length or wrong checksum, query the authoritative source.

### 2C: Discover Package Exports (after scaffolding)

Read actual TypeScript exports from installed packages:

```bash
# SE2 hooks
cat packages/nextjs/hooks/scaffold-eth/index.ts

# UI components
cat packages/nextjs/node_modules/@scaffold-ui/components/dist/types/index.d.ts

# Scaffold-eth component re-exports
cat packages/nextjs/components/scaffold-eth/index.tsx
```

What this catches: Import path errors. The LLM writes `import { Address } from "~~/components/scaffold-eth/Address"` but the actual export is from `@scaffold-ui/components`.

### 2D: Discover SE2 Deploy Pattern

```bash
cat packages/foundry/script/DeployHelpers.s.sol
cat packages/foundry/script/Deploy.s.sol
head -5 packages/nextjs/styles/globals.css   # CSS framework version
cat packages/nextjs/next.config.ts
cat packages/nextjs/scaffold.config.ts
```

### Discovery Artifact

```json
{
  "onChain": {
    "0xc1CBa3fCea344f92D9239c08C0568f6F2F0ee452": {
      "name": "wstETH (Base)",
      "functions": ["function balanceOf(address) view returns (uint256)", "..."],
      "note": "Bridged ERC20 only. No wrap(), unwrap(), or Lido staking functions."
    }
  },
  "packageExports": {
    "hooks": ["useScaffoldReadContract", "useScaffoldWriteContract", "..."],
    "uiComponents": {
      "@scaffold-ui/components": ["Address", "AddressInput", "Balance", "EtherInput"],
      "~~/components/scaffold-eth": ["RainbowKitCustomConnectButton"]
    }
  },
  "verifiedAddresses": {
    "wstETH": { "address": "0xc1CBa3...", "verified": true, "symbol": "wstETH", "decimals": 18 },
    "CLAWD/WETH Pool": { "address": "0xCD553...", "verified": true, "token0": "0x...", "token1": "0x..." }
  }
}
```

**discovery.json is passed to every subsequent code generation step.** The LLM cannot generate code against an interface that isn't in discovery.

### Verification

- Every address in PLAN.md verified with `cast code` (non-zero)
- Every token verified with `cast call symbol()`
- Every pool verified via factory `getPool()`
- discovery.json written with on-chain ABIs and package exports

---

## 7. Stage 3: Spec Verification Gate

**Input:** SPEC_REQUIREMENTS.md, PLAN.md, discovery.json
**Output:** Verified spec mapping
**Model:** claude-sonnet-4.6

**This gate prevents the WETH-vs-wstETH class of bugs.** Before writing any code, verify the plan actually implements the spec.

### Actions

Go through SPEC_REQUIREMENTS.md line by line. For each requirement:

1. Find where in PLAN.md it is addressed
2. Verify the plan's approach actually satisfies the requirement
3. Cross-reference against discovery.json — does the approach use functions that actually exist?

### DeFi-Specific Checks

- [ ] **Underlying asset is correct.** If the spec says "ETH staking yield," the vault MUST use wstETH/stETH as underlying (they appreciate), NOT WETH/ETH (which don't).
- [ ] **Yield mechanism is real.** Walk through `availableYield()` with numbers. If wstETH goes from 1.0 to 1.05 exchange rate, does the function return 5% of deposits?
- [ ] **Swap path exists.** Every pool in the swap path verified via factory `getPool()`.
- [ ] **Token behaviors match assumptions.** If discovery shows wstETH on Base is a plain ERC20 (no `wrap()`), the plan must not include `wstETH.wrap()`.
- [ ] **Token decimals correct.** USDC = 6, not 18. wstETH = 18. Wrong decimals break all math.
- [ ] **Owner is CLIENT.** Every `Ownable(_owner)` passes the client address.

### Verification

Every line in SPEC_REQUIREMENTS.md annotated with either:
- "VERIFIED — see PLAN.md section X, discovery.json confirms interface exists"
- "FAILED — plan says Y but spec says Z"

If ANY requirement fails: go back to Stage 1. **Do NOT proceed to code.**

---

## 8. Stage 4: Repository & Scaffold

**Output:** SE2 project with Git + GitHub
**Model:** minimax-m2.7

### Actions

1. Create build folder: `./builds/leftclaw-service-job-{id}_{timestamp}`
2. Scaffold:
   ```bash
   npx -y create-eth@latest . -s foundry
   ```
   - `-s foundry` selects the Solidity framework
   - No other flags exist (no --template, --chain, --skip-git)
   - Directory MUST be empty
3. Initialize Git + GitHub:
   ```bash
   git init
   git config user.name "clawdbotatg"
   git config user.email "clawdbotatg@users.noreply.github.com"
   git add -A && git commit -m "feat: scaffold SE2 Foundry"
   gh repo create clawdbotatg/leftclaw-service-job-{id} --public --source=. --remote=origin --push
   ```
4. Read `AGENTS.md` — it has the authoritative hook names, component names, and code style. Do not proceed without reading it.
5. Run package export discovery (Stage 2C) now that packages are installed.

### SE2 v2 Project Structure

```
packages/
├── foundry/
│   ├── contracts/          # Solidity source
│   ├── script/             # Deploy scripts (DeployHelpers.s.sol, Deploy.s.sol)
│   ├── test/               # Forge tests
│   └── foundry.toml        # Foundry config
├── nextjs/
│   ├── app/
│   │   ├── page.tsx        # Main page
│   │   └── layout.tsx      # Root layout (metadata, OG tags)
│   ├── components/
│   │   └── scaffold-eth/   # SE2 components
│   ├── contracts/
│   │   ├── deployedContracts.ts   # Auto-generated (NEVER manually edit)
│   │   └── externalContracts.ts   # Manually add external contracts
│   ├── hooks/scaffold-eth/        # SE2 hooks
│   ├── scaffold.config.ts         # Chain config, polling
│   ├── services/web3/wagmiConnectors.tsx  # Wallet config
│   └── styles/globals.css         # Theme variables
```

There is NO `packages/react-app/`. That was SE2 v1.

### Verification

- `forge build` succeeds in `packages/foundry`
- `packages/nextjs/app/page.tsx` exists
- GitHub repo visible at `https://github.com/clawdbotatg/leftclaw-service-job-{id}`

---

## 9. Stage 5: Smart Contracts

**Input:** PLAN.md, discovery.json
**Output:** Contract source files
**Model:** claude-opus-4.6

### 5.1 Delete Scaffold Defaults

```bash
rm packages/foundry/contracts/YourContract.sol
rm packages/foundry/script/DeployYourContract.s.sol
rm packages/foundry/test/YourContract.t.sol
```

### 5.2 Write Interfaces

Create `packages/foundry/contracts/interfaces/` for external protocols. **Only include functions you actually call.** Reference discovery.json for accurate function signatures.

### 5.3 Write Contracts

Location: `packages/foundry/contracts/`

**Mandatory patterns** (see [Solidity Rules](#31-solidity-rules) for full details):
- `Ownable2Step` (not `Ownable`) — prevents accidental ownership transfer
- `ReentrancyGuard` on every function that makes external calls
- `SafeERC20` for all token operations
- CEI pattern (Checks-Effects-Interactions)
- Custom errors, not `require` strings
- Events for every state change
- Constructor sets owner = deployer (transferred to client post-deploy)

**ERC4626 vaults:**
- `_decimalsOffset() >= 3` for inflation attack prevention
- Track principal separately if yield is redirected
- Override `_deposit`/`_withdraw` (internal hooks), NOT `deposit`/`withdraw` (public) — avoids double-nonReentrant
- For convenience deposit functions (depositETH, depositWETH): use `previewDeposit()` + `_mint()` directly — `super.deposit()` calls `transferFrom(msg.sender)` but tokens are already in the contract

**Uniswap V3 swaps:**
- Always set `amountOutMinimum > 0`
- TWAP oracle check BEFORE the swap, not after
- Wrap `pool.observe()` in try/catch (new pools may lack observation history)
- Multi-hop path: `abi.encodePacked(tokenA, fee1, tokenB, fee2, tokenC)`

**Access control:**
- Owner = client address (set post-deploy via ownership transfer)
- Keeper/harvester = separate role the owner can set
- Walkaway test: if the owner disappears, can users still withdraw? Answer must be yes.

### 5.4 Register External Contracts

Update `packages/nextjs/contracts/externalContracts.ts`:

```typescript
export default {
  8453: {
    CLAWD: {
      address: "0x9f86dB9fc6f7c9408e8Fda3Ff8ce4e78ac7a6b07",
      abi: [/* only functions the frontend calls */],
    },
  },
} as const;
```

Only include ABI entries the frontend actually calls. Never manually edit `deployedContracts.ts`.

### Verification

- All contracts have ReentrancyGuard, Ownable2Step, SafeERC20
- External calls only use functions from discovery.json
- Constructor args match PLAN.md
- No hardcoded addresses in contract logic (only in deploy scripts and external contracts)

**Stop condition:** Do NOT deploy. Do NOT write frontend.

---

## 10. Stage 6: Deploy Scripts

**Input:** Contract source files, PLAN.md
**Output:** Deploy scripts
**Model:** claude-sonnet-4.6

### Pattern

```solidity
contract DeployProject is ScaffoldETHDeploy {
    // All external addresses as constants (verified in discovery)
    address constant WSTETH = 0xc1CBa3fCea344f92D9239c08C0568f6F2F0ee452;
    address constant CLIENT = 0x34aA3F359A9D614239015126635CE7732c18fDF3;

    function run() external ScaffoldEthDeployerRunner {
        // 1. Deploy all contracts (deployer is initial owner)
        Vault vault = new Vault(WSTETH, deployer);
        Rewards rewards = new Rewards(address(vault), deployer);
        Harvester harvester = new Harvester(address(vault), address(rewards), deployer);

        // 2. Configure cross-references (requires owner)
        vault.setHarvester(address(harvester));
        rewards.setRewardDistributor(address(harvester));

        // 3. Transfer ownership to client
        vault.transferOwnership(CLIENT);
        rewards.transferOwnership(CLIENT);
        harvester.transferOwnership(CLIENT);

        // 4. Push to deployments array
        deployments.push(Deployment("Vault", address(vault)));
        deployments.push(Deployment("Rewards", address(rewards)));
        deployments.push(Deployment("Harvester", address(harvester)));
    }
}
```

**Critical: deployer-first pattern.** You can't configure cross-references after transferring ownership because only the owner can call admin functions. Deploy → configure → transfer.

### Verification

- Every constructor arg matches the contract constructor signature
- External addresses match [Verified Contract Addresses](#35-verified-contract-addresses)
- Ownership transferred to client as last step
- `forge build --skip test` succeeds

---

## 11. Stage 7: Tests

**Input:** Contract source files
**Output:** Test files
**Model:** claude-opus-4.6

Location: `packages/foundry/test/`

### What to Test (priority order)

1. **Constructor state** — all immutables set correctly
2. **Happy path** — full lifecycle (deposit → stake → harvest → claim → withdraw)
3. **Access control** — unauthorized callers revert
4. **Edge cases** — zero amounts, max amounts, first depositor
5. **Fuzz tests** — any function with math. Use `bound()` not `vm.assume()`.
6. **Integration** — full harvest cycle verifying yield capture, swap, burn, distribution

### Critical Test Rule

If a test assertion fails during the fix loop, **fix the test, not the contract.** The contract implements the spec; tests verify the contract. A failing assertion usually means the test expectation is wrong.

### Mock Pattern

For external protocols, use mocks with realistic values. Document what each mock simulates.

```solidity
contract MockERC20 is ERC20 {
    constructor(string memory name, string memory symbol) ERC20(name, symbol) {}
    function mint(address to, uint256 amount) external { _mint(to, amount); }
}
```

### Verification

Test files exist for each contract. Tests compile. Tests don't need to pass yet — that's Stage 8.

---

## 12. Stage 8: Compile & Test Gate

**Output:** Green build + green tests
**Model:** minimax-m2.7 (running commands), claude-sonnet-4.6 (fixing errors)

### Two-Phase Verify-Fix Loop

**Phase 1 — Compile:**
```bash
cd packages/foundry && forge build
```
On failure: parse error location, send broken file + error to fixer LLM, write fix, retry. Max 3 retries.

**Phase 2 — Test:**
```bash
cd packages/foundry && forge test -vv
```
On failure: diagnose whether bug is in contract or test. Fix the right file. Max 3 retries.

### Verification

- `forge build` exits 0, zero errors
- `forge test -vv` exits 0, all tests pass
- No `FAIL` in output

**Do NOT proceed with compilation errors or failing tests.**

---

## 13. Stage 9: Contract Verification Gate

**Input:** SPEC_REQUIREMENTS.md, contract source code
**Output:** Verified spec-to-code mapping
**Model:** claude-sonnet-4.6

**This gate prevents spec drift.** Code compiling and tests passing does NOT mean the code implements the spec correctly.

### Actions

Go through SPEC_REQUIREMENTS.md line by line. For each requirement, find the specific code that implements it and verify:

- [ ] **Underlying asset:** `asset()` returns the correct token (e.g., wstETH not WETH)
- [ ] **Yield calculation:** Walk through `availableYield()` with example numbers
- [ ] **Swap path:** `abi.encodePacked(...)` matches real Uniswap pools
- [ ] **Burn mechanics:** Correct percentage to correct address (0xdead)
- [ ] **Reward distribution:** `notifyRewardAmount` called with correct amount after CLAWD is transferred
- [ ] **Owner is CLIENT:** Every Ownable constructor passes client address (or deployer-first pattern)
- [ ] **No stub functions:** Every function has a real implementation. No `// TODO`.
- [ ] **No hardcoded test values:** All addresses, fees, parameters are mainnet values

If ANY requirement is not correctly implemented: fix the contracts and go back to Stage 8. **Do NOT proceed to audit with broken spec fidelity.**

---

## 14. Stage 10: Contract Audit

**Input:** Contract source files
**Output:** Audit findings as GitHub issues
**Model:** claude-sonnet-4.6
**Reference:** https://ethskills.com/audit/SKILL.md

### Standard Audit Checklist

**Critical:**
1. Reentrancy — guards on all external-calling functions
2. Access control — Ownable2Step, no leftover deployer privileges
3. Integer safety — overflow/underflow, unsafe casting
4. External call safety — return values checked, CEI pattern
5. Token handling — SafeERC20, no raw `transfer()`
6. Slippage protection — all swaps have minimum output
7. Oracle safety — TWAP checks, manipulation resistance
8. First depositor attacks — virtual shares for ERC4626

**DeFi-specific:**
9. Flash loan vectors — state manipulation in single block?
10. Sandwich attack vectors — swaps protected?
11. Yield accounting drift — principal tracking over many operations
12. Walkaway safety — users can always withdraw
13. Price manipulation — spot vs TWAP divergence

### Deep Audit (for complex contracts)

Run if: token swaps, multi-contract interactions, financial logic, >200 lines. Load specialized audit skills based on contract type:

| Contract Type | Load Skills |
|---|---|
| ERC4626 vaults | evm-audit-erc4626 (42+ items) |
| Staking/rewards | evm-audit-defi-staking (30+ items) |
| AMM/swaps | evm-audit-defi-amm (30+ items) |
| Oracle integration | evm-audit-oracles (29+ items) |

### File Issues

```bash
gh label create "contract-audit" --repo clawdbotatg/leftclaw-service-job-{id} --color "d93f0b" --force
gh issue create --repo clawdbotatg/leftclaw-service-job-{id} \
  --title "[SEVERITY] Finding title" \
  --body "**Location:** file:function\n**Description:** ...\n**Recommendation:** ..." \
  --label "job-{id},contract-audit"
```

### Verification

All findings filed as GitHub issues with severity labels. Audit report summarizes finding count by severity.

---

## 15. Stage 11: Contract Audit Fixes

**Model:** claude-opus-4.6

1. Fix by severity: Critical → must fix. High → must fix. Medium → fix or document. Low → fix if trivial.
2. After each fix: `forge build && forge test`
3. Close each issue referencing the fix commit
4. Re-run Stage 9 (Contract Verification Gate) to ensure fixes didn't break spec fidelity

### Verification

- Zero Critical/High findings open
- `forge test` still passes
- Spec verification gate still passes

---

## 16. Stage 12: Frontend Development

**Input:** Contract .sol files, AGENTS.md, scaffold.config.ts, discovery.json
**Output:** `packages/nextjs/app/page.tsx` and components
**Model:** claude-opus-4.6

See [Frontend Rules](#32-frontend-rules) for the complete reference. Key points:

### Configuration First

**scaffold.config.ts:**
```typescript
targetNetworks: [chains.base],  // or chains.foundry during Phase 1
pollingInterval: 3000,          // NOT 30000
```

**wagmiConnectors.tsx:**
- Change `appName` from "scaffold-eth-2" to your app name
- Add `phantomWallet` to the wallets array

### SE2 Branding Cleanup (mandatory)

- [ ] Footer: remove BuidlGuidl links, "Built with SE2", nativeCurrencyPrice badge
- [ ] Header: replace SE2 logo with project name, remove Debug Contracts link
- [ ] Tab title: project name, not "Scaffold-ETH 2"
- [ ] README: project content, not SE2 template
- [ ] Favicon: replace SE2 default
- [ ] manifest.json: update app name
- [ ] Block explorer: rename to `_blockexplorer-disabled` (crashes static export)

### Four-State Button Flow (mandatory)

Show exactly ONE primary button at a time:

```
1. Not connected  → Connect Wallet (RainbowKitCustomConnectButton)
2. Wrong network  → Switch to [Chain]
3. Needs approval → Approve (with dual-state locking)
4. Ready          → Action button
```

Never show Approve and Action simultaneously.

### Approval Button — Two-State Pattern

```tsx
const [approvalSubmitting, setApprovalSubmitting] = useState(false);
const [approveCooldown, setApproveCooldown] = useState(false);

const handleApprove = async () => {
  if (approvalSubmitting || approveCooldown) return;
  setApprovalSubmitting(true);
  try {
    await writeContractAsync({ functionName: "approve", args: [spender, amount] });
    setApproveCooldown(true);
    setTimeout(() => { setApproveCooldown(false); refetchAllowance(); }, 4000);
  } catch (e) { notification.error(getHumanError(e)); }
  finally { setApprovalSubmitting(false); }
};

<button disabled={isPending || approvalSubmitting || approveCooldown}>
  {(isPending || approvalSubmitting) && <span className="loading loading-spinner loading-sm mr-2" />}
  {isPending || approvalSubmitting ? "Approving..." : "Approve"}
</button>
```

### Critical Address Rule

**NEVER hardcode deployed contract addresses as string literals in React files.** Components needing a deployed address MUST call `useDeployedContractInfo("ContractName")` internally. The page composes components — it does NOT pass addresses as props.

### Build

```bash
yarn next:build
```

### Verification

- Build exits 0
- No raw wagmi hooks: `grep -rn "useWriteContract\|useReadContract" packages/nextjs/ | grep -v scaffold-eth | grep -v node_modules`
- No raw address inputs: `grep -rn 'type="text"' packages/nextjs/app/ | grep -i "addr\|0x"`
- No hardcoded dark: `grep -rn 'bg-\[#0\|bg-black' packages/nextjs/app/`

---

## 17. Stage 13: Frontend QA Audit

**Input:** Frontend source code
**Output:** PASS/FAIL checklist + GitHub issues
**Model:** claude-sonnet-4.6
**Reference:** https://ethskills.com/qa/SKILL.md

Do NOT fix anything — report only.

### Ship-Blocking (ALL must pass)

| # | Check | How to Verify |
|---|-------|---------------|
| 1 | Wallet shows BUTTON not text | Read page.tsx — look for `RainbowKitCustomConnectButton` or `openConnectModal` |
| 2 | Wrong network shows Switch | Look for chain ID check before action buttons |
| 3 | One button at a time | Four-state flow: Connect → Network → Approve → Action |
| 4 | Approve has two-state lock | Grep for `approvalSubmitting` AND `approveCooldown` |
| 5 | SE2 footer removed | No BuidlGuidl, Fork me, Support, nativeCurrencyPrice |
| 6 | SE2 tab title removed | No "Scaffold-ETH 2" in getMetadata/layout |
| 7 | SE2 README replaced | Project content, not template |
| 8 | Favicon replaced | Not SE2 default |
| 9 | No raw wagmi hooks | grep returns nothing outside scaffold-eth internals |
| 10 | No zero-address placeholders | No `0x000...0` in components |
| 11 | No hardcoded deployed addresses | No address string literals in components |

### Should-Fix (ALL should pass)

| # | Check | How to Verify |
|---|-------|---------------|
| 12 | Contract address with `<Address/>` | Grep for `<Address` in page.tsx |
| 13 | Address inputs use `<AddressInput/>` | No raw `<input type="text">` for addresses |
| 14 | USD values next to amounts | Check every amount display |
| 15 | OG image absolute URL | Not relative path |
| 16 | pollingInterval 3000 | Read scaffold.config.ts |
| 17 | `--radius-field: 0.5rem` | Both theme blocks in globals.css |
| 18 | No hardcoded dark backgrounds | `bg-base-200` not `bg-[#0a0a0a]` |
| 19 | Inline spinner not DaisyUI `loading` | `<span className="loading loading-spinner">` inside button |
| 20 | Phantom wallet included | `phantomWallet` in wagmiConnectors.tsx |
| 21 | Errors human-readable | Every catch has `notification.error()` |
| 22 | Amounts formatted | `formatEther`/`formatUnits`, never raw BigInt |
| 23 | appName changed | Not "scaffold-eth-2" in wagmiConnectors |
| 24 | manifest.json updated | Not "Scaffold-ETH 2 DApp" |
| 25 | Block explorer disabled | `app/blockexplorer` doesn't exist or renamed |
| 26 | Mobile deep linking | TX fires first, deep link after 2s delay |

File issues: `gh issue create --label "job-{id},frontend-audit"`

---

## 18. Stage 14: Frontend QA Fixes

**Model:** claude-opus-4.6

Fix every FAIL. Close issues with commit references. Rebuild: `yarn next:build`.

**Known issue — walletDeepLink:** The QA fixer sometimes introduces `import { writeAndOpen } from "~~/utils/scaffold-eth/walletDeepLink"` — a file that does NOT exist. If the build breaks after QA fixes, check for this import and remove it.

**Known issue — overwritten files:** The fixer can rewrite files beyond what's needed. `git diff` after fixes to catch unrelated changes.

---

## 19. Stage 15: Full Integration Audit

**Model:** claude-sonnet-4.6

One final pass across ALL components — contracts AND frontend together.

### Safety

- [ ] Users can always withdraw (no lockups, no admin freeze)
- [ ] Owner cannot steal deposits
- [ ] Reentrancy guards on all entry points
- [ ] SafeERC20 throughout
- [ ] TWAP/slippage on all swaps
- [ ] All privileged roles → client address

### Frontend-Contract Integration

- [ ] Frontend passes correct args to every contract function
- [ ] Slippage parameters are non-zero in frontend calls
- [ ] All contract addresses match deployed addresses
- [ ] Event names in `useScaffoldEventHistory` match contract events
- [ ] External contracts registered correctly in `externalContracts.ts`

### Build

```bash
forge build && forge test && yarn next:build
```

All three must pass. File issues for findings: `--label "job-{id},full-audit"`. Fix all.

---

## 20. Stage 16: Deploy Contracts

**Model:** claude-sonnet-4.6

### Configure RPC

In `foundry.toml`:
```toml
[rpc_endpoints]
base = "https://base-mainnet.g.alchemy.com/v2/${ALCHEMY_API_KEY}"
```

**NEVER use `mainnet.base.org` or any public RPC.**

### Deploy

```bash
cd packages/foundry
forge script script/DeployProject.s.sol \
  --rpc-url $ALCHEMY_RPC_URL \
  --private-key $ETH_PRIVATE_KEY \
  --broadcast \
  --verify
```

### Verify on Block Explorer

```bash
yarn verify --network base
```

Every deployed contract MUST show verified source with green checkmark. Unverified = trust red flag. Check manually on Basescan.

### Generate deployedContracts.ts

If `yarn deploy` was used, this is automatic. If `forge script` was used directly, run the TS ABI generator. Every contract entry needs: `address`, `abi`, `deployedOnBlock`.

### Verification

- All contracts deployed (addresses recorded)
- All verified on Basescan (green checkmark)
- `deployedContracts.ts` updated with real addresses
- Cross-references configured (harvester, distributor, etc.)
- Ownership transfer initiated to client

---

## 21. Stage 17: Live Contract Verification

**Model:** minimax-m2.7

Verify on-chain state matches expectations:

```bash
# Ownership
cast call <vault> "owner()(address)" --rpc-url $RPC
cast call <vault> "pendingOwner()(address)" --rpc-url $RPC

# Immutables
cast call <vault> "asset()(address)" --rpc-url $RPC        # must be wstETH
cast call <rewards> "stakingToken()(address)" --rpc-url $RPC # must be vault

# Cross-references
cast call <vault> "harvester()(address)" --rpc-url $RPC
cast call <rewards> "rewardDistributor()(address)" --rpc-url $RPC
```

If ANY value is wrong (especially zero address = critical bug), fix and redeploy.

---

## 22. Stage 18: Deploy Frontend to BGIPFS

**Model:** claude-sonnet-4.6

### Pre-Deploy

1. Update `scaffold.config.ts`: `targetNetworks: [chains.base]`
2. Create polyfill (see [BGIPFS Deploy Rules](#33-bgipfs-deploy-rules))
3. Remove `app/debug/` and `app/blockexplorer/` (use `force-dynamic`, incompatible with static export)

### Build

```bash
cd packages/nextjs
rm -rf .next out
NEXT_PUBLIC_IPFS_BUILD=true \
NEXT_PUBLIC_IGNORE_BUILD_ERROR=true \
NODE_OPTIONS="--require ./polyfill-localstorage.cjs" \
npm run build
```

### Upload

```bash
bgipfs upload packages/nextjs/out --config ~/.bgipfs/credentials.json
```

### Verify

```bash
curl -s "https://{CID}.ipfs.community.bgipfs.com/" | head -20
```

Must return HTML, not error page. CID must be different from any previous deploy (if code changed).

---

## 23. Stage 19: Live App Testing

**Model:** minimax-m2.7

Open the BGIPFS URL in a browser:
- [ ] Page loads without errors
- [ ] Contract data displays (TVL, balances)
- [ ] Wallet connects
- [ ] Network switching works
- [ ] No console errors

If issues found: file issues with label `deploy-app`, fix, rebuild, re-upload.

---

## 24. Stage 20: Live User Journey Walkthrough

Open the live app with a real wallet. Follow USERJOURNEY.md step by step.

- Actually click every button
- Actually connect your wallet
- Actually submit transactions (small amounts)
- At each step verify the UI matches USERJOURNEY.md

If ANY step fails: go back to the appropriate phase. Fix, redeploy, re-walk the ENTIRE journey until it works perfectly.

---

## 25. Stage 21: README & Delivery

### README.md Contents

- What the app does (2-3 sentences)
- Contract addresses on Base with Basescan links
- How to run locally
- Architecture decisions not obvious from code
- Client actions needed (acceptOwnership, keeper setup)

### README Must NOT Include

- SE2 template boilerplate
- Explanations of React, Solidity, Ethereum
- Padding or filler

### Delivery

1. Final git push
2. Call `completeJob(jobId, resultURL)` — `resultURL` = full IPFS gateway URL
3. Send live URL to client via `POST /api/job/{id}/messages` with type `bot_message`

---

## 26. Model Routing

| Task | Model | Why |
|------|-------|-----|
| Reading files, running commands, checking state | minimax-m2.7 | Cheap, fast |
| Writing PLAN.md, fixing errors, deployment | claude-sonnet-4.6 | Needs reasoning |
| Writing contracts, tests, frontend | claude-opus-4.6 | Needs deep domain knowledge |
| Running QA/audit checklists | claude-sonnet-4.6 | Needs judgment |
| Filing GitHub issues, git operations | minimax-m2.7 | Mechanical |

**Rules:**
- Any task that produces important code → opus
- Checking/verifying (ls, forge build) → minimax
- When using sonnet/opus: `skip_default_skills=true`, only pass needed skills
- Keep task descriptions SHORT for expensive models

### Token Limits per Step

| Step | Default | When to Bump |
|------|---------|-------------|
| Spec parsing | 4096 | — |
| Contract gen | 16384 | 32768 if truncated (3+ contracts) |
| Deploy scripts | 4096 | — |
| Tests | 32768 | Tests are verbose |
| Frontend gen | 16384 | 32768 for complex UIs |
| Fix loops | 16384 | — |

---

## 27. Skill Routing

| Task | Skills | Notes |
|------|--------|-------|
| Writing Solidity | `["ethskills"]` | |
| Scaffolding | `["scaffoldeth"]` | |
| Writing frontend | `[]` (no skills) | Preload AGENTS.md as a file instead |
| Writing PLAN.md | `["ethskills", "scaffoldeth"]` | |
| Deploying to IPFS | `["bgipfs"]` | |
| Deploying contracts | `["ethskills"]` | |
| Security audit | `["ethskills"]` + specialized audit skills | |
| Shell commands, git | `[]` | |

**NEVER pass `scaffoldeth` to opus/sonnet frontend sub-agents.** It tells them to run Steps 1-3 (scaffold, read AGENTS.md, read skill files) which wastes all iterations. Preload AGENTS.md as a file instead.

**`leftclaw` is orchestrator-only — never pass to sub-agents.**

---

## 28. File Preloading for Sub-Agents

Sub-agents waste iterations exploring the filesystem. Preload what they need:

| Task | Preload |
|------|---------|
| Contract writing | PLAN.md, foundry.toml, existing .sol files, DeployHelpers.s.sol |
| Frontend writing | AGENTS.md, scaffold.config.ts, page.tsx, the contract .sol file |
| Test fixing | Failing test file, contract file |
| Deployment | .env (root), foundry.toml, deploy script |
| Frontend build | scaffold.config.ts, deployedContracts.ts |

**Do NOT preload PLAN.md for frontend tasks.** The contract .sol has function signatures, AGENTS.md has hook API. Keep context small.

---

## 29. Retry & Escalation Rules

| Failure Count | Action |
|---------------|--------|
| 1st failure | Break task into smaller sub-tasks |
| 2nd failure (same goal) | Run diagnostic task first ("read the error") |
| 3rd failure | Include exact file content in task description, tell agent to write immediately |
| 3+ failures | Switch model or use heredoc approach. Never delegate same task >3 times. |

**Sub-agent iteration budgets:**
- Setup: 10
- Contracts (opus): 8
- Frontend (opus): 8
- Fixes: 10
- Deploy: 10
- Never below 5

---

## 30. Regression Protocol

| Bug Found In | Go Back To |
|-------------|-----------|
| Stage 19+ (live app) | Stage 12 (frontend) or Stage 5 (contracts) |
| Stage 17 (live contracts) | Stage 5 (contracts) |
| Stage 15 (integration audit) | Wherever root cause is |
| Stage 13 (frontend QA) | Stage 12 (frontend) |
| Stage 10 (contract audit) | Stage 5 (contracts) |
| Stage 3 (spec verification) | Stage 1 (architecture) |

When regressing on LeftClaw:
```
logWork(jobId, "Regression: audit found wstETH yield mechanism broken. See #12.", "prototype")
```

Always explain why. Always fix root cause, not symptom.

---

## 31. Solidity Rules

### Architecture
- Use OpenZeppelin. Check what's installed before writing custom implementations.
- `Ownable2Step` over `Ownable`
- `ReentrancyGuard` on all external-facing state-changing functions
- CEI pattern — state changes before external calls
- Custom errors over require strings
- `immutable`/`constant` where appropriate

### External Protocol Integration
- **Only call functions that exist in discovery.json.** If the ABI doesn't show `wrap()`, don't call it.
- On L2s, bridged tokens are plain ERC20s — no rich interfaces.
- For token conversions on L2, use Uniswap swaps, not native bridge calls.

### Deploy Scripts
- Always inherit `ScaffoldETHDeploy`, use `ScaffoldEthDeployerRunner` modifier
- Deploy contracts inline: `new MyContract(args)` inside `run()`
- Push to `deployments` array
- Wire cross-references after all deploys
- Deployer-first ownership: deploy → configure → transfer to client
- Zero-address checks in every constructor

### ERC4626 Vaults
- Virtual shares / `_decimalsOffset() >= 3`
- Underlying asset consistent (vault holds wstETH → asset IS wstETH)
- Override internal hooks `_deposit`/`_withdraw`, not public functions
- Principal tracking: `totalPrincipal` separate from `totalAssets()`
- Clamp principal to totalAssets() before yield calculations

### Approvals
- Never infinite (`type(uint256).max`). Approve exact or 3-5x.
- Use `forceApprove()` from SafeERC20 for tokens with non-zero-to-non-zero approval issues

---

## 32. Frontend Rules

### Imports
- SE2 hooks: `import { useScaffoldReadContract } from "~~/hooks/scaffold-eth"`
- UI components: `import { Address, AddressInput, Balance } from "@scaffold-ui/components"`
- Connect button: `import { RainbowKitCustomConnectButton } from "~~/components/scaffold-eth"`
- Use `~~` path alias always

### Components
- Every page: `"use client"` + `export default`
- Never raw wagmi hooks for contract interaction
- Never manually edit `deployedContracts.ts`
- External contracts → `externalContracts.ts`
- Contract addresses via `useDeployedContractInfo()`, never hardcoded

### Styling
- DaisyUI semantic classes: `btn btn-primary`, `bg-base-200`, `text-base-content`
- No hardcoded dark backgrounds
- `--radius-field: 0.5rem` in both theme blocks (fixes pill-shaped inputs)
- Tailwind v4: `@import "tailwindcss"` not v3 `@tailwind base;`

### UX Patterns
- Four-state button flow (Connect → Network → Approve → Action)
- Two-state approval locking (`approvalSubmitting` + `approveCooldown`)
- Button loading: inline `<span className="loading loading-spinner loading-sm" />`, NOT `className="loading"`
- USD values next to all amounts
- `<Address />` for all displayed addresses
- `<AddressInput />` for all address inputs
- `formatEther()`/`formatUnits()` for all amounts — never raw wei
- Every catch block → `notification.error()` with human-readable message
- Mobile deep linking: fire TX first, `setTimeout(openWallet, 2000)` after

### Error Handling

```typescript
import { getParsedError } from "~~/utils/scaffold-eth";
import { notification } from "~~/utils/scaffold-eth";

function getHumanError(e: unknown): string {
  const parsed = getParsedError(e);
  const map: [string, string][] = [
    ["SafeERC20FailedOperation", "Token transfer failed — check balance and approval"],
    ["InsufficientBalance", "Insufficient balance"],
    ["rejected", "Transaction rejected"],
  ];
  for (const [pat, msg] of map) {
    if (parsed.includes(pat)) return msg;
  }
  return parsed.includes("0x") ? "Transaction failed. Please try again." : parsed;
}
```

---

## 33. BGIPFS Deploy Rules

### Pre-Deploy

1. Remove `app/debug/` and `app/blockexplorer/` — incompatible with `output: "export"`
2. Set `NEXT_PUBLIC_IGNORE_BUILD_ERROR=true`
3. Verify `trailingSlash: true` in next.config.ts — IPFS gateways need it
4. Set `targetNetworks` to production chain (not foundry)

### Polyfill (Node 25+)

Create `packages/nextjs/polyfill-localstorage.cjs`:
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

**Must be in `packages/nextjs/`**, not project root.

### Build

```bash
cd packages/nextjs
rm -rf .next out
NEXT_PUBLIC_IPFS_BUILD=true \
NEXT_PUBLIC_IGNORE_BUILD_ERROR=true \
NODE_OPTIONS="--require ./polyfill-localstorage.cjs" \
npm run build
```

### Upload

```bash
bgipfs upload packages/nextjs/out --config ~/.bgipfs/credentials.json
```

### Validation

`yarn ipfs` exit code is NOT reliable — the script swallows errors. **The only valid signal is an IPFS CID in the output** (Qm... or bafy...). No CID = failed upload.

### OG Image Chicken-and-Egg

After first deploy, you know the CID. Rebuild with `NEXT_PUBLIC_PRODUCTION_URL=https://{CID}.ipfs.community.bgipfs.com`. The CID changes (new build), but old CID still works (IPFS = content-addressed).

### Common Failures

| Symptom | Cause | Fix |
|---------|-------|-----|
| `localStorage.getItem not a function` | Node 25+ | Add polyfill |
| Empty page on IPFS | Missing `trailingSlash: true` | Add to next.config.ts |
| Same CID after changes | Stale build cache | `rm -rf .next out` |
| "Page with dynamic" error | debug/blockexplorer pages | Delete them |
| Upload succeeds, no CID | bgipfs endpoint issue | Retry or check bgipfs status |

---

## 34. Security Checklist

### Solidity

- [ ] Token decimals handled correctly (USDC = 6, WETH = 18)
- [ ] Multiply before divide (no precision loss)
- [ ] No spot price oracle usage (use Chainlink or TWAP)
- [ ] No infinite approvals
- [ ] Reentrancy guards on external calls
- [ ] Access control on all state-changing functions
- [ ] Input validation (zero address, zero amount, bounds)
- [ ] Events emitted for all state changes
- [ ] CEI pattern followed
- [ ] SafeERC20 for all token transfers
- [ ] ERC4626: virtual shares for inflation protection
- [ ] TWAP: check BEFORE swap, not after
- [ ] pool.observe() wrapped in try/catch

### Frontend

- [ ] No private keys in code
- [ ] No API keys in frontend code (only public keys like RPC URLs)
- [ ] Input validation on addresses and amounts
- [ ] BigInt arithmetic for all token amounts (never floating point)

### Secrets

- All secrets in `.env` (gitignored)
- Pre-commit: `git diff --cached | grep -iE "apikey|secret|0x[a-fA-F0-9]{64}"`
- Never commit `.env`, `*.key`, `broadcast/`, `cache/`
- If accidentally committed: treat as compromised, rotate immediately

---

## 35. Verified Contract Addresses

Cross-reference EVERY address against this table. NEVER use an address you cannot verify.

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

## 36. SE2 Footguns

These are specific to Scaffold-ETH 2 and will bite you if you don't know about them.

| Footgun | Fix |
|---------|-----|
| `useScaffoldEventHistory` needs `deployedOnBlock` | Always include `deployedOnBlock` in `deployedContracts.ts` |
| `polyfill-localstorage.cjs` in wrong dir | Must be in `packages/nextjs/`, not project root |
| Build output in wrong dir | Upload `packages/nextjs/out/`, not `out/` at root |
| Block explorer crashes static export | Rename to `_blockexplorer-disabled` or delete |
| `deployedContracts.ts` manually edited | Never edit — auto-generated by `yarn deploy` |
| Default `pollingInterval: 30000` | Change to 3000 |
| `--radius-field: 9999rem` | Change to 0.5rem in both theme blocks |
| `appName: "scaffold-eth-2"` in wagmiConnectors | Change to your app name |
| `nativeCurrencyPrice` in Footer | Remove — renders ETH price badge on all networks |
| `bgipfs init` without `--endpoint` | Creates bad config pointing to localhost |
| `useWriteContract` instead of `useScaffoldWriteContract` | Scaffold hooks wait for confirmation; raw wagmi doesn't |
| Python f-strings for TS codegen | `{{ }}` produces literal braces. Use string concatenation. |
| Google Fonts via `<link>` tag | Use `next/font/google` in layout.tsx |

---

## 37. Common Failure Modes

Ordered by frequency from real builds.

### 1. Wrong Underlying Asset
**Symptom:** `availableYield()` always returns 0.
**Cause:** Vault uses WETH as underlying instead of wstETH. WETH doesn't appreciate.
**Prevention:** Stage 3 Spec Verification Gate checks underlying asset. Stage 9 Contract Verification Gate confirms `asset()` return value.
**Real occurrence:** Job #46 — complete rewrite required.

### 2. Hallucinated Interfaces
**Symptom:** Deploy reverts, `call to non-contract address`, or `function selector not found`.
**Cause:** LLM called `wstETH.wrap()` on Base where wstETH is a bridged ERC20 with no `wrap()`.
**Prevention:** Stage 2 Discovery fetches real ABIs. Code gen only uses discovered functions.

### 3. Hallucinated Addresses
**Symptom:** Transactions revert or tokens go to wrong contracts.
**Cause:** Agent guessed a contract address (often wrong length, wrong checksum, or from wrong chain).
**Prevention:** Every address verified with `cast call` in Stage 2. Cross-ref [Verified Addresses](#35-verified-contract-addresses).

### 4. Double nonReentrant
**Symptom:** `ReentrancyGuardReentrantCall` on deposit/withdraw.
**Cause:** Public function override adds `nonReentrant`, calls `super.deposit()` which also has it.
**Prevention:** Override `_deposit`/`_withdraw` (internal hooks). For convenience methods: `previewDeposit()` + `_mint()`.

### 5. transferFrom on Contract's Own Tokens
**Symptom:** `ERC20InsufficientAllowance` when depositing.
**Cause:** `super.deposit()` calls `transferFrom(msg.sender)` but tokens are already in the contract.
**Prevention:** For convenience deposits: `previewDeposit()` + `_mint()` directly.

### 6. Hardcoded Addresses in Frontend
**Symptom:** Frontend shows wrong contract or transactions fail after redeployment.
**Cause:** LLM copied addresses from `deployedContracts.ts` as string literals into components.
**Prevention:** Components use `useDeployedContractInfo()`. Page composes components, never passes addresses.

### 7. Zero Address in Deploy Script
**Symptom:** Deploy reverts with `ZeroAddress()`.
**Cause:** Deploy script has `address(0)` because LLM didn't know the real address.
**Prevention:** Cross-ref constructor args with Verified Addresses table.

### 8. Token Limit Truncation
**Symptom:** JSON parse error after LLM call. Raw output ends mid-code.
**Cause:** LLM output exceeds `max_tokens`.
**Prevention:** See Token Limits in [Model Routing](#26-model-routing). Bump when needed.

### 9. deployedContracts.ts Missing Fields
**Symptom:** TypeScript errors about `deployedOnBlock` or BigInt.
**Cause:** Missing `deployedOnBlock` field, or Python codegen f-string `{{ }}` produced literal braces.
**Prevention:** Always include `deployedOnBlock`. Use string concatenation for codegen, not f-strings.

### 10. walletDeepLink Ghost Import
**Symptom:** Build breaks after QA fix with "Module not found: walletDeepLink".
**Cause:** Fix LLM introduces import for non-existent file.
**Prevention:** After QA fixes, `git diff` to catch. Remove the import. Deep linking is nice-to-have, not ship-blocking.

### 11. Stale BGIPFS Upload
**Symptom:** Live app shows old content.
**Cause:** Didn't delete `.next` and `out` before rebuilding.
**Prevention:** Always `rm -rf .next out`. Verify CID changed.

### 12. TWAP Check After Swap
**Symptom:** Sandwich attack protection ineffective.
**Cause:** TWAP oracle check ran AFTER the swap instead of before.
**Prevention:** Audit checklist item. Code review in Stage 10.
**Real occurrence:** Job #46.

### 13. Deploy Script Transferred Ownership Too Early
**Symptom:** Post-deploy config fails — "not the owner."
**Cause:** Ownership transferred to client before configuring cross-references.
**Prevention:** Deployer-first pattern: deploy → configure → transfer. Stage 6 template enforces this.

### 14. Frontend "Please Connect" Text Instead of Button
**Symptom:** Users see a paragraph instead of a connect button.
**Cause:** Agent wrote `<p>Please connect your wallet</p>` instead of rendering `<RainbowKitCustomConnectButton />`.
**Prevention:** QA check #1 in Stage 13.

### 15. Subagent Reports "Done" Without Actually Doing It
**Symptom:** QA audit finds 15+ failures on items the subagent claimed were fixed.
**Cause:** Agent said "all SE2 branding removed" but didn't actually change the files.
**Prevention:** Every stage has concrete verification (grep, file read, build exit code). Never trust agent self-reports.
**Real occurrence:** Job #46.

---

## 38. QA Checklists

### Pre-Build (before writing code)

- [ ] Job spec read completely
- [ ] Client messages checked
- [ ] SPEC_REQUIREMENTS.md written with testable requirements
- [ ] PLAN.md has all 10 sections
- [ ] USERJOURNEY.md covers happy path + edge cases
- [ ] Discovery complete — all addresses verified
- [ ] Spec Verification Gate passed

### Pre-Deploy (before deploying to mainnet)

- [ ] `forge build` — zero errors
- [ ] `forge test` — all pass
- [ ] Contract audit complete, all findings fixed
- [ ] Frontend QA complete, all findings fixed
- [ ] Full integration audit passed
- [ ] No hardcoded secrets in any file
- [ ] scaffold.config.ts targets correct chain
- [ ] deployedContracts.ts ready for mainnet addresses
- [ ] Deployer has enough ETH for gas

### Pre-Ship (before marking job complete)

- [ ] Contracts deployed and verified on explorer
- [ ] Frontend deployed to BGIPFS
- [ ] Live app loads and displays contract data
- [ ] User journey walkthrough completed without failures
- [ ] README written with deployed addresses
- [ ] All GitHub issues closed
- [ ] Final git push completed
- [ ] BGIPFS URL tested and accessible

---

## 39. Automation Reference

### Verify-Fix Loop

The core auto-fix pattern used across build and test stages:

```
1. Run verify command (forge build, forge test, yarn next:build)
2. If exit code 0 → pass
3. If non-zero → gather error output + all source files
4. Send to fixer LLM with instructions:
   - Fix root cause, not symptom
   - If test fails, fix the test (not the contract)
   - Return complete file contents for changed files only
5. Write fixes back to project
6. Retry (max 3 attempts)
```

### Automated QA Checks

```bash
# SE2 branding still present?
grep -rn "scaffold-eth-2\|Scaffold-ETH 2\|BuidlGuidl" packages/nextjs/

# Dark backgrounds hardcoded?
grep -rn 'bg-\[#0\|bg-black\|bg-gray-9\|bg-zinc-9' packages/nextjs/app/

# Raw address inputs?
grep -rn 'type="text"' packages/nextjs/app/ | grep -i "addr\|0x"

# Missing error handling?
grep -rn "console.error\|console.log" packages/nextjs/app/ | grep -v node_modules

# Loading class on buttons?
grep -rn '"loading"' packages/nextjs/app/

# Raw wagmi?
grep -rn "useWriteContract\|useReadContract" packages/nextjs/ | grep -v scaffold-eth | grep -v node_modules

# OG image localhost?
grep -rn "localhost" packages/nextjs/utils/ packages/nextjs/app/layout.tsx

# Public RPC?
grep -n "mainnet.base.org\|base.llamarpc" packages/foundry/foundry.toml

# Leaked secrets?
git diff --cached | grep -iE "apikey|api_key|secret|password|0x[a-fA-F0-9]{64}"
```

### Shell Validation Rules

| Command | Pass Condition |
|---------|---------------|
| `create-eth` | `package.json` exists |
| `forge build` | Exit code 0 |
| `forge test` | Exit code 0, no FAIL in output |
| `yarn deploy` | Exit code 0, `deployedContracts.ts` has real address |
| `yarn next:build` | Exit code 0, no "Type error" or "Module not found" |
| `yarn verify` | Exit code 0 |
| `bgipfs upload` | CID in output (Qm... or bafy...) |

### Code Gen Validation Rules

| File Type | Pass Condition |
|-----------|---------------|
| `.sol` (contract) | Has `pragma solidity`, has contract/interface declaration |
| `.sol` (deploy) | Inherits `ScaffoldETHDeploy`, uses `ScaffoldEthDeployerRunner` |
| `.sol` (test) | Imports `forge-std/Test.sol`, extends `Test` |
| `.tsx` (component) | Has `export default`, no raw wagmi hooks |
| `deployedContracts.ts` | NEVER written by LLM — auto-generated only |

---

## Skill Reference URLs

| Skill | URL | When to Use |
|-------|-----|-------------|
| Orchestration | https://ethskills.com/orchestration/SKILL.md | Before planning phases |
| Security | https://ethskills.com/security/SKILL.md | Before writing contracts |
| Audit | https://ethskills.com/audit/SKILL.md | Contract audit stage |
| QA | https://ethskills.com/qa/SKILL.md | Frontend QA stage |
| Frontend Playbook | https://ethskills.com/frontend-playbook/SKILL.md | Before deploying frontend |
| Frontend UX | https://ethskills.com/frontend-ux/SKILL.md | Before building UI |
| Testing | https://ethskills.com/testing/SKILL.md | Before writing tests |
| BGIPFS | https://www.bgipfs.com/SKILL.md | IPFS deployment |
| SE2 Docs | https://docs.scaffoldeth.io/SKILL.md | SE2 patterns |
| SE2 AGENTS.md | In project root after scaffold | Hook names, components, style |
| LeftClaw Worker | https://leftclaw.services/admin/skill.md | Job management |
| LeftClaw Pipeline | https://leftclaw.services/admin/skill/build-pipeline | Build stages |
| OpenZeppelin | .agents/skills/openzeppelin/SKILL.md | OZ contract patterns |
