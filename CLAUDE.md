# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

GOW (Games on Whales) is a collection of Docker images consumed by [Wolf](https://github.com/games-on-whales/wolf) to stream games and desktop apps from a remote host. There is almost no application code here — the repo is Dockerfiles, shell startup scripts, Wolf app definitions (`wolf.config.toml`), and a Hugo documentation site ("Wildlife"). Images are published to `ghcr.io/games-on-whales/<name>` and `docker.io/gameonwhales/<name>`.

## Image hierarchy (the central concept)

Builds are layered, and a child image must be built/available before its children:

```
images/base        → ubuntu:25.04 + entrypoint + /opt/gow bash-lib (the GOW runtime)
images/base-app    → base + Wayland/X11 stack (gamescope, sway 1.9, mangohud, waybar), launch-comp.sh
images/base-emu    → base-app + emulators (RetroArch, PCSX2, RPCS3, Dolphin, etc.) downloaded as AppImages
apps/<name>        → built FROM base-app (or base-emu for emulators)
```

Dockerfiles take these build args, wired up by CI: `BASE_IMAGE` (→ base-app builds), `BASE_APP_IMAGE` (→ app/emu builds), and `IMAGE_SOURCE` (set as the OCI source label). A Dockerfile starts with `FROM ${BASE_APP_IMAGE}` and never hardcodes a base tag.

## How a container runs (the startup chain)

1. `images/base/.../entrypoint.sh` runs as root, sources `/opt/gow/bash-lib/utils.sh`, executes every `/etc/cont-init.d/*.sh` (user/device/nvidia setup), then drops to the `UNAME` user via `gosu` and execs `/opt/gow/startup.sh`.
2. `base-app` overrides `startup.sh` to wait for the X server (`wait-x11`), then exec `/opt/gow/startup-app.sh`.
3. Each app provides **`startup-app.sh`** (copied from its `build/scripts/startup.sh`). It typically does:
   ```bash
   source /opt/gow/launch-comp.sh
   launcher /path/to/binary
   ```
4. `launcher()` (in `base-app/build/scripts/launch-comp.sh`) wraps the binary in **gamescope** (`RUN_GAMESCOPE`), **sway** (`RUN_SWAY`), or runs it directly — controlled by env vars set in the app's `wolf.config.toml`.

`/opt/gow/bash-lib/utils.sh` provides shared helpers (`gow_log`, `github_download`) that scripts source rather than reimplement.

### Worked example: ZSNES

`apps/zsnes` is a minimal emulator app — a good template for tracing the whole flow:

1. **Wolf launches the container.** `apps/zsnes/assets/wolf.config.toml` tells Wolf to run `ghcr.io/games-on-whales/zsnes:edge` with `env = ['RUN_SWAY=true', 'GOW_REQUIRED_DEVICES=/dev/input/* /dev/dri/* /dev/nvidia*']` and a `base_create_json` granting `NET_RAW`/`MKNOD`/`NET_ADMIN` caps and input/dri device cgroup rules.
2. **The image was built** from `apps/zsnes/build/Dockerfile` (`FROM ${BASE_APP_IMAGE}`), which downloads SUPERZSNES into `/opt/zsnes` and copies `scripts/startup.sh` → `/opt/gow/startup-app.sh`.
3. **Boot as root:** `base`'s `entrypoint.sh` runs the `cont-init.d` scripts (user/device/nvidia setup), then `gosu`-drops to the `retro` user and execs `/opt/gow/startup.sh`.
4. **base-app's `startup.sh`** waits for X11, then execs `/opt/gow/startup-app.sh` — ZSNES's script:
   ```bash
   source /opt/gow/launch-comp.sh
   launcher /opt/zsnes/SUPERZSNES
   ```
5. **`launcher()`** sees `RUN_SWAY=true`, so it starts a sway session (copying `/cfg/sway/config`, appending the resolution and `exec /opt/zsnes/SUPERZSNES`) via `dbus-run-session -- sway`. The emulator renders into sway, which Wolf captures and streams.

To add a similarly simple app, copy this four-file layout, set the `RUN_*` env in `wolf.config.toml`, and point `launcher` at your binary.

## Anatomy of an app (`apps/<name>/`)

- `build/Dockerfile` — installs the app on top of `${BASE_APP_IMAGE}`, copies `scripts/startup.sh` to `/opt/gow/startup-app.sh`.
- `build/scripts/startup.sh` — the launch logic (see chain above).
- `assets/wolf.config.toml` — the `[[apps]]` block Wolf reads: title, icon URL, and `[apps.runner]` (docker image tag, `env` like `RUN_SWAY=true` / `GOW_REQUIRED_DEVICES`, and `base_create_json` for caps/devices).
- `_index.md` — Hugo content page shown on the Wildlife docs site (apps/ is mounted as `content/apps`).

To add a new app: create this directory layout, then register the image name in the build matrix in `.github/workflows/auto-build.yml` (`apps:` for normal apps, `emus:` for emulator images built on base-emu).

## Build & dev commands

Build locally (run from repo root; the trailing `.` / context matters):
```bash
# Base layers (only if you changed them)
docker build -t gow/base images/base/build .
docker build -t gow/base-app --build-arg BASE_IMAGE=gow/base images/base-app/build .

# An app against the published base-app
docker build -t gow/<name>:custom \
  --build-arg BASE_APP_IMAGE=ghcr.io/games-on-whales/base-app:edge \
  apps/<name>/build .
```
To test in Wolf, point the `image` field under `[apps.runner]` in Wolf's config at your locally built tag.

Documentation site (Hugo):
```bash
bin/build-website.sh   # runs `hugo --gc --minify` in website/, then build-toml.sh
bin/build-toml.sh      # concatenates every apps/*/assets/wolf.config.toml → website/public/apps.toml
```
The Hugo theme is a git submodule (`website/themes/hugo-book`); run `git submodule update --init` before building the site.

`bin/gen-moonlight-image.sh` regenerates the gradient `icon.png` files for apps (Python + Pillow).

## CI

`.github/workflows/auto-build.yml` triggers on pushes/PRs touching `images/**`, `apps/**`, or workflows. It builds the base layers first, then fans out apps and emus via the reusable `docker-build-and-publish.yml`. PRs build to tarball artifacts (no push); pushes to `master` push multi-arch manifests to GHCR and Docker Hub and refresh `buildcache-*` images. Forks push under their own GHCR namespace.

## Conventions

- Published image references use the `:edge` tag (latest master build).
- Dockerfiles clean apt lists (`rm -rf /var/lib/apt/lists/*`) in the same `RUN` that installs, to keep layers small.
- Emulators/tools are pulled as release AppImages/tarballs via the `github_download` / `gitlab_download` / `forgejo_download` helpers, not from apt, so versions track upstream releases.
