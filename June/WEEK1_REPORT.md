# FreightOnNervos - Weekly Development Report
**Reporting Period**: Since commit `14a1cd9`  
**Project**: FreightOnNervos - Decentralized Campaign Platform on CKB  
**Branch**: v3

---

## Executive Summary

This reporting period focused on turning the raffle flow into a much more complete full-stack feature, while also improving the core campaign feed experience, social interactions, wallet UX, modal persistence, and local developer workflow. The biggest step forward was the introduction of deterministic raffle settlement built on randomness commitment, reveal-driven winner selection, frontend settlement tooling, contract-side validation, and end-to-end testing.

Alongside that deeper raffle work, the frontend became significantly more polished and stateful: comments and campaign actions were persisted, ticket purchase flow gained a two-step activation path, wallet details became easier to inspect, campaign timing/status logic was refined, and the feed received multiple visual and interaction upgrades.

**Key Metrics**:
- 66 commits merged since `14a1cd9`
- 36 primary files changed across contract, frontend, deployment, docs, scripts, and tests
- 4,197 lines added
- 701 lines removed
- 3 major workstreams advanced in parallel: raffle lifecycle, frontend/social UX, and local devnet/testing support
- 1 substantial new end-to-end raffle lifecycle test added

---

## 1. Raffle Lifecycle and Deterministic Settlement

### 1.1 Randomness Commitment and Reveal Pipeline

**Problem Identified**: Raffle campaigns needed a safer and more auditable way to handle winner selection so outcomes could not be chosen after participants were known.

**Solution Implemented**:
- Added randomness commitment generation in the frontend using a 32-byte preimage and Blake2b hash
- Stored and surfaced randomness-related campaign fields in the app and off-chain campaign records
- Added support for randomness preimage handling during creation success and later settlement flows
- Extended campaign update payloads to carry `randomnessPreimage` and activation metadata

**Primary Files**:
- `frontend/lib/randomness.ts`
- `frontend/lib/transactions.ts`
- `frontend/app/create/_components/CreateCampaignModalContent.tsx`
- `frontend/app/api/campaign-records/[id]/route.ts`

**Impact**:
- Established the commit/reveal foundation for fairer raffle settlement
- Reduced ambiguity around how raffle randomness is generated and carried forward
- Enabled downstream deterministic settlement flows in both frontend and contract logic

### 1.2 Deterministic Winner Selection and Batch Delivery

**Problem Identified**: Raffle settlement required a deterministic winner-selection path that both frontend and contract could agree on, including ordering, shuffling, and reward distribution.

**Solution Implemented**:
- Added canonical participant ordering logic
- Derived campaign-specific shuffle seeds from revealed preimage + campaign identity
- Implemented deterministic shuffle with rejection sampling to avoid modulo bias
- Added frontend winner preview and contract-side winner validation for batch delivery
- Added settlement handling that distributes funds only to deterministic winners

**Primary Files**:
- `contracts/freight/src/instructions.rs`
- `contracts/freight/src/validations.rs`
- `frontend/lib/transactions.ts`
- `frontend/app/page.tsx`
- `tests/src/tests/raffle_e2e.rs`

**Impact**:
- Moved raffle rewards from a conceptual design toward a verifiable end-to-end implementation
- Tightened parity between frontend preview behavior and on-chain settlement checks
- Improved confidence that reward distribution follows a consistent rule set

### 1.3 Raffle Activation and Ticket Purchase Flow

**Problem Identified**: The first ticket buyer needed a reliable path to activate a raffle on-chain before buying, while later buyers should avoid repeating that activation step.

**Solution Implemented**:
- Added a lightweight API endpoint to mark campaigns as activated off-chain after activation tx submission
- Built a two-step purchase flow: activate first if needed, then buy the ticket
- Re-fetched campaign state after activation before continuing purchase
- Preserved better error/status feedback during the purchase sequence

**Primary Files**:
- `frontend/app/api/campaign-records/[id]/activate/route.ts`
- `frontend/lib/transactions.ts`
- `frontend/app/page.tsx`

**Impact**:
- Reduced race-condition-like confusion around first-entry raffle activation
- Improved purchase reliability for early participants
- Made raffle ticket entry feel more robust during live transitions from Created → Active

---

## 2. Contract Logic and On-Chain Validation Improvements

### 2.1 Expanded Freight Contract Behavior

