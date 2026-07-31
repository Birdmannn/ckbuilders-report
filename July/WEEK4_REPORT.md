# Freight on Nervos - Weekly Development Report
**Reporting Period**: Since commit `6189a54`  
**Project**: Freight on Nervos  
**Branch**: v3

---

## Executive Summary

This reporting period focused on turning the freight browsing experience into a much more coherent product surface, especially around the freight detail page, shared creation flows, wallet UX, and persistent campaign identity/data handling.

The biggest area of work was the freight detail experience. What had been a comparatively thin or unstable detail path became a much richer surface with a dedicated detail card component, stable campaign lookup logic, improved header behavior, independent left/right scroll regions, relocated comments, a new mountables panel, better loading/error handling, and a more informative details modal. At the same time, a major same-pattern refactor introduced reusable creation controls and modal orchestration that now work consistently across the home feed, the freight detail page, and the profile page.

The week also improved how campaign records are matched and updated between chain and database sources, reducing fragility around campaign lookup and allowing richer freight metadata—such as mounted forms, social state, settlement metadata, and randomness preimages—to survive across screens. Wallet behavior was also brought closer to a shared standard: balances and USD values now update in the background through common hooks, introspection/action affordances mount earlier, and the profile page no longer exposes a redundant disconnect-hover modal path.

At the code level, this was a substantial frontend integration week: large route-level refactors, reusable flow extraction, new UI components, stronger campaign-record normalization, and CSS/layout work that materially changed how the application behaves. The result is a freight UI that is more stable, more legible, and much closer to a unified product rather than a collection of partially divergent screens.

**Key Metrics**:
- 16 commits since `6189a54`
- 23 files changed
- 2,538 lines added
- 1,215 lines removed
- 4 major workstreams advanced in parallel: freight detail architecture, shared create-flow/header UX, campaign identity/data synchronization, and wallet/profile interaction refinement
- 1 new dedicated right-side mountables panel introduced for freight detail pages
- 1 new reusable freight detail surface component introduced

---

## 1. Freight Detail Page Rebuild and Interaction Design

### 1.1 Dedicated Freight Detail Surface

**Problem Identified**: The freight detail page needed a more robust and expressive primary card rather than relying on lighter feed-card assumptions. It also needed clearer display of freight metadata such as type, timing, creator information, raffle parameters, transaction references, and mounted deliverables.

**Solution Implemented**:
- Added a dedicated `CampaignDetailSurface` component to own the main freight detail card
- Centralized creator/address display, title and description rendering, countdown/status display, created date, explorer link, and mounted-forms affordance
- Added detail-specific handling for raffle display elements such as ticket price and winner count
- Added description truncation/expansion behavior tailored to the detail surface rather than the feed card

**Primary Files**:
- `frontend/app/_components/CampaignDetailSurface.tsx`
- `frontend/app/campaign/[campaignId]/page.tsx`

**Impact**:
- Gave freight detail pages a dedicated, product-quality surface
- Reduced coupling between feed-card rendering and detail-card rendering
- Made freight details materially easier to inspect and reason about

### 1.2 Stable Lookup, Parsing, and Chain Hydration

**Problem Identified**: Freight detail lookup was fragile when records were addressed by different identifiers or when database and chain sources did not align cleanly.

**Solution Implemented**:
- Added more robust campaign ID parsing and normalization on the detail route
- Introduced stable campaign identity helpers for matching records to on-chain campaigns using stable IDs, tx hashes, and legacy keys
- Improved detail-page fetch and hydration flow so a persisted campaign record can be matched against live chain data more reliably
- Tightened API-side campaign record normalization to better support the richer identity model

**Primary Files**:
- `frontend/app/campaign/[campaignId]/page.tsx`
- `frontend/lib/campaignIdentity.ts`
- `frontend/app/api/campaign-records/route.ts`

