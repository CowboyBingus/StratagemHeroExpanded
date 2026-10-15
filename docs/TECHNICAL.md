# Stratagem Hero Expanded: technical notes

Steam build 25480438 (EXE 1.8.46015.0). Offsets below are for that build; the mod checks both module hashes and
does nothing on any other build.

## How the arcade picks a stratagem

The arcade's round logic (game.dll `0x5970D0`, run by the arcade system's update `0x984BE0`) starts a round when
the synced round seed changes. It picks a tier from the round number: round 1 tier 4, round 2 tier 5, every
multiple of 3 a random themed tier 0-3, rounds 4-5 tier 6, round 7 tier 7, then tier 8. It seeds an LCG
(`x * 0x5851F42D4C957F2D + 0x14057B7EF767814F`) with the round seed, then calls the picker `0x598370` once for the
current stratagem and once per queue slot: min(round + 4, 15) stratagems per round.

The picker reads two static tables and nothing else:

| Table | game.dll | Layout |
| --- | --- | --- |
| Tiers | `0x263CCC0`, 9 x 68 bytes | `u32 banner` (localized title hash, 0 = none), then 8 x `{u32 group, f32 weight}`; group 0 ends the list |
| Groups | `0x263CF70`, 10 x 64 bytes | up to 16 `u32` stratagem types; 0 ends a group (a full group has none) |

It draws a group by weight, then a stratagem uniformly from that group. Group 0 can never be drawn, since 0 ends a
tier's list. That leaves 9 usable groups of 16 slots. Both tables lie at the start of game.dll's writable data
section (RVA `0x263C000`). The 76 bytes between them belong to other code (`0x263CF28`, `0x263CF38`). The only
other reader of the tier table is a getter for the banner (`0x596E70`).

The arcade's HUD widget (`0x17EED90`) and its input check take each stratagem's code (`+0x40` pointer, `+0x48`
count), icon (`+0xB0`) and HUD type (`+0xB8`, the icon colour) from its StratagemInfo, `[game+0x37CB600 + 8 *
type]`. The widget lays out at most 10 arrows. The StratagemType names are game.dll's own enum strings
(`0x21D4A40`, from entry 12); SOS Beacon is 145 (`0x91`).

### Base game

| Tier | Banner | Groups (weight) |
| --- | --- | --- |
| 0 | BACK-PACKING A PUNCH | 6 (1.0), 1 (0.5), 2 (0.2), 3 (0.05), 8 (0.075) |
| 1 | FLY LIKE AN EAGLE | 4 (1.0), then as tier 0 |
| 2 | RAIN DOWN FROM ABOVE | 7 (1.0), then as tier 0 |
| 3 | OVERWHELMING FIREPOWER | 9 (0.6), then as tier 0 |
| 4-8 | none | 1, 2, 3 and 8, with weight moving from group 1 to group 3 |

The groups hold 27 stratagems: 24 of the 96 loadout stratagems (all from launch) plus Resupply, Hellbomb and
Reinforce. Group 5 (four sentries) is in no tier.

## The change

Both tables are rewritten once per session with 8 groups (108 stratagems, 108 of 144 slots). Each stratagem goes
where its own settings class it: StratagemInfo `+0x188` (enum `0x6011FA2B`) gives 1-6 for orbitals, 7-10 for
Eagles, 11-20 for support weapons (11 melee, 19 expendables, 20 backpack-fed), 21-24 for backpacks, 26-28 for
vehicles, 29-32 for sentries, 33-35 for emplacements, 36 for mines and 0 for mission stratagems.

| Group | Class | Stratagems |
| --- | --- | --- |
| 1 | backpacks | 14, the C4 Pack included |
| 2 | Eagles | 9, Eagle Rearm included |
| 3 | orbitals | 13 |
| 4, 5 | support weapons | 16 + 16, the three melee weapons and the expendables included |
| 6 | sentries, emplacements | 9 + 5 |
| 7 | vehicles, mines | 9 + 4 |
| 8 | mission | 13: Reinforce, Resupply, SOS Beacon, Hellbomb, SEAF Artillery, Super Earth Flag, Upload Data, Seismic Probe, Prospecting Drill, Illumination Flare, Extraction, SEAF Squad, Nuke |

A tier lists at most 8 groups and the classes need 9, so sentries share a group with emplacements, and vehicles
with mines. Neither pair is a theme, and every group's weight follows its size, so sharing changes no
stratagem's odds.

The 96 loadout stratagems are the ones whose settings carry the `selectable` flag in this build. Extraction is
type 148, the landing-zone beacon players call at the extraction terminal; types 7, 98 and 111 are hellpad
variants.

Every tier lists all 8 groups (a tier holds at most 8). Tiers 4-8 weight each group by its size, so each
stratagem has a 1 in 108 chance. Tiers 0-3 keep their banners. Their theme groups (group 1, 2, 3, and 4 + 5)
together get the share vanilla gives its theme: 1.0 of 1.825 in tiers 0-2, 0.6 of 1.425 in tier 3. The remaining
groups share the rest evenly. The round structure, counts, timer and scoring are untouched.

