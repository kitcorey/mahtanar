# CLAUDE.md

Notes for future Claude sessions working in this repo.

## Why this fork exists

This is a personal fork of [tinwer-group/mahtanar](https://github.com/tinwer-group/mahtanar) — an ESPHome configuration for the Mitsubishi heat pump (MHK2) integration. The fork carries two site-specific commits on top of the upstream release branch:

- `d9d999c` — Apply site config for kitcorey deployment (static IP, secret names, hostname)
- `c891d20` — Enable MHK Fahrenheit correction (pins `external_components` to `muart-group/esphome-components@704211a…` so the in-flight PR #44 F-correction code is picked up; sets `mhk_fahrenheit_correction: true`)

`project.version` is set to `"1.4-mhk-f-correction"` so deployed firmware self-identifies.

## Branch model

- `origin` → `git@github.com:kitcorey/mahtanar.git` (this fork)
- `upstream` → `git@github.com:tinwer-group/mahtanar.git`
- Branch `1.4-release` mirrors `upstream/1.4-release` plus the two local commits above. It tracks **`origin/1.4-release`**, not upstream — pushes go to `origin`.
- The deployable release YAMLs live on the release branches (`1.x-release`). `upstream/main` holds docs (`README.md`, `PCB-REVISIONS.md`) and a tester YAML (`esphome-configs/tester-mahtanar.yaml`) — no production heat-pump config.

## Layout

```
esphome-configs/
  mahtanar-ethernet-default.yaml   ← the file that gets deployed (symlinked from ~/repos/esphome)
  mahtanar-wifi-default.yaml       ← upstream wifi variant; not used at this site
esphome-ecodan-hp-configs/         ← unrelated Ecodan project configs (upstream, landed in e3a36a2); no deploy
README.md
```

Only `mahtanar-ethernet-default.yaml` is wired into the deploy flow.

## Deploy flow

The ESPHome dashboard project at `~/repos/esphome` picks up this config via a symlink:

```
~/repos/esphome/mahtanar-heatpump.yml → ~/repos/mahtanar/esphome-configs/mahtanar-ethernet-default.yaml
```

Edit the file **in this repo** (`esphome-configs/mahtanar-ethernet-default.yaml`), not via the symlink. Then build/upload from `~/repos/esphome`:

```sh
cd ~/repos/esphome
uv run esphome run mahtanar-heatpump.yml
```

(`uv run` matters — the esphome CLI lives in that project's venv.)

Secrets (`api_encryption_key`, `ota_password`) come from `~/repos/esphome/secrets.yaml`. **Do not create a `secrets.yaml` in this repo** — the deploy resolves secrets from the esphome project directory, and a stray one here will only cause drift.

## Hardware / network

- Device: Mitsubishi heat pump with ethernet adapter board
- Static IP: `192.168.20.83`
- Hostname / ESPHome `name`: `heatpump` (friendly: `Heat Pump`)
- ESPHome `project.name`: `tinwer.mahtanarm`
- Note: on the ethernet variant, the MAC the router sees may not match the ESP's wifi MAC — the ethernet PHY has its own. Don't chase a "missing" wifi MAC in DHCP leases.

## Upgrading to a new upstream release

When tinwer-group cuts e.g. `1.5-release`, replay the two local commits:

```sh
git fetch upstream
git checkout -b 1.5-release upstream/1.5-release
git cherry-pick d9d999c c891d20    # site config, then MHK F correction
# resolve any conflicts (most likely in mahtanar-ethernet-default.yaml around project.version / external_components pin)
git push -u origin 1.5-release
```

Then update the symlink target in `~/repos/esphome` if the filename has changed on the new branch, and re-deploy.

When `muart-group/esphome-components` rolls `dev` → `main` and includes the F-correction, bump the `external_components` pin to a newer dev SHA or to `@main`, and consider whether the `mhk-f-correction` version suffix is still meaningful.
