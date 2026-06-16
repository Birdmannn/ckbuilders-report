# FreightOnNervos - Weekly Development Report
**Reporting Period**: Since commit `b5d6126`  
**Project**: FreightOnNervos - Decentralized Campaign Platform on CKB  
**Branch**: v3

---

## Executive Summary

This reporting period focused on hardening the campaign lifecycle after the recent raffle work, while also introducing the project’s first dedicated campaign detail page and improving the internal identity model used to keep off-chain campaign records aligned with mutable on-chain state.

The most important engineering work this week happened around validation and debugging: sprint-style campaign durations were introduced by lowering the contract minimum duration and then aligning the frontend authoring flow to the exact same constraints. That effort uncovered how tightly coupled the UI, transaction builders, contract validation, and deployment artifacts are. In parallel, settlement debugging around the Share2 flow exposed the importance of feeding the contract the exact participant set it expects, especially once raffle participation is represented through verified participant cells.

On the product side, the feed evolved from a pure list of compact cards into a system that now has the foundations for deeper reading and interaction via a dedicated campaign detail route, reusable card surfaces, comment panels, persistent header behavior, and a more polished skeleton-loading experience.

**Key Metrics**:
- 6 commits merged since `b5d6126`
- 20 primary files changed across contract, frontend, deployment, and tests
- 2,089 lines added
- 451 lines removed
- 3 major workstreams advanced in parallel: sprint-campaign validation, campaign detail UX, and settlement debugging
- 2 new frontend shared modules/components introduced for stability and reuse
- 1 new reporting artifact added for the current development period

---

## 1. Sprint Campaign Support and Validation Alignment

### 1.1 Lowering Campaign Duration for Sprint Freights

**Problem Identified**: The contract originally enforced a minimum task duration that was too large for sprint-style campaigns. The product direction needed to support freights that run for much shorter periods.

**Solution Implemented**:
- Lowered the contract-side minimum duration from the earlier longer threshold to support sprint-style freights
- Began aligning the frontend campaign-creation flow to the same timing rules so invalid authoring values would be blocked before transaction submission
- Updated timing controls toward hour+minute-based input rather than forcing awkward fractional-hour entry

**Primary Files**:
- `contracts/freight/src/utils.rs`
- `frontend/app/create/_components/CreateCampaignModalContent.tsx`
- `frontend/lib/campaignValidation.ts`
- `tests/src/tests.rs`

**Impact**:
- Opened the door for time-sensitive “sprint” campaigns
- Reduced the mismatch risk between what the UI allows and what the contract accepts
- Made timing behavior more intentional instead of implicitly controlled by float parsing and rounding

### 1.2 Centralized Campaign Validation in the Frontend

**Problem Identified**: Campaign creation rules were scattered across the UI and submit path. After the timing UI changed, this led to frontend/contract drift and campaign submission failures.

**Solution Implemented**:
- Introduced a shared frontend validation/normalization layer for campaign creation
- Centralized timing normalization, summary validation, raffle math checks, and reward-count checks in one reusable module
- Refactored the create flow so submission consumes normalized values rather than relying on ad hoc float conversion at the point of transaction assembly

**Primary Files**:
- `frontend/lib/campaignValidation.ts`
- `frontend/app/create/_components/CreateCampaignModalContent.tsx`

**Impact**:
- Reduced duplicated validation logic
- Made the create flow easier to reason about during debugging
- Improved confidence that the app is preparing contract-valid parameters before signing and submission

---

## 2. Contract and Deployment Evolution

### 2.1 Contract Refresh and Deployment Metadata Update

**Problem Identified**: Contract changes alone were not sufficient; deployment metadata and configured code-hash references also had to move in sync or the app would keep interacting with stale assumptions.

**Solution Implemented**:
- Refreshed deployment metadata and migration artifacts after contract updates
- Updated the configured frontend contract reference to point at the newer deployed contract state
- Kept the deployment records synchronized with the evolving contract behavior

**Primary Files**:
- `deployment/deploy-fresh.toml`
- `deployment/migration-fresh/2026-06-15-142354.json`
- `deployment/txs/deploy-fresh-info.json`
- `frontend/lib/contract.ts`