**Problem Identified**: The contract needed stronger lifecycle coverage for raffle campaigns and more explicit validation of participant, settlement, and status-transition behavior.

**Solution Implemented**:
- Extended `verify_participant` with raffle-specific ticket-sale timing and deposit checks
- Added stronger batch-delivery validation for rewarded outputs
- Added deterministic winner selection logic inside contract execution
- Expanded support for randomness hash submission and forward-only campaign status updates
- Tightened refund and participant linkage validation logic

**Primary Files**:
- `contracts/freight/src/instructions.rs`
- `contracts/freight/src/validations.rs`
- `contracts/freight/src/types.rs`

**Impact**:
- Improved correctness guarantees around raffle participation and payout
- Reduced room for inconsistent settlement outputs
- Strengthened the on-chain contract as the ultimate source of truth for lifecycle transitions

### 2.2 Transaction Builder and Encoding Updates

**Problem Identified**: The frontend transaction layer needed to match the richer contract behavior for create, activate, ticket purchase, and reward distribution actions.

**Solution Implemented**:
- Expanded transaction helpers for status updates, ticket buying, and batch reward delivery
- Added deterministic winner preview helpers shared by the settlement UI
- Updated create-campaign encoding to carry randomness hash and reward count inputs
- Refined contract/client interaction to better reflect current campaign and participant state

**Primary Files**:
- `frontend/lib/transactions.ts`
- `frontend/lib/encoding.ts`
- `frontend/lib/contract.ts`

**Impact**:
- Reduced mismatch risk between app intent and contract expectations
- Made the frontend capable of driving a broader set of on-chain actions directly
- Improved maintainability by concentrating more lifecycle logic in typed helper functions

---

## 3. Frontend UX, Social Layer, and Interaction Improvements

### 3.1 Campaign Social Actions and Comment Persistence

**Problem Identified**: Campaign cards needed more social interactivity, and those actions needed to persist instead of behaving like purely local UI state.

**Solution Implemented**:
- Added comment composer behavior and discard-confirmation flow
- Persisted social metadata such as comments, likes, bookmarks, and reshares in campaign records
- Improved comment editing/submission behavior and card-level interaction states

**Primary Files**:
- `frontend/app/page.tsx`
- `frontend/app/api/campaign-records/[id]/route.ts`
- `frontend/app/api/campaign-records/route.ts`

**Impact**:
- Made campaign cards feel more like active posts than static listings
- Preserved lightweight social context around campaigns
- Improved continuity between page interaction and stored record state

### 3.2 Wallet and Info Modal Improvements

**Problem Identified**: Wallet information and informational UI were not yet polished enough for smooth repeated use, especially around address visibility, chain awareness, and success/error surfaces.

**Solution Implemented**:
- Improved connect-wallet page behavior
- Added richer wallet detail display including address, balance, chain label, and copy feedback
- Reworked `FreightInfoModal` into a more capable project/info shell
- Moved success and support content into the shared info-modal system
- Added project links and contract visibility inside the modal header

**Primary Files**:
- `frontend/app/page.tsx`
- `frontend/app/_components/FreightInfoModal.tsx`
- `frontend/app/styles/header.css`
- `frontend/app/styles/wallet.css`

**Impact**:
- Better wallet affordance and transparency for users
- Cleaner reuse of modal UX across help, success, purchase, and settlement contexts
- Stronger product framing around contract and project metadata

### 3.3 Create Flow Persistence and Raffle Authoring Refinements

**Problem Identified**: The create flow still had friction around modal step persistence, save-draft responsiveness, and raffle-specific review semantics.

**Solution Implemented**:
- Improved modal persistence across both create/review steps
- Reduced lag on save-draft interactions
- Refined raffle preview/review UX and argument handling
- Continued tightening draft snapshot behavior and create-flow state handling

**Primary Files**:
- `frontend/app/create/_components/CreateCampaignModalContent.tsx`
- `frontend/app/page.tsx`
- `frontend/app/styles/create-modal.css`

**Impact**:
- Smoother campaign authoring experience
- Better continuity when switching between modal states
- More natural raffle creation flow and follow-through into publish/settlement behavior

---

## 4. Campaign Feed, Timing Logic, and Visual Polish

### 4.1 Timing, Status, and Distribution Indicators

