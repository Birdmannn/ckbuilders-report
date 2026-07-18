# Vert - Weekly Development Report
**Reporting Period**: Since commit `9e192bd`  
**Project**: Vert - Payment Readiness and Execution Planning SDK for Fiber  
**Branch**: main

---

## Executive Summary

This reporting period focused on evolving Vert from a working SDK/demo scaffold into a much more realistic product and infrastructure prototype. The biggest areas of progress were: (1) moving the payment flow closer to a live Fiber-backed experience, (2) introducing a same-repo hosted API path to support judge-facing deployment, (3) reshaping the UI toward a clearer Basic/Advanced workflow, and (4) building local multi-node tooling to validate channel topology, liquidity, and payment behavior across a realistic demo network.

The project also became substantially more explicit in its product positioning. Instead of treating Vert as a simple readiness checker, the implementation and documentation moved closer to the stronger idea that Vert is an execution-planning SDK that reasons about route quality, fallback paths, and, ultimately, aggregate liquidity across multiple routes.

At the code level, the week added invoice generation support, hosted API integration, local node/topology automation scripts, richer route/funding UI, and multiple rounds of refinement to the judge-facing demo surface. At the operational level, the team validated real two-node payment flow and made meaningful progress toward a four-node route-planning topology.

**Key Metrics**:
- 12 commits since `9e192bd`
- 30 primary files changed
- 1,557 lines added
- 397 lines removed
- 3 major workstreams advanced in parallel: hosted API/live payment flow, multi-node Fiber topology tooling, and judge-facing UI/UX refinement
- 1 successful two-node end-to-end local validation achieved (peer connect, channel ready, invoice creation, send payment, payment success)

---

## 1. Hosted API and Live Runtime Architecture

### 1.1 Same-Repo Hosted API App

**Problem Identified**: A judge-facing demo should not depend on browser-direct local Fiber RPC access. The project needed a hosted-runtime architecture that keeps Vert usable as an SDK while moving live payment infrastructure behind a backend layer.

**Solution Implemented**:
- Added a same-repo API application under `apps/api`
- Exposed a health endpoint and live workflow endpoints for evaluation, invoice generation, payment continuation, and payment status polling
- Preserved the modular architecture where `@vert/core` stays pure, `@vert/ui` stays reusable, and the backend becomes runtime support rather than the product itself

**Primary Files**:
- `apps/api/src/server.ts`
- `apps/api/src/routes/evaluate.ts`
- `apps/api/src/routes/invoices.ts`
- `apps/api/src/routes/payments.ts`

**Impact**:
- Created a deployment-ready backend surface for live judge demos
- Reduced dependency on browser-direct Fiber node access
- Positioned Vert for a managed-hosted runtime model without discarding the SDK structure

### 1.2 API-Backed Adapter Path

**Problem Identified**: The frontend needed a clean way to call a hosted runtime without embedding direct Fiber node logic in the browser path.

**Solution Implemented**:
- Added an API-backed adapter client that calls the hosted API instead of direct Fiber RPC
- Preserved the role of `@vert/adapters` as the integration layer that apps use
- Allowed `demo2` to talk to a configurable backend URL rather than hardcoded localhost-only runtime behavior

**Primary Files**:
- `packages/adapters/src/live/vertApiAdapter.ts`
- `packages/adapters/src/live/index.ts`
- `packages/adapters/src/index.ts`
- `apps/demo2/src/vite-env.d.ts`
- `apps/demo2/.env.example`

**Impact**:
- Kept the SDK architecture intact while making a hosted deployment path viable
- Improved environment flexibility for local, staging, and deployed use
- Made the frontend deployment less coupled to local machine assumptions

### 1.3 Environment and Build Path Adjustments

**Problem Identified**: Workspace-based deployment required better handling of package build order, hosted API URL configuration, and frontend deployment assumptions.

**Solution Implemented**:
- Added environment-based API URL support for `demo2`
- Fixed TypeScript/Vite environment typing for frontend config
- Added the API app into the monorepo build path
- Adjusted package build scripts to force-refresh project builds when necessary

**Primary Files**:
- `package.json`
- `apps/demo2/src/App.tsx`
- `apps/demo2/tsconfig.json`
- `apps/api/package.json`
- `apps/api/tsconfig.json`
- `packages/core/package.json`
- `packages/adapters/package.json`
- `packages/ui/package.json`

