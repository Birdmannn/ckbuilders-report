# FreightOnNervos - Weekly Development Report
**Reporting Period**: Since commit `b5fd03f`  
**Project**: FreightOnNervos - Decentralized Campaign Platform on CKB  
**Branch**: v3

---

## Executive Summary

This reporting period was a foundational build-out phase for FreightOnNervos. The repo moved from a campaign-first prototype into a much more structured application with better testing guidance, refreshed deployment metadata, improved settlement behavior, a full first-pass user profile and leaderboard system, and the first serious mountables/forms implementation path.

The largest product shift was the introduction of persistent user identity as a real application surface. Instead of only exposing wallets and campaigns, the app now has profile APIs, profile routes, editable display names, leaderboard ranking, shared profile UI, and a more capable shell/header model. In parallel, settlement logic and evidence UX became more mature: recipient handling, distribution amount visibility, raffle conclusion state, evidence tables, and settlement-related participant data were all improved.

The other major technical thread was mountables. What began as documentation and scaffolding evolved into typed forms-mountable persistence, create-flow interception around `#mounted`, Google Forms validation, Google account linking infrastructure, mounted-state rendering on campaign cards, and parity fixes between home-page and profile-page create flows.

**Key Metrics**:
- 79 commits in the inspected range since `b5fd03f`
- 59 files changed across frontend, contracts, deployment, tests, docs, and scripts
- 7,240 lines added
- 655 lines removed
- 1 full profile/leaderboard system introduced
- 1 reusable app-shell header introduced
- 1 first real mountables/forms integration path introduced
- 1 Google account linking + forms validation foundation introduced

---

## 1. Testing, Deployment, and Core Settlement Foundations

### 1.1 Testing Guidance and Deployment Metadata Were Formalized

**Problem Identified**: Early-stage iteration on contract and frontend settlement behavior was happening without enough synchronized testing guidance and deployment metadata. That makes debugging difficult because tests, deployed cells, and frontend assumptions can drift apart.

**Solution Implemented**:
- Added a dedicated testing guide for the project
- Refreshed deployment metadata and migration artifacts
- Updated contract/frontend metadata references so deployment information matched runtime assumptions more closely
- Adjusted supporting reclaim/deployment scripts and contract constants

**Primary Files**:
- `TESTING.md`
- `scripts/reclaim-contract-cell.sh`
- `deployment/txs/deploy-fresh-info.json`
- `deployment/migration-fresh/2026-06-23-131416.json`
- `deployment/migration-fresh/2026-07-02-143259.json`
- `contracts/freight/src/main.rs`
- `contracts/freight/src/utils.rs`
- `frontend/lib/contract.ts`

**Impact**:
- Testing and deployment work became easier to reason about
- Contract/frontend alignment improved
- Later settlement and participant changes had a firmer operational base

### 1.2 Settlement, Participant, and Distribution Behavior Was Expanded

**Problem Identified**: Settlement and participant flows needed better supporting infrastructure. The product needed stronger participant APIs, clearer settlement outputs, better evidence rendering, and more explicit distribution behavior.

**Solution Implemented**:
- Added campaign participant API support and later review/verification support around participants
- Improved frontend transaction/build behavior for settlement-related operations
- Added distribution amount visibility alongside recipients
- Added clearer concluded-raffle handling and refined settlement action flows
- Improved share/distribution follow-up behavior and linked settlement outputs more clearly

**Primary Files**:
- `frontend/app/api/campaign-participants/route.ts`
- `frontend/app/api/campaign-participants/[campaignId]/review/route.ts`
- `frontend/app/api/campaign-records/[id]/settle/route.ts`
- `frontend/app/api/campaign-records/[id]/route.ts`
- `frontend/app/api/campaign-records/route.ts`
- `frontend/lib/transactions.ts`
- `frontend/app/_hooks/useCampaignCardState.ts`
- `frontend/app/_hooks/useCampaignFeed.ts`
- `tests/src/tests.rs`

**Impact**:
- Settlement behavior became more explicit and feature-complete
- Recipient and review data could be surfaced more reliably
- The repo advanced from basic payout behavior toward a more inspectable settlement model

