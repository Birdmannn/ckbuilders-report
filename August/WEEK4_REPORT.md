# Freight on Nervos - Weekly Development Report
**Reporting Period**: Since commit `5436d3b`  
**Project**: Freight on Nervos  
**Branch**: v3

---

## Executive Summary

This reporting period focused on turning FON's mounted-app system from a basic registration/update mechanism into a clearer SDK-driven principle platform. The work introduced a formal principle model, richer TypeScript contracts, hosted-app registration secrets, webhook and polling support, canonical participant finalization, manual mounted-app evaluation, and a runnable demo app that shows how an external app can expose FON-compatible rules.

The largest architectural change was the introduction of parameterized principles. Instead of treating a mounted app principle as only an id, title, and description, principles now carry parameter schemas, defaults, readable formats, selected configuration, and normalized evaluation results. This aligns the implementation with the product model described in `PRINCIPLES.md`: a principle is a typed, app-owned predicate selected by a freight creator and evaluated against a participant in freight context.

A new app runtime layer now coordinates mounted-app activity across the host. It can build freight and participant snapshots, finalize participant eligibility across base participation, forms, and app principles, dispatch webhook evaluation events to hosted apps, and track delivery outcomes. The host API surface was expanded accordingly with manual evaluation, poll support, stronger registration metadata, and more complete install verification.

The SDK work was then exercised with a new `examples/checkbox-demo` project. The demo provides a local login, checkbox-based participant state, three parameterized demo principles, and hosted-app endpoints for manifest, registration, install verification, activity, polling, and direct evaluation. It gives future app integrators a concrete reference for how to define principles, evaluate participant state, and expose the endpoints FON expects.

**Key Metrics**:
- 2 commits in the inspected range
- 39 files changed
- 4,328 lines added
- 69 lines removed
- 1 new SDK package:
  - `packages/fon-sdk`
- 1 new hosted-app runtime module:
  - `frontend/lib/mountableAppRuntime.ts`
- 1 new demo project:
  - `examples/checkbox-demo`
- 4 major workstreams advanced:
  - parameterized mounted-app principles
  - SDK and hosted-app API contract
  - mounted-app evaluation, delivery, and finalization runtime
  - runnable checkbox demo project

---

## 1. Mounted-App Principles Were Formalized

### 1.1 Principle Definitions Became Rich, Parameterized Contracts

**Problem Identified**: Mounted-app principles were too thin to support real external app logic. A principle could be selected by id, but there was no first-class way to describe configurable inputs, defaults, readable rule text, or the exact rule a creator selected.

**Solution Implemented**:
- Expanded `MountableAppPrincipleDefinition` to include:
  - `paramsSchema`
  - `paramsDefaults`
  - `readableFormat`
  - `exampleReadableText`
- Added parameter definitions with support for:
  - strings
  - numbers
  - booleans
  - enums
  - minimum / maximum / step metadata
  - placeholders and option descriptions
- Added `MountableAppPrincipleSelection` so mounted freights can store selected rule configs, not only selected ids
- Added resolved selected-principle state that combines the app's definition with creator-selected params and display labels

**Primary Files**:
- `frontend/lib/fonMountablesSdk.ts`
- `frontend/app/_types/appMountable.ts`
- `frontend/app/_lib/appMountable.ts`

**Impact**:
- Made principles stable by id while allowing variable behavior through params
- Created the data shape needed for future creator UIs that configure principle inputs
- Preserved readable participant/creator-facing rule labels after a principle is mounted
- Brought implementation closer to the principles architecture documented in `PRINCIPLES.md`

### 1.2 Evaluation Request and Result Types Were Added

**Problem Identified**: FON needed a standard way to ask an app whether a participant satisfies a selected rule, and apps needed a standard way to return more than a bare boolean.

**Solution Implemented**:
- Added `EvaluateMountableAppPrincipleRequest`
- Added participant context fields:
  - `participantAddress`
  - `participantHandle`
  - `externalUserId`
- Added freight context fields:
  - campaign id
  - creator hash
  - chain creation timestamp
  - campaign type
  - task timing window
  - optional `asOf`
- Added `EvaluateMountableAppPrincipleResult` as the normalized mounted principle state shape
- Expanded `MountedAppPrincipleState` to preserve:
  - title
  - description
  - params
  - display label
  - fulfillment status
  - detail text
  - update timestamp

**Primary Files**:
- `frontend/lib/fonMountablesSdk.ts`
- `packages/fon-sdk/src/index.ts`

**Impact**:
- Established a consistent pull-style evaluation contract
- Made app responses explainable with `detail` and `updatedAt`
- Preserved the configured rule alongside the evaluation result
- Prepared FON for mounted apps such as games, leaderboards, forms, and external task systems