**Impact**:
- Reduced confusion between source changes and actually deployed contract behavior
- Improved confidence that frontend calls target the intended contract version
- Reinforced the need to treat contract deployment state as part of the product surface, not just an afterthought

---

## 3. Campaign Identity and Off-Chain Record Stability

### 3.1 Moving Away from Mutable Outpoint Identity

**Problem Identified**: Some campaign identity lookups depended too directly on mutable transaction/outpoint assumptions. This is fragile once campaigns evolve through multiple on-chain state transitions while still needing to match the same off-chain record.

**Solution Implemented**:
- Introduced a stable campaign identity layer for indexing and lookup
- Added shared utilities for building campaign record indexes and finding records by stable identifiers rather than only by mutable outpoint references
- Extended campaign record APIs and draft flows to store and use the new identity-related fields

**Primary Files**:
- `frontend/lib/campaignIdentity.ts`
- `frontend/app/api/campaign-records/route.ts`
- `frontend/app/api/campaign-records/[id]/route.ts`
- `frontend/app/api/campaign-records/drafts/route.ts`
- `frontend/app/create/_components/CreateCampaignModalContent.tsx`
- `frontend/app/page.tsx`
- `frontend/app/campaign/[campaignId]/page.tsx`
- `frontend/lib/transactions.ts`

**Impact**:
- Improved resilience of campaign-to-record matching
- Reduced dependence on brittle mutable references
- Prepared the app for richer detail views, settlement flows, and future state transitions without losing campaign identity continuity

---

## 4. Dedicated Campaign Detail Experience

### 4.1 Extracting Reusable Campaign Surface and Comment Rendering

**Problem Identified**: The feed page had grown a large inline card implementation that mixed compact presentation, social actions, and comment behavior. That made it difficult to add a focused detail experience cleanly.

**Solution Implemented**:
- Extracted a reusable campaign card surface component with feed/detail presentation modes
- Added a reusable comments panel for rendering campaign discussions outside the feed context
- Began cleaning up the home page so the feed and a dedicated detail surface can share the same visual building blocks

**Primary Files**:
- `frontend/app/_components/CampaignCardSurface.tsx`
- `frontend/app/_components/CampaignCommentsPanel.tsx`
- `frontend/app/page.tsx`
- `frontend/app/styles/campaign.css`

**Impact**:
- Reduced UI duplication risk
- Created a clearer separation between feed preview and full-detail presentation
- Made future detail-page iteration much safer than continuing to expand one giant inline card component

### 4.2 Adding the First Dedicated Campaign Detail Route

**Problem Identified**: The user needed a way to open a freight/campaign for deeper reading without forcing the home feed into a split master-detail layout that would fight the existing mobile-first design.

**Solution Implemented**:
- Added a dedicated campaign detail page route under the app directory
- Reused the extracted campaign card surface in detail mode
- Added right-side comment presentation and a more focused full-post reading experience
- Wired feed card navigation to route into the detail view rather than relying on an overlay-first interaction

**Primary Files**:
- `frontend/app/campaign/[campaignId]/page.tsx`
- `frontend/app/_components/CampaignCardSurface.tsx`
- `frontend/app/_components/CampaignCommentsPanel.tsx`
- `frontend/app/page.tsx`
- `frontend/app/styles/campaign.css`

**Impact**:
- Introduced a route-based detail experience aligned with the app’s feed-first UX
- Improved extensibility for full-post interactions
- Opened the door for richer per-campaign navigation and direct-link behavior

### 4.3 Keeping the Shared Header Present on the Detail Page

**Problem Identified**: Once the detail page was introduced, it needed to preserve the same core top-level controls and product framing as the home page rather than feeling like a disconnected sub-app.

**Solution Implemented**:
- Brought the info icon and wallet disconnect controls onto the campaign detail route
- Preserved the header through loading and loaded states
- Adjusted the detail-page header width so it matched the compact home-page header rather than stretching across the wider detail shell
- Replaced plain loading text with a skeleton-based detail loader for a more consistent feel

**Primary Files**:
- `frontend/app/campaign/[campaignId]/page.tsx`
- `frontend/app/_components/FreightInfoModal.tsx`
- `frontend/app/styles/campaign.css`
- `frontend/app/styles/header.css`
- `frontend/app/styles/wallet.css`

