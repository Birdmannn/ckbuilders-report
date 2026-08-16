# Freight on Nervos - Weekly Development Report
**Reporting Period**: Since commit `f979cc9`  
**Project**: Freight on Nervos  
**Branch**: v3

---

## Executive Summary

This reporting period focused on turning two previously partial systems into more complete product flows: mounted Google Forms verification on the freight detail page, and direct creator tipping for `SimpleTask` campaigns.

The first major workstream was a large expansion of the freight detail page. Mounted forms are no longer just descriptive metadata: creators can now link a Google account with response-read access, verify that access against the mounted form, sync pending participant claims, and manually approve or reject claims. Participants can link their Google identity, submit a verification check, and see whether their claim is pending, verified, or rejected. This is the first time the mounted-forms infrastructure added in the prior range has been brought together into a real end-user workflow on the freight detail surface.

The second major workstream changed how support funding behaves for `SimpleTask` campaigns. Instead of treating support like a normal escrow deposit into the campaign cell, deposits for `SimpleTask` are now routed as direct creator tips. That required coordinated contract changes, client transaction-building updates, card-state logic, API persistence updates, FBARS attribution changes, and profile transaction-history labeling so the behavior is visible and consistent across the stack.

A related refinement improved raffle behavior in the feed by separating “support” from “buy ticket” interactions and by tracking live sold-ticket counts from participant activity rather than relying only on deposited totals. Together, these changes make freight interactions more legible: mounted verification is actionable on detail pages, `SimpleTask` support behaves more like patronage, and raffle ticket state is closer to real-time usage.

**Key Metrics**:
- 2 commits in the inspected range
- 26 files changed
- 1,241 lines added
- 92 lines removed
- 1 major new helper module added:
  - `frontend/lib/campaignTipping.ts`
- 3 parallel workstreams advanced:
  - mounted Google Forms verification on freight detail pages
  - direct tipping/support for `SimpleTask`
  - raffle support/ticket-state refinement

---

## 1. Mounted Google Forms Verification Reached the Freight Detail Page

### 1.1 Creator-Side Response Access and Claim Review Flow

**Problem Identified**: The underlying Google-linking and forms-verification infrastructure already existed, but creators still needed a complete UI to actually operate mounted forms after a freight was published.

**Solution Implemented**:
- Added creator-side mounted-form actions directly on the freight detail page
- Enabled creators to:
  - link or relink a Google account for `forms_response_access`
  - refresh the linked grant
  - verify that the linked account can read the mounted form’s responses
  - reload and sync pending participant claims
  - manually approve or reject pending claims
- Persisted verified response-access metadata back into mounted form state on the selected freight record
- Surfaced pending/verified counts and claim review history in the mounted-form UI

**Primary Files**:
- `frontend/app/campaign/[campaignId]/page.tsx`
- `frontend/app/_components/CampaignMountablesPanel.tsx`

**Impact**:
- Made creator-side mounted-form moderation possible from the freight detail screen
- Turned response-access verification into an operational workflow instead of a hidden backend capability
- Gave creators a practical path to manage late or pending Google Forms matches

### 1.2 Participant-Side Google Identity Linking and Submission Verification

**Problem Identified**: Participants needed a simple path to prove that they submitted a mounted Google Form using the same linked identity, without relying on separate admin workflows.

**Solution Implemented**:
- Added participant-side mounted-form actions on the freight detail page
- Enabled participants to:
  - link or relink their Google identity
  - trigger claim verification against the mounted form
  - re-check pending submissions
  - see whether their claim is pending, verified, or rejected
  - view review notes and last-updated timestamps when present
- Added clearer participant notices for “verified,” “pending,” “rejected,” and “no matching response yet” outcomes

**Primary Files**:
- `frontend/app/campaign/[campaignId]/page.tsx`
- `frontend/app/_components/CampaignMountablesPanel.tsx`

**Impact**:
- Brought mounted-form verification into the participant-facing product flow
- Reduced ambiguity around what a participant should do after submitting a Google Form
- Closed more of the gap between mounted-form metadata and mounted-form usage

### 1.3 Mounted Forms Became Interactive Mountables

