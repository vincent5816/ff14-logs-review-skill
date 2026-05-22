---
name: ff14-logs-review
description: Use when reviewing Dragonsong's Reprise (DSR / 绝龙诗) FF Logs reports, wipe causes, death chains, mechanic responsibility, or progression pulls. This skill is intentionally limited to DSR-only review; fetch public FFLogs evidence, identify the first meaningful DSR mechanic deviation, separate first death from first responsibility, and output a concise judgement other agents can reuse.
version: 1.1.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [ffxiv, fflogs, raid-review, wipe-analysis, gaming]
    related_skills: []
---

# FF14 Logs Review Skill

## Overview

This skill turns a public FF Logs link for **Dragonsong's Reprise (DSR / 绝龙诗)** into an evidence-based raid review. It is designed for agents that need to answer DSR-specific questions like "how did this pull wipe?", "who caused the accident?", "was this a healer issue or a mechanic issue?", or "summarize our DSR prog mistakes from this report." It also includes a DSR **职业执行表** workflow for extracting per-job burst/healer-execution windows from clear logs and comparing them against high-quality / top-player execution.

Scope boundary: this skill is **DSR-only**. If the report is TOP, FRU, savage, criterion, or another duty, state that it is outside this skill's scope unless the user explicitly asks for an unsupported best-effort adaptation.

Core principle: **do not equate first death with first responsibility**. Start from the mechanic window, reconstruct the event chain, then identify the first meaningful deviation from expected play.

## When to Use

Use this skill when the user:

- Sends a DSR / Dragonsong's Reprise / 绝龙诗 FF Logs URL, report code, fight ID, death table, replay link, or exported combat-log snippet.
- Asks about a DSR wipe, death, mechanic accident, responsibility chain, or progression review.
- Wants DSR player-by-player or pull-by-pull feedback grounded in log evidence.
- Mentions FF14/FFXIV, FF Logs, DSR, Dragonsong's Reprise, 绝龙诗, 幻想龙诗绝境战, raid review, wipe review, or "判责/复盘".

Do **not** use this skill for:

- Non-DSR FF14 logs (TOP, FRU, savage, criterion, normal/extreme trials) unless explicitly doing an unsupported adaptation.
- General FF14 strategy writing with no log evidence.
- Ranking or parse coaching unless the user explicitly asks; use damage tables only as secondary context.
- Blaming players from the death list alone.

## Required Local References

This repo ships two DSR-specific source references. Load them before judging responsibility when available:

- `references/dsr-mechanic-responsibility-framework.md` — 「机制责任判断框架」. Use it to map the pull to the correct phase/mechanic, expected handling, and responsibility rules.
- `references/dsr-positive-example-library.md` — 「正例库｜机制减伤基准」. Use it only when the likely issue is numerical/mitigation/healing sufficiency; compare observed mitigation against clear-log examples rather than treating it as a universal minimum.
- `references/dsr-job-burst-window-table.md` — 「职业执行表」workflow. Use it when the user asks for job execution/burst-window analysis or data-table maintenance; `kill=true` is an eligibility/quality check, not a routing rule by itself.

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

### Step 0: 先判用户意图，再检查 `kill`

Do **not** route the task from `kill=true/false` alone. First classify the user's intent, then use `kill` only inside that branch.

**复盘 / 判责 / 谁的问题 / wipe**

- Use `references/dsr-mechanic-responsibility-framework.md` to identify the phase/mechanic, expected handling, and responsibility chain.
- `kill=false` / 开荒推进局：评价重点是**推进到了哪个阶段/机制** + **团灭原因是什么**；保留明确判责和置信度。
- `kill=true` / 过本复盘：可以指出机制执行质量和可优化空间，但不要因为通关就自动切到职业执行表或严格职业评价。

**职业手法评价 / 爆发窗口 / 与高质量玩家对比**