### 1.3 Normalization and Formatting Utilities Were Expanded

**Problem Identified**: Once principles became parameterized, FON needed safer normalization for incoming manifests, selections, params, sync modes, and returned principle states.

**Solution Implemented**:
- Added normalization for:
  - sync modes
  - positive integer poll intervals
  - principle parameter schemas
  - enum options
  - selected principle configs
  - normalized mounted principle states
- Added readable selection formatting using `readableFormat` and selected params
- Updated selected-principle resolution so selections are matched against registered definitions and merged with defaults

**Primary Files**:
- `frontend/lib/fonMountablesSdk.ts`
- `frontend/app/_lib/appMountable.ts`

**Impact**:
- Reduced risk from malformed hosted-app payloads
- Kept persisted mounted-app configs stable and readable
- Allowed old `selectedPrincipleIds` style inputs to coexist with the richer `selectedPrinciples` model

---

## 2. Hosted-App SDK and API Contract Were Introduced

### 2.1 A Local SDK Package Was Added

**Problem Identified**: External or hosted apps needed a small app-side API for defining principles, evaluating them, registering with FON, verifying installs, and sending participant updates.

**Solution Implemented**:
- Added `packages/fon-sdk`
- Exported the shared FON mountable types from the SDK package
- Added `createHostedAppSdk()` with:
  - `addPrinciple`
  - `evaluatePrinciple`
  - `listPrinciples`
- Added helper functions:
  - `registerWithFon`
  - `verifyInstall`
  - `sendParticipantUpdate`
  - `buildInstallationSecretHeaders`
  - `buildSelectedPrinciple`

**Primary Files**:
- `packages/fon-sdk/package.json`
- `packages/fon-sdk/src/index.ts`

**Impact**:
- Created a concrete SDK entrypoint for hosted app authors
- Turned the principle model into something an app can implement directly
- Gave demo and future integrations a reusable surface instead of copying host internals

### 2.2 Registration Now Supports Sync Modes and Secrets

**Problem Identified**: Mounted-app registration needed to describe how the app syncs with FON, and FON needed a secret for authenticated follow-up communication.

**Solution Implemented**:
- Extended app registration to include:
  - `activityWebhookUrl`
  - `pollUpdatesUrl`
  - `syncMode`
  - `pollIntervalSeconds`
- Validated webhook URLs when webhook delivery is enabled
- Validated polling URLs when poll sync is enabled
- Generated a registration secret on first registration
- Preserved an existing registration secret unless rotation is requested
- Stored a hashed version of the registration secret
- Returned sync metadata and the issued registration secret to the registering app

**Primary Files**:
- `frontend/app/api/mountables/apps/register/route.ts`
- `frontend/app/api/mountables/apps/route.ts`
- `frontend/lib/mongodb.ts`
- `frontend/lib/mountableAppRuntime.ts`

**Impact**:
- Made mounted-app registration operational rather than purely descriptive
- Enabled secure event delivery from FON to hosted apps
- Allowed apps to choose webhook, polling, or both as their sync model
- Created a foundation for tracking app communication lifecycle and failures

### 2.3 Install Verification Now Resolves Configured Principles

**Problem Identified**: The install verification flow accepted selected principle ids, but it did not fully preserve selected params, readable labels, app-returned principle definitions, or sync metadata.

**Solution Implemented**:
- Accepted both `selectedPrinciples` and legacy `selectedPrincipleIds`
- Forwarded normalized selected-principle configs to hosted app verify endpoints
- Resolved returned principles against registered manifest principles
- Stored selected principles with params, display labels, and required flags
- Stored app sync metadata on the mounted app config after verification
- Preserved app-provided config and admin notices
- Continued hashing the installation shared secret before persistence

**Primary Files**:
- `frontend/app/api/mountables/apps/verify/route.ts`
- `frontend/app/_lib/appMountable.ts`
- `frontend/app/_types/appMountable.ts`

**Impact**:
- Mounted app installs now retain the actual rules selected by the creator
- Hosts can later evaluate selected app principles with the right params
- The mounted-app config contains enough metadata to drive webhook and polling sync

---

## 3. Mounted-App Runtime and Finalization Were Added

### 3.1 Canonical Participant Finalization Was Centralized

**Problem Identified**: Participant eligibility needed a canonical status that combines base participation, mounted forms, and mounted app principles. Without this, separate verification systems could update independently without producing one clear participant outcome.