**Impact**:
- Reduced detail-page lookup failures
- Improved consistency between database-backed freight metadata and live chain freight data
- Made detail-page state more dependable across direct loads and navigation

### 1.3 Detail Header, Loader, and Transition Polish

**Problem Identified**: The freight detail page needed stronger transition behavior and cleaner perceived continuity with the feed. There were also rough edges around header spacing, initial loading placement, and shell expansion/contraction.

**Solution Implemented**:
- Refined the freight detail header behavior and reduced clutter above the main card
- Relocated the detail-page loader for clearer loading-state presentation
- Added stretch/contract shell transitions between feed and detail contexts
- Introduced mount-time reveal transitions for freight cards and detail content
- Adjusted viewport-height behavior and later reduced top spacing beneath the fixed detail header

**Primary Files**:
- `frontend/app/campaign/[campaignId]/page.tsx`
- `frontend/app/styles/campaign.css`
- `frontend/app/_components/CampaignList.tsx`
- `frontend/app/_components/CampaignFeedSection.tsx`

**Impact**:
- Improved navigation continuity between feed and freight detail
- Made the detail page feel less abrupt and more intentional
- Reduced wasted vertical space and improved first-glance scanning

---

## 2. Detail Layout: Mountables, Comments, and Independent Scrolling

### 2.1 Right-Side Mountables Panel

**Problem Identified**: The right side of the freight detail page needed to become more purposeful than a duplicate comments destination. Mounted deliverables are a real freight concept and needed a dedicated detail treatment.

**Solution Implemented**:
- Replaced the right-side comments emphasis with a dedicated `CampaignMountablesPanel`
- Added mountable count display in the panel header
- Rendered mounted forms with existing mounted-icon treatment and a richer information model including summary text, metadata, proof instructions, and external link
- Built the panel from real freight record data using existing forms-mountable helpers

**Primary Files**:
- `frontend/app/_components/CampaignMountablesPanel.tsx`
- `frontend/app/campaign/[campaignId]/page.tsx`
- `frontend/app/_lib/formsMountable.ts`

**Impact**:
- Made mounted freight deliverables visible and first-class in the detail experience
- Preserved a clear mountables count for users
- Set up a generic panel shape for future mountable types beyond forms

### 2.2 Inline Comments Under the Main Freight Card

**Problem Identified**: Comments belonged closer to the primary freight content rather than in a separate bordered right-side card.

**Solution Implemented**:
- Extended `CampaignCommentsPanel` with display variants so it can render either as a card or as an inline/borderless section
- Moved comments under the main left freight card on the detail page
- Preserved existing comment-author fallback logic, timestamps, count display, and empty-state behavior

**Primary Files**:
- `frontend/app/_components/CampaignCommentsPanel.tsx`
- `frontend/app/campaign/[campaignId]/page.tsx`

**Impact**:
- Made comments feel like part of the freight narrative rather than a detached sidebar object
- Reused the existing comment rendering logic instead of forking parallel UI
- Reduced visual duplication in the detail page layout

### 2.3 Independent Left/Right Desktop Scroll Behavior

**Problem Identified**: Long freight descriptions/comments and mountables content should not force both columns to move together on desktop.

**Solution Implemented**:
- Reworked the detail layout shell so left and right columns each have their own scrollable inner container on desktop
- Added CSS support for min-height propagation, bounded column wrappers, and inner `overflow-y: auto`
- Preserved stacked, normal page scrolling on smaller/mobile widths
- Added associated styling for inline comments and mountables list presentation

**Primary Files**:
- `frontend/app/styles/campaign.css`
- `frontend/app/campaign/[campaignId]/page.tsx`

**Impact**:
- Made the detail page significantly more usable on desktop
- Prevented long left-side content from degrading right-side inspection, and vice versa
- Improved responsiveness without sacrificing the mobile fallback behavior

---

## 3. Shared Create-Flow Orchestration Across Screens

### 3.1 Extracted Create Flow State Machine

