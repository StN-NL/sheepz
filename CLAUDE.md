# Sheepz

A Warcraft III custom map: multiplayer PvP arena where players control sheep and fight using a rotating arsenal of weapons. Existing project, not a fresh scaffold.

## Project type

WurstScript project, scaffolded via the VSCode Wurst extension. Uses two upstream dependencies:

- `wurstStdlib2` — the Wurst standard library
- `Frentity` (Frotty/Frentity) — entity/component framework layered on top of the stdlib

Build target is whatever `wurst.build` specifies — check that file before assuming Lua vs JASS mode.

## Directory map

- `wurst/` — source code (`.wurst` files). All map logic lives here. This project organizes it as:
  - `wurst/game/` — gameplay code (further subdivided into `assets/`, `general/`, `weapons/`)
  - `wurst/lib/` — shared utilities (`Damage`, `Status`, `Utils`, `CreepIds`)
  - `wurst/templates/` — hero and item definition templates for reuse
- `imports/` — assets that get auto-imported into the map on save (icons, models). Everything here ships in the map, so keep it to files the map actually uses. A file here replaces the same path inside `Sheepz.w3x`: `war3mapMap.blp` (custom lobby minimap, which the World Editor overwrites on save) and `war3mapPreview.tga` (256×256 lobby preview). The loading screen (`LoadingScreen.mdx` + `Fullscreen.tga`, 5.5 MB) stays in the base map as-is: converting it to BLP gave the wrong aspect ratio in-game.
- `assets/` — source art, tools and unused audio kept out of the map (not imported).
- `_build/` — generated. Compiled output and downloaded dependencies live here. **Gitignored.**
- `_build/dependencies/wurstStdlib2/` — Wurst standard library source. Grep here for real API signatures; do NOT invent them.
- `_build/dependencies/Frentity/` — Frentity source. Grep here for entity/component API before using it.
- `Sheepz.w3x` — the base map (terrain, object editor data, **unit/doodad placements**). Edited in the World Editor, not in code. The build pipeline compiles `wurst/` into this map to produce the playable output in `_build/`.
- `wurst/war3map.j` — **DO NOT EDIT.** This is regenerated from `Sheepz.w3x` on every build. To change anything in it (unit placements, initial player setup, camera bounds, etc.), edit `Sheepz.w3x` in the World Editor and the regenerated `war3map.j` will reflect the change.
- `wurst.build` — project config (name, dependencies, scenario data, player slots, etc.).
- `wurst_run.args` — compiler flags passed when running from VSCode.
- `wurst.dependencies` — pinned dependency revisions. Gitignored.

## Build and run

Use the run button in VSCode (or `F1 → Wurst: Build Wurst Map`), or build from the shell with `grill build Sheepz.w3x` in the project root. The output map lands in `_build/`.

Grill updates a cached copy of the map in place (`_build/cache/Sheepz_*_jass_cached.w3x`). After removing or renaming anything in `imports/`, delete that cache before building: otherwise removed imports linger, and a removed import that had replaced a base-map file deletes the original too.

If `_build/dependencies/` is empty on a fresh clone, the extension needs to run a setup pass first (open the folder in VSCode with the Wurst extension installed and trigger any Wurst command).

## What Claude Code edits vs. what the user edits

**Claude Code edits:** `.wurst` source files, `wurst.build`, `wurst_run.args`, `CLAUDE.md`. Programmatically generated object data via `@compiletime` in Wurst is fine.

**User edits in the World Editor:** terrain, doodad placement, base unit/ability data in the Object Editor, asset imports. If a task requires these, surface it as "you'll need to do X in the World Editor" rather than attempting it.

## API discovery

Before calling a stdlib or Frentity function, **grep `_build/dependencies/` for the actual name and signature.** Both libraries are niche enough that memorized API names are unreliable.

## Architecture

This is an existing codebase with established patterns. **Read what's there before adding to it.** Specifically:

- New weapons go in `wurst/game/weapons/` — match the structure and style of existing weapon files (`Bazooka.wurst`, `Grenade.wurst`, `Shotgun.wurst` are good reference examples of the conventional weapon shape).
- Timers inside a projectile that touch the projectile use `doAfterAlive` / `doPeriodicallyAlive` (end periodic ones with `stopTimer(cb)`), so they're cancelled when it's destroyed. Plain `doAfter` + `if not done` is unsafe: the instance may already be freed and recycled. Effects meant to outlive the projectile (e.g. Molotov fire) keep a plain timer and capture what they need in locals.
- Damage and status effects route through `wurst/lib/Damage.wurst` and `wurst/lib/Status.wurst`. Don't bypass these.
- Hero and item definitions follow the templates in `wurst/templates/`. Use them.
- Entity behaviour builds on Frentity primitives — check how `SheepEntity.wurst`, `MyUnitEntity.wurst`, and `MyProjectile.wurst` use it before writing new entity code.

When in doubt, mimic the closest existing example rather than introducing a new pattern.

## Development principles

These bias toward caution over speed. For trivial tweaks (renames, single-line fixes) use judgment.

### Think before coding

- **Surface assumptions, don't bury them.** If a task has multiple reasonable interpretations (e.g. "make the weapon stronger" — more damage? bigger AoE? faster cooldown?), name the options and ask. Don't pick silently.
- **Wurst's stdlib and Frentity are niche.** Verify API signatures by grepping `_build/dependencies/` before writing the call. Memorized JASS/Lua/other-language patterns are unreliable here.
- **If a task is easier in the World Editor than in code** (terrain tweaks, base unit stats, ability tooltips), say so before writing code that fights the editor.
- **For new gameplay content, ask which existing example to mimic.** Adding a weapon shouldn't be a green-field design exercise when 30 examples already exist.

### Simplicity first

- **Minimum Wurst that solves the problem.** No speculative flexibility, no configuration knobs nobody asked for, no error handling for cases that can't happen.
- **No premature abstractions.** If you find yourself building a framework to add one weapon, stop — copy the closest existing weapon and modify.
- **Verification is mostly "build + play in WC3".** The only automated safety net is `wurst/test/` (Wurst `@Test`, run via the VSCode "Run unit tests" command), which covers pure native-free helpers extracted into `wurst/lib/GameMath.wurst`. Everything touching WC3 natives has no test coverage, so simple, easy-to-reason-about code matters more here than in a project with a full test suite.
- Self-check: "Would the existing weapon files in this repo look like what I'm writing?" If not, reconsider.

### Surgical changes

- **Touch only what the task requires.** Don't reformat adjacent Wurst code, "improve" unrelated comments, or refactor things that aren't broken.
- **Match existing Sheepz style** even if you'd write it differently. Consistency with the codebase beats personal preference.
- **Files that are not yours to edit:**
  - `wurst/war3map.j` — regenerated from `Sheepz.w3x` on every build. Edits get silently overwritten.
  - `Sheepz.w3x` — binary, user-owned in the World Editor.
  - `_build/` — generated output and downloaded deps.
- **Clean up orphans you create** (imports/vars/functions your change made unused). Don't remove pre-existing dead code unless asked.
- If you notice unrelated issues while working, mention them — don't quietly fix them.

### Goal-driven execution

- **Frame tasks as verifiable in-game outcomes,** not "implement X". Before coding, write the success criterion: "after this change, the Shotgun fires 8 pellets in a 30° cone, each dealing 12 damage, observable in-game by clear visual spread + total damage on a stationary target." That criterion is what tells you when you're done.
- **For multi-step work, sketch the plan** (one line per step + how you'll know it worked) before touching code.
- **Compiles ≠ works.** After a Wurst build succeeds, explicitly tell the user what to look at in-game to confirm the feature behaves. Type-checks don't catch game-logic bugs.
- **Don't claim a feature works without a manual playtest signal** from the user. Saying "this should work" is fine; saying "this works" requires having seen it work.
