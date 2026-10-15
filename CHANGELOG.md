# v1.0

- The Stratagem Hero arcade draws from all 96 loadout stratagems and 12 common mission stratagems (108) instead of its 27 launch-era ones.
- Themed rounds keep their banners and now hold every backpack, Eagle, orbital or support weapon, grouped by the class the game gives each stratagem; the other rounds give every stratagem the same odds.
- Two tables in game.dll's data are written once per game session, after checking them against build 25480438's exact contents; nothing runs per frame afterwards.
- Every stratagem in the pool is checked for its own settings and a code the arcade can show before anything is written.