**Impact**:
- Improved reliability of workspace builds
- Reduced confusion in local vs hosted deployment configuration
- Created a clearer path toward one-click demo deployment for judges

---

## 2. Payment Flow Evolution: From Request-Driven to Invoice-First

### 2.1 Invoice Generation Support

**Problem Identified**: The demo flow was too request-parameter-driven and did not yet feel like a natural Fiber payment experience. Receiver-side invoice generation needed to become part of the flow.

**Solution Implemented**:
- Added backend support for generating receiver-side invoices
- Extended the live Fiber adapter with invoice generation logic
- Added frontend support for invoice generation state, notes, and integration into the continuation flow

**Primary Files**:
- `packages/adapters/src/live/fiberRpc.ts`
- `packages/adapters/src/live/fiberAdapter.ts`
- `packages/adapters/src/live/vertApiAdapter.ts`
- `apps/api/src/routes/invoices.ts`
- `apps/demo2/src/App.tsx`
- `apps/demo2/src/components/InvoiceBox.tsx`

**Impact**:
- Moved the product toward a more canonical payment request model
- Reduced the conceptual reliance on raw destination-only payment initiation
- Laid the foundation for a cleaner judge-facing payment story

### 2.2 Basic vs Advanced Payment Modes

**Problem Identified**: The judge-facing app needed a simpler, more approachable default flow while still preserving a path for technical inspection and direct payment debugging.

**Solution Implemented**:
- Added a Basic/Advanced toggle group
- Basic mode became invoice-first and more guided
- Advanced mode preserved the direct request-entry model for deeper operator/developer use
- Tuned the toggle layout to match the card width and fit the overall UI system more cleanly

**Primary Files**:
- `apps/demo2/src/App.tsx`
- `apps/demo2/src/components/ModeToggle.tsx`
- `apps/demo2/src/components/InvoiceBox.tsx`
- `apps/demo2/src/components/PaymentForm.tsx`

**Impact**:
- Made the main flow easier for judges to understand
- Kept advanced debugging capabilities available without overwhelming the primary demo
- Improved the product framing around “guided” versus “manual” payment behavior

### 2.3 Continue-to-Payment Flow Refinement

**Problem Identified**: The payment continuation flow needed to be more truthful and less dependent on premature invoice generation or ambiguous request semantics.

**Solution Implemented**:
- Moved invoice generation closer to the payment-intent moment in the Basic flow
- Preserved generated invoice state for continuation when available
- Kept direct payment continuation available in Advanced mode
- Ensured the continue flow can operate with either the request-based or invoice-based path, depending on context

**Primary Files**:
- `apps/demo2/src/App.tsx`
- `apps/demo2/src/components/VertifyModal.tsx`
- `packages/adapters/src/live/fiberAdapter.ts`
- `packages/adapters/src/live/vertApiAdapter.ts`
- `apps/api/src/routes/payments.ts`

**Impact**:
- Made the payment flow closer to real product intent
- Improved continuity between verification and actual execution
- Reduced the gap between demo UX and underlying Fiber semantics

---

## 3. Live Payment Continuation and Status Handling

### 3.1 Live Continuation Path Through `send_payment`

**Problem Identified**: After preflight, the system still needed to prove whether a payment could actually continue through live infrastructure rather than stopping at a static success message.

**Solution Implemented**:
- Added a real continuation path using `send_payment`
- Normalized the return shape into app-friendly lifecycle data
- Preserved strict separation between live and fallback behavior so mock-backed evaluation cannot masquerade as a live payment attempt

**Primary Files**:
- `packages/adapters/src/live/fiberRpc.ts`
- `packages/adapters/src/live/fiberAdapter.ts`
- `apps/demo2/src/App.tsx`
- `apps/demo2/src/components/VertifyModal.tsx`

**Impact**:
- Made the Vert demo more than a preflight checker
- Introduced a real downstream payment proof path
- Improved the credibility of the SDK preview experience

### 3.2 Payment Status Polling

**Problem Identified**: `send_payment` submission alone is not sufficient for a convincing demo; the app also needed to reflect evolving payment lifecycle states.

**Solution Implemented**:
- Added adapter-level status normalization for `Created`, `Inflight`, `Success`, and `Failed`
- Added app-level polling after submission for non-terminal results
- Updated modal rendering to distinguish in-progress versus terminal payment outcomes

