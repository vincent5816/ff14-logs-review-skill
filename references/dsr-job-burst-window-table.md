# DSR 职业执行表｜爆发窗口 workflow

Use this reference when building **DSR / 绝龙诗职业执行表** rows from a clear FFLogs report, especially when comparing a team's execution windows against strong/top-player logs. It captures the current per-job burst-anchor rules, Chinese skill-name mappings, DSR phase mapping caveats, and append-only table semantics learned from real DSR clear logs.

This public reference intentionally does **not** require a specific private Notion workspace. If you write into Notion, Airtable, a spreadsheet, or another database, preserve the field semantics and append-only/upsert rules below, and adapt database IDs to your own workspace.

## Scope and table target

- Scope: DSR / 绝龙诗 job execution windows only.
- Goal: record each job's actual 60s/90s/120s burst or healer execution window from FFLogs casts, then compare timing, skill order, potion placement, and mechanic alignment against high-quality / top-player reference logs.
- Preferred layout: one table per job, using the shared fields below. If one combined table is used instead, keep `职业` as the first/title field and include `log来源` for every row.

## Shared fields

Use this schema for every per-job execution table:

- Core fields:
  - `职业` title
  - `爆发类型` select: `60s`, `90s`, `120s`, `减伤爆发`, `群盾`
  - `第几次` number
  - `对应机制编号` rich_text
  - `对应阶段` rich_text
  - `时间戳` number in current database; preserve human-readable `m:ss` in `备注` if needed
  - `GCD技能` rich_text
  - `oGCD技能` rich_text
  - `爆发药` rich_text; potion usage belongs here, not mixed into oGCD
  - `log来源` url
  - `备注` rich_text

## Required interaction pattern

1. First read report metadata: fight kill status, kill/combat time, and the actual 8-player composition from the fight's damage table entries (not all `masterData.actors`, which includes actors from other pulls/reports).
2. Locate and inspect the target table/database schema before writing.
3. Process **one job only** per user confirmation. After writing that job, stop and ask/await confirmation before continuing another job.

## FFLogs extraction pattern

Prefer GraphQL:

- Report metadata + abilities + actors:
  - `reportData.report(code){ fights { id startTime endTime kill combatTime phaseTransitions { id startTime } } masterData(translate:true){ actors { id name type subType server } abilities { gameID name type } } }`
- Fight composition:
  - `table(fightIDs:[$fight], dataType:DamageDone)` and use `table.data.entries[].id` to identify actual players in the fight.
- Job events:
  - `events(fightIDs:[$fight], sourceID:$source, startTime:$s, endTime:$e, dataType:Casts, limit:10000)`; page using `nextPageTimestamp` if present.

## Burst window extraction rules

- Identify windows from the job's major burst markers.
  - Dark Knight anchors: `Living Shadow` / `掠影示现` = 120s, `Delirium` / `血乱` = 60s.
  - Viper anchors: `Serpent's Ire` / `蛇灵气` = 120s, `Reawaken` / `祖灵降临` = 60s / stored burst spender. `祖灵降临` casts within the surrounding 120s window belong to that 120s row. Consecutive `祖灵降临` casts within about 22s of the previous `祖灵降临` are usually one stored/double burst sequence, not separate independent 60s rows.
  - Dark Knight has no 90s burst row; do not use `Salted Earth` / `腐秽大地` as a 90s marker.
- Window range: start roughly 10s before the marker and extend far enough to include the complete burst sequence. For Viper 120s, use about `-10s` to `+35s` around `蛇灵气` so the co-released `祖灵降临` sequence is merged into the same 120s row; for standalone Viper 60s, about `-10s` to `+25s` around `祖灵降临` is usually enough. Include damaging GCD/oGCD casts plus marker skills; keep potion usage in the separate `爆发药` field.
- Merge/absorb closely overlapping markers when a 60s marker is released close to a 120s marker: the 60s marker belongs to that 120s row and should not create an independent 60s row. When adding a new job, compare the resulting row semantics against the existing 暗黑骑士 table: 60s rows should only be independent windows, not duplicate sub-windows inside a 120s row.
- Preserve the actual cast order within the window.
- Split skills into `GCD技能` and `oGCD技能`. If classification is uncertain, append `?` instead of guessing.
- Output 国服中文 skill names, not FFLogs English names. Use FFLogs `abilityGameID` with a Chinese data source such as cafemaker/wakingsands when possible; do not leave English skill names in `GCD技能`/`oGCD技能` after writing. Validate after insertion by searching representative text fields for English leftovers.
- `备注` should include the player name, raw anchor(s), and timing basis, e.g. `玩家：寄寄子丶；时间轴从P2战斗起点开始计时；窗口锚点：蛇灵气；120s窗口已合并同窗60s/祖灵降临技能。`

