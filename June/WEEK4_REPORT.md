# FreightOnNervos - Weekly Development Report
**Reporting Period**: Since commit `b4d88f3`  
**Project**: FreightOnNervos - Decentralized Campaign Platform on CKB  
**Branch**: v3

---

## Executive Summary

This reporting period pushed FreightOnNervos forward on four connected fronts: hardening raffle participant identity and settlement correctness, making settlement evidence and payout surfaces much clearer to users, introducing a real profile-and-leaderboard system, and laying the first meaningful groundwork for mountables with an initial Google-forms-oriented direction.

The most important technical thread was continued work on participant identity. The repo moved from vague or partially implicit handling toward a much more explicit model spanning contract, frontend, deployment metadata, tests, and documentation. That work matters not only for raffle settlement correctness, but also for future mountables, where reward eligibility depends on proving that a participant completed something external to the chain.

At the same time, the frontend gained a far more mature user identity layer. Instead of only exposing wallet and campaign-level identity, the app now has canonical profile routes, editable display names, rank and leaderboard visibility, and stronger shared-shell behavior. In parallel, settlement info surfaces were upgraded so users can see recipient evidence, payout amounts, distribution transaction links, and their own highlighted position more clearly. Finally, the first mountables/forms scaffolding was introduced across shared types, record APIs, draft persistence, and create-flow UX interception.

**Key Metrics**:
- 59 commits in the inspected range since `b4d88f3`
- 50 files changed across contract, frontend, docs, tests, API routes, and deployment metadata
- 5,346 lines added
- 641 lines removed
- 1 major shared profile screen introduced
- 1 canonical `/user/[handle]` route introduced
- 1 first-pass leaderboard system introduced
- 4 mountables/forms design and documentation artifacts advanced or added

---

## 1. Participant Identity and Settlement Reliability

### 1.1 Participant Identity Investigation Continued Across Contract and Frontend

**Problem Identified**: Raffle settlement correctness depends on participant identity being stable and consistently interpreted across the contract, frontend transaction builders, tests, and deployment assumptions. This remained a foundational risk area because participant eligibility is the basis for winner selection, settlement evidence, and future completion-gated features.

**Solution Implemented**:
- Continued documenting and formalizing the participant identity bug and its consequences
- Updated contract/runtime behavior around participant handling and campaign identity tracking
- Refreshed supporting frontend and test-layer assumptions to match the clarified model
- Updated deployment metadata and testing guidance so debugging and validation were working from the same mental model

**Primary Files**:
- `docs/RAFFLE_PARTICIPANT_IDENTITY_BUG.md`
- `contracts/freight/src/main.rs`
- `contracts/freight/src/utils.rs`
- `frontend/lib/contract.ts`
- `frontend/lib/transactions.ts`
- `tests/src/tests.rs`
- `deployment/txs/deploy-fresh-info.json`
- `TESTING.md`

**Impact**:
- Participant identity handling became more explicit and stable
- Settlement logic now rests on a clearer foundation for deterministic behavior
- The project is better positioned for future mountables that depend on off-chain completion proof and on-chain payout eligibility

### 1.2 Settlement Information and Distribution UX Became More Explainable

**Problem Identified**: Even when settlement logic works, users still need to understand who received rewards, why they received them, and where the actual distribution transaction went. Earlier payout surfaces were too ambiguous and too lightly structured for that.

**Solution Implemented**:
- Upgraded settlement info modal behavior and evidence presentation
- Structured recipient rendering more clearly instead of presenting settlement details as loose text
- Added connected-user highlighting so users can immediately identify themselves in winner/recipient output
- Added distribution amount display and distribution transaction linking
- Continued refining surrounding settlement action UX in feed and card surfaces

**Primary Files**:
- `frontend/app/page.tsx`
- `frontend/app/styles/create-modal.css`
- `frontend/app/_hooks/useCampaignCardState.ts`
- `frontend/app/_components/CampaignCard.tsx`
- `frontend/app/_components/CampaignCardSurface.tsx`
- `frontend/app/_components/CampaignFeedSection.tsx`
- `frontend/app/_components/CampaignList.tsx`

**Impact**:
- Settlement flows now provide much clearer evidence and payout context
- Users have a better understanding of how winners and recipients were derived
- The payout experience feels less opaque and more trustworthy

---

## 2. User Profile, Identity, and Leaderboard System

### 2.1 A Real Profile System Was Introduced

**Problem Identified**: The app previously exposed wallet-linked identity only indirectly through campaigns and scattered header state. There was no durable profile destination, no stable user route, and no coherent place to show user-specific status.

**Solution Implemented**:
- Introduced a reusable shared profile screen
- Added a canonical `/user/[handle]` route
- Added a reusable loading primitive for profile-bound async states
- Built the supporting API and hook layer for loading user profile data
- Added dedicated profile styling and integrated profile navigation into the shell/header behavior