### 1.3 Settlement UX Became Easier to Understand

**Problem Identified**: Settlement information was present but not especially legible. Users needed to understand recipients, evidence, payout amounts, and final settlement state more directly.

**Solution Implemented**:
- Reworked settlement information surfaces in the main UI
- Switched evidence presentation toward more structured table-like rendering
- Added clearer UI for payout amounts and distribution outputs
- Improved header/info modal behavior around settlement details
- Refined loading states so settlement and campaign-state transitions felt more deliberate

**Primary Files**:
- `frontend/app/page.tsx`
- `frontend/app/styles/create-modal.css`
- `frontend/app/_components/CampaignCard.tsx`
- `frontend/app/_components/CampaignCardSurface.tsx`
- `frontend/app/_components/CampaignFeedSection.tsx`
- `frontend/app/_components/CampaignFeedHeaderBar.tsx`
- `frontend/app/_components/CampaignList.tsx`
- `frontend/app/styles/campaign.css`

**Impact**:
- Settlement information became much more explainable to users
- Raffle and distribution results became easier to inspect and trust
- Info-modal UX matured into a core product surface instead of a purely auxiliary one

---

## 2. User Identity, Profiles, and Leaderboards

### 2.1 A Real Profile System Was Introduced

**Problem Identified**: The product initially lacked a durable user identity surface. Wallet data existed, but there was no canonical profile destination, no proper profile API, and no coherent user-facing home for reputation-like information.

**Solution Implemented**:
- Added persistent user-profile API support
- Added a dedicated profile data hook
- Introduced the first profile route/page and then refactored it into a reusable shared profile screen
- Added profile-specific styling and loading states
- Threaded profile access into home and shell interactions

**Primary Files**:
- `frontend/app/api/user-profiles/route.ts`
- `frontend/app/_hooks/useUserProfile.ts`
- `frontend/app/profile/page.tsx`
- `frontend/app/_components/ProfileScreen.tsx`
- `frontend/app/styles/profile.css`
- `frontend/app/_components/ThreeDotLoader.tsx`
- `frontend/app/page.tsx`

**Impact**:
- FreightOnNervos gained a real user identity destination
- Profile state moved out of ad hoc wallet-only presentation
- Future reputation, social, and proof-related features gained a clear UI home

### 2.2 The App Shell and Canonical User Routing Became Much Stronger

**Problem Identified**: As profile and wallet functionality expanded, the UI needed a more stable shared shell rather than route-specific duplicated header behavior.

**Solution Implemented**:
- Introduced a reusable `AppShellHeader`
- Reworked wallet actions, disconnect behavior, and route-context actions
- Added a canonical `/user/[handle]` route for profile pages
- Integrated shell-level navigation between home, profile, wallet, and leaderboard states

**Primary Files**:
- `frontend/app/_components/AppShellHeader.tsx`
- `frontend/app/user/[handle]/page.tsx`
- `frontend/app/page.tsx`
- `frontend/app/_components/ProfileScreen.tsx`
- `frontend/app/styles/header.css`
- `frontend/app/styles/wallet.css`

**Impact**:
- The app now behaves more like a cohesive application shell
- Profile navigation is more stable and explicit
- Later feature additions can reuse the same shell model instead of inventing new per-page behavior

### 2.3 Leaderboard, Rank, and Display Identity Were Added

**Problem Identified**: After profile support existed, the app still needed comparative user identity. There was no rank surface, no leaderboard model, and no clean distinction between canonical identity and editable presentation identity.

**Solution Implemented**:
- Added ranking logic through the profile API and hook layer
- Introduced leaderboard rendering and leaderboard modal behavior
- Added current-user visibility and leaderboard actions
- Separated stable handle identity from editable display-name presentation
- Improved profile loading, metadata, and display-name editing behavior

**Primary Files**:
- `frontend/app/api/user-profiles/route.ts`
- `frontend/app/_hooks/useUserProfile.ts`
- `frontend/app/_components/ProfileScreen.tsx`
- `frontend/app/styles/profile.css`
- `frontend/app/user/[handle]/page.tsx`

