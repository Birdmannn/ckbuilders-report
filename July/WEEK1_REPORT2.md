# Vert - Weekly Development Report
**Reporting Period**: Since commit `885cd8f`  
**Project**: Vert - Payment Readiness and Execution Planning SDK for Fiber  
**Branch**: main

---

## Executive Summary

This reporting period focused on turning an empty repository into a functioning Vert workspace with a reusable SDK core, shared UI package, multiple demo applications, live wallet connectivity, a live-or-fallback Fiber evaluation path, and a real downstream payment continuation flow. The biggest shift was moving the project from concept/specification into a concrete monorepo that can both explain and exercise Vert’s intended readiness and execution-planning experience.

The work started by establishing the repository structure, specifications, and package boundaries, then progressed into a working core evaluation engine, mock adapter scenarios, a reusable React UI kit, and a host demo application. From there, the project evolved into a stronger product demo: a dedicated wallet-first sample dapp (`apps/demo2`), a real JoyID wallet connection, a Fiber RPC-backed live adapter with explicit fallback behavior, form-driven payment input, and a continuation path that can submit and poll real payment state when live Fiber infrastructure is available.

At the same time, the project’s identity and presentation matured. The original internal naming evolved into **Vert**, the UI system was redesigned into a more polished enterprise-clean direction, and the demo surface became capable of showing both component-level previews and realistic product flows.

**Key Metrics**:
- 13 commits since `885cd8f`
- 71 files changed
- 2,677 lines added
- 633 lines removed
- 3 major workstreams advanced in parallel: SDK/core infrastructure, UI/demo experience, and live wallet/payment integration
- 2 core engine test files added and passing end-to-end in the current workspace

---

## 1. Repository Foundation and SDK Scaffolding

### 1.1 Repository Bootstrapping and Spec-First Structure

**Problem Identified**: The repository started effectively empty, with no code structure, no package layout, and no product documentation that could support implementation or external explanation.

**Solution Implemented**:
- Established a multi-package npm workspace
- Added a top-level README and implementation checklist
- Created a spec package to define features, architecture, schema, user flow, and demo story
- Chose a package layout that separates core logic, adapters, UI, and demo hosts

**Primary Files**:
- `package.json`
- `tsconfig.base.json`
- `.gitignore`
- `README.md`
- `docs/implementation-checklist.md`
- `specs/features.md`
- `specs/architecture.md`
- `specs/schema.md`
- `specs/user-flow.md`
- `specs/demo-story.md`

**Impact**:
- Turned the repo from an empty shell into an implementation-ready workspace
- Established a clear product and architectural source of truth before deeper coding began
- Reduced ambiguity for both development and judge-facing explanation

### 1.2 Workspace Packages and App Shells

**Problem Identified**: Vert needed to behave like a reusable SDK rather than a single one-off app, which required package boundaries and host app separation from the start.

**Solution Implemented**:
- Added `packages/core` for logic/contracts
- Added `packages/adapters` for integration and mock/live data sourcing
- Added `packages/ui` for reusable React presentation components
- Added `apps/demo` as the first host app and later `apps/demo2` as a more product-like sample dapp

**Primary Files**:
- `packages/core/package.json`
- `packages/adapters/package.json`
- `packages/ui/package.json`
- `apps/demo/package.json`
- `apps/demo2/package.json`

**Impact**:
- Preserved a clean SDK identity for the project
- Made it possible to evolve demos independently from the reusable packages
- Created a foundation that can later support additional host apps or a backend companion without rewriting the SDK core

---

## 2. Core Vert Engine and Deterministic Planning

### 2.1 Readiness Evaluation Engine

**Problem Identified**: Vert needed a deterministic preflight engine that could assess whether a payment is likely to succeed, explain failure/risk states, and remain reusable across products.

**Solution Implemented**:
- Defined the shared request/context/result type model
- Added validation for request and runtime context
- Implemented rule evaluation across route, liquidity, connectivity, fee, and asset signals
- Added readiness scoring and status mapping (`ready`, `risky`, `blocked`)

**Primary Files**:
- `packages/core/src/types.ts`
- `packages/core/src/utils/validation.ts`
- `packages/core/src/engine/rules.ts`
- `packages/core/src/engine/scoring.ts`
- `packages/core/src/engine/canPay.ts`

**Impact**:
- Created the core deterministic contract Vert is built around
- Allowed the same payment request to be explained consistently across demos and integrations
- Made the project feel like an actual SDK rather than a UI-only demo

### 2.2 Explanation and Action Model

