# FreightOnNervos - Weekly Development Report
**Reporting Period**: Since commit `77ba89c`  
**Project**: FreightOnNervos - Decentralized Campaign Platform on CKB  
**Branch**: v3

---

## Executive Summary

This reporting period focused on transforming the campaign creation flow into a more durable drafting experience, improving campaign feed usability, refining raffle-specific behavior, and stabilizing both backend reliability and frontend development workflow. The work delivered substantial UX gains across modal flows, campaign cards, countdown handling, and visual consistency, while also addressing technical issues in MongoDB connectivity and local development caching.

**Key Metrics**:
- 25 commits merged since `77ba89c`
- 15 primary files changed across frontend and backend support layers
- 2,255 lines added
- 539 lines removed
- 2 new component/preview files introduced during the period
- major improvements across drafts, campaign cards, countdowns, and developer workflow

---

## 1. Drafting and Campaign Creation Flow Improvements

### 1.1 Multi-Draft Authoring Workflow

**Problem Identified**: The campaign creation experience was too fragile. Users could lose in-progress work easily, there was no mature draft lifecycle, and draft loading/replacement behavior was not robust enough for repeated real-world use.

**Solution Implemented**: Built out a much more complete draft workflow inside the create campaign modal, including:

- Saving create-flow state as drafts
- Listing and loading saved drafts
- Deleting obsolete drafts
- Persisting more off-chain authoring state across the flow
- Integrating confirmation logic before destructive draft replacement or modal close

**Primary Files**:
- `frontend/app/create/_components/CreateCampaignModalContent.tsx`
- `frontend/app/page.tsx`
- `frontend/app/api/campaign-records/drafts/route.ts`
- `frontend/app/api/campaign-records/[id]/route.ts`

**Impact**:
- Reduced risk of losing work during campaign creation
- Made the composer behave more like a durable editing surface
- Improved author confidence in longer composition sessions
- Established the foundation for more advanced draft management later

### 1.2 Draft Retrieval UX Redesign

**Problem Identified**: The original draft retrieval behavior felt delayed and awkward. The drawer-style interaction waited too long on network activity and made loading drafts feel secondary rather than integrated.

**Solution Implemented**:
- Reworked draft retrieval into a fuller compose-surface state
- Added immediate loading feedback
- Added skeleton draft placeholders
- Improved helper/status copy while retrieving drafts
- Reworked the top-right arrow behavior for showing/hiding drafts
- Preserved animation for applying draft content back into compose state

**Primary Files**:
- `frontend/app/create/_components/CreateCampaignModalContent.tsx`
- `frontend/app/styles/create-modal.css`
- `frontend/app/styles/create-review.css`
- `frontend/app/page.tsx`

**Impact**:
- Better perceived responsiveness
- Clearer system feedback during retrieval
- More modal-native draft browsing experience
- Less reliance on awkward delayed drawer motion

### 1.3 Save-Draft Modal Refactor

**Problem Identified**: The information modal for Freight/help/create constraints and save-draft confirmation had too much inline responsibility in `page.tsx`, making it difficult to evolve.

**Solution Implemented**:
- Extracted a reusable `FreightInfoModal` shell
- Kept content composition page-owned so behavior remained flexible
- Reused the same shell for create constraints, help copy, and save-draft confirmation flows

**Primary Files**:
- `frontend/app/_components/FreightInfoModal.tsx`
- `frontend/app/page.tsx`
- `frontend/app/styles/header.css`

**Impact**:
- Cleaner separation between modal chrome and content
- Lower complexity in the page-level render tree
- Easier future iteration on modal-specific UX

---

## 2. Campaign Card and Feed Experience Improvements

### 2.1 Campaign Card Simplification and Reorganization

**Problem Identified**: Campaign cards contained too much repeated or low-value information, making them dense and harder to scan.

**Solution Implemented**:
- Removed several redundant metadata blocks
- Reduced repeated campaign type display
- Simplified how summary/deposit-related information appeared on cards
- Reorganized footer and countdown placement

**Primary Files**:
- `frontend/app/page.tsx`
- `frontend/app/styles/campaign.css`

**Impact**:
- Cleaner feed presentation
- Easier scanning of campaign cards
- Better visual hierarchy for actual decision-relevant data

### 2.2 Countdown and Timing UX Refinement

**Problem Identified**: The timing display was too verbose and displayed unnecessary zero-value units, making it harder to read quickly.

**Solution Implemented**:
- Reworked countdown formatting so leading zero-value units are omitted as time shrinks
- Added `--` for ended countdown states
- Preserved severity/tone coloring (`good`, `warn`, `danger`, `ended`)
- Improved the readability of active timing states across card sizes

**Impact**:
- Faster recognition of campaign timing
- Less visual noise in active countdowns
- More natural transition from long duration to short duration display

### 2.3 Expandable Card Content and Height Bounding

**Problem Identified**: Very long campaign descriptions caused card-height instability, while fully truncating them removed too much useful context.

