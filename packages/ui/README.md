# `packages/ui`

Shared, reusable React UI components for `apps/web` — not yet created.
Intended to hold presentation-layer building blocks that many screens need,
so the app doesn't end up with five slightly different button components.

## Candidates for this package (once built)

- Envelope budget card, category badge/pill, streak flame icon, Financial
  Health Score gauge, color-nudge progress bar (green → yellow → red).
- Base primitives (Button, Input, Modal, Toast) if not fully delegated to a
  headless library.

## A note on mobile reuse

React Native cannot directly consume web React components (different
rendering targets), so "shared UI" across web and mobile realistically
means **shared design tokens and shared component *logic*** (e.g., a
`useEnvelopeProgress()` hook), not shared JSX. If Phase 10 (mobile) wants
visual consistency, evaluate a cross-platform styling approach (e.g.,
NativeWind) at that time rather than guessing now — see
`docs/architecture.md`.

## Suggested stack

Tailwind CSS utility classes (consistent with `docs/architecture.md`'s
frontend choice) plus [shadcn/ui](https://ui.shadcn.com/)-style
copy-in components as a starting point, customized rather than pulled in
as an opaque dependency.