**Primary Files**:
- `packages/adapters/src/live/fiberAdapter.ts`
- `apps/demo2/src/App.tsx`
- `apps/demo2/src/components/VertifyModal.tsx`

**Impact**:
- Improved the realism of the payment lifecycle demo
- Reduced the chance of misrepresenting “accepted” as “succeeded”
- Gave the SDK preview a stronger operational feel

---

## 4. Multi-Node Fiber Topology Tooling and Route Demo Infrastructure

### 4.1 Four-Node Bootstrap and Operational Scripts

**Problem Identified**: Vert’s value depends on route and liquidity behavior, which cannot be meaningfully demonstrated with a single isolated node. The project needed repeatable local tooling for a multi-node Fiber environment.

**Solution Implemented**:
- Added helper scripts to bootstrap and inspect a local four-node Fiber topology
- Added node setup wrappers, verification helpers, peer connection scripts, channel opening scripts, topology checks, liquidity checks, and stop scripts
- Created a parallel script path in the local Fiber workspace to provision and run the nodes themselves

**Primary Files**:
- `scripts/setup_four_fiber_nodes.sh`
- `scripts/verify_four_fiber_nodes.sh`
- `scripts/connect_fiber_peers.mjs`
- `scripts/open_fiber_channels.mjs`
- `scripts/check_fiber_topology.mjs`
- `scripts/check_fiber_liquidity.mjs`
- `scripts/stop_four_fiber_nodes.sh`
- `scripts/fiber-topology-common.mjs`
- `scripts/channel-bootstrap-common.mjs`
- `scripts/two_node_validation.mjs`

**Impact**:
- Turned topology setup from ad hoc experimentation into repeatable operator tooling
- Created the basis for local route planning and liquidity experiments
- Strengthened Vert’s path toward a real execution-planning demo story

### 4.2 Two-Node Payment Proof

**Problem Identified**: Before validating complex route planning, the team needed proof that basic peer connect, channel open, invoice generation, and payment success worked at all.

**Solution Implemented**:
- Created a dedicated two-node validation script
- Successfully verified the direct `node1 -> node2` payment flow with:
  - peer connection
  - channel creation
  - invoice generation
  - payment submission
  - payment status polling
  - post-payment balance/state changes

**Primary Files**:
- `scripts/two_node_validation.mjs`

**Impact**:
- Proved the environment can support real payment flow, not just topology display
- Established a reliable reference baseline for future route-comparison work
- Reduced uncertainty before moving into multi-node path planning

### 4.3 Liquidity Interpretation and Route Capacity Reasoning

**Problem Identified**: Simply having nodes and channels is not enough; the team needed to understand directional liquidity and route bottlenecks across a multi-hop topology.

**Solution Implemented**:
- Added per-channel outbound/inbound liquidity inspection
- Added route summary computation for:
  - route A (`node1 -> node2 -> node4`)
  - route B (`node1 -> node3 -> node4`)
- Refined the route summary to use the strongest ready channel per edge rather than being distorted by stale or tiny duplicates
- Began identifying the difference between a structurally valid topology and a liquidity-rich topology

**Primary Files**:
- `scripts/check_fiber_liquidity.mjs`

**Impact**:
- Made the route-planning constraints more concrete
- Clarified where the network was strong versus still too weak for a convincing fallback-route demo
- Highlighted what needs to be improved before multi-part execution planning becomes believable

---

## 5. UI and Product Experience Refinement

### 5.1 SDK-Owned Preview Modal

**Problem Identified**: The Vertify preview UI was initially too tightly owned by the demo app rather than the SDK itself.

**Solution Implemented**:
- Moved the Vertify preview modal into the shared UI package
- Re-exported it into demo2 from the SDK layer
- Kept the preview shell dark, glassy, and visually distinct from the host app

**Primary Files**:
- `packages/ui/src/components/VertifyPreviewModal.tsx`
- `packages/ui/src/index.ts`
- `apps/demo2/src/components/VertifyModal.tsx`

**Impact**:
- Strengthened the case that the preview surface belongs to the SDK, not just the demo
- Improved architectural consistency between host app and shared UI layer
- Made the modal reusable across future host apps

### 5.2 Preview UI Styling and Typography Direction

**Problem Identified**: The preview needed a clearer visual identity and a more deliberate presentation system, especially in the SDK-owned modal.

