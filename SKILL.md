---
name: ff14-logs-review
description: Use when reviewing Dragonsong's Reprise (DSR / 绝龙诗) FF Logs reports, wipe causes, death chains, mechanic responsibility, or progression pulls. This skill is intentionally limited to DSR-only review; fetch public FFLogs evidence, identify the first meaningful DSR mechanic deviation, separate first death from first responsibility, and output a concise judgement other agents can reuse.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [ffxiv, fflogs, raid-review, wipe-analysis, gaming]
    related_skills: []
---

# FF14 Logs Review Skill

## Overview

This skill turns a public FF Logs link for **Dragonsong's Reprise (DSR / 绝龙诗)** into an evidence-based raid review. It is designed for agents that need to answer DSR-specific questions like “how did this pull wipe?”, “who caused the accident?”, “was this a healer issue or a mechanic issue?”, or “summarize our DSR prog mistakes from this report.” It also includes a DSR **职业执行表** workflow for extracting per-job burst/healer-execution windows from clear logs and comparing them against high-quality / top-player execution.

Scope boundary: this skill is **DSR-only**. If the report is TOP, FRU, savage, criterion, or another duty, state that it is outside this skill's scope unless the user explicitly asks for an unsupported best-effort adaptation.

Core principle: **do not equate first death with first responsibility**. Start from the mechanic window, reconstruct the event chain, then identify the first meaningful deviation from expected play.

## When to Use

Use this skill when the user:

- Sends a DSR / Dragonsong's Reprise / 绝龙诗 FF Logs URL, report code, fight ID, death table, replay link, or exported combat-log snippet.
- Asks about a DSR wipe, death, mechanic accident, responsibility chain, or progression review.
- Wants DSR player-by-player or pull-by-pull feedback grounded in log evidence.
- Mentions FF14/FFXIV, FF Logs, DSR, Dragonsong's Reprise, 绝龙诗, 幻想龙诗绝境战, raid review, wipe review, or “判责/复盘”.

Do **not** use this skill for:

- Non-DSR FF14 logs (TOP, FRU, savage, criterion, normal/extreme trials) unless explicitly doing an unsupported adaptation.
- General FF14 strategy writing with no log evidence.
- Ranking or parse coaching unless the user explicitly asks; use damage tables only as secondary context.
- Blaming players from the death list alone.

## Required Local References

This repo ships two DSR-specific source references. Load them before judging responsibility when available:

- `references/dsr-mechanic-responsibility-framework.md` — 「机制责任判断框架」. Use it to map the pull to the correct phase/mechanic, expected handling, and responsibility rules.
- `references/dsr-positive-example-library.md` — 「正例库｜机制减伤基准」. Use it only when the likely issue is numerical/mitigation/healing sufficiency; compare observed mitigation against clear-log examples rather than treating it as a universal minimum.
- `references/dsr-job-burst-window-table.md` — 「职业执行表」workflow. Use it when extracting all-job burst/healer-execution windows from DSR clear logs and comparing a team's timing, skill order, potion use, and mechanic alignment against high-quality / top-player reference logs.

## Input Parsing

For a URL like:

```text
https://www.fflogs.com/reports/<REPORT_CODE>?fight=<FIGHT_ID>
https://cn.fflogs.com/reports/<REPORT_CODE>?fight=<FIGHT_ID>
```

Extract:

- `REPORT_CODE`: the path segment after `/reports/`.
- `FIGHT_ID`: query parameter `fight`. If absent, inspect the report fights list and ask only if multiple plausible fights remain.
- Region/language host: `www.fflogs.com`, `cn.fflogs.com`, etc. Prefer the host in the user URL.

## FFLogs Public Data Workflow

### 1. Confirm report summary

Fetch the report summary endpoint from the same host:

```text
/reports/fights-and-participants/<REPORT_CODE>/0?lang=cn&
```

Useful fields:

- `fights[]`: `id`, `start_time`, `end_time`, `name`, `boss`, `bossPercentage`, `fightPercentage`, `lastPhaseForPercentageDisplay`, `phases`, `kill`, `combatTime`, `lastCastGameId`, `lastCastTime`.
- `friendlies[]`: real players usually have `server`; keep `id`, `name`, `type`, `server`.

