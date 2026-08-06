# FreightOnNervos - Weekly Development Report
**Reporting Period**: Profile tab system, transaction/feed UX work, and FBARS/JoyID seed-flow debugging  
**Project**: FreightOnNervos Frontend and Profile/Reward Experience  
**Branch**: v3

---

## Executive Summary

This reporting period focused on turning the profile page from a mostly static identity + graph surface into a much more functional user-history area, while also uncovering and resolving multiple issues in the FBARS and wallet-seeding experience.

The work ultimately fell into four tightly connected themes:

1. **Profile information architecture improvements** — adding a tabbed profile experience with Activity, Freights, and Transactions.
2. **Transaction and freight-history UX refinement** — building compact list views, refresh/caching behavior, and clearer transaction formatting.
3. **Raffle randomness UX hardening** — removing visible randomness preimage exposure from user-facing success/details surfaces while preserving settlement behavior.
4. **FBARS and JoyID debugging** — tracing failures in the wallet-seed path, understanding why initial FBARS were not appearing, and fixing the JoyID verification path so the seed flow can complete.

The profile work delivered meaningful product improvements, but the most important technical lesson of this period was about **reward/identity correctness**:

> A wallet-signing flow is only useful if the app can prove exactly what was signed, by whom, for which chain, and how that result is persisted.

That principle shaped the debugging work around wallet seeding, JoyID verification, and FBARS persistence.

---

## 1. Profile Page Evolution: From Static Summary to Tabbed History Surface

### 1.1 Introduction of Activity / Freights / Transactions Tabs

**Problem Identified**: The profile page was structurally too narrow. It mainly showed identity information plus an activity graph, but it did not expose a user’s freight participation history or transaction history in a first-class way.

**Solution Implemented**:
- Added a tabbed profile shell for:
  - `Activity`
  - `Freights`
  - `Transactions`
- Kept `Activity` as the default tab
- Preserved the existing graph behavior while making space for deeper profile history surfaces

**Primary Files**:
- `frontend/app/_components/ProfileScreen.tsx`
- `frontend/app/styles/profile.css`
- `frontend/app/_types/profileTabs.ts`

**Impact**:
- The profile now behaves like a real dashboard instead of a single long static section
- Information is segmented into clearer modes of use:
  - visual trend history
  - freight interaction history
  - transaction/reward history

### 1.2 Activity Tab Preserved as the Existing Graph Surface

**Problem Identified**: The profile’s graph was already useful, but it had to remain stable while the page architecture changed.

**Solution Implemented**:
- Kept the existing `ProfileAnalyticsSection` as the Activity tab body
- Preserved the existing analytics contract and SVG chart rendering
- Continued using profile-scoped analytics data from the existing hook/API pattern

**Primary Files**:
- `frontend/app/_components/ProfileAnalyticsSection.tsx`
- `frontend/app/_hooks/useProfileAnalytics.ts`
- `frontend/app/api/user-profiles/analytics/route.ts`

**Impact**:
- Activity remained familiar to the user
- The graph did not need to be rethought or rebuilt to support the broader profile redesign

---

## 2. Freights Tab: A Compact Freight Interaction History

### 2.1 First Freight Aggregation Layer

**Problem Identified**: There was no profile-specific way to answer a simple product question:

> “Which freights has this user actually engaged with?”

That logic existed in fragments across campaign records, participant rows, reward recipients, and comments, but not as a unified profile read model.

**Solution Implemented**:
- Added a dedicated profile freights endpoint and hook
- Aggregated freight interactions from four sources:
  - created freights
  - participated freights
  - commented freights
  - rewarded freights
- Produced one freight row per freight with:
  - latest interaction date
  - strongest interaction classification
  - stable campaign navigation target

**Primary Files**:
- `frontend/app/api/user-profiles/freights/route.ts`
- `frontend/app/_hooks/useProfileFreights.ts`
- `frontend/app/_types/profileTabs.ts`
- `frontend/lib/campaignIdentity.ts`

**Impact**:
- The profile gained a usable freight history layer built from existing persisted data
- The app can now present a concise archive of relevant freights without loading the global feed

