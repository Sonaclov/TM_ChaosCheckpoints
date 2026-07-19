# Chaos Checkpoints

A fast, free-roam custom game mode for **Trackmania (2020)**: random spawns, virtual checkpoints scattered across the map, and short rounds of at most 8–10 minutes.

## How it plays

- At round start every player is **spawned at a random landmark** of the map (start, checkpoint or finish block) — nobody starts in the same place.
- A handful of **chaos checkpoints** (3 by default) are active on the map at any time, shown as 3D markers on everyone's screen.
- Chaos checkpoints are **virtual**: they are not part of the map's race route. You claim one simply by **driving within its radius** (12 m by default). First one there takes it.
- Every claimed checkpoint is worth **+1 point** and instantly **respawns somewhere else** on the map, preferably far away from all players.
- Around 15% of checkpoints are **golden ★** and worth **+3 points**.
- **Giving up (Backspace/Delete) is a legit tactic** — it rerolls you to a new random spawn after a short delay. Crossing the map's real finish just respawns you too; the map's route doesn't matter.

### Winner conditions

1. **Instant win:** the first player to reach the points limit (**15** by default) wins the round on the spot.
2. **Timer win:** otherwise, the player with the most points when the round timer runs out wins.

The round timer defaults to **8 minutes** and is hard-clamped by the mode to **60–600 seconds** (max 10 minutes), so rounds always stay short. After the configured number of rounds (2 by default), the player with the highest total wins the map and the server moves on.

## Repository layout

```
Scripts/Modes/TrackMania/ChaosCheckpoints.Script.txt   The game mode (ManiaScript)
MatchSettings/ChaosCheckpoints.txt                     Example match settings
```

The layout mirrors the dedicated server's `UserData/` folder so you can copy it over 1:1.

## Installation (dedicated server)

1. Download the [Trackmania dedicated server](https://doc.trackmania.com/dedicated-server/) and set up your accounts as usual.
2. Copy the files into the server's `UserData/` folder:
   - `Scripts/Modes/TrackMania/ChaosCheckpoints.Script.txt` → `UserData/Scripts/Modes/TrackMania/`
   - `MatchSettings/ChaosCheckpoints.txt` → `UserData/Maps/MatchSettings/`
3. Edit `MatchSettings/ChaosCheckpoints.txt` and replace the placeholder `<map>` entry with your own maps.
4. Launch the server with the match settings, e.g.:

   ```
   TrackmaniaServer /title=Trackmania /game_settings=MatchSettings/ChaosCheckpoints.txt /dedicated_cfg=dedicated_cfg.txt
   ```

You can also switch a running server to the mode with your server controller of choice (PyPlanet, EvoSC, etc.) by setting the script name to `Modes/TrackMania/ChaosCheckpoints.Script.txt` and loading the match settings.

## Settings

| Setting | Default | Description |
| --- | --- | --- |
| `S_RoundTimeLimit` | `480` | Round length in seconds. Clamped to 60–600 (max 10 min). |
| `S_PointsLimit` | `15` | Points for an instant round win. `0` = play the full timer. |
| `S_RoundsPerMap` | `2` | Rounds per map before rotating. |
| `S_NbChaosCheckpoints` | `3` | Simultaneously active chaos checkpoints (1–8). |
| `S_ClaimRadius` | `12.0` | Claim radius in meters around a checkpoint. |
| `S_GoldenChance` | `15` | % chance a checkpoint is golden (+3 instead of +1). |
| `S_MinCheckpointDistance` | `60.0` | Preferred minimum distance from players when a new checkpoint spawns. |
| `S_RandomPlayerSpawn` | `True` | Random spawns/respawns. Disable to always spawn at the map start. |
| `S_ChatMessages` | `True` | Chat announcements for claims and results. |

### Suggested presets

- **Quick chaos (casual lobby):** `S_RoundTimeLimit=300`, `S_PointsLimit=10`, `S_NbChaosCheckpoints=4`, `S_GoldenChance=25`
- **Standard (default):** 8-minute rounds, first to 15, 3 checkpoints
- **Endurance-lite:** `S_RoundTimeLimit=600`, `S_PointsLimit=0` (pure timer, most points wins), `S_RoundsPerMap=1`

## Map recommendations

Because chaos checkpoint locations and spawn points are drawn from the map's landmarks, the mode shines on:

- Maps with **many checkpoints** (15+) spread over the whole map — more landmarks means more varied checkpoint locations.
- **Open / offroad-friendly** layouts where you can cut across the map instead of following the route.
- Flat-ish maps: markers are placed at landmark positions, so heavily stacked vertical maps can produce hard-to-reach checkpoints.

A map with 4 checkpoints in a straight line will technically work, but it will be a very short queue rather than chaos.

## Implementation notes

- Written in **ManiaScript** against the Trackmania 2020 dedicated server script pack; it extends Nadeo's `TrackmaniaBase` mode base and uses the standard `Race` and `Scores` libraries.
- Claiming is a plain **distance check** against `Player.Position` each tick — no dependency on the map's waypoint/route logic, which is what makes the checkpoints "virtual".
- Players that end up unspawned for any reason (join, finish, give up, fall off) are automatically respawned at a new random landmark after ~1.5 s.
- The HUD (top right: round timer, round info, top-5 standings) is a single UI layer fed via `netwrite` variables; the 3D checkpoint markers use the built-in marker system.
- Nadeo occasionally changes library signatures between script pack versions. The touchy calls are centralized on purpose: player spawning lives in `ChaosSpawnPlayer()` and all score handling in the small score helper block near the top of the script, so if your server's script pack ever complains, those are the only two places to adjust.
