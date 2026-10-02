# Ship Capture (夺取舰船)

A standalone Starsector mod (0.98a-RC8) that lets you seize enemy ships by hacking and boarding. Captured ships join your fleet after the battle.

## Install
Put the folder into your `mods` directory and enable it in the launcher (game version 0.98a-RC8). **LunaLib** is an optional prerequisite: install and enable it (recommended) to get the in-game Mod Settings UI; without it the mod still runs with the defaults in `data/config/modSettings.json`. This folder is an English version for reference; the Chinese version used in-game is `mods\夺取舰船`.

## Contents

### 1. Hullmod: Hacking Array (`sw_capture_array`)
Hacks and seizes control of enemy **unmanned ships** (Remnant automated ships, drone ships).
- Each equipped ship locks one target in range (prioritizes the target with the highest progress; multiple ships on the same target combine: final effect = base x 2^(n/(n+1))).
- Target leaving range pauses the hack and **keeps accumulated progress**; if another target is available it re-locks, otherwise it stays locked and waits.
- Target destroyed/lost: re-locks and transfers 60% of the old progress to the new target.
- At 100% progress: target is paralyzed for **5 seconds** (capture transition) -> switches to your side.
- Successful capture spreads progress to other hackable targets in **2000** range (+15% x size-multiplier ratio) and triggers hack cooldown (5s x size multiplier / locking ships).
- Values: range 2600; base rate 2%/s; ECM multiplier = 1 + ECM difference x 1.0 (clamped 0.25~5); size multiplier frigate 1.0 / destroyer 0.6 / cruiser 0.35 / capital 0.2.

### 2. Fighter Wing: Boarding Pod (`sw_boarding_wing`)
Bomber fighters that board enemy **manned ships** to seize control.
- Fighter stats: BOMBER, max engagement 4000; OMNI shield; hull 350 / armor 75; crew 15/1; max speed 300; refit time 6; tactic system: flare launcher (decoy); armament: 1x IR pulse laser (fighter).
- Boarding hit: only fighters with CR above 0 can count a hit - touching the target within **120** counts one hit (shared among fighters); after a successful hit ammo and CR are emptied, the fighter returns to resupply, and while CR is zero it cannot count again (no repeated hits).
- Hits needed: frigate **25** / destroyer **38** / cruiser **63** / capital **100**.
- At full count: target paralyzed for **10 seconds** -> switches to your side.
- Targeting prioritizes the player's locked target (R key) within engagement range.
- Purchase: military markets (generic_military), restocked each economy cycle.
- **One-shot Boarding Pod fighters**: when a boarded target (with accumulated hits) is **destroyed** (release **as soon as the wreck forms**), one-shot Boarding Pod fighters (ally side) are **spawned directly at its wreck position**: **expected fighters = floor(hits x sunk-return fraction)** (default 0.5, adjustable in Mod Settings); wings are the spawn unit (3 fighters each; surplus fighters are removed on the spot), so the actual count is >= expected. Fighters are **hard-driven each frame by a mod plugin** (facing + velocity straight toward the nearest manned enemy ship, overriding the vanilla fighter AI's orbiting/long-range/retreat behaviour; if the target dies they re-lock the nearest valid target, and **self-destruct only when no target exists**). On closing to contact range they count a boarding touch like normal pods (transferring the sunk ship's boarding progress to the new target) and **self-destruct after one successful touch** (no wreck, no resupply).

### 3. Hullmod: Boarding Driver Core (`sw_capture_boarding_driver`)
Drives electronic-interference malfunction rolls for all ongoing boarding (0 ordnance cost; links with the Boarding Pod wing only).
- Every **10 seconds**, each ship being boarded is checked once boarding progress passes 40%: chance = 20% + (progress - 40%) x 1% (40% -> 20%, 50% -> 30%, 70% -> 50%, 100% -> 80%).
- On success one random malfunction triggers: engines flame out (cannot move) / weapons lock (cannot fire) / shields fail (cannot raise).
- Durations: engine/shield 5s, weapon 12s; past 70% progress a **critical malfunction** triggers (double duration, 5% hull loss, no hull loss below 15% structure).

## Capture Transition & Post-Battle Ownership (both sides)
1. **Transition**: the target keeps its original side (no state change, captain kept), is paralyzed (no thrust, no turn, engine flameout, hold fire) and enters **effect-free phased state** (vanilla AI cannot target/attack phased ships; alpha forced opaque so no visual flicker). Duration: boarding 10s, hacking 5s (follows battle speed).
2. **Transition end**: switches to your side, exits phase, stops fire and rebuilds default AI; released fighter wings join you (newly released wings are corrected every frame).
3. **Slow repair**: after capture, +0.5% max hull per second, up to 40 ticks (20% total), once per ship.
4. **Battle end (when the enemy has no surviving deployed ships)**: restore original side -> clear captain -> **sink** (damage source = your ship equipped with the capture hullmod, so bounties settle as player kills) -> remove from the enemy battle/campaign fleet -> **directly join the player fleet** -> clear the hulk.

> Result: the captured ship no longer appears in the enemy's "Reserve" in the battle report, appears only on your side, bounties settle correctly, and there is no ship duplication.

