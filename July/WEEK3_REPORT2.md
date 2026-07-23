# Vert / Fiber - Weekly Development Report
**Reporting Period**: Since the Week 2 baseline and subsequent multiroute execution work  
**Project**: Vert - Payment Readiness and Execution Planning SDK for Fiber  
**Branch**: main

---

## Executive Summary

This reporting period focused on turning Vert’s route-planning story into something much more technically honest and operationally meaningful. The biggest lesson learned was that there is a major difference between:

- **route-aware planning metadata**, and
- **real multiroute live execution**.

At the start of this phase, Vert had already advanced beyond a simple readiness checker. It could discover live routes, compute ranked route options, attach route details to preflight output, and thread a `routeExecutionPlan` from analysis into the execute path. However, the team discovered that this was still not equivalent to a true multiroute executor.

The core improvements this week therefore fell into three categories:

1. **Vert SDK ergonomics and ownership improvements** — introducing a higher-level `Vert` object API, configurable sender authorities, SDK-owned modal presentation infrastructure, and updated SDK documentation.
2. **Route-planning and execution-structure improvements** — building split-aware planning types, execution-stable route references, aggregate-capacity-aware reasoning, and safer split-plan threading.
3. **Execution honesty and Fiber validation discipline** — explicitly failing closed for unvalidated multiroute live execution instead of pretending that provisional Fiber hints already guarantee split-route behavior.

This was an important maturity step. The codebase now better distinguishes between:
- what Vert can **plan**,
- what Vert can **surface to the app/UI**, and
- what Fiber has been **proven to execute live**.

That distinction is the most important architectural lesson of the week.

---

## 1. Higher-Level SDK Surface and Developer Experience

### 1.1 Introduction of a Stateful `Vert` SDK Facade

**Problem Identified**: The prior public API was still fragmented across adapter factories and app-owned orchestration. It was difficult to express the product vision through a single SDK object with reusable methods.

**Solution Implemented**:
- Added a first-pass `Vert` class facade in the adapters layer
- Exposed methods for:
  - `vertify(...)`
  - `vertifyOnModal(...)`
  - `execute(...)`
  - `executeWithEfficiency(...)`
  - `executeWithKeys(...)`
  - `generateInvoice(...)`
  - `getPaymentStatus(...)`
- Added authority registration via:
  - `new Vert([{ pubkey, key }])`
  - `addKey(...)`
  - `addKeys(...)`

**Primary Files**:
- `packages/adapters/src/vert.ts`
- `packages/adapters/src/types.ts`
- `packages/adapters/src/index.ts`

**Impact**:
- Moved the product from a factory/function surface toward a recognizable SDK object
- Created a clearer place to accumulate payment orchestration behavior over time
- Improved readability of demo and app integrations

### 1.2 Authority Model Clarification

**Problem Identified**: The desired notion of “sender node configuration” was initially ambiguous, especially around what `key` meant in this phase.

**What We Learned**:
- There is no reusable local signer/private-key execution layer in the current Vert/Fiber integration path
- In practice, `key` is best treated as **authority metadata** for now, not as a local signing primitive
- The configured `pubkey` identifies execution nodes or execution candidates; it does not magically turn app-side funds into channel liquidity on those nodes

**Implementation Outcome**:
- Kept `key` as metadata in the SDK surface
- Preserved `pubkey`-based sender selection behavior while deferring true signing semantics

**Impact**:
- Prevented the SDK from overpromising local-signing capability
- Kept the current API surface compatible with future settlement/provider work
- Clarified the difference between authority selection and actual liquidity execution

---

## 2. SDK-Owned Modal and Browser/React Integration

### 2.1 From App-Owned Modal Flow to SDK-Owned Modal Flow

**Problem Identified**: Even after earlier UI work, the modal experience still needed a clean SDK-owned entrypoint so host apps could call a single method and let the SDK own the Vertify/Commence presentation lifecycle.

**Solution Implemented**:
- Added presenter contracts to the adapters SDK types
- Made `vertifyOnModal(...)` a real SDK method instead of a stub
- Created a browser/React bridge package to imperatively mount the shared Vert modal UI
- Kept the core `Vert` facade headless while allowing browser-hosted apps to opt into SDK-owned modal behavior

**Primary Files**:
- `packages/adapters/src/types.ts`
- `packages/adapters/src/vert.ts`
- `packages/react/package.json`
- `packages/react/tsconfig.json`
- `packages/react/src/index.ts`
- `packages/react/src/createVertModalPresenter.tsx`

**Impact**:
- Turned `vertifyOnModal(...)` into a usable SDK feature in React/browser contexts
- Reduced host-app responsibility for modal lifecycle, execute action handling, and payment-status polling
- Improved the conceptual coherence of Vert as a product surface, not just a library of utilities

