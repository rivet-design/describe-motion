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

Trigger phrases the skill responds to:

- "Implement this animation"
- "Build this interaction"
- "Match this motion"
- "How would I code this" (with a video/gif attached)
- "Translate this video to code"

If the input is a static screenshot with no motion, this is the wrong tool — use a layout/UI description skill instead.

## Working with the artifact

The skill assumes you can extract frames. Quick references:

```bash
# Video → frames at 30fps
ffmpeg -i input.mp4 -vf "fps=30" frames/%04d.png

# Single frame at a timestamp
ffmpeg -i input.mp4 -ss 00:00:01.500 -frames:v 1 frame.png

# GIF → per-frame PNGs (with cumulative state)
magick input.gif -coalesce frames/%04d.png
```

If frame extraction isn't possible in the current environment, the skill marks affected fields as `~estimated`.

## License

MIT
