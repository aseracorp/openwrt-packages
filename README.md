# OpenWrt Packages — Cosmos Cloud (SnapRAID & mergerfs support packages)

Prebuilt **[Cosmos Cloud](https://cosmos-cloud.io/)** OpenWrt package — the
**primary application** — plus the **SnapRAID** and **mergerfs** support
packages it depends on, for release 25.12.5. Assembled / cross-compiled for
**every supported OpenWrt architecture** and published automatically as an
**apk package repository**.

- ☁️ **Cosmos Cloud** — the main self-hosted server / secure gateway app,
  shipped from its **official prebuilt binaries** (statically-linked Go) + web
  assets, for the architectures upstream publishes (amd64, arm64, 386, armv6,
  armv7, riscv64).
- ⚙️ **SnapRAID** — built from source with the OpenWrt SDK; pulled in as a
  Cosmos dependency on archs where it's built.
- 🔗 **mergerfs** — union filesystem, repackaged from prebuilt static binaries;
  pulled in as a Cosmos dependency on archs where it's built.
- 🗂️ apk format (OpenWrt **25.1+**); one directory per architecture.
- 🤖 Auto-built & re-published on push / schedule / manual dispatch.

---

## Add the repo & trust key (OpenWrt 25.1+, apk)

The repository is served from the `gh-pages` branch. Like official OpenWrt,
**each OpenWrt release version is kept as its own release + gh-pages version
directory** (older versions stay online at their own URLs):

- Versioned feed: `https://aseracorp.github.io/openwrt-packages/<version>/<arch>/packages.adb`
  (e.g. `.../25.12.5/x86_64/packages.adb`)
- Default "latest" alias (always the newest build):
  `https://aseracorp.github.io/openwrt-packages/<arch>/packages.adb`

`<version>` is the OpenWrt release (default `25.12.5`); `<arch>` is the device
package architecture. The large `.apk` files are stored on the matching GitHub
release: `https://github.com/aseracorp/openwrt-packages/releases/download/<version>/…`

`<arch>` is your device's OpenWrt package architecture (e.g. `aarch64_cortex-a72`,
`x86_64`, `mipsel_24kc`, ...). Find it with `apk arch` or the fw-selector.

**One-time setup** (replace `<arch>` with the real value):

```sh
# 1. Install the repository verification public key (required for trust)
mkdir -p /etc/apk/keys
wget -O /etc/apk/keys/aede713bafd53a86.pub \
  https://raw.githubusercontent.com/aseracorp/openwrt-packages/main/keys/aede713bafd53a86.pub

# 2. Add the repository (URL must end in /packages.adb and match YOUR arch)
echo "https://aseracorp.github.io/openwrt-packages/aarch64_cortex-a72/packages.adb" \
  > /etc/apk/repositories.d/customfeeds.list

# 3. Refresh the index and install Cosmos Cloud (pulls in snapraid & mergerfs
#    automatically where built for your arch)
apk update
apk add cosmoscloud
```

> **Important:** include the trailing `/packages.adb` and use your exact
> architecture. If you point apk at a bare directory it falls back to Alpine's
> `APKINDEX.tar.gz` layout against the wrong `/<arch>/` path and fails.

---

## Packages

### Cosmos Cloud (`cosmoscloud`) — primary

Prebuilt Cosmos Cloud (statically-linked Go binaries + web assets) for the
architectures that have an official archive:

| OpenWrt arch | Cosmos binary |
|---|---|
| `x86_64` | amd64 |
| `i386_pentium4` / `i386_pentium-mmx` | 386 |
| `aarch64_*` (cortex-a53/a72/a76/generic) | arm64 |
| `arm_cortex-a7` / `-a9` / `-a15` (armv7) | armv7 |
| `arm_arm1176jzf-s_vfp` (RPi Zero) | armv6 |
| `riscv64_generic` | riscv64 |

```sh
apk add cosmoscloud
# service
/etc/init.d/CosmosCloud enable
/etc/init.d/CosmosCloud start
```

Cosmos Cloud stores its config in `/etc/cosmos/` (set via `COSMOS_CONFIG_FOLDER`).
The runtime tree (binaries `cosmos`, `cosmos-launcher`, `nebula`, `restic`, web
assets, GeoLite DB) is installed under `/opt/cosmos/`.

#### Cosmos Cloud dependencies

`cosmoscloud` depends on the runtime tools it shells out to (bash, curl, yq,
docker/dockerd, samba4-server, avahi-nodbus-daemon for mDNS, and coreutils/disk
utils). It also pulls in two **support packages** of this repo as **conditional
dependencies** — only on architectures where a matching package is built:

- **`mergerfs`** — installed automatically when a mergerfs package exists for
  your arch.
