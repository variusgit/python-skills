---
name: frontend-best-practices
description: Production frontend guidance for TypeScript React applications — component boundaries, UI state handling, accessibility, responsiveness, maintainability, and proportional frontend architecture. Use for any frontend implementation or refactor.
---

# Frontend Best Practices

Production frontend guidance for TypeScript React applications that prioritize clear UI, explicit state handling, maintainability, and pragmatic delivery.

## When to use

- Any frontend implementation or refactor
- Component decomposition and state organization
- Forms, tables, dashboards, settings pages, and workflows
- Frontend audits for maintainability or overengineering

## Architecture boundaries

- Separate:
  - **presentation**: layout, markup, component composition
  - **server state**: fetched/mutated remote data
  - **form state**: controlled or validated user input
  - **view state**: local toggles, tabs, dialogs, expanded rows
- Keep domain or transport details out of low-level presentational components.
- Prefer container/presenter separation only when it improves clarity. Do not create ceremony for simple components.

## Component design

- Split components by responsibility, not by arbitrary size targets.
- Extract components when one of these becomes true:
  - the same UI pattern is reused
  - the parent mixes too many concerns
  - a subpart has its own states or interaction rules
- Keep one-off layout glue inline if extraction adds indirection without value.
- Prefer explicit props over hidden implicit context unless the state is truly cross-cutting.

## State handling

- Treat these states as normal design work, not edge polish:
  - loading
  - empty
  - validation error
  - request error
  - success confirmation
  - disabled / pending
  - destructive confirmation
- Local UI state stays local by default.
- Use global state only for truly shared cross-screen concerns.
- Do not put server data in client state stores just because it is fetched.

## Forms and workflows

- Forms should surface:
  - required vs optional fields
  - validation timing
  - pending submit state
  - field-level vs form-level errors
  - success state or post-submit navigation
- For multi-step flows, define:
  - step transitions
  - back/forward behavior
  - save/discard semantics
  - validation boundaries per step
- Do not make users guess whether an action succeeded, failed, or is still running.

## Layout and visual hierarchy

- Use clear hierarchy:
  - page title and primary intent
  - section grouping
  - dominant primary action
  - secondary actions visually subordinate
- Be consistent with spacing, density, and alignment within the same surface.
- Prefer predictable layout primitives: stack, grid, split panes, cards, table + detail, tabs, drawer, modal.
- Responsive behavior should be intentional: what collapses, wraps, hides, or changes order on smaller screens.

## Accessibility baseline

- Use semantic elements first (`button`, `label`, `table`, `nav`, `dialog` patterns).
- Ensure keyboard interaction works for all primary flows.
- Inputs must have labels and error messaging tied to the field.
- Focus behavior must be intentional after dialogs, navigation, and destructive actions.
- Do not encode critical meaning by color alone.

## Type safety

- Use TypeScript throughout the UI.
- Prefer explicit prop types and narrow unions for stateful UI.
- Model UI states clearly when behavior differs materially.
- Avoid `any`; if an external boundary is weakly typed, isolate and narrow it near the boundary.

## Simplicity and proportionality

- Start with:
  - local state
  - direct component composition
  - a thin typed API layer
  - straightforward CSS/styling primitives
- Extract patterns, hooks, or shared primitives only after real repetition or interaction complexity appears.
- Do not introduce frontend platform machinery for a small CRUD screen.

## Verification

- Run the repo-standard lint/typecheck/test commands.
- If the app is runnable, verify the changed screen or flow directly in the browser.
- Check the actual UI for:
  - spacing/hierarchy regressions
  - focus/keyboard behavior
  - empty/loading/error states
  - responsive breakpoints where relevant

## Failure modes

- Components that look fine but have no meaningful loading or error behavior
- Hidden state coupling between modal/table/form/detail surfaces
- Global state introduced for local concerns
- Over-generic components that are harder to use than the one screen they were extracted from
- Untyped API responses flowing through the UI unchecked
- Visual polish masking unclear hierarchy or broken interactions
