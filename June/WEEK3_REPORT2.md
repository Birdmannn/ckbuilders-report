# FreightOnNervos - Weekly Development Report
**Reporting Period**: Since commit `80095ca`  
**Project**: FreightOnNervos - Decentralized Campaign Platform on CKB  
**Branch**: v3

---

## Executive Summary

This reporting period focused on three intertwined tracks: strengthening raffle settlement correctness, reclaiming operational control over contract deployment capacity, and aggressively cleaning up the frontend architecture so the home feed and detail flows are easier to maintain. The most important technical finding was a deeper raffle settlement identity bug: participants were being tied to a mutable campaign outpoint, while settlement later searched against the newest campaign outpoint after state transitions. That meant legitimate buyers could disappear from settlement even though they had valid verified participant cells.

At the same time, the deployment workflow was improved through a reclaim-capacity script and accompanying documentation, helping reduce dependence on faucet-distributed test CKB. On the frontend side, the previously monolithic `frontend/app/page.tsx` underwent major extraction into components, hooks, and shared helpers, creating a more maintainable structure and reducing page-level coupling.

**Key Metrics**:
- 6 commits in the inspected range since `80095ca`
- 30 files changed across contract, frontend, docs, scripts, deployment metadata, and tests
- 2,937 lines added
- 1,826 lines removed
- 5 new focused frontend hooks introduced
- 5 new extracted frontend components introduced
- 3 documentation artifacts produced around operations, dev workflow, and raffle identity debugging

---

## 1. Contract Logic and Raffle Rule Evolution

### 1.1 Winner-Count Logic Was Relaxed to an Upper Bound

**Problem Identified**: In the raffle settlement path, the previous logic treated the configured reward count as an exact requirement. If a raffle was configured to take 5 winners but only 3 verified participants existed, settlement failed instead of rewarding those 3 participants.

**Solution Implemented**:
- Contract winner-count logic was adjusted so configured winners act as an upper bound rather than an exact required count
- Frontend settlement preview and payout logic were aligned to the same rule so the UI and contract no longer disagreed
- Focused tests were added and verified after rebuilding the contract artifact used by the test harness

**Primary Files**:
- `contracts/freight/src/instructions.rs`
- `frontend/lib/transactions.ts`
- `tests/src/tests.rs`

**Impact**:
- The system now reflects the product rule that available valid participants should still win even when the configured reward count is larger
- Removed an avoidable settlement failure mode
- Improved alignment between frontend preview behavior and contract enforcement

### 1.2 Raffle Settlement Error Mapping Was Clarified

**Problem Identified**: Settlement failures surfaced as numeric contract error codes, and the original interpretation of those codes was not always reliable from memory alone.

**Solution Implemented**:
- Error code mapping was rechecked directly from the contract enum
- Focused settlement tests were run to observe the actual error path during failure cases
- This debugging clarified which issues were true validation problems versus UI-side assumptions

**Primary Files**:
- `contracts/freight/src/errors.rs`
- `contracts/freight/src/instructions.rs`
- `tests/src/tests.rs`

**Impact**:
- Reduced guesswork during settlement debugging
- Improved confidence in how on-chain failures should be interpreted

---

## 2. Major Raffle Participant Identity Bug Discovery

### 2.1 The Core Bug

**Problem Identified**: Participant cells were originally linked to raffles using the campaign cell’s **current outpoint** at the time of ticket purchase. Because campaign cells are consumed and recreated during lifecycle transitions such as activation and later state updates, the current outpoint was not stable.

That created a dangerous mismatch:
- participant creation used the campaign outpoint that existed at purchase time
- settlement later searched for participants using the latest campaign outpoint
- if the campaign cell had been recreated in between, legitimate participant cells no longer matched

This could cause settlement to behave as though no eligible winners or no eligible participants existed, even when tickets had actually been purchased.

### 2.2 What Was Meant to Happen

