---
name: senior-frontend-product-ui
description: Design and implement production frontend features for TypeScript React applications, including CRUD/product UI, dashboards, forms, workflow screens, and UI recreation from screenshots or mockups. Use when building independent frontend tasks, translating visual references into interfaces, integrating with APIs, or refining product UX without frontend overengineering.
compatibility: opencode
---

# Senior Frontend Product UI (Production Delivery Skill)

This is a **task skill**: a reusable set of instructions that tells an agent *how to execute* senior-level frontend work for product, admin, dashboard, and workflow interfaces.

## What this skill does

- Designs and implements production-grade frontend features for TypeScript React applications.
- Translates screenshots, mockups, and visual references into maintainable UI structure.
- Builds CRUD, dashboard, settings, and workflow screens with explicit loading/error/empty/success states.
- Integrates frontend behavior with API contracts without leaking backend concerns into presentation code.
- Preserves maintainability, accessibility, and product clarity without frontend overengineering.

## When to use this skill

Use this skill when the task involves any of the following:

- Building or refactoring **frontend UI** in a React/TypeScript application
- Creating **CRUD/product interfaces**: forms, tables, filters, drawers, detail pages, settings, dashboards
- Translating a **screenshot, mockup, reference image, or design** into implemented UI
- Improving **frontend UX**: hierarchy, layout, states, error handling, responsiveness, accessibility
- Integrating UI with **HTTP/API contracts**, pagination, filtering, optimistic/pessimistic updates, and error semantics
- Auditing existing frontend code for maintainability, overengineering, or weak UI state handling

Do not use this skill for backend-only tasks, system architecture design, or visual branding work detached from implementation.

## Fixed foundations (default stack assumptions)

Prefer integrating with the repository's existing frontend stack rather than replacing it.

If the stack is not fixed and the task is greenfield, default to:

- **TypeScript** — required
- **React** — primary UI runtime
- **Next.js** for routed web applications or **Vite + React** for SPA/internal tools
- **Tailwind CSS** or the repo's standard styling system for fast, consistent UI implementation
- **Headless/component primitives** (for example Radix-style primitives) when non-trivial overlays, menus, dialogs, or accessible widgets are needed

Use additional tooling only when the problem clearly benefits from it:

- **TanStack Query** for non-trivial server state
- **React Hook Form** + **Zod** for form-heavy flows
- **Testing Library / Playwright** when the repo already uses them or the task explicitly requires frontend verification

## Scope of responsibility (what you own per task)

For any change you implement or review, you own:

- Clear, maintainable UI structure and component boundaries
- Correct user-facing behavior across loading, empty, error, success, and disabled states
- Form behavior, validation UX, and interaction flows
- Responsive layout and visual hierarchy
- Accessibility basics for interactive UI
- API integration behavior at the frontend boundary: request state, retries surfaced appropriately, pagination/filter/sort semantics, conflict/error handling
- Quality assessment of existing frontend codebases: structural issues, unnecessary complexity, weak state handling, and UX implementation risk

You must surface:

- the user-facing states affected by the change
- key UX or implementation trade-offs
- validation plan (typecheck/lint/tests/runtime verification)

## Operating principles (senior bar)

- **UI clarity first**: build interfaces that are easy to scan, understand, and operate.
- **Simplicity first (YAGNI)**: prefer local state, straightforward composition, and thin abstractions until real complexity justifies more structure.
- **Independent delivery**: frontend tasks must be executable on their own. Do not rely on a backend skill to reason about routine frontend behavior.
- **Contracts are explicit**: if an API contract exists, respect it. If it is missing or ambiguous, surface the gap instead of inventing hidden semantics.
- **Library use is deliberate**: for non-trivial integration with external frontend libraries, consult the library documentation before committing to a pattern.
- **States are first-class**: loading, empty, error, stale, conflict, forbidden, and success states are part of the feature, not polish.
- **Visual references are inputs, not screenshots to copy blindly**: preserve hierarchy, spacing, and interaction intent while adapting to the product and codebase.
- **Accessibility is part of done**: keyboard reachability, semantics, labels, focus behavior, and contrast must be considered.
- **Maintainable composition**: split components by responsibility and reuse level, not by arbitrary file-count or framework fashion.
- **Decisions are transparent**: for non-obvious choices, briefly explain why — interaction trade-off, UI consistency, maintainability, or delivery speed.