**Problem Identified**: Mounted forms were visible as part of freight metadata, but the panel itself needed to support richer action blocks and verification-specific UX.

**Solution Implemented**:
- Extended mountable item rendering so mounted forms can include custom action content, status messages, counts, and review controls
- Wired the detail page to build action-rich mountable items rather than only static summaries/links
- Preserved mounted-form metadata such as form id, validation date, proof instructions, and canonical form URL while layering interactive controls on top

**Primary Files**:
- `frontend/app/campaign/[campaignId]/page.tsx`
- `frontend/app/_components/CampaignMountablesPanel.tsx`

**Impact**:
- Shifted mounted forms from passive attachments to active verification surfaces
- Made the freight detail page the primary place to complete mounted-form verification work
- Improved cohesion between freight metadata, creator moderation, and participant actions

---

## 2. `SimpleTask` Support Was Reworked into Direct Creator Tipping

### 2.1 Contract-Side Deposit Semantics Changed for `SimpleTask`

**Problem Identified**: `SimpleTask` support was being handled like a normal campaign escrow deposit, but the intended behavior was closer to direct patronage of the creator.

**Solution Implemented**:
- Updated deposit validation so `SimpleTask` campaigns now follow a creator-tip transfer path instead of campaign-cell capacity growth
- Added `validate_creator_tip_transfer` to ensure:
  - the campaign cell itself does not absorb the extra support capacity
  - the tip is emitted as a dedicated output
  - the destination output lock matches the freight creator’s address
  - the tipped capacity equals the requested deposit amount
- Kept non-`SimpleTask` campaigns on the existing escrow-style deposit path
- Preserved the broader rule that campaigns in `Created` status can still accept support/funding, while changing how `SimpleTask` realizes that support

**Primary Files**:
- `contracts/freight/src/instructions.rs`
- `contracts/freight/src/types.rs`
- `contracts/freight/src/validations.rs`

**Impact**:
- Changed `SimpleTask` support from campaign escrow to direct creator payout
- Better aligned contract behavior with the intended product model for lightweight tasks
- Created a cleaner distinction between “fund a freight pool” and “tip the creator”

### 2.2 Frontend Support Logic Now Distinguishes Deposits from Tips

**Problem Identified**: Once `SimpleTask` support changed on-chain, the client also needed to differentiate support modes and present them clearly in the UI.

**Solution Implemented**:
- Added a dedicated `campaignTipping` helper module to centralize support-mode derivation and control-tag handling
- Introduced support-mode state that distinguishes:
  - `campaign_escrow`
  - `direct_creator`
- Introduced deposit-kind state that distinguishes:
  - `campaign_deposit`
  - `simple_task_tip`
- Added a `#non-tippable` control tag so support can be disabled intentionally for eligible descriptions
- Updated card-level support buttons, tooltips, labels, and modal copy so `SimpleTask` campaigns now read as “Tip creator” rather than “Deposit CKB”
- Added user-facing copy clarifying that `SimpleTask` tips go straight to the creator address

**Primary Files**:
- `frontend/lib/campaignTipping.ts`
- `frontend/app/_hooks/useCampaignCardState.ts`
- `frontend/app/_components/CampaignCard.tsx`
- `frontend/app/_components/CampaignCardSurface.tsx`
- `frontend/app/_components/CampaignDetailSurface.tsx`

**Impact**:
- Made the new support behavior understandable from the freight cards themselves
- Reduced mismatch between contract semantics and UI wording
- Added a cleaner support model for creators who want patronage rather than pooled funding

### 2.3 Client Transaction Building and Persistence Were Updated for Tipping

**Problem Identified**: Direct creator tips required different transaction construction, record persistence, and downstream bookkeeping than normal campaign deposits.

**Solution Implemented**:
- Added `sendCreatorTipShannons` and related helpers to build a direct tip transaction
- Enforced a minimum tip threshold so the creator can receive a standalone CKB output cell
- Updated deposit persistence APIs to record `kind` and `supportMode`
- Updated FBARS deposit handling so downstream event metadata also records whether support was escrow-backed or creator-directed
- Updated freight transaction-history assembly so profile timelines can label tip behavior differently from ordinary deposits