**Solution Implemented**:
- Added `finalizeCampaignParticipant`
- Computes:
  - whether the participant has base activity
  - whether required forms verification is satisfied
  - whether all enabled mounted app principles are satisfied
  - final `pending` or `verified` canonical status
  - reasons for pending state
- Persists canonical snapshots to a dedicated finalizations collection
- Mirrors canonical verification into the campaign participants collection as `participantKind: "canonical_verification"`
- Added a public API route to finalize a participant manually

**Primary Files**:
- `frontend/lib/mountableAppRuntime.ts`
- `frontend/app/api/campaign-participants/finalize/route.ts`
- `frontend/lib/mongodb.ts`

**Impact**:
- Created one source of truth for whether a participant is fully eligible
- Made forms and app-principle results composable
- Gave operators and frontend flows a direct route to refresh canonical participant status

### 3.2 Webhook Evaluation Dispatch Was Added

**Problem Identified**: Hosted apps needed to be notified when participant state changes so they can evaluate their selected principles or react to finalization events.

**Solution Implemented**:
- Added event construction for mounted-app activity:
  - event id
  - event type
  - freight context
  - selected principles
  - participant snapshot
  - canonical verification snapshot
  - source marker
- Added authenticated webhook delivery using the registration secret
- Added delivery history persistence with:
  - delivered / failed / skipped status
  - response status
  - response payload
  - error message when fetch fails
- Skipped delivery cleanly when manifest or webhook configuration is missing

**Primary Files**:
- `frontend/lib/mountableAppRuntime.ts`
- `frontend/lib/mongodb.ts`

**Impact**:
- Enabled push-style mounted-app sync
- Made hosted-app communication auditable through delivery records
- Preserved enough context for external apps to evaluate participant state accurately

### 3.3 Forms Claims Now Trigger Finalization and App Dispatch

**Problem Identified**: Mounted forms and mounted apps were separate systems. When a form claim changed, app evaluation and canonical finalization did not automatically refresh.

**Solution Implemented**:
- Updated form claim submission to finalize the participant after claim processing
- Dispatched mounted-app requests after a form claim update
- Updated forms sync to finalize newly verified participants
- Dispatched mounted-app requests after batch form sync
- Returned canonical verification data from the claim route

**Primary Files**:
- `frontend/app/api/campaign-records/[id]/forms/claim/route.ts`
- `frontend/app/api/campaign-records/[id]/forms/sync/route.ts`
- `frontend/lib/mountableAppRuntime.ts`

**Impact**:
- Made mounted forms and mounted apps part of the same eligibility pipeline
- Reduced stale verification states after a participant submits or syncs a form claim
- Improved the path from individual verification events to canonical campaign eligibility

### 3.4 App Update Ingestion Now Recomputes Canonical Eligibility

**Problem Identified**: Hosted apps could push principle state updates, but FON needed to normalize richer selected-principle state and recompute parent/canonical status after those updates.

**Solution Implemented**:
- Normalized incoming principle states against the mounted app's selected principles
- Preserved params and display labels in stored criteria state
- Computed child satisfaction from selected principle ids and pushed states
- Recomputed parent satisfaction across all enabled mounted apps for the participant
- Stored update history
- Finalized the participant after app update ingestion
- Dispatched a follow-up mounted-app event after ingestion, using `participant.finalized` when canonical status is verified

**Primary Files**:
- `frontend/app/api/mountables/apps/[mountableInstanceId]/updates/route.ts`
- `frontend/lib/fonMountablesSdk.ts`
- `frontend/lib/mountableAppRuntime.ts`

**Impact**:
- Made pushed app updates participate in the same canonical eligibility model as forms
- Improved stored criteria state quality by preserving rule metadata
- Helped downstream app deliveries react to verified vs pending participant transitions

### 3.5 Manual Evaluation and Polling Routes Were Added

**Problem Identified**: FON needed a way to manually trigger mounted-app evaluation and to query poll-based hosted apps.

**Solution Implemented**:
- Added `POST /api/mountables/apps/evaluate`
  - loads campaign and participant
  - filters enabled mounted apps
  - optionally narrows by app id or mountable instance id
  - finalizes the participant
  - dispatches evaluation requests
- Added `GET /api/mountables/apps/evaluate`
  - loads the registered app's poll URL
  - calls the app's polling endpoint
  - returns normalized poll result envelope

**Primary Files**:
- `frontend/app/api/mountables/apps/evaluate/route.ts`

**Impact**:
- Added an operator/developer path to request app evaluation on demand
- Added first-class poll sync support alongside webhook delivery
- Made both push and pull models viable in the current host architecture

---

## 4. Principle Documentation Was Added

### 4.1 `PRINCIPLES.md` Defines the Product Model

