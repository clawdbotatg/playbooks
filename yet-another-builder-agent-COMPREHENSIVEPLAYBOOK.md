# Comprehensive dApp Building Playbook

From spec file to finished, deployed, audited decentralized application. Every step, every gate, every verification. No shortcuts. No trust.

This playbook synthesizes six production build pipelines, ten ethskills.com standards, Scaffold-ETH 2 conventions, and lessons from dozens of real builds into a single authoritative process.

---

## Philosophy

Three principles govern every decision:

1. **Build the thing, then PROVE the thing works.** Every stage that produces code has a verification gate that checks the code does what the spec requires — not just that it compiles. AI agents say "done" when they mean "it runs." Running is not done.

2. **Never trust, always verify.** Never trust an LLM-generated address. Never trust a subagent's "all done." Never trust that a build passes because the exit code was 0. Verify every claim with an independent check.

3. **Fix root causes, not symptoms.** If Stage 7 reveals the architecture is wrong, go back to Stage 3. Moving forward with a broken foundation creates exponentially more work. The pipeline has gates — use them.

**The cardinal rule: spec fidelity over pipeline completion.** A build that compiles, passes tests, and deploys is worthless if it doesn't implement what the spec says.

---

## Table of Contents

### Pipeline Stages
1. [Stage 0: Read the Spec](#stage-0-read-the-spec)
2. [Stage 1: Discovery](#stage-1-discovery)
3. [Stage 2: Architecture Plan](#stage-2-architecture-plan)
4. [Stage 3: Spec Verification Gate](#stage-3-spec-verification-gate)
5. [Stage 4: Scaffold](#stage-4-scaffold)
6. [Stage 5: Write Contracts](#stage-5-write-contracts)
7. [Stage 6: Write Deploy Scripts](#stage-6-write-deploy-scripts)
8. [Stage 7: Write Tests](#stage-7-write-tests)
9. [Stage 8: Compile and Test](#stage-8-compile-and-test)
10. [Stage 9: Contract Audit](#stage-9-contract-audit)
11. [Stage 10: Contract Audit Fixes](#stage-10-contract-audit-fixes)
12. [Stage 11: External Contracts](#stage-11-external-contracts)
13. [Stage 12: Deploy to Live Chain](#stage-12-deploy-to-live-chain)
14. [Stage 13: Frontend Development](#stage-13-frontend-development)
15. [Stage 14: Frontend QA Audit](#stage-14-frontend-qa-audit)
16. [Stage 15: Frontend QA Fixes](#stage-15-frontend-qa-fixes)
17. [Stage 16: Semantic Code Review](#stage-16-semantic-code-review)
18. [Stage 17: Full Integration Audit](#stage-17-full-integration-audit)
19. [Stage 18: Deploy Frontend to IPFS](#stage-18-deploy-frontend-to-ipfs)
20. [Stage 19: Live App Verification](#stage-19-live-app-verification)
21. [Stage 20: User Journey Walkthrough](#stage-20-user-journey-walkthrough)
22. [Stage 21: README and Delivery](#stage-21-readme-and-delivery)

### Reference
- [A: Pipeline Flow Diagram](#appendix-a-pipeline-flow-diagram)
- [B: Prerequisites](#appendix-b-prerequisites)
- [C: Verified Contract Addresses](#appendix-c-verified-contract-addresses)
- [D: Security Checklist](#appendix-d-security-checklist)
- [E: Frontend UX Rules](#appendix-e-frontend-ux-rules)
- [F: QA Checklist (Full)](#appendix-f-qa-checklist-full)
- [G: SE2 Footguns](#appendix-g-se2-footguns)
- [H: Common Failures from Real Builds](#appendix-h-common-failures-from-real-builds)
- [I: Model Routing](#appendix-i-model-routing)
- [J: Skill Routing](#appendix-j-skill-routing)
- [K: Token Limit Guide](#appendix-k-token-limit-guide)
- [L: Verification Commands](#appendix-l-verification-commands)
- [M: File Locations](#appendix-m-file-locations)
- [N: Import Paths](#appendix-n-import-paths)
- [O: Regression Protocol](#appendix-o-regression-protocol)

---

## Stage 0: Read the Spec

**What:** Read everything about the job before writing a single line of code.
**Produces:** Mental model + `SPEC_REQUIREMENTS.md`
**Model:** cheap (reading, no code gen)

### 0.1 Read the Job Description

Read the complete spec. Identify:
- What contracts are needed (names, functions, interactions)
- What external protocols are involved (Uniswap, Lido, Chainlink, etc.)
- What the frontend should look like and do
- Who the client is (their wallet address = owner of everything deployed)
- What chain to deploy on (default: Base, chain ID 8453)

### 0.2 Read All Messages (if applicable)

Clients add requirements, preferences, and scope changes via chat AFTER posting the job. The on-chain description is the baseline; messages override it.

**Lesson learned:** A client posted a "Windows 95 aesthetic" requirement via chat. The on-chain description said nothing about it. Skipping messages shipped a completely wrong frontend.

### 0.3 Onchain Litmus Test

Ask: "Does this actually need a blockchain?" Put it onchain ONLY if it requires trustless ownership, trustless exchange, composability, censorship resistance, or permanent commitments. Everything else stays offchain.

**Most MVPs need 0-2 smart contracts.** Three contracts is the upper bound for initial release.

### 0.4 State Transition Audit

For every function in the planned contracts:

| Function | Who calls it? | Why would they? | What if nobody calls it? | Gas incentive? |
|----------|--------------|-----------------|--------------------------|----------------|

Smart contracts cannot execute themselves. Every function needs a caller who pays gas. If the answer to "who calls it?" is "the team" — redesign with aligned incentives.

### 0.5 Write SPEC_REQUIREMENTS.md

List every requirement as a numbered, testable item:

```
1. The vault accepts ETH, WETH, and wstETH deposits
2. Vault shares (clawdETH) are ERC4626 compliant
3. Yield from wstETH appreciation is redirected to buy CLAWD
4. 50% of purchased CLAWD is burned, 50% goes to staking rewards
...
```

Each item must be verifiable — "the vault uses wstETH as underlying" not "the vault works with ETH staking."

### Verification Gate

- [ ] Every technical requirement is captured as a numbered, testable item
- [ ] All external contract addresses are identified
- [ ] Target chain is identified
- [ ] Client/owner address is identified (if applicable)

**Do NOT proceed until you can explain the full user flow, every contract interaction, and every external protocol integration.**

---

## Stage 1: Discovery

**What:** Verify what actually exists before generating any code. This prevents hallucinated interfaces.
**Produces:** `discovery.json` or equivalent ground truth
**Model:** cheap (running cast commands, reading files)

### 1.1 Verify External Contract Addresses

For every address in the spec, verify it exists on the target chain:

```bash
# Check contract exists
cast code <address> --rpc-url $ALCHEMY_RPC_URL
# Returns 0x if no contract exists

# Verify token identity
cast call <address> "symbol()(string)" --rpc-url $ALCHEMY_RPC_URL
cast call <address> "decimals()(uint8)" --rpc-url $ALCHEMY_RPC_URL

# Verify Uniswap pool exists
cast call <FACTORY> "getPool(address,address,uint24)(address)" <TOKEN_A> <TOKEN_B> <FEE> --rpc-url $ALCHEMY_RPC_URL
# Zero address = pool doesn't exist
```

Cross-reference every address against [Appendix C: Verified Contract Addresses](#appendix-c-verified-contract-addresses). If an address doesn't match a known verified address, verify manually. **LLMs hallucinate addresses.**

### 1.2 Discover On-Chain Interfaces

For every external contract, fetch the real ABI:

```bash
cast interface <address> --chain base
```

This returns the actual function signatures. Store the result.

**What this catches:** Contracts that exist on mainnet with rich interfaces (e.g., Lido wstETH with `wrap()`, `unwrap()`, `stETH()`) but are bridged on L2s as simple ERC20s. Without discovery, the LLM writes code against the mainnet interface and the deploy reverts.

### 1.3 Identify Protocol Behaviors

For each external protocol:
- Is the token standard ERC20? Any non-standard behaviors (rebasing, fee-on-transfer, blocklist)?
- What pool fee tiers exist? (Uniswap V3: 100, 500, 3000, 10000)
- Is the protocol available on the target chain?
- On L2s, bridged tokens are plain ERC20s — no native protocol functions

### 1.4 Chain Selection (if not specified)

| Need | Chain | Why |
|------|-------|-----|
| Consumer/social, cheapest gas | Base | Coinbase integration, Smart Wallet |
| Deepest DeFi liquidity | Arbitrum | GMX, Pendle, widest protocol coverage |
| High-value DeFi, governance | Mainnet | Canonical security, composability |
| MEV protection | Unichain | TEE block building |
| Mobile/real-world payments | Celo | MiniPay, sub-cent fees |

### Verification Gate

- [ ] Every external address verified on-chain with `cast code` and `cast call`
- [ ] On-chain interfaces fetched for every external protocol
- [ ] Protocol behaviors documented (bridged vs native, fee tiers, etc.)
- [ ] No hallucinated addresses remain

---

## Stage 2: Architecture Plan

**What:** Write the architecture document and user journey before writing any code.
**Produces:** `PLAN.md`, `USERJOURNEY.md`
**Model:** medium-expensive (planning requires reasoning)

### 2.1 Write PLAN.md

The plan MUST contain:

1. **One-sentence summary** of what the app does
2. **Architecture diagram** (text-based) showing contract relationships
3. **Contract specifications** for each contract:
   - Name and purpose
   - Inheritance chain (e.g., ERC4626, Ownable2Step, ReentrancyGuard)
   - State variables with types
   - Every public/external function with signature, access control, and purpose
   - Events and custom errors
   - Which external protocols it calls and how (exact function signatures from discovery)
4. **Yield/reward mechanism** (for DeFi):
   - Where does value come from?
   - How is it captured?
   - How is it converted?
   - How is it distributed?
5. **Deploy script** with exact constructor arguments, including all external addresses
6. **Post-deploy configuration** steps (setHarvester, setRewardDistributor, etc.)
7. **Frontend specification**:
   - What pages/tabs exist
   - What each section shows and does
   - What SE2 hooks to use for each interaction
8. **External addresses** — every address used, verified on-chain
9. **Security considerations** — attack vectors and mitigations

### 2.2 Write USERJOURNEY.md

Document what the user sees and does at every step:

**Happy path (step by step):**
1. User opens the app URL (no wallet connected — what do they see?)
2. User clicks Connect Wallet
3. User is on wrong network — Switch Network flow
4. User performs primary action (each deposit type, stake, etc.)
5. User sees the result — success state

**Edge cases (MUST cover all):**
- No wallet installed
- Wrong network connected
- Insufficient balance (gas AND token)
- Transaction rejected by user
- Transaction reverted on-chain
- Slow/pending transaction
- Multiple rapid clicks (double-submit prevention)
- Mobile wallet via WalletConnect

### Verification Gate

- [ ] PLAN.md has all 9 sections
- [ ] Every external address in PLAN.md was verified in Stage 1
- [ ] USERJOURNEY.md covers the happy path AND at least 4 edge cases
- [ ] Plan only references functions that exist in discovery (no imagined interfaces)
- [ ] Plan only references imports that exist in SE2 (no imagined packages)

---

## Stage 3: Spec Verification Gate

**What:** Before writing any code, verify the plan actually implements the spec. This catches architectural bugs before they become code bugs.
**Produces:** Annotated SPEC_REQUIREMENTS.md
**Model:** medium (needs judgment)

Go through `SPEC_REQUIREMENTS.md` line by line. For each requirement:

1. Find where in PLAN.md it is addressed
2. Verify the plan's approach actually satisfies the requirement
3. Mark: "VERIFIED — see PLAN.md section X" or "FAILED — plan says Y but spec says Z"

### DeFi-Specific Verification

- [ ] **Underlying asset is correct.** If the spec says "ETH staking yield," the vault MUST use wstETH/stETH as underlying (they appreciate), NOT WETH/ETH (which don't appreciate on their own)
- [ ] **Yield mechanism is real.** Walk through the yield calculation with numbers. If `availableYield()` would return 0 under normal conditions, the mechanism is broken
- [ ] **Swap path exists on-chain.** For every swap, verify the pool exists via factory's `getPool()`
- [ ] **External addresses are real.** Every address verified with `cast call`
- [ ] **Owner vs deployer is correct.** Client address gets ownership, not deployer
- [ ] **Token decimals are correct.** USDC = 6, WETH = 18, wstETH = 18

### Verification Gate

- [ ] Every line in SPEC_REQUIREMENTS.md has a VERIFIED or FAILED annotation
- [ ] Zero FAILED items remain (all fixed in PLAN.md)
- [ ] If ANY requirement failed, PLAN.md was updated before proceeding

**If ANY requirement cannot be verified: go back to Stage 2. Do NOT proceed to code.**

---

## Stage 4: Scaffold

**What:** Create the SE2 project skeleton.
**Produces:** Empty SE2 project with dependencies installed
**Model:** cheap (shell commands only)

### 4.1 Scaffold

```bash
npx -y create-eth@latest -s foundry <project-name>
cd <project-name> && yarn install
```

Use Foundry flavor (default). Use kebab-case naming.

### 4.2 Read AGENTS.md

After scaffolding, read `AGENTS.md` in the project root. It contains the authoritative reference for hooks, components, conventions, and code style. Do not proceed until you've read it.

### 4.3 Read Relevant Skill Files

Based on what the spec requires, read skills from `.agents/skills/<name>/SKILL.md`:

| Building with... | Read skill |
|-----------------|------------|
| Any smart contract | `security`, `testing` |
| OpenZeppelin contracts | `openzeppelin` |
| ERC-721 / NFTs | `erc-721` |
| Token approvals / DeFi | `frontend-ux` |
| External contract interaction | `addresses` |
| Sign-in with Ethereum | `siwe` |

### Verification Gate

- [ ] `packages/foundry/contracts/YourContract.sol` exists
- [ ] `packages/nextjs/scaffold.config.ts` exists
- [ ] `forge build` succeeds in `packages/foundry`
- [ ] AGENTS.md has been read

---

## Stage 5: Write Contracts

**What:** Implement all Solidity contracts.
**Produces:** `.sol` files in `packages/foundry/contracts/`
**Model:** expensive (Solidity correctness is critical)

### 5.1 Delete Scaffold Defaults

```bash
rm packages/foundry/contracts/YourContract.sol
```

### 5.2 Write Contracts

One contract per file. For each:
1. Start with inheritance chain and state variables
2. Write constructor
3. Write view functions
4. Write state-changing functions
5. Write admin functions
6. Add events and custom errors

### Mandatory Patterns

**Security (from ethskills/security):**
- `Ownable2Step` over `Ownable` — prevents accidental ownership transfer
- `ReentrancyGuard` on every function that makes external calls
- `SafeERC20` for all token operations (`safeTransfer`, `safeTransferFrom`, `forceApprove`)
- CEI pattern (Checks-Effects-Interactions) — state changes before external calls
- Custom errors over require strings — `error ZeroAddress();` not `require(addr != address(0), "zero")`
- Emit events for every state change
- No infinite approvals — approve exact amounts or 3-5x
- No `tx.origin`
- Constructor validates all inputs (zero address checks)

**ERC4626 Vaults:**
- `_decimalsOffset()` >= 3 for inflation attack prevention (virtual shares)
- Track principal separately from yield if yield is redirected
- Override `_deposit`/`_withdraw` (internal hooks), NOT public functions (double nonReentrant)
- For convenience deposit functions (depositETH), use `previewDeposit()` + `_mint()` directly — don't call `super.deposit()` which does `transferFrom(msg.sender, ...)`

**Uniswap V3 Swaps:**
- Always set `amountOutMinimum` > 0 (slippage protection)
- Always set `deadline` (block.timestamp for on-chain calls)
- TWAP oracle check BEFORE the swap, not after
- Wrap `pool.observe()` in try/catch (new pools may lack observation history)

**Access Control:**
- Owner = client address (not deployer, not your address)
- Walkaway test: if the owner disappears, can users still withdraw? The answer must be yes

**External Integration:**
- Only call functions that exist in discovery (Stage 1)
- On L2s, bridged tokens are plain ERC20s — no native protocol functions
- For token conversions on L2, use Uniswap swaps, not native protocol functions

### 5.3 Compile

```bash
cd packages/foundry && forge build
```

### Verification Gate

- [ ] Every contract from PLAN.md has a corresponding `.sol` file
- [ ] `forge build` exits 0 with zero errors
- [ ] No stub functions or `// TODO` comments
- [ ] All contracts have ReentrancyGuard, Ownable2Step, SafeERC20

---

## Stage 6: Write Deploy Scripts

**What:** Create Foundry deploy scripts following the SE2 pattern.
**Produces:** `Deploy*.s.sol` files in `packages/foundry/script/`
**Model:** medium (templated pattern)

### 6.1 Delete Default

```bash
rm packages/foundry/script/DeployYourContract.s.sol
rm packages/foundry/test/YourContract.t.sol
```

### 6.2 Write Deploy Scripts

Use `ScaffoldETHDeploy` base contract with `ScaffoldEthDeployerRunner` modifier:

```solidity
contract DeployMyContract is ScaffoldETHDeploy {
    function run() external ScaffoldEthDeployerRunner {
        MyContract c = new MyContract(arg1, arg2);
        // Push to deployments array for ABI generation
    }
}
```

**Constructor arguments:**
- External addresses: hardcode the verified addresses for the target chain (this is the one place hardcoded addresses are acceptable)
- Owner: use deployer-first pattern — deploy with deployer as owner, configure cross-references, THEN transfer ownership to client
- If contract A needs B's address and vice versa: deploy A, deploy B(address(A)), then A.setB(address(B))
- Zero-address checks: `if (_addr == address(0)) revert ZeroAddress();`

### 6.3 Compile

```bash
cd packages/foundry && forge build --skip test
```

### Verification Gate

- [ ] `forge build --skip test` passes
- [ ] Deploy.s.sol exists and imports all individual deploy scripts
- [ ] Constructor arguments match the contract constructors
- [ ] External addresses match [verified addresses](#appendix-c-verified-contract-addresses)
- [ ] Deployment order handles dependencies (deploy token before vault that needs token address)

---

## Stage 7: Write Tests

**What:** Generate Foundry tests covering all contract functionality.
**Produces:** Test files in `packages/foundry/test/`
**Model:** expensive (needs understanding of contract semantics)

### Test Priority (from ethskills/testing)

1. **Unit tests** — edge cases, failure modes, access control. NOT getters.
2. **Fuzz tests** — any function with math. Minimum 1000 runs. Use `bound()` not `vm.assume()`.
3. **Fork tests** — any interaction with external protocols. Fork the target chain.
4. **Invariant tests** — stateful protocols. Properties that must always hold.

### What to Test

- Constructor state (all immutables set correctly)
- Happy path for every public/external function
- Revert cases: unauthorized callers, invalid inputs, insufficient balances
- Edge cases: zero amounts, max amounts, boundary conditions
- Full lifecycle: deploy -> interact -> verify state
- For DeFi: full harvest cycle (yield appears -> harvester pulls -> swap -> burn/distribute -> claim)

### Mock Pattern

For contracts that interact with external ERC20 tokens:

```solidity
contract MockERC20 is ERC20 {
    constructor(string memory name, string memory symbol) ERC20(name, symbol) {}
    function mint(address to, uint256 amount) external { _mint(to, amount); }
}
```

### Verification Gate

- [ ] At least one test file per contract
- [ ] Tests cover happy path, reverts, and edge cases

**Tests are NOT verified here — that happens in Stage 8.**

---

## Stage 8: Compile and Test

**What:** Build everything and run all tests. Auto-fix on failure.
**Produces:** Passing build + passing tests
**Model:** expensive (for fix attempts)

### 8.1 Compile

```bash
cd packages/foundry && forge build
```

If compilation fails, diagnose the error and fix. Up to 3 attempts.

### 8.2 Test

```bash
cd packages/foundry && forge test -vv
```

If tests fail, diagnose:
- If a test assertion fails, **the test expectation is probably wrong** — fix the test, not the contract
- If a test reverts unexpectedly, check the contract logic
- Up to 3 fix attempts per phase

### Verification Gate

- [ ] `forge build` exits 0 (all contracts, tests, scripts compile)
- [ ] `forge test -vv` exits 0 (all tests pass)
- [ ] No test was removed to make the suite pass

### Escalation

If the fix loop exhausts retries:
1. Try a smarter model (switch from sonnet to opus)
2. Read the error output and fix manually
3. If tests are fundamentally wrong, delete and regenerate from Stage 7

---

## Stage 9: Contract Audit

**What:** Systematic security audit. File findings. Do NOT fix anything.
**Produces:** Audit report with GitHub issues
**Model:** medium (needs judgment)
**Reference:** https://ethskills.com/audit/SKILL.md

### 9.1 Standard Audit

Read every contract file. Check:

**Critical checks:**
1. Reentrancy — guards on all external-calling functions
2. Access control — Ownable2Step correct, no leftover deployer privileges
3. Integer safety — overflow/underflow, unsafe casting, multiply before divide
4. External call safety — return values checked, CEI pattern followed
5. Token handling — SafeERC20, no raw transfer()
6. Slippage protection — all swaps have minimum output > 0
7. Oracle safety — TWAP checks, manipulation resistance
8. First depositor attacks — virtual shares for ERC4626
9. Token decimals — USDC is 6, not 18; handled correctly throughout

**DeFi-specific checks (if applicable):**
10. Flash loan vectors — can someone manipulate state in a single block?
11. Sandwich attack vectors — are swaps protected with TWAP?
12. Yield accounting — can principal and yield drift?
13. Walkaway safety — can users always withdraw?
14. Infinite approvals — none allowed (approve exact or 3-5x)

See [Appendix D: Security Checklist](#appendix-d-security-checklist) for the complete checklist.

### 9.2 Deep Audit (for complex contracts)

**SKIP if simple:** <100 lines, no swaps, no reentrancy risk, basic storage.

**DO if complex:** Token swaps, multi-contract interactions, financial logic, >200 lines.

The deep audit uses specialist checklists from ethskills.com/audit:
- `evm-audit-general` (cross-cutting)
- `evm-audit-precision-math` (math vulnerabilities)
- `evm-audit-erc20` (token interactions)
- `evm-audit-erc4626` (vault-specific, 42+ items)
- `evm-audit-defi-staking` (staking mechanisms)
- `evm-audit-oracles` (if using TWAP/price feeds)
- `evm-audit-defi-amm` (if doing swaps)

### 9.3 File Issues

For every Medium+ finding, create a GitHub issue with:
```
## [SEVERITY] Finding title
**Location:** file:function
**Description:** What's wrong and why it matters
**Recommendation:** Concrete fix with code
```

### Verification Gate

- [ ] Every contract has been reviewed
- [ ] All findings are filed as GitHub issues
- [ ] Findings include severity, location, description, and recommendation

**Do NOT fix anything in this stage. Report only.**

---

## Stage 10: Contract Audit Fixes

**What:** Fix every finding from the audit.
**Produces:** Fixed contracts, closed issues, passing tests
**Model:** expensive (contract code changes)

### 10.1 Fix by Severity

1. **Critical** — must fix, no exceptions
2. **High** — must fix
3. **Medium** — fix or document as accepted with clear reasoning
4. **Low** — fix if trivial, otherwise document

### 10.2 Recompile and Retest

```bash
cd packages/foundry && forge build && forge test -vv
```

### 10.3 Close Issues

Each issue closed with a commit reference.

### Verification Gate

- [ ] Zero Critical/High findings remain open
- [ ] `forge build` exits 0
- [ ] `forge test` still passes after all fixes
- [ ] Every issue has a commit reference

---

## Stage 11: External Contracts

**What:** Register external contract ABIs for the frontend.
**Produces:** `packages/nextjs/contracts/externalContracts.ts`
**Model:** medium

### 11.1 Write externalContracts.ts

For every token/protocol the frontend reads from:

```typescript
import { GenericContractsDeclaration } from "~~/utils/scaffold-eth/contract";

const externalContracts = {
  8453: {  // Chain ID as NUMBER key
    WETH: {
      address: "0x4200000000000000000000000000000000000006",
      deployedOnBlock: 0,  // REQUIRED by SE2 type system
      abi: [
        // Only include functions the frontend calls
      ],
    },
  },
} as const;

export default externalContracts satisfies GenericContractsDeclaration;
```

**Rules:**
- Chain ID MUST be a number key (not string)
- EVERY contract MUST include `deployedOnBlock: 0`
- For ERC20 tokens: include name, symbol, decimals, totalSupply, balanceOf, transfer, transferFrom, approve, allowance, plus Transfer and Approval events
- Only include ABI entries the frontend actually calls
- Never manually edit `deployedContracts.ts` — it's auto-generated

### Verification Gate

- [ ] Every external address from the spec is in the file
- [ ] `GenericContractsDeclaration` import exists
- [ ] `yarn next:build` passes (type check)

---

## Stage 12: Deploy to Live Chain

**What:** Deploy contracts to the target network and verify on block explorer.
**Produces:** Live contracts, verified source, updated `deployedContracts.ts`
**Model:** cheap-medium (shell commands + config)

### CRITICAL: Deploy BEFORE Frontend

Stage 12 MUST run before Stage 13. The frontend needs `deployedContracts.ts` with real deployed addresses. Without it, the LLM will hardcode addresses or use placeholders — the #1 bug class.

### 12.1 Configure RPC

In `packages/foundry/foundry.toml`:
```toml
[rpc_endpoints]
base = "https://base-mainnet.g.alchemy.com/v2/${ALCHEMY_API_KEY}"
```

**NEVER use public RPCs.** Always Alchemy with API key. If `ALCHEMY_API_KEY` is not available, STOP.

### 12.2 Fund the Deployer

```bash
yarn generate    # Generate deployer account (if needed)
yarn account     # View deployer address and balance
```

Fund the deployer with ETH on the target chain.

### 12.3 Deploy

```bash
yarn deploy --network base
```

### 12.4 Verify on Block Explorer

```bash
yarn verify --network base
```

Every contract MUST show verified source with a green checkmark. Unverified contracts are a trust red flag.

### 12.5 Verify On-Chain State

```bash
# Check ownership
cast call <address> "owner()(address)" --rpc-url $ALCHEMY_RPC_URL

# Check cross-references
cast call <vault> "harvester()(address)" --rpc-url $ALCHEMY_RPC_URL

# Verify underlying asset
cast call <vault> "asset()(address)" --rpc-url $ALCHEMY_RPC_URL
```

### 12.6 Post-Deploy Configuration

If contracts need cross-references (vault.setHarvester, rewards.setRewardDistributor):
1. Make the calls as deployer (temporary owner)
2. Transfer ownership to client
3. Log the exact calls the client needs to make (e.g., `acceptOwnership()`)

### Verification Gate

- [ ] All contracts deployed (addresses recorded)
- [ ] All contracts verified on block explorer (green checkmark)
- [ ] `deployedContracts.ts` updated with real chain ID and addresses
- [ ] Cross-references configured correctly
- [ ] On-chain state matches plan (ownership, parameters, etc.)
- [ ] No constructor argument was zero address (`0x000...0`)

---

## Stage 13: Frontend Development

**What:** Build the complete frontend using SE2 hooks and DaisyUI.
**Produces:** Working frontend that builds (`yarn next:build` passes)
**Model:** expensive (React + SE2 patterns + contract integration)
**Reference:** https://ethskills.com/frontend-ux/SKILL.md

### 13.1 Configuration (Deterministic — Do First)

**scaffold.config.ts:**
```typescript
targetNetworks: [chains.base],  // Match target chain
pollingInterval: 3000,          // NOT 30000 (30s lag)
```

**wagmiConnectors.tsx:**
- Change `appName: "scaffold-eth-2"` to your app name
- Add `phantomWallet` to the wallets array

### 13.2 SE2 Branding Cleanup

This is NOT optional. Every item must be addressed:

- [ ] **Footer.tsx** — Remove BuidlGuidl links, "Fork me", "Support," `nativeCurrencyPrice` badge
- [ ] **Header.tsx** — Replace SE2 logo/name with project name; remove "Debug Contracts" nav
- [ ] **layout.tsx / getMetadata.ts** — Change title, description; remove "Scaffold-ETH 2"
- [ ] **README.md** — Replace entirely with project content
- [ ] **Favicon** — Replace `public/favicon.ico` or add favicon.svg
- [ ] **manifest.json** — Change "Scaffold-ETH 2 DApp" to your app name
- [ ] **Block explorer** — Remove or disable `app/blockexplorer/` (crashes static export)

### 13.3 Styling

**globals.css — DaisyUI theme:**
- Change `--radius-field: 9999rem` to `--radius-field: 0.5rem` in BOTH theme blocks
- Custom colors go in the theme, not inline

**Dark mode rule:**
- NEVER hardcode dark backgrounds (`bg-[#0a0a0a]`, `bg-black`, `bg-zinc-900`)
- Use DaisyUI semantic variables: `bg-base-100`, `bg-base-200`, `text-base-content`
- Exception: if intentionally dark-only, force `data-theme="dark"` on `<html>` AND remove `<SwitchTheme/>`

### 13.4 SE2 Hooks (MANDATORY — Never Raw Wagmi)

```typescript
// CORRECT
import { useScaffoldReadContract, useScaffoldWriteContract } from "~~/hooks/scaffold-eth";
import { useDeployedContractInfo } from "~~/hooks/scaffold-eth";

// WRONG — never use these directly
import { useReadContract } from "wagmi";
import { useWriteContract } from "wagmi";
```

Verify: `grep -rn "useWriteContract\|useReadContract" packages/nextjs/ | grep -v scaffold-eth | grep -v node_modules` — any match = bug.

### 13.5 SE2 Components

| Need | Use | Never Use |
|------|-----|-----------|
| Display address | `<Address />` from `@scaffold-ui/components` | Raw truncated hex |
| Input address | `<AddressInput />` from `@scaffold-ui/components` | `<input type="text">` |
| Display balance | `<Balance />` | Raw wei number |
| Input ETH | `<EtherInput />` | `<input type="number">` |
| Connect wallet | `<RainbowKitCustomConnectButton />` | Custom button |

### 13.6 Contract Address Resolution (THE #1 BUG CLASS)

**NEVER** hardcode a deployed contract address as a string literal in any React file.
**NEVER** pass a deployed contract address as a prop from page.tsx to a child component.

Components that need a deployed address MUST call `useDeployedContractInfo("ContractName")` internally:

```typescript
// INSIDE the component that needs it
const { data: vaultInfo } = useDeployedContractInfo("ClawdETHVault");
const vaultAddress = vaultInfo?.address;
```

`deployedContracts.ts` is provided for REFERENCE — addresses must NOT be copied into components.

### 13.7 Four-State Button Flow (MANDATORY)

Show exactly ONE primary button at a time:

```
1. Not connected  → Connect Wallet (RainbowKitCustomConnectButton)
2. Wrong network  → Switch to [Chain]
3. Needs approval → Approve button
4. Ready          → Action button
```

**Never show Approve and Action buttons simultaneously.**

### 13.8 Approval Pattern with Two-State Locking

`isPending` from wagmi drops to `false` when the wallet returns a tx hash — NOT when the tx confirms. This creates a window for double-submit.

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
  } catch (e) {
    notification.error(getHumanError(e));
  } finally {
    setApprovalSubmitting(false);  // MUST be in finally (handles rejection)
  }
};

<button disabled={isPending || approvalSubmitting || approveCooldown}>
  {(isPending || approvalSubmitting) && <span className="loading loading-spinner loading-sm mr-2" />}
  {isPending || approvalSubmitting ? "Approving..." : "Approve"}
</button>
```

**Why `finally`:** If user rejects the tx, `approvalSubmitting` must clear. Without `finally`, a rejected tx locks the button permanently.

**Why `approveCooldown`:** After `writeContractAsync` resolves, the node's cache may not reflect the new allowance. The 4-second cooldown + refetch ensures the UI reads updated state.

### 13.9 Button Loading States

```tsx
// WRONG — DaisyUI "loading" class replaces content with full-width spinner
<button className={`btn ${isPending ? "loading" : ""}`}>Approve</button>

// RIGHT — inline spinner, text stays visible
<button className="btn btn-primary" disabled={isPending}>
  {isPending && <span className="loading loading-spinner loading-sm mr-2" />}
  {isPending ? "Approving..." : "Approve"}
</button>
```

### 13.10 Error Handling (MANDATORY)

Every `writeContractAsync` catch block MUST show a human-readable notification. Never `console.error` alone. Never show raw hex selectors.

```typescript
import { getParsedError } from "~~/utils/scaffold-eth";
import { notification } from "~~/utils/scaffold-eth";

function getHumanError(e: unknown): string {
  const parsed = getParsedError(e);
  const ERROR_MESSAGES: [string, string][] = [
    ["SafeERC20FailedOperation", "Token transfer failed — check balance and approval"],
    ["InsufficientBalance", "Insufficient balance"],
    // Map ALL contract errors from forge inspect
  ];
  for (const [pattern, message] of ERROR_MESSAGES) {
    if (parsed.includes(pattern)) return message;
  }
  if (parsed.includes("Encoded error signature") || parsed.includes("0x")) {
    return "Transaction failed. Please try again.";
  }
  return parsed;
}
```

### 13.11 Display Standards

- Show USD values next to ALL token/ETH amounts where possible
- Format with `formatEther`/`formatUnits` — never show raw wei
- Display deployed contract address using `<Address />`
- Every `"use client"` directive at top of interactive files

### 13.12 Build

```bash
yarn next:build
```

### Verification Gate

- [ ] `yarn next:build` exits 0
- [ ] All generated components are imported and used (no dead code)
- [ ] No raw wagmi hooks in app/components code
- [ ] No hardcoded contract addresses in React files
- [ ] scaffold.config.ts targets the correct chain
- [ ] SE2 branding fully removed (run verification commands from [Appendix L](#appendix-l-verification-commands))

---

## Stage 14: Frontend QA Audit

**What:** Run the complete QA checklist. Report only — do not fix.
**Produces:** PASS/FAIL checklist, GitHub issues for failures
**Model:** medium
**Reference:** https://ethskills.com/qa/SKILL.md

See [Appendix F: QA Checklist](#appendix-f-qa-checklist-full) for the complete 25-item checklist.

### Ship-Blocking (ALL must pass)

| # | Check |
|---|-------|
| 1 | Wallet connection shows a BUTTON, not text |
| 2 | Wrong network shows a Switch button |
| 3 | One button at a time (Connect -> Network -> Approve -> Action) |
| 4 | Approve button locked with `approvalSubmitting` + `approveCooldown` (both states) |
| 5 | Contracts verified on block explorer |
| 6 | SE2 footer branding removed |
| 7 | SE2 tab title removed |
| 8 | No zero-address placeholder (`0x000...0`) in frontend code |
| 9 | No hardcoded deployed addresses in components |
| 10 | Uses `useScaffoldWriteContract` (not raw wagmi) |

### Should-Fix

| # | Check |
|---|-------|
| 11 | Contract address displayed with `<Address/>` |
| 12 | Address inputs use `<AddressInput/>` |
| 13 | OG image is absolute URL |
| 14 | pollingInterval is 3000 |
| 15 | Favicon updated |
| 16 | `--radius-field` not 9999rem |
| 17 | No hardcoded dark backgrounds |
| 18 | Button loaders use inline spinner |
| 19 | Error messages human-readable (not raw hex) |
| 20 | Phantom wallet in RainbowKit |
| 21 | SE2 README replaced |
| 22 | Approve cooldown after TX |
| 23 | Mobile deep linking for wallet |
| 24 | appName changed in wagmiConnectors |
| 25 | Block explorer disabled/removed |

### Automated Checks

Run the commands from [Appendix L: Verification Commands](#appendix-l-verification-commands) for each item.

### Verification Gate

- [ ] Every item checked and marked PASS or FAIL
- [ ] GitHub issues filed for all FAILs

**Do NOT fix anything. Report only.**

---

## Stage 15: Frontend QA Fixes

**What:** Fix every FAIL from Stage 14. Rebuild.
**Produces:** Fixed frontend, closed issues, passing build
**Model:** expensive

### 15.1 Fix Each Issue

For each open issue, fix the code and close with commit reference.

### 15.2 KNOWN ISSUES — Watch For These

**walletDeepLink ghost import:** QA fixers sometimes introduce `import { writeAndOpen } from "~~/utils/scaffold-eth/walletDeepLink"` — a file that does NOT exist. If the build breaks after fixes, remove this import. Replace `writeAndOpen(() => writeFn({...}), connector?.id)` with just `writeFn({...})`.

**Unrelated file overwrites:** QA fixers can rewrite files beyond what's needed (block explorer pages, scaffold-eth internals). If the build breaks, `git diff` to see what changed and `git checkout --` unrelated files.

### 15.3 Rebuild

```bash
yarn next:build
```

### Verification Gate

- [ ] `yarn next:build` exits 0
- [ ] ALL ship-blocker items now PASS
- [ ] ALL should-fix items now PASS (or documented as accepted)
- [ ] No new imports of non-existent files

---

## Stage 16: Semantic Code Review

**What:** LLM semantic review of code against the spec. Catches what grep cannot.
**Produces:** Review report with fixes
**Model:** expensive (needs strong reasoning)

Unlike Stage 14 (pattern-based grep checks), this stage reasons about the CODE against the SPEC:

| Issue Type | Example |
|-----------|---------|
| Wrong abstraction | Address passed as prop instead of using `useDeployedContractInfo` internally |
| Dead code | Component imported but never rendered |
| Silent errors | catch block with `console.error` instead of `notification.error` |
| Spec drift | Frontend behavior contradicts the job spec |
| Wrong hook patterns | Raw wagmi instead of scaffold-eth hooks |

### Review Checklist

- [ ] Every component that needs a contract address resolves it via `useDeployedContractInfo` internally
- [ ] Every write call has a catch block with `notification.error` and human-readable message
- [ ] Every component in `components/` is imported and used somewhere
- [ ] Frontend behavior matches the spec (USERJOURNEY.md)
- [ ] No `any` types unless absolutely unavoidable
- [ ] Components do ONE thing — no god components

### Verification Gate

- [ ] Zero critical issues remain
- [ ] Build still passes after fixes

---

## Stage 17: Full Integration Audit

**What:** One final pass on everything — contracts AND frontend together.
**Produces:** Final audit report
**Model:** medium

### Safety Check
- [ ] Users can always withdraw (no lockups, no admin freeze)
- [ ] Owner cannot steal user deposits
- [ ] Reentrancy guards on all entry points
- [ ] SafeERC20 used throughout
- [ ] All privileged roles set to client address

### Frontend-Contract Integration
- [ ] Frontend passes correct args to every contract function (ABI match)
- [ ] Slippage parameters are non-zero in frontend calls
- [ ] All contract addresses in frontend match deployed addresses
- [ ] Event names in `useScaffoldEventHistory` match contract events
- [ ] Every requirement in SPEC_REQUIREMENTS.md is implemented and working

### Build Verification
```bash
cd packages/foundry && forge build      # contracts still compile
yarn next:build                          # frontend still builds
```

### Secrets Check
- [ ] No private keys, API keys, or mnemonics in any committed file
- [ ] `.gitignore` includes `.env`, `node_modules`, `out/`, `cache/`, `broadcast/`
- [ ] `grep -rn "0x[a-fA-F0-9]{64}" packages/` returns no private keys

### Verification Gate

- [ ] All items pass
- [ ] Both `forge build` and `yarn next:build` succeed

---

## Stage 18: Deploy Frontend to IPFS

**What:** Build static frontend and upload to IPFS.
**Produces:** Live URL at `https://<CID>.ipfs.community.bgipfs.com/`
**Model:** cheap (shell commands)

### 18.1 Create Polyfill (Node 25+ Compatibility)

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

### 18.2 Pre-Deploy Checklist

- [ ] `scaffold.config.ts` points at production network (not foundry/localhost)
- [ ] `next.config.ts` has `output: "export"` and `trailingSlash: true` when `NEXT_PUBLIC_IPFS_BUILD=true`
- [ ] `app/debug/` and `app/blockexplorer/` removed (incompatible with static export)
- [ ] OG image uses absolute URL (set `NEXT_PUBLIC_PRODUCTION_URL`)

### 18.3 Build

```bash
cd packages/nextjs
rm -rf .next out    # Clean previous builds
NEXT_PUBLIC_IPFS_BUILD=true \
  NODE_OPTIONS="--require ./polyfill-localstorage.cjs" \
  npm run build
```

### 18.4 Upload

```bash
# Via bgipfs API
curl -s -X POST "https://upload.bgipfs.com/api/v0/add?wrap-with-directory=true&pin=true" \
  -H "X-API-Key: $BGIPFS_API_KEY" \
  -F "file=@index.html;filename=index.html" ...

# Or via bgipfs CLI
npx bgipfs upload packages/nextjs/out --endpoint https://upload.bgipfs.com
```

**WARNING:** `yarn ipfs` / `npx bgipfs` exit code is NOT reliable. The script swallows errors. The only valid signal is an IPFS CID in the output (starts with `Qm` or `bafy`).

### 18.5 Convert CID

```bash
# Convert CIDv0 (Qm...) to CIDv1 (bafy...) for subdomain gateway
CID=$(npx -y cid-tool base32 "$ROOT_HASH")
LIVE_URL="https://${CID}.ipfs.community.bgipfs.com/"
```

### Verification Gate

- [ ] `out/` directory exists with files
- [ ] CID was returned from upload
- [ ] `curl -s "$LIVE_URL" | head` returns HTML (not error page)
- [ ] CID changed from previous deploy (if applicable)

---

## Stage 19: Live App Verification

**What:** Verify the deployed app works end-to-end.
**Produces:** Verification report
**Model:** cheap

### 19.1 Basic Checks

```bash
# HTML renders
curl -s "$LIVE_URL" | head -20

# Title is correct (not "Scaffold-ETH 2")
curl -s "$LIVE_URL" | grep -i "<title>"

# OG metadata points to absolute URL (not localhost)
curl -s "$LIVE_URL" | grep -i "og:image"
```

### 19.2 Browser Testing

- [ ] Page loads without errors
- [ ] Browser console shows no errors
- [ ] Contract data loads (TVL, balances, etc.)
- [ ] Wallet connection works
- [ ] Network switching works (if applicable)

### 19.3 If Issues Found

Go back to Stage 13 (frontend). Fix locally, rebuild, re-upload to IPFS. Verify the CID changed.

See [Appendix O: Regression Protocol](#appendix-o-regression-protocol).

---

## Stage 20: User Journey Walkthrough

**What:** Walk through USERJOURNEY.md step by step on the live app as a real user.
**Produces:** Pass/fail for each step
**Model:** cheap or human

### Actions

1. Open the live app (IPFS URL)
2. Follow EVERY step in USERJOURNEY.md
3. At each step verify:
   - The UI matches what USERJOURNEY.md describes
   - Transactions succeed (or fail gracefully with readable errors)
   - State updates correctly after each transaction

### If ANY Step Fails

Go back to the appropriate earlier stage. Fix. Redeploy. Re-walk the ENTIRE journey.

**Never skip broken steps. Never ship with known broken flows.**

---

## Stage 21: README and Delivery

**What:** Write README and deliver the final product.
**Produces:** Complete, shipped dApp

### 21.1 README Must Include

- What the app does (1-2 sentences)
- Contract addresses on target chain with block explorer links
- How to run locally (`yarn fork`, `yarn deploy`, `yarn start`)
- Architecture decisions (non-obvious implementation details)
- Client actions needed (acceptOwnership, set up keeper, etc.)
- Live app URL

### 21.2 README Must NOT Include

- SE2 template boilerplate
- Explanations of React, Solidity, or Ethereum
- Padding or filler content

### 21.3 Final Push

```bash
# Verify no secrets
grep -rn "0x[a-fA-F0-9]{64}" packages/
grep -rE "g\.alchemy\.com/v2/[A-Za-z0-9]" packages/

git add -A
git commit -m "feat: complete <project-name>"
git push
```

### 21.4 Deliver

- Live URL: `https://<CID>.ipfs.community.bgipfs.com/`
- GitHub repo URL
- Contract addresses with block explorer links
- Verification status

---

## Appendix A: Pipeline Flow Diagram

```
SPEC ──► DISCOVERY ──► PLAN ──► SPEC VERIFICATION GATE
                                        │
                                        ▼
SCAFFOLD ──► CONTRACTS ──► DEPLOY SCRIPTS ──► TESTS ──► COMPILE & TEST
                                                              │
                                                              ▼
CONTRACT AUDIT ──► AUDIT FIXES ──► EXTERNAL CONTRACTS
                                          │
                                          ▼
DEPLOY TO CHAIN ──► (real addresses in deployedContracts.ts)
        │
        ▼
FRONTEND DEV ──► FRONTEND QA ──► QA FIXES ──► SEMANTIC REVIEW
        │
        ▼
INTEGRATION AUDIT ──► DEPLOY TO IPFS ──► LIVE VERIFICATION
        │
        ▼
USER JOURNEY WALKTHROUGH ──► README ──► DELIVER
```

Every `──►` includes a gate. If the gate fails, go back. No exceptions.

---

## Appendix B: Prerequisites

### Required Tools

| Tool | Purpose |
|------|---------|
| Node.js (v18+) | Next.js builds, yarn |
| yarn | SE2 package manager |
| Foundry (forge, anvil, cast) | Solidity compilation, testing, deployment |
| git + gh | Version control, GitHub CLI |
| jq | JSON processing |
| curl | API calls, verification |
| npx | Scaffolding, CID conversion |

### Required Secrets (in `.env`, NEVER committed)

```
ALCHEMY_API_KEY        # Dedicated RPC (NEVER public RPCs)
PRIVATE_KEY            # Deployer wallet (or use keystore)
BGIPFS_API_KEY         # IPFS upload
```

### RPC Rule

**NEVER** use `mainnet.base.org`, `base.llamarpc.com`, `eth.llamarpc.com`, or any public RPC. Always Alchemy endpoints with API key. If `ALCHEMY_API_KEY` is not available, STOP and ask for it.

---

## Appendix C: Verified Contract Addresses

Cross-reference every address against this table. LLMs hallucinate addresses.

### Base (Chain ID: 8453)

| Contract | Address |
|----------|---------|
| WETH | `0x4200000000000000000000000000000000000006` |
| wstETH | `0xc1CBa3fCea344f92D9239c08C0568f6F2F0ee452` |
| USDC | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` |
| DAI | `0x50c5725949A6F0c72E6C4a641F24049A917DB0Cb` |
| cbETH | `0x2Ae3F1Ec7F1F5012CFEab0185bfc7aa3cf0DEc22` |
| Uniswap V3 Router | `0x2626664c2603336E57B271c5C0b26F421741e481` |
| Uniswap V3 Factory | `0x33128a8fC17869897dcE68Ed026d694621f6FDfD` |
| Uniswap V3 Quoter V2 | `0x3d4e44Eb1374240CE5F1B871ab261CD16335B76a` |
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

### Alchemy RPC Endpoints

```
Base:     https://base-mainnet.g.alchemy.com/v2/<API_KEY>
Mainnet:  https://eth-mainnet.g.alchemy.com/v2/<API_KEY>
Arbitrum: https://arb-mainnet.g.alchemy.com/v2/<API_KEY>
Optimism: https://opt-mainnet.g.alchemy.com/v2/<API_KEY>
```

---

## Appendix D: Security Checklist

Run against every contract. From ethskills.com/security/SKILL.md.

### Critical

- [ ] **Token decimals:** Never assume 18 — use `IERC20Metadata(token).decimals()`
- [ ] **Reentrancy:** CEI pattern or `ReentrancyGuard` on every external-calling function
- [ ] **SafeERC20:** All token calls use `safeTransfer`/`safeTransferFrom`/`forceApprove`
- [ ] **Oracle safety:** Chainlink results checked for staleness (`updatedAt` within threshold)
- [ ] **Vault inflation:** ERC4626 uses virtual shares (`_decimalsOffset()` >= 3) or minimum deposit
- [ ] **No infinite approvals:** Approve exact amounts or 3-5x, never `type(uint256).max`
- [ ] **Slippage protection:** All swaps have `amountOutMinimum` > 0
- [ ] **Access control:** `Ownable2Step`, not `Ownable`; owner = client, not deployer
- [ ] **Input validation:** Zero address checks, zero amount checks, reasonable bounds on all params
- [ ] **Events:** Emitted for every state change

### Important

- [ ] **MEV awareness:** Deadline parameters on swaps
- [ ] **Multiply before divide:** No precision loss in math
- [ ] **Custom errors:** Over require strings (gas efficiency + better UX)
- [ ] **No `tx.origin`:** Only `msg.sender`
- [ ] **Walkaway safety:** Users can always withdraw even if owner disappears
- [ ] **TWAP before swap:** Oracle check BEFORE the swap, not after
- [ ] **try/catch on `pool.observe()`:** New pools may lack observation history
- [ ] **Constructor validation:** All inputs validated in constructor

---

## Appendix E: Frontend UX Rules

Nine rules from ethskills.com/frontend-ux/SKILL.md. These govern every frontend.

### 1. Pending State for Every Write
Every `writeContractAsync` shows a spinner. Users must never wonder "did it work?"

### 2. Four-State Button Flow
`Connect -> Switch Network -> Approve -> Execute`. One button at a time.

### 3. Address UX
- Display with `<Address />` (blockie + ENS + explorer link)
- Input with `<AddressInput />` (ENS resolution + validation)
- Never raw `0x` strings

### 4. USD Context
Show USD equivalents next to token amounts where possible.

### 5. RPC Reliability
Never public RPCs. Always Alchemy. `pollingInterval: 3000` for L2s.

### 6. Theme Semantics
DaisyUI semantic classes. Custom colors in theme, not inline. No hardcoded dark backgrounds.

### 7. Error Translation
Every catch block maps contract errors to plain English. Raw hex never reaches users.

### 8. Metadata
Title, description, OG image updated. OG image absolute URL. SE2 branding removed everywhere.

### 9. Human-Readable Amounts
`formatEther`/`formatUnits` for display. Never raw wei. Use `.toFixed()` for precision.

---

## Appendix F: QA Checklist (Full)

The complete 25-item checklist. Run after every frontend build.

### Ship-Blocking (must ALL pass)

| # | Check | Verify With |
|---|-------|-------------|
| 1 | Wallet connection shows BUTTON not text | Look for `RainbowKitCustomConnectButton` or `openConnectModal` |
| 2 | Wrong network shows Switch button | Look for `chainId !== base.id` check |
| 3 | One button at a time | Visual: Connect -> Network -> Approve -> Action |
| 4 | Approve has two-state protection | Grep: `approvalSubmitting` AND `approveCooldown` both present |
| 5 | Contracts verified on explorer | Check each address on Basescan |
| 6 | SE2 footer removed | Read Footer.tsx — no BuidlGuidl/Fork me/Support |
| 7 | SE2 tab title removed | Read layout.tsx/getMetadata.ts — no "Scaffold-ETH 2" |
| 8 | No zero-address in frontend | `grep -rn "0x000...0" app/ components/` |
| 9 | No hardcoded deployed addresses | No `0x...` address literals matching deployedContracts.ts |
| 10 | Uses useScaffoldWriteContract | `grep -rn "useWriteContract" | grep -v scaffold-eth` = empty |

### Should-Fix

| # | Check | Verify With |
|---|-------|-------------|
| 11 | `<Address/>` for contract display | Grep `<Address` in app/components |
| 12 | `<AddressInput/>` for address inputs | `grep 'type="text"' app/ \| grep -i addr` = empty |
| 13 | OG image absolute URL | Check layout.tsx — uses https://, not relative path |
| 14 | pollingInterval 3000 | Read scaffold.config.ts |
| 15 | Favicon updated | Check public/favicon.* |
| 16 | --radius-field 0.5rem | Read globals.css — both theme blocks |
| 17 | No hardcoded dark backgrounds | `grep 'bg-\[#0\|bg-black\|bg-gray-9' app/` = empty |
| 18 | Inline spinner on buttons | `grep '"loading"' app/ components/` on button className = empty |
| 19 | Error messages readable | Every catch block calls `notification.error` with human message |
| 20 | Phantom wallet | Read wagmiConnectors.tsx — phantomWallet present |
| 21 | README replaced | Read README.md — project content, not SE2 template |
| 22 | Approve cooldown | Grep `setTimeout.*refetch\|cooldown` in components |
| 23 | Mobile deep linking | Grep `metamask://\|openWallet\|deep.link` |
| 24 | appName changed | Read wagmiConnectors.tsx — not "scaffold-eth-2" |
| 25 | Block explorer disabled | `ls app/blockexplorer` should not exist |

---

## Appendix G: SE2 Footguns

These are specific to Scaffold-ETH 2 and will bite you if you don't know about them.

| Footgun | Fix |
|---------|-----|
| `pollingInterval: 30000` default | Change to 3000 in scaffold.config.ts |
| `--radius-field: 9999rem` makes pill-shaped inputs | Change to 0.5rem in BOTH theme blocks |
| `appName: "scaffold-eth-2"` in wagmiConnectors | Change to your app name |
| `nativeCurrencyPrice` in Footer | Remove — shows ETH price badge on all networks |
| Block explorer crashes static export | Rename to `_blockexplorer-disabled` or delete |
| `deployedContracts.ts` manually edited | NEVER edit — auto-generated by `yarn deploy` |
| Missing `deployedOnBlock` in contract entries | SE2's useScaffoldEventHistory breaks without it |
| Font loading via `<link>` tag for Google Fonts | Use `next/font/google` in layout.tsx |
| `polyfill-localstorage.cjs` in wrong directory | Must be in `packages/nextjs/`, not project root |
| Build output in wrong directory | Upload `packages/nextjs/out/`, not `out/` at root |
| `useWriteContract` instead of `useScaffoldWriteContract` | Scaffold hooks wait for confirmation; raw wagmi doesn't |
| `bgipfs init` without `--endpoint` flag | Creates bad config pointing to localhost |
| `yarn ipfs` exit code | NOT reliable — swallows errors. Only valid signal = CID in output |
| Node 25+ `localStorage.getItem is not a function` | Add polyfill to NODE_OPTIONS |
| `trailingSlash: true` missing | Every route except `/` returns 404 on IPFS gateways |

---

## Appendix H: Common Failures from Real Builds

These are not theoretical — they happened on actual builds across multiple projects.

### Architecture Failures

| Failure | Symptom | Root Cause | Prevention |
|---------|---------|------------|------------|
| Wrong underlying asset | `availableYield()` returns 0 | Vault uses WETH (doesn't appreciate) instead of wstETH | Stage 3 spec verification gate |
| Hallucinated addresses | Transactions revert mysteriously | LLM guessed an address | Stage 1 discovery — `cast call` every address |
| Bridged token = plain ERC20 | Deploy reverts calling `wrap()` | L2 bridge wraps tokens as simple ERC20, no native functions | Stage 1 discovery — `cast interface` |

### Contract Failures

| Failure | Symptom | Root Cause | Prevention |
|---------|---------|------------|------------|
| Double nonReentrant | `ReentrancyGuardReentrantCall` on deposit | Public override adds nonReentrant, super has it too | Override `_deposit`/`_withdraw` (internal hooks) |
| transferFrom on own tokens | `ERC20InsufficientAllowance` | `super.deposit()` does transferFrom but tokens already in contract | Use `previewDeposit()` + `_mint()` for convenience functions |
| Zero address in constructor | Deploy reverts | Deploy script has `address(0)` for a token | Cross-ref constructor args with verified addresses |
| TWAP after swap | Sandwich attack works | Oracle check ran AFTER swap instead of BEFORE | TWAP check MUST precede swap |
| Deploy script ownership order | Admin functions can't be called | Transferred ownership before configuring cross-references | Deployer-first pattern: configure, then transfer |

### Frontend Failures

| Failure | Symptom | Root Cause | Prevention |
|---------|---------|------------|------------|
| Hardcoded addresses | Wrong contract or stale address | LLM copied address from deployedContracts.ts as literal | Use `useDeployedContractInfo()` always |
| walletDeepLink ghost import | `Module not found` build error | QA/Carlos fixer introduced non-existent import | Remove import, delete file if created |
| "Please connect your wallet" text | No connect button visible | LLM wrote text instead of rendering ConnectButton | QA check #1 |
| QA fixer overwrites unrelated files | Build breaks after fixes | Fixer rewrote block explorer pages | `git checkout --` unrelated files |
| Approve double-submit | Users waste gas | `isPending` drops before on-chain confirmation | Two-state pattern with `finally{}` |
| OG image localhost | Social sharing broken | `NEXT_PUBLIC_PRODUCTION_URL` not set | Set before production build |
| Stale BGIPFS upload | Live app shows old content | Didn't delete `.next` and `out` before rebuild | Always `rm -rf .next out` first |
| Public RPC rate limits | 429 errors, slow loads | Using mainnet.base.org | Always use Alchemy |

### Pipeline Failures

| Failure | Symptom | Root Cause | Prevention |
|---------|---------|------------|------------|
| Token limit truncation | JSON parse error | LLM output exceeds max_tokens | Increase token limit (see [Appendix K](#appendix-k-token-limit-guide)) |
| Frontend before deploy | Placeholder addresses | Phase 9 ran before Phase 8 | Enforce: deploy BEFORE frontend |
| Test fixes contract logic | Behavior changes | Fix loop changed contract to match wrong test | Review diffs — fix the test, not the contract |
| Subagent says "all done" | 15/21 QA items still failing | LLM didn't actually make changes | Never trust — run verification commands |

---

## Appendix I: Model Routing

Not all stages need the same model. Use cheaper models for simple tasks, expensive for complex reasoning.

| Stage | Recommended Model | Notes |
|-------|------------------|-------|
| 0 (read spec) | cheap | Reading, no generation |
| 1 (discovery) | cheap | Running cast commands |
| 2 (plan) | medium-expensive | Needs architectural reasoning |
| 3 (spec verify) | medium | Needs judgment |
| 4 (scaffold) | cheap | Shell commands |
| 5 (contracts) | expensive | Solidity correctness is critical |
| 6 (deploy scripts) | medium | Templated pattern |
| 7 (tests) | expensive | Needs contract semantics |
| 8 (compile/test) | expensive | Fix attempts need reasoning |
| 9 (audit) | medium | Checklist analysis |
| 10 (audit fixes) | expensive | Contract changes |
| 11 (external contracts) | medium | ABI generation |
| 12 (deploy) | cheap | Shell commands |
| 13 (frontend) | expensive | React + SE2 + contract integration |
| 14 (QA) | medium | Pattern analysis |
| 15 (QA fixes) | expensive | Code changes |
| 16 (semantic review) | expensive | Deep reasoning |
| 17 (integration audit) | medium | Cross-checking |
| 18 (IPFS deploy) | cheap | Shell commands |
| 19-20 (verification) | cheap | Checking |
| 21 (delivery) | medium | Documentation |

**Model tiers:**
| Tier | Model | Cost | Use for |
|------|-------|------|---------|
| cheap | Minimax M2.7 | ~$0.01 | Shell commands, reading, checking |
| medium | Claude Sonnet 4.6 | ~$0.10 | Plans, audits, fixes, frontend |
| expensive | Claude Opus 4.6 | ~$0.30 | Contracts, tests, complex frontend, semantic review |

---

## Appendix J: Skill Routing

| Stage | Skills to Load |
|-------|---------------|
| Writing contracts | `security`, `testing`, `openzeppelin` (if using OZ) |
| Writing deploy scripts | `orchestration` |
| Writing tests | `testing` |
| Frontend development | Read AGENTS.md as file; `frontend-ux`, `qa` |
| DeFi vaults/staking | `security` (ERC4626 section) |
| NFT collection | `erc-721` |
| External protocols | `addresses` |
| Production deploy | `frontend-playbook`, `ship` |
| Audit | `audit` + specialized audit skills |

All skills: `https://ethskills.com/<skill>/SKILL.md`
SE2 docs: `https://docs.scaffoldeth.io/SKILL.md`

---

## Appendix K: Token Limit Guide

Different stages need different `max_tokens` values when using LLM APIs. These were determined empirically.

| Stage | Default | When to Increase | Max Useful |
|-------|---------|-------------------|------------|
| Spec parsing | 4096 | Rarely | 4096 |
| Contract writing | 16384 | 3+ complex contracts (DeFi, ERC4626) | 32768 |
| Deploy scripts | 4096 | 5+ contracts | 8192 |
| External contracts | 4096 | 10+ external contracts | 16384 |
| Test writing | 32768 | 5+ contracts, complex scenarios | 65536 |
| Frontend | 16384 | 5+ components, complex UI | 32768 |
| QA fixes | 16384 | Many files to fix | 32768 |
| Semantic review | 16384 | Many files | 32768 |

**How to detect truncation:** If JSON parsing fails and raw output looks cut off mid-sentence, the token limit was too low. Increase and retry.

**One file per step for complex builds:** If the spec calls for 3+ contracts, generate each contract in its own LLM call to avoid truncation.

---

## Appendix L: Verification Commands

Quick verification commands for common checks. Run these after every build stage.

```bash
# === SE2 BRANDING ===
grep -rn "scaffold-eth-2\|Scaffold-ETH 2\|BuidlGuidl" packages/nextjs/ | grep -v node_modules
grep -c "9999rem" packages/nextjs/styles/globals.css
ls packages/nextjs/app/blockexplorer 2>/dev/null && echo "FAIL: blockexplorer exists"

# === RAW WAGMI ===
grep -rn "useWriteContract\|useReadContract" packages/nextjs/ | grep -v scaffold-eth | grep -v node_modules

# === DARK BACKGROUNDS ===
grep -rn 'bg-\[#0\|bg-black\|bg-gray-9\|bg-zinc-9' packages/nextjs/app/

# === ADDRESS INPUTS ===
grep -rn 'type="text"' packages/nextjs/app/ | grep -i "addr\|owner\|recip\|0x"

# === LOADING CLASS ON BUTTONS ===
grep -rn '"loading"' packages/nextjs/app/ packages/nextjs/components/ | grep -i "btn\|button\|className"

# === OG IMAGE ===
grep -rn "localhost" packages/nextjs/utils/ packages/nextjs/app/layout.tsx

# === SECRETS ===
grep -rn "0x[a-fA-F0-9]{64}" packages/
grep -rE "g\.alchemy\.com/v2/[A-Za-z0-9]" packages/

# === PUBLIC RPC ===
grep -n "mainnet.base.org\|base.llamarpc\|eth.llamarpc" packages/foundry/foundry.toml

# === APPROVAL FLOW ===
grep -c "approvalSubmitting\|approveCooldown" packages/nextjs/app/page.tsx packages/nextjs/components/*.tsx 2>/dev/null

# === BUILDS ===
cd packages/foundry && forge build && echo "CONTRACTS: PASS" || echo "CONTRACTS: FAIL"
cd ../.. && yarn next:build && echo "FRONTEND: PASS" || echo "FRONTEND: FAIL"
```

---

## Appendix M: File Locations

| What | Where |
|------|-------|
| Smart contracts | `packages/foundry/contracts/` |
| Contract tests | `packages/foundry/test/` |
| Deploy scripts | `packages/foundry/script/` |
| Deploy helpers (don't modify) | `packages/foundry/script/DeployHelpers.s.sol` |
| Foundry config | `packages/foundry/foundry.toml` |
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
| Metadata | `packages/nextjs/utils/scaffold-eth/getMetadata.ts` or `app/layout.tsx` |

---

## Appendix N: Import Paths

```typescript
// Scaffold hooks (ALWAYS use these — never raw wagmi)
import { useScaffoldReadContract, useScaffoldWriteContract } from "~~/hooks/scaffold-eth";
import { useDeployedContractInfo } from "~~/hooks/scaffold-eth";
import { useScaffoldEventHistory } from "~~/hooks/scaffold-eth";

// UI components
import { Address, AddressInput, Balance, EtherInput } from "@scaffold-ui/components";

// Connect button
import { RainbowKitCustomConnectButton } from "~~/components/scaffold-eth";

// Notifications and error parsing
import { notification } from "~~/utils/scaffold-eth";
import { getParsedError } from "~~/utils/scaffold-eth";

// Viem utilities
import { formatEther, parseEther, formatUnits, parseUnits } from "viem";

// Wagmi (only for wallet state, NOT for contract interaction)
import { useAccount, useChainId, useSwitchChain } from "wagmi";

// RainbowKit
import { useConnectModal } from "@rainbow-me/rainbowkit";

// Next.js
import type { NextPage } from "next";
```

**Path alias:** `~~` maps to `packages/nextjs/` root. Always use it.

---

## Appendix O: Regression Protocol

When a later stage reveals a problem from an earlier stage, go back. Always fix the root cause, not the symptom.

```
Stage 20 bug (user journey fails)    → Stage 13 (frontend) or Stage 5 (contracts)
Stage 19 bug (live app broken)       → Stage 13 (frontend), rebuild + re-upload
Stage 17 bug (integration)           → wherever the root cause is
Stage 14 bug (QA)                    → Stage 13 (frontend)
Stage 9 bug (contract audit)         → Stage 5 (contracts)
Stage 3 bug (spec verification)      → Stage 2 (plan)
```

**Rules:**
- Bug in production frontend → fix locally, rebuild, re-upload to IPFS
- Bug in contract logic → fix contracts, retest, redeploy everything
- Never patch production bugs without proper phase regression
- When regressing, explain why. Fix root cause, not symptom.
- After fixing, re-run ALL subsequent gates (not just the one that failed)

---

## Quick Reference: Full Pipeline Commands

```bash
# Stage 4: Scaffold
npx -y create-eth@latest -s foundry my-dapp && cd my-dapp && yarn install

# Stage 5-6: Contracts
cd packages/foundry && forge build

# Stage 7-8: Tests
cd packages/foundry && forge test -vv

# Stage 12: Deploy
yarn deploy --network base
yarn verify --network base

# Stage 13: Frontend
yarn next:build

# Stage 18: IPFS
cd packages/nextjs && rm -rf .next out
NEXT_PUBLIC_IPFS_BUILD=true NODE_OPTIONS="--require ./polyfill-localstorage.cjs" npm run build
# Upload out/ to IPFS

# Verification
forge build && echo "PASS" || echo "FAIL"
yarn next:build && echo "PASS" || echo "FAIL"
```
