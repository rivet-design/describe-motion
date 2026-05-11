# Example 04 — Website URL: "How is the fade-in on yafafits.com done?"

This example shows the URL-input flow. The user gave a live URL and a verbal hint instead of a media file. Two sub-examples: the happy path (page fetches cleanly) and the fallback (fetch blocked).

---

## Input

> "How would I describe the fade-in animation on https://www.yafafits.com/?"

## Happy path (WebFetch succeeds)

### What the agent does

1. Calls `WebFetch` on the URL with a prompt asking for: framework detection, `@keyframes` definitions, `transition` declarations, inline `transform`/`opacity` styles, and animation library imports.
2. Matches "fade-in" against elements with initial `opacity: 0` plus a transition or class toggle.
3. If multiple candidates, asks one clarifying question. Otherwise produces the spec directly from source.

### Hypothetical fetch result (illustrative)

```
Framework: Framer (data-framer-component-type on most elements, framer.js bundle)
Imports: motion utilities (Framer-built, not raw framer-motion)
Inline style on hero <section>:
  style="opacity:0;transform:translateY(20px);transition:opacity 0.6s cubic-bezier(0.44,0,0.56,1) 0.1s, transform 0.6s cubic-bezier(0.44,0,0.56,1) 0.1s"
Attribute: data-framer-appear-id="hero-section"
JS: IntersectionObserver toggles a class on enter, which sets opacity:1 and transform:translateY(0)
```

### Output

```
# Motion spec: yafafits.com hero fade-in

## Source
- Artifact: https://www.yafafits.com/
- Input type: URL
- Frames inspected: n/a — source-read
- Framework detected: Framer

## Source code findings
- Inline style on hero section: `opacity:0; transform:translateY(20px); transition: opacity 0.6s cubic-bezier(0.44,0,0.56,1) 0.1s, transform 0.6s cubic-bezier(0.44,0,0.56,1) 0.1s`
- Trigger mechanism: IntersectionObserver toggles a class on viewport entry, which sets opacity:1 and transform:translateY(0)
- Element targeted: hero section (data-framer-appear-id="hero-section")
- Animation library imports: Framer's built-in motion utilities (no external framer-motion bundle)

## Trigger
- Event: intersection (element enters viewport)
- Element receiving event: hero section

## Elements animated
1. Hero section — the top section of the page (headline + image)

## States
- A (initial): opacity 0, translateY 20px
- B (final): opacity 1, translateY 0

## Per-property motion
### Hero section
- `opacity`: 0 → 1, duration 600ms, easing cubic-bezier(0.44, 0, 0.56, 1), delay 100ms
- `transform.translateY`: 20px → 0, duration 600ms, easing cubic-bezier(0.44, 0, 0.56, 1), delay 100ms

## Choreography
- Parallel: opacity and translateY share identical timing — they animate as one motion
- Transform origin: default (not relevant for translation)

## Edge cases
- Reduced motion: not visible in source — recommend adding `@media (prefers-reduced-motion: reduce)` that sets transition-duration to 0ms and final state immediately
- Interrupt: not applicable — one-shot on viewport entry
- Exit: not visible — element does not animate out when scrolled past
- Loop: one-shot (IntersectionObserver typically unobserves after first entry)
- Performance flags: none — `opacity` and `transform` only

## Recommended implementation
- Library: CSS + IntersectionObserver, OR framer-motion's `whileInView`
- Reason: the source already uses the simplest possible pattern (CSS transition + class toggle). Match it. If using React, framer-motion's `whileInView={{ opacity: 1, y: 0 }}` with `initial={{ opacity: 0, y: 20 }}` and `transition={{ duration: 0.6, ease: [0.44, 0, 0.56, 1], delay: 0.1 }}` is a 1:1 translation.

## Confidence
- High on: every numeric value (duration, easing, delay, translateY distance) — read directly from source
- High on: trigger mechanism (IntersectionObserver pattern visible)
- Medium on: whether the entire hero is one element or whether sub-children also stagger (would need to inspect children for individual delays)
```

---

## Fallback path (WebFetch returns 403, as actually happens for yafafits.com)

### What the agent does

The fetch failed. Instead of inventing values, ask the user for the source directly.

### Output

```
I couldn't fetch yafafits.com (403 Forbidden — the site blocks automated requests). To describe the fade-in precisely, I need one of:

**Option A — Paste the source for the animating element (fastest, most accurate):**
1. Open the site in Chrome.
2. Right-click the element that fades in → Inspect.
3. In Elements panel: right-click the highlighted node → Copy → Copy outerHTML. Paste that here.
4. In the Styles panel of the same node, copy any rules containing `transition`, `transform`, `opacity`, or `animation`. Paste those too.

**Option B — Use Chrome's Animations panel (gives exact timing visually):**
1. DevTools → ⋮ menu → More tools → Animations.
2. Refresh the page; the panel records every animation that plays.
3. Click the animation bar for the fade-in. The panel shows duration and easing curve.
4. Screenshot it and share.

**Option C — Screen recording fallback:**
Record the fade-in at 60fps and share the file. I'll fall back to frame-counting, but values will be ~estimated.

Once you share any of the above, I'll produce the full spec.
```

### Why this matters

The skill's core rule is *numbers over adjectives*. When the source is unreachable and frames are unavailable, asking is correct — fabricating cubic-bezier values from a verbal description would produce confident-looking but wrong specs, which is worse than no spec.
