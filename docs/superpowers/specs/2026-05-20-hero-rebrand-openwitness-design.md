# Hero Section Rebrand: OpenWitness.org

**Date:** 2026-05-20
**Scope:** Landing page hero section only (`crates/aqua-timestamp/src/landing.rs`, lines 577-592)

## Problem

The current hero uses unearned superlatives ("Fastest multi-chain publishing", "Most trusted cross-jurisdiction anchoring") that state facts the service cannot yet back up. The branding references "Aqua Timestamp Service", which is being replaced by the public-facing name OpenWitness.org.

## Design

### Content

| Element | Value |
|---------|-------|
| Eyebrow | `OpenWitness.org` |
| Headline (h1) | `Your free anchor for trusted time` |
| Body | "We timestamp your data across blockchains and qualified authorities, funded by institutions so individuals never pay. The service itself runs on its own protocol: every operational decision is auditable." |
| Pill 1 | `Free by design` |
| Pill 2 | `Accountable by protocol` |
| Pill 3 | `No coin, no account` |

### Design principles behind the copy

1. **Honest utility:** Lead with what the service does today, frame the bigger vision as a stated goal. Nothing is claimed that cannot be verified on this page.
2. **Accountable institution as lead value:** The structural differentiator (Aqua-on-Aqua self-auditing) is in the body copy, not buried in a footer.
3. **Design-principle pills:** Each pill is a structural commitment that is true now and remains true as the service scales. No aspirational metrics.

### What changes

- Eyebrow: "Aqua Timestamp Service" becomes "OpenWitness.org"
- Headline: shorter, personal ("Your"), no drama
- Body: two sentences covering what/how/why instead of mood-setting prose
- Pills: 2 superlatives become 3 design principles
- Pill colors: add a third color for the new pill (existing: blue, green; new: amber/orange for the third)

### What does NOT change

- Page structure, sections below the hero, ORL badge, SSE live stats, funding goals, footer
- CSS class names (`.hero`, `.hero-eyebrow`, `.hero-body`, `.value-pills`, `.value-pill`)
- No new JavaScript or interactivity

## Implementation

Single file edit in `crates/aqua-timestamp/src/landing.rs`, lines 577-592. Replace the hero section HTML string content. Add one CSS rule for the third pill color.