**Impact**:
- The system now has a first-pass reputation/ranking model
- Canonical user routing is more stable
- Editable presentation identity no longer has to be tightly coupled to route identity

### 2.4 The Profile Experience Was Refined Repeatedly

**Problem Identified**: Once the profile page existed, layout, hero behavior, loading, editing, leaderboard controls, and transitions all needed iterative refinement before the experience felt coherent.

**Solution Implemented**:
- Reworked profile photo placement and handle layout
- Refined hero layout and font sizing
- Added silent handling for non-critical balance-refresh failures
- Added leaderboard actions and supporting UI tweaks
- Added profile-hero transition behavior and profile textbox focus fixes

**Primary Files**:
- `frontend/app/_components/ProfileScreen.tsx`
- `frontend/app/styles/profile.css`
- `frontend/app/_hooks/useWalletInfo.ts`

**Impact**:
- The profile UI became substantially more polished and resilient
- Loading and editing flows became easier to understand
- The profile page moved closer to product-quality behavior rather than experimental UI

---

## 3. Wallet, Header, Loading, and Shared UX Refinement

### 3.1 Wallet and Header Interactions Were Reworked Into a More Useful Shell

**Problem Identified**: As create modals, info modals, wallet state, profile state, and leaderboard state accumulated, header and wallet behavior became a systems problem rather than a simple component problem.

**Solution Implemented**:
- Added a more reusable wallet/header shell
- Introduced a minimal disconnect modal direction
- Improved route-sensitive wallet/home/profile actions
- Refined header info modal behavior and visual layout
- Improved chain, balance, and compact wallet presentation

**Primary Files**:
- `frontend/app/_components/AppShellHeader.tsx`
- `frontend/app/_components/FreightInfoModal.tsx`
- `frontend/app/_hooks/useWalletInfo.ts`
- `frontend/app/styles/header.css`
- `frontend/app/styles/wallet.css`
- `frontend/app/page.tsx`
- `frontend/app/campaign/[campaignId]/page.tsx`

**Impact**:
- Shared shell behavior became more consistent
- Wallet interactions became less intrusive and more deliberate
- The product gained a clearer navigation and overlay model

### 3.2 Loading and Feed/Header States Were Cleaned Up

**Problem Identified**: Loading behavior across campaign feed, profile, settlement, and wallet refresh states was uneven and often visually noisy.

**Solution Implemented**:
- Re-envisioned campaign loading states and finalized supporting loading styling
- Added a reusable loader primitive
- Reduced unnecessary visible failures from background wallet refresh
- Continued refining feed/header bars and related page-level status handling

**Primary Files**:
- `frontend/app/_components/CampaignFeedHeaderBar.tsx`
- `frontend/app/_components/ThreeDotLoader.tsx`
- `frontend/app/_hooks/useWalletInfo.ts`
- `frontend/app/_components/ProfileScreen.tsx`
- `frontend/app/styles/campaign.css`

**Impact**:
- Loading behavior became more intentional and less distracting
- Recoverable failures became quieter
- Shared UI state feels less fragile during async transitions

---

## 4. Mountables, Forms, and Google Integration

### 4.1 The Mountables Direction Was Documented and Typed

**Problem Identified**: External completion surfaces needed a real data model before UX could be meaningful. The project lacked durable configuration types, campaign-record threading, and written implementation direction.

**Solution Implemented**:
- Added two major implementation/design documents for the v4/mountables direction
- Added shared mountable and campaign-record types
- Added forms-mountable helper logic
- Began threading forms configuration through draft and campaign-record persistence

**Primary Files**:
- `V4_IMPLEMENTATION.md`
- `V4_IMPLEMENATATION2.md`
- `frontend/app/_lib/formsMountable.ts`
- `frontend/app/_types/formsMountable.ts`
- `frontend/app/_types/campaignRecords.ts`
- `frontend/app/api/campaign-records/route.ts`
- `frontend/app/api/campaign-records/[id]/route.ts`
- `frontend/app/api/campaign-records/drafts/route.ts`
- `frontend/app/_hooks/useCampaignFeed.ts`

