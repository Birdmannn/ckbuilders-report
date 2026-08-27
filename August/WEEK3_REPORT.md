# Freight on Nervos - Weekly Development Report
**Reporting Period**: Since commit `1bb7ff5`  
**Project**: Freight on Nervos  
**Branch**: v3

---

## Executive Summary

This reporting period focused on two large expansions of the Freight model: introducing a first real platform for third-party app mountables, and substantially upgrading raffle accounting so creator support, ticket sales, and withdrawable escrow are treated as distinct concepts.

The first major workstream pushed mountables beyond forms and locks into a more general app-based system. Campaign creators can now discover registered SDK apps, verify an install token against an app-provided verification endpoint, choose which principles that app should enforce, and attach verified app mountables directly to a freight. The backend now supports app registration, app listing, install verification, and signed update ingestion from mounted apps so participant eligibility can be tracked over time.

The second major workstream reshaped raffle mechanics on-chain and in the frontend. Raffles now track ticket-sales escrow separately from creator support, can optionally route a percentage of creator support into the reward pool, and expose a creator-withdraw path when remaining support is no longer part of the raffle pool. This is a meaningful model shift: raffle economics are no longer represented as one undifferentiated deposit balance.

A third supporting workstream improved end-to-end persistence and refresh behavior across create, detail, feed, and profile surfaces. Campaign records now preserve mounted apps, raffle support-pool percentage, and creator-withdraw metadata; profile tabs now use dirtiness-aware refresh flows; and the freight detail page and card surfaces are better aligned with the richer record model.

This period also included product-concept groundwork for an NDAO/iCKB-based raffle mountable. While that document is not a shipped runtime feature by itself, it captures the direction for a transparent, round-based, yield-backed raffle experience that could sit naturally inside the broader mountables framework.

**Key Metrics**:
- 2 commits in the inspected range
- 49 files changed
- 4,894 lines added
- 892 lines removed
- 12 new files added
- 5 major new API routes added:
  - `frontend/app/api/mountables/apps/register/route.ts`
  - `frontend/app/api/mountables/apps/route.ts`
  - `frontend/app/api/mountables/apps/verify/route.ts`
  - `frontend/app/api/mountables/apps/[mountableInstanceId]/updates/route.ts`
  - `frontend/app/api/campaign-records/[id]/withdraw/route.ts`
- 4 major workstreams advanced in parallel:
  - third-party app mountables
  - raffle accounting and creator withdraw flows
  - record persistence and profile/feed/detail refresh behavior
  - NDAO raffle mountable concept design

---

## 1. Third-Party App Mountables Became a Real Freight Capability

### 1.1 Registered App Manifests, SDK Helpers, and Verification Routes

**Problem Identified**: Forms and locks established the mountables concept, but Freight still lacked a general mechanism for attaching external apps with their own eligibility logic and installation lifecycle.

**Solution Implemented**:
- Added a new SDK-oriented app mountables layer with shared normalization, hashing, URL validation, JSON config handling, and principle-selection helpers
- Added backend routes for:
  - registering mountable apps
  - listing registered apps
  - verifying an app installation via the app’s own `verifyInstallUrl`
  - ingesting later app updates for mounted instances
- Added timestamp-window derivation helpers so mounted apps can receive campaign timing context in a normalized way
- Standardized app manifest fields such as `appId`, `appName`, `description`, `principles`, `verifyInstallUrl`, `supportsTimestampQuery`, and defaults/config schema

**Primary Files**:
- `frontend/lib/fonMountablesSdk.ts`
- `frontend/lib/mountableTiming.ts`
- `frontend/app/api/mountables/apps/register/route.ts`
- `frontend/app/api/mountables/apps/route.ts`
- `frontend/app/api/mountables/apps/verify/route.ts`
- `frontend/app/api/mountables/apps/[mountableInstanceId]/updates/route.ts`
- `frontend/lib/mongodb.ts`

**Impact**:
- Created a reusable platform layer for Freight-integrated apps instead of special-casing every future mountable
- Gave external apps a clear registration and verification contract
- Made mounted apps first-class backend entities rather than purely frontend configuration

### 1.2 Creator-Side App Mounting in the Create Flow