### 2.2 Demo App Refactor to Use SDK Modal Flow

**Problem Identified**: The judge-facing demo was still manually coordinating some modal and flow behavior that should belong to the SDK layer.

**Solution Implemented**:
- Refactored `demo2` to instantiate Vert with a modal presenter
- Migrated the app to call `vert.vertifyOnModal(...)` directly
- Simplified the app so the SDK owns more of the preview/continue experience

**Primary Files**:
- `apps/demo2/src/App.tsx`
- `apps/demo2/package.json`
- `apps/demo2/tsconfig.json`
- `apps/demo2/vite.config.ts`

**Impact**:
- Strengthened the product story that Vert can own the verification/presentation loop
- Reduced duplication between app-level UI flow and shared SDK flow
- Created a cleaner reference integration for future apps

---

## 3. Shared Route Metrics and Execution Gating Improvements

### 3.1 Shared Efficiency Logic

**Problem Identified**: Route efficiency calculations were previously local to the UI card, making it harder to reuse the same logic across execution helpers and presentation.

**Solution Implemented**:
- Extracted route efficiency and pass-possibility logic into shared core utilities
- Updated the route funding UI to consume the shared helpers instead of private local copies

**Primary Files**:
- `packages/core/src/utils/efficiency.ts`
- `packages/core/src/index.ts`
- `packages/ui/src/components/VertFundingRoutesCard.tsx`

**Impact**:
- Reduced duplication
- Improved consistency between SDK execution helpers and UI scoring
- Created a cleaner base for threshold-based execution checks

### 3.2 `executeWithEfficiency(...)` and Explicit Threshold Behavior

**Problem Identified**: There needed to be a practical way to gate live execution on route quality.

**Solution Implemented**:
- Added `executeWithEfficiency(...)`
- Reused shared route-efficiency scoring in the Vert facade
- Returned structured skip/fail behavior when efficiency thresholds are not met

**Primary Files**:
- `packages/adapters/src/vert.ts`
- `packages/core/src/utils/efficiency.ts`

**Impact**:
- Introduced a more explicit execution policy surface
- Improved developer control over when live payment attempts should be allowed
- Reinforced the idea that Vert is choosing *how* and *when* to attempt execution, not just whether a route exists

---

## 4. What We Learned About Multiroute Execution

This was the most important technical discovery of the week.

### 4.1 Vert Already Had Multiroute Planning, But Not Multiroute Execution

**State at the beginning of this phase**:
- live route discovery existed
- route ranking existed
- split allocation planning existed
- `routeExecutionPlan` existed
- `execute()` could thread a plan through the API and live adapter layers

At first glance, that looked like “multiroute execute.”

**What we learned**:
That was still mostly a **control-plane story**, not a **proven data-plane execution path**.

Specifically:
- route allocations were being computed
- but allocation amounts were not actually enforced in live submission
- display route identifiers were being reused as transport hints
- Fiber hint fields (`max_parts`, `trampoline_hops`) were being populated without proof that they meant what the SDK assumed
- `hop_hints` existed in the RPC surface but were not yet used

This led to the key lesson:

> A multiroute payment plan is not the same thing as a multiroute payment executor.

---

## 5. Route-Execution Planning Maturity Improvements

### 5.1 Explicit Route Execution Planning Types

**Problem Identified**: Split allocation data was too thin and too UI-oriented to be trusted as an execution contract.

**Solution Implemented**:
- Extended route execution planning types in core
- Added execution-oriented metadata to allocations and route details
- Preserved a distinction between human-readable route identity and execution-oriented route identity

**Primary Files**:
- `packages/core/src/types.ts`
- `packages/core/src/utils/routing.ts`
- `packages/adapters/src/live/fiberAdapter.ts`

**What changed**:
- `RouteDetails` gained `executionRouteId`
- `RouteExecutionAllocation` gained:
  - `executionRouteId`
  - `hopNodes`
  - `channelIds`
- `RouteExecutionPlan` gained execution-oriented ranked route references

**Impact**:
- Reduced overloading of UI route labels as backend execution tokens
- Preserved more of the real route/hop structure for future live execution steering
- Created a safer base for Fiber hint generation later

### 5.2 Aggregate-Capacity-Aware Route Planning

**Problem Identified**: The rules/preflight system still primarily reasoned about the *best single route*, which could incorrectly block payments that are only feasible when split across multiple routes.

**Solution Implemented**:
- Added aggregate-capacity reasoning in the rules layer
- Changed the semantics so a payment that exceeds the best single route but fits within total visible route capacity is no longer treated the same as a truly impossible payment

