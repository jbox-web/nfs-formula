# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

This is one of the 32 formula forks under `../` (`github.com/jbox-web/nfs-formula`).
The parent `../CLAUDE.md` describes the shared toolchain, CI and conventions; this file
only records what differs here. **This fork predates the layout the parent describes**:
no `package/` / `config/` / `service/` split, no root `init.sls` (so `nfs` alone is not
a state), no `osarchmap.yaml`, no `libtofs.jinja`, no `test/salt/`.

## Commands

```bash
bundle install
bin/kitchen list                                  # 3 instances: default-<platform>
bin/kitchen test default-debian-13-3008-2         # destroy + converge + verify + destroy
pre-commit run --all-files
```

There is a single suite, `default`. Its `state_top` is `nfs._mapdata` + `nfs.server`,
and its pillar is `pillar.example` itself (`pillars_from_files`), so editing
`pillar.example` changes what the test converges. `kitchen.vagrant.yml` holds upstream's
FreeBSD/OpenBSD Vagrant platforms; the CI does not use it.

Pre-commit lints the whole tree with rubocop and yamllint (`always_run: true`,
`pass_filenames: false`).

## What the formula does

Flat states under `nfs/`, each importing `nfs` from `nfs/map.jinja`:

- `nfs.server` — includes `nfs.service`, installs `pkgs_server`, renders
  `exports_file` (`/etc/exports`) from `nfs/files/exports`. On a change it runs
  `exportfs -ra` (`onchanges`) instead of restarting the service, so client mounts
  survive; FreeBSD alone keeps a `watch_in` restart, plus `sysrc.managed` for
  `nfs:server:mountd_flags`.
- `nfs.service` — `service.running`/`enable` when `enabled`, `service.dead` otherwise.
  `service_name` may be a string (`name:`) or a list (`names:`).
- `nfs.client` — installs `pkgs_client`.
- `nfs.mount` / `nfs.unmount` — include `nfs.client`, then one `mount.mounted` /
  `mount.unmounted` per entry of `nfs:mount` / `nfs:unmount` (state ID = `mountpoint`).
  Per-entry `opts`, `persist`, `mkmnt` fall back to the map's `mount_opts`,
  `persist_mount` / `persist_unmount`, `mkmnt`; `opts` is omitted entirely when neither
  is set (Debian's `mount_opts` is `null`).

## `map.jinja` — the old nested `filter_by`

Four levels, each merged into the previous: `defaults.yaml` (keyed `nfs`, `base='nfs'`)
<- `osfamilymap.yaml` <- `osmap.yaml` <- `oscodenamemap.yaml` (grain `oscodename`, not
in the parent's chain) <- `osfingermap.yaml` <- the **whole** `nfs` pillar. There is no
`nfs:lookup` layer: pillar keys override map keys directly. `osfamilymap.yaml` is the
only level with Debian content (`nfs-common`, `nfs-kernel-server`, service `nfs-server`);
the other three cover Fedora, openSUSE Tumbleweed, Debian 8 and CentOS 6. Its Gentoo
entry uses Jinja, which `import_yaml` renders.

Exports bypass the map: `nfs/files/exports` reads `nfs:server:exports` (`<dir>: <opts>`)
straight from the pillar.

## Tests

- `test/integration/default/` has no `controls/` directory: `kitchen verify` runs zero
  InSpec controls, so a green instance only proves that `nfs.server` converges.
- `nfs._mapdata` runs, but there are no `files/_mapdata/*.yaml` reference dumps.
- `nfs.client`, `nfs.mount` and `nfs.unmount` are not in any suite.

## Known defects and stale upstream text

- `nfs/unmount.sls` renders `- True` / `- False` with no key (the `persist:` key is
  missing).
- `docs/README.rst` gives `opts` a default of `vers=3`; the map default is `null` and no
  `vers=` is ever added. Its testing example (`debian-9-2019-2-py3`) predates the
  platforms in `kitchen.yml`.
- `nfs/*.sls` lack the `# -*- coding: utf-8 -*-` / `# vim:` header that the YAML and
  `_mapdata` files carry.
