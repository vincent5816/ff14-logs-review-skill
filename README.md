# FF14 Logs Review Skill

A reusable agent skill for **Dragonsong's Reprise (DSR / 绝龙诗) FF Logs wipe review only**.

> **Scope:** this skill is intentionally limited to **DSR / 绝龙诗复盘**. It is not a general FF14 raid-review skill for TOP, FRU, savage, criterion, or other duties unless another agent explicitly adapts and extends it.

## What it does

For DSR/绝龙诗 reports, this skill helps agents:

- Parse public FF Logs report links and fight IDs.
- Use FFLogs summary, deaths, and replay-segment endpoints.
- Map deaths and damage windows back to DSR phase/mechanic responsibility chains.
- Separate **first death** from **first responsibility**.
- Distinguish root cause, collateral deaths, and terminal wipe events.
- Produce concise, evidence-based wipe reviews with confidence labels.

## How other agents should use it

1. **Load this skill when the user sends a DSR/绝龙诗 FFLogs link** or asks how a DSR pull wiped.
2. **Parse the report URL**:
   - `REPORT_CODE`: the segment after `/reports/`
   - `FIGHT_ID`: the `fight=` query parameter
3. **Fetch fight metadata** from:
   ```text
   /reports/fights-and-participants/<REPORT_CODE>/0?lang=cn&
   ```
4. **Confirm the fight is Dragonsong's Reprise / 幻想龙诗绝境战.**
   - If it is not DSR, tell the user this skill is DSR-only and either stop or clearly label any further analysis as outside-scope.
5. **Fetch the deaths table**:
   ```text
   /reports/deaths/<REPORT_CODE>/<FIGHT_ID>/<START>/<END>/0/0/0/-1.0.-1.-1/0/Any/0/<START>?lang=cn&
   ```
6. **Use replay segments around the suspected mechanic window** when timing, position, mitigation, or HP state matters:
   ```text
   /reports/replaysegment/<REPORT_CODE>/<BOSS_ID>/<SEGMENT_START>/<SEGMENT_END>
   ```
7. **Judge by mechanic window, not by death order**:
   - Identify the terminal wipe event.
   - Walk backwards to the earliest meaningful abnormality.
   - Separate first death, first-responsibility candidate, collateral victims, and terminal event.
8. **Return a compact judgement** with:
   - Conclusion
   - DSR phase/mechanic
   - Log facts
   - Responsibility chain
   - Confidence level
   - Missing evidence, if any

## Recommended agent prompt

```text
Use the ff14-logs-review skill. The scope is DSR/Dragonsong's Reprise only.
Given this FFLogs URL, identify the DSR phase/mechanic, fetch deaths and replay evidence, then explain how the pull wiped. Do not equate first death with first responsibility. Separate root cause, collateral deaths, and terminal event. If the fight is not DSR, say the skill is out of scope.
```

## Install

For Hermes Agent-style skills:

```bash
mkdir -p ~/.hermes/skills/gaming/ff14-logs-review
cp SKILL.md ~/.hermes/skills/gaming/ff14-logs-review/SKILL.md
```

Then start a new agent session. The agent should load/use the skill only for DSR/绝龙诗 FFLogs review requests.

## Files

- `SKILL.md` — the reusable DSR-only review skill.
- `references/fflogs-endpoints.md` — endpoint and parsing notes.
- `examples/dsr-p6-wipe-review.md` — example DSR output shape.

## License

MIT
