# Roadmap

## Done

- Versioned routing with launcher page
- v1 simple prototypes (stable)
- v2 branching prototypes (beta)
- Content review experimental pass on v1
- PDF summarisation before generation
- Design history pages at /design-history
- GOV.UK rebrand applied to v2 (v1 pending)
- GDS Tetris loading screen designed and approved, not yet built in

## In progress

- Confirm v2 rebrand works, then apply to v1

## v3 — Editor mode (planned)

- Toolbar baked into generated prototype layout above GOV.UK header
- White bar, blue bottom border, service name and last updated on left, Editor mode button on right
- Password protected editor mode
- Six commands: review, improve content, apply sensitive service rules, simplify to reading age 9, add a question, validate
- Command runs Tetris updating screen, rebuilds prototype, reloads on completion
- Refresh-safe via status.json in prototype repo
- spec.json persisted in prototype repo for iterative updates
- Simple shared password auth, no user accounts

## v4 and beyond

- Research synthesis mode
- Iteration support (regenerate individual pages)
- Task list pattern
- Additional GDS patterns: date inputs, checkboxes, address lookup, interruption cards, content pages

## Key design decisions

- No iframe toolbar — Render headers block embedding, breaks on Safari and mobile
- No Netlify — GDS kit requires Node.js, Netlify is static only
- Versioned routing — keeps stable versions safe while testing new ones
- Second Claude call for content review — focused call produces better output than one large call
- PDF summarisation — brief-aware extraction better than raw truncation
