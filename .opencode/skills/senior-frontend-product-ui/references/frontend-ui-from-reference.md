---
name: frontend-ui-from-reference
description: Translate screenshots, mockups, and visual references into implemented frontend UI with accurate hierarchy, spacing, and interaction behavior. Use when the user provides a screenshot, design image, mockup, or asks to recreate a UI.
---

# UI From Reference

Use this guide when the user gives a screenshot, mockup, design reference, or asks to recreate an existing interface.

## Goal

Reproduce the **intent** of the UI accurately:

- layout
- spacing rhythm
- visual hierarchy
- component breakdown
- interaction cues
- responsive adaptation

Do not copy pixels blindly if that would damage semantics, responsiveness, or maintainability.

## Workflow

1. **Read the reference before coding**
   - Identify page structure first: header, filters, content area, actions, side panels, dialogs, table/cards, footer.
   - Identify repeated patterns: rows, cards, stat blocks, form groups, tabs.
   - Identify dominant actions and information hierarchy.

2. **Decompose into implementation units**
   - page shell
   - section containers
   - reusable primitives
   - domain-specific widgets
   - interaction surfaces (drawer, modal, dropdown, tabs)

3. **Identify states the image does not show**
   - loading
   - empty
   - error
   - hover / focus / selected
   - disabled / pending
   - validation states

4. **Implement hierarchy before decoration**
   - get spacing and composition right first
   - then typography emphasis
   - then interaction states
   - then small visual polish

5. **Adapt intelligently**
   - preserve the reference's structure and feel
   - adapt naming, text, and behavior to the actual product
   - align with the repository's component/styling conventions

## What to preserve

- section ordering
- relative spacing density
- information grouping
- primary vs secondary action emphasis
- dominant interaction pattern (table-first, card-first, form-first, wizard-first)

## What may change

- exact copy text
- literal colors when the product theme differs
- icon set
- low-value decorative details
- layout details required for responsiveness or accessibility

## Good translation behavior

- If the reference suggests a table with inline actions, preserve that interaction model.
- If the reference is clearly a detail page with side metadata, keep that structure.
- If the reference is visually dense, preserve hierarchy with consistent spacing rather than collapsing everything into generic cards.
- If the reference is minimal, do not add extra chrome just to make it look “richer”.

## Common traps

- Recreating the rough look but losing the layout hierarchy
- Ignoring missing states because the image only shows one static frame
- Making the UI “cleaner” in a way that changes task flow
- Introducing unnecessary component abstraction before the structure is understood
- Matching desktop appearance while breaking mobile/tablet behavior

## Verification checklist

- The implemented screen reads in the same order as the reference
- Primary actions are visually obvious
- Spacing rhythm is coherent
- Repeated elements are consistent
- Hidden states have been designed, not left accidental
- The UI still works without the reference image next to it