**Problem Identified**: A raw readiness result is not enough for real integration; products need human-readable explanations and ordered next actions.

**Solution Implemented**:
- Added reusable explanation-layer helpers
- Mapped result data into user-facing headline/subheadline bundles
- Added ordered suggested actions for continue, reduce amount, retry later, switch asset, and other follow-up behaviors

**Primary Files**:
- `packages/core/src/explain/copy.ts`
- `packages/core/src/explain/explain.ts`
- `packages/core/src/errors.ts`

**Impact**:
- Improved the usability of the engine for wallets and apps
- Created cleaner separation between machine logic and user-facing copy
- Enabled richer reusable UI surfaces later in the cycle

### 2.3 FluxLoom-Style Execution Planning Inside Vert

**Problem Identified**: The project needed to go beyond simple readiness and show deterministic execution planning with best-route selection, fallback behavior, and next-step generation.

**Solution Implemented**:
- Extended the type system with `ExecutionPlan`, `RoutePlan`, and `ExecutionStep`
- Added a deterministic route-ranking and planning helper
- Attached execution plans directly to the preflight result
- Added test coverage for plan generation and integration with `canPay()`

**Primary Files**:
- `packages/core/src/engine/plan.ts`
- `packages/core/src/engine/plan.test.ts`
- `packages/core/src/engine/canPay.test.ts`
- `packages/core/src/index.ts`

**Impact**:
- Elevated Vert from a readiness-only idea into a planning-capable SDK
- Created a stronger demo story around “can pay?” followed by “how should this execute?”
- Made advanced-mode UI and post-verification flows significantly more meaningful

---

## 3. Adapter Layer: Mock Scenarios, Live Fiber RPC, and Payment Continuation

### 3.1 Mock Adapter and Demo Scenarios

**Problem Identified**: The project needed deterministic data and repeatable scenarios before any live integration existed.

**Solution Implemented**:
- Added a mock adapter and fixture-backed scenarios
- Defined ready, risky, and blocked sample flows
- Used these scenarios to drive both the original demo and UI gallery surfaces

**Primary Files**:
- `packages/adapters/src/types.ts`
- `packages/adapters/src/mock/fixtures.ts`
- `packages/adapters/src/mock/scenarios.ts`
- `packages/adapters/src/index.ts`

**Impact**:
- Allowed Vert to be demonstrated before live infrastructure was available
- Made UI and engine work testable and deterministic
- Created a fallback path that remains useful even after live support was introduced

### 3.2 Live Fiber Adapter and RPC Integration

**Problem Identified**: To make the demo honest and useful to judges, Vert needed to evaluate live payment conditions from actual Fiber infrastructure rather than only from local scenarios.

**Solution Implemented**:
- Added a Fiber JSON-RPC client wrapper
- Implemented RPC access for node info, channel listing, graph queries, payment submission, and payment status lookup
- Added a live adapter that transforms RPC responses into Vert’s `RuntimeContext`
- Added explicit fallback-to-mock behavior when the live endpoint is unavailable

**Primary Files**:
- `packages/adapters/src/live/fiberRpc.ts`
- `packages/adapters/src/live/fiberAdapter.ts`
- `packages/adapters/src/live/index.ts`
- `packages/adapters/src/index.ts`

**Impact**:
- Introduced the first genuinely live data path in the repository
- Allowed Vert to attempt truthful preflight against a running Fiber node/API
- Preserved demo continuity by clearly labeling fallback behavior when live access fails

### 3.3 Real Payment Continuation and Status Polling

**Problem Identified**: A judge-facing demo needed more than a preflight result — it needed a proof step showing whether a positive verification could actually continue into a real payment flow.

**Solution Implemented**:
- Added `send_payment` continuation support in the live adapter
- Blocked “continue” from pretending to be live when the evaluation source was fallback/mock
- Added `get_payment` lookup support and normalized payment phases (`in_flight`, `succeeded`, `failed`, `unknown`)
- Wired demo2 to poll non-terminal payment states after submission

**Primary Files**:
- `packages/adapters/src/live/fiberRpc.ts`
- `packages/adapters/src/live/fiberAdapter.ts`
- `apps/demo2/src/App.tsx`
- `apps/demo2/src/components/VertifyModal.tsx`

**Impact**:
- Made the live demo significantly more convincing
- Allowed the app to distinguish between submit acceptance and final payment outcome
- Reduced the risk of overstating what a live verification result really proved

---

## 4. UI Package and Visual System Evolution

### 4.1 Initial Reusable UI Kit

