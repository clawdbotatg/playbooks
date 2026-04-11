# Comprehensive dApp Building Playbook

From spec file to finished, deployed, audited decentralized application. Every step has three parts: **do the thing**, **verify the thing works**, **verify it does what it's supposed to**. Fix repeatedly until solid. Then continue.

This playbook synthesizes six build pipelines (leftclaw-services, clawd-builder, clawd-DSA-LCS-builder, ethereum-servicer, yet-another-builder-agent, clawd-measure-normalized) and the full ethskills.com skill library into a single authoritative process.

---

## Philosophy

1. **Spec fidelity over pipeline completion.** A build that compiles and deploys is worthless if it doesn't implement the spec. Every code stage has a verification gate that checks correctness, not just compilation.
2. **Build the thing, prove the thing works, prove it does the right thing.** Three-part verification at every stage.
3. **Never optimize for pipeline progress.** If a late stage reveals the architecture is wrong, go back. Moving forward with a broken foundation creates exponentially more work.
4. **Fork real chains, use real protocols.** Empty local chains hide integration bugs. Always fork the target network.
5. **AI is sloppy.** AI hallucinates interfaces, uses wrong import paths, calls functions that don't exist on L2 bridged tokens, says "done" when it isn't, skips testing, and leaves stubs. Every stage assumes this and verifies against reality.

---

## Pipeline Overview

```
SPEC ──► DISCOVERY ──► PLAN ──► SPEC GATE ──► CONTRACTS ──► CONTRACT GATE
                                                               │
                                                               ▼
                                              CONTRACT AUDIT ──► AUDIT FIXES
                                                               │
                                                               ▼
                                              FRONTEND ──► FRONTEND QA ──► QA FIXES
                                                               │
                                                               ▼
                                              INTEGRATION AUDIT
                                                               │
                                                               ▼
                                              DEPLOY CONTRACTS ──► LIVE TEST
                                                               │
                                                               ▼
                                              DEPLOY FRONTEND ──► LIVE WALKTHROUGH
                                                               │
                                                               ▼
                                              README ──► DELIVER
```

Every arrow includes a gate. If the gate fails, go back. No exceptions.

---

## Stage 0: Read the Spec

### What you do

Read the full spec. Read it again. Extract every technical requirement.

### Actions

1. Read the job description / spec file completely
2. Read ALL client messages and communications — clients add requirements, preferences, and scope changes after the initial spec. Chat is authoritative; the spec is the baseline.
3. Extract and document:
   - **What the app does** (one sentence)
   - **What tokens/protocols are involved** (exact contract addresses — verify on-chain, never guess)
   - **What chain** (default: Base)
   - **What contracts are needed** (names, functions, interactions)
   - **What the frontend does** (pages, flows, design)
   - **Who owns what** (client address gets ALL privileged roles)
   - **What external APIs/protocols** are needed

### Onchain vs offchain litmus test

Put it onchain ONLY if it requires: trustless ownership, trustless exchange, composability with other protocols, censorship resistance, or permanent commitments. Everything else stays offchain.

**Most MVPs need 0-2 smart contracts.** Three is the upper bound for a first release.

### State transition audit

For every planned contract function:

| Function | Who calls it? | Why would they? | What if nobody calls it? | Gas incentive needed? |
|----------|--------------|-----------------|--------------------------|----------------------|

Smart contracts cannot execute themselves. No cron, no scheduler. Every function needs a caller who pays gas. If your answer to "who calls it?" is "the team" — redesign with aligned incentives.

### Deliverable

`SPEC_REQUIREMENTS.md` — every requirement as a numbered, testable item.

### Verification