- **`snapraid`** — installed automatically when a snapraid package exists for
  your arch.

So a single `apk add cosmoscloud` brings in all three where available.

> The mips/mipsel/powerpc/loongarch/etc architectures have **no** upstream
> prebuilt Cosmos Cloud binary, so they are not part of the Cosmos feed.

---

### SnapRAID (`snapraid`) — dependency / standalone

Prebuilt **14.9** for **every** OpenWrt architecture (the widest coverage),
includes man pages. Normally installed automatically as a Cosmos dependency; it
can also be installed standalone where its features (parity) are needed:

```sh
apk add snapraid
snapraid --version
```

---

### mergerfs (`mergerfs`) — dependency / standalone

Prebuilt **2.42.0** (static binaries) for the architectures that have an
official static build: amd64 (`x86_64`), i386 (`i386_pentium*`), arm64
(`aarch64_*`), armhf (`arm_cortex-a7/-a9/-a15`), riscv64 (`riscv64_generic`).
Normally a Cosmos dependency; can be installed standalone for a FUSE merged
mount:

```sh
apk add mergerfs
mergerfs --version
```

Installs `/usr/sbin/mergerfs`, `/usr/bin/mergerfs-fusermount`,
`/usr/sbin/mount.mergerfs`, and the man page. (The Linux `fuse` kernel module is
needed to actually mount.)

---

## If the package is unsigned

If `apk` reports `UNTRUSTED signature`, the publishing signing key wasn't
configured (i.e. the `APK_SIGNING_KEY` secret is missing). Install with trust
disabled:

```sh
apk add --allow-untrusted --repository \
  https://aseracorp.github.io/openwrt-packages/<arch>/packages.adb snapraid
```

---

## Build it yourself

Via `feeds.conf`:

```
src-git packages https://github.com/aseracorp/openwrt-packages.git
```

```sh
./scripts/feeds update -a
./scripts/feeds install cosmoscloud snapraid mergerfs
make menuconfig     # select Network > cosmoscloud, Utilities > snapraid/mergerfs
make package/cosmoscloud/compile package/snapraid/compile package/mergerfs/compile
```

> `cosmoscloud` and `mergerfs` require their prebuilt release archives present
> under `package/<name>/prebuilt/` before compiling — the CI workflow fetches
> them automatically.

---

## CI / release workflow

`.github/workflows/build.yml`:
- Repackages Cosmos Cloud (prebuilt) and mergerfs (prebuilt static binaries)
  and builds SnapRAID from source, for every supported architecture, with the
  matching OpenWrt **25.12.5** SDK.
- **Storage split (keeps gh-pages tiny):**
  - The large `.apk` binaries are uploaded to a GitHub **Release**
    (`https://github.com/aseracorp/openwrt-packages/releases/download/<tag>/`),
    one asset per package+arch: `<name>_<version>_<arch>.apk`.
  - The per-architecture index `packages.adb` (just a few KB) is generated with
    an **absolute pkgname-spec** pointing at that release, so `apk` fetches the
    packages straight from the release. Only the tiny index + public key are
    committed to `gh-pages` — no package blobs live in the git repo.
- The whole feed is **signed** (apk EC P-256), so installs verify without
  `--allow-untrusted`.
- Cosmos version and the release tag (OpenWrt version) are configurable via
  `workflow_dispatch` (defaults `0.23.0-unstable013` / `25.12.5`).

### Setting up package signing (OpenWrt apk uses an **EC P-256** key, not usign)

```sh
openssl ecparam -name prime256v1 -genkey -noout -out private.key
openssl ec -in private.key -pubout > <KEYID>.pub   # commit as keys/<KEYID>.pub
```
`KEYID` = the first 16 hex characters of the SHA-512 of the SEC1 uncompressed
EC public point (`0x04||X||Y`). Derive it with:
```sh
openssl pkey -pubin -in <KEYID>.pub -pubout -outform DER | \
  tail -c 65 | openssl dgst -sha512 | head -c 16
```
1. Commit `keys/<KEYID>.pub`.
2. Set `KEYID` in the workflow `env` (already configured).
3. Add `base64 private.key` as the repo secret **`APK_SIGNING_KEY`**.

When the secret is present, the SDK signs each index (`apk mkndx --sign`);
otherwise packages are published unsigned (install with `--allow-untrusted`).

---

## License

- Cosmos Cloud: Apache-2.0 with Commons Clause (© azukaar / Cosmos)
- SnapRAID: GPL-3.0-or-later (© Andrea Mazzoleni, https://www.snapraid.it/)
- mergerfs: GPL-3.0-or-later (© Antonio SJ Musumeci, https://github.com/trapexit/mergerfs)
- This OpenWrt feed packaging: GPL-2.0 (OpenWrt standard)