**Problem Identified**: The project needed a portable UI layer that could render Vert results consistently without baking all presentation into one host app.

**Solution Implemented**:
- Added reusable result-oriented components:
  - readiness card
  - execution plan card
  - reason list
  - suggested actions
  - diagnostics drawer
- Exported those components through a package-level barrel

**Primary Files**:
- `packages/ui/src/components/PaymentReadinessCard.tsx`
- `packages/ui/src/components/ExecutionPlanCard.tsx`
- `packages/ui/src/components/ReasonList.tsx`
- `packages/ui/src/components/SuggestedActions.tsx`
- `packages/ui/src/components/DiagnosticsDrawer.tsx`
- `packages/ui/src/index.ts`

**Impact**:
- Preserved the separation between Vert logic and Vert presentation
- Enabled multiple demo hosts to share the same UI primitives
- Made the project more credible as an SDK rather than a custom app skin

### 4.2 Enterprise-Clean UI Redesign

**Problem Identified**: The initial UI package was structurally correct but visually minimal and not yet polished enough for a convincing product or hackathon demo.

**Solution Implemented**:
- Expanded the token system beyond raw colors into spacing, typography, radii, shadows, and status themes
- Redesigned all major UI kit components around a cleaner, calmer, more product-grade visual language
- Improved information hierarchy and scanability in both readiness and execution plan views

**Primary Files**:
- `packages/ui/src/styles/tokens.ts`
- `packages/ui/src/components/PaymentReadinessCard.tsx`
- `packages/ui/src/components/ExecutionPlanCard.tsx`
- `packages/ui/src/components/ReasonList.tsx`
- `packages/ui/src/components/SuggestedActions.tsx`
- `packages/ui/src/components/DiagnosticsDrawer.tsx`

**Impact**:
- Increased the polish and judge-readiness of the SDK UI layer
- Made the design system reusable instead of one-off
- Improved the perceived maturity of the project

### 4.3 Action Callback Wiring

**Problem Identified**: The UI kit initially rendered actions purely for display; it could not participate in real product flows like payment continuation.

**Solution Implemented**:
- Added action callback support to `SuggestedActions`
- Passed callbacks through `PaymentReadinessCard`
- Enabled the host app to react to `PROCEED` and other action events through the shared UI instead of relying on standalone app-local buttons only

**Primary Files**:
- `packages/ui/src/components/SuggestedActions.tsx`
- `packages/ui/src/components/PaymentReadinessCard.tsx`

**Impact**:
- Made the UI layer more interactive and integration-ready
- Reduced the gap between demo-only UI and host-controlled real action flows
- Helped bridge SDK presentation with actual payment behavior

---

## 5. Demo Applications and Product Experience

### 5.1 Demo App as Preview Host

**Problem Identified**: The UI package was not a standalone app, so the project needed a consistent preview host for component and flow development.

**Solution Implemented**:
- Added and refined `apps/demo` as the main Vite host app for Vert
- Built scenario-driven readiness + planning flows
- Later evolved it into a hybrid demo + gallery preview surface

**Primary Files**:
- `apps/demo/src/App.tsx`
- `apps/demo/src/components/ResultPanel.tsx`
- `apps/demo/src/components/UiGallery.tsx`
- `apps/demo/src/components/PaymentForm.tsx`

**Impact**:
- Gave the team a concrete place to validate all shared UI states
- Allowed Vert to be shown as both a flow and a component library
- Preserved `packages/ui` as a library instead of trying to make it runnable directly

### 5.2 Wallet-First Real-Life Sample Dapp (`apps/demo2`)

**Problem Identified**: A scenario-driven component host is useful for development, but judges also need a more product-like experience that resembles a real consumer dapp.

**Solution Implemented**:
- Added a second host app, `apps/demo2`
- Designed a wallet-first flow with:
  - header
  - info affordance
  - connect-wallet interaction
  - main collectible/offer card
  - `Vertify` CTA
- Added a centered modal for the SDK UI experience

**Primary Files**:
- `apps/demo2/src/App.tsx`
- `apps/demo2/src/components/Header.tsx`
- `apps/demo2/src/components/HeroCard.tsx`
- `apps/demo2/src/components/VertifyModal.tsx`
- `apps/demo2/src/components/WalletInfoModal.tsx`

**Impact**:
- Created a much more realistic judge-facing demo surface
- Better conveyed how Vert would sit inside a real app experience
- Provided a stronger “use case” narrative beyond abstract tooling

### 5.3 Real JoyID Wallet Connection and Wallet Context

