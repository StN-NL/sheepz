# Changelog

Newest first. Each version has a player-facing part (the in-game **What's New** quest shows
the latest two, from `WHATS_NEW` in `wurst/game/general/Miscellaneous.wurst`) and an
**Under the hood** part for developers, with the commits that made each change.

## 1.2.0 — unreleased

Not yet playtested; the notes may change before release.

### For players

**New**
- **Sheep bounce off each other.** Get flung into another sheep and you both take a hit that
  grows with the impact speed. The damage is credited to whoever did the flinging, so bazooka
  a rival into a bystander and both hits are yours.
- **Explosions set off explosives.** Any blast detonates dynamite, mines, grenades, cluster
  bombs, bazooka rockets and guided missiles caught in it, a frame later. Each one still
  belongs to whoever placed it, so the kills go to its owner. Line three dynamites up and the
  chain ripples through.
- **Ticking explosives slow everyone near them.** Grenades, cluster bombs, dynamite, armed
  mines and a landed Holy Hand Grenade drag nearby sheep to a crawl until they go off. A
  swelling Baloonicide halves the speed of everyone around it. You are never slowed by your own.
- **Grapple.** A new weapon. The hook latches onto the first thing it reaches - a sheep, a
  tree, or the ground or cliff face it hits - and reels you towards it while you hang on.
  Against another sheep the pull is shared by weight, so hooking someone bigger drags you to
  them and hooking someone smaller drags them to you. Arriving at speed sets off the new
  sheep-on-sheep bounce. Letting go leaves your momentum intact and spares you the landing.
- **Momentum Reverser.** A new weapon. Cast it and every velocity within 400 range flips:
  sheep, every projectile in the air, and your own. Reversing your fall sends you back up and
  out of fall damage; reversing a sheep that was launched slams it into the ground. Only the
  direction changes, never the speed.
- **Trees grow back.** A tree felled by a blast returns after two minutes, with a growth
  animation. Until now every explosion stripped the map permanently.
- **The Villager Kid can be fought back.** It used to be untouchable. Now any blast, the
  Baseball Bat, Fus Ro Dah and the Boomer Hammer knock it flying, a hurt kid flies further
  than a fresh one, and enough damage kills it. It picks its chase back up on landing. Killing
  one scores nothing, since it isn't a sheep.

**Changed**
- **Size is weight.** A bigger sheep is shifted less by the same blast, so the kill leader gets
  harder to move as they level up and a fully swollen Baloonicide barely budges.
- **Big sheep are easier to hit.** A grown sheep's hitbox grows with it.
- Every blast sets off explosives, hard landings and Bladestorm waves included.

### Under the hood
- Sheep-on-sheep collision in `SheepEntity`: an elastic bounce along the contact line using
  size-based mass, with impact damage mirroring fall damage. Each pair is resolved once per
  tick by the lower-numbered sheep and then held off for 0.5 s (`470fe1e`).
- Size-based mass (`MyUnitEntity.mass()`, `SIZE_MASS_EXPONENT`) divides the velocity
  `vec3.knockback` adds, so it covers every `blast()` caller. The HP-based knockback factor
  stays and multiplies with it. `MyProjectile.checkHit3` grows the hit distance with the
  target's scale (`SHEEP_HIT_RADIUS_GROWTH`), and the unit search around a projectile was
  widened to match (`3938a78`).
- Live-projectile registry (`liveProjectiles` in `MyProjectile`), shared by the detonation and
  slow-zone features (`413266a`).
- Detonation runs off a `BlastAreaListener` hook that `MyProjectile` registers with
  `SheepEntity`: the reverse dependency would be circular. Detonation defers through
  `doAfterAlive(0)`, so an explosion can't set itself off and a chain spreads over frames
  (`413266a`).
- Slow zones report themselves to nearby sheep each tick (`vec3.reportSlowZone`) rather than
  being polled, for the same dependency reason. A sheep keeps the strongest report of the tick;
  nothing stacks and nothing lingers (`4488d65`).
- New pure helpers in `GameMath` (`massFromScale`, `bounceVelN`, `collisionDamage`,
  `hitRadius`, `slowedSpeed`) with 15 tests in `GameMathTest` (`3938a78`, `470fe1e`,
  `4488d65`).