**Problem Identified**: The mounted-app principle system needed a clear conceptual contract so future work would not drift into hardcoded ids or app-specific freight logic.

**Solution Implemented**:
- Added `PRINCIPLES.md`
- Defined a principle as an app-owned predicate
- Established the key rules:
  - stable principle ids
  - variable behavior in params
  - human-readable rule text
  - app-side evaluation
  - results richer than booleans
- Documented creator and participant flows
- Documented push vs pull evaluation models
- Included examples such as play count, rank range, leaderboard presence, and score threshold

**Primary Files**:
- `PRINCIPLES.md`

**Impact**:
- Gave the SDK and runtime work an explicit design target
- Clarified how games, leaderboards, forms, and future apps should integrate
- Reduced the risk of principle-id proliferation

### 4.2 Interface Notes Were Added

**Problem Identified**: The new TypeScript contract needed implementation-level notes that connect the conceptual principle model to the actual files and types.

**Solution Implemented**:
- Added `PRINCIPLES2.md`
- Documented:
  - principle definitions
  - parameter schema
  - selected principle configs
  - evaluation requests
  - normalized runtime states
  - hosted-app SDK interface
  - verification/install flow changes

**Primary Files**:
- `PRINCIPLES2.md`

**Impact**:
- Created a technical bridge between design principles and implementation
- Made it easier to review or extend the TypeScript API without reverse-engineering the diff

---

## 5. Checkbox Demo Project Was Added

### 5.1 A Runnable Hosted-App Demo Was Created

**Problem Identified**: The SDK and principle architecture needed a concrete demo that shows how a hosted app defines principles and evaluates participant state.

**Solution Implemented**:
- Added `examples/checkbox-demo`
- Built a small Next.js app with:
  - local demo login
  - checkbox-based participant activity
  - live principle evaluation output
  - readable fulfillment details
  - responsive UI styling
- Added three parameterized demo principles:
  - `demo-checkbox-completed`
  - `demo-checkbox-count-at-least`
  - `demo-checkbox-set-completed`
- Stored demo participant state in memory for local development

**Primary Files**:
- `examples/checkbox-demo/app/page.tsx`
- `examples/checkbox-demo/app/styles.css`
- `examples/checkbox-demo/lib/checkboxDemoApp.ts`

**Impact**:
- Gave SDK users a working reference app
- Demonstrated parameterized principles with simple checkbox behavior
- Made the mounted-app concept easier to understand by turning abstract predicates into visible UI state

### 5.2 Demo Hosted-App Endpoints Were Implemented

**Problem Identified**: A useful SDK demo needed to expose the same HTTP surface FON expects from a hosted app.

**Solution Implemented**:
- Added demo endpoints:
  - `GET /api/manifest`
  - `POST /api/register`
  - `POST /api/verify-install`
  - `POST /api/activity`
  - `GET /api/poll`
  - `POST /api/evaluate`
  - `GET/POST /api/participant-state`
- Added `.env.example`
- Added README instructions for running the demo and registering it with a local FON frontend
- Added a local SDK shim so the demo can consume the sibling SDK source during local Next builds

**Primary Files**:
- `examples/checkbox-demo/app/api/manifest/route.ts`
- `examples/checkbox-demo/app/api/register/route.ts`
- `examples/checkbox-demo/app/api/verify-install/route.ts`
- `examples/checkbox-demo/app/api/activity/route.ts`
- `examples/checkbox-demo/app/api/poll/route.ts`
- `examples/checkbox-demo/app/api/evaluate/route.ts`
- `examples/checkbox-demo/app/api/participant-state/route.ts`
- `examples/checkbox-demo/lib/fonSdk.ts`
- `examples/checkbox-demo/README.md`

**Impact**:
- Demonstrated the full hosted-app lifecycle, not just frontend checkbox state
- Gave FON a local app target for registration, install verification, activity delivery, polling, and evaluation
- Created a practical integration fixture for future SDK work

### 5.3 Demo Build and Generated Artifacts Were Managed

**Problem Identified**: Running and building the demo produced generated artifacts that should not be committed.

**Solution Implemented**:
- Added demo-specific ignore rules for:
  - `.next`
  - `node_modules`
  - `tsconfig.tsbuildinfo`
- Kept source, config, docs, and package lockfile commit-ready
- Verified that removing `.next` does not affect the ability to rerun the demo because Next regenerates it

**Primary Files**:
- `.gitignore`
- `examples/checkbox-demo/package.json`
- `examples/checkbox-demo/package-lock.json`
- `examples/checkbox-demo/next.config.ts`
- `examples/checkbox-demo/tsconfig.json`

