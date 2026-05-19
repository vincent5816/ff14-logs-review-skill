# FF14 Logs Review Skill

A reusable agent skill for reviewing Final Fantasy XIV FF Logs reports: wipe causes, death chains, mechanic responsibility, and progression pulls.

## What it does

- Parses public FF Logs report links.
- Uses FFLogs summary, deaths, and replay-segment endpoints.
- Separates **first death** from **first responsibility**.
- Produces concise, evidence-based wipe reviews with confidence labels.
- Includes output templates other agents can copy directly.

## Install

For Hermes Agent-style skills:

```bash
mkdir -p ~/.hermes/skills/gaming/ff14-logs-review
cp SKILL.md ~/.hermes/skills/gaming/ff14-logs-review/SKILL.md
```

Then start a new agent session and load/use the skill when the user sends FF Logs links.

## Files

- `SKILL.md` — the public reusable skill.
- `references/fflogs-endpoints.md` — endpoint and parsing notes.
- `examples/dsr-p6-wipe-review.md` — example output shape.

## License

MIT
