# Example 02 — Staggered list entrance

## Input

A 1.2s gif showing a sidebar nav of 6 items appearing on page load. Items fade in and slide up slightly, one after another from top to bottom. Each item starts before the previous one has finished.

## Output

```
# Motion spec: sidebar nav staggered entrance

## Source
- Artifact: sidebar-load.gif (1.2s, 30fps coalesced frames)
- Frames inspected: 36
- Estimated framerate: 30

## Trigger
- Event: mount (page load — items appear when route renders)
- Element receiving event: sidebar container

## Elements animated
1. Nav items × 6 — each list item in the sidebar

## States
- A (initial): each item invisible and offset 8px below its final position
- B (final): each item visible at its final position

## Per-property motion
### Nav item (applied identically to all 6, with staggered delay)
- `opacity`: 0 → 1, duration 280ms, easing ease-out, delay (see choreography)
- `transform.translateY`: 8px → 0, duration 280ms, easing ease-out, delay (see choreography)

## Choreography
- Stagger: 60ms between items, direction top → bottom
  - Item 1: delay 0ms
  - Item 2: delay 60ms
  - Item 3: delay 120ms
  - Item 4: delay 180ms
  - Item 5: delay 240ms
  - Item 6: delay 300ms
- Overlap: each item starts 60ms after previous, finishes 280ms later — so item N+1 starts while item N is still mid-animation (~21% in). This is what makes it feel like a wave rather than a queue.
- Transform origin: default

## Edge cases
- Reduced motion: skip translateY, keep opacity, set stagger to 0 (all fade in together at 120ms)
- Interrupt: not applicable — entrance only fires once on mount
- Exit: not visible in artifact — recommend no animation on unmount (items disappear with route)
- Loop: one-shot
- Performance flags: none — only `opacity` and `transform`

## Recommended implementation
- Library: framer-motion
- Reason: `staggerChildren: 0.06` on parent variant matches the spec exactly; children share one variant

## Confidence
- High on: stagger interval, properties, duration
- Medium on: exact translateY distance (could be 6–10px; 8px is a reasonable default and within frame resolution)
```