If direct HTTP returns 403 but browser access works, navigate to the report page first and call `fetch()` from that browser context.

### 2. Fetch deaths table

```text
/reports/deaths/<REPORT_CODE>/<FIGHT_ID>/<START>/<END>/0/0/0/-1.0.-1.-1/0/Any/0/<START>?lang=cn&
```

Parse rows with `tr[onclick^="return setFilterDeath"]`.

Extract:

- Time in fight.
- Player name.
- Fatal ability.
- Overkill / one-shot marker.
- Last three attacks.
- Damage taken.
- Healing received.

### 3. Fetch replay segment for mechanism windows

Use fight start plus the death/mechanic time window:

```text
/reports/replaysegment/<REPORT_CODE>/<BOSS_ID>/<SEGMENT_START>/<SEGMENT_END>
```

Replay events can include:

- `type`: `begincast`, `cast`, `damage`, `death`, `applybuff`, `removebuff`, `heal`.
- `ability.name`, `ability.guid`.
- `sourceID`, `targetID`.
- `amount`, `overkill`, `unmitigatedAmount`, `absorbed`, `mitigations`.
- `sourceResources` / `targetResources`: `hitPoints`, `maxHitPoints`, `x`, `y`, `facing`, `nextX`, `nextY`, `nextTimestamp`.

Replay coordinates are approximate evidence, not video proof. Use them to support timing/position claims and mark confidence accordingly.

### 4. Optional damage table

Fetch only when output/DPS is relevant:

```text
/reports/table/damage-done/<REPORT_CODE>/<FIGHT_ID>/<START>/<END>/source/0/0/0/0/0/0/-1.0.-1.-1/0/Any/Any/0/<BOSS_ID>?lang=cn&
```

Do not compare raw DPS across jobs as a fairness judgement. Use same-job percentile/active time when available.

## Analysis Method

### Step A: Identify the terminal wipe event

The last visible mass death is often a consequence. Record it, but do not stop there.

Examples:

- Raidwide/enrage kills everyone → check prior deaths, missing soaks, mitigation, or output loss.
- Stack marker kills multiple people → check whether stack count was short because someone died earlier.
- Debuff explosion kills survivors → check why the debuff holder died before resolution.

### Step B: Find the earliest meaningful abnormality

Walk backwards from the terminal event:

1. First death or first HP collapse.
2. First lethal mechanic hit or one-shot marker.
3. First missing soak/stack/tower/tether/cleanse/interrupt.
4. First positional impossibility in replay coordinates.
5. First cast where mitigation/healing was insufficient despite correct positioning.

If an earlier death was recovered and did not structurally affect the wipe, mention it but do not over-weight it.

### Step C: Classify the root cause

Use one of these cause classes:

- `mechanic-position`: wrong side, wrong safe spot, wrong in/out, clipped by AoE.
- `mechanic-assignment`: wrong tower/soak/stack/marker/partner/order.
- `boss-facing`: boss orientation or tank movement changed the mechanic basis.
- `tank-mitigation`: tank buster, shared tank stack, autos, or required cooldown missing.
- `party-mitigation`: raidwide/stack damage lacked planned mitigation.
- `healing-hp`: players entered a damage hit too low or were not recovered in time.
- `death-chain`: visible failure was caused by earlier dead/missing player.
- `enrage-output`: no earlier execution issue; boss was not killed in time.
- `disconnect/reset`: wipe was caused by disconnect, manual reset, or non-mechanic reason.
- `uncertain`: evidence is insufficient; list missing proof.

### Step D: Separate roles in the responsibility chain

Always distinguish:

- **首死者 / first death**: first player logged as dead.
- **首责候选 / first-responsibility candidate**: player/role/action that first deviated from correct handling.
- **连带受害者 / collateral victims**: players killed by another failure.
- **终端事件 / terminal event**: final raidwide/soak/enrage/explosion that ended the pull.

## Evidence Rules

