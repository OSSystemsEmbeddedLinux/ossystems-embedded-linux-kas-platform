# O.S. Systems Embedded Linux — kas Platform

[kas](https://kas.readthedocs.io/) build platform for the **oel** distribution
([O.S. Systems Embedded Linux](https://github.com/OSSystemsEmbeddedLinux)), built
on OpenEmbedded-Core and targeting the **wrynose** Yocto Project release.

kas checks out the required layers, generates `bblayers.conf`/`local.conf`, and
drives BitBake, so a full build is a single command. The repository ships one
ready-to-build configuration per supported machine.

## Contents

- [Prerequisites](#prerequisites)
- [Quick start](#quick-start)
- [Repository layout](#repository-layout)
- [Developer-local setup](#developer-local-setup)
- [Reproducibility](#reproducibility)
- [Maintainer](#maintainer)

---

## Prerequisites

- **`kas`** — install with `pip install kas`, or run it via the
  [`kas-container`](https://kas.readthedocs.io/en/latest/userguide/kas-container.html)
  wrapper, on a host with the usual Yocto/OE build dependencies.

  Nix users can instead enter a ready-made shell that bundles kas and the host
  tools via the [`yocto-env.nix`](https://github.com/OSSystems/yocto-env.nix)
  flake:

  ```sh
  nix develop github:OSSystems/yocto-env.nix
  ```
- **A `local.yml`** — copy the tracked template and edit it with your local
  paths/preferences. It is gitignored (developer-local, not committed):

  ```sh
  cp local.yml.example local.yml
  ```

---

## Quick start

```sh
kas build kas/oel-qemux86-64.yml:local.yml --target core-image-base
```

- Configs are composed left-to-right with `:` (later overrides earlier).
- Pick any machine by choosing the matching `kas/oel-<machine>.yml`.
- The **image/target is passed by hand** with `--target`; it is deliberately not
  baked into a YAML file. With no `--target`, kas falls back to `$KAS_TARGET`,
  then a config's `target:`, then the default `core-image-minimal`.
- Checkout only (no build): `kas checkout kas/oel-qemux86-64.yml:local.yml`
- Interactive BitBake shell: `kas shell kas/oel-qemux86-64.yml:local.yml`

### Run the built image in QEMU

The machines are QEMU targets, so `runqemu` can boot the result directly:

```sh
kas shell kas/oel-qemux86-64.yml:local.yml -c "runqemu qemux86-64 nographic"
```

---

## Repository layout

The `kas/` configuration and the `local.yml.example` template are tracked;
everything else is generated or developer-local.

```
├── kas/
│   ├── base.yml            # repos + defaults + the oel distro (shared foundation)
│   └── oel-<machine>.yml   # one per machine; ties oel to that machine
├── local.yml.example       # template for local.yml (tracked)
└── local.yml               # developer-local settings; gitignored (not committed)
```

| Path                | Tracked? | Purpose                                       |
|---------------------|----------|-----------------------------------------------|
| `kas/`              | yes      | The kas configuration                         |
| `local.yml.example` | yes      | Template for `local.yml`                      |
| `local.yml`         | no       | Your local settings (`.gitignore`)            |
| `sources/`          | no       | Layers checked out by kas (`.gitignore`)      |
| `build/`            | no       | BitBake build directory (`.gitignore`)        |

### `base.yml`

The shared foundation: layer repositories, the default branch
(`wrynose`), `build_system: oe`, and `distro: oel`. It is included by every
machine file and is not built directly.

Repositories: `bitbake` (branch `2.18`), `openembedded-core` (layer `meta`),
`meta-ossystems-base` (provides the `oel` distro), and `ye`.

### `oel-<machine>.yml`

One product file per machine, each including `base.yml` and setting `machine:`.
Available machines (all from OE-core):

`qemuarm` · `qemuarm64` · `qemuarmv5` · `qemuloongarch64` · `qemumips` ·
`qemumips64` · `qemuppc` · `qemuppc64` · `qemuriscv32` · `qemuriscv64` ·
`qemux86` · `qemux86-64`

### `local.yml`

Your personal, machine-local overrides — passed last on the command line.
Gitignored and created from `local.yml.example`; see
[Developer-local setup](#developer-local-setup).

---

## Developer-local setup

`local.yml` holds your host-specific settings and is not committed (it differs
per developer and machine). Create it from the template:

```sh
cp local.yml.example local.yml
```

The template enables passwordless root login for development, via an upstream OE
config fragment:

```yaml
local_conf_header:
  local: |
    OE_FRAGMENTS += "core/yocto/root-login-with-empty-password"
```

Add host-specific overrides in the same block — all optional, with BitBake
defaults applied when omitted:

| Variable              | Purpose                                          |
|-----------------------|--------------------------------------------------|
| `DL_DIR`              | Download cache, shareable across projects        |
| `SSTATE_DIR`          | Shared-state (sstate) cache, shareable too       |
| `OE_TERMINAL`         | Terminal for `devshell` / `menuconfig`           |
| `BB_PRESSURE_MAX_CPU` | CPU-pressure throttle for BitBake scheduling     |

Paths such as `DL_DIR`/`SSTATE_DIR` must be **absolute** — kas runs BitBake with
`HOME` pointed at a private temporary directory, so `${HOME}` will not resolve to
your real home:

```conf
DL_DIR = "/home/you/yocto/downloads"
SSTATE_DIR = "/home/you/yocto/sstate-cache"
```

The `oel` distro already sets `BB_HASHSERVE_DB_DIR ??= "${SSTATE_DIR}"`, so the
hash-equivalence DB follows your sstate cache automatically.

---

## Reproducibility

The repos track branches (`wrynose`, `master`, bitbake `2.18`) with no pinned
commits — convenient for development, but not reproducible. For a release, pin
exact commits with a lockfile:

```sh
kas dump --lock --inplace kas/oel-qemux86-64.yml
```

This writes `kas/oel-qemux86-64.lock.yml`, which kas loads automatically and
which overrides only the commit IDs — branches keep floating for day-to-day work.

---

## Maintainer

Maintained by [O.S. Systems](https://www.ossystems.com.br/). Issues and merge
requests are welcome on the project repository.
