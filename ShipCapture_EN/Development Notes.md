# Ship Capture - Development Notes

> For future developers (human or AI). This document records the full architecture, values and driver
> relationships of this mod, plus every problem encountered during development, the failed attempts and
> the final solutions, so that the mod can be optimized or extended without repeating mistakes.
> Mod id: `sw_capture`, game version: Starsector 0.98a-RC8.

---

## 1. Mod overview

This mod is built on the skeleton of *Superweapons Arsenal 3.0 (modified base)*. Core gameplay: seize control of enemy ships.

| Module | Type | id | Role |
| --- | --- | --- | --- |
| Hacking Array | hullmod | `sw_capture_array` | Locks 1 enemy **unmanned** ship (Remnant automated ships / drone ships) and hacks it over time; seizes control at full progress |
| Boarding Pod | fighter wing | wing=`sw_boarding_wing`, fighter=`sw_boarding_pod_wing` | Touches enemy **manned** ships to accumulate boarding hits; seizes control at full count |
| Boarding Driver Core | hullmod (0 OP) | `sw_capture_boarding_driver` | Drives all boarding-side logic (malfunctions / transition / repair) - only linked to the Boarding Pod |
| One-shot boarding fighter | mechanic (no entity definition) | — | When a ship with accumulated boarding hits is destroyed, releases one-shot boarding fighters proportional to its count x return fraction to keep boarding ("revenge") |

### Key values (defaults; adjustable in LunaLib Mod Settings)

- Hits required: frigate 25 / destroyer 38 / cruiser 63 / capital 100
- Boarding touch range: 120; max engagement range: 4000; fighter max speed: 300; fighter replacement time: 6
- Hack range: 2600; hack chain range: 2000; hack transition 5 s; board transition 10 s
- Sunk-return fraction: 0.5 (one-shot fighters released = accumulated count x 0.5, rounded down)
- Malfunction check interval: board 10 s / hack 5 s; chance = 0.2 + (progress - 40%) x 1.0
  (20% at 40%, 30% at 50%, 50% at 70%, 80% at 100%)
- Malfunction durations: hack 40%-70% band 2.5 s, above 70% 4 s; weapon malfunction 8/16 s;
  board engine/shield 5 s, weapon 12 s; critical malfunction x2
- Post-capture repair: +0.5% max hull per second, 40 ticks total (40 x 0.5% = 20%)

### Unified capture flow (shared by boarding and hacking)

```
Progress full
  -> mark "captured" (KEY_CAPTURED; Boarding Pod / Hacking Array immediately stop targeting and counting)
  -> transition (paralyzed + phase-like with no VFX; keeps original faction and captain; duration board 10 s / hack 5 s)
  -> transition done (tickCaptureTransitions): switch faction to yours + clear captain + finishCapture
      (owned fighter wings switch to you, rebuild AI with ceasefire, fix original ownership)
  -> unconditionally set repair marker (+0.5% hull per second, stops after 40 ticks)
  -> battle end (enemy has no surviving deployed ships / full retreat / isCombatOver):
      restore original faction -> truly sink it (damage source = a ship equipped with a capture hullmod, for bounty settlement)
      -> clear captain -> remove from enemy combat manager (deployed/reserve) -> add to player campaign fleet
```

---

## 2. Module driver relationships (important)

This is the part most prone to bugs. First, who drives what every frame:

```
Engine-level per-frame plugin CaptureCombatPlugin (EveryFrameCombatPlugin; called every frame by the engine, mode-independent)
   |- every frame: HijackUtil.tickCaptureTransitions (transition/repair)
   |- every frame: HijackUtil.tickBattleEndSalvage (battle-end takeover check, throttled to 0.5 s in battle)
   |- every frame: HijackUtil.checkGhostRelease (target destroyed -> release one-shot fighters)
   |- damage listener: reportDamageApplied -> tickBattleEndEarly (take over the instant the enemy's last real ship is sunk)
   `- 3 s after combat over: HijackUtil.postBattleRecover (campaign-layer backstop takeover)

Hullmod BoardingDriver (engine calls advanceInCombat only in combat mode; NOT called in strategy mode)
   |- every frame: tickCaptureTransitions / tickCapturedWingAllegiance / tickBattleEndSalvage
   `- every frame + every 10 s: malfunction countdown and rolls for all boarded targets

Hullmod CaptureArray (Hacking Array, same as above)
   |- every frame: hack target selection/hold/progress, malfunction tick and roll
   `- every frame: tickCapturedWingAllegiance / tickBattleEndSalvage

Campaign layer:
   CombatPluginRegistrar (sector-persistent per-frame script) -> registers the engine-level plugin as soon as a combat engine is created
   PostBattleRecover (CampaignEventListener) -> reportBattleOccurred registers the engine-level plugin +
     reportBattleFinished calls postBattleRecover
   WingSupplier (EconomyTickListener) -> daily restock of Boarding Pod wings + Boarding Driver Core at military markets
```

**Key lesson: in Starsector, a hullmod's `advanceInCombat` is only called by the engine while the player is in combat mode (has a player ship); it is NOT called in strategy mode (fleet-control panel).** Therefore any logic that must keep running in every mode (transition timing, takeover detection) has to live in the engine-level plugin `EveryFrameCombatPlugin`, registered by a sector-level script / campaign listener (do not rely on the hullmod's first call to trigger it).

### Performance optimization (first release cleanup, 2026-10)

1. **Same-frame dedup throttling**: `tickCaptureTransitions` / `tickCapturedWingAllegiance` are called every frame by 3 drivers (engine-level plugin + 2 hullmods). Added a WeakHashMap timestamp table keyed by `CombatEngineAPI` instance (`TRANSITION_TICKED` / `WING_TICKED`): only the first driver actually runs the full-battlefield pass in a given frame, the rest return immediately.
2. **Battle-end scan throttled to 0.5 s**: the "enemy has no surviving deployed ships" check in `tickBattleEndSalvage` does not need to run every frame (sinking/retreat is gradual; 0.5 s is enough to detect it with ample buffer before the settlement panel is generated); after `isCombatOver` it runs every frame unthrottled so takeover always wins.
3. **Target search de-rate**:
   - `CaptureArray.findBestHackTarget` (full-battlefield scan) re-searches every 0.5 s, keeping the old target / waiting in between;
   - `GhostBoardingFighterAI.findNearestTarget` re-searches every 0.25 s, and re-searches immediately when the cached target is invalidated (no self-destruct inside the throttle window; waits for the next search point).
4. **Loop merging**: `BoardingDriver` originally ran two full-battlefield passes over boarded targets every frame (malfunction tick + roll); merged into one pass, and the `allBoardingHits()` snapshot copy dropped from 2 to 1 per frame.
5. **Default value fix**: `CaptureConfig.BOARDING_RANGE` default changed 180 -> 120 (consistent with modSettings.json / LunaSettings so behavior does not drift when config load fails).

---

## 3. Pitfall log (problem -> attempt -> result -> conclusion)

> Grouped by topic. Every entry is a real experience; the adopted solution is marked, failed approaches are kept to save future developers from retrying them.

### 3.1 Mod startup errors

**1. Bare `%` in CSV text crashes the game**
- Symptom: opening a LunaLib Mod Settings panel crashes, `Fatal: Conversion` / `JSONObject["fieldID"] not found`.
- Root cause: bare `%` in LunaSettings.csv description text; LunaLib treats it as a format string and blows up.
- Fix: write `%` as `%%` in all text; files must be UTF-8 **without BOM**.
- Lesson: check every CSV parsed by the game / LunaLib for bare `%` and BOM.

**2. Quotes / newlines in CSV text cause `Mismatched quotes`**
- Symptom: `Fatal: Mismatched quotes in the string; last quote: [...]`.
- Root cause: English quotes or cross-line text in hull_mods.csv descriptions shift CSV parsing.
- Fix: use Chinese quotes "" in descriptions, one line per field; desc must not contain English double quotes.
- Lesson: desc fields in hull_mods.csv / wing_data.csv are "one field per line"; quotes and newlines inside break the structure.

**3. JSON field missing / wrong type**
- `JSONObject["name"] not found`, `JSONObject["cost_frigate"] is not a number`, `JSONObject["id"] not found`, `JSONObject["fieldID"] not found`.
- Root cause: wing_data.csv / ship_data.csv etc. corrupted (column misalignment, deleted field, wrong type); or JSON/CSV parsing hit blank/comment lines.
- Lesson: keep column count and header consistent when editing data tables; cost fields must be numbers; id fields cannot be empty; CSV `#` comment lines go above the header, not in the data area.