### 2.2 Compact Freight Rows and Interaction Styling

**Problem Identified**: A full card layout was too visually heavy for the profile context. The user asked for a denser list treatment.

**Solution Implemented**:
- Changed the Freights tab rows into a one-line listing
- Included:
  - compact date at the left
  - colored interaction dot
  - truncated freight title
  - meta information on the right
- Preserved special rewarded styling with purple treatment
- Removed card-style borders and later added subtle dividers between rows for list readability

**Primary Files**:
- `frontend/app/_components/ProfileFreightsSection.tsx`
- `frontend/app/styles/profile.css`

**Impact**:
- Freights now read as a compact profile list rather than a feed clone
- The history is easier to scan quickly
- Rewarded items still stand out without occupying too much visual space

---

## 3. Transactions Tab: Public On-chain / Off-chain History

### 3.1 New Profile Transactions Read Model

**Problem Identified**: The app had transaction-like data spread across:
- published campaign records
- participant records
- settlement records
- deposit records
- FBARS event logs

But there was no unified profile transaction history.

**Solution Implemented**:
- Added a transactions endpoint that merges multiple sources into one latest-first timeline
- Included both:
  - **on-chain** rows (create, activate, participate, deposit, settlement, reward receipt)
  - **off-chain** FBARS rows
- Added transaction coverage notes so legacy data gaps are surfaced honestly

**Primary Files**:
- `frontend/app/api/user-profiles/transactions/route.ts`
- `frontend/app/_hooks/useProfileTransactions.ts`
- `frontend/app/_types/profileTabs.ts`
- `frontend/lib/txBalanceDelta.ts`
- `frontend/lib/ckbClient.ts`

**Impact**:
- Profiles now have a public financial/event history surface
- The app can distinguish between recorded chain-affecting actions and off-chain FBARS activity
- Historical incompleteness is acknowledged instead of hidden

### 3.2 Transaction Formatting Improvements

**Problem Identified**: The first transaction design was too verbose and did not align with the desired list feel.

**Solution Implemented**:
- Reworked transactions into compact rows with:
  - day on the far left
  - linked tx hash in the center
  - amount on the far right
- Shortened tx-hash display substantially
- Switched tx hash styling to monospace
- Kept amounts in Cruyff Sans
- Applied requested amount colors:
  - green for positive
  - grey for zero
  - red for negative
- Rendered on-chain values as **USD** strings rather than `$`-only shorthand
- Treated off-chain transaction rows as FBARS rows

**Primary Files**:
- `frontend/app/_components/ProfileTransactionsSection.tsx`
- `frontend/app/styles/profile.css`
- `frontend/app/api/user-profiles/transactions/route.ts`

**Impact**:
- Transactions are now much easier to skim
- The visual structure matches the intended profile-history feel better than the earlier expanded-card approach

### 3.3 Manual Refresh and Caching Behavior

**Problem Identified**: The profile tabs were refetching too eagerly, creating poor UX and unnecessary loading states.

**Solution Implemented**:
- Added simple in-memory per-query caching for Freights and Transactions
- Changed both tabs to fetch/update only when the user explicitly clicks a refresh button
- Added a minimal borderless refresh control with underline-on-hover behavior
- Introduced “click refresh to load” empty-state messaging when no cached data exists yet

**Primary Files**:
- `frontend/app/_hooks/useProfileFreights.ts`
- `frontend/app/_hooks/useProfileTransactions.ts`
- `frontend/app/_components/ProfileFreightsSection.tsx`
- `frontend/app/_components/ProfileTransactionsSection.tsx`
- `frontend/app/_components/ProfileScreen.tsx`
- `frontend/app/styles/profile.css`

**Impact**:
- Reduced gratuitous loading behavior
- Made the history tabs feel more deliberate and stable
- Improved perceived responsiveness once data has been cached

---

## 4. Raffle Randomness UX: Hiding the Preimage Without Breaking Settlement

### 4.1 Removing Visible Preimage Exposure

**Problem Identified**: The raw randomness preimage was being shown in places where it did not belong from a UX or trust perspective, such as success/info surfaces and raffle detail surfaces.

