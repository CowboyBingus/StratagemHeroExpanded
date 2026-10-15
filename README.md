# Stratagem Hero Expanded v1.0

The Stratagem Hero arcade on your Super Destroyer draws from every stratagem a loadout can hold, plus the
mission stratagems every squad calls: 108 in all, instead of the 27 launch-era ones it uses in the base game.

**Status:** checked live on the ship: the tables are written at load and still hold at shutdown. With the first
build (the same 108 stratagems, grouped by hand) the game's own picker reached all 108 in every round type. This
build has not yet been played in the arcade. See [docs/TECHNICAL.md](docs/TECHNICAL.md).

## What changes

The arcade picks its stratagems from two fixed tables in the game: rounds of different kinds, each a weighted list
of stratagem groups. In the base game those groups hold 27 stratagems, all from the game's launch (the Railgun,
Eagle 500kg Bomb, Orbital Laser and so on), so every stratagem added since never comes up.

The mod rewrites both tables once per game session:

- **All 96 loadout stratagems**, Warbond ones included: support weapons, backpacks, Eagles, orbitals, sentries,
  emplacements, mines, vehicles and exosuits.
- **12 mission stratagems:** Reinforce, Resupply, SOS Beacon, Hellbomb, Eagle Rearm, SEAF Artillery, Super Earth
  Flag, Upload Data, Seismic Probe, Prospecting Drill, Orbital Illumination Flare and Extraction.
- **Themed rounds stay.** Every third round is still one of "BACK-PACKING A PUNCH", "FLY LIKE AN EAGLE", "RAIN
  DOWN FROM ABOVE" or "OVERWHELMING FIREPOWER". Each now holds every backpack, Eagle, orbital or support weapon,
  as the game itself classes them (the C4 Pack is a backpack, Eagle Rearm an Eagle, the melee weapons support
  weapons). The theme comes up as often as in the base game.
- **Even odds in the other rounds:** every stratagem in the pool is equally likely.

Codes, icons, timing, scoring and the round lengths are the game's own: the mod changes only which stratagems
come up.

## Requirements and install

- Helldivers 2 Steam build **25480438** (EXE 1.8.46015.0). The mod checks the EXE and game.dll hashes and does
  nothing on any other build.
- [Bingus Shared Loader v18 or newer](https://github.com/CowboyBingus/BingusSharedLoader/releases/latest).

Import the release ZIP into Arsenal or HD2MM together with the loader, enable and deploy both, then restart the
game. The log is `%LOCALAPPDATA%\CowboyBingus\Helldivers2\Logs\StratagemHeroExpanded.log`.

Before writing, the mod checks that both tables still hold build 25480438's exact contents. It also checks that
each of the 108 stratagems has its own settings, with a code of 1-10 inputs (the most the arcade can show) and a
known icon colour. It writes only into game.dll's data pages, never into code. On anything unexpected it logs the
reason and changes nothing. At shutdown it logs whether the tables still held.

## Multiplayer

The player whose game runs the cabinet (expected to be the host) decides which stratagem you enter, and that
choice is synced to everyone. Each player's own game draws the "up next" icons, so a player without the mod can
see a different preview strip than the stratagems that actually come. The stratagem you have to enter is always
the right one.

## Leaderboard

The game still sends your final score to its online mini-game leaderboard, as in the base game. With 108
stratagems instead of 27, those scores come from a different, harder pool.

## Cost

- No work per frame once the tables are written.
- At startup: if the game has not loaded its stratagem settings yet, the mod reads one memory value per frame
  until it has. Then, once: 112 memory reads, 2-3 memory-page queries and 2 writes (1,252 bytes); then the mod
  removes its update hook. One memory read at shutdown.
- Not yet measured in the game.

## Build and test

Clone [Bingus Shared Loader](https://github.com/CowboyBingus/BingusSharedLoader) beside this repository (or set
`BINGUS_SHARED_LOADER` to its path). The tests need LuaJIT (`HD2_LUAJIT`, or `luajit` on `PATH`) and the installed
game's `bin/lua51.dll` (set `HD2_LUA51_DLL` if the game is not in the default Steam folder).

```
python scripts/build.py
```

It runs the tests in LuaJIT and in the game's own `lua51.dll` (the game is never started), checks that the
vendored [Bingus Shared Runtime](https://github.com/CowboyBingus/BingusSharedRuntime) files are byte-identical to
the canonical ones when the workspace has them, and writes `releases/Stratagem-Hero-Expanded-v1.0.zip`.

## License

Zero-Clause BSD (0BSD), see `LICENSE`.

**AI disclosure:** Claude Opus 5.5 assisted with research, implementation, tests and documentation.