**Problem Identified**: Campaign status/timing needed clearer frontend derivation, and raffle settlement states needed to be more visible in the feed.

**Solution Implemented**:
- Refined countdown/status derivation logic for Created, Active, Completed, and Cancelled states
- Added settlement-specific glow/blinker cues for pending raffle distribution
- Synchronized visual timing for distribution indicators
- Replaced or refined iconography in some raffle-delay contexts

**Primary Files**:
- `frontend/app/page.tsx`
- `frontend/app/styles/campaign.css`
- `frontend/app/styles/animations.css`

**Impact**:
- Clearer state awareness directly from the campaign feed
- Better visual emphasis on raffles that still require reward distribution
- Improved readability of live time-based transitions

### 4.2 Header, Hover, and Card Interaction Refinement

**Problem Identified**: Feed interactions had layering, hover, and dismissal issues that made the UI feel less stable than intended.

**Solution Implemented**:
- Pinned the header and corrected header layering/z-index issues
- Fixed create-modal dismissal edge cases triggered by header interactions
- Tuned info hover behavior and tooltip layering
- Improved hover handling on campaign cards and related action states

**Primary Files**:
- `frontend/app/page.tsx`
- `frontend/app/styles/header.css`
- `frontend/app/styles/campaign.css`

**Impact**:
- More stable top-level navigation and overlays
- Fewer accidental interaction regressions
- Better perceived polish across the campaign list

### 4.3 Experimental Rainbow and Marquee Styling

**Problem Identified**: The project continued exploring a retro-highlight visual identity, but experimental work needed controlled surfaces and iteration.

**Solution Implemented**:
- Added and refined a rainbow preview page
- Iterated on rainbow border behavior and campaign highlight treatment
- Continued marquee/grid visual exploration for the mountables area
- Adjusted halo/highlight behavior to reduce over-styling

**Primary Files**:
- `frontend/app/rainbow-button-preview/page.tsx`
- `frontend/app/styles/rainbow-preview.css`
- `frontend/app/styles/marquee.css`
- `frontend/app/styles/campaign.css`

**Impact**:
- Preserved room for visual experimentation without blocking core product work
- Helped shape a more distinct FreightOnNervos UI identity
- Separated playful iteration from more functional feed and raffle work

---

## 5. Testing, Documentation, and Developer Workflow

### 5.1 Raffle Documentation and End-to-End Testing

**Problem Identified**: The new raffle model required both a written fairness explanation and a test that exercised the full lifecycle.

**Solution Implemented**:
- Added dedicated documentation describing commitment, reveal, participant ordering, deterministic shuffling, and reward distribution
- Added a substantial raffle e2e test covering creation, randomness commitment, activation, ticket purchase, completion, and delivery
- Hooked the new raffle test into the broader Rust test suite

**Primary Files**:
- `docs/RAFFLE_WINNER_SELECTION.md`
- `tests/src/tests/raffle_e2e.rs`
- `tests/src/tests.rs`

**Impact**:
- Made the fairness model easier to explain and audit
- Added stronger regression protection around the most complex path in the product
- Improved confidence in future raffle-related iteration

### 5.2 Local Devnet and Project Tooling

**Problem Identified**: Contract iteration and end-to-end testing benefit from a repeatable local devnet workflow and faster operational setup.

**Solution Implemented**:
- Added a more complete `scripts/devnet.sh` workflow for start/stop/clean of local node/miner processes
- Added Makefile targets to wrap devnet lifecycle commands
- Refreshed deployment metadata and migration artifacts to match ongoing contract iteration

**Primary Files**:
- `scripts/devnet.sh`
- `Makefile`
- `deployment/txs/deploy-fresh-info.json`
- `deployment/migration-fresh/2026-06-08-070127.json`

**Impact**:
- Reduced friction for local contract testing and deployment repetition
- Improved reproducibility of the dev environment
- Supported the larger full-stack raffle push with better local infrastructure

---

## 6. Code Quality and Architecture Impact

### 6.1 Heaviest Activity Areas

The most structurally significant files in this period were:
- `frontend/app/page.tsx`
- `frontend/lib/transactions.ts`
- `contracts/freight/src/instructions.rs`
- `contracts/freight/src/validations.rs`
- `tests/src/tests/raffle_e2e.rs`
- `frontend/app/create/_components/CreateCampaignModalContent.tsx`
- `frontend/app/_components/FreightInfoModal.tsx`
- `scripts/devnet.sh`
- `docs/RAFFLE_WINNER_SELECTION.md`

