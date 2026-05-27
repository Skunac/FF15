# ADR 0003: Twig + Live Components over a separate SPA

**Status:** Accepted
**Date:** 2026-05-27

## Context

FF15 requires interactive UI: filter bars on the tier list, an opponent
selector on the build optimizer, sortable tables, detail drawers. Three
broad architectural options exist:

1. Pure server-rendered Twig with full page reloads.
2. Server-rendered Twig augmented with Symfony UX (Live Components, Stimulus,
   Turbo) for partial updates and light interactivity.
3. Symfony as a JSON API consumed by a separate SPA (React, Vue, Svelte).

The project is a portfolio piece intended to showcase Symfony depth. The
interactivity requirements are real but bounded — filters, selectors, sortable
tables, no real-time features.

## Decision

We will use option 2: server-rendered Twig with Symfony UX Live Components,
Stimulus, and Turbo for interactivity.

Live Components will handle filter bars, selectors, and any component whose
state lives on the server. Stimulus will handle small client-side concerns
(chart rendering, clipboard, sortable columns). Turbo will handle partial
page navigation and Turbo Frames for detail drawers.

## Consequences

**Positive:**

- One codebase, one language, one deployment artifact. No build pipeline for a
  separate frontend, no API contract to keep in sync.
- Server-side authority over rendered state — security, validation, and
  authorization all live in one place.
- Live Components provide AJAX-driven re-rendering without writing AJAX code.
  Interactivity is declared in Twig and PHP.
- Demonstrates the modern Symfony UX ecosystem, which is a relevant signal for
  a Symfony-focused portfolio.
- Faster initial page loads than an SPA — no JavaScript bundle to parse before
  content is visible.

**Negative:**

- No client-side state management. Interactions that benefit from offline
  capability or zero-latency local updates are not supported. None of FF15's
  features require this.
- Live Component re-renders involve a server round-trip. For very chatty
  interactions this would matter; for filter selections at human pace it does
  not.
- Real-time push (server-initiated updates without user action) is not
  supported by Live Components alone. FF15 has no such requirement — see
  ADR-0008 if added later.

## Rejected alternatives

**Pure server-rendered Twig with full page reloads.** Rejected because every
filter change causing a full reload would feel dated and degrade the perceived
quality of the project.

**Separate SPA frontend (React/Vue) consuming a Symfony JSON API.** Rejected
because:

- It would double the surface area (two codebases, two build pipelines, an API
  contract).
- The interactivity requirements do not justify the complexity.
- The portfolio goal is to showcase Symfony depth, not frontend framework
  proficiency. A separate SPA would dilute the signal.
- An API consumer doesn't exist, so API Platform (or similar) would be added
  to serve an SPA that has no other reason to exist — circular justification.

## Future review

If the project later gains requirements that genuinely need an SPA (offline
support, complex client-side state, mobile companion app), a separate ADR
should propose an API layer (see ADR-0005 on API Platform deferral). Until
then, this stack remains appropriate.