The intended product behavior was:
- a raffle remains the same logical raffle across state transitions
- participants remain attached to that raffle regardless of cell recreation
- settlement should always find valid verified buyers for the same logical campaign

### 2.3 The Fix Direction

**Solution Implemented / In Progress**:
- Began migrating participant identity away from mutable outpoint linkage toward a stable identity based on campaign fields that do not change across recreations
- Updated participant schema, frontend encoding, and participant lookup paths toward stable identity fields
- Adjusted contract validation logic to match participants against stable campaign identity instead of the latest outpoint

**Primary Files**:
- `contracts/freight/src/types.rs`
- `contracts/freight/src/utils.rs`
- `contracts/freight/src/validations.rs`
- `contracts/freight/src/instructions.rs`
- `frontend/lib/encoding.ts`
- `frontend/lib/transactions.ts`
- `tests/src/tests.rs`
- `tests/src/tests/raffle_e2e.rs`

**Impact**:
- Corrects the underlying conceptual model for participant-to-campaign linkage
- Prevents valid buyers from becoming invisible after campaign cell recreation
- Establishes stable identity as the right long-term contract/frontend model

### 2.4 Documentation Produced

A dedicated note was written to capture this bug clearly:
- `docs/RAFFLE_PARTICIPANT_IDENTITY_BUG.md`

That document explains:
- the original logic
- intended behavior
- actual broken behavior
- why the old approach was wrong
- what was learned from the debugging process

---

## 3. Deployment Capacity Recovery and Operational Improvements

### 3.1 Reclaiming Capacity from an Obsolete Contract Cell

**Problem Identified**: Repeated public testnet deployments were locking capacity into obsolete freight contract cells, making development increasingly constrained by faucet limits.

**Solution Implemented**:
- Identified the exact fresh deployment outpoint from deployment metadata
- Verified that the contract cell was still live on testnet
- Created a reclaim helper script to build, inspect, sign, and optionally send a reclaim transaction
- Added safety checks to ensure the target live cell matches the expected owner lock args before reclaiming
- Successfully reclaimed the fresh deployment capacity back to the owner address

**Primary Files**:
- `scripts/reclaim-contract-cell.sh`
- `deployment/migration-fresh/2026-06-19-171848.json`
- `deployment/txs/deploy-fresh-info.json`
- `frontend/lib/contract.ts`

**Impact**:
- Recovered a large amount of testnet capacity from an obsolete deployment
- Reduced faucet dependency pressure
- Established a reusable reclaim workflow for future obsolete contract cells

### 3.2 Operational Documentation

Two operational docs were written to capture the broader lessons:
- `docs/CONTRACT_CELL_CAPACITY_AND_LOCAL_FRONTEND_TESTING.md`
- `docs/DEVNET_FRONTEND_SIGNER_WORKFLOW.md`

These explain:
- how contract/code cells lock capacity on CKB
- when reclaiming is safe vs dangerous
- why browser-wallet-based local devnet testing is awkward
- how to think about local contract iteration, local frontend work, and public testnet validation as separate loops

---

## 4. Frontend Architectural Refactor

### 4.1 `page.tsx` Was Split Into Smaller Components

**Problem Identified**: The homepage (`frontend/app/page.tsx`) had grown into a large monolithic file containing feed rendering, card behavior, modal rendering, wallet state, and multiple orchestration layers in one place.

**Solution Implemented**:
- Extracted multiple UI components from `page.tsx`
- Turned the page into more of a composition root rather than a single implementation surface

**New Components Added**:
- `frontend/app/_components/MountablesPanel.tsx`
- `frontend/app/_components/CampaignFeedHeaderBar.tsx`
- `frontend/app/_components/CampaignList.tsx`
- `frontend/app/_components/CampaignCard.tsx`
- `frontend/app/_components/CampaignFeedSection.tsx`

**Impact**:
- Reduced homepage complexity
- Improved readability and maintainability
- Made it much easier to evolve isolated pieces of feed and detail behavior independently