**Solution Implemented**:
- Removed user-facing preimage display from:
  - create/submission success info modal
  - raffle freight detail displays
- Kept the underlying preimage storage and settlement logic intact so raffle settlement continues to work

**Primary Files**:
- `frontend/app/page.tsx`
- `frontend/app/_components/ProfileScreen.tsx`
- `frontend/app/campaign/[campaignId]/page.tsx`
- `frontend/app/_hooks/useCampaignCardState.ts`
- `frontend/app/_hooks/useCreateCampaignFlow.ts`

**Impact**:
- Reduced unnecessary exposure of internal randomness material
- Preserved deterministic settlement behavior while improving user-facing discretion

### 4.2 Settlement Still Uses Stored Randomness

**What was preserved**:
- Raffle preimages are still generated, stored, and used for settlement/distribution logic
- Share/distribution flows still rely on stored preimages where needed
- The UX change was about **visibility**, not about deleting the underlying settlement mechanism

**Impact**:
- Better UX without sacrificing functional correctness for raffle resolution

---

## 5. FBARS Work: Persistence, Costs, and Event History

### 5.1 FBARS Persistence Model Clarified

**Problem Identified**: It was not immediately obvious where FBARS were persisted, leading to confusion when inspecting campaign-related data.

**What We Confirmed**:
- FBARS are persisted in:
  - `userProfiles.fbars`
  - `userProfiles.weeklyFbarsState`
- Event history is persisted in:
  - `fbarEvents`

**Primary Files**:
- `frontend/lib/fbars.ts`
- `frontend/lib/mongodb.ts`
- `frontend/app/api/user-profiles/route.ts`
- `frontend/app/api/fbars/freight-create/route.ts`
- `frontend/app/api/fbars/deposit/route.ts`
- `frontend/app/api/fbars/interaction/route.ts`

**Impact**:
- Clarified that FBARS are profile/accounting state, not freight-record state
- Made it easier to reason about why a user might not see balance changes where they expected them

### 5.2 Freight Creation FBARS Cost

**Problem Identified**: Freight creation had to enforce the FBARS gating rule but that rule was easy to misread if looking only at isolated snippets.

**Solution / Confirmation**:
- Confirmed that freight creation checks the current `userProfiles.fbars`
- If `fbars < 20`, creation is rejected
- If creation proceeds, a `freight-create` FBARS event with negative delta is applied

**Primary Files**:
- `frontend/app/api/fbars/freight-create/route.ts`
- `frontend/lib/fbars.ts`

**Impact**:
- The cost model is explicit and persisted through the FBARS ledger
- Helped separate “can I create a freight?” from “where are FBARS actually stored?”

---

## 6. Wallet Seed FBARS: The Main Debugging Story of the Period

This was the most important debugging thread of the reporting period.

### 6.1 Initial Problem: No Initial FBARS Showing Up

**Problem Reported**:
Even after wallet connection and signing, initial FBARS did not appear as expected, despite a large connected-wallet balance.

### 6.2 First Discovery: Auto-Seeding Prompt Was Unprovoked

**Problem Identified**:
The initial wallet-seed flow was auto-triggering too aggressively from profile-loading logic, causing unprovoked signing prompts on app launch.

**Solution Implemented**:
- removed the passive auto-seed behavior
- began moving toward a flow tied to intentional wallet connection instead of passive page load

**Primary Files**:
- `frontend/app/_hooks/useUserProfile.ts`

**Impact**:
- stopped surprising CCC signing prompts on initial page load
- exposed the need for a better “intentional connect” seed trigger

### 6.3 Second Discovery: The Auto-Seed Trigger Could Loop

**Problem Identified**:
The first correction still allowed repeated signing loops after connect because the seed intent could remain armed and re-trigger across multiple mounted profile hooks.

**Solution Implemented**:
- added wallet-seed intent tracking and in-flight guarding
- tied automatic seeding to intentional wallet-connect entry points instead of passive mount behavior
- attempted to ensure one automatic seed attempt per intended connect flow