- [ ] Every requirement is testable ("the vault uses wstETH as underlying" not "the vault works")
- [ ] Every external address is verified: `cast code <addr> --rpc-url $RPC` returns non-zero
- [ ] Every token verified: `cast call <addr> "symbol()(string)" --rpc-url $RPC`
- [ ] Chain is specified
- [ ] Ambiguities are resolved (escalate, don't guess)

### Skills to read

- `https://ethskills.com/ship/SKILL.md` — architecture patterns, onchain litmus test
- `https://ethskills.com/concepts/SKILL.md` — incentive design, state transitions
- `https://ethskills.com/gas/SKILL.md` — real costs by chain
- `https://ethskills.com/l2s/SKILL.md` — chain selection

**STOP. Do not write code until this stage passes.**

---

## Stage 1: Discovery (Anti-Hallucination)

### What you do

Before any code generation, verify what actually exists. This stage prevents the single largest class of AI build failures: calling functions that don't exist, importing from paths that don't exist, using interfaces from the wrong chain.

### 1A: Discover on-chain interfaces

For every external contract address, fetch the REAL ABI from the target chain:

```bash
cast interface <address> --chain base
```

Store the actual function signatures. Example critical finding: wstETH on mainnet has `wrap()`, `unwrap()`, `stETH()`. wstETH on Base is a **bridged ERC20 with none of those functions.** Without discovery, the AI writes code against the mainnet interface and the deploy reverts.

### 1B: Discover installed package exports

After scaffolding (or against a reference SE2 project), read the actual exports:

```bash
cat packages/nextjs/hooks/scaffold-eth/index.ts
cat packages/nextjs/components/scaffold-eth/index.tsx
```

Store which hooks and components exist and their exact import paths. Critical finding: `Address` is exported from `@scaffold-ui/components`, NOT from `~~/components/scaffold-eth/Address`. Without discovery, every frontend build fails with import errors.

### 1C: Discover project structure

```bash
cat packages/foundry/script/DeployHelpers.s.sol   # deploy pattern
cat packages/nextjs/scaffold.config.ts              # config shape
head -5 packages/nextjs/styles/globals.css          # CSS framework version
```

### Deliverable

`discovery.json` containing:
- `onChain` — every external contract's real function signatures
- `packageExports` — every hook, component, and its import path
- `projectStructure` — deploy pattern, CSS version, config details

### Verification

- [ ] Every external contract has verified function signatures
- [ ] Every import path is confirmed against actual package exports
- [ ] No imagined interfaces — if discovery doesn't show `wrap()`, no code may call `wrap()`

### Why this matters

This is the stage AI agents skip. Without it:
- Contracts call functions that don't exist on L2 bridged tokens → deploy reverts
- Frontend imports from wrong paths → build fails 3+ times
- Deploy scripts use wrong constructor patterns → silent failure

**STOP. Do not plan or write code without discovery artifacts.**

---

## Stage 2: Scaffold

### Actions

```bash
npx -y create-eth@latest -s foundry <project-name>
cd <project-name>
```

Use Foundry flavor (default 2026). Kebab-case naming.

### Post-scaffold

1. Read `AGENTS.md` — contains correct hook names, components, code style, available skills
2. Read every relevant skill in `.agents/skills/<name>/SKILL.md`
3. Initialize git repo and push to GitHub

### Verification

- [ ] `packages/foundry/` exists
- [ ] `packages/nextjs/` exists
- [ ] `forge build` compiles default contract
- [ ] `AGENTS.md` has been read

---

## Stage 3: Architecture Plan

### Actions

Write `PLAN.md` covering:

1. **One-sentence summary** of what the app does
2. **Architecture diagram** (text) showing contract relationships
3. **Contract specifications** — for each contract:
   - Name, purpose, inheritance chain
   - State variables with types
   - Every function with signature, access control, behavior
   - Events and custom errors
   - External protocol calls (exact function signatures from discovery.json)
4. **Value flow** — how does money/yield/value move through the system?
5. **Deploy script** — constructor arguments, all external addresses
6. **Post-deploy configuration** — cross-contract wiring, ownership transfer
7. **Frontend specification** — pages, sections, flows, SE2 hooks for each interaction
8. **External addresses** — every address, verified with cast calls
9. **Security considerations** — attack vectors and mitigations

Write `USERJOURNEY.md` covering:

1. User lands on page (not connected)
2. User connects wallet
3. User on wrong network → Switch flow
4. User performs each action (step by step, each click, each tx)
5. User sees result
6. Edge cases: no wallet, wrong network, zero balance, insufficient allowance, tx rejected, tx reverted, pending state, rapid clicks

### Plan rules

- **Use only what discovery found.** If discovery shows a token is a plain ERC20, the plan must not call rich protocol functions.
- **Every external call maps to a real function signature in discovery.json.**
- **Every frontend import maps to a real export in discovery.json.**

### Verification

- [ ] PLAN.md has all 9 sections
- [ ] USERJOURNEY.md covers happy path + 5+ edge cases
- [ ] Every external address verified with a cast call
- [ ] No imagined interfaces — every call maps to discovery

### Skills to read

- `https://ethskills.com/orchestration/SKILL.md` — three-phase system
- `https://ethskills.com/security/SKILL.md` — vulnerability patterns
- `https://ethskills.com/standards/SKILL.md` — token standards, ERC-8004, x402

---

## Stage 4: Spec Verification Gate

### What you do

Before writing ANY code, verify the plan actually implements the spec. This gate prevents the entire class of "built the wrong thing" failures.

### Actions

Go through `SPEC_REQUIREMENTS.md` line by line. For each requirement:

1. Find where in PLAN.md it is addressed
2. Verify the approach actually satisfies the requirement
3. Check specifics:
   - [ ] **Underlying asset is correct.** If spec says "ETH staking yield," vault MUST use wstETH (which appreciates), NOT WETH (which doesn't).
   - [ ] **Yield mechanism is real.** Can you explain exactly how yield appears? If `availableYield()` would return 0, the mechanism is broken.
   - [ ] **Swap paths exist.** For every swap: `cast call <FACTORY> "getPool(address,address,uint24)(address)" <A> <B> <FEE> --rpc-url $RPC`. Zero address = pool doesn't exist.
   - [ ] **Token decimals are correct.** USDC = 6 decimals, wstETH = 18. Wrong decimals = all math breaks.
   - [ ] **Owner is client.** Constructor passes client address, not `msg.sender`.
   - [ ] **No stub logic.** Every function has a real implementation path.

### Verification

Every line in SPEC_REQUIREMENTS.md has: "VERIFIED — see PLAN.md section X" or "FAILED — plan says Y but spec says Z."

If ANY requirement fails: go back to Stage 3. Do NOT proceed to code.

---

## Stage 5: Smart Contract Development

*Skip this entire stage if the app needs zero contracts.*

### 5.1 Configure target network

- `scaffold.config.ts`: set `targetNetworks` (e.g., `chains.base`), `pollingInterval` (2000 for L2, 3000 for mainnet)
- `foundry.toml`: add Alchemy RPC endpoint. Never use public RPCs.
- Register external contracts in `packages/nextjs/contracts/externalContracts.ts`

### 5.2 Write contracts

Location: `packages/foundry/contracts/`

**Mandatory patterns:**
- OpenZeppelin base contracts (check what's installed in `lib/openzeppelin-contracts/`)
- `Ownable2Step` over `Ownable` — two-step prevents accidental ownership transfer
- `ReentrancyGuard` on every function with external calls
- `SafeERC20` for ALL token transfers — never raw `transfer()`/`transferFrom()`
- CEI pattern (Checks-Effects-Interactions) — state changes before external calls
- Custom errors over require strings: `error ZeroAddress();` not `require(addr != address(0), "zero")`
- Events for EVERY state change (events are your frontend API)
- Input validation: zero addresses, zero amounts, reasonable bounds
- Never `type(uint256).max` for approvals
- Never `tx.origin`
- All privileged roles set to client address

**ERC4626 vaults:**
- Virtual shares: `_decimalsOffset() returns (uint8) { return 3; }` — prevents inflation attacks
- Override `_deposit`/`_withdraw` (internal hooks), NOT `deposit`/`withdraw` (public)

**Uniswap V3 swaps:**
- Always set `amountOutMinimum > 0` (slippage protection)
- Always set `deadline`
- TWAP check BEFORE the swap
- Wrap `pool.observe()` in try/catch

### 5.3 Write deploy script

Location: `packages/foundry/script/`

- Inherit `ScaffoldETHDeploy`, use `ScaffoldEthDeployerRunner` modifier
- Deploy contracts inline: `new MyContract(args)` inside `run()`
- Push to `deployments` array
- Wire cross-references after all deploys
- Deployer-first pattern: deploy with deployer as owner, configure, then `transferOwnership()` to client
- Zero-address checks in every constructor

### 5.4 Write tests

Location: `packages/foundry/test/`

**Priority (from testing skill):**
1. **Unit tests** — edge cases, failure modes, access control. NOT getters.
2. **Fuzz tests** — any function with math. Min 1000 runs. Use `bound()` not `vm.assume()`.
3. **Fork tests** — any interaction with external protocols. Fork mainnet with `anvil --fork-url`.
4. **Invariant tests** — stateful protocols. Properties that must always hold.

```bash
forge test -vvv                    # all tests
forge test --match-test testFuzz   # fuzz tests
forge test --fork-url $RPC_URL     # fork tests
```

### 5.5 Deploy locally

```bash
yarn fork --network base     # Fork real chain — never use yarn chain
cast rpc anvil_setIntervalMining 1   # Enable block mining
yarn deploy                  # Deploy to fork
```

Frontend MUST target `chains.foundry` (chain ID 31337) during local dev, not the forked network.

### Verification (three parts)

**Does it compile?**
- [ ] `forge build` exits 0

**Does it work?**
- [ ] `forge test` passes all tests
- [ ] `deployedContracts.ts` is generated with real addresses

**Does it do the right thing?**
- [ ] Walk through SPEC_REQUIREMENTS.md again — every requirement maps to contract code
- [ ] `vault.asset()` returns the correct token
- [ ] Yield calculation produces correct numbers with example inputs
- [ ] Swap paths match real pools
- [ ] Owner is set to client address
- [ ] No stub functions, no TODO comments, no empty bodies
- [ ] No hardcoded test values

### Skills to read

- `https://ethskills.com/security/SKILL.md` — vulnerability patterns
- `https://ethskills.com/testing/SKILL.md` — test methodology
- `https://ethskills.com/openzeppelin/SKILL.md` (if using OZ)
- `https://ethskills.com/erc-721/SKILL.md` (if building NFTs)

---

## Stage 6: Contract Audit

### Actions

1. Run automated tools: `slither packages/foundry/contracts/`
2. Fetch `https://ethskills.com/audit/SKILL.md` — master routing for 20 parallel security domain checklists
3. Based on contract type, load relevant specialized skills (general, math, ERC20, ERC4626, staking, oracles, AMM, etc.)
4. Run each checklist systematically
5. File GitHub issues for every finding: severity, location, description, recommendation

### Manual security checklist

- [ ] Token decimals handled correctly (USDC = 6, WETH = 18)
- [ ] Multiply before divide (no precision loss from truncation)
- [ ] No spot price oracle usage (use Chainlink or TWAP 30+ min)
- [ ] No infinite approvals
- [ ] Reentrancy guards on external calls
- [ ] Access control on all state-changing functions
- [ ] Input validation (zero address, zero amount, bounds)
- [ ] Events emitted for all state changes
- [ ] MEV protection (slippage limits on swaps)
- [ ] No `tx.origin` usage
- [ ] ERC4626 inflation protection (virtual shares)

### Fix all findings

1. Fix each issue
2. Close with commit reference
3. Re-run `forge build && forge test` after each fix
4. Re-run Slither after all fixes

### Verification

- [ ] Zero open audit issues
- [ ] All tests still pass after fixes
- [ ] Slither clean (or all findings addressed)

---

## Stage 7: Frontend Development

### 7.1 Branding — FIRST THING

Remove ALL SE2 default branding. AI agents treat the scaffold as sacred — don't.

- [ ] `app/layout.tsx` — change title and description
- [ ] `components/Header.tsx` — change app name, navigation
- [ ] `components/Footer.tsx` — remove BuidlGuidl links, "Fork me", support links
- [ ] Replace favicon
- [ ] Delete or repurpose `app/debug/` and `app/blockexplorer/`
- [ ] Replace `README.md`

### 7.2 Styling

DaisyUI semantic classes only, never raw Tailwind colors:

```tsx
// YES
<div className="bg-base-200 text-base-content">
<button className="btn btn-primary">

// NO — breaks theme toggle
<div className="bg-[#0a0a0a] text-white">
```

Fix pill-shaped inputs: `--radius-field: 0.5rem` in `globals.css` (both theme blocks).

Verify no hardcoded dark backgrounds:
```bash
grep -rn 'bg-\[#0\|bg-black\|bg-gray-9\|bg-zinc-9' packages/nextjs/app/
```

### 7.3 Wallet connection flow (CRITICAL)

**The #1 AI agent mistake:** writing `<p>Please connect your wallet</p>` instead of a Connect Wallet **button**.

Show exactly ONE primary button at a time:

```
1. Not connected  → Connect Wallet button (RainbowKitCustomConnectButton)
2. Wrong network  → Switch to [Chain] button
3. Needs approval → Approve button (dual-state locked)
4. Ready          → Action button
```

Never show Approve and Action simultaneously.

**Approval double-submit prevention (TWO states required):**

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
    setApprovalSubmitting(false); // MUST be in finally — releases on rejection
  }
};

<button disabled={isPending || approvalSubmitting || approveCooldown}>
  {(isPending || approvalSubmitting) && <span className="loading loading-spinner loading-sm mr-2" />}
  {isPending || approvalSubmitting ? "Approving..." : "Approve"}
</button>
```

- `approvalSubmitting` covers click-to-wallet-hash gap
- `approveCooldown` covers confirm-to-cache-refresh gap

### 7.4 SE2 hooks and components

**Use discovery.json import paths.** Never guess.

Hooks (never use raw wagmi):

| Need | Use | Never |
|------|-----|-------|
| Read contract | `useScaffoldReadContract` | `useReadContract` |
| Write contract | `useScaffoldWriteContract` | `useWriteContract` |
| Events | `useScaffoldEventHistory` | `useWatchContractEvent` |

Verify no raw wagmi:
```bash
grep -rn "useWriteContract\|useReadContract" packages/nextjs/app/ | grep -v node_modules
```

Components (never use raw HTML for web3 data):

| Need | Component | Never |
|------|-----------|-------|
| Display address | `<Address />` | Raw truncated hex |
| Input address | `<AddressInput />` | `<input type="text" placeholder="0x...">` |
| Display balance | `<Balance />` | Raw wei number |
| Input ETH | `<EtherInput />` | `<input type="number">` |

### 7.5 Display standards

- USD values next to ALL token/ETH amounts: `"0.5 ETH (~$1,250)"`
- Contract address displayed with `<Address />`
- Human-readable amounts with `formatEther()` / `parseEther()`
- Every contract error mapped to human-readable message — no silent catches, no raw hex
- Button loading: inline `<span className="loading loading-spinner loading-sm" />` inside button, NOT DaisyUI `loading` class on button

### 7.6 Mobile deep linking

RainbowKit v2 does NOT auto-deep-link. Implement:

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

Rules: fire TX first, deep link second. Skip if `window.ethereum` exists. Check WC session data for wallet detection. Wrap EVERY write call.

### 7.7 API routes (if needed)

For external API proxying, caching, server-side logic:

```typescript
// packages/nextjs/app/api/<name>/route.ts
import { NextRequest, NextResponse } from "next/server";
export async function GET(request: NextRequest) {
  return NextResponse.json({ data });
}
```

### 7.8 Verify in browser

```bash
yarn start   # http://localhost:3000
```

**Does it load?**
- [ ] Page loads without errors
- [ ] No console errors

**Does it work?**
- [ ] Every button works
- [ ] All data displays correctly
- [ ] Loading states during async operations
- [ ] Error states show human-readable messages

**Does it do the right thing?**
- [ ] Walk through USERJOURNEY.md step by step in the browser
- [ ] Every happy path works
- [ ] Every edge case handled (wrong network, zero balance, tx rejected)
- [ ] No stubs, no placeholder data, no TODO text

### Skills to read

- `https://ethskills.com/frontend-ux/SKILL.md` — 9 mandatory UX patterns
- `https://ethskills.com/qa/SKILL.md` — full QA checklist
- `https://docs.scaffoldeth.io/SKILL.md` — SE2 builder guide

---

## Stage 8: Frontend QA

### Process

Give the app to a fresh reviewer (human or agent). They report PASS/FAIL only — they do NOT fix anything.

### Ship-blocking (must all pass)

- [ ] Wallet connection shows a BUTTON, not text
- [ ] Wrong network shows a Switch button
- [ ] One button at a time (Connect → Network → Approve → Action)
- [ ] Approve button locked with BOTH `approvalSubmitting` AND `approveCooldown`
- [ ] Contracts verified on block explorer (green checkmark)
- [ ] SE2 footer branding removed
- [ ] SE2 tab title removed
- [ ] SE2 README replaced
- [ ] No raw wagmi hooks outside scaffold-eth internals

### Should-fix

- [ ] Contract address displayed with `<Address />`
- [ ] Every address input uses `<AddressInput />`
- [ ] USD values next to all amounts
- [ ] OG image is absolute production URL
- [ ] `pollingInterval` is 2000-3000 (not 30000)
- [ ] RPC overrides set AND env vars confirmed on hosting
- [ ] Bare `http()` fallback removed from wagmiConfig
- [ ] Favicon updated
- [ ] `--radius-field` changed from `9999rem` to `0.5rem`
- [ ] Contract errors mapped to human-readable messages
- [ ] No hardcoded dark backgrounds
- [ ] Button loaders use inline spinner, not DaisyUI `loading` class
- [ ] Phantom wallet in RainbowKit wallet list
- [ ] Mobile deep linking for all transaction buttons
- [ ] Mobile wallet detection checks WC session data

### Automated checks

```bash
grep -rn "useWriteContract\|useReadContract" packages/nextjs/app/
grep -rn 'type="text"' packages/nextjs/app/ | grep -i "addr\|owner\|0x"
grep -rn 'bg-\[#0\|bg-black\|bg-gray-9\|bg-zinc-9' packages/nextjs/app/
grep -rn '"loading"' packages/nextjs/app/
grep -n "og:image\|images:" packages/nextjs/app/layout.tsx
```

### Fix all findings

File issues, fix each, close with commit, re-verify. Repeat until all ship-blockers pass.

---

## Stage 9: Full Integration Audit

One final pass across ALL components together.

- [ ] Every requirement in SPEC_REQUIREMENTS.md is implemented and working
- [ ] No critical security defects
- [ ] No risk of locked or lost funds
- [ ] No stub functions or TODO comments
- [ ] All external addresses verified on-chain
- [ ] Deploy script constructor args match contract constructors
- [ ] Frontend reads/writes match contract function signatures
- [ ] Event names in `useScaffoldEventHistory` match contract events
- [ ] scaffold.config.ts targets correct chain
- [ ] No secrets in committed code
- [ ] `.gitignore` excludes `.env`, `*.key`, `broadcast/`, `cache/`

Fix anything found. This is the last chance before live deployment.

---

## Stage 10: Deploy Contracts to Live Network

*Skip if zero-contract app.*

### 10.1 Deploy

```bash
yarn deploy --network base
```

Verify `deployedContracts.ts` updated with live addresses.

### 10.2 Verify on block explorer

```bash
yarn verify --network base
```

**Every contract must show verified source with green checkmark.** Check manually on Basescan. If bytecode only — not verified.

### 10.3 Post-deploy

- Call cross-contract configuration functions (setHarvester, setRewardDistributor, etc.)
- Transfer ownership to client: `transferOwnership(clientAddress)` — client must call `acceptOwnership()` (Ownable2Step)
- Or transfer to Safe multisig (recommended: 2-of-3)

### 10.4 Test on live network

```bash
# Verify contract state
cast call <ADDR> "asset()(address)" --rpc-url $RPC
cast call <ADDR> "owner()(address)" --rpc-url $RPC
```

Run `yarn start` against live contracts. Test every flow with real wallet and small amounts.

### Verification

- [ ] Every contract verified on block explorer
- [ ] Ownership transferred correctly
- [ ] All view functions return expected values
- [ ] At least one successful live transaction

---

## Stage 11: Deploy Frontend

### Pre-deploy checklist

- [ ] `scaffold.config.ts` points at production network
- [ ] RPC overrides use env vars that are SET on hosting (not just referenced in code)
- [ ] OG image is absolute production URL
- [ ] Tab title is app name
- [ ] SE2 branding removed
- [ ] No secrets in committed code: `grep -r "0x[a-fA-F0-9]{64}" packages/nextjs/`
- [ ] Phantom wallet in wallet list

### Option A: IPFS (preferred for trustless deployment)

```bash
rm -rf packages/nextjs/.next packages/nextjs/out
```

Remove `app/debug/` and `app/blockexplorer/` (they use `force-dynamic`, incompatible with `output: "export"`).

```bash
NEXT_PUBLIC_PRODUCTION_URL="https://yourapp.eth.link" \
  NEXT_PUBLIC_IPFS_BUILD=true \
  NEXT_PUBLIC_IGNORE_BUILD_ERROR=true \
  yarn build
yarn ipfs
```

**Critical:** `yarn ipfs` exit code is NOT reliable (swallows errors). The only valid signal is an IPFS CID in the output: `Qm...` (44 chars) or `bafy...` (50+ chars). No CID = failed.

Requirements:
- `trailingSlash: true` in next.config.ts (IPFS gateways need it)
- `output: "export"` (auto-set by `NEXT_PUBLIC_IPFS_BUILD=true`)
- Verify CID actually changed from last deploy

### Option B: Vercel

**Cloud build (if it fits in memory):**
```bash
yarn vercel:yolo --yes --prod
```

**Local build + prebuilt (for SE2 monorepos that OOM on free tier):**
```bash
cd packages/nextjs
npx vercel pull --yes --scope <scope>
NEXT_PUBLIC_IGNORE_BUILD_ERROR=true npx vercel build --prod
npx vercel deploy --prebuilt --prod --scope <scope>
```

Known issue: `@vercel/next` passes `--localstorage-file` without valid path, creating broken `localStorage`. Fix with webpack BannerPlugin polyfill for `globalThis.localStorage` in server chunks (see next.config.ts webpack section, `isServer` block).

Set env vars on Vercel: `vercel env add NEXT_PUBLIC_ALCHEMY_API_KEY`. Verify: `vercel env ls`.

### Option C: ENS Subdomain

Two mainnet transactions:
1. Create subdomain in ENS app
2. Set IPFS content hash

Use `.eth.link` gateways for mobile support.

### Push to GitHub

```bash
# Verify no secrets
grep -r "0x[a-fA-F0-9]{64}" packages/
grep -rE "g\.alchemy\.com/v2/[A-Za-z0-9]" packages/

git add -A && git commit -m "Deploy to production" && git push
```

---

## Stage 12: Live User Journey Walkthrough

Open the LIVE app (IPFS/Vercel/ENS URL) in a browser with a REAL wallet. Follow `USERJOURNEY.md` step by step.

- Actually click every button
- Actually connect your wallet
- Actually submit transactions with real (small) amounts
- Actually verify results on the block explorer

### Regression protocol

| Bug location | Action |
|-------------|--------|
| Frontend display/UX | Fix locally, redeploy frontend |
| Contract logic | Return to Stage 5, fix, retest, redeploy EVERYTHING |
| Missing spec requirement | Return to Stage 3, update plan, implement, audit, redeploy |

Never patch production directly. Always fix locally, test, then redeploy.

### Verification

- [ ] Every step in USERJOURNEY.md works on the live deployment
- [ ] No errors in browser console
- [ ] Transactions confirm on-chain
- [ ] Results match expected behavior

---

## Stage 13: Documentation & Delivery

### Write README.md

Include:
- What the app does (2-3 sentences)
- Contract addresses and chain
- How to run locally
- Architecture decisions
- Link to live app

Do NOT include: SE2 boilerplate, explanations of React/Solidity/Ethereum, padding.

### Deliver

- App is live at production URL
- All features work
- No secrets exposed
- README complete

---

## Reference: Secret Management

**Never commit secrets.** Bots scan GitHub in real-time and exploit leaked keys within seconds.

Before every commit:
```bash
git diff --cached --name-only | grep -iE '\.env|key|secret|private'
grep -r "0x[a-fA-F0-9]{64}" packages/
grep -rE "g\.alchemy\.com/v2/[A-Za-z0-9]" packages/
```

`.gitignore` must include: `.env`, `.env.*`, `*.key`, `*.pem`, `broadcast/`, `cache/`

**If you accidentally commit a secret:** assume compromise. Transfer all funds immediately. Rotate key. Clean git history with `git filter-repo`.

Never use public RPCs. Always Alchemy with API key. If `ALCHEMY_API_KEY` is missing, STOP.

---

## Reference: Model Routing

| Stage | Model tier | Rationale |
|-------|-----------|-----------|
| Spec parsing, env checks, shell commands | Cheap (Haiku/Minimax) | No creativity needed |
| Architecture plan, spec verification, audits, frontend QA | Medium (Sonnet) | Reasoning required, fixable output |
| Contract code gen, complex frontend | Expensive (Opus) | Solidity correctness is critical, hard to fix |
| Deploy scripts, tests | Medium (Sonnet) | Templated patterns |

---

## Reference: Retry & Fix Protocol

Every stage that produces output follows this loop:

```
1. Do the thing
2. Verify the thing compiled/ran
3. If failed: diagnose error, fix, retry (max 3 attempts)
4. Verify the thing does what it's supposed to
5. If wrong: identify what's wrong, fix, re-verify (max 3 attempts)
6. If still failing after retries: STOP, report what failed, escalate
```

**Auto-fix strategies:**

| Error | Fix |
|-------|-----|
| Solidity compiler error | Parse error, send broken file + error to LLM for fix |
| Forge test failure | Diagnose if bug is in contract or test, fix the right one |
| Module not found (npm) | `yarn add <package>`, retry |
| Module not found (local) | Fix import path using discovery.json |
| TypeScript error | Send file + error to LLM for fix |
| Deploy revert | Check constructor args against discovery.json |
| IPFS no CID | Check if debug pages removed, retry |

---

## Reference: File Locations

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

---

## Reference: Commands

```bash
# Development
yarn chain                         # Start local Anvil (empty chain)
yarn fork --network base           # Fork Base mainnet (preferred)
cast rpc anvil_setIntervalMining 1 # Enable block mining on fork
yarn deploy                        # Deploy contracts locally
yarn start                         # Start frontend

# Quality
forge test -vvv                    # Run all contract tests
forge test --match-test testFuzz   # Fuzz tests only
slither packages/foundry/contracts/  # Security analysis
yarn lint && yarn format           # Code quality
yarn next:build                    # Build frontend

# Production
yarn deploy --network base         # Deploy to live network
yarn verify --network base         # Verify on block explorer
yarn ipfs                          # Deploy frontend to IPFS
yarn vercel:yolo --yes --prod      # Deploy frontend to Vercel
```

---

## Reference: Import Paths

```typescript
// Scaffold hooks (CORRECT names — not useScaffoldContractRead/Write)
import { useScaffoldReadContract, useScaffoldWriteContract } from "~~/hooks/scaffold-eth";
import { useScaffoldEventHistory } from "~~/hooks/scaffold-eth";

// UI components
import { Address, AddressInput, Balance, EtherInput } from "@scaffold-ui/components";

// Connect button
import { RainbowKitCustomConnectButton } from "~~/components/scaffold-eth";

// Utilities
import { notification, getParsedError } from "~~/utils/scaffold-eth";

// Viem
import { formatEther, parseEther, formatUnits, parseUnits } from "viem";
```

---

## Reference: Skill URLs

All skills at `https://ethskills.com/<name>/SKILL.md`:

| Skill | When to read |
|-------|-------------|
| `ship` | Before starting any dApp — architecture, archetypes, anti-patterns |
| `orchestration` | Before planning — three-phase system, secret management |
| `concepts` | When designing incentive mechanisms |
| `security` | Before writing any contract |
| `testing` | Before writing tests |
| `openzeppelin` | When using OZ contracts |
| `erc-721` | When building NFTs |
| `standards` | For ERC-8004, x402, EIP-7702, EIP-3009 |
| `gas` | When estimating costs or choosing chains |
| `l2s` | When choosing or deploying to L2 |
| `tools` | When setting up dev environment |
| `wallets` | When handling keys, multisig, account abstraction |
| `frontend-ux` | Before building any UI with wallet interaction |
| `frontend-playbook` | Before deploying frontend (IPFS, Vercel, ENS) |
| `qa` | After deployment, before sharing |
| `audit` | For high-value contract security review (20 parallel domains) |
| `indexing` | When you need historical onchain data |
| `money-legos` | When composing with DeFi protocols |

SE2 docs: `https://docs.scaffoldeth.io/SKILL.md`
SE2 agents: `https://github.com/scaffold-eth/scaffold-eth-2/blob/main/AGENTS.md`
