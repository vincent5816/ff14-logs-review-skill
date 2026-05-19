# FFLogs Endpoint Notes

These endpoint patterns are useful for public FFXIV FF Logs reports. Some requests may return 403 outside a browser session. If that happens, open the report page first and execute `fetch()` from the browser context.

## Summary / Fights / Participants

```text
/reports/fights-and-participants/<REPORT_CODE>/0?lang=cn&
```

Key fields:

- `fights[].id`
- `fights[].start_time`
- `fights[].end_time`
- `fights[].boss`
- `fights[].bossPercentage`
- `fights[].fightPercentage`
- `fights[].lastPhaseForPercentageDisplay`
- `fights[].phases`
- `friendlies[].id`
- `friendlies[].name`
- `friendlies[].type`
- `friendlies[].server`

## Deaths

```text
/reports/deaths/<REPORT_CODE>/<FIGHT_ID>/<START>/<END>/0/0/0/-1.0.-1.-1/0/Any/0/<START>?lang=cn&
```

Parse `tr[onclick^="return setFilterDeath"]` and extract columns:

1. time
2. player name
3. fatal ability
4. overkill / one-shot marker
5. last attacks
6. damage taken
7. healing received

## Replay Segment

```text
/reports/replaysegment/<REPORT_CODE>/<BOSS_ID>/<SEGMENT_START>/<SEGMENT_END>
```

Useful event fields:

- `timestamp`
- `type`
- `sourceID`, `targetID`
- `ability.name`, `ability.guid`
- `amount`, `overkill`, `unmitigatedAmount`, `absorbed`
- `mitigations[]`
- `sourceResources` / `targetResources`: HP, max HP, x/y coordinates, facing, next positions.

## Damage Done

```text
/reports/table/damage-done/<REPORT_CODE>/<FIGHT_ID>/<START>/<END>/source/0/0/0/0/0/0/-1.0.-1.-1/0/Any/Any/0/<BOSS_ID>?lang=cn&
```

Use this only when output, enrage, active time, or job performance is relevant.
