# FF14 Logs Review Skill

A reusable agent skill for **Dragonsong's Reprise (DSR / 绝龙诗) FF Logs review, job-execution comparison, and reference-table maintenance**.

> **Scope:** this skill is intentionally limited to **DSR / 绝龙诗复盘**. It is not a general FF14 raid-review skill for TOP, FRU, savage, criterion, or other duties unless another agent explicitly adapts and extends it.

## What it does

For DSR/绝龙诗 reports, this skill helps agents:

- Parse public FF Logs report links and fight IDs.
- Use FFLogs summary, deaths, replay-segment, GraphQL casts, and table endpoints.
- Map deaths and damage windows back to DSR phase/mechanic responsibility chains.
- Separate **first death** from **first responsibility**.
- Build **职业执行表** rows for every job from actual burst/healing-execution windows.
- Compare a team's execution timing, burst anchors, potion usage, and mechanic alignment against high-quality / top-player reference logs.
- Distinguish root cause, collateral deaths, and terminal wipe events.
- Produce concise, evidence-based wipe reviews with confidence labels.
- Route tasks by **user intent first**, then use `kill=true/false` only as an eligibility/evaluation signal.

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
9. **Classify user intent before using `kill`**:
   - 复盘/判责：use the DSR responsibility framework; `kill=false` usually means progression-SOP tone, `kill=true` can be stricter but only for analysis.
   - 职业手法评价：use clear-log execution tables for comparison only when the user asks for job-performance/burst-window analysis.
   - 正例库/职业执行表新增：treat as data maintenance; do not apply analysis persona, memes, or player-skill critique.
10. **When creating a 职业执行表**, use `references/dsr-job-burst-window-table.md`:
   - Confirm the actual 8-player composition and target job source ID.
   - Extract cast events and identify that job's real 60s/90s/120s anchors.
   - Write one row per burst/healer-execution window, preserving Chinese skill names and actual cast order.
   - Use the table to compare timing, skill order, potion use, and mechanic alignment against top-player / high-quality reference logs only when the user asks for comparison/evaluation.

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
- `references/dsr-mechanic-responsibility-framework.md` — DSR / 绝龙诗「机制责任判断框架」exported from the source Notion document; use this as the judgement source of truth.
- `references/dsr-positive-example-library.md` — DSR / 绝龙诗「正例库｜机制减伤基准」exported from the source Notion database; use this for observed clear-log mitigation baselines.
- `references/dsr-job-burst-window-table.md` — DSR / 绝龙诗「职业执行表」workflow covering all observed jobs' burst/healer-execution anchors, Chinese skill mappings, table schema, and top-player comparison workflow.
- `references/fflogs-endpoints.md` — endpoint and parsing notes.
- `examples/dsr-p6-wipe-review.md` — example DSR output shape.
- `CHANGELOG.md` — public update log.

## License

MIT