**Problem Identified**: Home, profile, and freight detail screens were carrying increasingly similar create-modal logic. Without extraction, the behavior would drift and become difficult to maintain.

**Solution Implemented**:
- Introduced a shared `useCreateCampaignFlow` hook to centralize create-modal orchestration
- Moved save-draft confirmation, create-step transitions, mountable-form validation state, submission success handling, and close/reset behavior into the shared flow layer
- Preserved the ability for each screen to plug its own info-modal mode type into the flow

**Primary Files**:
- `frontend/app/_hooks/useCreateCampaignFlow.ts`
- `frontend/app/page.tsx`
- `frontend/app/_components/ProfileScreen.tsx`
- `frontend/app/campaign/[campaignId]/page.tsx`

**Impact**:
- Reduced duplicated modal/control logic across major routes
- Made creation behavior more consistent and maintainable
- Created a reusable internal pattern for future feature work around freight creation

### 3.2 Reusable Create Launcher and Header Controls

**Problem Identified**: Create-launch affordances and top-right modal controls needed to behave consistently across pages rather than being reassembled differently each time.

**Solution Implemented**:
- Added `CreateCampaignLauncher` to own the create-modal shell, backdrop, and FAB trigger pattern
- Added `CreateCampaignHeaderActions` to own reset/draft-list/review-back behavior in a compact shared control surface
- Connected those shared components into the home, profile, and freight detail screens

**Primary Files**:
- `frontend/app/_components/CreateCampaignLauncher.tsx`
- `frontend/app/_components/CreateCampaignHeaderActions.tsx`
- `frontend/app/page.tsx`
- `frontend/app/_components/ProfileScreen.tsx`
- `frontend/app/campaign/[campaignId]/page.tsx`

**Impact**:
- Improved consistency of create-freight interaction patterns across the app
- Reduced route-specific UI drift around create controls
- Made later create-flow improvements cheaper to apply across all surfaces

### 3.3 Shared Create Info and Mounted-Forms Support

**Problem Identified**: The create flow needed a shared explanatory/info model and stronger mounted-forms support that did not live only inside one route implementation.

**Solution Implemented**:
- Added shared create-info copy helpers for preview, typing, constraints, and notes
- Expanded mounted-forms selection/validation flow in the shared create orchestration path
- Ensured mounted forms can be represented and persisted through normalized campaign records

**Primary Files**:
- `frontend/app/_lib/createCampaignInfo.ts`
- `frontend/app/_hooks/useCreateCampaignFlow.ts`
- `frontend/app/api/campaign-records/route.ts`
- `frontend/app/create/_components/CreateCampaignModalContent.tsx`

**Impact**:
- Improved consistency of create-flow messaging
- Made mounted forms more durable as a freight feature rather than a one-off UI experiment
- Better aligned persistence, validation, and UX around mountables

---

## 4. Feed, Card, and Record Model Improvements

### 4.1 Freight Card UI and Interaction Upgrades

**Problem Identified**: Feed cards needed clearer freight state presentation and interaction polish, particularly around raffle behavior, marquee/hover behavior, and mounted affordances.

**Solution Implemented**:
- Updated freight card UI and interaction behavior
- Refined marquee behavior and card interaction polish
- Expanded `useCampaignCardState` so feed cards better understand richer freight state such as raffle settlement, mounted state, and persisted social/settlement metadata
- Improved how feed cards build record payloads back into persistence

**Primary Files**:
- `frontend/app/_components/CampaignCard.tsx`
- `frontend/app/_components/CampaignCardSurface.tsx`
- `frontend/app/_hooks/useCampaignCardState.ts`
- `frontend/app/_components/MountablesPanel.tsx`

**Impact**:
- Made freight cards more informative and interactive
- Improved feed/detail consistency for raffle and mountable state
- Strengthened the connection between UI actions and persistent record updates

### 4.2 Campaign Record Schema Enrichment

