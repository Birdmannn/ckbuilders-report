# Vert - Weekly Development Report
**Reporting Period**: Since commit `3cab2d6`  
**Project**: Vert - Payment Readiness and Execution Planning SDK for Fiber  
**Branch**: main

---

## Executive Summary

This reporting period focused on turning Vert into a much more complete SDK product rather than only a demo-backed evaluation engine. The biggest areas of progress were: (1) introducing a higher-level `Vert` SDK facade for app developers, (2) expanding the core engine from route-readiness evaluation into explicit route execution planning, (3) deepening the live Fiber adapter so it can discover and rank routes from observed topology data, (4) adding a React integration package that mounts the SDK-owned modal in browser apps, and (5) reworking the judge-facing demo to behave more like a real reference integration of the SDK.

The codebase also became much more explicit about the distinction between simply asking whether a payment can pass and determining how it should pass. Instead of stopping at primary/fallback summaries, the core model now carries route details, per-hop capacity information, ranked execution routes, and split-payment planning outputs. That change is reflected consistently across `@vert/core`, `@vert/adapters`, `@vert/ui`, the new `@vert/react` package, the hosted API, and `apps/demo2`.

At the UI level, the preview experience was refined repeatedly to reduce noise while surfacing more meaningful route information. The SDK modal now presents efficiency and pass-possibility gauges, expandable route detail, and live payment lifecycle feedback. At the packaging level, the repository now includes a dedicated React bridge package that makes the Vert modal easier to embed directly into browser-hosted applications.

**Key Metrics**:
- 14 commits since `3cab2d6`
- 31 primary files changed
- 2,035 lines added
- 484 lines removed
- 1 new package added: `@vert/react`
- 4 major workstreams advanced in parallel: SDK productization, core route-planning evolution, live Fiber route discovery, and judge-facing integration/UI simplification

---

## 1. SDK Productization and Packaging

### 1.1 High-Level `Vert` SDK Facade

**Problem Identified**: Vert had core logic, adapters, and UI pieces, but it still lacked a clear high-level integration surface that a wallet or app developer could adopt as the main product API.

**Solution Implemented**:
- Added a new `Vert` class that acts as the primary SDK facade
- Centralized evaluation, modal launch, payment execution, invoice generation, and payment-status polling behind one developer-facing API
- Added sender-authority handling so app integrations can configure pubkeys up front and reuse them across Vertify and execution flows
- Added execution helpers such as threshold-gated execution, analysis reuse, and route-selection-aware continuation behavior

**Primary Files**:
- `packages/adapters/src/vert.ts`
- `packages/adapters/src/types.ts`
- `packages/adapters/src/index.ts`

**Impact**:
- Gave Vert a much stronger identity as an SDK rather than just a set of lower-level helpers
- Reduced integration complexity for host apps
- Created a more durable public API surface for future development

### 1.2 New React Bridge Package

**Problem Identified**: The repository had shared UI, but there was no clean browser-facing bridge that could mount the SDK-owned modal directly into React apps without app-specific modal plumbing.

**Solution Implemented**:
- Added a new `@vert/react` package
- Introduced a browser/React modal presenter that mounts `VertifyPreviewModal` into a document-attached container
- Wired the presenter to execution and payment-status polling controls so the SDK modal can manage its own review-and-continue runtime behavior

**Primary Files**:
- `packages/react/package.json`
- `packages/react/src/createVertModalPresenter.tsx`
- `packages/react/src/index.ts`
- `packages/react/tsconfig.json`

**Impact**:
- Created a reusable browser integration path for the SDK
- Strengthened the separation between product surface and demo app code
- Made Vert easier to embed into third-party React applications

### 1.3 Monorepo Build and Workspace Integration Updates

**Problem Identified**: Once Vert gained a new app-facing package, the workspace and demo integration path needed to reflect the expanded product surface.

**Solution Implemented**:
- Added `@vert/react` to the repository build path
- Added the new package as a dependency of `apps/demo2`
- Added a Vite alias so local demo development can resolve the new package source directly during development

**Primary Files**:
- `package.json`
- `apps/demo2/package.json`
- `apps/demo2/vite.config.ts`
- `apps/demo2/tsconfig.json`