### 6.2 Technical Debt Reduction

**Achievements**:
- Reduced separation risk between frontend raffle preview logic and contract settlement logic by aligning deterministic winner selection behavior
- Consolidated more modal responsibilities into reusable/shared patterns
- Improved persistence of off-chain campaign metadata for comments, activation state, and randomness preimage
- Added local tooling that reduces manual setup cost for contract iteration

---

## 7. Challenges and Solutions

### Challenge 1: Making raffle fairness explainable and enforceable
**Solution**: Combined randomness commitment, reveal verification, deterministic participant ordering, seeded shuffle logic, and e2e tests so the raffle path is both implemented and documented.

### Challenge 2: Handling first-ticket activation without burdening every buyer
**Solution**: Introduced a two-step activation + purchase flow and persisted activation metadata off-chain so later buyers can skip redundant work.

### Challenge 3: Growing feed interactivity without losing UI stability
**Solution**: Iteratively refined header layering, hover states, comment persistence, wallet detail surfaces, and card interaction logic across the main page.

### Challenge 4: Supporting full-stack iteration locally
**Solution**: Added devnet lifecycle tooling and Makefile wrappers so contract deployment and local chain testing are easier to repeat.

---

## 8. Impact Assessment

### Quantitative Impact
- **Commit Volume**: 66 commits since `14a1cd9`
- **Code Change Volume**: 4,197 lines added / 701 removed
- **Files Touched**: 36 files in the inspected range
- **Major Areas Improved**: raffle lifecycle, deterministic settlement, social actions, wallet UX, create-flow persistence, visual polish, local devnet support
- **New Documentation/Test Surfaces**: 1 dedicated raffle fairness doc and 1 major raffle end-to-end lifecycle test

### Qualitative Impact
- **Fairness / Trust**: raffle winner selection is now significantly more concrete, documented, and test-backed
- **User Experience**: ticket purchase, settlement feedback, wallet inspection, comments, and feed interactions feel more product-ready
- **Reliability**: stronger contract validation and transaction-builder parity reduce lifecycle mismatches
- **Developer Experience**: local devnet tooling and richer tests make iteration safer and faster

---

## 9. Next Week's Priorities

### Immediate Focus
1. Finish hardening the raffle settlement path, especially around edge cases and any remaining preimage/reveal recovery flows
2. Continue refining campaign card interactions and settlement-state presentation in the feed
3. Validate social interaction persistence and wallet-driven actions in more real runtime scenarios

### Strategic Initiatives
1. Expand automated coverage for comment persistence, ticket purchase transitions, and settlement edge cases
2. Finalize any remaining authority or lifecycle questions around randomness submission versus create-time configuration
3. Continue separating experimental visual treatments from core campaign and settlement UX

### Documentation
1. Keep raffle fairness and settlement documentation aligned with actual contract behavior
2. Document any finalized operational flows for activation, distribution, and creator-only settlement actions

---

## 10. Lessons Learned

1. **Raffle fairness needs both implementation and narrative clarity**: commit/reveal systems are not enough unless participant ordering and winner derivation are also explicit.
2. **Frontend and contract parity matters most in settlement flows**: previewing winners and validating payouts must follow the same deterministic rules.
3. **A richer campaign feed quickly becomes product infrastructure**: comments, wallet surfaces, timing states, and modal reuse all reinforce trust in the main experience.
4. **Small UI layering bugs have outsized product impact**: pinned headers, z-index fixes, and hover correctness materially affect perceived quality.
5. **Local chain tooling is a multiplier for contract-heavy work**: repeatable devnet workflows make deeper full-stack changes much easier to validate.

---

## 11. Conclusion

This reporting period delivered one of the project’s more important full-stack advances so far: raffles moved much closer to a complete, defensible lifecycle with deterministic settlement, clearer frontend support, contract-side enforcement, written fairness documentation, and end-to-end test coverage.

At the same time, the application itself became noticeably more polished. Campaign cards are more interactive, wallet and modal surfaces are more useful, social behavior is more persistent, and the overall product flow is better equipped to support the deeper on-chain logic now landing underneath it.

---

**Report Prepared By**: Development Team  
**Date**: June 9, 2026  
**Commit Range**: `14a1cd9^..HEAD`