**Primary Files**:
- `packages/core/src/engine/rules.ts`
- `packages/core/src/engine/canPay.ts`
- `packages/core/src/engine/plan.ts`
- `packages/core/src/utils/routing.ts`

**What changed**:
- best single-route capacity and aggregate route capacity are now treated differently
- split-required cases are surfaced as risky/limited rather than immediately hard-blocked
- execution plans can now narrate split allocation steps instead of only primary/fallback retry sequencing

**Impact**:
- Made multiroute-capable payments reachable by the SDK
- Improved honesty of the preflight layer
- Reduced the semantic mismatch between planning and execute

### 5.3 Split-Aware Execution Narration

**Problem Identified**: Even when `routeExecutionPlan.requiresSplit === true`, the user-facing execution plan still spoke mostly in terms of one primary route plus fallback retries.

**Solution Implemented**:
- Updated execution-step generation to describe split allocations when split execution is required

**Primary Files**:
- `packages/core/src/engine/plan.ts`
- `packages/core/src/engine/plan.test.ts`

**Impact**:
- Aligned plan narration with the real plan structure
- Reduced confusion between “fallback route” and “parallel or multipart allocation”
- Improved the credibility of Vert’s execution-plan explanations

---

## 6. Honest Live Execution: Fail Closed Until Fiber Semantics Are Proven

### 6.1 Why This Was Necessary

**Problem Identified**: The live adapter was starting to pass multiroute hints into Fiber RPC, but without enough evidence that Fiber would actually honor those hints in the expected way.

The dangerous version of this would have been:
- plan a split allocation
- attach `max_parts`
- attach path-like `trampoline_hops`
- call it “multiroute execution”

That would have been misleading.

### 6.2 Safer Behavior Implemented

**Solution Implemented**:
- Refactored execute/live submission behavior so split execution now fails closed instead of pretending live multiroute support is already validated
- Added richer attempt metadata in the SDK result layer
- Preserved single-route live execution while refusing to silently mislabel split execution as real

**Primary Files**:
- `packages/adapters/src/types.ts`
- `packages/adapters/src/vert.ts`
- `packages/adapters/src/live/fiberAdapter.ts`
- `packages/adapters/src/live/vertApiAdapter.ts`
- `apps/api/src/routes/payments.ts`

**What changed**:
- split plans are still produced and surfaced
- split attempt metadata is included in results
- live adapter returns a structured failure for split execution until route-steering semantics are validated
- single-route live payment execution continues to work normally

**Impact**:
- Prevented the SDK from overstating its live multiroute capability
- Preserved trustworthiness of the product surface
- Drew a clear line between “planned” and “proven” execution behavior

This is arguably the most important architectural maturity improvement of the week.

---

## 7. Fiber-Specific Insights and Open Questions

### 7.1 Current Fiber Surface

From the current Fiber integration work, we now understand:
- `send_payment` supports fields like:
  - `max_parts`
  - `trampoline_hops`
  - `hop_hints`
- but the codebase has not yet proven how those fields behave under real split execution

### 7.2 What We Learned Specifically

1. **`max_parts` is only a coarse hint unless validated**
   It may allow multipart behavior, but it does not by itself prove that the planner’s per-route allocations are being honored.

2. **`trampoline_hops` is not safe to use with UI-style route IDs**
   Human-readable joined node paths are not a trustworthy execution contract.

3. **`hop_hints` looks more promising but remains unimplemented/untested**
   The route discovery layer already has enough hop/channel data to potentially build real hints, but the exact expected Fiber shape still needs validation.

4. **Observability remains a major question**
   Even if Fiber accepts split-routing hints, Vert still needs enough feedback to map actual multipart outcomes into SDK results.

### 7.3 Validation Infrastructure Added

**Solution Implemented**:
- Added a dedicated multiroute validation script to start probing split-execution-related Fiber behavior

**Primary File**:
- `scripts/multiroute_validation.mjs`

**Impact**:
- Shifted the project from assumption-driven integration toward evidence-driven validation
- Created a concrete next step for proving real live split execution

---

## 8. Documentation Improvements

### 8.1 README Updated for the New SDK Surface

**Problem Identified**: The repository documentation still leaned too heavily on product narrative and did not clearly explain the current SDK usage patterns.

**Solution Implemented**:
- Added concrete README documentation for:
  - `new Vert()`
  - `new Vert([{ pubkey, key }])`
  - `addKey()` / `addKeys()`
  - `vertify(...)`
  - `vertifyOnModal(...)`
  - `execute(...)`
  - `executeWithEfficiency(...)`
  - `executeWithKeys(...)`
- Documented that `key` is currently metadata only
- Documented current route-aware execution as a first-pass planning/plumbing layer rather than a fully validated multiroute engine
- Clarified sender-pubkey fallback behavior as the SDK evolved

**Primary File**:
- `README.md`