**Impact**:
- Mountables moved from concept to typed implementation groundwork
- Draft and record persistence now understand forms configuration
- Future provider-specific work has a much stronger base

### 4.2 The Create Flow Learned to Branch on `#mounted`

**Problem Identified**: Even with typed scaffolding, the create experience still needed a real UX path for mountables. Users needed a way to branch into mountable selection before continuing campaign creation.

**Solution Implemented**:
- Added the first mountables chooser UX
- Added draft-aware interception of the create flow when `#mounted` is detected
- Added the forms mountable subflow and staged modal transitions
- Refined related mountables styling, hover behavior, drafts behavior, sticky modal behavior, and multi-stage transitions
- Finalized mountables icon behavior and polished the first-phase UI

**Primary Files**:
- `frontend/app/create/_components/CreateCampaignModalContent.tsx`
- `frontend/app/page.tsx`
- `frontend/app/styles/create-modal.css`
- `frontend/app/styles/campaign.css`
- `frontend/app/_components/CampaignCardSurface.tsx`

**Impact**:
- Mountables gained a real product entry point in the create flow
- The app can now react to `#mounted` instead of merely documenting it
- Forms mountables became visible in both create UX and campaign surfaces

### 4.3 Google Forms Validation and Google Account Linking Were Added

**Problem Identified**: A forms mountable is only useful if the system can validate forms inputs and associate relevant external identity/account state with users.

**Solution Implemented**:
- Added Google Forms parsing/validation support
- Added Google auth/linking infrastructure and API routes
- Added user-profile support for linked Google account data
- Added review-route support and participant-side forms-related behavior
- Refined marker parsing/finalization in forms processing

**Primary Files**:
- `frontend/app/api/google-forms/validate/route.ts`
- `frontend/app/_hooks/useGoogleLink.ts`
- `frontend/app/api/google/link/start/route.ts`
- `frontend/app/api/google/link/route.ts`
- `frontend/app/api/google/link/nonce/route.ts`
- `frontend/app/api/google/link/callback/route.ts`
- `frontend/app/api/google/link/complete/route.ts`
- `frontend/lib/googleAuth.ts`
- `frontend/lib/googleForms.ts`
- `frontend/app/api/user-profiles/route.ts`
- `frontend/app/api/campaign-participants/[campaignId]/review/route.ts`
- `frontend/app/api/campaign-participants/route.ts`

**Impact**:
- The first mountable now has real provider-specific backend infrastructure
- User identity and external forms identity started converging into one model
- The repo is much closer to a usable forms-based mounted workflow

### 4.4 Mounted-State Surfaces Were Extended Across the App

**Problem Identified**: Even after the home-page create flow learned mountables, the mounted state still needed better visibility and parity in other surfaces.

**Solution Implemented**:
- Added mounted-state indicators on campaign cards
- Improved mounted icon sizing and spacing
- Added profile-page parity so `#mounted` now opens the mountables modal during creation from the user profile page as well

**Primary Files**:
- `frontend/app/_components/CampaignCardSurface.tsx`
- `frontend/app/styles/campaign.css`
- `frontend/app/_components/ProfileScreen.tsx`

**Impact**:
- Mounted campaigns are now easier to identify visually
- The create experience is more consistent between home and profile flows
- Mountables are becoming a platform concept instead of a one-page experiment

---

## 5. Challenges and Solutions

### Challenge 1: Testing and deployment context were too easy to desynchronize
**Solution**: Added explicit testing guidance and refreshed deployment metadata so development, testing, and runtime assumptions were better aligned.

### Challenge 2: Settlement correctness needed both backend and frontend attention
**Solution**: Improved participant APIs, settlement record flows, frontend transaction handling, and payout/evidence UI together instead of treating them as isolated problems.

### Challenge 3: User identity needed a stable route model and an editable presentation model
**Solution**: Built profile APIs/routes/hooks and separated handle identity from display-name editing.

### Challenge 4: Shared shell complexity grew quickly as profile, wallet, and modal states multiplied
**Solution**: Introduced a reusable shell/header and iteratively tightened overlay, loading, and action behavior.