**Primary Files**:
- `frontend/lib/walletSeed.ts`
- `frontend/app/_hooks/useUserProfile.ts`
- `frontend/app/page.tsx`
- `frontend/app/_components/ProfileScreen.tsx`
- `frontend/app/campaign/[campaignId]/page.tsx`

**Impact**:
- reduced looping prompts
- made the seed attempt lifecycle much clearer

### 6.4 Third Discovery: JoyID Identity Parsing Was Wrong

**Problem Identified**:
Wallet seed requests were failing with identity/public-key parsing errors because JoyID identities are not raw secp256k1 compressed public keys.

Observed failures included:
- invalid byte parsing
- 33-byte public-key assumptions failing

**Solution Implemented**:
- parsed JoyID `identity` as structured JSON
- extracted `keyType` and `publicKey`
- stopped routing JoyID identities through plain `SignerCkbPublicKey` logic for address derivation

**Primary Files**:
- `frontend/lib/googleAuth.ts`
- `frontend/node_modules/@ckb-ccc/joy-id/src/ckb/index.ts` (investigated)
- `frontend/node_modules/@ckb-ccc/core/src/signer/ckb/signerCkbPublicKey.ts` (investigated)

**Impact**:
- fixed the first class of JoyID-specific verification failures
- moved the seed flow closer to the correct credential semantics

### 6.5 Fourth Discovery: JoyID Credential Verification Was Missing an Absolute Server URL

**Problem Identified**:
Even after JoyID identity parsing improved, verification still failed because `verifyCredential(...)` required an absolute JoyID server URL and was falling back to an undefined/relative URL in server context.

Observed failure:
- `TypeError: Only absolute URLs are supported`

**Solution Implemented**:
- passed an explicit JoyID API base URL into the credential verification path
- used network-aware selection:
  - mainnet → `https://api.joy.id/api/v1`
  - testnet → `https://api.testnet.joyid.dev/api/v1`

**Primary Files**:
- `frontend/lib/googleAuth.ts`
- `frontend/lib/ckbClient.ts`

**Impact**:
- fixed the server-side JoyID credential lookup path
- removed the last known infrastructure blocker preventing the seed route from completing

### 6.6 Diagnostic Logging Added

**Problem Identified**:
We needed to know exactly where the wallet-seed flow was breaking: nonce creation, signing, address derivation, JoyID verification, award application, or persistence.

**Solution Implemented**:
Added temporary logging around:
- client nonce response
- client signMessage payload
- seed response status/payload
- server-side wallet-seed route state
- JoyID derived/expected address checks
- award result and persisted profile state

**Primary Files**:
- `frontend/app/_hooks/useUserProfile.ts`
- `frontend/lib/googleAuth.ts`
- `frontend/app/api/user-profiles/seed/route.ts`

**Impact**:
- Made the debugging path observable end-to-end
- Allowed the team to move from guesswork to evidence-driven fixes

### 6.7 Final Confirmed Result

The latest run showed the wallet seed route returning:
- `ok: true`
- `alreadySeeded: false`
- `awardedFbars: 93`
- the correct wallet balance in shannons

This indicates that the JoyID-related verification blockers were successfully removed and the initial FBARS award path can now complete for the tested case.

**Important caveat**:
Although the latest successful response strongly indicates the seed flow now works, the broader persistence/display path should still be validated in the live UI and database state for a full end-to-end confirmation.

---

## 7. Challenges Encountered

### Challenge 1: Profile data existed in fragments but not as profile-level history
**Solution**: Added dedicated profile freights and transactions APIs and client hooks to aggregate existing campaign/participant/reward/accounting data into profile-specific views.

### Challenge 2: Verbose history views created poor UX
**Solution**: Iteratively compressed Freights and Transactions into one-line list treatments with compact dates, truncated identifiers, and clearer amount semantics.

### Challenge 3: Overexposing raffle randomness in user-facing surfaces
**Solution**: Hid randomness preimages from info/detail UX while preserving storage and settlement behavior internally.

### Challenge 4: Wallet-seed FBARS appeared missing
**Solution**: Investigated the full signing and persistence chain, added logs, corrected auto-trigger semantics, fixed JoyID identity parsing, and then fixed JoyID credential verification URL/configuration.

