# vd_jobscreator · Jobs Creator for VORP (RedM)

Public home of **vd_jobscreator**: release notes and developer documentation.
The resource itself is sold through Tebex and delivered via the Cfx.re Portal. This repository contains **no source code**.

- 📦 **Buy:** https://vdk-store.tebex.io/
- 💬 **Support:** https://discord.gg/MSeMV2Js4c
- 📝 **Changelog:** [CHANGELOG.md](CHANGELOG.md)
- 🏷️ **Releases:** see the [Releases](../../releases) tab. Each tag matches the `version` in `fxmanifest.lua`.

---

## Contents

- [Versions and updates](#versions-and-updates)
- [Server exports](#server-exports)
  - [Jobs and grades](#jobs-and-grades)
  - [Online player](#online-player)
  - [Editing jobs](#editing-jobs)
  - [Editing a player's job](#editing-a-players-job)
  - [Duty](#duty)
  - [Error codes](#error-codes)
  - [Data shapes](#data-shapes)
- [Examples](#examples)

---

## Versions and updates

- The current version is in `fxmanifest.lua` (`version '1.0.0'`).
- Every release is listed in [CHANGELOG.md](CHANGELOG.md) with what changed, new `config.lua` options and any SQL to run.
- To update: keep your `config.lua` and `locales/`, replace the resource folder, then apply the new config options and SQL from the changelog.

---

## Server exports

All exports are **server-side**. Call them from another resource's server script:

```lua
local vdjc = exports['vd_jobscreator']
local job = vdjc:GetJob('sheriff')
```

Grades are numbered from **0** (the same value VORP stores in `characters.jobgrade`). Wherever an export takes `gradeIndexOrName`, you can pass either that number or the grade's `name` string.

### Jobs and grades

| Export | Arguments | Returns |
|---|---|---|
| `GetJob` | `jobName` | Full job table, or `nil`. Treat it as read-only |
| `GetJobLabel` | `jobName` | Label string, or `nil` |
| `JobExists` | `jobName` | `boolean` |
| `GetJobList` | — | Array of job summaries |
| `GetJobGrades` | `jobName` | Array of grades |
| `GetJobGrade` | `jobName, gradeIndexOrName` | `{ name, label, salary, bossActions, index }`, or `nil` |

```lua
local vdjc = exports['vd_jobscreator']

if vdjc:JobExists('sheriff') then
    print(vdjc:GetJobLabel('sheriff'))            -- "Valentine Sheriff"
end

for _, job in ipairs(vdjc:GetJobList()) do
    print(job.name, job.label, job.gradeCount)
end

local byIndex = vdjc:GetJobGrade('sheriff', 2)
local byName  = vdjc:GetJobGrade('sheriff', 'deputy')
print(byName.label, byName.salary, byName.index)
```

### Online player

| Export | Arguments | Returns |
|---|---|---|
| `GetPlayerJob` | `source` | `{ job, jobLabel, gradeIndex, gradeName, gradeLabel, isBoss }`, or `nil` if the player has no character loaded |
| `IsJobBoss` | `source, jobName` | `boolean`: the player is on `jobName` with a boss grade |

```lua
local info = exports['vd_jobscreator']:GetPlayerJob(source)
if info and info.isBoss then
    print(('%s is boss of %s'):format(GetPlayerName(source), info.jobLabel))
end
```

### Editing jobs

| Export | Arguments | Returns |
|---|---|---|
| `SetJobLabel` | `jobName, newLabel, actorSource?` | `true, job` or `false, errorCode` |
| `SetJobGrades` | `jobName, grades, actorSource?` | `true, job` or `false, errorCode` |

- `SetJobGrades` **replaces the whole grade list**, with the same validation as the panel's Grades tab.
- `actorSource` is only used for the history log. Omit it or pass `0` when there is no player behind the call.
- Changes are saved to the database and sent to every player straight away, exactly like an edit made in the panel.

```lua
local ok, result = exports['vd_jobscreator']:SetJobGrades('sheriff', {
    { name = 'recruit', label = 'Recruit', salary = 15,    bossActions = false },
    { name = 'deputy',  label = 'Deputy',  salary = 25.50, bossActions = false },
    { name = 'sheriff', label = 'Sheriff', salary = 50,    bossActions = true  },
}, 0)

if not ok then print('SetJobGrades failed: ' .. result) end
```

`salary` accepts decimals and is rounded to the cent. It is the amount added to the employee's paycheck balance every `Config.Paycheck.tickMinutes`.

### Editing a player's job

| Export | Arguments | Returns |
|---|---|---|
| `SetPlayerJob` | `source, jobName, gradeIndexOrName?` | `true, { job, gradeIndex, gradeName }` or `false, errorCode`. Grade defaults to `0` |
| `SetPlayerGrade` | `source, jobName, gradeIndexOrName` | `true, { gradeIndex, gradeName }` or `false, errorCode`. Fails if the player is not on `jobName` |

Both apply to the online character immediately (the player gets a notification and their job points refresh). VORP saves the character as usual.

```lua
-- Hire into the sheriff as a deputy
local ok, res = exports['vd_jobscreator']:SetPlayerJob(source, 'sheriff', 'deputy')

-- Promote
local ok2, res2 = exports['vd_jobscreator']:SetPlayerGrade(source, 'sheriff', 'sheriff')
```

### Duty

| Export | Arguments | Returns |
|---|---|---|
| `IsOnDuty` | `source, jobName` | `boolean` |
| `AnyoneOnDuty` | `jobName` | `boolean`: at least one employee of `jobName` is on duty |
| `SetDuty` | `source, jobName, state` | `true, state` or `false, errorCode` |
| `ToggleDuty` | `source, jobName` | `true, newState` or `false, errorCode` |

Always change duty through `SetDuty` / `ToggleDuty`. They keep the duty state, on-duty blips, history log, Discord webhooks and the weekly hours report in sync. Don't write the `duty_<jobName>` state bag yourself.

```lua
RegisterCommand('duty', function(src)
    local info = exports['vd_jobscreator']:GetPlayerJob(src)
    if not info then return end
    local ok, state = exports['vd_jobscreator']:ToggleDuty(src, info.job)
    if not ok then print('duty failed: ' .. state) end
end)
```

### Error codes

Exports that can fail return `false, errorCode`. Common codes:

| Code | Meaning |
|---|---|
| `job_not_found` | No job with that name |
| `player_not_found` | No character loaded for that `source` |
| `invalid_grade` | No grade with that index or name |
| `not_on_this_job` | The player isn't currently on that job |
| `dead` | Duty can't be changed while the player is dead |
| `invalid_label` | Label is empty or longer than 50 characters |
| `invalid_payload` / `invalid_section` | Malformed data passed to an edit export |

Grade validation in `SetJobGrades` can return more specific codes. Every code has a readable translation under `error.<code>` in `locales/*.lua`.

### Data shapes

**Grade** (entries of `GetJobGrades`):

```lua
{ name = 'deputy', label = 'Deputy', salary = 25.5, bossActions = false }
```

**Job summary** (entries of `GetJobList`):

```lua
{ name = 'sheriff', label = 'Valentine Sheriff', type = 'law', playerCount = 9, gradeCount = 4, ... }
```

**Job** (`GetJob`): the full configuration as edited in the panel. Main fields:

| Field | Content |
|---|---|
| `name`, `label`, `type` | Identity |
| `grades` | Array of grades |
| `blips`, `bossMenus`, `garages`, `dutyPoints`, `stashes`, `paycheckPoints`, `safes`, `doorlocks`, `billingHistoryPoints` | Arrays of points `{ id, label, coords = {x,y,z,w}, minGrade, interaction, ... }` |
| `shops` | Shops `{ id, label, minGrade, items, locations, ... }` |
| `offlineShop` | Offline shop config |
| `safeConfig`, `billing` | Section settings |

Treat the returned table as **read-only**. To change a job, use the edit exports or the panel.

---

## Examples

**Only let on-duty sheriffs use something:**

```lua
local function isOnDutySheriff(src)
    local vdjc = exports['vd_jobscreator']
    local info = vdjc:GetPlayerJob(src)
    return info ~= nil and info.job == 'sheriff' and vdjc:IsOnDuty(src, 'sheriff')
end
```

**Show a message when no doctor is on duty:**

```lua
if not exports['vd_jobscreator']:AnyoneOnDuty('doctor') then
    TriggerClientEvent('chat:addMessage', src, { args = { 'Clinic', 'No doctors on duty right now.' } })
end
```

**Boss-only command:**

```lua
RegisterCommand('bossnote', function(src, args)
    if not exports['vd_jobscreator']:IsJobBoss(src, 'saloon') then return end
    -- ...
end)
```