### Challenge 5: Mountables required typed persistence before polished UX could exist
**Solution**: Documented the architecture first, added shared types and helpers, then layered create-flow interception and forms UI on top.

### Challenge 6: A forms mountable needs provider-specific trust and validation, not generic UI only
**Solution**: Added Google Forms validation and Google account linking infrastructure early, so the first mountable has a real backend path.

### Challenge 7: Feature parity drifted between home-page and profile-page create flows
**Solution**: Extended the same `#mounted` interception/mountables modal behavior into the profile-page creation flow.

---

## 6. What Was Learned

1. **Settlement UX has to be explainable, not just technically correct**; recipients, amounts, and evidence all need structured presentation.
2. **User identity becomes a cross-cutting concern quickly** once profiles, rankings, wallet state, and external providers all meet in the same app.
3. **A reusable shell/header matters early** because modal and route complexity compounds fast.
4. **Mountables need typed persistence and provider-specific validation before they can feel real**.
5. **Google-backed forms workflows introduce both product value and identity complexity**, so account-linking and validation must be designed together.
6. **Feature parity across entry points matters**; if a create flow exists on both home and profile pages, mountables behavior has to match.
7. **Small UX fixes accumulate into major product quality gains** when they target loading, focus, spacing, transitions, and modal continuity.

---

## 7. Impact Assessment

### Quantitative Impact
- **Commit Volume**: 79 commits since `b5fd03f`
- **Code Change Volume**: 7,240 insertions / 655 deletions in the inspected range
- **Files Touched**: 59 files across frontend, contracts, deployment, tests, scripts, and docs
- **Top-Level Focus**: 49 frontend files, 3 deployment files, 2 contract files, plus tests/docs/script work
- **New Major Docs**: 3 (`TESTING.md`, `V4_IMPLEMENTATION.md`, `V4_IMPLEMENATATION2.md`)
- **New Profile Route**: 1 canonical user route (`/user/[handle]`)
- **New Shared Shell Primitive**: 1 (`AppShellHeader`)
- **New External-Provider Foundation**: Google account linking + forms validation stack introduced

### Qualitative Impact
- **Correctness**: settlement and participant flows became stronger and more inspectable
- **Product Identity**: FreightOnNervos now has a real user-profile and leaderboard model
- **UX Coherence**: wallet, header, loading, profile, and create flows are more integrated
- **Platform Readiness**: mountables/forms moved from exploratory idea into real typed and validated scaffolding
- **Extensibility**: the repo now has a plausible foundation for completion-based external workflows

---

## 8. Next Week's Priorities

### Immediate Focus
1. Complete forms mountables end-to-end from creation through proof/review/eligibility
2. Continue tightening settlement and participant review behavior around mounted completions
3. Keep home and profile creation flows behaviorally identical for mountables and drafts

### Strategic Initiatives
1. Turn the Google-linked forms flow into a fuller verification path for participants
2. Surface mountable summaries and mounted-state context in more campaign views
3. Continue refining the user identity layer so profile, leaderboard, and external identity work together cleanly

### Documentation
1. Keep the v4/mountables implementation docs synchronized with actual shipped behavior
2. Expand testing guidance as forms and participant review paths mature
3. Keep deployment metadata and contract/frontend assumptions aligned as settlement behavior evolves

---

## 9. Conclusion

This period was one of the most important structural expansions of FreightOnNervos so far. The app now has better testing and deployment guidance, stronger settlement and participant handling, a real profile-and-leaderboard experience, a reusable shell/header model, and meaningful mountables/forms infrastructure with Google-specific validation and linking support.

The main theme across the work was moving from isolated features to connected systems. Settlement became more inspectable, identity became more durable, the shell became more reusable, and mountables stopped being just a design idea and started becoming an implementable workflow. That shift gives the project a much stronger base for future proof-driven participation, mounted external tasks, and broader user-facing platform behavior.

---

**Report Prepared By**: Development Team  
**Date**: July 11, 2026  
**Commit Range**: `b5fd03f..8f164ee(HEAD)`
