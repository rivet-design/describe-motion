---
name: describe-motion
description: Translate a motion artifact (video, gif, screen recording, animated screenshot sequence, or a live website URL) into a precise implementation prompt for a coding agent. Use whenever the user provides motion — either a media file or a URL with a verbal description ("how is the fade-in on this site done", "describe the hover on yafafits.com", "match the scroll animation at <url>") — and wants to recreate it in code. Also fires on phrases like "implement this animation", "build this interaction", "match this motion", "how would I code this", "translate this video to code". Produces a structured prompt covering elements, triggers, states, per-property timing & easing, choreography, and edge cases.
---

# describe-motion

You translate a motion artifact into an implementation prompt. Motion is the part of UI that visual descriptions usually fail to capture — duration, easing, stagger, overshoot, interrupt behavior, and choreography are invisible in a static screenshot but make or break the feel. This skill exists to extract those properties precisely so a downstream coding agent can implement the interaction without guessing.

The output is a prompt, not code. You hand it to whoever is going to build the thing.

## When to use this

Use whenever the user gives you motion and wants it implemented. Three input shapes are supported:

1. **Media file** — video, gif, screen recording, or frame sequence.
2. **Live URL + verbal description** — the user points at a site and describes the interaction they want to replicate (e.g., "the fade-in on yafafits.com").
3. **DOM/CSS snippet** — the user pastes the relevant markup and wants it described or improved.

If the input is a static screenshot with no motion and no URL, this skill is the wrong tool — use a layout/UI description skill instead. If the user wants you to build the thing directly, run this skill first and then implement from the output.

## How to read the artifact

Before describing anything, get frames you can actually inspect. Guessing from memory of a single playback is the most common failure mode.

- **Video (.mov/.mp4/.webm):** extract frames at the boundaries of each motion phase. For a known framerate, step through with `ffmpeg -i input.mp4 -vf "fps=30" frames/%04d.png` and look at the frames around state changes. For ad-hoc inspection, `ffmpeg -i input.mp4 -ss <timestamp> -frames:v 1 frame.png` pulls a single frame.
- **GIF:** `magick input.gif -coalesce frames/%04d.png` (ImageMagick) gives you per-frame stills with cumulative state.
- **Screen recording at unknown fps:** assume 30 or 60fps; count frames between the first visible movement and settle, multiply by frame duration (33ms at 30fps, 16ms at 60fps).
- **Frame sequence already provided:** look at every frame, not just first/last. The middle frames reveal the easing.

If you cannot extract frames (no tooling, web-only context), say so explicitly in the output and mark timing/easing fields as `~estimated`.

## How to read a URL input

When the user gives you a URL plus a verbal description ("how is the fade-in done on this site"), the source code is the ground truth. Prefer reading it over guessing from visual inspection — the actual CSS/JS gives you exact duration, easing, and property values without any frame counting.

### Step 1 — Fetch the page

Use `WebFetch` (or an equivalent tool) to pull the page HTML and any inline CSS. Ask it specifically for:

- The framework or builder (Framer, Webflow, Next.js, Squarespace, custom). Identify from meta tags, `data-*` attributes, script src patterns, and class naming.
- Every `@keyframes` definition (name + keyframe values).
- Every `transition:` declaration (property, duration, timing-function, delay).
- Inline `style="..."` attributes containing `transform`, `opacity`, or `transition`.
- Script tags importing animation libraries: `framer-motion`, `gsap`, `lottie`, `motion`, `react-spring`, `animejs`.

### Step 2 — Match the verbal description to a specific element

The user said "the fade-in" or "the hover" — narrow that to a selector. Use their wording as a filter:

- "Hero", "header" → the element near the top of the body, often `<section>` or `<header>`.
- "Card", "tile" → repeated children of a grid container.
- "Button", "CTA" → `<a>` or `<button>` elements with prominent styling.
- "On scroll" → look for `IntersectionObserver` JS, or Framer's `data-framer-appear-id` and `data-framer-appear-animation`.
- "On load", "fade-in" → look for elements with initial `opacity: 0` in inline styles, plus a transition or class-toggle that brings them to 1.

If multiple elements match, ask the user one clarifying question rather than guessing.

### Step 3 — Framework-specific hints

Different builders expose motion differently. Knowing the builder narrows where to look:

- **Framer-built sites** — look for `data-framer-component-type`, `data-framer-appear-id`, and inline styles with `transform`. Framer often inlines the animation as initial state + a transition triggered by an `IntersectionObserver`. Easing values are typically cubic-bezier strings in CSS variables like `--framer-transition-timing-function`.
- **Webflow** — `data-w-id` attributes link to interactions defined in a JSON blob loaded by `webflow.js`. The motion config is in that JSON; find the matching ID.
- **React + Framer Motion** — look for `motion.*` components in the JSX (if SSR'd source is readable) and inline `style` attributes that Framer Motion writes (`transform: translateY(...)`, `opacity: ...`). Bundle-minified `transition={...}` objects may still be visible as JSON literals.
- **GSAP** — look for `gsap.from(`, `gsap.to(`, `ScrollTrigger.create(`. GSAP's timeline calls usually survive minification by name.
- **CSS-only** — `@keyframes` plus `animation:` shorthand on an element. Easiest to read directly.

### Step 4 — Fallback when WebFetch is blocked or insufficient

Some sites block automated fetches (403/Cloudflare) or load animations entirely from JS bundles that resist parsing. When that happens, do not invent values. Instead, ask the user for one of:

- **A DOM snippet** — they right-click the animating element → Inspect → copy the outer HTML, plus the relevant rules from the Styles panel.
- **A network HAR or a paste of the CSS file** — for keyframes and transitions.
- **A screen recording** — fall back to the media-file path above and treat timing as estimated.
- **A devtools Animations panel screenshot** — Chrome's Animations panel shows duration and easing curves for any animation currently playing on the page.

State the fallback clearly: "I couldn't fetch the page (403). To get exact values, paste the DOM and Styles for the animating element, or share a screen recording."

### Step 5 — Cross-check against the verbal description

If the user said "subtle slide up and fade" but the source shows only `opacity` changing, trust the source — and note the discrepancy. Either the slide is happening via a different mechanism (e.g., parent container), or the user is misremembering. Surface it: "Source shows opacity-only; you described a slide. Was there a translateY you'd like me to add, or were you describing a different element?"

## The five-phase pass

Run these in order. Do not skip ahead — later phases depend on observations from earlier ones.

### Phase 1 — Elements & trigger

Identify what's moving and what causes it to move.

- **Elements:** which DOM nodes change between frames. Name them descriptively (e.g., "modal panel", "backdrop overlay", "list item", "submit button label").
- **Trigger:** what initiates the motion. Common triggers: `hover`, `click`, `focus`, `mount` (appears on render), `unmount` (disappears), `scroll`, `intersection` (enters viewport), `drag`, `route change`, `state change` (e.g., loading → success), `keypress`.
- **Direction:** is this an entrance, exit, hover, toggle, or continuous loop?

If the artifact only shows the "to" state, note that the trigger is inferred and flag uncertainty.

### Phase 2 — States

Enumerate the discrete states. Most interactions are A → B (entrance) or A → B → A (toggle). Some have intermediate states (A → B → C, e.g., button: idle → loading → success).

For each state, note the value of every property that will animate.

### Phase 3 — Per-property motion

For each animated property on each element, capture:

- **Property** — `transform.translateY`, `transform.scale`, `opacity`, `background-color`, `filter.blur`, `height`, `border-radius`, `clip-path`, etc. Prefer compositor-friendly properties (`transform`, `opacity`) when the choice is visible.
- **From → to** — concrete values. `opacity: 0 → 1`, `translateY: 8px → 0`, `scale: 0.96 → 1`.
- **Duration** — in milliseconds, measured from frame count. Round to common UI durations when within 1 frame: 150, 200, 250, 300, 400, 500ms.
- **Easing** — see the easing guide below. Identify by character, not by guessing a cubic-bezier from memory.
- **Delay** — does this property start at t=0 or later?

### Phase 4 — Choreography

How the per-property animations relate to each other across elements.

- **Parallel:** everything starts at t=0. Default and most common.
- **Sequence:** B starts after A finishes. Note the boundary.
- **Stagger:** N siblings animate with a fixed offset between each (e.g., list items fade in 50ms apart). Note the stagger interval and direction (first→last, last→first, center-out).
- **Overlap:** B starts before A finishes. Note the overlap duration — this is what makes motion feel coordinated rather than mechanical.
- **Anchor / origin:** does scale/rotate happen from a non-default origin? (e.g., dropdown scales from `top center`, not `center center`).

### Phase 5 — Edge cases & quality

These are the things that distinguish a polished implementation from a brittle one. Cover all of them, even if the answer is "not visible in artifact — recommend default".

- **Reduced motion:** what should happen with `prefers-reduced-motion: reduce`? Default: skip transforms, keep opacity, shorten to ~0ms.
- **Interrupt:** if the user re-triggers mid-animation (hovers out then back in), does it snap, reverse from current value, or queue? Springs handle this naturally; CSS transitions need explicit handling.
- **Exit symmetry:** is the exit the reverse of the entrance, or different? (e.g., modal enters with scale+fade but exits with fade only.)
- **Loop behavior:** one-shot, loop, ping-pong, or on-demand?
- **Performance:** any property that would force layout (`height`, `width`, `top`)? Flag and suggest a transform-based alternative if visually equivalent.

## Easing identification guide

Watch the speed across the animation, not just the endpoints.

- **Linear** — constant speed throughout. Feels mechanical. Rare in good UI; appears in progress bars and loops.
- **Ease-out** — fast start, slow end. The element decelerates into place. Most common entrance easing. Feels responsive.
- **Ease-in** — slow start, fast end. Element accelerates away. Best for exits.
- **Ease-in-out** — slow-fast-slow. Smooth transitions between two states where neither is "home".
- **Spring** — overshoots past the target value, then settles. Look for any frame where the element passes its final position. Characterized by `stiffness` and `damping`, not duration.
- **Anticipation** — moves slightly opposite first, then forward. Borrowed from animation principles; rare but distinctive.
- **Back-out / overshoot** — like ease-out but overshoots slightly before settling. Common in playful UIs.

Tell springs apart from ease-out: ease-out approaches the target monotonically. Springs cross it.

## Library hints

Suggest one based on what you observed. The downstream agent can override.

- **CSS transitions** — simple state changes, single property, no interrupt complexity.
- **CSS keyframes** — loops, multi-step sequences with fixed timing.
- **Framer Motion (React)** — springs, gestures, layout animations, exit animations via `AnimatePresence`, stagger via `staggerChildren`.
- **React Spring** — physics-driven motion, when spring feel matters more than precise timing.
- **GSAP** — complex sequences, timeline choreography, SVG morphing, fine-grained control.
- **Native View Transitions API** — page/route transitions where browser support allows.

## Output template

Always produce this exact shape. If a field is not observable, write `not visible in artifact` — do not omit the field.

```
# Motion spec: <short name>

## Source
- Artifact: <filename, URL, or description>
- Input type: <video | gif | recording | URL | DOM snippet>
- Frames inspected: <count or "single playback only" or "n/a — source-read">
- Estimated framerate: <30/60/unknown — omit if URL>
- Framework detected: <Framer | Webflow | React + Framer Motion | GSAP | CSS-only | unknown — only for URL>

## Source code findings (URL/DOM inputs only — omit otherwise)
- Relevant CSS rules: <quoted selectors + declarations>
- Relevant inline styles: <quoted style attributes>
- Animation library imports: <list>
- Element targeted: <selector that matched the verbal description>

## Trigger
- Event: <hover|click|mount|...>
- Element receiving event: <selector or description>

## Elements animated
1. <element name> — role in interaction
2. ...

## States
- A (initial): <description>
- B (final): <description>
- (intermediate states if any)

## Per-property motion
### <element 1>
- `<property>`: <from> → <to>, duration <ms>, easing <type>, delay <ms>
- ...

### <element 2>
- ...

## Choreography
- <parallel | sequence | stagger | overlap with details>
- Transform origin: <if non-default>

## Edge cases
- Reduced motion: <behavior>
- Interrupt: <snap | reverse | queue>
- Exit: <symmetric reverse | different — describe>
- Loop: <one-shot | loop | ping-pong>
- Performance flags: <any layout-thrashing properties>

## Recommended implementation
- Library: <css | framer-motion | gsap | react-spring | view-transitions>
- Reason: <one line>

## Confidence
- High / Medium / Low on: <which fields>
- Estimates flagged with ~ in the spec above
```

## Working principles

- **Numbers over adjectives.** "Fast" is useless; "180ms" is implementable. If you cannot get a number, write `~estimated` and a range.
- **Frame counting beats vibes.** A snappy-feeling animation is usually 150–250ms; a deliberate one 300–500ms. But measure when you can.
- **Identify easing by shape, not by name.** Watch where the element is at 25%, 50%, 75% of the duration. Even spacing = linear. Compressed at the end = ease-out.
- **Distinguish observed from inferred.** Anything you didn't see in the artifact is an inference. Mark it.
- **One spec per interaction.** If the artifact shows three separate interactions, produce three specs, not one merged blob.

## See also

`examples/` in this repo for worked input → output pairs covering modal entrance, staggered list, and spring hover.