**Solution Implemented**:
- Added max-height behavior to cards
- Added `Read more...` / `Show less` behavior for longer descriptions
- Preserved quote formatting for lines beginning with `>`
- Kept footer and actions visible even when descriptions expand

**Impact**:
- More stable feed layout
- Better balance between brevity and full readability
- More predictable card height behavior in mixed-content feeds

### 2.4 Background Refresh Behavior

**Problem Identified**: Refreshing campaigns cleared already-visible cards and replaced them with `Loading campaigns…`, causing unnecessary flicker and poor UX.

**Solution Implemented**:
- Separated initial loading from background refresh behavior
- Preserved visible campaigns while refreshing in the background
- Added support for indicating unseen campaigns near the `Freights` header
- Added a path for scrolling to newly applied top-of-feed content

**Impact**:
- More stable feed during refresh
- Better perceived performance
- Reduced disruption for users already reviewing visible campaigns

---

## 3. Raffle-Specific Experience Enhancements

### 3.1 Raffle Display Logic on Campaign Cards

**Problem Identified**: Raffle campaigns were being presented too similarly to non-raffle campaigns, despite having a fundamentally different participation model.

**Solution Implemented**:
- Added raffle-specific card behavior
- Displayed remaining tickets instead of deposit progress where appropriate
- Added ticket price exposure near the campaign type row
- Improved raffle-specific card semantics across the feed

**Impact**:
- Better alignment between UI and raffle mechanics
- Clearer user understanding of how raffle participation works
- Reduced ambiguity compared to deposit-style presentation

### 3.2 Raffle Participation Constraints

**Problem Identified**: Users should not be able to buy tickets under invalid campaign conditions.

**Solution Implemented**:
- Disabled raffle purchase before the raffle starts
- Disabled purchase when tickets are exhausted
- Disabled purchase when campaign capacity has been reached
- Disabled purchase when the campaign is completed or cancelled

**Impact**:
- Reduced invalid interaction attempts
- Clearer participation boundaries
- Safer frontend behavior aligned with campaign state

### 3.3 Raffle Preview/Review Semantics in Create Flow

**Problem Identified**: The create/review modal was using generic `max deposit` terminology even for raffles, which is less meaningful than ticket-count semantics.

**Solution Implemented**:
- Changed raffle preview/review inputs to use `Number of tickets`
- Derived raffle max amount from ticket count × ticket price where needed
- Updated validation and preview language accordingly

**Impact**:
- More natural raffle authoring workflow
- Better mental mapping between author intent and platform semantics
- Less confusing campaign setup for raffle creators

---

## 4. Visual Styling and Theme Improvements

### 4.1 Dark Mode Corrections

**Problem Identified**: Several UI regions, especially campaign cards, still carried light-mode assumptions into dark mode, resulting in white surfaces or washed-out content inside otherwise dark screens.

**Solution Implemented**:
- Updated card surfaces to use theme-aware foreground/background behavior
- Removed the white interior effect in dark mode
- Rebalanced contrast after some intermediate changes made card details too gray
- Refined how metadata and card-detail emphasis behaves across themes

**Primary Files**:
- `frontend/app/page.tsx`
- `frontend/app/styles/campaign.css`

**Impact**:
- Better dark-mode consistency
- Fewer visual regressions between themes
- Stronger readability without reintroducing harsh contrast

### 4.2 Floating Create Button Refinement

**Problem Identified**: In dark mode, the floating create button did not appear visually lifted because the shadow treatment was too black or too bright when revised.

**Solution Implemented**:
- Tuned dark-mode shadowing for the floating create button
- Iterated on softness/brightness until the button felt elevated without glowing too aggressively

**Impact**:
- Better perceived depth in dark mode
- More consistent floating-action-button behavior across themes

### 4.3 Experimental Rainbow Border and Pixel/Grid Styling

**Problem Identified**: Certain highlight treatments and retro visual ideas needed a safe place to be explored without destabilizing core product UI.

**Solution Implemented**:
- Added an isolated rainbow border preview route
- Created reusable highlight-border logic for experimentation on selected cards
- Iterated on marquee/mountables background styling with dot, grid, and pixel-aligned experiments
- Tuned retro marquee sizing to align more closely with the background pattern

**Files Added / Modified**:
- `frontend/app/rainbow-button-preview/page.tsx`
- `frontend/app/styles/rainbow-preview.css`
- `frontend/app/styles/campaign.css`
- `frontend/app/styles/marquee.css`

**Impact**:
- Safer experimentation with decorative UI
- Better separation between prototype styling and production styling
- Stronger retro identity direction for the mountables area

---

## 5. Backend Reliability and Developer Workflow

### 5.1 MongoDB Connection Reliability

**Problem Identified**: A failed first MongoDB connection attempt could poison the cached global client promise and cause subsequent fetches to keep failing.

**Solution Implemented**:
- Updated `frontend/lib/mongodb.ts` so failed initial connections clear the cached promise and allow later retries

**Impact**:
- Better resilience against transient Mongo connectivity failures
- More recoverable behavior during API access
- Reduced chance of repeated failures after one bad connection attempt