**Solution Implemented**:
- Introduced modal-specific typography pairing using:
  - `Space Grotesk` for major headings/key labels
  - `DM Sans` for body text inside the preview modal
- Adjusted the modal to a darker, glassy, centered preview shell
- Reduced horizontal overflow and improved width handling for inner content such as execution plan data

**Primary Files**:
- `apps/demo2/index.html`
- `packages/ui/src/components/VertifyPreviewModal.tsx`
- `packages/ui/src/components/ExecutionPlanCard.tsx`
- `packages/ui/src/styles/tokens.ts` (temporary/global changes later scoped back)

**Impact**:
- Improved visual distinction between host app and SDK preview
- Made the Vertify experience more intentional and product-like
- Reduced layout issues inside the modal

### 5.3 Funding and Route Visibility in the Preview

**Problem Identified**: The preview modal should surface the immediate operational facts a user or judge cares about before diving into deeper planning detail.

**Solution Implemented**:
- Added a funding/routes card to the SDK UI
- Surfaced route candidate count, funding/liquidity confidence, primary route, and fallback routes before the deeper evaluation and execution plan cards

**Primary Files**:
- `packages/ui/src/components/VertFundingRoutesCard.tsx`
- `packages/ui/src/components/VertifyPreviewModal.tsx`
- `packages/ui/src/index.ts`

**Impact**:
- Improved scanability of the most important live routing facts
- Made the preview UI more aligned with Vert’s route-planning identity
- Helped bridge raw runtime state and user-facing explanation

### 5.4 Demo2 UI Interaction Refinements

**Problem Identified**: The frontend still needed multiple rounds of visual and interaction cleanup to feel strong enough for a public-facing demo.

**Solution Implemented**:
- Refined the test dapp header and info-dropdown behavior
- Improved card layout and footer spacing
- Added footer-level amount input alongside Vertify
- Reworked the Basic/Advanced toggle width and card alignment
- Shifted invoice and request surfaces into more coherent visual hierarchy

**Primary Files**:
- `apps/demo2/src/App.tsx`
- `apps/demo2/src/components/Header.tsx`
- `apps/demo2/src/components/HeroCard.tsx`
- `apps/demo2/src/components/ModeToggle.tsx`
- `apps/demo2/src/components/InvoiceBox.tsx`
- `apps/demo2/src/components/PaymentForm.tsx`

**Impact**:
- Made the judge-facing demo much more polished
- Clarified the distinction between guided and advanced paths
- Reduced friction in interacting with the payment request and preview UI

---

## 6. Documentation and Product Narrative Changes

### 6.1 README Repositioned Around Execution Planning

**Problem Identified**: The project needed a stronger explanation of Vert as an execution-planning SDK rather than a simple readiness checker.

**Solution Implemented**:
- Rewrote the README around the idea that Vert determines whether a payment is:
  - individually routable
  - or collectively routable across multiple paths
- Framed aggregate liquidity and split-route planning as part of the core product direction
- Clarified the package roles, deployment direction, and repository structure accordingly

**Primary Files**:
- `README.md`

**Impact**:
- Improved the conceptual story of Vert
- Better aligned the documentation with the long-term multi-route product vision
- Made the SDK’s future value proposition clearer to readers and judges

### 6.2 V2 and Deployment Narrative

**Problem Identified**: The repository needed stronger written artifacts explaining the intended SDK direction and deployment model beyond the immediate hackathon implementation.

**Solution Implemented**:
- Added a V2 implementation direction document
- Added a deployment guide covering hosted frontend, hosted API, and Fiber node runtime expectations
- Clarified the relationship between SDK product identity and managed runtime support

**Primary Files**:
- `V2_IMPLEMENTATION.md`
- `DEPLOYMENT.md`

**Impact**:
- Created a stronger roadmap and deployment narrative
- Made the hosted-API direction easier to explain
- Supported the shift from local/dev assumptions to a judge-facing deployment mindset

### 6.3 Weekly Reporting and Project Traceability

**Problem Identified**: As the codebase evolved quickly, the project needed durable reporting and historical documentation of what changed and why.

**Solution Implemented**:
- Added a first Vert-specific weekly report covering the repo’s initial buildout
- Continued project-level explanation and repository framing through documentation updates

**Primary Files**:
- `WEEK1_REPORT2.md`