**Impact**:
- Kept the monorepo aligned with the new packaging architecture
- Reduced friction when iterating on the React bridge locally
- Reinforced Vert’s evolution from internal prototype to distributable SDK surface

---

## 2. Core Engine Evolution: From Readiness to Execution Planning

### 2.1 Richer Route and Execution Types

**Problem Identified**: The original planning model was not expressive enough to carry the route-level and hop-level information needed for real execution planning.

**Solution Implemented**:
- Expanded core route modeling to include:
  - sender/receiver node identity
  - full node paths
  - per-hop channel details
  - effective route capacity
  - execution-specific route identifiers
- Added execution-allocation and route-execution-plan types
- Added richer result metadata so downstream layers can carry route details cleanly

**Primary Files**:
- `packages/core/src/types.ts`
- `packages/core/src/index.ts`

**Impact**:
- Established the canonical shared data model for route-aware planning
- Made route information portable across core, adapter, API, and UI layers
- Created a much stronger foundation for future multi-path execution work

### 2.2 Route Ranking and Allocation Planning

**Problem Identified**: Vert needed a deterministic way to choose between feasible routes and to reason about whether a payment should use one route or be split across several.

**Solution Implemented**:
- Added utilities to rank routes by feasibility, path length, capacity, and fee viability
- Added a planner that allocates the requested amount across ranked routes until the payment is either fully covered, partially covered, or blocked
- Added explicit route-execution-plan statuses for `planned`, `limited`, and `blocked`
- Added summary generation for single-route and split-route strategies

**Primary Files**:
- `packages/core/src/utils/routing.ts`
- `packages/core/src/engine/plan.ts`

**Impact**:
- Moved Vert beyond basic route summaries and into explicit execution planning
- Made the split-versus-single route decision a first-class part of the core engine
- Improved the SDK’s ability to explain not just whether a payment is viable, but how it should be attempted

### 2.3 Readiness Rules Aligned with Real Capacity Signals

**Problem Identified**: High-level readiness reasoning needed to better reflect actual visible capacity rather than relying mostly on coarse route status categories.

**Solution Implemented**:
- Updated rule evaluation to compare requested payment amounts against real visible route capacity in shannons
- Added reasoning for the distinction between:
  - insufficient single-route capacity
  - sufficient aggregate capacity requiring split execution
  - tight but still potentially viable capacity
- Improved evidence generation so results can explain requested amount, best-route capacity, and aggregate capacity more concretely

**Primary Files**:
- `packages/core/src/engine/rules.ts`
- `packages/core/src/engine/canPay.ts`
- `packages/core/src/utils/efficiency.ts`

**Impact**:
- Increased the truthfulness of readiness explanations
- Better aligned reasoning with the actual route data now being collected from live topology
- Clarified the product direction toward collective routability, not only single-path success

---

## 3. Live Fiber Route Discovery and Hosted Runtime Evolution

### 3.1 Topology-Aware Live Route Discovery

**Problem Identified**: The live adapter needed to reason about actual route candidates rather than only exposing a simplified live/mock result with minimal topology intelligence.

**Solution Implemented**:
- Expanded the live Fiber adapter to inspect multiple node snapshots in the demo topology
- Built directed edges from ready channels using observed outbound capacity
- Added adjacency construction and path discovery across the visible network
- Added route description output that computes effective capacity, total fee, and per-hop details for each discovered route

**Primary Files**:
- `packages/adapters/src/live/fiberAdapter.ts`
- `packages/core/src/types.ts`
- `packages/core/src/utils/routing.ts`

**Impact**:
- Made live Vertify results meaningfully topology-aware
- Increased realism in route ranking and route explanation
- Positioned Vert for richer execution-planning behavior over real network observations

### 3.2 Invoice Decoding and Payment Lifecycle Normalization

**Problem Identified**: Invoice-based flows and payment execution needed a more structured bridge from Fiber RPC behavior into app- and SDK-friendly state transitions.

**Solution Implemented**:
- Added invoice decoding support to infer receiver pubkey and amount from parsed Fiber invoices
- Added normalized payment-phase handling for created, in-flight, succeeded, failed, and unknown states
- Preserved the distinction between live and mock execution outcomes
- Added clearer execution result shaping so the SDK can explain when execution was submitted, skipped, or currently unsupported