## Config
All values live in `data/config/modSettings.json` -> `CaptureSettings` (restart the game to apply; defaults are used if read fails). Key items: `hackRange`, `hackBaseRatePerSec`, `ewMultPerPoint`, `sizeMultFrigate/Destroyer/Cruiser/Capital`, `hackSpreadRange`, `hackSpreadPct`, `hackLockTransferPct`, `hackCooldownBaseSec`, `hackOmegaAllowed` (Hacking Array: allow hacking Tesseract/Omega ships, default **false**), `hackOmegaSpeedMult` (0.5, max 1), `hackStackPowerBase` (2.0), `arrayOpFrigate`, `boardingRange`, `boardingEngageRange`, `boardPhasedAllowed` (default **false**), `boardOmniShieldAllowed` (default **true**), `ghostReturnFrac` (set to 0 to disable the sunk-return mechanic), `boardingHitsFrigate`, `malfChanceBase40`, `malfChancePerProgressPoint`, `malfCheckIntervalSec`, `malfDuration40/70`, `malfDuration40Weapon/70Weapon`, `malfBoardingCheckIntervalSec`, `malfBoardingThreshold`, `malfBoardingMajorThreshold`, `malfBoardingDuration` (engine), `malfBoardingDurationShield`, `malfBoardingDurationWeapon`, `malfBoardingMajorMult`, `malfMajorHullDamageFrac`, `malfMajorHullDamageMinLevel`, `captureDelaySec` (boarding transition, 10), `captureDelayHackSec` (hacking transition, 5), `captureRepairRatePerSec`, `captureRepairCount`.

**Linked sliders (fewer redundant controls)**: only the frigate-tier values are exposed as sliders - `arrayOpFrigate` (Hacking Array OP on a frigate) and `boardingHitsFrigate` (boarding touches to capture a frigate). Other tiers derive automatically: destroyer x1.9 / cruiser x2.8 / capital x3.8 for OP cost, and destroyer x1.5 / cruiser x2.5 / capital x4.0 for boarding hits (rounded).

**Mod Settings (LunaLib) items**: Hacking tab - `hackOmegaAllowed` (Hacking Array: allow hacking Tesseract ships), `hackOmegaSpeedMult` (0.05~1, hack-speed multiplier for Omega ships), `arrayOpFrigate`; Boarding tab - `boardPhasedAllowed`, `boardOmniShieldAllowed`, `boardingHitsFrigate`, `wingBaseValue` (20000), `wingOpCost` (15), `wingFighterCount` (3), `wingMaxSpeed` (320), `wingAcceleration` (350), `wingDeceleration` (250), `wingMinCrew` (1), `wingRefitTime` (6), `wingHitpoints` (350), `wingArmor` (75), `wingFluxCapacity` (300), `wingFluxDissipation` (75).
> Static data items (market value / OP cost / fighters per wing / min crew / refit time / Hacking Array OP, marked with ★) have no runtime API: the mod writes them back to the data files when you save, and they take effect **after restarting the game**. Combat-stat items (max speed / accel / decel / hull / armor / flux) take effect **immediately**.

## Notes
- Hacking Array affects only unmanned ships; Boarding Pod only manned ships; they do not conflict.
- The Boarding Driver Core must be installed on at least one of your ships (0 OP, any ship) for boarding malfunctions to trigger; multiple installations do not stack.
- Transition/repair timing uses engine time (follows battle speed: at 2x speed the boarding transition takes 5s, hacking 2.5s).
- Known limitation: hullmods are not driven by the engine while the battle is in fleet-control (strategy) mode all the way; captures completed in that mode may skip the transition animation (final ownership is unaffected). It recovers after entering combat mode once.

## General Settings & Presets (LunaLib "Preset Slots" tab)

### Enemy ships can install this mod's hullmods/wings (enemyCanUse, default ON)
- ON (default): enemy ships equipped with this mod's hullmods/wings work from the enemy perspective - enemy Hacking Arrays hack YOUR unmanned ships, enemy Boarding Pods board YOUR manned ships, and captured ships go to the ENEMY.
  Note: vanilla enemies have no way to obtain these hullmods/wings (no blueprint/trade spread). The default ON mainly reserves symmetric gameplay for players using mods that spread equipment, or for custom enemy loadouts. Turn it OFF if you do not want enemies to capture your ships.
- OFF: this mod's hullmods on enemy ships are inert (only player-side ships are affected).

### Preset save / load
Use the toggle + 5 fixed slots (presets_1 ~ presets_5) scheme:
1. Select a slot: enable the matching "Slot N selected" toggle in the Preset Slots tab (if multiple slots are ON, the lowest-numbered one is used).
2. Save: turn ON "Save preset toggle", then click Save All. Within ~1s the mod writes the whole configuration to <mod root>/presets saves/presets_<n>.json; the log shows "preset saved to slot N". Turn the toggle OFF after saving; re-enable to save again.
   Note: presets never store the preset-related toggles (Save preset / Load preset / Slot N selected) - those stay OFF no matter their current state when a preset is applied.
3. Load: turn ON "Load preset toggle", then click Save All. The mod applies the selected slot immediately (combat attributes at once; static items such as OP cost / market value fully apply after a game restart) and persists it into modSettings.json. Turn the toggle OFF after loading.
4. Guard: if both toggles are ON or both OFF, nothing happens; with no slot selected, only a log hint is printed.

Presets are stored in the <mod root>/presets saves/ folder (one .json per slot; delete a file to clear that slot). The EN version keeps its own preset folder.