**Problem Identified**: Even with backend registration and verification support, creators still needed a practical UI for discovering, configuring, verifying, and attaching app mountables during freight creation.

**Solution Implemented**:
- Added a dedicated `MountableAppsConfigurator` surface in the create flow
- Enabled creators to:
  - browse registered apps
  - select an app to mount
  - enter an install token
  - choose one or more published principles
  - verify the installation before saving it into the freight
  - remove previously mounted app configs
- Extended create-flow state to track mounted app configs, selected principle IDs, install token input, verification loading/error states, and mountables summary counts
- Persisted verified app mountables back into draft/publish payloads alongside forms and locks

**Primary Files**:
- `frontend/app/_components/MountableAppsConfigurator.tsx`
- `frontend/app/_hooks/useCreateCampaignFlow.ts`
- `frontend/app/create/_components/CreateCampaignModalContent.tsx`
- `frontend/app/_lib/appMountable.ts`
- `frontend/app/_lib/appMountablePayload.ts`
- `frontend/app/_types/appMountable.ts`
- `frontend/app/_types/campaignRecords.ts`

**Impact**:
- Made app mountables usable from the freight creation workflow rather than only from backend APIs
- Extended Freight’s creation model from “content + mountable metadata” toward “content + verified external app capability”
- Established a pattern that future SDK apps can plug into without redesigning the create surface

### 1.3 App-Sourced Participant Updates and Principle Satisfaction Tracking

**Problem Identified**: Mounted apps needed a secure way to report participant state back into Freight so campaign eligibility could reflect app-specific actions and not just manual creator review.

**Solution Implemented**:
- Added mounted-app update ingestion keyed by `mountableInstanceId`
- Secured updates using an installation secret hash derived during app verification
- Normalized per-principle fulfillment state, timestamps, and status messages from app updates
- Computed both:
  - `childSatisfied` for the specific mounted app instance
  - `parentSatisfied` across all mounted apps on the freight
- Extended participant records so mounted app state can be stored and revisited over time
- Recorded update history separately for auditability and future debugging

**Primary Files**:
- `frontend/app/api/mountables/apps/[mountableInstanceId]/updates/route.ts`
- `frontend/app/api/campaign-participants/route.ts`
- `frontend/app/api/campaign-participants/[campaignId]/review/route.ts`
- `frontend/app/_lib/appMountable.ts`
- `frontend/app/_types/appMountable.ts`

**Impact**:
- Enabled mounted apps to become active eligibility providers rather than decorative attachments
- Added a path for principle-based participant verification that can evolve outside the core Freight app
- Improved the long-term viability of mountables as a true extensibility layer

---

## 2. Raffle Accounting Was Reworked into Separate Pools and Withdraw Flows

### 2.1 The Campaign Cell Model Now Separates Ticket Sales from Creator Support

**Problem Identified**: Raffle deposits had previously been treated too much like one general funding bucket, which made it hard to express distinct rules for ticket revenue, creator support, and support-pool sharing.

**Solution Implemented**:
- Expanded the on-chain `Campaign` layout from 174 bytes to 198 bytes
- Added new campaign fields for:
  - `ticket_sales_total`
  - `creator_support_total`
  - `support_pool_bps`
- Updated campaign creation args and encoding/decoding to carry a raffle support-pool percentage end to end
- Tightened campaign validation so `support_pool_bps` is only valid where the raffle model supports it

**Primary Files**:
- `contracts/freight/src/types.rs`
- `contracts/freight/src/utils.rs`
- `contracts/freight/src/instructions.rs`
- `contracts/freight/src/main.rs`
- `contracts/freight/src/validations.rs`
- `frontend/lib/encoding.ts`
- `frontend/lib/contract.ts`
- `frontend/lib/campaignValidation.ts`

**Impact**:
- Replaced the older “single escrow bucket” model with a more expressive raffle accounting structure
- Created the on-chain basis for richer raffle economics and clearer creator-support handling
- Improved alignment between business rules and data representation

### 2.2 Ticket Revenue and Creator Support Now Affect Reward Pools Differently

**Problem Identified**: Raffle payout logic needed to distinguish between ticket-funded reward pools and optional creator-support contributions, especially once direct support semantics became more important elsewhere in the product.