### 4.2 Page Logic Was Split into Focused Hooks

**Solution Implemented**:
- Introduced focused hooks for different state/orchestration responsibilities instead of one giant page component or one oversized all-purpose hook

**New Hooks Added**:
- `frontend/app/_hooks/useCampaignFeed.ts`
- `frontend/app/_hooks/useWalletInfo.ts`
- `frontend/app/_hooks/useInfoModalState.ts`
- `frontend/app/_hooks/useTicketPurchaseFlow.ts`
- `frontend/app/_hooks/useCampaignCardState.ts`

**Impact**:
- Reduced state entanglement in `page.tsx`
- Improved testability and future refactor safety
- Created clearer boundaries between display, asynchronous behavior, and chain interactions

### 4.3 Shared Pure Helpers Were Extracted

**Solution Implemented**:
- Extracted pure reusable helpers from page-level code into shared frontend modules

**New Helper Modules Added**:
- `frontend/lib/campaignDisplay.ts`
- `frontend/lib/clipboard.ts`

These handle common concerns like:
- display/status derivation
- amount formatting
- creator decoding
- wallet/network display
- clipboard behavior

**Impact**:
- Reduced duplication across feed and detail routes
- Encouraged a cleaner separation between pure logic and React stateful orchestration

---

## 5. Campaign Detail Experience and Width Transition Work

### 5.1 Dedicated Campaign Detail Route Continued to Evolve

**Problem Identified**: The project needed a more focused detail experience for campaigns beyond compact feed cards.

**Solution Implemented**:
- Continued work on the dedicated campaign detail route
- Reused the extracted campaign card surface in detail mode
- Kept the header visible across loading and loaded states
- Added skeleton-based loading rather than plain loading text

**Primary Files**:
- `frontend/app/campaign/[campaignId]/page.tsx`
- `frontend/app/_components/CampaignCardSurface.tsx`
- `frontend/app/_components/CampaignCommentsPanel.tsx`
- `frontend/app/styles/campaign.css`

### 5.2 Width-Transition Refinement Began

**Problem Identified**: Clicking a freight should feel spatially continuous rather than abruptly jumping between the feed and detail route.

**Solution Implemented**:
- Introduced shared width tokens and shell classes for feed/detail alignment
- Continued staging a feed-to-detail width expansion model where the shell/header can visually widen before or during navigation

**Primary Files**:
- `frontend/app/styles/campaign.css`
- `frontend/app/page.tsx`
- `frontend/app/campaign/[campaignId]/page.tsx`

**Impact**:
- Established a better visual language for route transitions
- Improved continuity between compact feed cards and elongated detail presentation

---

## 6. Settlement Builder and Wallet Error Hardening

### 6.1 Wallet Null-Output Error Investigation

**Problem Identified**: The wallet threw a `TypeError: null is not an object (evaluating 'c.output')` on transaction paths.

**Solution Implemented**:
- Investigated the frontend transaction builders and identified sparse/placeholder output construction as a likely cause
- Removed the most suspicious sparse output shape from the raffle purchase builder
- Tightened related transaction reliability logic in nearby flows

**Primary Files**:
- `frontend/lib/transactions.ts`
- `frontend/app/page.tsx`

**Impact**:
- Reduced the risk of wallet-side transaction normalization failures
- Improved transaction builder explicitness and reliability

### 6.2 Settlement Zero-Divisor Guard

**Problem Identified**: Share2 settlement could still fail with a zero-divisor style error when no eligible winners were found.

**Solution Implemented**:
- Added frontend-side guards so settlement fails gracefully when effective winner count is zero
- Improved the user-facing message rather than surfacing a raw invalid divisor failure

**Primary Files**:
- `frontend/lib/transactions.ts`
- `frontend/app/_hooks/useCampaignCardState.ts`

**Impact**:
- Improved failure behavior while the deeper participant-identity fix was being worked through
- Reduced confusing settlement UX during debugging

---

## 7. Challenges and Solutions