**Problem Identified**: A fake wallet toggle was not convincing enough for a serious live demo.

**Solution Implemented**:
- Integrated JoyID real wallet connection
- Normalized JoyID session state
- Reflected connected wallet/account information in the UI
- Added wallet-balance lookup through the CKB RPC path for connected users

**Primary Files**:
- `apps/demo2/src/joyid.ts`
- `apps/demo2/src/App.tsx`
- `apps/demo2/src/components/Header.tsx`
- `apps/demo2/src/components/HeroCard.tsx`

**Impact**:
- Replaced a fake connect/disconnect story with a real wallet interaction
- Strengthened the credibility of the live demo
- Created a clearer distinction between wallet identity and payment target/request data

### 5.4 Real Payment Request Input

**Problem Identified**: Even after live evaluation and continuation were added, the request itself remained hardcoded, which weakened the realism of the flow.

**Solution Implemented**:
- Added a controlled `PaymentRequest` state to demo2
- Added a local payment form for destination, amount, asset, max fee, and payment intent ID
- Replaced duplicated hardcoded request literals in both evaluation and continuation
- Cleared stale verification/payment state whenever the user edits the request

**Primary Files**:
- `apps/demo2/src/App.tsx`
- `apps/demo2/src/components/PaymentForm.tsx`
- `apps/demo2/src/components/HeroCard.tsx`

**Impact**:
- Made the demo’s payment flow significantly more honest and interactive
- Ensured preflight and continuation operate on the same live request data
- Improved the realism of the wallet-first sample dapp

---

## 6. Naming, Product Positioning, and Documentation Evolution

### 6.1 Product Naming Consolidation to Vert

**Problem Identified**: The project initially used multiple conceptual names, which risked making the product harder to explain and brand consistently.

**Solution Implemented**:
- Renamed both the package namespace and user-facing product language to **Vert**
- Updated package names, workspace scripts, import aliases, demo copy, and documentation
- Regenerated the lockfile and verified no stale product/package naming remained in the active source tree

**Primary Files**:
- `package.json`
- `package-lock.json`
- `apps/demo/package.json`
- `apps/demo2/package.json`
- `packages/*/package.json`
- `apps/demo2/src/App.tsx`
- `README.md`
- `specs/*.md`

**Impact**:
- Simplified the project story for both judges and collaborators
- Reduced namespace inconsistency across code and docs
- Made the repo easier to navigate and present

### 6.2 Product Spec and Story Refinement

**Problem Identified**: The project needed written artifacts that kept pace with rapid architecture and product-surface iteration.

**Solution Implemented**:
- Reworked README and specs around the Vert identity
- Clarified readiness vs planning concepts
- Kept architecture, schema, and flow docs aligned with the actual implementation path

**Primary Files**:
- `README.md`
- `docs/implementation-checklist.md`
- `specs/features.md`
- `specs/architecture.md`
- `specs/schema.md`
- `specs/user-flow.md`
- `specs/demo-story.md`

**Impact**:
- Improved product clarity
- Reduced drift between code and intended narrative
- Made the repo significantly easier to explain in external review contexts

---

## 7. Code Quality, Testing, and Workflow Impact

### 7.1 Core Test Coverage and Build Integrity

**Problem Identified**: As the repository became more ambitious, it needed at least foundational test coverage and repeatable build verification around the most important logic.

**Solution Implemented**:
- Added tests for:
  - readiness evaluation
  - execution-plan generation
- Repeatedly verified builds and tests after major architecture/UI/payment changes
- Cleaned up issues like stale generated `.js` / `.d.ts` files interfering with runtime or test expectations during iteration

**Primary Files**:
- `packages/core/src/engine/canPay.test.ts`
- `packages/core/src/engine/plan.test.ts`
- workspace build/test scripts in `package.json`

**Impact**:
- Increased confidence in the core engine behavior
- Reduced risk of silent regressions as the repo evolved quickly
- Improved the reliability of demo-driven iteration

### 7.2 Architectural Discipline Under Fast Iteration

**Problem Identified**: The project moved quickly from an empty repo to a fairly rich monorepo with wallet integration, live adapter work, shared UI, and multiple apps — a setup that could easily become tangled.

**Solution Implemented**:
- Preserved clear boundaries:
  - `core` remains pure logic
  - `ui` remains reusable presentation
  - `adapters` remain integration logic
  - apps remain host surfaces
- Used plan-first/spec-first iteration before each larger implementation jump
- Kept demo-specific behavior from leaking too deeply into shared packages when not necessary