**Impact**:
- Kept the git tree focused on source artifacts
- Preserved reproducible dependency installation through the lockfile
- Avoided committing generated Next.js build output or installed packages

---

## 6. Challenges and Solutions

### Challenge 1: Principles needed to support future app logic without hardcoding app-specific behavior into FON
**Solution**: Introduced stable principle ids with parameter schemas, defaults, selected configs, readable labels, and normalized evaluation results.

### Challenge 2: Push and pull sync models both needed a place in the host architecture
**Solution**: Added webhook delivery, poll URL registration, poll route support, manual evaluation dispatch, and sync-mode validation.

### Challenge 3: Forms, app principles, and base participation needed one final eligibility answer
**Solution**: Added canonical participant finalization that evaluates forms satisfaction, mounted-app principle satisfaction, and base activity together.

### Challenge 4: Hosted-app communication needed to be auditable
**Solution**: Added mounted-app delivery records that capture skipped, delivered, and failed webhook dispatches with response payloads or error messages.

### Challenge 5: The SDK needed a practical example, not only internal types
**Solution**: Added the checkbox demo project with real app-side principles, API endpoints, local participant state, and a UI that demonstrates fulfillment changes.

---

## 7. Impact Assessment

### Quantitative Impact
- **Commit Volume**: 2 commits in the inspected range
- **Code Change Volume**: 4,328 lines added / 69 removed
- **Files Touched**: 39 files in the inspected range
- **Largest New Implementation Areas**:
  - `examples/checkbox-demo`, with 20 new files
  - `frontend/lib/mountableAppRuntime.ts`, with 474 new lines
  - `frontend/lib/fonMountablesSdk.ts`, with a large expansion of shared types and normalizers
  - `packages/fon-sdk`, with the first local SDK package entrypoint

### Qualitative Impact
- **SDK Readiness**: FON now has a concrete SDK surface for hosted app authors
- **Principle Model Maturity**: Mounted-app rules now support typed params, readable labels, defaults, and richer evaluation results
- **Runtime Coherence**: Forms, app updates, and base participation now feed into a canonical participant verification model
- **Sync Flexibility**: Hosted apps can be modeled through webhook, polling, or both
- **Developer Experience**: The checkbox demo makes the integration path visible and runnable
- **Operational Visibility**: Webhook delivery attempts are persisted, making hosted-app communication easier to inspect

---

## 8. Next Week's Priorities

### Immediate Focus
1. Harden the SDK package boundary so demo apps can import the package directly without local shims
2. Add tests around principle normalization, selected-principle resolution, and mounted-app update ingestion
3. Exercise the webhook and poll flows against a local FON frontend with real mounted campaigns
4. Refine the creator UI for configuring principle params instead of only selecting principles

### Strategic Initiatives
1. Move the principle system toward a stable external integration contract
2. Expand the demo pattern into more realistic apps, such as leaderboards or scored games
3. Continue unifying participant verification across forms, apps, and campaign-native participation
4. Add better observability for app deliveries, poll runs, and finalization transitions

### Documentation
1. Document the SDK package usage from the hosted app author's perspective
2. Add endpoint examples for register, verify install, update push, poll, and direct evaluation
3. Clarify how selected principle params should be rendered and edited in the creator flow

---

## 9. Lessons Learned

1. **Principle ids should stay boring**: the variable behavior belongs in params, not in a growing list of special-case ids.
2. **A good SDK contract needs both types and lifecycle helpers**: app authors need to define principles, register apps, verify installs, and send updates through one understandable surface.
3. **Mounted verification needs a canonical answer**: forms and app-principle checks are more useful when they roll up into one participant eligibility snapshot.
4. **Sync systems need audit trails early**: delivery records make webhook failures and skipped sync attempts easier to reason about.
5. **A demo reveals integration friction quickly**: building the checkbox app exposed package-boundary and generated-artifact concerns that pure type design would have missed.

---

## 10. Conclusion

This reporting period moved FON's mounted-app system into a more durable SDK-oriented architecture. Principles are now richer, parameterized, and explainable; hosted apps can register sync metadata and secrets; FON can finalize participant eligibility across forms and app principles; and app communication can happen through webhook dispatch, polling, or manual evaluation.

The new checkbox demo turns that architecture into a working integration example. It shows how a hosted app can define app-owned predicates, let FON mount selected configurations, evaluate participant state, and return normalized results with human-readable details. The next step is to harden the package boundary, add tests, and evolve the creator configuration UI so these new principle params can be managed directly by freight creators.

---

**Report Prepared By**: Birdmannn  
**Date**: September 1, 2026  
**Commit Range**: `5436d3b..HEAD`