**Impact**:
- Preserved continuity between feed and detail navigation
- Reduced the feeling that clicking into a freight leaves the main application frame
- Improved perceived polish during loading transitions

---

## 5. Feed and Visual Interaction Refinements

### 5.1 Purple Blink and Settlement-State Visual Recovery

**Problem Identified**: After extracting the shared campaign surface, the pending settlement indicator on raffle cards lost its intended purple blinking behavior.

**Solution Implemented**:
- Reconnected the “take {reward count}” display to the same pending/settled state classes used by the blink styling
- Restored the visual cue that tells users a raffle still has undistributed rewards

**Primary Files**:
- `frontend/app/_components/CampaignCardSurface.tsx`
- `frontend/app/styles/campaign.css`

**Impact**:
- Recovered an important settlement-state affordance in the feed
- Helped maintain visual parity during the component extraction process
- Reduced the chance that unresolved raffle settlements go unnoticed

### 5.2 Pinned Header Behavior on the Freight Detail Page

**Problem Identified**: The new freight detail page needed the same persistent top-level navigation feel as the home feed, but the first pass stretched and recontextualized the header too much.

**Solution Implemented**:
- Pinned the header on the freight detail page
- Constrained it back to the same compact home-page width and layout behavior
- Preserved the visible info and wallet actions even when entering a specific freight

**Primary Files**:
- `frontend/app/campaign/[campaignId]/page.tsx`
- `frontend/app/styles/campaign.css`

**Impact**:
- Improved continuity of the app shell across routes
- Reduced perceived context switching when navigating from feed to detail
- Reinforced the idea that the detail page belongs to the same browsing experience

---

## 6. Settlement Debugging and On-Chain Learning

### 6.1 Debugging Share2 Settlement Failures

**Problem Identified**: Clicking Share2 for raffle settlement began failing with `TransactionFailedToVerify`, and the initial assumption about the exact contract error mapping was uncertain.

**Solution Implemented**:
- Traced the batch-deliver path from frontend Share2 handling through transaction assembly into the Rust contract validation layers
- Verified the contract’s exact error enum mapping rather than relying on guesswork
- Audited participant collection and discovered that the frontend settlement preview/build path needed to align more tightly with the contract’s verified-only participant expectations
- Updated participant fetching on the frontend so settlement logic works from the contract-eligible participant set
- Added or strengthened focused settlement tests around invalid conditions

**Primary Files**:
- `contracts/freight/src/errors.rs`
- `contracts/freight/src/instructions.rs`
- `contracts/freight/src/validations.rs`
- `frontend/lib/transactions.ts`
- `frontend/app/page.tsx`
- `tests/src/tests.rs`

**Impact**:
- Improved parity between frontend preview logic and contract-side settlement rules
- Reduced ambiguity when debugging settlement failures
- Surfaced where the frontend had been too optimistic about which participant cells were eligible for payout

### 6.2 Clarifying Ticket-Purchase Verification Semantics

**Problem Identified**: During settlement debugging, an important question emerged: does buying a raffle ticket actually verify the participant on-chain, or is there another hidden verification step?

**Solution Implemented**:
- Inspected the raffle branch of `verify_participant(...)`
- Confirmed that the raffle ticket purchase flow does create a participant output with verified status and that the contract validates that participant was added correctly
- Used that insight to rule out one whole class of settlement hypotheses and focus instead on winner/input/output mismatches

**Primary Files**:
- `contracts/freight/src/instructions.rs`
- `frontend/lib/transactions.ts`
- `frontend/app/page.tsx`

**Impact**:
- Narrowed the real settlement debugging search space
- Reinforced understanding of the raffle lifecycle from purchase through settlement
- Improved confidence that the bug was in settlement selection/shape, not in participant verification during purchase

---

## 7. Challenges and Solutions

### Challenge 1: Supporting sprint campaigns without breaking validation parity
**Solution**: Centralized create-campaign normalization and validation, then aligned contract rules, frontend timing controls, and tests around the same minimum-duration semantics.

### Challenge 2: Debugging contract changes when tests still appear to use old behavior
**Solution**: Reconfirmed how test artifacts and deployment metadata relate to contract source, and treated deployment/build state as part of the debugging process rather than assuming source edits alone explain runtime behavior.

