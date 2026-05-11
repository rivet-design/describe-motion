# Example 03 — Spring button hover

## Input

A short clip of a primary CTA button. On hover, the button scales up. At one frame, the button is visibly slightly larger than its resting hover size — it overshoots and settles. On unhover, it returns smoothly to its idle size without overshooting.

## Output

```
# Motion spec: primary CTA hover

## Source
- Artifact: cta-hover.webm (clip showing hover-in and hover-out)
- Frames inspected: 48 (60fps, both directions)
- Estimated framerate: 60

## Trigger
- Event: hover (pointerenter / pointerleave)
- Element receiving event: <button class="primary-cta">

## Elements animated
1. CTA button — single element

## States
- A (idle): scale 1
- B (hovered): scale 1.04 (settled)
- Overshoot frame observed at ~scale 1.06 between A and B

## Per-property motion
### CTA button (hover-in)
- `transform.scale`: 1 → 1.04 (overshoots to ~1.06 then settles), spring physics
  - Spring params (estimated): stiffness 300, damping 18, mass 1
  - Equivalent timing reference: ~350ms total motion, with overshoot peak at ~180ms

### CTA button (hover-out)
- `transform.scale`: 1.04 → 1, duration 200ms, easing ease-out, no overshoot

## Choreography
- Single property, single element — no choreography
- Transform origin: center center

## Edge cases
- Reduced motion: replace with `background-color` change only; no scale
- Interrupt: spring physics handle this natively — if user unhovers mid-bounce, spring continues from current value toward idle. If implementing with CSS, use `transition: transform 200ms ease-out` on both states so it interpolates from current value.
- Exit (unhover): asymmetric — no overshoot on exit. This is intentional; overshoot on both directions feels frantic.
- Loop: on-demand (every hover)
- Performance flags: none — `transform` only

## Recommended implementation
- Library: framer-motion or react-spring
- Reason: native spring physics; overshoot is the defining feature and is awkward to express in CSS. Framer's `whileHover={{ scale: 1.04 }}` with `transition={{ type: "spring", stiffness: 300, damping: 18 }}` captures the spec directly.

## Confidence
- High on: presence of overshoot, asymmetric exit, scale endpoints
- Medium on: spring stiffness/damping exact values — these are estimated from overshoot magnitude and settle time, may need tuning
```