**4. Hullmod description text crashes on mouse hover**
- Symptom: game runs, hovering a hullmod crashes with Fatal: Conversion errors (single Chinese characters quoted in the log).
- Root cause: desc text contains character sequences the engine treats as escapes/formatting.
- Fix: rewrote the desc following the style of the working "Hacking Array" hullmod description; avoid special characters.
- Lesson: hullmod desc text must be conservative - no English quotes, no bare `%`, no sequences that could be interpreted.

**5. starfarer.api.jar version mismatch**
- Symptom: `Fatal: WARNING:AoTD ... You must replace the 'starfarer.api.jar'`.
- Root cause (background): some mods (AoTD) require a specific starfarer.api.jar version.
- Lesson: compile against the vanilla 0.98a core jars; do not replace the core api because of another mod's requirement.

### 3.2 Fighter wing behavior

**6. Wing without a mothership is not driven by the engine**
- Symptom: spawning a wing with `spawnShipOrWing` produces no fighters.
- Root cause: vanilla fighter wings depend on a mothership existing / being deployed to drive respawn and sortie.
- Attempt: hidden carrier scheme (gemini/condor/heron/legion slot conversion) -> worked but complex; later dropped.
- Final (Plan B): spawn `spawnShipOrWing("sw_boarding_wing")`, then grab `wing.getWingMembers()` to obtain the fighter entities and `engine.removeEntity` the extras; fighters are **fully driven** by our engine-level plugin `GhostBoardingFighterAI` (every frame setFacing + rewrite `getVelocity()` vector to force a straight charge at the nearest manned enemy ship).

**7. Fighter AI orbits / long-range fires / flies away**
- Symptom: boarding fighters do not approach the enemy, orbit at range firing weapons, or inexplicably fly away from the carrier.
- Root cause: vanilla bomber/fighter AI behavior (keep distance, avoid, attack) conflicts with the "touch-range boarding" requirement.
- Attempt: NullAI takeover -> fighters do not move; phased hidden carrier -> AI retreats without releasing;
- Final: keep vanilla fighter AI (normal shield/evasion) for **regular Boarding Pods**; one-shot fighters are fully hard-driven by the engine-level plugin.

**8. Wing infinite respawn while mothership alive; "remove when alive=0" unreliable**
- Root cause: vanilla wings are continuously replenished by the mothership.
- Lesson: for "one-shot" semantics, do not rely on vanilla wing lifecycle; manage fighter entities explicitly with an engine-level plugin (spawn -> drive -> self-destruct -> remove).

**9. One-shot fighter double-count bug (each fighter counted twice)**
- Symptom: a ship with initial count 12, after 5 one-shot fighters board: expected 17, observed 22 (= 5 x 2).
- Root cause: `BoardingPod` (regular boarding logic) did not skip one-shot fighters; the same fighter was counted by both `GhostBoardingFighterAI` and `BoardingPod`; the customData marker set at `spawnShipOrWing` spawn time is unreliable.
- Final: static weak-reference identity set `HijackUtil.GHOST_FIGHTERS` (same-instance judgment as the AI plugin, necessarily consistent) + customData marker as a second safeguard; `BoardingPod` checks the set/marker at entry and skips; the AI plugin also removes on self-destruct/cleanup.

**10. One-shot boarding missile attempt failed**
- Attempt: release "one-shot missiles with similar hull value" instead of fighters -> missiles get eaten by wreck collisions / uncontrollable; failed.
- Lesson: Starsector missile entities are unsuitable as "touch-count units"; revert to the one-shot fighter scheme.