**Solution Implemented**:
- Updated raffle deposit/application logic so creator support can be tracked separately from ticket-purchase revenue
- Added support-pool basis points so a raffle can route some or all creator support into the eventual reward pool
- Updated reward-pool calculation so payouts come from:
  - ticket sales total
  - plus any configured support-pool contribution
- After settlement, preserved the remaining creator-attributable support balance separately from ticket-funded payout logic

**Primary Files**:
- `contracts/freight/src/types.rs`
- `contracts/freight/src/instructions.rs`
- `frontend/lib/transactions.ts`
- `frontend/lib/campaignDisplay.ts`
- `frontend/app/_hooks/useCampaignCardState.ts`
- `frontend/app/create/_components/CreateCampaignModalContent.tsx`

**Impact**:
- Made raffle economics more transparent and configurable
- Allowed creators to choose whether support partly boosts the prize pool
- Prevented creator support from being silently merged into ticket-based winner payouts

### 2.3 Creator Withdrawals Were Added for Remaining Raffle Support

**Problem Identified**: Once raffle support and ticket sales were separated, the system needed an explicit path for creators to withdraw remaining creator support when it was no longer locked into the reward pool.

**Solution Implemented**:
- Added a new on-chain `creator_withdraw` instruction
- Added validation to ensure the withdraw output:
  - reduces campaign escrow by the exact withdrawable amount
  - pays out to the creator’s address
  - preserves campaign-state correctness after the transfer
- Added frontend transaction-building for creator withdrawals
- Added an authenticated backend route to record withdraw metadata on campaign records
- Updated campaign cards/feed state so successful withdraws update local UI and persisted record state

**Primary Files**:
- `contracts/freight/src/instructions.rs`
- `contracts/freight/src/validations.rs`
- `frontend/lib/transactions.ts`
- `frontend/app/api/campaign-records/[id]/withdraw/route.ts`
- `frontend/app/_hooks/useCampaignCardState.ts`
- `frontend/app/_hooks/useCampaignFeed.ts`
- `frontend/app/_components/CampaignCard.tsx`
- `frontend/app/_components/CampaignList.tsx`

**Impact**:
- Added a real lifecycle endpoint for creator-owned support after raffle completion/cancellation conditions are met
- Reduced ambiguity around which escrow remains prize-bound versus creator-withdrawable
- Completed the raffle accounting model with a visible, actionable creator exit path

---

## 3. Campaign Records and UI Surfaces Were Upgraded for Richer Mountable State

### 3.1 Campaign Records Now Preserve App Mountables and Withdraw Metadata

**Problem Identified**: The campaign record schema needed to carry more than forms/locks and generic raffle metadata if the new app platform and withdraw flows were going to persist cleanly.

**Solution Implemented**:
- Extended campaign-record typing and persistence to support:
  - `mountables.apps`
  - `raffleSupportPoolPercent`
  - `withdrawalTxHash`
  - `withdrawnAt`
  - `withdrawnByAddress`
  - `withdrawnAmountShannons`
- Updated both campaign-record create and update routes so the new fields are validated and stored safely
- Kept draft/published record handling aligned with the growing freight model

**Primary Files**:
- `frontend/app/_types/campaignRecords.ts`
- `frontend/app/api/campaign-records/route.ts`
- `frontend/app/api/campaign-records/[id]/route.ts`

**Impact**:
- Made the new mountable-app and withdraw workflows durable across refreshes and page transitions
- Reduced the risk that advanced freight state would live only in local UI state
- Improved the long-term schema readiness of campaign records

### 3.2 Detail, Feed, and Card Surfaces Now Reflect the Expanded Model

**Problem Identified**: Once app mountables and withdrawable raffle support existed in the data model, the major frontend surfaces needed to present them coherently.

**Solution Implemented**:
- Extended the freight detail page to carry mounted app metadata and richer record state
- Updated cards and feed/list state so withdrawal readiness, support-pool metadata, and app-related state can flow through interaction surfaces
- Kept mounted forms state more persistently recorded while adding the new app-mountable layer
- Added supporting visual and metadata treatment in campaign UI components and styles

