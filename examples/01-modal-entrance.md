# Example 01 — Modal entrance

## Input

A 1.5s screen recording (60fps) of a settings modal appearing after clicking a "Preferences" button. The backdrop fades in. The modal panel grows slightly and fades in, settling without overshoot. The modal's content (a header and three rows) is already laid out when the panel appears — no stagger inside.

## Output

```
# Motion spec: settings modal entrance

## Source
- Artifact: settings-modal.mov (1.5s screen recording)
- Frames inspected: 90 (60fps, full clip)
- Estimated framerate: 60

## Trigger
- Event: click
- Element receiving event: "Preferences" button in top nav

## Elements animated
1. Backdrop overlay — dims the page behind the modal
2. Modal panel — the container card holding the settings UI

## States
- A (initial): backdrop hidden, modal not in DOM
- B (final): backdrop at 50% black, modal centered and at rest

## Per-property motion
### Backdrop overlay
- `opacity`: 0 → 1, duration 200ms, easing ease-out, delay 0ms
- `background-color`: stays rgba(0,0,0,0.5) throughout (only opacity animates)

### Modal panel
- `opacity`: 0 → 1, duration 220ms, easing ease-out, delay 0ms
- `transform.scale`: 0.96 → 1, duration 220ms, easing ease-out, delay 0ms

## Choreography
- Parallel: backdrop and modal start at t=0
- Transform origin: center center (default — modal grows symmetrically)

## Edge cases
- Reduced motion: skip scale, keep opacity, shorten to 80ms
- Interrupt: not visible in artifact — recommend snap-to-final if user re-clicks during entrance
- Exit: not visible in artifact — recommend symmetric reverse (scale 1 → 0.96, opacity 1 → 0, 180ms ease-in)
- Loop: one-shot
- Performance flags: none — both `opacity` and `transform` are compositor-friendly

## Recommended implementation
- Library: framer-motion
- Reason: `AnimatePresence` handles exit cleanly; `initial`/`animate`/`exit` props match the spec one-to-one

## Confidence
- High on: durations, easing, properties animated
- Medium on: exact scale start value (0.96 vs 0.95 — within 1 frame of difference)
- Estimates flagged: exit behavior (not shown in artifact)
```