**Primary Files**:
- `frontend/lib/transactions.ts`
- `frontend/app/api/campaign-deposits/route.ts`
- `frontend/app/api/fbars/deposit/route.ts`
- `frontend/app/api/user-profiles/transactions/route.ts`

**Impact**:
- Ensured direct tips are represented consistently across chain interaction, Mongo persistence, FBARS events, and user profile history
- Created the data needed to analyze or present tip behavior separately from standard freight deposits
- Prevented `SimpleTask` tipping from becoming a one-off frontend-only behavior

---

## 3. Raffle Interaction State and Support UX Were Refined

### 3.1 Live Sold-Ticket Counts Now Track Participant Activity More Directly

**Problem Identified**: Raffle ticket state was too dependent on deposited totals, which could drift from the real number of participant entries and make live ticket availability harder to represent correctly.

**Solution Implemented**:
- Introduced `liveSoldTicketCount` handling in campaign record state
- Updated campaign-feed hydration to derive raffle sold-ticket counts from participant counts when available
- Updated ticket-purchase callbacks so the next sold-ticket count is propagated immediately after a ticket purchase
- Adjusted display helpers so settlement and live-state rendering can choose the right sold-ticket number depending on campaign status

**Primary Files**:
- `frontend/app/_hooks/useCampaignFeed.ts`
- `frontend/app/_hooks/useCampaignCardState.ts`
- `frontend/app/_hooks/useTicketPurchaseFlow.ts`
- `frontend/lib/campaignDisplay.ts`
- `frontend/app/_types/campaignRecords.ts`
- `frontend/app/api/campaign-records/route.ts`
- `frontend/app/api/campaign-records/[id]/route.ts`

**Impact**:
- Improved accuracy of live raffle inventory shown in the feed
- Reduced reliance on inferred state from deposit totals alone
- Prepared the product for clearer settlement and participation accounting

### 3.2 Support and Ticket Purchase Actions Were Separated More Cleanly

**Problem Identified**: Raffle cards needed clearer action boundaries so users could distinguish supporting a raffle from actually buying a ticket.

**Solution Implemented**:
- Updated campaign card behavior and state logic so raffle surfaces can expose both support and ticket-purchase actions distinctly
- Refined disabled-state logic for remaining tickets, inactive freight, and lock-gated interaction states
- Preserved settlement-related actions while making sold-ticket and remaining-ticket affordances easier to interpret

**Primary Files**:
- `frontend/app/_components/CampaignCard.tsx`
- `frontend/app/_hooks/useCampaignCardState.ts`
- `frontend/app/_components/CampaignFeedSection.tsx`
- `frontend/app/_components/CampaignList.tsx`
- `frontend/app/page.tsx`

**Impact**:
- Made raffle interaction choices clearer at the card level
- Reduced confusion between “support this freight” and “enter this raffle”
- Improved feed-level UX for one of the most interaction-heavy freight types

### 3.3 Gift Logic Was Tightened Around Escrow-Oriented Campaigns

**Problem Identified**: Once `SimpleTask` moved to direct tipping, gift-deliverable assumptions also needed to stay aligned with the remaining escrow-backed flows.

**Solution Implemented**:
- Narrowed or aligned gift-related logic so gift behavior stays coupled to the freight types and funding semantics that still fit the escrow-based model
- Updated create-flow and helper logic accordingly

**Primary Files**:
- `frontend/lib/giftDeliverables.ts`
- `frontend/app/create/_components/CreateCampaignModalContent.tsx`

**Impact**:
- Reduced the risk of gift mechanics drifting out of sync with support semantics
- Kept the product model more coherent as `SimpleTask` behavior diverged from pooled-funding campaigns

---

## 4. Challenges and Solutions

### Challenge 1: Mounted-form infrastructure existed, but the product workflow was incomplete
**Solution**: Concentrated the missing work on the freight detail page and mountables panel so linking, verification, syncing, and manual review all live in one visible surface.