- Use `references/dsr-job-burst-window-table.md` when the user asks for job execution, burst-window comparison, or high-quality player comparison.
- `kill=true` is the normal prerequisite for stable comparison against clear-log execution tables.
- `kill=false` should not be compared against clear-log execution tables unless the user explicitly asks; give only directional advice based on the player's alive-time casts.
- 职业执行表数据来自国服各职业顶尖玩家，代表接近理论最优的执行，**不是「合格标准」，是「改进参照」**。差距说明提升空间，不代表当前执行有问题。

**正例库 / 职业执行表新增 / 数据维护**

- If the user asks to add, supplement, append, create a table, or “只新增不要动旧数据”, treat it as data maintenance.
- `kill=true` is only an eligibility/quality check for extracting clean reference facts; it does **not** imply critique, visible judgement, or job-performance evaluation.
- Do not evaluate player skill unless the user also explicitly asks for “评价/对比/复盘”.
- Output only an operation report: what was added, what was verified, what was omitted, and whether old data was left unchanged.

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

## Output Tone

Tone is **intent-bound, not `kill`-bound**. These tone rules apply only to user-visible analysis tasks — **复盘/判责** and **职业手法评价**. They do not apply to data-maintenance tasks such as adding 正例库 rows or creating/updating 职业执行表 rows.

### 复盘 / 判责：开荒推进局（通常 `kill=false`）

Tone: warm, encouraging, clear. Keep the responsibility conclusion and confidence explicit. End with a concrete SOP for the responsible player(s), written from their perspective, telling them exactly what to do next time this mechanic appears.

Rules:
- State the responsibility conclusion clearly. Do not soften or obscure who caused the wipe.
- Close with a "下次这样做" section: step-by-step actions the responsible player should take when this mechanic appears again. One or two sentences per step, written as direct instructions.
- Tone is "我知道你可以做到" not "没关系随便打".
- No sarcasm, no memes.

Example closing:
> 下次遇到这个机制，你可以这样做：判定前先看清安全区方向，贴着圣龙侧站定，等俯冲判定结束再移动。不要提前跑，走早了反而进入危险区。

### 复盘 / 判责或职业手法评价：过本分析（`kill=true`）

Tone: direct, precise, demanding **only when the user asked for analysis/evaluation**. Point out execution gaps without expanding criticism beyond the evidence. Use community memes where there is a clear, natural match — never force one.

For 职业手法评价:
- `kill=true`: strict optimization notes are allowed; compare against high-quality reference windows when requested.
- `kill=false`: give directional improvement advice only; do not shame total output or compare against clear-log totals.

### 正例库 / 职业执行表新增等数据维护

Do **not** use a persona, mentor voice, memes, or strict/soft analysis tone. Use a concise, verifiable operation-report style:

- 新增范围：which mechanisms/jobs/windows/rows were added.
- 验证结果：what was re-read or checked after writing.
- 遗漏项：what was intentionally left blank or unavailable.
- 数据安全：state whether old data was left unchanged / append-only behavior was preserved.

Do not evaluate player skill in this branch unless the user explicitly also asks for “评价/对比/复盘”.

### Meme Reference Library

Meme usage rules:
- Only use a meme when the situation directly matches its original context (e.g. 龙骑 died from a jump skill → 躺尸龙/999；忍者 released 通灵之术 → 兔忍；tank kept moving → 螺旋T).
- One meme per output maximum. If no meme fits naturally, omit entirely.
- Meme appears as a light aside, not the main point. The main point is always the execution gap.
- All ability and mechanic names in Simplified Chinese using official CN server translations.

Use only entries from this list. Do not invent new memes or use memes not listed here.

**龙骑士**
- 躺尸龙 — 龙骑频繁死亡的刻板印象，尤其是跳跃技能硬直期间被 boss 技能命中。适用：龙骑因跳跃/硬直死亡时。
- 999 / 9999 — 龙骑跳进圈里或华丽送命时的自嘲宏。适用：龙骑用跳跃技能送命的瞬间。
- 龙肠 / 红肠 / 香肠 — 巨龙视线连线拉得很长时。适用：龙骑和队友连线异常长时。