Not used: rewriting the arcade's runtime state each round (`[game+0x3326808]`, queue at `+2672`, synced current
stratagem at `+2824`). It needs a read per frame on the ship and a write per round, misses the first frame, and
would have to replicate the synced field. A detour of the picker would be a code patch.

## Order of work (once per session)

1. Loader API 1 with a log; a second deployed copy does nothing (`StratagemHeroExpanded`).
2. Build gate: SHA-256 of `helldivers2.exe` and `game.dll` (Bingus Shared Runtime's `verify_build`).
3. Stratagem settings loaded: the pointer of the probe type (124, Reinforce) is set. If not, a self-removing
   update hook checks that one pointer each frame and gives up after 600 s of game time.
4. Data check: a data-only game update can change these without changing the binaries. One read of the pointer
   table, then for each of the 108 types its StratagemInfo: it exists, its `+0` type is its own, its code has 1-10
   inputs and a pointer, and its HUD type is 0-4. A stratagem without settings would crash the arcade's widget.
5. One read of both tables. Build 25480438's exact bytes means write; the mod's own bytes mean already applied;
   anything else refuses.
6. Page check: each region of both ranges must be committed `MEM_IMAGE` memory of game.dll, read-write or
   copy-on-write (or their executable forms, since a packed module may keep data pages executable). The runtime's
   checked write accepts only private memory, so the mod queries these pages itself (`VirtualQuery` under the
   runtime's private name). It never changes page protection.
7. Two `WriteProcessMemory` calls (612 and 640 bytes; the 76 bytes between the tables are never written), then
   one read back.
8. At shutdown, one read: the log says whether the tables still held.

## Multiplayer

The cabinet's owner (entity flag `+20` bit 0; expected to be the host) picks the current stratagem and syncs it
(state `+0x20`). Every peer fills its own queue from the shared seed and its own tables, so a peer without the
mod can show a different "up next" strip. The stratagem to enter is always the synced one, and the input check
reads that stratagem's own code.

## Leaderboard

At game over, the online arcade sends the score to `"%s/LeaderBoard/MiniGame/Player"` (miniGameId 4171714171)
with the host's and the interactor's platform account ids. The mod leaves this as it is.

## Cost per frame

| Path | Calls | In-game cost |
| --- | --- | --- |
| Waiting for the stratagem settings (startup only, if they load after the mod) | 1 memory read per frame | unmeasured in game (one ReadProcessMemory, about 1-2 us) |
| The applying frame | 112 memory reads, 2-3 page queries, 2 writes | unmeasured in game (about 0.2 ms of reads and 0.3 ms per page query, once) |
| Every frame afterwards | none: no hook | 0 |
| Shutdown | 1 memory read | once |

Compiled code: none of it is hot; it runs once, interpreted.

## Live checks

2026-10-15, the release build, test harness, packages condition (Shared Loader v19 and this mod only), solo ship,
clean run (ship reached, quit through the engine, no dump, patches restored):
- At load: `applied (108 stratagems in 8 groups; pages 0x04, 0x04), at load.`
- Read from outside the game: both tables hold exactly this build's encoding of the class groups, on
  `PAGE_READWRITE` image pages of game.dll.
- At shutdown: `At shutdown: the arcade still draws from the expanded pool.`

2026-10-09, the first build (the same 108 stratagems, grouped by hand instead of by class), same conditions:
- At load: `applied (108 stratagems in 8 groups; pages 0x04, 0x04), at load.` The stratagem settings were
  already loaded when the loader started addons, so no update hook was installed.
- Read from outside the game: both tables hold the expanded pool; their pages are `PAGE_READWRITE` image memory
  of game.dll.
- The game's own picker (`0x598370`, called from the test agent) drew 4,000 times per tier on the ship: all 108
  stratagems in each of the 9 tiers, never an empty slot.
- At shutdown: `At shutdown: the arcade still draws from the expanded pool.`

Still to do: play the arcade with the mod. That covers every added stratagem's icon and code (icons are looked up
per stratagem, so warbond icons on the ship are unverified), a 9-input code (Bastion), the four themed banners,
the preview strip with two players, and the frame probe's cost.

## Tests

`python scripts/build.py` runs, in LuaJIT and in the game's own `lua51.dll`:
- The vanilla encoding equals bytes copied from the running game's image, byte for byte.
- Every group holds exactly its classes (checked against each stratagem's `+0x188` class from the settings data),
  and each themed group holds all of its class.
- A port of the picker computes every stratagem's odds per tier: vanilla reaches its 27, the expanded tables reach
  all 108 in every tier, with even odds outside themed rounds and vanilla's theme shares in themed ones.
- The mod against a synthetic game.dll in FFI memory, with exact call budgets per scenario
  (`tests/frame_budget.lua`): refusals for every check, failed and lost writes, and the page check on this
  process's own image pages.
- FFI name clashes, hostile update neighbours (below, above, re-hooking) and no C type growth per waiting frame.
- The assembled entry compiles as one chunk, refuses outside the game and runs only once.
