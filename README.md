# mc

packwiz modpack for the private Fabric server. The pack definition is served at
`https://mc-pack.han-diatonic.uk/pack.toml`; the same manifest drives the itzg
server container and every Prism client.

- Minecraft `26.1.2`, Fabric loader `0.20.1`
- 44 mods: 13 `both`, 23 `client`, 8 `server`
- Client configs ship with the pack: `config/`, `options.txt`, `servers.dat`

## Layout

| Path | Purpose |
| --- | --- |
| `pack.toml` | pack name, version, Minecraft and loader versions |
| `index.toml` | generated file hashes; never edit by hand |
| `mods/*.pw.toml` | mod metadata, no jars |
| `config/`, `options.txt`, `servers.dat` | client files shipped as-is |
| `custom/railway-0.1.0-alpha.8.jar` | server-only mod hosted by direct URL |
| `index.html` | public landing page; not installed into clients |
| `.packwizignore` | keeps `custom/`, `README.md`, `index.html` out of the index |

No mod jars are committed to git except `custom/`. Everything else is resolved
from Modrinth by hash at install time, so client and server get identical files.

## Sides

`side` in each `mods/*.pw.toml` controls who installs it:

| Side | Count | Contents |
| --- | --- | --- |
| `both` | 13 | libraries and shared mods: fabric-api, cloth-config, collective, jamlib, DH, c2me, lithium, krypton, ferrite-core, sound-physics, voice chat, Xaero's maps |
| `client` | 23 | Sodium, JEI, Litematica/MaLiLib, Xaero's deps, UI/visual mods, client-only libraries |
| `server` | 8 | BlueMap + sign markers, clumps, spark, ore/tree/right-click harvest, railway |

Notes:

- `jei` is client-only; it is not needed on the dedicated server.
- Xaero's map mods stay `both` because the server previously ran them.
- Internal files (configs, `options.txt`, `servers.dat`) have no side and are
  written on both ends. That is harmless on the server; the dedicated server
  ignores `options.txt`.

## Day-to-day commands

`packwiz` is installed at `~/.local/bin/packwiz` (main-branch CI build; `~/.local/bin`
is on `PATH`).

Add a mod at a known version, then set its side:

```bash
packwiz mr add <slug> --version-id <Modrinth version ID>
$EDITOR mods/<slug>.pw.toml    # side = "both" | "client" | "server"
packwiz refresh
```

Update everything, or one mod:

```bash
packwiz update --all
packwiz refresh

packwiz update <slug>
```

Remove (equivalent to deleting the metafile and refreshing):

```bash
rm mods/<slug>.pw.toml
packwiz refresh
```

Pin a mod against automated updates (interactive picker):

```bash
packwiz pin
```

Preview locally before publishing:

```bash
packwiz serve     # http://localhost:8080/pack.toml
```

## Release

Bump `version` in `pack.toml`, then mirror the repo to the web root:

```bash
packwiz refresh
rsync -a --delete --exclude .git ./ great-hornbill:/mnt/user/appdata/minecraft/mc/client_pack/
curl -sSI https://mc-pack.han-diatonic.uk/pack.toml
```

`--delete` makes the site an exact mirror of this repo; put anything that must
stay on the host into the repo. Caddy serves `*.toml` as `text/plain` and hides
`.git` already; the rsync excludes it anyway. `Cache-Control: no-cache` is set
site-wide, so a replaced manifest is never served stale.

## Server

The itzg container installs the server side on every start:

```text
PACKWIZ_URL=https://mc-pack.han-diatonic.uk/pack.toml
```

Do not also set `MODRINTH_PROJECTS` — two managers would fight over `mods/`.

First switch only: the installer adds what the pack defines but does not prune
unknown jars, so move the old directory aside:

```bash
ssh great-hornbill 'mv /mnt/user/appdata/minecraft/mc/mods /mnt/user/appdata/minecraft/mc/mods.pre-packwiz'
docker restart minecraft-mc
```

Expected result: 21 jars in `/data/mods` (13 `both` + 8 `server`), including
`railway-0.1.0-alpha.8.jar` fetched from `mc-pack/custom/`.

Server load defaults live in `config/DistantHorizons.toml` (4 threads,
generation distance 2048, request rate 10); spark is server-side for
`/spark tps` and profiling.

## Clients (Prism)

1. Put `packwiz-installer-bootstrap.jar` (v0.0.3 tested) in the instance's
   `.minecraft/` directory.
2. Instance settings → Custom commands → pre-launch:

```text
"$INST_JAVA" -jar "$INST_MC_DIR/packwiz-installer-bootstrap.jar" -g -s client https://mc-pack.han-diatonic.uk/pack.toml
```

3. Launch once. Expected: 36 jars, 72 config files, `options.txt`, `servers.dat`.

Every later launch re-checks the manifest and updates only changed files.
The installer needs the URL reachable; a failed run aborts the launch (no
offline first start).

## Verification

```bash
curl -sSI https://mc-pack.han-diatonic.uk/pack.toml
curl -sSI https://mc-pack.han-diatonic.uk/custom/railway-0.1.0-alpha.8.jar

# full dry-run without touching a real instance:
tmp=$(mktemp -d) && cd "$tmp" \
  && java -jar packwiz-installer-bootstrap.jar -g -s server https://mc-pack.han-diatonic.uk/pack.toml \
  && ls mods | wc -l    # 20
```

## Config behavior

packwiz-installer records what it installed in `packwiz.json` inside the
instance, so changes are selective rather than wholesale:

| Change in this repo | Effect on an existing client |
| --- | --- |
| file untouched | local edits are preserved |
| file edited (or version bumped) | local copy is overwritten at next launch |
| file removed | local copy is deleted at next launch |

Consequences:

- Shipping a config sets it for new clients and leaves existing edits alone
  until you push a change to that file.
- Pushing a config edit overwrites the whole file on every client, so treat
  each shipped file as the authoritative default for that area.
- Removing a file is destructive for players (it is deleted on next launch).
  `options.txt` and `servers.dat` stay in the pack for this reason even though
  they are player-specific.
- `README.md` is in `.packwizignore`; it is published for humans but not
  installed into clients.

## Gotchas

- Mod versions are pinned in `mods/*.pw.toml`. Voice chat (`2.6.27`) and c2me
  (`0.4.0-beta.1.0`) must match on both ends; the pack is the single source of
  truth, so update them once and both sides follow.
- packwiz does not delete jars it does not manage. Clean stale files manually
  after big changes.
- `options.txt` is a frozen baseline (`renderDistance:10`,
  `simulationDistance:6`, keybinds, GUI scale). It ships because removing it
  would delete the file from existing clients, not because it should track
  changes. Editing it resets every client's options on next launch.
- The server start depends on the pack URL being reachable, and the client
  launch depends on it too.
