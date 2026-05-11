# describe-motion

A Claude Code skill that turns motion artifacts (videos, gifs, screen recordings) into precise implementation prompts for a coding agent.

Motion is the part of UI that static descriptions usually fail to capture — duration, easing, stagger, overshoot, and interrupt behavior are invisible in a screenshot but determine whether the result feels right. `describe-motion` is a structured pass that pulls those properties out of the artifact and emits a prompt the next agent can implement against.

## What it produces

A markdown spec covering:

- Trigger and animated elements
- Discrete states (A → B, or A → B → C)
- Per-property motion: from/to values, duration in ms, easing, delay
- Choreography across elements: parallel, sequence, stagger, overlap, transform origin
- Edge cases: reduced motion, interrupt behavior, exit symmetry, loop, performance flags
- Recommended library

See [`SKILL.md`](./SKILL.md) for the full framework and [`examples/`](./examples) for worked input → output pairs.

## Install

Clone into your Claude Code skills directory:

```bash
git clone https://github.com/rivet-design/describe-motion ~/.claude/skills/describe-motion
```

Then invoke with `/describe-motion` or let Claude trigger it automatically when you attach a video/gif and ask to recreate the interaction.

## When to use

Three input shapes are supported:

1. **Media file** — video, gif, screen recording, or frame sequence.
2. **Live URL + verbal description** — e.g., "how is the fade-in on yafafits.com done". The skill reads the page source for exact CSS/JS values, and falls back to asking for a DOM snippet or recording when the page can't be fetched.
3. **DOM/CSS snippet** — paste the markup and styles directly.

Trigger phrases the skill responds to:

- "Implement this animation"
- "Build this interaction"
- "Match this motion"
- "How would I code this" (with a video/gif attached)
- "Translate this video to code"
- "How is the [animation] on [url] done"
- "Describe the [hover/scroll/fade] at [url]"

If the input is a static screenshot with no motion and no URL, this is the wrong tool — use a layout/UI description skill instead.

## Working with the artifact

### Media files

The skill extracts frames when possible:

```bash
# Video → frames at 30fps
ffmpeg -i input.mp4 -vf "fps=30" frames/%04d.png

# Single frame at a timestamp
ffmpeg -i input.mp4 -ss 00:00:01.500 -frames:v 1 frame.png

# GIF → per-frame PNGs (with cumulative state)
magick input.gif -coalesce frames/%04d.png
```

If frame extraction isn't possible in the current environment, the skill marks affected fields as `~estimated`.

### URL inputs

For URLs, the skill prefers reading the actual source over visual inspection — exact CSS values beat frame counting. It uses `WebFetch` to pull HTML and inline CSS, then narrows to the element the user described.

When fetching is blocked (Cloudflare, 403, JS-heavy bundles), the skill asks the user for a DOM snippet or a Chrome Animations-panel screenshot rather than inventing values. See [`examples/04-website-url-fade-in.md`](./examples/04-website-url-fade-in.md) for both paths.

## License

MIT