**Primary Files**:
- `packages/adapters/src/live/fiberAdapter.ts`
- `packages/adapters/src/live/fiberRpc.ts`
- `packages/adapters/src/types.ts`
- `packages/adapters/src/vert.ts`

**Impact**:
- Improved truthfulness of invoice-first and execution flows
- Made the payment lifecycle easier to present inside the SDK modal
- Reduced ambiguity between planned execution and validated live execution

### 3.3 Hosted API Contract Refinement

**Problem Identified**: The hosted API needed to expose the richer route-aware live payload and to accept the new sender/receiver/invoice evaluation contract used by the SDK.

**Solution Implemented**:
- Updated `/api/evaluate` to require a sender pubkey plus either a receiver pubkey or invoice
- Normalized request payloads before evaluation
- Returned route details alongside the evaluation payload so downstream SDK/UI layers can render richer live topology information
- Kept the API aligned with the hosted-runtime architecture already established in the repo

**Primary Files**:
- `apps/api/src/routes/evaluate.ts`
- `apps/api/src/routes/payments.ts`
- `packages/adapters/src/live/vertApiAdapter.ts`

**Impact**:
- Improved the backend contract for SDK-driven evaluation
- Kept browser integrations cleaner by placing route-aware evaluation behind the hosted API
- Preserved architectural consistency between the public app and live Fiber runtime support

---

## 4. Demo2 as a Reference SDK Integration

### 4.1 Demo App Rebuilt Around the `Vert` SDK

**Problem Identified**: The judge-facing demo still owned too much payment and modal behavior directly instead of acting like a realistic consumer of the Vert SDK.

**Solution Implemented**:
- Refactored `apps/demo2` to instantiate `Vert` directly
- Wired the app to the new React modal presenter instead of handling all preview runtime behavior inline
- Shifted Vertify initiation into `vert.vertifyOnModal(...)`
- Preserved wallet context and request construction while moving more runtime behavior into the SDK layer

**Primary Files**:
- `apps/demo2/src/App.tsx`
- `packages/adapters/src/vert.ts`
- `packages/react/src/createVertModalPresenter.tsx`

**Impact**:
- Made `demo2` more credible as a reference integration for other apps
- Reduced the amount of one-off demo-specific payment orchestration
- Improved the repository’s ability to demonstrate the product in SDK-first terms

### 4.2 Basic vs Advanced Input Paths Became Clearer

**Problem Identified**: The demo needed a cleaner distinction between a guided invoice-first flow and a lower-level manual route-testing flow.

**Solution Implemented**:
- Continued the Basic/Advanced mode pattern with a more minimal advanced surface
- Added pubkey input validation for sender and receiver fields
- Added invoice validation for the basic flow
- Added clearer form behavior around generated invoices versus pasted invoices and direct receiver entry

**Primary Files**:
- `apps/demo2/src/App.tsx`
- `apps/demo2/src/components/InvoiceBox.tsx`
- `apps/demo2/src/components/PaymentForm.tsx`
- `apps/demo2/src/components/HeroCard.tsx`

**Impact**:
- Reduced confusion in the judge-facing workflow
- Preserved advanced testing paths without overwhelming the default experience
- Improved the realism of the invoice-first integration story

### 4.3 Wallet Context and Operator Feedback Improvements

**Problem Identified**: The demo needed stronger feedback loops around wallet context and runtime state so the hosted flow felt more concrete.

**Solution Implemented**:
- Preserved JoyID wallet integration while adding clearer wallet-state handling
- Added wallet-balance lookup through CKB RPC based on the connected wallet address
- Preserved sender/receiver balance-tracking surfaces around invoice generation and execution-related flows

**Primary Files**:
- `apps/demo2/src/App.tsx`
- `apps/demo2/src/components/HeroCard.tsx`

**Impact**:
- Improved operator visibility during the live demo path
- Made the app feel less like a static form and more like a real payment client
- Strengthened the end-to-end story for judge-facing interactions

---

## 5. SDK UI Refinement and Route Visualization

### 5.1 Modal Evolved Into the Main SDK Execution Surface

**Problem Identified**: The preview modal needed to become more than a static summary pane. It had to act as the main SDK-owned review and execution surface.

**Solution Implemented**:
- Continued evolving `VertifyPreviewModal` as the central SDK modal
- Added support for execution controls, source labeling, continue messaging, and payment status feedback
- Wired the modal into the React presenter so it can launch, execute, poll status, and close cleanly in browser apps