### Challenge 3: Preventing campaign identity drift as the app grows more stateful
**Solution**: Introduced a stable campaign identity/indexing layer so off-chain records no longer rely purely on fragile mutable references.

### Challenge 4: Extracting a new detail experience without destabilizing the feed
**Solution**: Broke the work into reusable UI surfaces, kept the feed compact, and moved toward a route-based detail view instead of forcing a split-shell architecture onto a feed-first app.

### Challenge 5: Chasing settlement failures through frontend and contract layers simultaneously
**Solution**: Used the contract as the source of truth, audited participant eligibility, corrected the frontend participant set to match verified-only contract expectations, and expanded focused test coverage.

---

## 8. What Was Learned

1. **Validation changes are only safe when the UI, transaction builder, contract, and tests move together**: changing one layer in isolation is usually what creates the hardest bugs.
2. **Stable identity is a prerequisite for richer UX**: once campaigns can be revisited, commented on, updated, and settled over time, brittle outpoint-only identity becomes a real product limitation.
3. **Route-based detail views fit this app better than persistent split layouts**: the current architecture is feed-first and mobile-friendly, and trying to force master-detail behavior would have created unnecessary complexity.
4. **Settlement debugging depends on frontend/contract participant parity**: if the frontend previews winners from a participant set the contract would reject, the transaction can look correct in the UI and still fail on-chain.
5. **Component extraction often reveals hidden coupling**: moving card presentation into shared components exposed how much behavior was tied to the original inline card, but it also made the UI architecture healthier going forward.
6. **Contract error interpretation needs discipline**: reading the enum and then verifying the actual failing path/test behavior is better than reasoning from an assumed numeric code alone.

---

## 9. Impact Assessment

### Quantitative Impact
- **Commit Volume**: 6 commits since `b5d6126`
- **Code Change Volume**: 2,089 lines added / 451 removed
- **Files Touched**: 20 files in the inspected range
- **Major Areas Improved**: sprint-campaign support, validation parity, stable campaign identity, route-based campaign detail UX, settlement debugging, and deployment alignment
- **New Reusable Surfaces**: 2 extracted UI components and 1 dedicated detail route
- **New Shared Logic**: 2 important shared frontend modules (`campaignValidation`, `campaignIdentity`)

### Qualitative Impact
- **Product Flexibility**: campaigns can evolve toward faster-lived sprint use cases
- **UX Maturity**: the platform now has the beginnings of a real detail experience rather than only a feed
- **Reliability**: frontend/contract parity improved both for creation and settlement
- **Debugging Depth**: this period significantly improved understanding of how on-chain failures surface and how to investigate them end-to-end

---

## 10. Next Week's Priorities

### Immediate Focus
1. Finish wiring full interactive comment/composer parity into the campaign detail page
2. Continue hardening Share2 settlement until a completed raffle can be settled reliably end-to-end
3. Clean up remaining temporary/debug code and warning-level debt exposed during extraction and contract debugging

### Strategic Initiatives
1. Strengthen exact-error settlement coverage so future regressions are easier to pinpoint
2. Continue moving shared campaign/record lookup logic out of the feed page into reusable helpers
3. Decide which feed actions should stay feed-only versus also appearing in the dedicated detail experience

### Documentation
1. Keep settlement debugging notes aligned with the actual contract error mapping and participant-validation behavior
2. Document the new stable campaign identity model and why it replaced mutable outpoint-centric lookup assumptions

---

## 11. Conclusion

This week was less about launching one huge new feature and more about making FreightOnNervos structurally stronger. Sprint-style campaign support, centralized validation, stable campaign identity, and a dedicated campaign detail page all pushed the app toward a more durable architecture.

Just as importantly, the debugging work around settlement and contract validation taught us a lot about where frontend assumptions can quietly drift away from on-chain truth. That learning is already paying off: the app now has better shared validation, better participant filtering for settlement, and a clearer path for continuing to harden the raffle lifecycle without guessing blindly.

---

**Report Prepared By**: Development Team  
**Date**: June 16, 2026  
**Commit Range**: `b5d6126..HEAD`