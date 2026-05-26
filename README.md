# Weekend Mobile Projects

A catalog of weekend-scoped mobile app and game ideas, designed to be built by Claude Code agents using React Native (Expo).

## Structure

```
IDEAS.md          # 18 ideas organized by complexity (3 tiers)
CLAUDE.md         # Agent instructions for this repo
projects/
  noodle/
    SPEC.md       # Detailed spec — ready for an agent to build
```

## Each idea is scoped for one weekend

- **Quick Wins** (6–10 hours) — single screen, minimal state
- **Solid Builds** (12–16 hours) — multiple screens, moderate logic
- **Stretch Goals** (16–24 hours) — backend or complex game logic

## Workflow

1. Browse `IDEAS.md`
2. Pick an idea
3. It either already has a `projects/<slug>/SPEC.md` or you ask Claude to write one
4. Feed the spec to Claude Code — it builds the project

## Tech Stack

| Layer | Tool |
|-------|------|
| Apps & simple games | React Native (Expo, managed workflow) |
| Complex 3D games | Unity |
| 2D assets | Affinity Designer / Inkscape |
| 3D assets | Blender |
| Sound | Procedural generation or CC0 libraries |
| Agent | Claude Code with relevant MCPs |

## Current Projects

| # | Project | Tier | Status |
|---|---------|------|--------|
| 1 | [Noodle](projects/noodle/SPEC.md) | Quick Win | Specced — ready to build |

## License

MIT