**Primary Files**:
- `frontend/app/_components/ProfileScreen.tsx`
- `frontend/app/user/[handle]/page.tsx`
- `frontend/app/_components/ThreeDotLoader.tsx`
- `frontend/app/_hooks/useUserProfile.ts`
- `frontend/app/api/user-profiles/route.ts`
- `frontend/app/styles/profile.css`
- `frontend/app/globals.css`
- `frontend/app/_components/AppShellHeader.tsx`
- `frontend/app/styles/wallet.css`
- `frontend/app/styles/header.css`
- `frontend/app/page.tsx`

**Impact**:
- FreightOnNervos now has a real user identity surface, not just campaign-centric identity exposure
- The app shell can navigate into user-specific context more naturally
- Future reputation, social, and proof-related features now have a more appropriate home

### 2.2 First-Pass Leaderboard and Rank Logic Was Added

**Problem Identified**: Once user profiles existed, the project still lacked any comparative ranking surface for reputation-like identity. There was no first-class way to show relative standing or navigate across users in a leaderboard context.

**Solution Implemented**:
- Added leaderboard and rank typing through the profile data layer
- Implemented ranking logic based first on FBARS, with alphabetical-handle tie-breaking
- Added leaderboard modal behavior to the profile experience
- Added current-user highlighting and centered scroll-to-user behavior
- Added rank display in the profile hero surface

**Primary Files**:
- `frontend/app/_hooks/useUserProfile.ts`
- `frontend/app/api/user-profiles/route.ts`
- `frontend/app/_components/ProfileScreen.tsx`

**Impact**:
- Introduced the first comparative identity surface for the product
- Created a visible reputation/ranking model that can evolve later
- Improved profile-to-profile navigation and self-location within leaderboard output

### 2.3 Handle and Display Name Were Separated

**Problem Identified**: Route identity and user-facing display identity were too tightly coupled. If the same field is used for both canonical routing and editable presentation, profile editing can destabilize URLs and linked references.

**Solution Implemented**:
- Split **handle** and **display name** into different conceptual roles
- Kept `/user/[handle]` as the stable route identity
- Moved profile editing toward display-name changes instead of route-key changes
- Adjusted supporting display-name generation and edit behavior accordingly

**Primary Files**:
- `frontend/app/_components/ProfileScreen.tsx`
- `frontend/app/_hooks/useUserProfile.ts`
- `frontend/app/api/user-profiles/route.ts`
- `frontend/app/user/[handle]/page.tsx`

**Impact**:
- Improved long-term identity modeling for the app
- Protected canonical profile URLs from casual presentation edits
- Created a more flexible foundation for future social/profile features

---

## 3. Header, Wallet, and Modal Shell Refinement

### 3.1 Wallet and Header Actions Were Reworked Into a Better App Shell

**Problem Identified**: As more modal states accumulated — wallet modal, info modal, create modal, leaderboard modal, and settlement overlays — the shell behavior became increasingly sensitive to z-index ordering, close semantics, and context-dependent actions.

**Solution Implemented**:
- Refined disconnect modal layout and wallet-modal compactness
- Improved balance formatting into a more compact single-line expression
- Updated chain display formatting
- Refined introspect/home actions depending on current page context
- Adjusted z-index and overlay layering between wallet, leaderboard, and info modal states
- Added temporary replacement actions during leaderboard state
- Reduced blur collisions and improved overlay interaction around the fixed header

**Primary Files**:
- `frontend/app/_components/AppShellHeader.tsx`
- `frontend/app/styles/wallet.css`
- `frontend/app/styles/header.css`
- `frontend/app/page.tsx`
- `frontend/app/_components/ProfileScreen.tsx`

**Impact**:
- The header now behaves more like a real application shell
- Modal interactions are less collision-prone and easier to reason about
- User context switches between home, wallet, profile, and leaderboard are smoother

### 3.2 Loading and Secondary Failure States Were Cleaned Up

**Problem Identified**: Recoverable refresh failures, especially wallet-balance refreshes, were leaking into primary page state and creating unnecessary user-facing noise.

**Solution Implemented**:
- Centralized profile loading behind a single loader gate
- Stopped treating balance-refresh failures as primary visible errors
- Restricted major visible failure surfaces to real profile-load failures instead of secondary telemetry churn

**Primary Files**:
- `frontend/app/_components/ProfileScreen.tsx`
- `frontend/app/_components/ThreeDotLoader.tsx`
- `frontend/app/_hooks/useWalletInfo.ts`
- `frontend/app/_hooks/useUserProfile.ts`

**Impact**:
- Cleaner loading experience
- Less noisy profile UI during transient wallet refresh issues
- Better separation between critical failures and recoverable background failures

---

## 4. Mountables and Forms Groundwork

### 4.1 The First Mountable Direction Was Defined

**Problem Identified**: The product needs a way to attach external completion surfaces to campaigns, but there was not yet a structured provider model, stored configuration shape, or create-flow interception mechanism for mountables.

**Solution Implemented**:
- Produced implementation design docs for the mountables/forms direction
- Added shared forms-mountable configuration types and helpers
- Added shared campaign-record typing needed for persistence
- Started threading mountable configuration through draft and campaign-record APIs
- Began wiring create-modal interception around `#mounted` so the flow can branch into mountable selection before continuing

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
- `frontend/app/create/_components/CreateCampaignModalContent.tsx`
- `frontend/app/page.tsx`

