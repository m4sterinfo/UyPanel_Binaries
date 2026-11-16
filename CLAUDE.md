# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Docker-based, host-isolated build system that compiles UyPanel runtime **binaries** —
`nginx`, `nginx_rtmp` (RTMP/FLV), and `php-fpm 8.3` (with ionCube and OPcache) — for several
Linux distributions. There is no application source code here; the deliverable is one
self-contained `out/<target>.tar.gz` per distribution.

## Common commands

```bash
./build_all.sh              # build all targets (debian 12/13, ubuntu 20/22/24, rocky 9)
./build_all.sh debian       # group: all Debian targets
./build_all.sh ubuntu24     # single target
./build_all.sh rocky9       # Rocky Linux 9
./build_all.sh --help

./check_versions.sh         # dry-run: report newer upstream releases
./check_versions.sh --apply # write updates into versions.json
```

- A target is **skipped** if its `out/<target>.tar.gz` already exists. Delete the archive to force a rebuild.
- Build dependencies are baked into the image layer from `build/deps_debian.sh` / `build/deps_rocky9.sh`
  (cached by Docker; refresh with `docker build --no-cache` or `docker builder prune`).
- Compiler output is cached by ccache in per-target Docker volumes `uypanel-ccache-<target>`;
  `docker volume rm uypanel-ccache-<target>` resets one.
- Per-target build logs land in `logs/<target>.log`.
- `out/`, `logs/`, and `downloads/` are gitignored (build artifacts / caches).
- `downloads/` self-prunes: after pre-download, `build_all.sh` deletes every file there that the
  current `versions.json` no longer names (old versions after a bump).

## There are no unit tests or linters

The only "tests" are binary-validation checks baked into `docker/entrypoint.sh`, run inside the
container after compilation and **before** packaging. If any check fails the build aborts and no
archive is produced. They verify each binary exists, reports a sane version, and includes required
modules/extensions (nginx http_ssl/v2/realip/etc., RTMP/FLV module, PHP `curl`/`mbstring`/`pdo_mysql`/
`gd`/`opcache`/`sodium`/…). To exercise them, run a build for the relevant target.

## Architecture / control flow

```
build_all.sh (host)
  ├─ download_deps()      pre-fetch all sources once → downloads/ (mounted RO into every container)
  ├─ docker build         debian/ubuntu share docker/debian/Dockerfile via --build-arg BASE_IMAGE;
  │                       rocky uses docker/rocky/Dockerfile
  └─ docker run -e TARGET=<target>
         └─ docker/entrypoint.sh         dispatch by TARGET
               ├─ build/all.sh           Debian + Ubuntu (auto-adapts to distro/version)
               │   └─ rocky → build/rocky9.sh
               ├─ <binary validation tests>
               └─ package /home/uypanel → /build/out/<target>.tar.gz
```

Key conventions to preserve when editing:

- **`versions.json` is the single source of truth** for both dependency versions and download URLs.
  Build scripts read it at runtime — never hardcode versions or URLs in scripts. Each `url` is an
  ordered **mirror list** tried in turn; `{VERSION}` and `{ARCH}` are substituted. `check_versions.sh`
  (and the weekly `.github/workflows/check-versions.yml` action) auto-bump versions and commit them.

- **Build output goes to `/home/uypanel/bin/{nginx,nginx_rtmp,php}/`** inside the container; the host's
  `out/` is bind-mounted to `/build/out` to receive the archive.

- **PHP extension ordering matters**: in `build/all.sh::main`, ionCube loads first, then OPcache —
  this order is reflected in php.ini and must be kept.

  (Full detail on these conventions is also injected on demand via the live-rules
  plugin when editing `versions.json`, `build_all.sh`, or `build/*.sh`.)

## Adding a new distribution

1. Add a build script under `build/` only if the package base differs (Debian/Ubuntu reuse `build/all.sh`).
2. Register the `TARGET` case in `docker/entrypoint.sh`.
3. Add a CLI case + `build` call in `build_all.sh`.
   The Dockerfile usually does not need changes (BASE_IMAGE handles Debian/Ubuntu variants).