## Mechanic mapping

- Map each burst timestamp to the DSR judgement framework index (`DSR 事故判定逻辑｜机制责任判断框架`) using fight-relative time plus phase transitions.
- Important DSR FFLogs quirk: public FFLogs DSR fight timelines may start at **攻略 P2 / King Thordan**, not at doorboss P1. Do not assume fight-relative `0:00` means the framework's `P1`. First inspect `phaseTransitions` and enemy actor/cast names:
  - `King Thordan` at `0:00` → framework P2 begins at fight start.
  - next `Nidhogg` phase → framework P3.
  - `left eye/right eye` phase → framework P4.
  - later `King Thordan` incarnation → framework P5.
  - `Nidhogg + Hraesvelgr` → framework P6.
  - `Dragon-king Thordan` → framework P7.
  This avoids the bug where rows appear as P1/P2/P3 and then jump directly to P6 because the doorboss offset was applied incorrectly.
- Fill:
  - `对应机制编号`: e.g. `P2-2`
  - `对应阶段`: concise phase/mechanic description, e.g. `P2一运后半：隐秘二次引导 + 穿天 + 六塔`
- Do not invent a mechanism if the framework cannot be matched; leave the uncertain field blank or mark uncertainty in chat.

## Table writing and verification

- Upsert by `(职业, log来源, 爆发类型, 第几次)` when rerunning to avoid duplicate rows; `第几次` is counted separately per burst type, so `(职业, log来源, 第几次)` alone can collide between 60s and 120s rows.
- Create one row per burst window.
- If separating one large table into per-job tables, with Notion API `2025-09-03` a `POST /databases` response may create a database/data_source pair whose data source initially contains only the default `Name` title even if `properties` were supplied. Immediately `GET /data_sources/{id}` and, if needed, `PATCH /data_sources/{id}` to rename `Name` to `职业` and add the copied schema before inserting rows. Do not assume the schema exists just because database creation returned 200.
- Some database APIs, including Notion, do not expose reliable visual column-order controls. When the user asks for the same header order as an existing per-job table, copy the existing schema and create/patch fields in the same order, then verify the field set; if the API returns a different property order, report that the schema matches but visual column-order verification/reordering may require the Notion UI.
- After writing, re-query the database/data source filtered by `职业` + `log来源` and verify row count plus representative rows: first, middle, last.

## Viper 国服中文 mapping observed in this workflow

Use FFLogs `abilityGameID` as the stable key and translate before writing Notion fields. Common Viper mappings:

- `Steel Fangs` / `咬噬尖齿`
- `Reaving Fangs` / `穿裂尖齿`
- `Hunter's Sting` / `猛袭利齿`
- `Swiftskin's Sting` / `疾速利齿`
- `Flanksting Strike` / `侧击獠齿`
- `Flanksbane Fang` / `侧裂獠齿`
- `Hindsting Strike` / `背击獠齿`
- `Hindsbane Fang` / `背裂獠齿`
- `Vicewinder` / `强碎灵蛇`
- `Hunter's Coil` / `猛袭盘蛇`
- `Swiftskin's Coil` / `疾速盘蛇`
- `Reawaken` / `祖灵降临`
- `First Generation` / `祖灵之牙一式`
- `Second Generation` / `祖灵之牙二式`
- `Third Generation` / `祖灵之牙三式`
- `Fourth Generation` / `祖灵之牙四式`
- `Uncoiled Fury` / `飞蛇之尾`
- `Death Rattle` / `蛇尾击`
- `Twinfang Bite` / `双牙连击`
- `Twinblood Bite` / `双牙乱击`
- `Slither` / `蛇行`
- `Serpent's Ire` / `蛇灵气`
- Role/common actions: `Sprint` / `冲刺`, `Second Wind` / `内丹`, `True North` / `真北`, `Arm's Length` / `亲疏自行`, `Feint` / `牵制`, `Leg Sweep` / `扫腿`
- Potion example: `Grade 3 Gemdraught of Dexterity [HQ]` / `3级巧力宝药（HQ）`

## Reaper / 钐镰客 anchors

For 钐镰客 in DSR, confirm anchors from the actual log before writing rows:

- `120s` burst marker = `神秘环` / `Arcane Circle`; merge nearby `暴食` / `Gluttony`, `夜游魂衣` / `Enshroud`, `大丰收` / `Plentiful Harvest`, `团契` / `Communio`, and high-value shroud GCDs into the same 120s row.
- `60s` burst marker = standalone `暴食` / `Gluttony`, but only when it is outside the surrounding `神秘环` 120s window. Do not create duplicate 60s rows for `暴食` casts absorbed into 120s `神秘环` rows.
- `夜游魂衣` / `Enshroud` is a resource/spender burst entry and can appear multiple times close together, especially around 120s windows or downtime. If the user asks for 120s/60s windows, include it in the oGCD list within the nearest window rather than using it as a separate row anchor.
- Window range: for 120s use about `-8s` to `+42s` around `神秘环`; for standalone 60s use about `-8s` to `+32s` around `暴食`.
- Keep `爆发药` separate from GCD/oGCD. Exclude pure mitigation/movement/utility such as `神秘纹`, `牵制`, `冲刺`, `浴血`, `亲疏自行`, `地狱入境/出境`, and `播魂种` unless the user asks for a mitigation or full execution table.
- Common CN skill names: `切割`, `增盈切割`, `地狱切割`, `死亡之影`, `死亡之涡`, `灵魂切割`, `绞决`, `缢杀`, `虚无收割`, `交错收割`, `团契`, `收获月`, `大丰收`, `神秘环`, `暴食`, `夜游魂衣`, `绞决爪`, `缢杀爪`, `夜游魂切割`, `真北`.

## Ninja / 忍者 anchors

For 忍者 in DSR, confirm anchors from the actual log before writing rows:

- `120s` burst marker = `介毒之术` / `Dokumori`; merge nearby `攻其不备` / `Trick Attack`, `生杀予夺` / `Kassatsu`, `梦幻三段` / `Dream Within a Dream`, `天地人` / `Ten Chi Jin`, `命水` / `Meisui`, `分身之术` / `Bunshin`, `残影镰鼬` / `Phantom Kamaitachi`, and high-value ninjutsu into the same 120s row.
- `60s` burst marker = standalone `攻其不备` / `Trick Attack`, but only when it is outside the surrounding `介毒之术` 120s window. Do not create duplicate 60s rows for `攻其不备` casts absorbed into 120s `介毒之术` rows.
- `分身之术` / `Bunshin` can appear on roughly 90s cadence or drift with downtime/hold. If the user only asks for 120s/60s windows, do not force extra 90s rows; include `分身之术`/`残影镰鼬` in the oGCD list when they land inside a 120s/60s window. If the user explicitly wants all burst cadences, standalone Bunshin windows can be written as `90s` rows with uncertainty noted.
- Window range: for 120s use about `-8s` to `+35s` around `介毒之术`; for standalone 60s use about `-8s` to `+30s` around `攻其不备`.
- In GCD lists, prefer the resolved damaging weaponskills/ninjutsu (`雷遁之术`, `冰晶乱流之术`, `水遁之术`, etc.) rather than raw mudra button spam (`天之印`/`地之印`/`人之印`), unless the user asks for exact mudra execution.
- Keep `爆发药` separate from GCD/oGCD. Exclude pure mitigation/movement/utility such as `牵制`, `残影`, `缩地`, `内丹`, `浴血`, `亲疏自行`, and `冲刺` unless the user asks for a mitigation or full execution table.
- Common CN skill names: `双刃旋`, `绝风`, `旋风刃`, `强甲破点突`, `飞刀`, `雷遁之术`, `水遁之术`, `冰晶乱流之术`, `风魔手里剑`, `月影雷兽牙`, `月影雷兽爪`, `介毒之术`, `攻其不备`, `生杀予夺`, `梦幻三段`, `天地人`, `命水`, `分身之术`, `残影镰鼬`, `六道轮回`, `真北`.

## Monk / 武僧 anchors

For 武僧 in DSR, confirm anchors from the actual log before writing rows:

- `120s` burst marker = `义结金兰` / `Brotherhood`; merge nearby `红莲极意` / `Riddle of Fire`, `疾风极意` / `Riddle of Wind`, `震脚` / `Perfect Balance`, `阴阳斗气斩` / `The Forbidden Chakra`, and blitz GCDs like `凤凰舞` / `Rising Phoenix`, `梦幻斗舞` / `Phantom Rush`, `苍气炮` / `Elixir Field` into the same 120s row.
- `60s` burst marker = standalone `红莲极意` / `Riddle of Fire`, but only when it is outside the surrounding `义结金兰` 120s window. Do not create duplicate 60s rows for `红莲极意` casts absorbed into 120s `义结金兰` rows.
- `疾风极意` / `Riddle of Wind` is a 90s personal buff in modern Monk logs. If the user only asks for 120s/60s windows, do not force extra 90s rows; include `疾风极意` in the oGCD list when it lands inside a 120s/60s window. If the user explicitly wants all burst cadences, standalone `疾风极意` windows can be written as `90s` rows.
- Window range: for 120s use about `-12s` to `+40s` around `义结金兰`; for standalone 60s use about `-18s` to `+35s` around `红莲极意`, so early `疾风极意`/`震脚` prep is captured.
- Keep `爆发药` separate from GCD/oGCD. Exclude pure mitigation/movement/utility such as `牵制`, `真言`, `金刚极意`, `金刚周天`, `浴血`, `冲刺`, `亲疏自行`, `轻身步法`, `演武`, and non-damaging meditation casts unless the user asks for a mitigation or full execution table.
- Common CN skill names: `双龙脚`, `连击`, `双掌打`, `正拳`, `破碎拳`, `崩拳`, `凤凰舞`, `梦幻斗舞`, `苍气炮`, `六合星导脚`, `破坏神脚`, `四面脚`, `地烈劲`, `义结金兰`, `红莲极意`, `疾风极意`, `震脚`, `阴阳斗气斩`, `真北`.

## Machinist / 机工士 anchors

For 机工士 in DSR, confirm anchors from the actual log before writing rows:

- `120s` burst marker = `野火` / `Wildfire`; merge nearby `枪管加热` / `Barrel Stabilizer`, `超荷` / `Hypercharge`, `后式自走人偶` / `Automaton Queen`, `整备` / `Reassemble`, and co-timed `回转飞锯` / `Chain Saw` into the same 120s row.
- `60s` burst marker = standalone `回转飞锯` / `Chain Saw`, but only when it is outside the surrounding `野火` window. Do not create a duplicate 60s row for `回转飞锯` casts absorbed into 120s `野火` rows.
- Window range: for 120s use about `-18s` to `+35s` around `野火` so pre-window `枪管加热` and `后式自走人偶` are captured; for standalone 60s use about `-8s` to `+25s` around `回转飞锯`.
- `钻头` / `Drill` and `空气锚` / `Air Anchor` are frequent GCD damage tools, not independent 60s/120s anchors in this table; include them in the GCD list when they occur inside the chosen window.
- Keep `爆发药` separate from GCD/oGCD. Exclude pure mitigation/utility such as `策动`, `武装解除`, `内丹`, `亲疏自行`, food, and `冲刺` from burst rows unless the user asks for a mitigation table.
- Common CN skill names: `热分裂弹`, `热独头弹`, `热狙击弹`, `烈焰弹`, `钻头`, `空气锚`, `回转飞锯`, `野火`, `枪管加热`, `超荷`, `后式自走人偶`, `整备`, `虹吸弹`, `弹射`.

## Gunbreaker / 绝枪战士 anchors

For 绝枪战士 in DSR, confirm anchors from the actual log before writing rows:

- Observed DSR logs may show no distinct `120s` personal-burst anchor for Gunbreaker. Do not force alternating 120s rows unless the cast pattern contains a clearly separate 120s marker.
- `60s` burst marker = `无情` / `No Mercy`, with same-window `血壤` / `Bloodfest` merged into that 60s row when the log shows them together. In the observed DSR Gunbreaker workflow, every `无情` was paired close to `血壤`, so all rows were `60s` and there were no independent `120s` rows.
- Window range: about `-10s` to `+28s` around `无情` or the paired `血壤`/`无情` cluster, enough to include `音速破`, `倍攻`, `烈牙` combo, `爆破领域`, `弓形冲波`, and continuation oGCDs.
- Keep `爆发药` separate from GCD/oGCD. Exclude pure tank mitigation/stance actions from burst rows unless the user asks for a mitigation table.
- Common CN skill names: `利刃斩`, `残暴弹`, `迅连斩`, `烈牙`, `猛兽爪`, `凶禽爪`, `爆发击`, `倍攻`, `音速破`, `命运之环`, `撕喉`, `裂膛`, `穿目`, `超高速`, `爆破领域`, `弓形冲波`, `无情`, `血壤`, `弹道`, `闪雷弹`.

## Dancer / 舞者 anchors

For 舞者 in DSR, confirm anchors from the actual log before writing rows:

- `120s` burst marker = `技巧舞步` / `Technical Step` and its `技巧舞步结束` / `Technical Finish`. In the same 120s row, merge nearby `进攻之探戈` / `Devilment`, `流星舞` / `Starfall Dance`, `提拉纳` / `Tillana`, and any `百花争艳` / `Flourish` that lands inside the Technical window.
- `60s` burst marker = standalone `百花争艳` / `Flourish`, but only when it is outside the surrounding 120s `技巧舞步` window. Do not create a duplicate 60s row for `百花争艳` casts absorbed into a 120s row.
- Window range: for 120s use about `-10s` to `+45s` around `技巧舞步`; for standalone 60s use about `-10s` to `+30s` around `百花争艳`.
- `标准舞步` / `Standard Step` and `标准舞步结束` / `Standard Finish` are frequent maintenance/damage actions (roughly 30s cadence), not independent 60s/120s anchors in this table; include them in the GCD list when they occur inside the chosen window.
- Keep `爆发药` separate from GCD/oGCD. Common CN skill names: `瀑泻`, `喷泉`, `逆瀑泻`, `坠喷泉`, `剑舞`, `标准舞步`, `双色标准舞步结束`, `技巧舞步`, `四色技巧舞步结束`, `提拉纳`, `流星舞`, `百花争艳`, `进攻之探戈`, `扇舞·序`, `扇舞·破`, `扇舞·急`, `扇舞·终`, `前冲步`, `防守之桑巴`, `治疗之华尔兹`, `即兴表演`, `即兴表演结束`.

## Black Mage / 黑魔法师 anchors

For 黑魔法师 in DSR, confirm anchors from the actual log before writing rows:

- 黑魔没有团辅型 `60s`/`120s` 爆发；不要强行从 `三连咏唱` / `Triplecast`、`即刻咏唱` / `Swiftcast` 创建 60s 爆发行。它们通常是读条/移动工具，写入所在窗口的 oGCD 列即可。
- Use `黑魔纹` / `Ley Lines` as the personal burst / stationary-casting window anchor. Its cooldown is 120s, but DSR downtime and holding can make observed intervals irregular; create one `120s` row per actual `黑魔纹` cast rather than assuming exact 120s cadence.
- Merge nearby `详述` / `Amplifier` and `魔泉` / `Manafont` into the surrounding `黑魔纹` row. If either occurs far away from every `黑魔纹`, treat it as a log-specific exception and mention the uncertainty in `备注` rather than inventing a 60s row.
- Window range: about `-15s` to `+45s` around `黑魔纹`, wide enough to include pre-place potion/opening casts and post-anchor `魔泉`/`详述` sequences.
- Common CN skill names: `爆炎` (Fire III), `炽炎` (Fire IV), `冰封` (Blizzard III), `冰澈` (Blizzard IV), `暴雷` (Thunder III), `绝望`, `悖论`, `异言`, `灵极魂`, `星灵移位`, `秽浊`, `黑魔纹`, `详述`, `魔泉`, `三连咏唱`, `即刻咏唱`, `魔罩`, `醒梦`, `昏乱`, `以太步`, `沉稳咏唱`.

## Paladin / 骑士 anchors

For 骑士 in DSR, use **only 60s burst rows** unless the user explicitly changes the schema. The user corrected that Paladin has no separate 120s burst row in this table.

- `Fight or Flight` / `战逃反应`: `60s` anchor.
- `Requiescat` / `安魂祈祷` is part of the same 60s window, not a 120s anchor.
- Window range: about `-10s` to `+25s` around `战逃反应`, enough to include `沥血剑` / `安魂祈祷` / `悔罪` combo and common oGCDs.
- Common GCD translations: `Fast Blade`/`先锋剑`, `Riot Blade`/`暴乱剑`, `Royal Authority`/`王权剑`, `Atonement`/`赎罪剑`, `Supplication`/`祈告剑`, `Sepulchre`/`葬送剑`, `Goring Blade`/`沥血剑`, `Holy Spirit`/`圣灵`, `Confiteor`/`悔罪`, `Blade of Faith`/`信念之剑`, `Blade of Truth`/`真理之剑`, `Blade of Valor`/`英勇之剑`, `Imperator`/`帝王令`.
- Common oGCD/burst translations: `Fight or Flight`/`战逃反应`, `Requiescat`/`安魂祈祷`, `Circle of Scorn`/`厄运流转`, `Expiacion`/`偿赎剑`, `Intervene`/`调停`, `Blade of Honor`/`荣耀之剑`.
- Keep mitigation actions such as `圣盾阵`, `铁壁`, `预警`, `壁垒`, `神圣领域`, `干预`, `圣光幕帘`, `武装戍卫` out of `oGCD技能` unless the user asks for a mitigation table.

## Samurai / 武士 anchors

For 武士 in DSR, confirm anchors from the actual log before writing rows:

- `120s` burst marker = `意气冲天` / `Ikishoten`; in observed DSR Samurai logs it is followed closely by `奥义斩浪` / `Ogi Namikiri`, `返斩浪` / `Kaeshi: Namikiri`, and usually `必杀剑·闪影` / `Hissatsu: Senei`. This confirms the real 120s burst anchor.
- `60s` personal-burst marker = standalone `明镜止水` / `Meikyo Shisui`, but only when it is outside the surrounding `意气冲天` 120s window. Do not create duplicate 60s rows for `明镜止水` casts used as prep inside an `意气冲天` window.
- Window range: for 120s use about `-15s` to `+35s` around `意气冲天` so pre-window `明镜止水` and potion use can be absorbed; for standalone 60s use about `-8s` to `+30s`, capped before the next 120s window so it does not swallow `意气冲天`.
- Keep `爆发药` separate from GCD/oGCD. Exclude pure mitigation/movement/utility such as `天眼通`, `牵制`, `真北`, `默想`, `冲刺`, and `叶隐` unless the user asks for a mitigation/full-execution table.
- Common CN skill names: `刃风`, `阵风`, `士风`, `雪风`, `月光`, `花车`, `彼岸花`, `纷乱雪月花`, `返雪月花`, `奥义斩浪`, `返斩浪`, `意气冲天`, `明镜止水`, `必杀剑·震天`, `必杀剑·闪影`, `必杀剑·晓天`, `照破`.

## Red Mage / 赤魔法师 anchors

For 赤魔法师 in DSR, confirm anchors from the actual log before writing rows:

- `120s` burst marker = `鼓励` / `Embolden`. Use one `120s` row per actual `鼓励` cast.
- In the observed DSR Red Mage workflow, no stable independent `60s` burst anchor was present. Do **not** force 60s rows from melee combo finishers (`魔回刺` / `魔交击斩` / `魔连攻` / `赤神圣` / `赤核爆` / `焦热` / `决断`) or from `促进` / `Acceleration`.
- `魔元化` / `Manafication` is a personal resource/burst-prep tool that may drift with downtime. Merge it into the surrounding `鼓励` row only when it actually lands in that window; if it occurs far from `鼓励`, note the drift rather than inventing a separate 60s row.
- Window range: about `-10s` to `+38s` around `鼓励`, enough to include the melee combo, `赤神圣`/`赤核爆`, `焦热`, `决断`, `飞刺`, `六分反击`, and displacement/engagement oGCDs. Keep potion usage in `爆发药`; do not pull a potion from ~50s away into a 120s row just because the next `鼓励` was delayed by downtime.
- Common CN skill names: `激荡`, `赤暴雷`, `赤暴风`, `魔回刺`, `魔交击斩`, `魔连攻`, `赤神圣`, `赤核爆`, `焦热`, `决断`, `鼓励`, `魔元化`, `促进`, `飞刺`, `六分反击`, `短兵相接`, `即刻咏唱`, `昏乱`, `抗死`.

## Sage / 贤者 anchors

For 贤者 in DSR, confirm anchors from the actual log before writing rows:

- 贤者 has no raid-buff style `120s`/`60s` damage burst. Do not force a party-burst row if the log lacks one.
- Use `魂灵风息` / `Pneuma` as the observed high-value personal damage/heal window anchor when the user asks for 120s/60s burst windows. Its nominal cooldown is 120s, but DSR downtime and healing plans can make actual casts irregular; create one `120s` row per actual `魂灵风息` cluster, collapsing duplicate FFLogs cast events within about 3s.
- No stable independent `60s` anchor was observed in the Sage DSR workflow. Do **not** create 60s rows from `发炎III` / `Phlegma III` (charge-based high-value GCD, about 40-45s usage), `箭毒II` / `Toxikon II`, or `均衡注药III` / `Eukrasian Dosis III`. Include those GCDs in the nearest `魂灵风息` window if they occur there.
- Window range: about `-18s` to `+35s` around `魂灵风息`; for healers it is acceptable to include healing/mitigation oGCDs in `oGCD技能` because the table records execution within the burst/mechanic window, but keep pure movement like `冲刺`/`神翼` out unless relevant.
- Common CN skill names: `注药III`, `均衡注药III`, `发炎III`, `失衡II`, `魂灵风息`, `箭毒II`, `诊断`, `预后`, `均衡诊断`, `均衡预后`, `心关`, `拯救`, `坚角清汁`, `灵橡清汁`, `寄生清汁`, `白牛清汁`, `活化`, `自生II`, `输血`, `泛输血`, `整体论`, `根素`, `混合`, `消化`, `醒梦`, `即刻咏唱`.

## Scholar / 学者 anchors

For 学者 in DSR, confirm anchors from the actual log before writing rows:

- `120s` burst marker = `连环计` / `Chain Stratagem`; it is the Scholar's outgoing raid-buff contribution in `DamageDone.table.entries[].given` and appears as the only stable party-burst anchor in observed DSR Scholar logs.
- No stable independent `60s` burst anchor was observed in the Scholar DSR workflow. `以太超流` / `Aetherflow` appears on ~60s cadence but is a resource refresh, and `能量吸收` / `Energy Drain` dumps are irregular or clustered around `连环计`; do **not** create standalone 60s rows from them unless the user explicitly asks for resource-management windows.
- Window range: about `-10s` to `+35s` around `连环计`. For healers it is acceptable to include healing/mitigation oGCDs in `oGCD技能` because the table records execution inside the burst/mechanic window; keep pure movement or anti-knockback out unless relevant.
- Keep `爆发药` separate from GCD/oGCD. Common CN skill names: `蛊炎法`, `蛊毒法`, `毁坏`, `破阵法`, `鼓舞激励之策`, `士气高扬之策`, `连环计`, `以太超流`, `能量吸收`, `转化`, `野战治疗阵`, `疾风怒涛之计`, `异想的幻光`, `炽天召唤`, `慰藉`, `仙光的低语`, `异想的祥光`, `展开战术`, `秘策`, `生命回生`, `深谋远虑之策`, `不屈不挠之策`, `生命活性法`, `醒梦`, `即刻咏唱`.

## White Mage / 白魔法师 anchors

For 白魔法师 in DSR, confirm anchors from the actual log before writing rows:

- `120s` personal burst marker = `神速咏唱` / `Presence of Mind`. Use one `120s` row per actual `神速咏唱` cast; DSR downtime/holding can make timings drift, so do not assume exact 120s intervals.
- No stable independent `60s` burst anchor was observed in the White Mage DSR workflow. `法令` / `Assize` is roughly 40s and drifts with fight timing, and `苦难之心` / `Afflatus Misery` is a resource GCD; do **not** create 60s rows from either unless the user explicitly asks for all high-value healing/damage resources.
- Window range: about `-10s` to `+35s` around `神速咏唱`, enough to include the accelerated `闪灼` sequence, nearby `天辉`, `苦难之心`, and `法令` if it lands in the window. For healers it is acceptable to include healing/mitigation oGCDs in `oGCD技能`; keep pure movement/anti-knockback like `冲刺` and `沉稳咏唱` out unless relevant.
- Keep `爆发药` separate from GCD/oGCD. Common CN skill names: `闪灼`, `天辉`, `苦难之心`, `狂喜之心`, `安慰之心`, `医济`, `愈疗`, `复活`, `再生`, `神速咏唱`, `法令`, `神名`, `庇护所`, `神祝祷`, `全大赦`, `节制`, `礼仪之铃`, `水流幕`, `无中生有`, `醒梦`, `即刻咏唱`, `天赐祝福`, `以太变移`.

## Warrior / 战士 anchors

For 战士 in DSR, confirm anchors from the actual log before writing rows:

- In observed DSR Warrior logs, no distinct native `120s` personal-burst anchor was present. Do not force alternating 120s rows unless the cast pattern contains a clearly separate 120s marker.
- `60s` burst marker = `原初的解放` / `Inner Release`. Merge same-window `战嚎` / `Infuriate`, `狂魂` / `Inner Chaos`, `裂石飞环` / `Fell Cleave`, `蛮荒崩裂` / `Primal Rend`, `动乱` / `Upheaval`, and `猛攻` / `Onslaught` into the 60s row.
- Window range: about `-10s` to `+28s` around `原初的解放`, enough to include pre-window `战嚎` / `动乱` and the full `裂石飞环` / `蛮荒崩裂` sequence. Keep potion usage in `爆发药` when it falls inside the 60s window; do not use potion timing alone to invent 120s rows.
- Keep mitigation/tank utility such as `原初的血气`, `原初的勇猛`, `摆脱`, `复仇`, `战栗`, `泰然自若`, `死斗`, `铁壁`, `雪仇`, `挑衅`, `退避`, `守护`, `解除守护`, and `冲刺` out of `oGCD技能` unless the user asks for a mitigation/full-execution table.
- Common CN skill names: `重劈`, `凶残裂`, `暴风斩`, `暴风碎`, `裂石飞环`, `狂魂`, `蛮荒崩裂`, `战嚎`, `原初的解放`, `猛攻`, `动乱`, `超压斧`, `秘银暴风`, `原初大地（极限技）`.

## Summoner / 召唤师 anchors

For 召唤师 in DSR, confirm anchors from the actual log before writing rows:

- `120s` burst marker = `灼热之光` / `Searing Light`. Merge same-window `龙神召唤` / `Summon Bahamut` or `不死鸟召唤` / `Summon Phoenix`, `能量吸收`, `溃烂爆发`, `死星核爆`, and `龙神迸发` / `不死鸟迸发` into the 120s row.
- `60s` burst marker = standalone `龙神召唤` or `不死鸟召唤`, but only when it is outside the surrounding `灼热之光` 120s window. Do not create duplicate 60s rows for demi-summons absorbed into a 120s `灼热之光` row.
- Window range: for 120s use about `-10s` to `+38s` around `灼热之光`; for standalone 60s use about `-10s` to `+32s` around `龙神召唤`/`不死鸟召唤`.
- Keep `爆发药` separate from GCD/oGCD. Exclude pure mitigation/utility such as `守护之光`, `昏乱`, `即刻咏唱`, `醒梦`, `冲刺`, `沉稳咏唱`, and `苏生之炎` unless the user asks for a mitigation/full-execution table. If a caster LB appears inside a window, write it as `星体风暴（极限技）` rather than `LB`.
- Common CN skill names: `毁荡`, `毁绝`, `星极脉冲`, `灵泉之炎`, `红宝石之仪`, `黄宝石之仪`, `绿宝石之仪`, `深红旋风`, `深红强袭`, `山崩`, `螺旋气流`, `灼热之光`, `龙神召唤`, `不死鸟召唤`, `能量吸收`, `溃烂爆发`, `死星核爆`, `龙神迸发`, `不死鸟迸发`, `土神召唤Ⅱ`, `火神召唤Ⅱ`, `风神召唤Ⅱ`.

## Astrologian / 占星术士 anchors

For 占星术士 in DSR, confirm anchors from the actual log before writing rows:

- `120s` burst marker = `占卜` / `Divination`. Merge nearby `星极抽卡` / `Astral Draw`, `灵极抽卡` / `Umbral Draw`, card plays, `王冠之领主` / `Lord of Crowns`, `王冠之贵妇` / `Lady of Crowns`, `光速` / `Lightspeed`, and relevant healing/mitigation oGCDs into the same 120s row.
- `60s` burst marker = standalone `星极抽卡` / `Astral Draw` or `灵极抽卡` / `Umbral Draw`, but only when it is outside the surrounding `占卜` 120s window. Do not create duplicate 60s rows for draws absorbed into a 120s `占卜` row.
- Window range: for 120s use about `-10s` to `+38s` around `占卜`; for standalone 60s use about `-8s` to `+30s` around `星极抽卡`/`灵极抽卡`.
- For healers it is acceptable to include healing/mitigation oGCDs in `oGCD技能` because this table records execution inside the burst/mechanic window; keep pure movement like `冲刺` out unless relevant. Keep potion usage in `爆发药`.
- Common CN skill names: `落陷凶星`, `焚灼`, `阳星相位`, `阳星`, `吉星相位`, `占卜`, `星极抽卡`, `灵极抽卡`, `太阳神之衡`, `放浪神之箭`, `战争神之枪`, `世界树之干`, `建筑神之塔`, `河流神之瓶`, `王冠之领主`, `王冠之贵妇`, `光速`, `地星`, `星体爆轰`, `大宇宙`, `小宇宙`, `中间学派`, `天星交错`, `天星冲日`, `命运之轮`, `天宫图`, `先天禀赋`, `擢升`, `醒梦`, `即刻咏唱`.

## Dark Knight example anchors

For DRK in DSR, common anchors and classification:

- `Living Shadow` / `掠影示现`: `120s`
- `Delirium` / `血乱`: `60s`
- `Salted Earth` / `腐秽大地`: include inside the surrounding 60s/120s row when present; do **not** create a standalone `90s` row from it.

Common DRK GCD set observed in this workflow:

- `Hard Slash` / `重斩`
- `Syphon Strike` / `吸收斩`
- `Souleater` / `噬魂斩`
- `Bloodspiller` / `血溅`
- `Unleash` / `释放`
- `Stalwart Soul` / `刚魂`

Common DRK oGCD/support set observed in this workflow:

- `Edge of Shadow` / `暗影锋`
- `Living Shadow` / `掠影示现`
- `Salted Earth` / `腐秽大地`
- `Salt and Darkness` / `腐秽黑暗`
- `Carve and Spit` / `精雕怒斩`
- `Shadowbringer` / `暗影使者`
- `Delirium` / `血乱`
- `The Blackest Night` / `至黑之夜`
- `Oblation` / `献奉`
- `Dark Missionary` / `暗黑布道`
- `Reprisal` / `雪仇`
- `Rampart` / `铁壁`
- `Dark Mind` / `弃明投暗`
- `Shadow Wall` / `暗影墙`
- `Living Dead` / `行尸走肉`
