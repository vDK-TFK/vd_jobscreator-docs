# Changelog

All notable changes to **vd_jobscreator**. Versions follow `MAJOR.MINOR.PATCH`:

- **MAJOR**: breaking changes (config renamed, exports changed, manual migration needed).
- **MINOR**: new features, backwards compatible.
- **PATCH**: bug fixes.

Each entry lists **new config options** and **SQL to run** when there are any. When updating, keep your `config.lua` and `locales/`, then apply what's listed here.

---

## [Unreleased]

### Added
- `AddSocietyMoney(jobName, amount, reason?)` server export: adds money to the job's primary safe, for scripts with a society/`AddMoneyToSociety` option (see [README](README.md#society-money)).

---

## [1.0.0] - 2026-10-14

First public release.

### Added
- In-game admin panel (`/jobscreator`) with dashboard, job list, change history, global shops, webhooks and settings.
- Point placement in the world with crosshair, live NPC/prop preview and teleport to saved points.
- Job sections: grades, blips, boss menu, garages, duty, stashes, shops, offline shop, paychecks, safes, billing, doors (gs_doorlocks) and Discord webhooks.
- Global shops and global blips.
- Weekly on-duty hours report to Discord.
- `/cservicio` command showing how many people are on duty per job.
- Spanish and English languages.
- Server exports for jobs, grades, player job and duty (see [README](README.md#server-exports)).

### SQL
- New install: run `vd_jobscreator.sql` and `vdk_invoice_item.sql`.

---

<!--
Template for new versions (copy above the latest entry):

## [X.Y.Z] - YYYY-MM-DD

### Added
-

### Changed
-

### Fixed
-

### Config
- New: `Config.Something = value` (what it does)

### SQL
- Run once:
  ```sql
  ```
-->