**Primary Files**:
- `frontend/app/campaign/[campaignId]/page.tsx`
- `frontend/app/_components/CampaignCard.tsx`
- `frontend/app/_components/CampaignCardSurface.tsx`
- `frontend/app/_components/CampaignFeedSection.tsx`
- `frontend/app/_components/CampaignList.tsx`
- `frontend/app/_components/CampaignMountablesPanel.tsx`
- `frontend/app/styles/campaign.css`
- `frontend/lib/campaignDisplay.ts`

**Impact**:
- Kept the UI aligned with the richer campaign record and contract model
- Made advanced raffle and mountable state more visible and actionable
- Reduced the chance that backend feature growth would outpace what users can actually see or do

---

## 4. Profile and App Refresh Behavior Became More Deliberate

### 4.1 Dirtiness-Aware Profile Caching and Tab Refreshes

**Problem Identified**: As freight interactions became richer, profile analytics, freights, and transaction history needed a more reliable way to refresh after publish, verification, and other state-changing actions.

**Solution Implemented**:
- Added dirty-key tracking for profile analytics, freights, and transactions
- Updated profile hooks to refresh when cached data is marked stale rather than relying only on naive initial fetch behavior
- Improved user-profile caching and invalidation behavior so state changes propagate more cleanly through the app
- Explicitly marked related profile views dirty after campaign actions from feed, profile, and detail surfaces

**Primary Files**:
- `frontend/app/_hooks/useProfileAnalytics.ts`
- `frontend/app/_hooks/useProfileFreights.ts`
- `frontend/app/_hooks/useProfileTransactions.ts`
- `frontend/app/_hooks/useUserProfile.ts`
- `frontend/app/page.tsx`
- `frontend/app/_components/ProfileScreen.tsx`
- `frontend/app/campaign/[campaignId]/page.tsx`
- `frontend/app/providers.tsx`

**Impact**:
- Improved freshness of profile tabs after user actions
- Reduced stale-data drift between feed, detail, and profile views
- Made the frontend state model more resilient as Freight interactions became more stateful

### 4.2 Feed Hydration and Card Orchestration Continued to Mature

**Problem Identified**: With app mountables, withdrawable support, and richer raffle metadata landing together, feed orchestration needed cleanup so these newer states could coexist smoothly.

**Solution Implemented**:
- Refined campaign-feed merging and refresh behavior
- Updated campaign-card state derivation for support-pool percentages, withdraw readiness, and new record fields
- Preserved earlier raffle/tipping behavior while layering in the next set of freight lifecycle controls

**Primary Files**:
- `frontend/app/_hooks/useCampaignFeed.ts`
- `frontend/app/_hooks/useCampaignCardState.ts`
- `frontend/app/_components/CampaignFeedSection.tsx`
- `frontend/app/page.tsx`

**Impact**:
- Helped keep the main freight feed coherent while the underlying model expanded significantly
- Reduced friction between newly added lifecycle states and the existing card interaction system
- Improved the maintainability of feed/card orchestration under a more complex freight model

---

## 5. NDAO Raffle Mountable Concept Was Documented

### 5.1 DAO/iCKB Yield-Pooled Raffle Direction

**Problem Identified**: The mountables framework needed clearer forward-looking product direction for more advanced financial or yield-based freight experiences.

**Solution Implemented**:
- Added a dedicated concept document for an NDAO raffle mountable
- Framed the concept as:
  - transparent
  - opt-in
  - round-based
  - yield-backed via NervosDAO and iCKB composability
- Defined the key product principle that users must understand they are pooling yield for probabilistic rewards rather than simply performing ordinary DAO staking
- Outlined open questions around round structure, ticket formulas, winner tapering, exit-or-stay behavior, and principal/yield treatment

**Primary Files**:
- `NDAO.md`

**Impact**:
- Captured a concrete direction for a more advanced raffle mountable family
- Connected the new app-mountables framework to a plausible next-generation product concept
- Improved documentation around how Freight could present more complex financial interactions honestly

---

## 6. Challenges and Solutions

### Challenge 1: Mountables needed to evolve from one-off types into a general extension system
**Solution**: Introduced a manifest-driven app mountables platform with registration, verification, SDK normalization helpers, and participant update ingestion.

### Challenge 2: Raffle funding semantics had become too coarse
**Solution**: Split raffle accounting into ticket sales, creator support, and support-pool contribution fields, then updated reward and withdraw logic around those distinctions.