- Tree regrowth: `blastArea` queues a regrow for each destructable the blast actually felled
  (life above 0 before, at or below 0 after), restored with `restoreLife(getMaxLife(), true)`
  after `TREE_REGROW_TIME`. A `HashSet<destructable>` keeps one pending timer per tree, and the
  timer is plain so it outlives whatever felled it (`6148751`).
- Villager Kid takes hits: `MyProjectile.knockable` plus a `takeBlast` hook that VillagerKid
  overrides for hit points (`VILLAGER_KID_HP`) and sqrt(maxHP/HP) knockback scaling, matching
  `SheepEntity.blast`. The velocity maths moved into `vec3.knockbackVel`, shared by sheep and
  projectiles. `BlastAreaListener` widened to carry the blast's source and falloff parameters;
  `forKnockablesInRange` serves blasts, the Bat and the new `onHitProjectile` listener used by
  Fus Ro Dah and the Boomer Hammer (`146e4a7`).
- Grapple: one `GrappleHook extends MyProjectile` handles flight, pull and release, so its
  ondestroy is the single cleanup path. Flight overrides `setPhysics` for a fast flat arc;
  latching comes from `onHit3D` (sheep), a per-tick `forDestructablesInRange` (trees) and
  `onGroundHit` (ground and cliff faces, which are the same event for projectiles). The pull
  immobilises the caster with the `lifeCount` guard Word of Power uses, and splits by
  size-based mass against a sheep. The rope is a pool of chain effects repositioned each
  tick. Release calls `skipFallDamage` for the caster only.
