---
name: frontend-api-integration
description: Build API-driven frontend UI for CRUD, dashboards, filters, pagination, and mutations with explicit contract handling and reliable user-facing states. Use when frontend work depends on backend/API behavior.
---

# Frontend API Integration

Use this guide for frontend features that depend on APIs, remote data, CRUD flows, or backend-driven state transitions.

## Contract-first behavior

- Respect existing API contracts; do not silently invent semantics.
- If contract details are missing, surface the ambiguity:
  - pagination behavior
  - filter semantics
  - sort semantics
  - conflict vs validation vs permission errors
  - mutation success shape
- Keep API-specific data shaping close to the boundary, not spread across presentational components.

## Read flows

- For list surfaces, define explicitly:
  - initial loading
  - empty state
  - error state
  - refresh/reload behavior
  - pagination or infinite-scroll behavior
  - filter and sort interactions
- Do not ship large data surfaces with unclear “what happens next” behavior after filter or pagination changes.

## Mutation flows

- Every mutation should define:
  - pending state
  - success feedback
  - validation error behavior
  - general error behavior
  - destructive action confirmation when needed
- Be explicit about optimistic vs pessimistic updates.
- If stale data after mutation would confuse users, refetch or reconcile intentionally.

## CRUD screens

For resource-oriented UI, think in lifecycle, not isolated components:

- **List**: table/cards, search, filters, pagination, empty state
- **Read one**: detail structure, metadata, related actions
- **Create**: defaults, validation timing, success redirect/confirmation
- **Update**: dirty-state handling, conflict behavior, save/discard semantics
- **Delete**: confirmation, repeated delete behavior, post-delete navigation

If a resource intentionally omits one of these operations, make the UI reflect that clearly rather than leaving dead affordances.

## Error semantics

- Differentiate:
  - validation errors
  - permission errors
  - not found
  - conflict
  - transient/server failure
- The UI should react differently when the semantics differ.
- Do not flatten all failures into the same generic toast if the user needs to act differently.

## Frontend/backend boundary

- API clients/hooks may understand transport details.
- Domain-specific presentational components should understand user meaning, not transport mechanics.
- Avoid burying retries, cache invalidation, or mutation behavior deep inside unrelated UI components.

## Verification

- Verify that UI state matches backend semantics after:
  - successful create/update/delete
  - validation failure
  - permission or conflict error
  - reload or refetch
- Check whether user-facing affordances still make sense when data is stale, partial, or empty.

## Failure modes

- UI controls present operations the API does not actually support
- Loading spinners with no empty/error recovery path
- Mutations succeed but the screen remains stale or contradictory
- Filters and pagination reset unexpectedly without clear intent
- Contract mismatches handled by ad-hoc frontend workarounds instead of explicit boundary code