**Problem Identified**: Campaign records needed to support more than basic title/description state if the app was going to retain social state, mountables, settlement information, and randomness-related metadata.

**Solution Implemented**:
- Extended campaign record typing to cover additional freight metadata
- Expanded API normalization to persist richer fields like forms mountables, social state, settlement recipient data, sold ticket count, settled participant count, and randomness preimage
- Added stronger server-side normalization/validation for mounted Google Forms

**Primary Files**:
- `frontend/app/_types/campaignRecords.ts`
- `frontend/app/api/campaign-records/route.ts`
- `frontend/app/api/campaign-records/[id]/route.ts`

**Impact**:
- Made richer freight experiences persistable
- Reduced risk that UI-level freight metadata would diverge from stored records
- Improved long-term viability of the freight record model

---

## 5. Wallet, Header, and Profile Experience Refinement

### 5.1 Shared Wallet-Info Behavior on Feed and Detail

**Problem Identified**: Wallet action mounting and USD/balance updates had diverged across routes, especially on the freight detail page.

**Solution Implemented**:
- Consolidated home/detail wallet behavior around the shared `useWalletInfo` hook and `AppShellHeader`
- Enabled background wallet fetching so balances and USD values can update without waiting for the modal to be opened
- Made the introspect/profile action mount earlier by allowing fallback username generation from the wallet address when a profile record is not yet loaded

**Primary Files**:
- `frontend/app/page.tsx`
- `frontend/app/campaign/[campaignId]/page.tsx`
- `frontend/app/_hooks/useWalletInfo.ts`
- `frontend/app/_components/AppShellHeader.tsx`

**Impact**:
- Improved consistency of wallet UX across feed and detail screens
- Reduced gating of useful wallet actions on profile-fetch timing
- Made balance/USD data feel more live and reliable

### 5.2 Profile Page Refactor and Wallet Simplification

**Problem Identified**: The profile page had accumulated a large amount of route-specific behavior and still exposed a redundant disconnect-hover modal/home action pattern.

**Solution Implemented**:
- Refactored the profile screen onto the shared header/create-flow patterns
- Preserved profile hero, leaderboard, display-name editing, and balance display while simplifying how header and create logic are wired
- Removed the disconnect hover modal from the profile page and removed the redundant Home action there, since the info button already serves that role

**Primary Files**:
- `frontend/app/_components/ProfileScreen.tsx`
- `frontend/app/_components/AppShellHeader.tsx`

**Impact**:
- Reduced redundant wallet interaction on profile
- Improved consistency between profile and the rest of the app shell
- Simplified profile-specific maintenance burden

---

## 6. Freight Information Modal Improvements

### 6.1 More Informative Freight Details Modal

**Problem Identified**: The freight detail info modal was too generic and did not explain freight-specific facts that matter to users—especially freight type and raffle randomness semantics.

**Solution Implemented**:
- Updated the freight detail info modal to display the particular freight type
- Added explicit indication of whether randomness is used
- For raffle freights, added an explanation of how randomness works, including the committed randomness hash model and deterministic winner selection inputs
- Surfaced raffle details such as ticket price, winner count, and stored randomness preimage when available

**Primary Files**:
- `frontend/app/campaign/[campaignId]/page.tsx`

**Impact**:
- Made the info modal freight-specific instead of generic
- Improved user understanding of raffle behavior and deterministic randomness
- Better exposed the product semantics already present in code and data

---

## 7. Challenges and Solutions

### Challenge 1: The freight detail route had become the convergence point for many behaviors
**Solution**: Split responsibilities into stronger units: a dedicated detail surface, reusable comments/mountables panels, shared create-flow helpers, and cleaner data-matching utilities.

### Challenge 2: Database records and chain freight cells did not always align cleanly
**Solution**: Added stable campaign identity helpers, tx-hash fallback matching, legacy-key fallback logic, and API-side normalization improvements.