### Challenge 5: Distinguishing on-chain vs off-chain transaction semantics
**Solution**: Built a public transactions read model that separates chain-affecting events from FBARS events and formats them according to user-facing product expectations.

---

## 8. Impact Assessment

### Structural Impact
- Added profile-specific freights and transactions APIs
- Added profile-tab client hooks and UI components
- Added shared profile-tab types
- Added campaign deposit persistence for transaction history
- Extended campaign-record shape to preserve activation/settlement metadata
- Hardened JoyID wallet-seed verification and diagnostics

### Qualitative Impact
- **Profile usability improved**: the page now supports actual history inspection
- **Reward/accounting transparency improved**: FBARS and transaction paths are more explicit in both backend and UI
- **Debuggability improved**: the wallet-seed path is now observable and much better understood
- **Raffle UX improved**: sensitive randomness data is less exposed without breaking settlement logic

---

## 9. Remaining / Unresolved Items

1. **Wallet-seed persistence should still be verified against the actual database state**, not just the HTTP success response.
2. **Transactions currently format on-chain values in USD and off-chain values in FBARS**, but additional product polish may still be desirable once more live examples are available.
3. **The transactions feed still does not surface every possible FBARS event type** (for example, wallet seed visibility in the final profile transaction UI should be double-checked against current route output).
4. **Legacy data gaps remain** for older activation or reward records that predate the newer metadata fields.
5. **The current code still contains temporary diagnostic logging** around wallet-seed verification, which should be removed or downgraded once confidence is restored.

---

## 10. Next Priorities

### Immediate Focus
1. Confirm live DB persistence for:
   - `userProfiles.fbars`
   - `weeklyFbarsState`
   - `walletFbarsSeededAt`
   - `walletFbarsSeedBalanceShannons`
   - `fbarEvents.kind = wallet-seed`
2. Clean up or reduce temporary wallet-seed diagnostic logging after confirming stability
3. Continue refining the Freights and Transactions tab polish from real usage feedback

### Strategic Follow-Up
1. Revisit whether wallet seed should remain a one-time balance-derived grant or use a different formula/product rule
2. Expand transaction history to surface all desired FBARS and reward events consistently
3. Decide whether the profile tabs should eventually support deeper drill-down states or filtering

---

## 11. Lessons Learned

1. **A successful signature prompt is not the same thing as a successful seeded award**: the system must verify what happened after signing, not just that signing occurred.
2. **Wallet provider identity formats matter**: JoyID identity payloads cannot be treated like standard secp256k1 public keys.
3. **Good UX requires explicit loading control**: automatic refetching created unnecessary churn in the profile tabs, while manual refresh plus caching produced a much better experience.
4. **Visibility changes can improve trust**: hiding raw randomness preimages from user-facing UX reduced noise and overexposure without breaking settlement.
5. **Profile history is a product surface, not just a debug surface**: compact lists, color semantics, transaction grouping, and refresh behavior matter as much as the backend aggregation itself.

---

## 12. Conclusion

This reporting period made meaningful progress in two areas at once: user-facing profile UX and backend/accounting correctness.

On the UX side, the profile page now supports real navigation between Activity, Freights, and Transactions, with lighter list treatments, caching, refresh controls, and clearer transaction formatting. On the backend side, the work clarified how FBARS are persisted, how freight/accounting events should be modeled, and why the initial wallet-seed flow was failing for JoyID-backed wallets.

The biggest technical success of the period was not just adding new tabs — it was turning a vague “FBARS didn’t show up” complaint into a fully traced execution path with concrete fixes for:
- signing-trigger timing
- repeat-attempt behavior
- JoyID identity parsing
- JoyID credential verification URL/configuration
- and reward persistence observability

That combination of product-facing polish and systems-level debugging creates a much stronger base for the next phase of FreightOnNervos profile and rewards work.

---

**Report Prepared By**: Birdmannn  
**Date**: August 6, 2026  
**Baseline Context**: Current v3 frontend profile/history and FBARS/JoyID integration work