- Momentum Reverser: `SheepEntity.momentumReverser()` flips velocity on the caster, on living
  sheep found with `forUnitsInRange`, and on every live projectile via the new
  `forProjectilesInRange`. Projectiles keep their source and owner, and sheep keep their
  tormentor, so credit is unchanged by the flip. Anything that sets its own velocity each
  tick (Guided Missile steering, the Boomer Hammer's return, a chasing Villager Kid) undoes
  the flip on the next tick.
- Version bumped to 1.2.0 at the start of the work, so builds stop overwriting the previous
  release; `CLAUDE.md` now says to do that, and what `X.Y.Z` means for a map (`60ebb7f`).

## 1.1.0 — unreleased

Not yet playtested; the notes may change before release.

### For players

**Changed**
- **Falling hurts more.** Hard landings deal about 10% more damage and start from slightly
  lower heights.
- **Cliffs are solid.** Getting blasted into a cliff stops you at the cliff face instead of
  launching you on top of it.
- **Smooth hills.** Sheep walk down slopes instead of hopping, at their normal speed.
- **Water puts out fire.** Shallow water extinguishes a burning sheep too, also when you're set
  alight while standing in it. You still only drown in deep water.
- **Explosions land where they hit.** Rockets, grenades and friends explode right on the ground
  instead of just below it, so blasts hit slightly harder up close.
- Knockback down a slope slides a little less far.
- Standing still facing a cliff no longer pushes you away from it.

**New**
- **What's New** in the quest log, and a pointer to it when the game starts.

### Under the hood
- Own ground-contact module (`GroundContact`) replaces Frentity's `PhysicsModule` for sheep and
  projectiles (`71e4713`).
  - The contact response sees the impact velocity; surface friction comes after it. Bounce
    thresholds rescaled so bounces are unchanged; fall damage now uses the true impact speed
    with the same constants (25 / 5 / 5).
- Ground contact resolved right after the move, at the point where the body reached the ground;
  one on-the-ground rule; units stick to slopes (`UNIT_STICK_DISTANCE`) (`fb89280`).
- Cliff check looks ahead along the actual movement instead of the facing, and stops horizontal
  velocity at a wall (`164b2d7`).
- Water: `isTerrainWater()` (any water: fire, splashes) next to `isTerrainDeepWater()`
  (swimming, drowning); projectile hooks renamed to `enforceWater` / `isInWater` /
  `onEnterWater` / `onExitWater`; Worm Rider keeps ending in any water (`fa3d7da`).
- Physics maths extracted to `GameMath` with equivalence tests against the old behaviour.
- Version constant `VERSION` in `Miscellaneous` (keep in sync with `wurst.build`).

## 1.0.9 — 2026-09-28

### For players

**Fixed**
- Sheep could become impossible to kill after taking a tiny bit of damage.
- Riot Shield no longer floats in the sky after you die.
- Shots no longer explode mid-air on sheep waiting in heaven.
- Landmines, fire and rain left behind by players who quit no longer hurt anyone or give out
  points.
- Players who quit no longer count toward the end of the game, or win it. If the host quits,
  the next player can use the settings (including closing a paused settings menu).
- Flame Thrower, Minigun, Ascension and Worm Rider stop when you die.
- Blasts, Chicken Bomb, Villager Kid and Web ignore dead sheep and sheep in heaven.
- A Chicken Bomb that never finds a victim explodes by itself after 30 seconds.
- Jet pack and parachute reset properly when you die.
- Dying while webbed, ascending or teleporting no longer affects your next life; dying during
  a teleport no longer teleports you afterwards.
- Fus Ro Dah's shockwave moves again.
- Word of Power shards spawn at the right height.
- Weapon draft: no longer freezes the game when weapons are disabled (rarity 0), and no re-roll
  or double pick while the draft is open.
- Rare glitches where grenades, bombs and other timed weapons could misfire.
- The fire-extinguish effect plays.
- Single player: no second sheep when you're not in the first slot.
- Settings: Air Strike with 0 shards no longer breaks the game; a jet pack cooldown shorter
  than its duration no longer cuts the next flight short.
- The loading screen looks right again.
- No more lag spike the first time a weapon is fired.

**Changed**
- Only the host can change settings; everyone else sees them greyed out, with sliders following
  the host and the chosen mode and map highlighted. With **Proclaim Anarchy**, everyone can.
- Lowering Max Stacks mid-game trims stacks players already have.
- "Player has left the game" messages show again.
- Runs on Warcraft III 2.0 and newer.

### Under the hood
- Review fixes: Fus Ro Dah, Status timed expiries, draft counts, tormentor refs, immobilize
  releases after death (`fb161a3`); Teleport and Word of Power death guards, optimizer enabled
  (`6aa996f`).
- Projectile timers cancelled on destroy via `doAfterAlive` / `doPeriodicallyAlive` (`407ac59`).
- Code review fixes (`2de8f3f`) and bug hunt fixes (`817e436`): lethal threshold below 0.405 HP,
  inert leaver weapons (`isInGame`), leave handling via `EventListener`, target practice slots,
  Air Strike division by zero, jet pack flight id, Baseball Bat locals, heaven collisions, Riot
  Shield on death, menu locking and slider mirroring, Max Stacks trim.
- Newest stdlib/Frentity; lobby preview and minimap restored to the 1.0.6 versions because the
  newer ones break the loading screen on WC3 3.0 (`8b691e5`).
- `wc3Patch: v2.0`, so 3.0-only natives fail to compile (`b2f6841`).
- Frentity `ALL_PLAYERS` with equivalence tests, stdlib `Preloader`, Villager Kid scans the
  sheep only (`0d4ef22`).
- CI: unit tests and map build on push (`5925761`).
- Found and fixed before release: the weapon hotkey loop bound no keys (`hotkeys.length`
  compiled to -1), and casting untargeted weapons by direct order was refused by the dummy
  (`8b691e5`).

## 1.0.7 – 1.0.8

Not released.

## 1.0.6 — 2026-08-25

Weapons and gameplay update (`0ba273c`), including Bladestorm stopping when you die. No
detailed notes were kept. Under the hood: `GameMath` with unit tests, hero and item definition
templates.

## Earlier versions

No notes were kept; the dates are those of the builds.

| Version | Date |
|---|---|
| 1.0.5 | 2026-06-09 |
| 1.0.4 | 2026-06-09 |
| 1.0.3 | 2025-02-16 |
| 1.0.2 | 2025-02-03 |
| 1.0.1 | 2025-02-03 |
| 1.0.0 | 2025-02-03 ("Version 1", `2fcc850`) |
| 0.1.10 | 2025-02-03 |
| 0.1.9 | 2025-02-02 |
| 0.1.8 | 2025-01-21 |
| 0.1.6 | 2024-12-17 |
| 0.1.5 | 2024-12-09 |
| 0.1.4 | 2024-11-10 |
| 0.1.3 | 2024-11-05 |
| 0.1.2 | 2024-10-31 |
| 0.1.1 | 2024-10-23 |
| 0.1.0 | 2024-10-23 |
| 0.0.8 | 2024-07-06 |
| 0.0.7 | 2024-06-12 |