### Challenge 3: Shared create behavior was at risk of drifting across routes
**Solution**: Extracted `useCreateCampaignFlow`, `CreateCampaignLauncher`, and `CreateCampaignHeaderActions` so home, profile, and detail pages now rely on the same create-flow primitives.

### Challenge 4: Detail-page usability broke down with long content in a two-column layout
**Solution**: Reworked the detail layout to support independent desktop scrolling and repositioned comments/mountables into a clearer left/right information structure.

### Challenge 5: Wallet UX had diverged between screens
**Solution**: Reused the shared wallet hook/header flow, enabled background balance fetching, and removed redundant hover-modal behavior on profile.

---

## 8. Impact Assessment

### Quantitative Impact
- **Commit Volume**: 16 commits since `6189a54`
- **Code Change Volume**: 2,538 lines added / 1,215 removed
- **Files Touched**: 23 files in the inspected range
- **Primary Areas Improved**: freight detail architecture, shared create flow, campaign identity synchronization, wallet/profile/header UX, mountables/comments layout, freight info modal content
- **Major Structural Additions**:
  - `CampaignDetailSurface`
  - `CampaignMountablesPanel`
  - `CreateCampaignLauncher`
  - `CreateCampaignHeaderActions`
  - `useCreateCampaignFlow`

### Qualitative Impact
- **Product Coherence**: The app now behaves more like a single freight product with shared patterns rather than separate route-specific implementations
- **Detail-Page Maturity**: Freight detail pages are now significantly richer, clearer, and more resilient
- **Maintainability**: Shared create-flow and wallet/header patterns reduced duplication and future rework cost
- **Data Reliability**: Stable campaign identity and normalization improvements made freight matching and persistence less brittle
- **UX Quality**: Mountables, comments placement, spacing, transitions, and modal explanations now better reflect the product’s actual concepts

---

## 9. Next Week's Priorities

### Immediate Focus
1. Continue refining the freight detail experience, especially around final polish of spacing, copy, and modal clarity
2. Extend mountables beyond the current forms-first implementation where appropriate
3. Keep hardening campaign record persistence and synchronization between chain data and stored metadata

### Strategic Initiatives
1. Continue unifying app-shell behavior across feed, profile, and freight detail pages
2. Build on the shared create-flow foundation to support more creation modes without route duplication
3. Further improve freight-type explainability, especially for raffle settlement and mounted deliverables

### Documentation
1. Keep weekly reports current as major UX and architecture work continues
2. Replace the placeholder project README with a clearer product and architecture overview
3. Document the freight data model and stable campaign identity approach more explicitly

---

## 10. Lessons Learned

1. **Freight detail pages deserve their own architecture**: trying to stretch feed-card assumptions too far leads to brittle behavior and unclear UI.
2. **Shared flow extraction pays off quickly**: once create behavior spans multiple routes, centralizing it reduces both bugs and divergence.
3. **Data identity matters as much as UI polish**: a good detail experience depends on robust matching between persisted records and live chain data.
4. **Layout is product behavior, not just styling**: independent scroll regions, comments placement, and header spacing materially affect usability.
5. **Info modals should explain real product semantics**: users benefit when freight type, randomness usage, and settlement behavior are surfaced explicitly.

---

## 11. Conclusion

This reporting period substantially strengthened Freight on Nervos as a usable frontend product. The biggest gains came from rebuilding the freight detail page into a much more capable experience, extracting shared create-flow infrastructure across major screens, improving campaign identity/data synchronization, and reducing wallet/header inconsistency.

The codebase is now in a better position to evolve because major user-facing workflows—viewing freights, creating freights, inspecting mountables, understanding raffle behavior, and interacting with wallet controls—are more clearly structured and less dependent on one-off page logic. The detail-page rebuild in particular moved a large amount of functionality from ad hoc behavior into deliberate product architecture.

---

**Report Prepared By**: Birdmannn  
**Date**: July 31, 2026  
**Commit Range**: `6189a54..6ddfd07`