**11. Boarding Pods do not close in (regular boarding fighters)**
- Symptom: regular boarding fighters only fire energy weapons at range.
- Root cause: default fighter AI engagement behavior.
- Fix (user's scheme): wing type "bomber", engagement range 4000, max crew 15, min crew 1, defense "all-shield";
  a successful touch **zeroes the fighter's ammo and CR**, letting vanilla resupply logic (auto-recall when CR/ammo zero) create the "board -> return to resupply -> sortie again" loop;
  only fighters with CR > 0 can count, mechanically preventing repeated touches.

### 3.3 Capture / post-battle ownership (the longest debugging history of this mod)

**12. Enemy "Reserve" still shows the original ship in the battle report**
- Symptom: captured ships already joined the player fleet, but the enemy battle-report "Reserve" still lists the original - "same ship on both sides" (ship duplication).
- Attempt chain:
  1. `addFleetMember` directly into the player fleet at capture moment -> still remains in report;
  2. `reportBattleFinished` backstop -> too late, the settlement snapshot is already generated;
  3. `DamageListener.reportDamageApplied` performs the takeover the instant "the enemy's last real ship is sunk" -> works, but complex;
  4. Copy-and-sink scheme (copy an identical new ship into the player fleet; original reverted to its old faction, CR and hull zeroed) -> "Reserve" still showed the original (copy succeeded but the original was not moved to "Sunk or Scrapped");
  5. **Reverted to the final scheme**: after transition, switch to your side + at battle end (no enemy deployment / full retreat) restore original faction -> truly sink (damage source = a ship equipped with a capture hullmod) -> clear captain -> remove from enemy manager (deployed/reserve) -> add to player campaign fleet.
- Lesson: the battle-report snapshot is finalized when `isCombatOver` is determined, **later than the plugin's advance**; only the engine's internal damage event callback can get before the snapshot; "sinking" must be a real `engine.applyDamage` kill (the engine records it as destroyed, so the report panel puts it under "Sunk or Scrapped").

**13. Capture logic fails in strategy mode (never entering combat mode)**
- Symptom: in full strategy mode, captured ships skip the transition, keep their captain, and do not join the player fleet after battle; everything works after entering combat mode once.
- Root cause: in strategy mode the hullmod `advanceInCombat` is not called by the engine -> the transition driver stalls.
- Attempt: engine-level plugin + lazy registration on first hullmod call -> still depended on having entered combat mode; campaign-level `reportBattleOccurred` registration -> backstop;
- Final: sector-persistent script `CombatPluginRegistrar` polls `Global.getCombatEngine()` every frame and calls `ensureRegistered` as soon as a combat engine is created (idempotent, deduped by weak refs keyed on engine instance).

**14. Campaign members created mid-battle have no CR baseline**
- Symptom: after `addFleetMember` of a copied ship into the player campaign fleet mid-battle, the deploy panel shows "no CR (star-with-ban)", auto-deploy gives the "lightning (forced retreat)" flag and it retreats; manual deploy from the G panel works.
- Root cause: members added directly to the campaign fleet mid-battle lack campaign-layer CR data / deployment authorization baseline.
- Lesson: do not expect direct mid-battle writes to the campaign fleet to deploy properly; deployment semantics must go through vanilla `DeployedFleetMemberAPI` / the deploy queue. This mod eventually abandoned "mid-battle copy into the player fleet" in favor of unified post-battle takeover of the original.

**15. Sink-then-revive = wrecked, does not revive**
- Attempt: at the capture moment, "truly sink" the target then revive -> the target becomes a wreck and cannot revive.
- Lesson: the engine's sink flow is irreversible; any revival approach hits residual wreck state.

**16. Phased hidden carrier AI retreats and does not release fighters**
- Attempt: phased + fully invisible hidden carrier auto-releases fighters -> AI retreats, no release.
- Fix: force-release the boarding fighters once by code, then remove the hidden carrier 4 s after release (engine-level plugin with its own lifecycle, modeled on vanilla ShardSpawner motherships fading out).
- Note: this scheme (Plan A) was fully replaced by Plan B (no hidden carrier; spawn fighters directly).

**17. Bounty-style mission settlement**
- Requirement: count as a player kill so bounties settle correctly.
- Fix: the sink damage source is a player ship equipped with Boarding Driver Core / Hacking Array (`findCaptureSourceShip`); falls back to the player flagship / engine when missing.

### 3.4 Timing and battle speed

**18. Real-time millisecond timing breaks battle speed**
- Attempt: `System.currentTimeMillis()` for transition timing -> at 2x battle speed the transition still took real 10 s.
- Fix: always use `engine.getTotalElapsedTime(false)` (false = live battle clock excluding pause, follows battle speed).
- Lesson: battle timing only trusts engine time; never use the system clock.

### 3.5 Toolchain lessons (build/deploy)

- `javac` on Windows does not expand `*.java` wildcards, and PowerShell does not auto-split a single string with spaces - use an array argument `@(Get-ChildItem ... | ForEach-Object FullName)` with `& javac ... @srcs`.
- Bilingual dual-jar workflow: `src` (ZH) -> copy to `src_en` -> run `en_floats.py` (replaces user-visible strings such as floating text with English) -> compile and package separately; the ZH jar deploys to the live ZH mod folder and its backup copy (the two MD5s must match), the EN jar goes to the `ShipCapture_EN` backup folder.
- Verification with `VerifyCapture.java` (key class tag check) under the game jre: `java -cp "capture_dev;Capture.jar;core cp" VerifyCapture <jar path>`.
- All data files (CSV/JSON/text) are UTF-8 without BOM; PowerShell's `Get-Content` decodes as ANSI by default, so Chinese mojibake in console output is normal and does not mean the file is corrupted.

### 3.6 Mod Settings adjustable "static data items" (write-back to data files)

**19. Fighter wing value/OP/wing size/required crew/replacement time and Hacking Array OP have no runtime API**
- Attempt: override with `MutableShipStatsAPI` -> after compile verification these fields **only take effect at data-file load time** (wing_data.csv / ship_data.csv / hull_mods.csv); the stats interface cannot change them.
- Fix: two categories -
  - **Static data items** (market value / OP / wing size / crew / replacement time / Hacking Array OP): LunaLib settings change callback + `CaptureConfig.applyStaticDataOverrides()` on `onGameLoad` writes current values **back to the data files** (only writes when a field value actually changed, then reads back to verify), with a "restart the game to apply" notice.
  - **Combat stat items** (speed / acceleration / deceleration / hull / armor / flux capacity / flux dissipation): `MutableShipStatsAPI` per-frame `modifyFlat(STAT_MOD_ID, setting value - base value)` (same id overwrites, no stacking); takes effect immediately after saving.
- CSV write-back detail: row-region replacement (`csvFieldSpan` locates the target field's raw span) - replace only the field value, not the whole row, to avoid breaking desc/tags columns containing commas/quotes; double-quoted fields are escaped per CSV rules (`"` -> `""`).
- Lesson: not every "number" in a data file can be changed at runtime; first check whether a stats interface exists, then decide "runtime override" vs "write-back and restart".

**20. 0.98a API method name pitfalls (visible at compile time)**
- `ShipAPI` has **no** `getFaction()`; `ShipHullSpecAPI` has **no** `getMaxSpeed()/getAcceleration()/getDeceleration()`; `MutableShipStatsAPI` has `getEffectiveArmorBonus()`/`getArmorBonus()` but **no** `getEffectiveArmor()`.
- Correct access: base values like speed use `stats.getMaxSpeed().getBaseValue()` (raw value without any modifier); hull/armor/flux use `spec.getHitpoints()/getArmorRating()/getFluxCapacity()/getFluxDissipation()`.
- No direct faction API in combat: Tesseract (Omega) ships share the vanilla `tesseract` hull; matching by `ship.getHullSpec().getHullId()` + variant id prefix fully covers it (`HijackUtil.isOmega`).
- Lesson: before writing a new API call, check method names with `javap -cp starfarer.api.jar <interface>` instead of guessing.

### 3.7 v2.1.0 new mechanics and slimmed sliders (2026-10)

**21. Boarding target toggles (phased / all-shield) and Omega hack speed multiplier**
- Three new adjustable items: `boardPhasedAllowed` (default no), `boardOmniShieldAllowed` (default yes), `hackOmegaSpeedMult` (0.05~1, default 0.5).
- Implementation: boarding toggles live in the **unified touch entry** `HijackUtil.addBoardingHit` (shared by regular Boarding Pods and one-shot fighters - one check covers everything), and the Boarding Pod target search, touch-target invalidation checks, and one-shot fighter target selection all exclude untouchable targets (avoiding "approached but not counted, wasted touch cycle").
- Phased check via `ShipAPI.isPhased()`; all-shield = `CombatEntityAPI.getShield()` is OMNI type and raised (`ShieldAPI.getType() == ShieldType.OMNI && isOn()`).
- The Omega coefficient sits at the end of the CaptureArray rate formula: `rate = base x ewar x size x multi-ship multiplier x omegaMult`; omegaMult != 1 only when `HACK_OMEGA_ALLOWED && isOmega(target)`.

**22. Sunk-return fraction = 0 disables the return mechanic**
- `checkGhostRelease` skips when `hits <= 0 || GHOST_RETURN_FRAC <= 0f` - setting it to 0 is equivalent to disabling "one-shot fighters on target destruction" without a separate toggle.

**23. Slimming redundant sliders (derived by linked formulas)**
- Principle: Mod Settings only exposes the "frigate tier" adjustable items (`arrayOpFrigate`, `boardingHitsFrigate`); destroyer/cruiser/capital are always derived by formula (rounded), avoiding contradictory sliders:
  - Hacking Array OP: destroyer x1.9 / cruiser x2.8 / capital x3.8 (default 8 -> 15/22/30, exactly reproducible)
  - Hits required: destroyer x1.5 / cruiser x2.5 / capital x4.0 (default 25 -> 38/63/100, exactly reproducible)
- Old slider declarations in modSettings.json / LunaSettings.csv removed (stale keys in old saves are no longer read; code uses `arrayOpFor(HullSize)` / `hitsFor(HullSize)` derivation; hitsNeeded also goes through derivation).
- The static write-back (`applyStaticDataOverrides`) also writes the four tier OP values using the derived values, keeping data files consistent with settings.
- Lesson: the capital value of the HullSize enum is `CAPITAL_SHIP` (not CAPITAL); confirm with javap before writing switch cases.

---

## 4. Future optimization suggestions (for successors)

- One-shot fighters are generated in "wing units (3 fighters)", so the actual released count is >= expected (rounded up to a multiple of 3); if exact counts are needed, switch to finer-grained generation.
- `tickBattleEndSalvage` and `tickBattleEndEarly` overlap (both are "enemy has no deployments -> takeover"); currently deduped with `EARLY_HANDLED`/`KEY_COPIED`; consider merging in the future.
- The malfunction system currently picks one of "engine/weapons/shields"; it could be extended to combined malfunctions or differentiated by ship type.
- `CAPTURED_MEMBERS` is a strong-reference list; normal flow removes ships one by one in `postBattleRecover`; if a battle is aborted abnormally (player force-quit), the cache may persist until the next call - currently backstopped with "remove if already in the player fleet"; keep observing.
- All static state tables use weak references (WeakHashMap / WeakSet); hulks are cleaned up by the engine and disappear automatically - no manual cleanup needed.

---

## 5. References and credits

This mod referenced the code and data organization of the following mods during development (see the author's note in mod_info.json):

- Ship and Weapon Pack (id=`swp`, author DarkRevenant)
- Iron Shell (id=`timid_xiv`, authors Techpriest & Selkie & Avanitia)
- Superweapons Arsenal 3.0 (id=`superweapons`, authors Mira / KindaStrange / mllhild) - skeleton source of this mod
- LunaLib (id=`lunalib`, author Lukas04) - Mod Settings dependency
- Vanilla ShardSpawner / ShardFadeIn / ShardFadeOut (decompiled reference for hidden-entity lifecycle management)

*This note is maintained with each mod version; updated after every round of in-game acceptance testing.*

## 6. v2.2.0: enemy-can-use toggle + Mod Settings presets

### Can enemy ships install it (enemyCanUse, default yes)
- Requirement: control whether enemy ships can also install and use the mod's items (default yes).
- Implementation: new `CaptureConfig.ENEMY_CAN_USE` (LunaSettings General & Preset tab Boolean).
  - The three items (Hacking Array / Boarding Pod / Boarding Driver Core) previously hard-coded `ship.getOwner() != 0 -> return` (player-only);
    changed to `HijackUtil.isEnemyInstalled(ship)` (excludes enemy installers only when the toggle is off).
  - Target judgment made perspective-based: `HijackUtil.isValidTarget(ship, attackerOwner)` overload (target and capturer just need to be on **different factions**, no longer hard-coded owner==0 as the player's target); CaptureArray.validHackTarget(t, ship), BoardingPod touch-target invalidation
    (`curTarget.getOwner() == ship.getOwner()`), and BoardingDriver malfunction judgment (`t.getOwner() == ship.getOwner()`)
    all changed to "by the installer's faction".
  - Capture flow made faction-based: captureShip records `KEY_CAPTURER_OWNER` (installer faction); tickCaptureTransitions faction switch,
    finishCapture fighter-wing ownership, tickCapturedWingAllegiance continuous correction, and postBattle backstop all act on the capturer's faction.
  - **Post-battle takeover is player-only (capturerOwner==0)**: doCaptureTransfer gates this; ships captured by enemies settle through the vanilla enemy-fleet flow and never join the player fleet (avoids wrongly absorbing enemy-captured ships and avoids ship duplication).
  - Malfunctions: the "cleared on owner change" check changed from `owner==0` to the `KEY_CAPTURING/KEY_CAPTURED` markers (independent of concrete faction numbers); checkThreshold gained an installerOwner parameter (no roll when target and installer share a faction).
- Note: vanilla enemies have no way to obtain the items (no blueprints/trade spread), so default "yes" does not change current behavior; it only matters for future spread scenarios.

### Mod Settings preset save / load
- Requirement: save/load the whole configuration in Mod Settings with one click.
- LunaLib survey: LunaSettingsData supports Int/Double/Float/Boolean/String/Color/Keycode/Radio/Text/Header,
  **no standalone button type**; LunaSettings only provides get methods (cannot write back to the UI), so we used:
  **String command fields + value-change detection** (changing back to "none" also triggers once; after changing back to "none" it can trigger again).
- New 3 String settings (General & Preset tab): presetName (name), presetSaveCmd (save command), presetLoadCmd (load command).
- New class PresetManager: save = read CaptureSettings from modSettings.json and write `data/config/presets/<name>.json`;
  load = read preset -> write back to modSettings.json (persistent; consistent with LunaLib after restart) + `CaptureConfig.applyFrom(preset)`
  applied directly to static fields (no disk merged-cache detour; effective immediately in the session; static data items fully effective after restart).
- New class PresetWatcher (EveryFrameScript, campaign-layer polling every second, runWhilePaused=true): detects command value changes to trigger save/load.
- CaptureConfig refactor: the 55 opt assignments in `load()` extracted into `public static void applyFrom(JSONObject)`,
  shared by load() and preset load; new `public static String lunaString(String key)` reflection read for String settings.
- Lesson: the CampaignScript interface is not in `com.fs.starfarer.api.campaign`; campaign-layer scripts must implement
  `com.fs.starfarer.api.EveryFrameScript` (advance + isDone + runWhilePaused).

## 7. v2.2.1: preset rework (toggles + slots)

### Background
- v2.2.0 used "String command fields + value-change detection" (presetName / presetSaveCmd / presetLoadCmd);
  in-game testing showed no visible function (UI stayed at: preset name preset1 / save none / load none).

### New scheme (per user request)
- **Mod Settings gained a "Preset Slots" tab**:
  - Save preset toggle presetSaveEnabled (Boolean, default false)
  - Load preset toggle presetLoadEnabled (Boolean, default false)
  - Slot selected toggles presetSlot1 ~ presetSlot5 (Boolean, default false)
- **Preset storage directory**: mod root `presets saves/` (user-specified), files presets_1.json ~ presets_5.json.
- **Rules**:
  - Save and load toggles **both on or both off -> no action**;
  - Multiple slots on -> use the **lowest-numbered** slot (getSelectedSlot() scans 1 to 5);
  - No slot on -> log a hint only, no action;
  - Triggered by value change (off->on); the user then turns the toggle off manually so it can trigger again.
- Code:
  - PresetManager rewritten: SLOT_COUNT=5 / SLOT_PREFIX=presets_ / SLOT_DIR_NAME="presets saves";
    savePreset(int)/loadPreset(int)/getSelectedSlot(); load keeps "write back to modSettings.json + applyFrom immediate effect".
  - PresetWatcher rewritten: polls the two Boolean toggles; does nothing and syncs remembered values when save==load;
    on trigger calls the PresetManager method.
  - CaptureConfig: removed lunaString (old command scheme only); lunaBool changed from private to public (PresetWatcher/PresetManager read toggles).
- Data files: LunaSettings.csv removed the old 3 command rows, added 8 rows for the preset tab (synced across the three ZH/EN directories);
  modSettings.json removed presetName/presetSaveCmd/presetLoadCmd, added 7 new keys (default false);
  mod_info.json version 2.2.1.
- Cleanup: the old data/config/presets directory does not exist (the old scheme never saved successfully), nothing to clean; the root `presets saves` directory is pre-created.
- Lesson: PowerShell inline strings with escaped quotes easily fail to parse (README update round); use a python script file instead.

## 8. v2.2.2: preset tab cleanup + preset toggles fixed to OFF

### Background (user feedback)
- The "Preset Slots" tab appeared **twice** in LunaLib Mod Settings (duplicate tab in the tree), and the "General & Presets" tab was redundant (enemyCanUse actually displays at the top of the Preset Slots content area).

### Investigation and conclusion
- LunaSettings.csv text has no duplicate rows (cap_header_presets / presetSlot1 each appear once); the duplication is a **LunaLib render-layer** behavior:
  a Header row (fieldName equal to the tab name, e.g. "Preset Slots") renders as a tree entry, and the field rows' tab grouping renders another entry -> the same tab name shows twice.
- Fix: **removed the cap_header_general and cap_header_presets Header rows** (the 4 Header rows from v2.1.0 have fieldName != tab and work fine, so they stay);
  enemyCanUse's tab changed from "General & Presets" to "Preset Slots" (English "Preset Slots"), and the "General & Presets" tab disappears with the Header rows.
- Slot toggle naming: `Slot N selected` (Chinese labels only, irrelevant to EN).

### Preset toggles fixed to OFF
- Requirement: saving a preset must **not** save the ON/OFF state of "Save preset", "Load preset", and "Slot N selected"; they are fixed to OFF.
- Implementation (PresetManager):
  - New constant PRESET_TOGGLE_KEYS = the 7 preset-related keys;
  - savePreset: before writing the slot file, remove these 7 keys from a **copy** of the CaptureSettings (new JSONObject(s.toString())) (never mutate the original);
  - loadPreset: when writing back to modSettings.json, build a merged copy and **re-add** the 7 keys as false before writing into CaptureSettings;
    CaptureConfig.applyFrom is unaffected (the 7 keys do not participate in static fields).
- org.json.JSONObject.remove(String) is available in json.jar (compile-verified).

### Data / deploy
- LunaSettings.csv (three ZH/EN copies): deleted 2 Header rows, enemyCanUse tab changed, slot toggles renamed;
- mod_info.json version 2.2.0 -> 2.2.2;
- Capture.jar: the Chinese dual-directory MD5s match (AE69EA4D93E8D85EA055AE6DB349B134), the English one is independent;
- ZH user guide / README synced.

## 9. Custom Boarding Pod sprite + sprite-swap toggle (version unchanged; extra feature on 2.2.2)

### Requirement (user)
- Generate a pixel-art sprite for the Boarding Pod matching the vanilla style (selected design: base "extreme low-pixel B", minus the hull-breaching teeth -> boarding-craft shape).
- New "sprite swap" toggle in Mod Settings (default on): ON = custom pixel sprite, OFF = vanilla wasp sprite.

### Sprite production flow
- image_gen generated 2048px images (white background, extreme low pixel, nose up, no engine flame) -> user picked after several trial rounds;
- Python (PIL+numpy) processing: four-corner BFS over the white connected region -> dark-body mask protected by MaxFilter(81) dilation (so white wings are not mistakenly removed) ->
  background made transparent -> crop to the body bounding box (6px padding) -> nearest-neighbor resize to 36x42 (2x the vanilla wasp 18x21, within the user's specified range);
- Engine flame is NOT drawn into the sprite - Starsector renders engine exhaust via engineSlots dynamically (the ship file already has two HIGH_TECH engine slots at (-7,4)/(-7,-4), matching the new sprite's twin rear nozzles).

### Sprite-swap toggle implementation
- Data: LunaSettings.csv new `useCustomFighterSprite` (Boarding tab); modSettings.json true in all three copies; CaptureConfig static field + loadFromLuna read;
- Runtime: CaptureCombatPlugin calls tickFighterSprite() every frame - for every ShipAPI in the engine that isFighter() with hullId=sw_boarding_pod and not yet recorded, call
  `ship.setSprite(SpriteAPI)` (ShipAPI.setSprite(SpriteAPI) is an official interface, confirmed via javap);
  SpriteAPI statically lazily cached via Global.getSettings().getSprite(String) (SettingsAPI has a single-arg overload);
  WeakHashMap<ShipAPI,Boolean> records applied fighters: new fighter instances (respawn) are re-applied automatically; destroyed/recalled instances drop their keys automatically;
  on toggle change (lastUseCustom comparison) clear the record and re-apply everything -> settings changes take effect next frame mid-battle;
- Background: the vanilla fighter sprite wasp_ftr.png is only 18x21 pixels; SWP-style mod fighter sprites are on the order of 100~220px;
  display size = sprite's actual pixels (1px ~ 1 game unit); 36x42 is a bit larger than vanilla but fits mod convention; collision is still governed by collisionRadius=15.

### Deploy
- Three copies of graphics/ships/sw_boarding_pod.png (ZH / ZH backup / EN); ship file spriteName stays vanilla (OFF = naturally vanilla, ON = code switch);
- jar recompiled and deployed to all three; Chinese dual-directory MD5s match; version stays 2.2.2 (user requested no formal version bump before the big release).

### Sprite-swap fix (no-sprite bug)
- Symptom: sprite swap fails whether the toggle is ON or OFF; the Boarding Pod fighter shows only engine flame, no hull sprite (transparent).
- Investigation: the sprite file itself is fine (36x42 RGBA, complete body); no setSprite exceptions in the log. Inferred `getSprite(String)` returns null at runtime (or setSprite receives null), so the fighter's main sprite was cleared -> only engine flame rendered.
- Fix (SWP data-layer path + runtime null guard):
  1. ship data `spriteName` changed directly to `graphics/ships/sw_boarding_pod.png` (data-layer fallback = load-time, guaranteed to display, same as how *Ship and Weapon Pack* applies fighter sprites); `center` synced to [18,21] (aligned to the 36x42 new sprite's center).
  2. The vanilla wasp sprite copied to `graphics/ships/sw_boarding_pod_wasp.png` (the OFF-state sprite source is fully controlled).
  3. Code `tickFighterSprite`: when getSprite returns null/throws, do NOT call setSprite (keep the data-layer sprite; never setSprite(null)); on load failure set a failed flag to avoid spamming the log every frame; on toggle change reset the flag to allow retry; added diagnostic logs for matched fighter count / current setting count.
- Lesson: `ship.setSprite(null)` leaves a fighter with only engine flame and no hull - any runtime sprite swap must null-check first; the data-layer spriteName is a more reliable display guarantee than runtime setSprite.

### Sprite-swap OFF-state fix (round 2)
- Symptom: ON state (data-layer spriteName) shows the new sprite fine; switching OFF back to the vanilla sprite fails.
- Root cause: `tickFighterSprite` matched fighters by `ship.getHullSpec().getHullId()`, but in 0.98a fighter instances (no FighterShipAPI type; fighters are ordinary ShipAPI) getHullSpec() can be null or return unexpected values -> matched=0, so setSprite never ran (log had no "Boarding Pod sprite" line to confirm).
- Fix: match by `ship.getVariant().getHullVariantId()` == "sw_boarding_pod_wing" (variant id is unique and stable); added a one-time-per-battle fighter sample diagnostic log (prints the first 5 fighters' hullSpec/variant) for future localization.
- Pending in-game confirmation: whether OFF-state switching succeeds; if setSprite has no effect on fighter rendering (still shows the data-layer sprite), the log will show "matched fighters X, current setting Y" with Y>0 but no visual change - then a data-layer/render-layer scheme is needed.

---

## v3.0 major version update log

### Changes in this version
1. **Boarding-side CR-triggered extra hit**: HijackUtil new `tickBoardingCrBonus(engine, amount)` (engine-time accumulation; linear chance of +1 hit based on target CR), constant KEY_BOARD_CR_TIME in customData; called every frame by CaptureCombatPlugin.advance.
2. **Boarding consumes marines**: HijackUtil new `marinesFor()` (Light 5 / Medium 10 / Heavy 20) and `consumeMarines()` (player side only, owner=0, requires cargo, skipped in simulation battles, one-shot fighters exempt); called on BoardingPod touch, toggle `boardMarinesEnabled`.
3. **Hack-side CR/overload bonus**: CaptureArray hack rate `rate = rate * (1 + HijackUtil.crFactor(target))`, extra +`hackOverloadExtra` while overloaded (toggle `hackOverloadEnabled`); new `progressSnapshot()` public static snapshot for arc rendering.
4. **Progress arc rendering**: CaptureCombatPlugin overrides `renderInWorldCoords` + `drawProgressArc` (GL11 lines: gray full circle + colored arc, boarding orange / hacking cyan, radius = collision radius + 34, line width 9/viewMult, 72 segments, clockwise from -pi/2); toggle `arcRenderEnabled`; when ON, boarding/hack progress floating text is hidden.
5. **Capture blacklist**: HijackUtil new `BLACKLIST_FILE_NAME` (originally the ZH-named file, later renamed, see 10.2), `reloadBlacklist()`, `isBlacklisted()` (# comments / blank lines ignored); blacklist excludes both hacking (CaptureArray.validHackTarget) and boarding (HijackUtil.isValidTarget, captureShip interception); ModPlugin loads at startup + reloads on configReloaded each battle; file created in all three mod roots.
6. **Battle-end backstop**: HijackUtil new `tickBattleEndBackstop(engine, amount)` (enemy's surviving deployed ships zero for 10 s -> enemy fleet orderFullRetreat), BACKSTOP_DELAY=10f, BACKSTOP_TIMER weak-referenced by engine instance; called every frame by CaptureCombatPlugin.advance (while the battle is not over). References the enemyTimer backstop idea from *Boarding Attack*'s RC_MonsterBallEveryFrameCombatPlugin.
7. **Config system**: CaptureConfig adds 13 fields (hackCr* 6 + boardCr* 6 + arcRenderEnabled) with applyFrom/loadFromLuna synced; three modSettings.json copies get 13 new keys; three LunaSettings.csv copies get 13 new rows (Hack tab 6 / Board tab 6 / General tab 1; English copy adds English rows + trailing arcRenderEnabled). **Note**: the English LunaSettings once kept a Chinese tab label for the Heavy sliders (legacy), fixed in 3.0.2.
8. **Version**: 2.2.2 -> 3.0.0; mod_info.json description updated with new mechanics and the warning; credits added *Boarding Attack* (zouyx).

### Pitfall records (for future developers)
- **FleetSide package path**: `com.fs.starfarer.api.combat.FleetSide` does not exist; the correct package is `com.fs.starfarer.api.mission.FleetSide` (javac "cannot find symbol" means this). Always verify other package names with `jar tf starfarer.api.jar`.
- **Compile output mojibake**: javac errors output in GBK; piping `2>&1 | Out-String` in PowerShell swallows the output (exit 1 with no stdout); capture with `$o = & javac ... 2>&1; $o | Select-Object -First N` to display correctly.
- **GL11 arc rendering**: `lwjgl.jar` / `lwjgl_util.jar` must be on the compile classpath (inside the game's starsector-core), otherwise javac cannot find org.lwjgl symbols; the rendering API (GL11.glBegin/glVertex2f/glLineWidth) matches *Boarding Attack*, LWJGL2-compatible.
- **JSON write trap**: replacing strings to write a multi-line description easily produces raw control characters that break JSON parsing; always rebuild the whole JSON with `json.dumps(ensure_ascii=False, indent=2)` and re-validate with a second json.load.
- **PowerShell here-string**: content in `@'...'@` passed to python gets `\n` escaped into real newlines by python source parsing; any text needing literal escape sequences must be assembled with explicit `\n` inside python, not via PowerShell-side escapes.
- **Bilingual jar deploy**: of the three jars/Capture.jar copies, the English directory must hold the English build (renamed from Capture_EN.jar); a Chinese jar was once mistakenly placed there (hash check: A934DE...=ZH, 553194...=EN).

---

## 10. v3.0.0 release-prep wrap-up (2026-10-04)

### 10.1 Uninstall helper: attempt and rollback (important lesson)

- Requirement: allow safely uninstalling the mod from an ongoing save without starting a new game.
- Attempt: new UninstallHelper (LunaLib toggle + one-shot cleanup logic: outside battle, scan the player fleet/stations, strip the mod's hullmods and fighter wings from captured ships, free their OP / replace with same-tier vanilla content) -> wired into CaptureConfig / ModPlugin / PresetManager exclusion keys -> compiled and deployed to all three directories.
- Test result: **did not work reliably** (incomplete/unstable cleanup); the user decided on a full rollback.
- Handling: source fully reverted (backed up in `C:\Users\25283\AppData\Local\Temp\swbuild\uninstall_helper_backup\`); LunaSettings / modSettings / docs all restored to the "cannot safely uninstall" statement:
  captured ships' variants persist this mod's hullmod/fighter ids, so deleting the mod from a save that used it breaks the save on load; **you must start a new game**.
- Conclusion: safe uninstall from an ongoing save needs a mechanism that can traverse and rewrite persisted variants, decoupled from post-battle flows like the battle-end takeover; if the feature is restarted later, first do a "dry-run scan" (count only, no rewrite) to validate the cleanup rules before going live.

### 10.2 Blacklist file rename (pre-release consistency)

- Reason: the blacklist file was originally named in Chinese (a bracketed ZH title), awkward for English environments and forum docs.
- Changes: code constant `BLACKLIST_FILE_NAME` -> `capture_blacklist.txt` (ZH/EN sources synced); file renamed in all three roots;
  ZH user guide (official + backup), README (EN), mod_info.json (ZH/EN description) synced; the forum post FAQ already used the English name.
- Note: the rename must touch **the code constant + the file + all docs** together; renaming only the file with the code unchanged silently breaks the blacklist (it just fails to load).

### 10.3 Progress arc rendering: final conclusion (floating text is the default)

- Attempt direction A (thicker arc + persistent rendering) / B (adjustable alpha) both ineffective; after direction C the arc became fully invisible; multiple investigations (GL11 compatibility, drawArc geometry, view transform) did not root-cause it.
- User decision: **floating text by default**; the arc renderer stays in the code but defaults to OFF (arcRenderEnabled default false), and the Mod Settings toggle remains.
- Docs / forum post / mod_info descriptions unified to: "shown as floating text; the colored arc is experimental, not reliably implemented, defaults to OFF".
- Lesson: if a custom WorldRenderer draw does not appear in a specific scenario, suspect the **render layer / camera culling / alpha compositing** first, not the geometry parameters.

### 10.4 Zip naming convention

- ZH package `ShipCapture_ZH_vX.Y.Z.zip` (Chinese text only inside the archive); EN package `ShipCapture_EN_vX.Y.Z.zip` - **the EN package name must not contain Chinese**.
- The pack script (swbuild/pack_37.py) output names are synced; it excludes .xlsx / presets saves / forum_post_bbcode.txt; after repacking it runs testzip + CSV no-BOM + jar checks.
### 10.5 Tesseract/Omega hijack failure (post-v3.0.0 fix, 2026-10)

- Symptom: with the "Hijack Array: allow hijacking Tesseract/Omega" toggle OFF, the Tesseract is STILL hijacked; adjusting "hijack Omega speed multiplier" has NO effect.
- Root cause (second pass, corrected):
  - In 0.98a the vanilla Tesseract (Omega faction) variants DO carry the automated hullmod (the unmanned-ship check is drone / automated, and the Tesseract falls into the automated category). The first pass wrongly assumed "Tesseract does not carry automated" - that conclusion was wrong and is corrected here.
  - alidHackTarget originally ran if (isUnmanned(t)) return true; return HACK_OMEGA_ALLOWED && isOmega(t); - the Tesseract was let through by isUnmanned first, so the toggle could not stop it.
  - The speed multiplier in the CaptureArray progress code is (HACK_OMEGA_ALLOWED && isOmega(target)) ? HACK_OMEGA_SPEED_MULT : 1f - with the toggle OFF it is always 1, so the multiplier appears dead.
  - The LunaLib read path is fully functional: user settings live in the game's saves/common/LunaSettings/sw_capture.json.data (LazyLib loadCommonJSON resolves the common path to saves\common); the file exists with all fields, and getBoolean("sw_capture", key) reads the user's value.
- Fix: alidHackTarget now checks Omega FIRST and decides by the toggle only: if (isOmega(t)) return HACK_OMEGA_ALLOWED; return isUnmanned(t); - toggle ON allows hijacking (at the configured speed multiplier), toggle OFF excludes the Tesseract entirely. Normal unmanned ships are unaffected.
- Lessons: (1) never assume a hullmod loadout from old versions or hearsay; "unmanned" (drone/automated) and "Omega class" are separate sets requiring separate toggle control, and Omega must be checked BEFORE the unmanned check or automated lets it leak through. (2) LunaLib user settings are not inside the mod folder - they live in the game's saves/common/LunaSettings/ as <modId>.json.data; when a toggle "does nothing", check that file first before suspecting the reflection path.
### 10.6 Tesseract/Omega hijack dual-version packaging (v3.0.0, 2026-10)

- Symptom: after the 10.5 fix (Omega decided by toggle only), hijacking the Tesseract became IMPOSSIBLE - because the user's old save saves/common/LunaSettings/sw_capture.json.data had hackOmegaAllowed:false, overriding the modSettings.json default.
- Decision: stop using a runtime toggle for Omega; ship TWO package versions (ZH and EN each):
  - Omega-Allowed version (A): modSettings.json hackOmegaAllowed=true; LunaSettings.csv drops the "allow hijacking Tesseract/Omega" toggle row (always allowed, no UI needed), keeps the "Omega speed multiplier" slider (adjustable, default 0.5).
  - Omega-Banned version (B): modSettings.json hackOmegaAllowed=false; LunaSettings.csv drops both the toggle row and the speed multiplier row (Omega fully excluded).
  - Code is identical in both versions: CaptureConfig.loadFromLuna() NO LONGER reads LunaLib's hackOmegaAllowed (version-fixed) - the old-save override can never interfere again; HACK_OMEGA_ALLOWED comes only from modSettings.json.
- Packages: ShipCapture_ZH_OmegaAllowed_v3.0.0.zip / ShipCapture_ZH_OmegaBanned_v3.0.0.zip / ShipCapture_EN_OmegaAllowed_v3.0.0.zip / ShipCapture_EN_OmegaBanned_v3.0.0.zip (English names contain no Chinese, per naming rule).
- Layout: live ZH mod folder = Omega-Allowed (A); ZH backup / ShipCapture_EN backup = A; the B-build ZH backups (Omega banned) = B.
- Hijack speed formula (reference for future tuning):
  progress/sec = HACK_BASE_RATE x EW-factor x size-factor x stack-factor x omega-factor x (1 + CR/overload-bonus)
  - EW-factor = clamp(1 + (myEW - enemyEW) x EW_MULT_PER_POINT, EW_MULT_MIN, EW_MULT_MAX)
  - stack-factor = HACK_STACK_POWER ^ (n/(n+1)) (n = arrays locking the same target)
  - omega-factor = (toggle on && target is Omega) ? HACK_OMEGA_SPEED_MULT : 1
  - size-factor = target size mult (frigate 1 / destroyer 0.6 / cruiser 0.35 / capital 0.2, adjustable)
  - CR/overload-bonus = linear bonus when CR <= threshold (default 0.1~0.5, capped at CR 30%); +0.2 when overloaded (toggle, default on)
- Lesson: "runtime toggle" and "version difference" are different needs - when a behavior must be deterministically on or off, fixing it at compile/data level (version packages) beats a runtime toggle; a runtime toggle must also handle old-save overrides.
### 10.7 A-version rollback + B-version explicit exclusion (v3.0.0, 2026-10)

- Symptom: after 10.6 dual-version packaging, the A (Omega-allowed) version STILL failed to hijack the Tesseract in playtests. Code-wise A should let Omega through (isOmega -> HACK_OMEGA_ALLOWED(true), plus isUnmanned fallback), but real-battle behavior was unstable - whether the Tesseract carries automated and whether the isOmega hull/variant-id check works in 0.98a battle was never battle-verified (the "Tesseract carries automated" claim was inferred, not measured in combat).
- Decision (user instruction): roll A back to the logic that was battle-verified to hijack the Tesseract (first-fix version); make B exclude Omega explicitly in code. The A and B jars are now CODE-DIFFERENT (no longer data-only):
  - A (Omega-allowed): if (isUnmanned(t)) return true; return HACK_OMEGA_ALLOWED && isOmega(t); - the Tesseract is let through unconditionally via isUnmanned (automated), with the isOmega branch as a fallback (HACK_OMEGA_ALLOWED fixed true in this version).
  - B (Omega-banned): if (isOmega(t) || isUnboardable(t)) return false; return isUnmanned(t); - Omega excluded with DOUBLE assurance: isOmega by hull/variant id, and isUnboardable by the vanilla UNBOARDABLE tag (verified present on the 0.98a Tesseract in ship_data.csv). Normal unmanned ships (Remnant/derelict drones) are unaffected.
- Build/deploy: 4 jars compiled (A/B x ZH/EN); live ZH folder = A; each backup dir gets its matching jar; all 4 zips rebuilt and verified (testzip OK, in-zip jar hashes match deployed).
- Lesson: whether the Tesseract can be hijacked must be decided by battle testing - inferences about automated hullmods or variant ids may not match reality. Shipping two deterministic semantics (always allow / always exclude) is more robust than one runtime toggle over an uncertain check. The vanilla UNBOARDABLE tag is a reliable discriminator for the Omega class and is used as B's second check.
### 10.8 Version bump to 3.0.1（2026-10）

- The A (Omega-allowed) and B (Omega-banned) versions both passed playtesting; version bumped 3.0.0 -> 3.0.1 (patch: Omega-hacking dual-build fix).
- Updated: 5 mod_info.json files (live ZH A, backup ZH A/B, backup EN A/B) to version=3.0.1; all 4 zips rebuilt and verified (testzip OK, in-zip mod_info version=3.0.1, jar hashes match deployed: A-ZH=5FDB7C8F / B-ZH=D30759D9 / A-EN=D096353E / B-EN=6FAEAA73).
- Old v3.0.0 zips kept (user backup habit, managed by user).


### 10.9 v3.0.2: Marine-shortfall linkage + float counts + percentage floaters + full English localization (2026-10)

- Requirement (user): link the "boarding fighters consume marines" mechanic to boarding counts - with marine consumption enabled (default on), if a boarding contact "would consume marines" but the cargo hold is short, that contact's final count gain is multiplied by the Marine Shortfall Coefficient (default 0.5, adjustable slider); the first shortfall of each battle shows the in-battle message "Not enough marines! Boarding efficiency reduced." (modeled on the vanilla "deployed long; readiness will drop" hint, engine.getCombatUI().addMessage).
- Implementation:
  - HijackUtil.BOARDING_HITS / LAST_HINT_HITS changed from Map<ShipAPI,Integer> to Map<ShipAPI,Float> (the coefficient can produce fractional counts);
  - consumeMarines now returns boolean (short = cargo < need; deducts what is left, does not block the boarding; non-player / no cargo / toggle off return false); added warnMarineShortfall() (combat engine customData, first-time per battle);
  - BoardingPod on successful contact: if consumeMarines reports a shortfall, effectiveGain = gain x CaptureConfig.MARINE_SHORT_MULT;
  - Boarding floaters now show percentage progress: String.format(Locale.ROOT, "Boarding %.2f%%", pct) (current count / total needed x 100%, 2 decimals) - no messy fractional integers;
  - CaptureConfig adds MARINE_SHORT_MULT = 0.5 (LunaLib slider boardMarinesShortfallMult, range 0.05~1).
- Data: the live ZH-A LunaSettings.csv gains the boardMarinesShortfallMult row (Boarding tab) and modSettings.json a default 0.5; all 4 backup dirs synced (EN copies use English descriptions).
- Build/deploy: ZH A/B recompiled (src_b had MISSED the v3.0.2 file sync - fixed by copying the latest HijackUtil/BoardingPod/CaptureConfig/BoardingDriver from src into src_b); EN A/B recompiled. 4 jars, 5 dirs deployed, 4 v3.0.2 zips rebuilt and verified (testzip OK, in-zip jar hashes match deployed: A-ZH=DF923909 / B-ZH=23BD9268 / A-EN=E09D0B05 / B-EN=6CAADC2C).
- Version: 5 mod_info.json files -> 3.0.2; the four v3.0.1 zips moved to the old-version archive folder.
- **Full EN localization (user requirement: NO Chinese anywhere in the EN builds, including all code comments)**:
  - New en_localize.py + 8 translation-map shards (en_map_1~8.json, covering all 16 Java files), whole-line matching replaces every Chinese line in src_en / src_en_b (keys must match file lines byte-for-byte including indentation);
  - Key runtime strings translated accurately: "Boarding %.2f%%", "Not enough marines! Boarding efficiency reduced.", "Hacking X%", "Hack chain +X%", etc.;
  - Then purged Chinese leftovers from EN data/doc files: LunaSettings.csv tab label (24 rows), ship_data.csv design type "Unique", sw_boarding_pod.ship hullName -> "Boarding Pod", blacklist file header comments, README title, and Chinese path/history references inside the development notes.
- Lessons:
  - Localizing code comments needs whole-line map + per-line verification, never keyword replacement: every JSON key must equal the file line (indentation included) to hit; Java-quoted strings inside JSON keys/values need correct escaping (escaped quote followed by the string's closing quote).
  - Cross-directory syncs must be verified against the actual file content, not trusted from "already synced" records: src_b's HijackUtil was still the pre-v3.0.2 version (Integer map / void consumeMarines) while src had the new one; found by comparing after compile, fixed by re-copying the latest files.
  - javac on a Chinese Windows emits GBK-encoded stderr; capture subprocess output and decode with GBK (UTF-8 decoding throws UnicodeDecodeError).