**Primary Files**:
- `packages/ui/src/components/VertifyPreviewModal.tsx`
- `packages/react/src/createVertModalPresenter.tsx`
- `packages/adapters/src/types.ts`

**Impact**:
- Made the modal a true product surface rather than a passive preview
- Improved SDK ownership of the live review-and-continue flow
- Tightened the relationship between analysis, execution, and feedback in one UI layer

### 5.2 Efficiency and Pass-Possibility Graphs

**Problem Identified**: The route/funding summary needed a clearer visual language to communicate whether the payment looks healthy and how strong the observed route set really is.

**Solution Implemented**:
- Expanded the funding/routes card with visual score gauges for:
  - route efficiency
  - pass possibility
- Added expandable advanced detail for discovered routes, effective capacity, route status, node path, and per-hop information
- Improved route detail formatting for long pubkeys and path strings

**Primary Files**:
- `packages/ui/src/components/VertFundingRoutesCard.tsx`
- `packages/ui/src/components/VertifyPreviewModal.tsx`

**Impact**:
- Made the route-planning story much easier to scan quickly
- Added more defensible visual explanation of live route quality
- Improved the balance between summary-level clarity and deep diagnostics

### 5.3 Repeated Noise Reduction and Layout Simplification

**Problem Identified**: As more route and execution data appeared in the modal, the UI risked becoming too noisy for a judge-facing flow.

**Solution Implemented**:
- Iterated repeatedly on header layout, modal positioning, section reduction, and advanced-surface minimization
- Removed or collapsed lower-value detail where it was distracting the main story
- Kept advanced diagnostics available without letting them dominate the default experience

**Primary Files**:
- `packages/ui/src/components/VertifyPreviewModal.tsx`
- `packages/ui/src/components/SuggestedActions.tsx`
- `apps/demo2/src/components/HeroCard.tsx`
- `apps/demo2/src/components/PaymentForm.tsx`

**Impact**:
- Improved the readability of the SDK preview surface
- Helped the product presentation stay focused on the most meaningful information
- Made the live demo more polished and easier to defend in front of judges

---

## 6. Documentation and Product Narrative Changes

### 6.1 README Repositioned Around SDK and Execution Strategy

**Problem Identified**: The project documentation needed to catch up with the product’s shift from a readiness checker toward a route-aware execution-planning SDK.

**Solution Implemented**:
- Rewrote the README around the idea that Vert determines not only whether a payment can pass, but how it should pass
- Framed single-route versus collectively routable multi-route execution as a core product distinction
- Added explicit examples for:
  - creating a `Vert` instance
  - running `vert.vertify(...)`
  - mounting the SDK modal in React/browser apps
  - continuing to payment through the SDK

**Primary Files**:
- `README.md`

**Impact**:
- Made the current product direction much clearer to readers
- Improved the repository’s external story for judges and developers
- Better aligned documentation with the actual API surface now present in the codebase

### 6.2 Product Boundaries Became Clearer Across Packages

**Problem Identified**: As more features landed, the repo needed a clearer articulation of how responsibilities are split across core logic, adapters, UI, and app integration.

**Solution Implemented**:
- Reinforced package-level boundaries in code and docs:
  - `@vert/core` for deterministic evaluation and planning logic
  - `@vert/adapters` for SDK facade and runtime integration
  - `@vert/ui` for the shared product surface
  - `@vert/react` for browser/React embedding
- Reflected that packaging story in README examples and workspace wiring

**Primary Files**:
- `README.md`
- `package.json`
- `packages/react/package.json`
- `packages/adapters/src/vert.ts`

**Impact**:
- Improved architectural clarity
- Made Vert’s repo structure easier to explain
- Reduced ambiguity about what the actual product now is

---

## 7. Challenges and Solutions

### Challenge 1: Vert needed a real SDK surface, not just internal layers
**Solution**: Added the `Vert` facade, expanded adapter types, and rewired the demo to use the SDK directly.

### Challenge 2: Route planning needed to move beyond primary/fallback summaries
**Solution**: Added richer route models, route ranking, split-allocation planning, and execution-plan summaries throughout `@vert/core`.