### Challenge 3: New mountable and raffle states risked living only in local UI memory
**Solution**: Extended campaign record persistence so app mountables, support-pool percentages, and withdrawal metadata survive across create, detail, feed, and profile flows.

### Challenge 4: Richer freight actions made stale profile data more likely
**Solution**: Added dirty-refresh patterns and explicit invalidation hooks so profile analytics, freights, and transactions refresh more intentionally after user actions.

### Challenge 5: Advanced raffle concepts needed honest framing before implementation
**Solution**: Wrote the NDAO/iCKB concept doc with explicit transparency rules and user-consent framing instead of treating it like ordinary DAO yield behavior.

---

## 7. Impact Assessment

### Quantitative Impact
- **Commit Volume**: 2 commits in the inspected range
- **Code Change Volume**: 4,894 lines added / 892 removed
- **Files Touched**: 49 files in the inspected range
- **New Files Added**: 12
- **Largest Structural Themes**:
  - app mountables platform and SDK helpers
  - raffle accounting / creator withdraw support
  - profile and feed refresh/state-management upgrades
- **Contract Model Expansion**: campaign cell data increased from **174 bytes** to **198 bytes**

### Qualitative Impact
- **Extensibility**: Freight can now begin to host third-party apps as mounted capability providers rather than only static metadata types
- **Economic Clarity**: Raffles now represent ticket revenue, creator support, and prize-pool contribution more honestly and flexibly
- **Lifecycle Completeness**: Creator support on raffles now has an explicit post-campaign withdraw path
- **State Durability**: Campaign records and profile hooks are better equipped for richer, more stateful freight interactions
- **Product Direction**: The NDAO concept gives the mountables system a clearer next-step vision beyond today’s shipped form and lock mechanics

---

## 8. Next Week's Priorities

### Immediate Focus
1. Harden the app-mountables verification and update-ingestion flow with more end-to-end testing across real SDK apps
2. Continue refining raffle withdraw and support-pool UX so creators and participants can understand the new economics clearly
3. Polish detail and feed presentation for mounted apps, support percentages, and withdrawal states

### Strategic Initiatives
1. Expand the mountables framework into a stable extension surface for future third-party and protocol-linked apps
2. Continue separating raffle business rules into explicit accounting buckets rather than overloading generic deposit logic
3. Build on the NDAO/iCKB concept with clearer implementation constraints and first-version boundaries

### Documentation
1. Keep weekly reports current as the mountables platform and raffle model continue to evolve
2. Document the mounted-app manifest and verification contract for app developers
3. Add developer-facing notes on creator-withdraw behavior and support-pool accounting

---

## 9. Lessons Learned

1. **Mountables become much more powerful when they are platformized**: a general manifest/verification/update model creates more leverage than shipping each new mountable as a standalone special case.
2. **Economic semantics need dedicated fields**: once raffles involve ticket sales, creator support, and optional prize-pool sharing, a single “deposits” number is not enough.
3. **Persistence has to grow with feature depth**: richer freight workflows quickly break down if app state, withdraw state, or support-pool config is not stored durably.
4. **Frontend freshness matters more as workflows become stateful**: dirty-refresh logic becomes necessary once multiple screens can mutate the same freight lifecycle.
5. **Concept documents help prevent misleading product framing**: the NDAO mountable notes are valuable because they state the transparency and consent requirements before implementation details harden.

---

## 10. Conclusion

This reporting period marked a significant expansion of Freight’s ambition. The product moved beyond forms and locks toward a real third-party app mountables platform, giving campaigns a way to attach verified external capability with principle-based participant checks. At the same time, raffles were reworked so creator support, ticket-funded rewards, and withdrawable escrow are modeled more explicitly both on-chain and in the frontend.

The result is a more extensible and economically coherent system. Freight can now start to act as a host for mounted applications rather than only a static campaign container, and raffle mechanics are better aligned with the distinct roles of participant entry, creator support, prize funding, and post-campaign withdrawal. There is still follow-up work to harden these systems and refine the UX, but the underlying model for the next stage of Freight is now substantially stronger.

---

**Report Prepared By**: Birdmannn  
**Date**: August 26, 2026  
**Commit Range**: `1bb7ff5..5436d3b`