- One-shot / huge overkill normally indicates mechanic or position failure, not healer failure.
- Normal repeated raid damage with low HP before the last hit can be healer/mitigation/coverage failure.
- If a debuff holder dies and then the debuff explodes, the explosion is usually downstream; investigate why the holder died.
- If a stack/soak kills players, check count and prior deaths before blaming healers.
- If a mechanic depends on boss facing, verify facing/tank movement before blaming individual safe-spot choices.
- If several players die simultaneously, inspect the mechanic window, not alphabetical/death-table order.
- If the user has a static-specific strat or responsibility document, use it as source of truth for assignments.

## Output Template

Default concise answer:

```markdown
## 结论
- 首责候选：<player/role/mechanic or “not enough evidence”>
- 置信度：高 / 中高 / 中 / 低
- 核心理由：<one sentence>

## 日志事实
- 战斗：<duty/boss>, fight=<id>, 进度=<phase/percent>
- 终端事件：<time + event>
- 首个异常：<time + event>
- 首死者：<name + fatal ability>
- 连带受害者：<names or none>

## 判责链条
1. <expected mechanic handling, if known>
2. <what logs/replay show>
3. <why this caused the wipe>

## 还需要的证据
- <only include if confidence is below high>
```

For chat apps, compress to 3-6 bullets while preserving evidence and confidence.

## Pull-by-Pull Report Template

When the user asks for a full report review:

```markdown
## 总览
| Pull | Phase / % | Root Cause | First abnormality | Confidence |
|---|---:|---|---|---|
| 22 | P6 47.9% | healing-hp / death-chain | 12:26 fourth hit killed 4 | 中高 |

## 已较确定的局
### Pull <id>
- 结论：...
- 证据：...
- 链条：...

## 不确定/需补证据的局
- Pull <id>: missing replay positions / assignments / mitigation plan.
```

## Implementation Notes for Agents

### Browser-context extraction sketch

```js
const code = 'REPORT_CODE';
const report = await fetch(`/reports/fights-and-participants/${code}/0?lang=cn&`).then(r => r.json());
const fight = report.fights.find(f => f.id === FIGHT_ID);
const players = Object.fromEntries(
  report.friendlies.filter(p => p.server).map(p => [p.id, {name: p.name, job: p.type}])
);
```

### Death table parser sketch

```js
function clean(s) { return (s || '').replace(/\s+/g, ' ').trim(); }
function parseDeaths(html) {
  const doc = new DOMParser().parseFromString(html, 'text/html');
  return [...doc.querySelectorAll('tr[onclick^="return setFilterDeath"]')].map((tr, idx) => {
    const t = [...tr.children];
    return {
      order: idx + 1,
      time: clean(t[0]?.innerText),
      name: clean(t[1]?.innerText),
      fatal: clean(t[2]?.innerText) || '-',
      overkill: clean(t[3]?.innerText),
      last: clean(t[4]?.innerText),
      damageTaken: clean(t[5]?.innerText),
      healingReceived: clean(t[6]?.innerText),
    };
  }).filter(d => /^\d+:\d\d/.test(d.time));
}
```

## Common Pitfalls

1. **Blaming the first corpse.** The first dead player can be a victim. Trace the mechanic window.
2. **Calling every multi-death a healer issue.** One-shots and missing assignments are not healable.
3. **Ignoring recovered early deaths.** Mention them, but judge whether they caused the final wipe.
4. **Overclaiming from replay coordinates.** Coordinates support a hypothesis; they are not as strong as video.
5. **Forgetting static assignments.** If the group uses custom assignments, generic strategy may be wrong.
6. **Treating terminal enrage as root cause.** Check prior deaths and downtime before saying “DPS issue”.
7. **Mixing languages without lookup.** Use FFLogs ability names from the report language; include English only when helpful for cross-reference.

## Verification Checklist

- [ ] Report code and fight ID were parsed correctly.
- [ ] Fight start/end, phase, percent, and players were confirmed.
- [ ] Death table was fetched and earliest deaths were inspected.
- [ ] Replay or event details were checked for the suspected mechanism window when responsibility depends on timing/position.
- [ ] First death, first responsibility, collateral victims, and terminal event were separated.
- [ ] Confidence level matches evidence quality.
- [ ] Missing evidence is listed if judgement is uncertain.