### Challenge 3: The browser integration story was incomplete
**Solution**: Added the `@vert/react` presenter package so the SDK modal can mount and operate inside React/browser apps without app-specific modal logic.

### Challenge 4: Live topology reasoning needed to be based on actual observed routes
**Solution**: Expanded the Fiber live adapter to collect node snapshots, construct directed edges, discover node paths, and derive route details from live channel information.

### Challenge 5: The modal risked becoming too dense as more diagnostics were added
**Solution**: Iterated repeatedly on layout, removed excess sections, collapsed noisy detail, and emphasized the most meaningful metrics and actions first.

### Challenge 6: Split-route execution planning outpaced currently validated live execution semantics
**Solution**: The code now plans split execution explicitly while still treating unsupported live split submission honestly instead of pretending it is already production-ready.

---

## 8. Impact Assessment

### Quantitative Impact
- **Commit Volume**: 14 commits since `3cab2d6`
- **Code Change Volume**: 2,035 lines added / 484 removed
- **Files Touched**: 31 files in the inspected range
- **New Product Surface**: 1 new package added (`@vert/react`)
- **Primary Areas Improved**: SDK facade, route-execution planning, topology-aware live discovery, browser modal integration, judge-facing flow simplification, route visualization

### Qualitative Impact
- **Product Maturity**: Vert now looks much more like a real SDK offering rather than only a working demo plus internal logic
- **Architectural Strength**: Package boundaries are clearer and the new React bridge makes the integration story significantly more believable
- **Planning Depth**: The system is materially closer to true execution planning because it now models ranked routes, allocations, and split strategies directly
- **Demo Credibility**: `demo2` is now a much stronger reference integration because it consumes the SDK rather than bypassing it
- **Presentation Quality**: The modal and route diagnostics are more persuasive, more focused, and easier to scan in a live demo setting

---

## 9. Next Week's Priorities

### Immediate Focus
1. Validate and harden live execution semantics for route-aware continuation, especially where split-route planning is available but live split submission is not yet trusted
2. Continue hardening the hosted API and live adapter assumptions so route discovery does not depend too heavily on demo-topology conventions
3. Keep refining the SDK-owned modal so route insight and payment lifecycle feedback remain strong without reintroducing UI noise

### Strategic Initiatives
1. Extend Vert from split-route planning into a more fully validated multi-path execution model
2. Decide how much of the new React/browser integration should be treated as first-class SDK distribution versus demo-support packaging
3. Continue deepening route-quality scoring so fees, capacity, and path quality produce even more trustworthy execution recommendations

### Documentation
1. Keep the README aligned with the evolving SDK API surface
2. Document the `@vert/react` integration pattern more explicitly as it stabilizes
3. Continue weekly reporting so the productization path remains easy to audit historically

---

## 10. Lessons Learned

1. **A strong facade matters**: once the SDK gained a single `Vert` entry point, the product story became much easier to understand and integrate.
2. **Route planning needs first-class types**: split execution, ranked routes, and hop-level detail cannot be bolted on cleanly without shared core models.
3. **Embedding matters as much as logic**: a planning SDK is more believable when it ships an actual browser/React integration path instead of leaving all presentation work to host apps.
4. **Observed topology is essential**: richer planning requires route discovery based on real channels and capacities, not only abstract route labels.
5. **UI restraint is part of product quality**: as route intelligence grows, selective reduction of noise becomes as important as adding more detail.
6. **Honest capability boundaries improve credibility**: explicitly marking unsupported live split execution is better than implying the feature is already fully operational.

---

## 11. Conclusion

This reporting period moved Vert decisively toward being a real SDK product. The repository now includes a high-level `Vert` facade, a new React integration package, richer route and execution-plan models, deeper topology-aware live route discovery, and a demo application that behaves much more like a true consumer of the SDK.

Just as importantly, the codebase now reflects a stronger product thesis: Vert is not only trying to predict whether a payment can pass, but to determine the best strategy for getting it through. The current implementation now carries that idea across core planning logic, adapter behavior, hosted API responses, UI presentation, and browser integration. The next major challenge is to validate more of the live execution side so the increasingly sophisticated planning model can be matched by equally credible runtime behavior.

---

**Report Prepared By**: Birdmannn  
**Date**: July 23, 2026  
**Commit Range**: `3cab2d6..HEAD`