**Impact**:
- Made the SDK more approachable to external developers
- Better aligned documentation with the current actual implementation
- Reduced confusion between aspirational architecture and implemented behavior

### 8.2 Product-Architecture Lesson Captured in Documentation

This week’s README and planning work also reinforced a deeper point:

> Vert’s long-term value is not just in exposing payment functions, but in modeling the boundary between planning, execution, and settlement honestly.

That principle now shows up more clearly in the SDK documentation and planning notes.

---

## 9. Challenges Encountered

### Challenge 1: Mistaking route planning for route execution
**Solution**: Added execution-stable route metadata, fail-closed live split behavior, and a validation script so the project stops treating unvalidated hints as proven execution control.

### Challenge 2: Single-route readiness rules blocking aggregate-feasible payments
**Solution**: Added aggregate-capacity-aware rule handling so split-feasible payments can survive preflight.

### Challenge 3: Too much app-owned UI orchestration
**Solution**: Moved more of the flow into the SDK/modal presenter pattern and gave `vertifyOnModal(...)` a real browser-hosted path.

### Challenge 4: SDK docs lagging behind implementation
**Solution**: Rewrote the README’s SDK usage section around the actual `Vert` API surface and current execution story.

### Challenge 5: Trustworthiness of live multiroute claims
**Solution**: Chose to block unvalidated split execution instead of pretending the Fiber submission path is already proven.

---

## 10. Impact Assessment

### Quantitative / Structural Impact
- Introduced a new `@vert/react` bridge package for SDK-owned browser modal behavior
- Added execution-stable route metadata across core/adapters/live route discovery
- Expanded route execution planning types and test coverage
- Added a dedicated multiroute validation script
- Refactored demo2 to use the SDK-owned modal path

### Qualitative Impact
- **SDK maturity improved**: Vert now feels more like a product-facing SDK and less like a thin wrapper around internal adapters
- **Execution honesty improved**: the system is much clearer about what is planned vs what is validated live
- **Fiber integration maturity improved**: the team now has a much better map of where Fiber semantics must be proven before multiroute execution can be claimed
- **Documentation quality improved**: SDK usage and architecture are easier to explain to developers and reviewers

---

## 11. Next Priorities

### Immediate Focus
1. Validate real Fiber semantics for:
   - `max_parts`
   - `hop_hints`
   - `trampoline_hops`
2. Determine whether split invoice execution and split keysend execution behave differently
3. Decide what the first truly enabled live multiroute mode should be based on evidence, not assumption

### Strategic Follow-Up
1. Revisit sender selection semantics so omitted sender identity can fan out across all configured execution nodes if that remains the desired product direction
2. Clarify long-term settlement/intermediary architecture if Vert is expected to coordinate execution across app-managed nodes using app-managed balances
3. Evolve split result types further if Fiber exposes enough multipart observability to report real per-part outcomes

### Documentation / Reporting
1. Keep the README aligned with whichever multiroute live-execution semantics are actually validated
2. Document the exact meaning of Fiber routing-control fields once proven
3. Continue weekly reporting with explicit separation between:
   - planning capability
   - integration capability
   - validated live behavior

---

## 12. Lessons Learned

1. **A multiroute plan is not a multiroute executor**: this became the defining lesson of the week.
2. **Execution contracts matter more than elegant scoring heuristics**: before optimizing route selection, the project had to ensure that planned routes can even be represented honestly at runtime.
3. **Failing closed is a feature, not a regression**: refusing to overclaim live multiroute execution improves product trust.
4. **Aggregate-capacity reasoning is essential**: otherwise a split-capable planner will remain trapped behind single-route preflight logic.
5. **SDK ergonomics and architectural cleanliness can coexist**: the `Vert` facade plus React presenter model showed that the SDK can be easier to use without collapsing all environment concerns into one package.

---

## 13. Conclusion

This reporting period significantly improved both the product shape and the intellectual honesty of Vert.

On the product side, Vert now has a much clearer SDK identity: a real `Vert` object, a browser/React modal integration path, richer execution helpers, and better developer-facing documentation.

On the execution side, the project made a deeper and more valuable kind of progress: it learned exactly where the line is between route-aware planning and true multiroute live execution. Rather than papering over that gap, the implementation now preserves split plans, execution-safe route metadata, and aggregate-capacity-aware reasoning while explicitly blocking unvalidated live split execution.

That is a strong foundation. The next step is not guessing harder — it is validating Fiber’s real routing-control semantics and then enabling multiroute execution only where the underlying network/runtime proves it can support Vert’s claims.

---

**Report Prepared By**: Birdmannn  
**Date**: July 23, 2026  
**Baseline Context**: Week 2 reporting + subsequent SDK and multiroute execution work