### Challenge 2: Changing `SimpleTask` support semantics required full-stack coordination
**Solution**: Updated the contract path, frontend transaction builder, support-state helpers, API persistence, FBARS metadata, and transaction-history labeling together.

### Challenge 3: Raffle counts needed to feel live without breaking settlement behavior
**Solution**: Added `liveSoldTicketCount` as a separate tracked concept and used participant-driven counts for active raffles while preserving settlement snapshot handling.

### Challenge 4: The UI needed to distinguish multiple support behaviors without adding confusion
**Solution**: Introduced explicit support modes and deposit kinds, then reflected them in action labels, tooltips, modal copy, and downstream record metadata.

---

## 5. Impact Assessment

### Quantitative Impact
- **Commit Volume**: 2 commits in the inspected range
- **Code Change Volume**: 1,241 lines added / 92 removed
- **Files Touched**: 26 files in the inspected range
- **Largest Concentrated Change**: `frontend/app/campaign/[campaignId]/page.tsx` received the biggest single expansion, accounting for most of the mounted-form detail-page workflow
- **Frontend vs Contract Mix**:
  - 23 frontend/UI/API files changed
  - 3 contract-side Rust files changed

### Qualitative Impact
- **Mounted Verification Maturity**: Mounted Google Forms now have a true operational detail-page flow for both creators and participants
- **Support Model Clarity**: `SimpleTask` support now behaves and reads like tipping rather than escrow funding
- **Data Model Quality**: Support interactions now carry richer metadata (`kind`, `supportMode`, `liveSoldTicketCount`) across persistence and display layers
- **Raffle Readability**: Ticket inventory and card actions are better aligned with real participant behavior
- **System Coherence**: Contract behavior, frontend UX, and profile/history reporting are more closely synchronized than before

---

## 6. Next Week's Priorities

### Immediate Focus
1. Continue polishing mounted-form verification UX on the freight detail page, especially around edge cases and creator moderation flows
2. Harden the direct creator tipping path with more real-world validation and transaction-flow testing
3. Continue refining raffle card state so support, entry, and settlement behavior stay readable across all statuses

### Strategic Initiatives
1. Extend mountables further so future freight-linked apps or deliverables can reuse the same action-oriented panel pattern
2. Keep separating support semantics by campaign type where product behavior meaningfully differs
3. Improve reporting and analytics around support interactions now that tip-vs-deposit metadata is available

### Documentation
1. Keep weekly reports current as mounted verification and tipping flows evolve
2. Document the direct creator tipping model alongside existing freight funding semantics
3. Add more implementation notes around how mounted-form verification and claim review are expected to work end-to-end

---

## 7. Lessons Learned

1. **Infrastructure is only half the feature**: mounted-form support became much more valuable once the detail page exposed the real creator and participant workflows.
2. **Funding semantics should match product intent**: `SimpleTask` support is clearer when treated as creator tipping instead of generic escrow funding.
3. **Metadata matters downstream**: explicit `kind`, `supportMode`, and live ticket-count fields make UI, analytics, and profile history more reliable.
4. **Raffle state needs two views of truth**: live activity and settled history should be tracked separately so the feed can stay responsive without losing post-settlement integrity.
5. **One large surface can unlock a lot of dormant work**: most of the mounted-forms progress came from fully wiring the freight detail page rather than scattering small updates across many screens.

---

## 8. Conclusion

This reporting period pushed the freight product from infrastructure-heavy groundwork into more usable interaction flows. Mounted Google Forms now support real creator moderation and participant verification on the freight detail page, which makes the earlier Google-linking and forms-claim backend work materially useful. At the same time, `SimpleTask` campaigns were given a clearer support identity through direct creator tipping, backed by coordinated changes across the contract, client transactions, persistence, FBARS events, and profile history.

The result is a more coherent product model: mounted verification now has an end-user workflow, support semantics are better aligned with campaign intent, and raffle cards reflect live ticket state more accurately. There is still follow-up work to polish and harden these systems, but the core behavior for this next slice of freight interaction is now in place.

---

**Report Prepared By**: Birdmannn  
**Date**: August 28, 2026  
**Commit Range**: `f979cc9..1bb7ff5`