**Impact**:
- Helped the project grow without collapsing into a single tangled app
- Preserved the case that Vert is an SDK, not just a demo
- Made future backend/API or additional demo work easier to reason about

---

## 8. Challenges and Solutions

### Challenge 1: Turning an empty repo into a credible SDK quickly
**Solution**: Started with a spec-first workspace scaffold, then built package boundaries, core contracts, mock adapters, UI components, and host apps incrementally.

### Challenge 2: Avoiding “fake live” demos
**Solution**: Added explicit live-vs-fallback labeling, real JoyID wallet connect, live Fiber RPC attempts, and truthful continuation gating so the app does not silently pretend fallback behavior is live.

### Challenge 3: Preserving SDK architecture while making the demo compelling
**Solution**: Kept logic in `core`, integration in `adapters`, rendering in `ui`, and experiential flows in host apps (`demo` and `demo2`).

### Challenge 4: Moving from preflight to meaningful proof
**Solution**: Added a real continuation path (`send_payment`) and later strengthened it with payment status polling so the post-verify result is more than a static success message.

### Challenge 5: Reconciling multiple product names and evolving concepts
**Solution**: Consolidated everything under `Vert` and rewrote package names, imports, docs, and demo copy to keep the story coherent.

---

## 9. Impact Assessment

### Quantitative Impact
- **Commit Volume**: 13 commits since `885cd8f`
- **Code Change Volume**: 2,677 lines added / 633 removed
- **Files Touched**: 71 files in the inspected range
- **Primary Areas Advanced**: SDK scaffolding, deterministic evaluation/planning, shared UI system, mock + live adapters, JoyID wallet integration, live payment continuation, demo host apps
- **Test Surfaces Added/Strengthened**: 2 core engine test files with 7 passing tests in the current state

### Qualitative Impact
- **Product Clarity**: Vert is now much easier to explain as a reusable SDK with demos rather than an abstract concept
- **Demo Credibility**: The project now supports a much stronger judge-facing narrative with real wallet connect, live-or-fallback evaluation, and post-submit payment lifecycle feedback
- **SDK Maturity**: The separation between packages and host apps makes the codebase feel more like real reusable infrastructure
- **Developer Velocity**: The monorepo shape and repeated build/test verification enabled rapid iteration without fully losing structure

---

## 10. Next Week's Priorities

### Immediate Focus
1. Continue hardening the live payment flow, especially around richer polling behavior, timeout handling, and clearer terminal-state UX
2. Decide whether the judge-facing deployment should talk directly to Fiber RPC or through a hosted backend/API companion
3. Tighten the real payment request semantics so destination and amount mapping are unambiguous against actual Fiber expectations

### Strategic Initiatives
1. Add a proper hosted/demo deployment architecture for judges if live local-node dependence should be removed
2. Consider whether additional shared form or status primitives should move from app-local code into `packages/ui`
3. Expand adapter capabilities beyond the current minimal RPC set if more truthful route/liquidity evaluation is needed

### Documentation
1. Keep README and specs aligned with the live evaluation/payment behavior now implemented
2. Document the expected live Fiber endpoint requirements for a real demo environment
3. Clarify how the live adapter behaves when it falls back to mock data

---

## 11. Lessons Learned

1. **A believable SDK demo needs both reusable abstractions and a strong host app**: building packages alone is not enough for judges to understand the product.
2. **Wallet realism and payment realism are separate milestones**: connecting a real wallet is valuable, but it does not automatically make the evaluation or payment path real.
3. **Explicit fallback labeling is critical for trust**: if live infrastructure is missing, the demo must say so clearly rather than pretending.
4. **Core purity pays off**: keeping `@vert/core` free from transport logic made it much easier to evolve adapters and demos independently.
5. **Product naming matters earlier than expected**: consolidating on Vert reduced confusion across code, demos, and specs.

---

## 12. Conclusion

Since the first commit, Vert has progressed from a blank repository into a structured SDK workspace with a meaningful technical and product story. It now includes deterministic preflight logic, execution planning, a reusable UI system, multiple host demos, a real JoyID wallet connection, a live-or-fallback Fiber adapter, a live payment continuation path, and polling-backed payment lifecycle feedback.

Just as importantly, the project has become easier to demonstrate and reason about. The codebase now clearly separates reusable SDK layers from host apps, the UI is far more polished, and the demo can show a path from wallet connection to readiness verification to payment continuation in a way that is much more credible than a purely mocked prototype.

---

**Report Prepared By**: Development Team  
**Date**: July 11, 2026  
**Commit Range**: `885cd8f..HEAD`