### 5.2 Development Workflow Stabilization

**Problem Identified**: Repeated stale CSS/hydration/dev-cache issues were slowing local frontend iteration.

**Solution Implemented**:
- Switched the frontend dev script to webpack mode
- Repeatedly validated local build health while reducing Turbopack-related sticky state problems

**Changed File**:
- `frontend/package.json`

**Impact**:
- More predictable local iteration
- Fewer stale CSS/HMR artifacts
- Reduced time spent debugging dev-only state issues

---

## 6. Code Quality and Architecture Impact

### 6.1 Heaviest Activity Areas

The most structurally significant files in this period were:
- `frontend/app/create/_components/CreateCampaignModalContent.tsx`
- `frontend/app/page.tsx`
- `frontend/app/styles/create-modal.css`
- `frontend/app/styles/campaign.css`
- `frontend/app/styles/marquee.css`
- `frontend/app/_components/FreightInfoModal.tsx`
- `frontend/app/api/campaign-records/drafts/route.ts`
- `frontend/app/api/campaign-records/[id]/route.ts`
- `frontend/lib/mongodb.ts`
- `frontend/package.json`

### 6.2 Technical Debt Reduction

**Achievements**:
- Reduced duplicated modal-shell responsibility by extracting `FreightInfoModal`
- Improved state ownership between page-level orchestration and create-modal internals
- Introduced clearer raffle-specific logic rather than overloading generic deposit semantics
- Improved resilience of backend connection handling and frontend dev workflow

---

## 7. Challenges and Solutions

### Challenge 1: Durable Draft UX Inside a Rich Modal Editor
**Solution**: Separated draft persistence, retrieval, replacement confirmation, and editor hydration concerns so the create flow could behave more like a real editor.

### Challenge 2: Raffle Semantics Diverging from Generic Campaign Semantics
**Solution**: Introduced explicit raffle-specific preview, card, and participation behavior rather than forcing raffles through generic deposit-oriented labels.

### Challenge 3: Visual Iteration Causing Theme Regressions
**Solution**: Iteratively rebalanced dark-mode contrast and moved experimental treatments into isolated preview surfaces where appropriate.

### Challenge 4: Local CSS/Hydration Instability in Development
**Solution**: Switched the dev workflow to webpack mode and repeatedly validated with clean builds to separate real CSS errors from stale dev-server state.

---

## 8. Impact Assessment

### Quantitative Impact
- **Commit Volume**: 25 commits since `77ba89c`
- **Code Change Volume**: 2,255 lines added / 539 removed in the main range inspected earlier
- **Feature Areas Improved**: drafts, campaign cards, raffles, countdowns, dark mode, modal architecture, backend reliability, dev workflow
- **New Surfaces Introduced**: reusable info modal shell and isolated rainbow preview route

### Qualitative Impact
- **User Safety**: less risk of losing campaign draft work
- **Feed Usability**: cleaner cards, better refresh behavior, better timing display
- **Raffle Clarity**: better mapping of UI to raffle mechanics
- **Developer Experience**: fewer stale frontend-state issues and clearer UI composition boundaries

---

## 9. Next Week's Priorities

### Immediate Focus
1. Finalize any remaining campaign card visual polish and special-card treatments
2. Continue refining success/confirmation modal behavior in the create flow
3. Exercise the newer draft workflows in more runtime scenarios

### Strategic Initiatives
1. Expand test coverage around create/draft/raffle behavior
2. Continue separating experimental visual treatments from core production UI
3. Continue tightening state ownership boundaries between page-level orchestration and child modal/editor components

### Documentation
1. Keep reporting and change tracking aligned with actual user-facing improvements
2. Document any finalized raffle-specific authoring/participation rules if they stabilize further

---

## 10. Lessons Learned

1. **Modal UX benefits from state ownership discipline**: separating shell behavior from content behavior made it easier to evolve confirmations and notifications.
2. **Draft systems need replacement logic, not just save/load endpoints**: the destructive edge cases are where real editor UX breaks down.
3. **Raffle flows deserve first-class semantics**: generic campaign labels and controls create confusion quickly.
4. **Frontend visual polish can easily create theme regressions**: small styling changes need to be checked against both light and dark mode continuously.
5. **Dev-server instability can obscure real progress**: switching the dev workflow and repeatedly validating builds helps separate environment issues from actual code issues.

---

## 11. Conclusion

This reporting period delivered meaningful improvements to FreightOnNervos across both user-facing workflow and technical reliability. The create campaign experience is now far more durable through stronger draft handling, the campaign feed is cleaner and more context-aware, raffle behavior is more explicit, and both dark mode and developer workflow are more stable.

The most important outcome is that the product now behaves less like a collection of disconnected surfaces and more like a coherent authoring and browsing experience, with clearer state transitions, better feedback, and stronger resilience against both UX mistakes and technical failures.

---

**Report Prepared By**: Development Team  
**Date**: May 28, 2026  
**Commit Range**: `77ba89c..HEAD`