**Impact**:
- Improved continuity and retrospective clarity
- Made the repo easier to audit and explain over time

---

## 7. Challenges and Solutions

### Challenge 1: Moving from a static prototype to a believable live payment demo
**Solution**: Added live adapter logic, hosted API paths, real JoyID wallet connect, real `send_payment` continuation, and status polling while keeping fallback behavior explicitly labeled.

### Challenge 2: Distinguishing route topology from actual route quality
**Solution**: Built a four-node topology, then added liquidity-checking scripts and route-summary logic so the team could reason about effective route capacity rather than just node connectivity.

### Challenge 3: Avoiding confusion between invoice-driven and raw request-driven flows
**Solution**: Added a Basic/Advanced split, began invoice-first migration, and kept the raw form as a fallback/advanced path rather than the primary UX.

### Challenge 4: Keeping the SDK identity intact while adding hosted runtime support
**Solution**: Preserved package boundaries and moved backend/runtime behavior into `apps/api` and `@vert/adapters` while keeping `@vert/core` pure and `@vert/ui` reusable.

### Challenge 5: Aligning the UI with the product architecture
**Solution**: Moved the preview modal into the SDK package, refined visual hierarchy, and made the modal the shared product surface rather than a one-off demo-only implementation.

---

## 8. Impact Assessment

### Quantitative Impact
- **Commit Volume**: 12 commits since `9e192bd`
- **Code Change Volume**: 1,557 lines added / 397 removed
- **Files Touched**: 30 files in the inspected range
- **Primary Areas Improved**: hosted API architecture, invoice-first flow, live payment continuation, multi-node topology tooling, route/liquidity inspection, SDK-owned preview UI
- **Major Validation Achieved**: one real two-node end-to-end Fiber payment proof

### Qualitative Impact
- **Product Maturity**: Vert is now much closer to a believable SDK + hosted demo product instead of just a code experiment
- **Demo Credibility**: real payment flow, invoice generation, and status polling are all now significantly more concrete
- **Operational Clarity**: the team now has better visibility into topology health, route bottlenecks, and where the live demo still needs more liquidity
- **Documentation Strength**: README, V2, deployment, and reporting artifacts now better express the project’s real direction

---

## 9. Next Week's Priorities

### Immediate Focus
1. Finish the invoice-first migration so Vertify is fully driven by generated invoice truth instead of the raw request fallback path
2. Strengthen the fallback route liquidity in the four-node topology so Vert can demonstrate a more meaningful multi-route story
3. Continue hardening the hosted API and deployment path for a public judge-facing demo

### Strategic Initiatives
1. Expand toward true multi-path payment planning beyond primary/fallback route selection
2. Decide how much of the local topology tooling should become more formal operator infrastructure
3. Continue refining the SDK preview experience so the shared UI tells the product story more clearly

### Documentation
1. Keep README and deployment docs aligned with the actual invoice-first/live-path behavior
2. Document the local four-node topology and route-capacity assumptions more explicitly
3. Continue weekly reporting as the architecture and topology mature

---

## 10. Lessons Learned

1. **A route-planning SDK needs both logic and topology**: without enough directional liquidity, even a well-designed planner cannot prove much.
2. **Hosted architecture matters early for judge demos**: browser-direct node access is fine for local testing, but a hosted API path is much easier to defend operationally.
3. **Invoice-first UX is easier to explain than raw routing params**: it aligns better with real payment intuition and reduces user confusion.
4. **Two-node proof is the right first milestone**: proving a direct channel payment works gives a stable baseline before scaling to richer multi-node planning.
5. **The UI must match the architecture**: moving the preview modal into the SDK package clarified both ownership and product identity.

---

## 11. Conclusion

This reporting period pushed Vert meaningfully beyond its initial SDK scaffold and into a much more realistic product state. The project now includes a same-repo hosted API, invoice-generation infrastructure, real payment continuation and polling, stronger local topology tooling, and a clearer divide between guided and advanced demo flows.

Just as importantly, the team learned that route planning quality depends not only on the software architecture but also on the actual liquidity structure of the network being demonstrated. The four-node topology is now structurally present, the two-node payment proof is working, and the next challenge is making the multi-route path strong enough that Vert can demonstrate the richer execution-planning story it is designed to tell.

---

**Report Prepared By**: Birdmannn  
**Date**: July 16, 2026  
**Commit Range**: `9e192bd..HEAD`