**战士**
- 战士的脑子 / 脑子不太好使 — 战士循环简单、蛮力硬打的刻板印象。适用：称赞/调侃战士完全不需要治疗时。

**骑士**
- 骑士别圣灵了快奶人 — 骑士沉迷输出忘记开深仁厚泽。适用：骑士疯狂输出而坦克血量告急时。

**黑骑 / 绝枪**
- 我盾太厚听不清 — 黑骑对减伤/协防请求无动于衷。适用：黑骑忽视团队减伤需求时。
- 超火流星虚晃一枪 — 绝枪开无敌血量骤降到 1，奶妈惊慌失措。适用：绝枪血量骤降而奶妈拼命奶时。

**占星术士**
- 星天开门 — 爆发期还在重复摸牌导致迟迟发不出去。适用：占星在爆发窗口内还在重抽牌时。

**赤魔法师**
- 赤天开门 — 赤魔连续复活多人宛如开门迎客。适用：赤魔在团灭后把全队陆续拉活时。

**忍者**
- 生杀风遁 — 忍者循环出现低级错误。适用：忍者循环明显打错时。
- 兔忍 — 忍者结印失败召唤出密西迪亚兔。适用：忍者突然召唤出兔子时。

**坦克通用**
- 螺旋T / 陀螺T — 坦克拉怪后一直绕着怪转，导致 boss 面向持续变化，近战无法打身位。适用：坦克不停走位导致队友身位技能打空时。
- 铁头 — 故意不躲 boss 技能硬吃伤害。适用：某 DPS 明知有圈还在吃伤害时。

**治疗通用**
- 本分奶 — 只回血不输出。适用：治疗跑完副本输出为零时。
- 放生 — 奶妈对反复犯同类错误的玩家放弃救治。适用：奶妈宣布不再救某人，或某人第三次踩同一个圈时。

**通用场合**
- 雷电法王 / 不动明王 — 被点名后站着不动连累全队。适用：有人被点名后站着不动时。
- 冰三火三 — 黑魔只会基础循环、输出极低。适用：黑魔输出垫底时。
- 低贱红 — 输出职业排本等待时间极长。适用：红职感叹排本等了很久时。
- 月八亏 — 游戏出现离谱玩法或奇葩机制时。适用：某机制设计令人匪夷所思时。

**钐镰客**
- 地狱出殡 — 地狱出境位移方向打反送命。适用：镰刀位移方向打反冲出地图/坠崖时。

Example with meme:
> 第三次爆发窗口开晚了将近 4 秒，义结金兰落在了古代爆震之后，少打了整整一个破碎星云。另外这局龙骑又去拥抱地板了，999。

Example without meme:
> 学者第二次野花覆盖了无尽轮回分摊，但展开慢了 2 秒，盾量只覆盖了前两段，后四段是裸吃的。

## Output Template

Default concise answer:

```markdown
## 结论
- 首责候选：<player/role/mechanic or "not enough evidence">
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
6. **Treating terminal enrage as root cause.** Check prior deaths and downtime before saying "DPS issue".
7. **Mixing languages without lookup.** Use FFLogs ability names from the report language; include English only when helpful for cross-reference.
8. **Comparing progression pulls to kill-log execution.** Burst windows, GCD counts, and damage totals are meaningless comparisons between a wipe pull and a clean kill. Never do this for kill=false logs.

## Verification Checklist

- [ ] Report code and fight ID were parsed correctly.
- [ ] Fight start/end, phase, percent, and players were confirmed.
- [ ] **User intent was classified before using `kill=true/false` to choose the evaluation/data-maintenance framework.**
- [ ] Death table was fetched and earliest deaths were inspected.
- [ ] Replay or event details were checked for the suspected mechanism window when responsibility depends on timing/position.
- [ ] First death, first responsibility, collateral victims, and terminal event were separated.
- [ ] Confidence level matches evidence quality.
- [ ] Missing evidence is listed if judgement is uncertain.