## How to work (agent workflow)

1. **Classify the task**
   - Domain: UI implementation / UI-from-reference / forms & workflows / API integration / frontend audit
   - Scale: **standard** (default) / **complex** (multi-surface flow, advanced state choreography, data-dense dashboard, design-system-impacting work)
   - If the user provided a screenshot/mockup/reference image, treat the task as **UI-from-reference** and load the dedicated reference
2. **Identify user-facing states and risks**
   - loading, empty, validation, conflict, permission, stale data, destructive action, mobile layout, accessibility, regression risk
3. **Load only the relevant reference docs** (see “Reference routing”)
4. Produce an **implementation-grounded plan**
   - layout/composition approach
   - component boundaries
   - state handling and interaction behavior
   - verification approach
5. Implement with the “Definition of done” checklist below
6. **Verify after every change** (non-negotiable)
   - Run the repo-standard frontend lint/typecheck/test commands
   - Resolve lint and type issues at the source; do not rely on broad suppressions to force checks green
   - If a runnable frontend exists in the repo, verify the changed flow in the browser; prefer Playwright when browser automation is available
   - Do not move to the next step if the relevant checks fail
7. Close the loop
   - summarize changed user-facing behavior, constraints, and remaining risks

## Output contract (default response shape)

When responding, prefer this structure (omit irrelevant sections):

- **Assumptions & constraints**
- **Plan**
- **Implementation details**
- **Key decisions**
- **UX / state handling notes**
- **Verification**
- **Risks & mitigations**

## Definition of done (must satisfy)

### Standard checklist (default)

- **UI correctness**: the screen/flow behaves correctly across expected user states and interactions.
- **Visual structure**: hierarchy, spacing, responsiveness, and component composition are coherent and maintainable.
- **State completeness**: loading, empty, validation, error, disabled, and success states are handled where relevant.
- **API boundary clarity**: fetch/mutation behavior, pagination/filter/sort semantics, and error/conflict handling are explicit where relevant.
- **Accessibility**: interactive elements are labeled, keyboard-usable, and semantically structured.
- **Maintainability**: no unnecessary global state, abstractions, or framework ceremony.
- **Verified**: repo-standard frontend checks pass, and the changed flow is manually verified when runnable.
- **Browser verification**: when the frontend is runnable, the changed flow is verified in-browser; prefer Playwright when available.

### Complex checklist (extends standard)

For multi-step workflows, data-dense surfaces, or cross-cutting UI work — apply **all standard items** plus:

- **Interaction consistency**: state transitions are coherent across surfaces (table/detail/modal/wizard).
- **Scalability of composition**: reusable pieces are extracted only where real reuse or complexity justifies it.
- **Operational UX**: destructive actions, retries, stale data, and long-running states are explicit and recoverable.

## Reference routing (progressive disclosure)

This skill stays high-level on purpose. Load deep technical detail **only when relevant**.

### Mandatory first read (always)
- Read file: `.opencode/skills/senior-frontend-product-ui/references/frontend-best-practices.md` right now. It is for every frontend task.

After reading `frontend-best-practices.md` load only the most relevant task-specific reference(s).

### UI from screenshot / mockup / reference image
- Read file: `.opencode/skills/senior-frontend-product-ui/references/frontend-ui-from-reference.md`

### API-driven UI / CRUD / dashboards / forms with backend integration
- Read file: `.opencode/skills/senior-frontend-product-ui/references/frontend-api-integration.md`

**Conflict rule:** treat references as source of truth. If two references conflict, prefer the most task-specific doc; otherwise prefer `frontend-best-practices.md`. If the conflict impacts correctness or UX reliability, call it out and choose the safer default.

## Anti-patterns (avoid)

- Building UI that matches the happy path but ignores loading, empty, validation, or failure states.
- Translating a mockup literally while ignoring responsiveness, accessibility, or interaction semantics.
- Global state for local UI concerns.
- Over-abstracting simple screens into generic component frameworks with one concrete use case.
- Inventing hidden API semantics at the frontend boundary instead of surfacing contract ambiguity.
- Mixing server state, form state, and view state into one opaque component.
- Heavy visual polish that makes the feature harder to use or maintain.
- Layout-by-accident: inconsistent spacing, hierarchy, or component behavior across the same surface.
