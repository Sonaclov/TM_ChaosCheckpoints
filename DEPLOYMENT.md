# Chaos Checkpoints — Deployment Manual

This manual walks you through deploying the Chaos Checkpoints game mode on a
Trackmania (2020) dedicated server, from a blank machine to a public server
running the mode, plus how to operate and update it afterwards.

**Contents**

1. [What you are deploying](#1-what-you-are-deploying)
2. [Prerequisites](#2-prerequisites)
3. [Step 1 — Download the dedicated server](#3-step-1--download-the-dedicated-server)
4. [Step 2 — Create a server account](#4-step-2--create-a-server-account)
5. [Step 3 — Install the mode files](#5-step-3--install-the-mode-files)
6. [Step 4 — Add maps](#6-step-4--add-maps)
7. [Step 5 — Configure the server](#7-step-5--configure-the-server)
8. [Step 6 — Launch](#8-step-6--launch)
9. [Step 7 — Verify the deployment](#9-step-7--verify-the-deployment)
10. [Running as a service (production)](#10-running-as-a-service-production)
11. [Operating the server](#11-operating-the-server)
12. [Updating the mode](#12-updating-the-mode)
13. [Troubleshooting](#13-troubleshooting)
14. [Quick checklist](#14-quick-checklist)

---

## 1. What you are deploying

Two files from this repository go onto the server, mirroring the server's
`UserData/` folder:

| Repository file | Server destination |
| --- | --- |
| `Scripts/Modes/TrackMania/ChaosCheckpoints.Script.txt` | `UserData/Scripts/Modes/TrackMania/` |
| `MatchSettings/ChaosCheckpoints.txt` | `UserData/Maps/MatchSettings/` |

The script is loaded by name (`Modes/TrackMania/ChaosCheckpoints.Script.txt`)
from the match settings file. No compilation step exists: ManiaScript is
compiled by the server when the mode loads, so "deploying" is copying files
and restarting (or reloading) the mode.

## 2. Prerequisites

- A machine with a public IP (or correctly forwarded ports) running
  **Windows or Linux (x64)**. 1 vCPU / 1 GB RAM is plenty for a single server.
- A **Ubisoft account that owns Trackmania** (needed once, to create the
  dedicated server account).
- Open / forwarded ports (defaults):
  - **2350 UDP and TCP** — game traffic
  - **5000 TCP** — XML-RPC, only needed if you use a server controller; keep
    it firewalled from the internet either way
- On Linux: `unzip` (or `p7zip`) to extract the package. The server is a
  self-contained binary; no other runtime is required.

## 3. Step 1 — Download the dedicated server

Official documentation: <https://doc.trackmania.com/dedicated-server/>

```bash
# Linux example
mkdir -p /opt/trackmania && cd /opt/trackmania
wget https://files.v04.maniaplanet.com/server/TrackmaniaServer_Latest.zip
unzip TrackmaniaServer_Latest.zip
chmod +x TrackmaniaServer
```

On Windows, download the same zip, extract it somewhere like
`C:\TrackmaniaServer\`, and use `TrackmaniaServer.exe` in the commands below.

The extracted tree contains, among others:

```
TrackmaniaServer            (the executable)
UserData/
  Config/                   (dedicated_cfg templates)
  Maps/
    MatchSettings/
  Scripts/
Logs/
```

## 4. Step 2 — Create a server account

The server authenticates against Nadeo's master server with its own account
(separate from your player account):

1. Go to your player page at <https://www.trackmania.com> (log in with the
   Ubisoft account that owns the game).
2. Open the **Dedicated servers** section and create a new server account.
3. Note the generated **server login** and **password** — they go into
   `dedicated_cfg.txt` in Step 5.

One server account = one running server instance. Create one account per
instance if you plan to run several.

## 5. Step 3 — Install the mode files

Copy the two files from this repository into the server's `UserData/` tree:

```bash
# from a clone of this repository
cp Scripts/Modes/TrackMania/ChaosCheckpoints.Script.txt \
   /opt/trackmania/UserData/Scripts/Modes/TrackMania/

cp MatchSettings/ChaosCheckpoints.txt \
   /opt/trackmania/UserData/Maps/MatchSettings/
```

Create the `Scripts/Modes/TrackMania/` directory if it does not exist yet.
The path matters: the match settings reference the script as
`Modes/TrackMania/ChaosCheckpoints.Script.txt`, resolved relative to
`UserData/Scripts/`.

## 6. Step 4 — Add maps

1. Put your `.Map.Gbx` files under `UserData/Maps/` (subfolders are fine,
   e.g. `UserData/Maps/Chaos/`).
2. Edit `UserData/Maps/MatchSettings/ChaosCheckpoints.txt` and replace the
   placeholder map entry:

```xml
<map>
	<file>Chaos/MyChaosMap.Map.Gbx</file>
</map>
<map>
	<file>Chaos/AnotherMap.Map.Gbx</file>
</map>
```

Paths are relative to `UserData/Maps/`. Any valid `TM_Race` map works, but
the mode plays best on **open maps with 15+ checkpoints spread across the
whole map** — landmarks are the pool for both random spawns and virtual
checkpoint locations (see the README's map recommendations).

## 7. Step 5 — Configure the server

Copy the template config and edit it:

```bash
cd /opt/trackmania/UserData/Config
cp dedicated_cfg.default.txt dedicated_cfg.txt
```

The minimum you must change in `dedicated_cfg.txt`:

```xml
<masterserver_account>
	<login>YOUR_SERVER_LOGIN</login>
	<password>YOUR_SERVER_PASSWORD</password>
</masterserver_account>

<server_options>
	<name>My Chaos Checkpoints Server</name>
	<comment>Random spawns, grab checkpoints, first to 15!</comment>
	<max_players>32</max_players>
	<!-- leave passwords empty for a public server -->
	<password></password>
	<password_spectator></password>
</server_options>
```

Also worth reviewing in `<system_config>`:

- `server_port` (default 2350) — must match your firewall/port-forward rules.
- `xmlrpc_port` (default 5000) and `xmlrpc_allowremote` — keep
  `xmlrpc_allowremote` set to `False` unless a controller runs on another
  host, and never expose the port publicly.

Never commit `dedicated_cfg.txt` anywhere: it contains the server password.

## 8. Step 6 — Launch

From the server directory:

```bash
# Linux
./TrackmaniaServer /title=Trackmania \
    /dedicated_cfg=dedicated_cfg.txt \
    /game_settings=MatchSettings/ChaosCheckpoints.txt
```

```bat
:: Windows
TrackmaniaServer.exe /title=Trackmania ^
    /dedicated_cfg=dedicated_cfg.txt ^
    /game_settings=MatchSettings/ChaosCheckpoints.txt
```

Notes:

- `/dedicated_cfg` is relative to `UserData/Config/`, `/game_settings` is
  relative to `UserData/Maps/` — pass them exactly as above.
- The process detaches on Linux (`nodaemon` is available via
  `/nodaemon` if you want it in the foreground, e.g. for containers).

## 9. Step 7 — Verify the deployment

1. **Check the logs.** In `Logs/`, the newest `ConsoleLog.*.txt` should show
   the master server login succeeding and the map loading. A mode script
   error (bad path, compile issue) is printed here — see
   [Troubleshooting](#13-troubleshooting).
2. **Find the server in-game.** Live → Servers (arcade room list). Filter by
   your server name. It can take a minute to appear after first login.
3. **Join and sanity-check the mode:**
   - You spawn at a random landmark, not necessarily the start block.
   - Blue `◆ +1` / gold `★ +3` markers are visible on screen.
   - Driving into a marker awards points, posts a chat message, and the
     marker jumps elsewhere.
   - The HUD (top right) shows the round timer counting down from 8:00
     (default), the round/points header, and the standings.
   - Pressing the give-up key respawns you somewhere else after ~1.5 s.

## 10. Running as a service (production)

### systemd (Linux)

Create a dedicated user and a unit file so the server survives reboots:

```bash
sudo useradd -r -d /opt/trackmania -s /usr/sbin/nologin trackmania
sudo chown -R trackmania:trackmania /opt/trackmania
```

`/etc/systemd/system/trackmania-chaos.service`:

```ini
[Unit]
Description=Trackmania Dedicated Server - Chaos Checkpoints
After=network-online.target
Wants=network-online.target

[Service]
Type=forking
User=trackmania
WorkingDirectory=/opt/trackmania
ExecStart=/opt/trackmania/TrackmaniaServer /title=Trackmania /dedicated_cfg=dedicated_cfg.txt /game_settings=MatchSettings/ChaosCheckpoints.txt
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now trackmania-chaos
sudo systemctl status trackmania-chaos
```

If you prefer a foreground process (simpler supervision, containers), add
`/nodaemon` to `ExecStart` and change `Type=forking` to `Type=simple`.

### Firewall (ufw example)

```bash
sudo ufw allow 2350/udp
sudo ufw allow 2350/tcp
# do NOT open 5000 publicly; controllers should connect from localhost
```

## 11. Operating the server

### Changing mode settings

All gameplay options (round length, points limit, extras like combos,
frenzy, overtime, shuffle…) live in
`UserData/Maps/MatchSettings/ChaosCheckpoints.txt` — see the settings table
in the README. Edit the file and restart the server (or reload the match
settings from your controller) to apply. Settings changed while a round is
running are picked up live by most options (e.g. claim radius, checkpoint
count), but round timing is computed at round start.

### Server controllers (optional but recommended)

For admin commands, live map/matchsettings management and records, run a
controller against the XML-RPC port:

- **PyPlanet** — <https://pypla.net>
- **EvoSC** — <https://github.com/EvoEsports/EvoSC>

Point the controller at `127.0.0.1:5000` with the credentials from the
`<authorization_levels>` section of `dedicated_cfg.txt` (change the default
`SuperAdmin` password!). To switch a running server to this mode from a
controller, set the script name to
`Modes/TrackMania/ChaosCheckpoints.Script.txt` and load the match settings
file.

### Logs

- `Logs/ConsoleLog.*.txt` — server console, master server connection, script
  errors.
- `Logs/GameLog.*.txt` — gameplay events.

Rotate or clean these periodically; they grow forever otherwise.

## 12. Updating the mode

1. Pull the new version of this repository.
2. Copy `Scripts/Modes/TrackMania/ChaosCheckpoints.Script.txt` over the old
   one on the server (and the match settings file if new settings were
   added — new settings fall back to their defaults if absent).
3. Restart the server, or use your controller to restart the map/script.
   The script is compiled at load time, so a restart is all it takes.

Tip: keep the previous script as `ChaosCheckpoints.Script.txt.bak` on the
server until you have verified a new version loads cleanly.

## 13. Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Log says the game settings file cannot be found | Wrong `/game_settings` path | The path is relative to `UserData/Maps/` — use `MatchSettings/ChaosCheckpoints.txt`. |
| Log says the mode script cannot be found | Script not in `UserData/Scripts/Modes/TrackMania/` | Re-check Step 3; the `<script_name>` in the match settings must match the path under `UserData/Scripts/`. |
| Script compile error in `ConsoleLog` mentioning `Race::Start`, `Scores::…` or a library path | Your server's script pack version has a different library signature | These calls are intentionally centralized: spawning in `ChaosSpawnPlayer()`, scoring in the score-helper block near the top of the script. Adjust the one or two lines the compiler names, per the error message. Open an issue with the log excerpt if you get stuck. |
| Server never appears in the in-game browser | Master server login failed, or ports blocked | Check `ConsoleLog` for authentication errors (wrong server login/password), verify 2350 UDP+TCP are reachable from outside. |
| Players connect but markers/HUD don't show | Mode loaded but a different match settings file is active | Confirm the log shows `ChaosCheckpoints.Script.txt` loading; if a controller is installed, make sure it didn't switch the mode back to Time Attack. |
| "Very few landmarks" warning in chat, checkpoints all in the same spots | Map has too few checkpoints | Use maps with 15+ spread-out checkpoints; the landmark pool is what creates variety. |
| Rounds feel too long/short | Defaults not tuned for your lobby | Adjust `S_RoundTimeLimit` (clamped 60–600 s) and `S_PointsLimit` in the match settings; see the README presets. |

## 14. Quick checklist

- [ ] Dedicated server package extracted, `TrackmaniaServer` executable
- [ ] Server account created; login/password in `dedicated_cfg.txt`
- [ ] `ChaosCheckpoints.Script.txt` in `UserData/Scripts/Modes/TrackMania/`
- [ ] `ChaosCheckpoints.txt` in `UserData/Maps/MatchSettings/` with real maps
- [ ] Maps copied under `UserData/Maps/`
- [ ] Ports 2350 UDP+TCP open; 5000 TCP closed to the internet
- [ ] Server launches, log shows master server login + mode loading
- [ ] Joined in-game: random spawn, markers visible, claims score points
- [ ] (Production) systemd unit enabled, SuperAdmin password changed