**Impact**:
- Mountables moved from idea-stage into actual typed scaffolding
- Draft persistence and record shapes now have a path for forms configuration
- The create flow now has the beginnings of a real provider-selection/validation UX

### 4.2 Current Mountables Status

**Solution Implemented / In Progress**:
- UI and draft persistence groundwork exists
- Mountable chooser interaction has been introduced and iterated
- The end-to-end creation/review/proof/verification lifecycle remains incomplete

**Impact**:
- The project now has real infrastructure for a first mountable
- But forms are still clearly an in-progress subsystem rather than a finalized user feature

---

## 5. Challenges and Solutions

### Challenge 1: Participant identity affected more than one layer
**Solution**: Treated participant identity as a cross-layer system concern, not just a contract bug or frontend bug, and updated docs, code, tests, and deployment guidance together.

### Challenge 2: Editable user identity risked breaking canonical routes
**Solution**: Split stable route handle from editable display name, preserving canonical profile paths while allowing user-facing customization.

### Challenge 3: Modal layering kept regressing as the shell gained more features
**Solution**: Reworked header and modal interactions incrementally, making backdrop behavior, replacement actions, and z-index ordering more explicit.

### Challenge 4: Settlement proof was technically present but not readable enough
**Solution**: Rebuilt recipient and evidence surfaces so payout information is more structured, linked, and user-comprehensible.

### Challenge 5: Mountables needed real data scaffolding before polished UX
**Solution**: Introduced shared types, helper modules, and API threading first, then began iterating on the chooser and create-flow interception on top of that base.

### Challenge 6: Background refresh errors were polluting primary UX states
**Solution**: Reduced noisy error propagation and treated wallet-balance refresh as recoverable telemetry rather than a fatal user-facing state.

---

## 6. What Was Learned

1. **Participant identity must be modeled explicitly across contract, frontend, and verification flows**; fuzzy assumptions become settlement bugs later.
2. **User-facing naming and route identity should not be the same thing** when profiles are meant to be editable and link-stable.
3. **Modal layering becomes a product-level systems problem quickly** once a shell has wallet, info, create, leaderboard, and settlement overlays at once.
4. **Readable evidence matters almost as much as correct evidence** in payout flows; users need to understand the result, not just receive it.
5. **Mountables need provider-specific trust models**; typed scaffolding and validation rules are a prerequisite for any serious forms integration.
6. **Secondary telemetry failures should usually degrade quietly** instead of hijacking primary route state.

---

## 7. Impact Assessment

### Quantitative Impact
- **Commit Volume**: 59 commits since `b4d88f3`
- **Code Change Volume**: 5,346 insertions / 641 deletions in the inspected range
- **Files Touched**: 50 files across contract, frontend, docs, APIs, tests, and deployment metadata
- **New Canonical Profile Route**: 1 (`/user/[handle]`)
- **New Shared Profile Screen**: 1
- **New Leaderboard System**: 1 first-pass implementation
- **Forms/Mountables Design + Scaffolding Artifacts**: 4 major files/docs advanced or introduced

### Qualitative Impact
- **Correctness**: participant identity and settlement behavior are on a much clearer footing
- **User Experience**: settlement evidence, wallet/header interactions, and profile flows are substantially stronger
- **Identity Model**: the system now distinguishes stable route identity from editable display identity
- **Platform Readiness**: forms mountables are no longer purely conceptual; the repo now contains real persistence and UX scaffolding for them
- **Maintainability**: shared shell and profile abstractions reduce drift across home and profile experiences

---

## 8. Next Week's Priorities

### Immediate Focus
1. Complete the `#mounted` → mountable chooser flow end-to-end
2. Finalize forms mountable draft/create/review persistence behavior
3. Continue tightening participant identity and verification assumptions for completion-based eligibility

### Strategic Initiatives
1. Add provider-specific validation and auth rules for the first forms mountable
2. Surface mountable summaries on campaign cards, detail pages, and info surfaces
3. Connect mountable completion proof into participant verification and payout eligibility rules

### Documentation
1. Keep the participant-identity debugging note synchronized with the final implemented model
2. Keep forms/mountables design docs aligned with whatever provider-specific v1 path is chosen
3. Refresh testing guidance as mountables move from scaffolding into real flows

---

## 9. Conclusion

This period moved FreightOnNervos from a campaign-only application toward a more complete platform model. The repo now has a clearer participant identity foundation for settlement, better user-facing payout evidence, a real profile-and-leaderboard system, and the first serious scaffolding for mountables.

The most important theme was explicitness: explicit participant identity, explicit profile identity, explicit shell/modal behavior, and explicit data shapes for forms-based mountables. That shift matters because future features — especially proof-based eligibility and mounted external workflows — depend on stable identity and understandable user flows, not just on isolated UI additions.

---

**Report Prepared By**: Development Team  
**Date**: July 1, 2026  
**Commit Range**: `b4d88f3..b5fd03f(HEAD)`