### Challenge 1: Hidden contract/frontend drift after stateful raffle changes
**Solution**: Treated the contract as the source of truth, then aligned frontend settlement preview and participant lookup to what the contract actually accepts.

### Challenge 2: Contract deployments consuming too much testnet capacity
**Solution**: Built a reclaim flow and reclaimed capacity from an obsolete fresh deployment instead of accepting faucet scarcity as a fixed constraint.

### Challenge 3: Browser-wallet assumptions made local iteration painful
**Solution**: Documented a better split between local contract testing, local frontend work, and future dev-signer support for devnet use.

### Challenge 4: `page.tsx` refactor created dependency tangles
**Solution**: Rejected the oversized “do everything” modal hook approach and instead moved toward smaller focused hooks, fixing declaration order and event-handler coupling issues incrementally.

### Challenge 5: Participants disappearing from settlement despite valid purchases
**Solution**: Identified the identity mismatch between mutable campaign outpoints and stable logical raffle identity, then began the schema migration toward stable participant identity.

---

## 8. What Was Learned

1. **Mutable outpoints are dangerous as long-term business identity keys** when cells are expected to be consumed and recreated.
2. **Stable identity should be designed into both contract and frontend together**; one-sided fixes are not enough.
3. **Operational tooling matters as much as contract logic** on CKB, especially when capacity is locked into code cells and faucet access is limited.
4. **Focused hooks are safer than oversized abstractions during large frontend refactors** because they reduce coupling and make bugs easier to localize.
5. **Visible runtime errors often hide deeper identity problems**: what first looks like a math or wallet bug can actually be a data-model bug.

---

## 9. Impact Assessment

### Quantitative Impact
- **Commit Volume**: 6 commits since `80095ca`
- **Code Change Volume**: 2,937 insertions / 1,826 deletions in the inspected range
- **Files Touched**: 30 files across contract, frontend, tests, scripts, docs, and deployment metadata
- **New Hooks Added**: 5
- **New Components Added**: 5
- **New Documentation Files Added**: 3
- **Reclaimed Capacity**: one obsolete fresh deployment consumed and returned to the owner address

### Qualitative Impact
- **Correctness**: settlement logic and participant identity are moving toward a sounder long-term model
- **Developer Experience**: capacity recovery and devnet/frontend strategy are now much clearer
- **Maintainability**: homepage architecture is significantly improved through decomposition into components, hooks, and helpers
- **Operational Maturity**: the project now has real procedures for reclaiming code-cell capacity and thinking about local iteration more sustainably

---

## 10. Next Week's Priorities

### Immediate Focus
1. Finish the stable participant identity migration completely and verify it end-to-end in real flows
2. Clean the remaining lint warnings in `CreateCampaignModalContent.tsx`
3. Continue refining the campaign detail route and width-transition UX

### Strategic Initiatives
1. Add or formalize a dev-only frontend signer path for local devnet use
2. Continue separating page-level orchestration into focused hooks where it adds real value
3. Expand stable-identity testing across refund and any remaining participant-related flows

### Documentation
1. Keep participant-identity migration notes synchronized with the final contract/frontend schema
2. Update operational docs with the finalized reclaim and local-development workflow after the next tooling pass

---

## 11. Conclusion

This period was about moving FreightOnNervos from a fragile, ad hoc phase into a more durable architecture and workflow. The work since `80095ca` did not just patch bugs — it exposed and corrected deeper assumptions about identity, settlement, deployment capacity, and frontend structure.

The biggest engineering lesson was that a CKB application with recreating state cells must be designed around **stable logical identity**, not just around whatever outpoint happens to exist at the moment. At the same time, the frontend was reshaped into a far healthier structure, and the repo now has better operational guidance for reclaiming contract capacity and escaping a faucet-bound development loop.

---

**Report Prepared By**: Development Team  
**Date**: June 23, 2026  
**Commit Range**: `80095ca..b4d88f3(HEAD)`